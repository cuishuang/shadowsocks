# 基于 Golang 实现的 Shadowsocks 源码解析

你好，欢迎来到**洲洋的编程课堂**！

本人邮箱：w910820618@126.com ，欢迎交流讨论学习相关的内容。

欢迎转载，转载请标注 ：**洲洋的编程课堂**。

本教程的Youtube视频在：https://youtube.com/playlist?list=PLRwNnzR6FpVVBs2uYVV_ivZJnfrYAj0GP

本教程的B站视频在：https://www.bilibili.com/video/BV1sy4y1H7KC/

参考项目：https://github.com/shikanon/socks5proxy

---

本教程是基于 **github.com/shadowsocks/go-shadowsocks2**项目，我将通过分析该项目的源码来帮助大家学习如何通过Golang来实现一个隧道代理转发工具。

我会把重点代码罗列出来，方便大家在阅读源码的时候能够找到代码的主线。

本教程主要会从以下四个问题入手：

1. 什么是隧道代理？
2. 本教程的实验环境是什么样的？实现本教程中的示例都需要提前准备哪些条件？
3. Shadowsocks是如何实现隧道代理的？
4. 我们可以学习到哪些技术点？

## 1. 什么是隧道代理？

隧道代理是两个技术的结合，本别是隧道+代理两个技术的结合，分别来解释一下这个词语。

### 1.1 代理 （Proxy）

代理（英语：Proxy）也称网络代理，是一种特殊的网络服务，允许一个网络终端（一般为客户端）通过这个服务与另一个网络终端（一般为服务器）进行非直接的连接。一些网关、路由器等网络设备具备网络代理功能。一般认为代理服务有利于保障网络终端的隐私或安全，防止攻击。

![](https://raw.githubusercontent.com/w910820618/shadowsocks/master/images/proxy.jpg)

上图中，代理服务器既是服务器又是客户端。客户端向代理发送请求报文，代理服务器必须向服务器一样，正确的处理请求和连接，然后返回响应。同时，代理自身要向服务器发送请求，这样，其行为必须像正确的客户端一样，要发送请求并接收响应。

代理服务器的特点：

- 客户端不知道真正的服务器是谁，服务器也不知道客户端是什么样的
- 客户端同代理服务器，代理服务器同服务器，这两者之间使用的通讯协议是一样的
- 代理服务器会对接收的请求进行解析，重新封装后再发送给服务器；在服务器响应后，对响应进行解析，重新封装后再发送给客户端。

### 1.2 隧道 （Tunnel）

隧道（英语：Tunneling ）是一种网络通讯协议，在其中，使用一种网络协议（发送协议），将另一个不同的网络协议，封装在负载部分。使用隧道的原因是在不兼容的网络上传输数据，或在不安全网络上提供一个安全路径。

![](https://raw.githubusercontent.com/w910820618/shadowsocks/master/images/tunnel_pro.jpg)

隧道的特点：

- 该协议是为承载协议自身以外的流量而编写的协议
- 允许数据从一个网络移动到另一个网络
- 只关心流量的传输，不对承载的流量进行解析

## 2. 本教程的实验环境是什么样的？实现本教程中的示例都需要提前准备哪些条件？

![](https://raw.githubusercontent.com/w910820618/shadowsocks/master/images/tunnel.jpg)

上图就是本实验的网络架构图，下面说明一下架构图的含义：

- 其中有两个192.168.0.0和192.168.1.0网段的IP 的虚拟机，用它作为**中转机**；
- 192.168.1.105、192.168.1.106以及192.168.1.107这三台机器作为**目的端**；
- 192.168.0.103 这台机器作为**客户端**；

本实验中所需要的基本开发环境：

- 三台Ubuntu 18.04 的虚拟机
- Golang 的编译环境 （1.15以上）
- IDE 可自选

实验的目的是： 搭建从192.168.0.103 到 192.168.0.104 之间的隧道代理，使 客户端（192.168.0.103）可以访问到 目的端 的内容。

## 3. Shadowsocks是如何实现隧道代理的？

我们以实现TCP隧道代理为例，分别针对客户端和服务端源码中的关键代码来和大家进行分析以下问题：

1. 客户端如何处理数据流
2. 服务端如何处理数据流
3. 目的端IP的读写
4. 数据流的转发是如何实现的

### 3.1 客户端如何处理数据流

我截取了tcp.go中**tcpLocal**函数的源码：

```golang
func tcpLocal(addr, server string, shadow func(net.Conn) net.Conn, getAddr func(net.Conn) (socks.Addr, error)) {
        l, err := net.Listen("tcp", addr) // 监听本地IP
        ...
        for {
                c, err := l.Accept()
                ...
                go func() {
                        tgt, err := getAddr(c)  // 获取目的端IP
                        ...
                        rc, err := net.Dial("tcp", server)  // 向服务端发数据

                        if _, err = rc.Write(tgt); err != nil { // 在数据流中写入目的端IP
                               
                        }
                        
                        if err = relay(rc, c); err != nil { // 数据流 Copy        
                        }
                }()
        }
}
```

在客户端中一共涉及到了三个IP，它们分别是**本地IP**、**服务端IP**、**目的端IP**。

我来解释一下上述代码的含义。

当客户端程序启动的时候，客户端监听本地的代理地址——127.0.0.1:8803，然后我们再把系统中的代理地址设置为127.0.0.1:8803，这个时候通过浏览器访问的数据流就都会从代理地址出去，从而数据流就会进入到客户端程序中。

当客户端程序接收到数据流之后，先从配置文件中找到该数据流要去的目的端IP，并将目的端IP写进数据流中。这样当服务端接收到数据流的时候就可以知道目的端IP。

### 3.2 服务端如何处理数据流

这段是我截取自tcp.go中**tcpRemote**函数的代码。

```golang
func tcpRemote(addr string, shadow func(net.Conn) net.Conn) {
	l, err := net.Listen("tcp", addr) // 监听本地端口
	...
	for {
		c, err := l.Accept()
		...
		go func() {
			defer c.Close()
			...
			sc := shadow(c)
			...
			tgt, err := socks.ReadAddr(sc) // 读取目的端IP
			...
			rc, err := net.Dial("tcp", tgt.String()) // 向目的端发送数据
			...
			if err = relay(sc, rc); err != nil { // 流量转发
				
			}
		}()
	}
}
```

服务端处理数据流的方式与客户端类似，都是先监听本地端口，当接收到请求之后向目的地址进行流量发送。

### 3.3 目的端IP的读写

#### 3.3.1 客户端如何写入目的端IP？

为了让大家可以更清晰地理解代码，我们先回答另一个问题目的端IP是怎么来的？

我们看main.go文件中的代码，

```golang
if flags.TCPTun != "" {
			for _, tun := range strings.Split(flags.TCPTun, ",") {
				p := strings.Split(tun, "=")
				go tcpTun(p[0], addr, p[1], ciph.StreamConn)
			}
		}
```

从代码中我们可以看到两点：

1. 我们可以同时建设多条隧道；
2. 目的端的IP来自等号的左侧；

接着我们看tcp.go中的**tcpTun**函数：

```golang
func tcpTun(addr, server, target string, shadow func(net.Conn) net.Conn) {
	tgt := socks.ParseAddr(target)
	if tgt == nil {
		logf("invalid target address %q", target)
		return
	}
	logf("TCP tunnel %s <-> %s <-> %s", addr, server, target)
	tcpLocal(addr, server, shadow, func(net.Conn) (socks.Addr, error) { return tgt, nil })
}
```

我想一些对golang语法不是太熟悉的小伙伴们，估计对``shadow func(net.Conn) net.Conn``以及``func(net.Conn) (socks.Addr, error) { return tgt, nil }``这两个参数传递有点困惑。

**在golang中可以把函数作为一种类型，并且可以把函数作为参数进行传递**。

```golang
func(net.Conn) (socks.Addr, error) {
      return tgt, nil 
}
```

把代码重新调整一下格式，大家就不难看出其实这就是一个接收``net.Conn``作为参数返回将``tgt``作为返回值的函数。

在tcp隧道代理中目的端IP是通过``tgt := socks.ParseAddr(target)``格式化出来的。

我们梳理清楚了客户端如何得到目的端IP，接下来让我们看一下客户端是如何将IP写入数据流中。

写入数据流的操作是通过``rc.Write(tgt)``，它其实就是``conn.Write()``。说白了，客户端就是将目的IP 通过 ``conn.Write()``写入到数据流中。

#### 3.3.2 服务端如何解析出目的端IP呢？

``tgt, err := socks.ReadAddr(sc)``是服务端获取目的IP的入口。

进入``ReadAddr()``函数，可以看到它接受的是一个io.Reader类型的参数，并将它再传递给``readAddr()``函数进行具体的处理。

```golang
func ReadAddr(r io.Reader) (Addr, error) {
	return readAddr(r, make([]byte, MaxAddrLen))
}
```

接下来进入``readAdder()``函数，来看看具体的处理流程。

```golang
func readAddr(r io.Reader, b []byte) (Addr, error) {
	if len(b) < MaxAddrLen {
		return nil, io.ErrShortBuffer
	}
	_, err := io.ReadFull(r, b[:1]) // read 1st byte for address type
	if err != nil {
		return nil, err
	}

	switch b[0] {
	case AtypDomainName:
		_, err = io.ReadFull(r, b[1:2]) // read 2nd byte for domain length
		if err != nil {
			return nil, err
		}
		_, err = io.ReadFull(r, b[2:2+int(b[1])+2])
		return b[:1+1+int(b[1])+2], err
	case AtypIPv4:
		_, err = io.ReadFull(r, b[1:1+net.IPv4len+2])
		return b[:1+net.IPv4len+2], err
	case AtypIPv6:
		_, err = io.ReadFull(r, b[1:1+net.IPv6len+2])
		return b[:1+net.IPv6len+2], err
	}

	return nil, ErrAddressNotSupported
}
```

这里面使用主要使用了``io.ReadFull()``函数，这个函数可以把对象Reader中的数据读出来，然后存入一个缓冲区buf中，以便其他代码可以处理buf中的数据。

我们可以通过这段代码看出，我们可以通过buf中的第一位判断IP地址的类型，然后根据不同类型的IP来截取buf中对应长度的内容就可以获得IP地址了。

### 3.4 数据流的转发是如何实现的
我们来看一下tcp.go文件中的``relay()``函数。
```golang
func relay(left, right net.Conn) error {
	var err, err1 error
	var wg sync.WaitGroup
	var wait = 5 * time.Second
	wg.Add(1)
	go func() {
		defer wg.Done()
		_, err1 = io.Copy(right, left)
		right.SetReadDeadline(time.Now().Add(wait)) // unblock read on right
	}()
	_, err = io.Copy(left, right)
	left.SetReadDeadline(time.Now().Add(wait)) // unblock read on left
	wg.Wait()
	if err1 != nil && !errors.Is(err1, os.ErrDeadlineExceeded) { // requires Go 1.15+
		return err1
	}
	if err != nil && !errors.Is(err, os.ErrDeadlineExceeded) {
		return err
	}
	return nil
}
```
本教程中最关键的代码而就是``io.Copy()``函数，这就是代理的核心，代理的本质，是转发两个相同方向路径上的stream(数据流)。
再看一下``io.Copy(）``的源码：
```golang
func Copy(dst Writer, src Reader) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	buf := make([]byte, 32*1024)
	for {
		nr, er := src.Read(buf)
		if nr > 0 {
			nw, ew := dst.Write(buf[0:nr])
			if nw > 0 {
				written += int64(nw)
			}
			if ew != nil {
				err = ew
				break
			}
			if nr != nw {
				err = ErrShortWrite
				break
			}
		}
		if er == EOF {
			break
		}
		if er != nil {
			err = er
			break
		}
	}
	return written, err
```
阻塞式的从src输入流读数据到dst输出流。由于``io.Copy()``是阻塞式的复制，所以我们需要使用到协程来提高程序的运行效率。
## 4. 我们可以学习到哪些技术点？
最后，我总结一下通过解析shadowsockets源码学习到的技术点有以下3点：
1. 可以通过将目的端IP写入数据流中的方式让服务端知道目的IP；
2. 在golang中可以通过io.Copy()函数实现流量的转发
3. 遇到阻塞式函数的时候为了避免程序阻塞可以使用goroutine的方式讲阻塞函数单独放到一个协程中；
4. ``sync.WaitGroup``函数的作用是Wait()可以阻塞到队列中的所有任务都执行完才解除阻塞

---

欢迎关注我的公众号，不定期分析技术好文。

![](https://raw.githubusercontent.com/w910820618/shadowsocks/master/images/qrcode_for_gh_4afc5ec351d9_430.jpg)

