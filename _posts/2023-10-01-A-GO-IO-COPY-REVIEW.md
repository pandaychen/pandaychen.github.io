---
layout:     post
title:      再看 io.Copy
subtitle:   梳理一下容易被遗漏的细节问题
date:       2023-19-01
author:     pandaychen
header-img:
catalog: true
tags:
    - Golang
---

##  0x00    前言
前文 [神奇的 Golang-IO 包](https://pandaychen.github.io/2020/01/01/MAGIC-GO-IO-PACKAGE/) 描述过 `io.Copy` 系列方法的一般使用，最近在做流量代理项目中，遇到几个不错的 issue：

-   [Behavior of tun2socks with closed TCP ports #222](https://github.com/xjasonlyu/tun2socks/discussions/222)
-   [Fix: tcp relay](https://github.com/xjasonlyu/tun2socks/pull/219)
-   [[Bug] TCP relay is incorrect](https://github.com/xjasonlyu/tun2socks/issues/218)
-   [Fix: wrap net.Conn to avoid using *net.TCPConn.(ReadFrom) #1209](https://github.com/Dreamacro/clash/pull/1209)

上面的 issue 主要讨论在两个双向 conn 中，如何正确的处理关闭（读 / 写 / 全部）的问题，这里的 conn 可能是 `tcp.Conn`、`net.Conn` 等

##  0x01    问题引入

####    问题一

```GO
func main() {
        listener1, listener2 := listeners()
        go server2(listener2)
        var wg sync.WaitGroup
        wg.Add(1)
        go func() {
                server1(listener1, listener2)
                wg.Done()
        }()
        conn, err := net.DialTCP("tcp", nil, listener1.Addr().(*net.TCPAddr))
        if err != nil {
                panic(err)
        }
        conn.Write(data)
        wg.Wait()
}

func server1(listener1, listener2 *net.TCPListener) {
        conn1, err := listener1.AcceptTCP()
        if err != nil {
                panic(err)
        }
        conn2, err := net.DialTCP("tcp", nil, listener2.Addr().(*net.TCPAddr))
        if err != nil {
                panic(err)
        }
        var wg sync.WaitGroup
        wg.Add(1)
        go func() {
                io.Copy(conn1, conn2)
                wg.Done()
        }()
        wg.Add(1)
        go func() {
                io.Copy(conn2, conn1)
                wg.Done()
        }()
        wg.Wait() // FIXME blocked here
        conn1.Close()
        conn2.Close()
}

func server2(listener2 *net.TCPListener) {
        conn, err := listener2.AcceptTCP()
        if err != nil {
                panic(err)
        }
        buf := make([]byte, len(data))
        conn.Read(buf)
        fmt.Println(buf) // NOTE here out the []byte{1, 2, 3, 4}
        if bytes.Compare(buf, data) != 0 {
                panic("Data error")
        }
        conn.Write(data)
        conn.Close()
}

func listeners() (*net.TCPListener, *net.TCPListener) {
        tcpAddr1, err := net.ResolveTCPAddr("tcp", "127.0.0.1:8810")
        if err != nil {
                panic(err)
        }
        tcpAddr2, err := net.ResolveTCPAddr("tcp", "127.0.0.1:8899")
        if err != nil {
                panic(err)
        }
        listener1, err := net.ListenTCP("tcp", tcpAddr1)
        if err != nil {
                panic(err)
        }
        listener2, err := net.ListenTCP("tcp", tcpAddr2)
        if err != nil {
                panic(err)
        }
        return listener1, listener2
}
```

上面这段代码输出后卡住？原因是因为 `conn2` 不会被关闭，`io.Copy` 不会接收到 `io.EOF` 且不会返回。解决的方法是在 goroutine 中关闭对方连接，以便解除阻塞另一个 `io.Copy`，如下：

```go
wg.Add(1)
go func() {
    io.Copy(conn1, conn2)
    // conn2 has returned EOF or an error, so we need to shut down the
    // other half of the duplex copy.
    conn1.Close()
    wg.Done()
}()

wg.Add(1)
go func() {
    io.Copy(conn2, conn1)
    conn2.Close()
    wg.Done()
}()
```

####    问题二

再回顾下 frp 的这段双向流复制的代码，两个思考题：

1.  `to`/`from` 被 `Close()` 了两次，是否有问题？
2.  若 `to`/`from` 出现了速率不一致的（差距较大），直接 `Close()` 是否有影响？

```golang
// Join two io.ReadWriteCloser and do some operations.
func Join(c1 io.ReadWriteCloser, c2 io.ReadWriteCloser) (inCount int64, outCount int64) {
	var wait sync.WaitGroup
	pipe := func(to io.ReadWriteCloser, from io.ReadWriteCloser, count *int64) {
		defer to.Close()
		defer from.Close()
		defer wait.Done()

		buf := pool.GetBuf(16 * 1024)
		defer pool.PutBuf(buf)
		*count, _ = io.CopyBuffer(to, from, buf)
	}

	wait.Add(2)
	go pipe(c1, c2, &inCount)
	go pipe(c2, c1, &outCount)
	wait.Wait()
	return
}
```

1.  关闭一个已经关闭的 `io.Closer`（如文件、网络连接等）是安全的，因为它们通常会忽略重复关闭的操作

####    问题三
来源于 issue：[Fix: wrap net.Conn to avoid using *net.TCPConn.(ReadFrom) ](https://github.com/Dreamacro/clash/pull/1209)，重点摘要如下：

考虑这样的情况，在传入参数有任意一个是 *net.TCPConn (当然 rightConn 不可能是 TCP 毕竟有 TcpTracker）

```GO
// relay copies between left and right bidirectionally.
func relay(leftConn, rightConn net.Conn) {
 	ch := make(chan error)

 	go func() {
 		buf := pool.Get(pool.RelayBufferSize)
 		_, err := io.CopyBuffer(leftConn, rightConn, buf)
 		pool.Put(buf)
 		leftConn.SetReadDeadline(time.Now())
 		ch <- err
 	}()

 	buf := pool.Get(pool.RelayBufferSize)
 	io.CopyBuffer(rightConn, leftConn, buf)
 	pool.Put(buf)
 	rightConn.SetReadDeadline(time.Now())
 	<-ch
}
```

调用到 `CopyBuffer`，调用链如下：

```go
func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	if buf != nil && len(buf) == 0 {
		panic("empty buffer in CopyBuffer")
	}
	return copyBuffer(dst, src, buf)
}

func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src) // 到此为止
	}

        // Copy
    //......
}
```

由于 `*net.TCPConn` 实现了 `ReadFrom` 函数会直接交给 `*net.TCPConn.(ReadFrom)` 处理：

```GO
// ReadFrom implements the io.ReaderFrom ReadFrom method.
func (c *TCPConn) ReadFrom(r io.Reader) (int64, error) {
	if !c.ok() {
		return 0, syscall.EINVAL
	}
	n, err := c.readFrom(r) // 实现
	if err != nil && err != io.EOF {
		err = &OpError{Op: "readfrom", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
	}
	return n, err
}

func (c *TCPConn) readFrom(r io.Reader) (int64, error) {
	if n, err, handled := splice(c.fd, r); handled {
		return n, err
	}
	if n, err, handled := sendFile(c.fd, r); handled {
		return n, err
	}
	return genericReadFrom(c, r)
}

// Fallback implementation of io.ReaderFrom's ReadFrom, when sendfile isn't
// applicable.
func genericReadFrom(w io.Writer, r io.Reader) (n int64, err error) {
	// Use wrapper to hide existing r.ReadFrom from io.Copy.
	return io.Copy(writerOnly{w}, r)
}
```

可以看到最终的 `readFrom` 实现方法：
1.  先尝试 Zero Copy - `splice`
2.  要求 Reader 是一个 `*net.TCPConn`
3.  再尝试 Zero Copy - `sendFile`
4.  要求 Reader 是一个 `*os.File`
5.  回退到常规复制 `genericReadFrom`，最终会使 `CopyBuffer` 使用的 来自 `pool` 的 `buffer` 失去意义


##  0x02    透明代理的场景
先描述下笔者项目的场景，如下：

![arch-flow]()


##  0x0 


##  0x0 解决



##  0x0 参考