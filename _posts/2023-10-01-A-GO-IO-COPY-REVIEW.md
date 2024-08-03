---
layout:     post
title:      再看 io.Copy
subtitle:   梳理一下容易被遗漏的细节问题
date:       2023-10-01
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
-   [关于 tcp 流量的阻塞问题 #100](https://github.com/eycorsican/go-tun2socks/issues/100)

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

考虑这样的情况，在传入参数有任意一个是 `*net.TCPConn` (当然 `rightConn` 不可能是 TCP 毕竟有 TcpTracker），该问题最终会导致传入 `CopyBuffer` 的 `sync.Pool` 内存池失效

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

调用到标准库的 `io.CopyBuffer`，调用链如下（主要是前面的断言判断影响到内存池的作用）：

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
	// 如果 源Reader 实现了 WriterTo 接口,直接调用该方法 将数据写入到 目标Writer 当中
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	// 同理，如果 目标Writer 实现了 ReaderFrom 接口,直接调用ReadFrom方法
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

再看一个SSH端口转发（同样内存池会失效）的例子，在下面的`handleClient`方法中，`client` 的具体类型是 `*net.TCPConn`，而 `remote` 的具体类型是 `ssh.Channel`：

```go
// Get default location of a private key
func privateKeyPath() string {
  return os.Getenv("HOME") + "/.ssh/id_rsa"
}

// Get private key for ssh authentication
func parsePrivateKey(keyPath string) (ssh.Signer, error) {
  buff, _ := ioutil.ReadFile(keyPath)
  return ssh.ParsePrivateKey(buff)
}

// Get ssh client config for our connection
// SSH config will use 2 authentication strategies: by key and by password
func makeSshConfig(user, password string) (*ssh.ClientConfig, error) {
  key, err := parsePrivateKey(privateKeyPath())
  if err != nil {
    return nil, err
  }

  config := ssh.ClientConfig{
    User: user,
    Auth: []ssh.AuthMethod{
      ssh.PublicKeys(key),
      ssh.Password(password),
    },
  }

  return &config, nil
}

// Handle local client connections and tunnel data to the remote serverq
// Will use io.Copy - http://golang.org/pkg/io/#Copy
func handleClient(client net.Conn, remote net.Conn) {
  defer client.Close()
  chDone := make(chan bool)

  // Start remote -> local data transfer
  go func() {
    _, err := io.Copy(client, remote)
    if err != nil {
      log.Println("error while copy remote->local:", err)
    }
    chDone <- true
  }()

  // Start local -> remote data transfer
  go func() {
    _, err := io.Copy(remote, client)
    if err != nil {
      log.Println(err)
    }
    chDone <- true
  }()

  <-chDone
}

func main() {
  // Connection settings
  sshAddr := "remote_ip:22"
  localAddr := "127.0.0.1:5000"
  remoteAddr := "127.0.0.1:5432"

  // Build SSH client configuration
  cfg, err := makeSshConfig("user", "password")
  if err != nil {
    log.Fatalln(err)
  }

  // Establish connection with SSH server
  conn, err := ssh.Dial("tcp", sshAddr, cfg)
  if err != nil {
    log.Fatalln(err)
  }
  defer conn.Close()

  // Establish connection with remote server
  remote, err := conn.Dial("tcp", remoteAddr)
  if err != nil {
    log.Fatalln(err)
  }

  // Start local server to forward traffic to remote connection
  local, err := net.Listen("tcp", localAddr)
  if err != nil {
    log.Fatalln(err)
  }
  defer local.Close()

  // Handle incoming connections
  for {
    client, err := local.Accept()
    if err != nil {
      log.Fatalln(err)
    }

    handleClient(client, remote)
  }
}
```

## 0x02 一些细节

####  net.Conn 关闭连接的场景

1、完全关闭读和写 <br>
当完成了数据的发送和接收，并且不再需要使用这个连接时；或者当发生错误，如无法恢复的读写错误时，可以调用 `Close()` 方法，它会关闭整个连接，包括读和写

2、单独关闭读或写 <br>

-  `net.TcpConn` 提供了 `CloseWrite` 方法，当已经完成数据发送（写入），但仍然需要接收对方的响应时。当本端希望通过关闭写入来通知对方你已经完成了数据发送，但仍然希望接收剩余的响应数据时，可以调用该方法通知对端
-  `CloseRead` 方法，当本端已经接收到所有需要的数据，但仍然需要向对方发送数据时；当本端希望通知对方不再接收数据，但仍然需要继续发送数据时，可以调用该方法

##  0x03 业界的实现
介绍几个典型的 pipe 转发实现：

####    clash 的实现
clash 的实现 [在此](https://github.com/Dreamacro/clash/blob/master/common/net/relay.go#L9-L24)：

```golang
type ReadOnlyReader struct {
	io.Reader
}

type WriteOnlyWriter struct {
	io.Writer
}
// Relay copies between left and right bidirectionally.
func Relay(leftConn, rightConn net.Conn) {
	ch := make(chan error)

	go func() {
		// Wrapping to avoid using *net.TCPConn.(ReadFrom)
		// See also https://github.com/Dreamacro/clash/pull/1209

                // 从 rightConn 读取，写入 leftConn
		_, err := io.Copy(WriteOnlyWriter{Writer: leftConn}, ReadOnlyReader{Reader: rightConn})
		leftConn.SetReadDeadline(time.Now())
		ch <- err
	}()

	io.Copy(WriteOnlyWriter{Writer: rightConn}, ReadOnlyReader{Reader: leftConn})
	rightConn.SetReadDeadline(time.Now())
	<-ch
}
```

注意，上面第一个 `io.Copy` 退出的时候，说明这段逻辑已经退出了，无法向 `leftConn` 继续写入了，这里合理的设置 `leftConn.SetReadDeadline(time.Now())` ，该方法设置了 `leftConn` 的读取截止时间为当前时间，这意味着在当前时间之后，`leftConn` 的任何读取操作都将立即返回错误。这里设置读取截止时间的目的是在 `io.Copy(WriteOnlyWriter{Writer: rightConn}, ReadOnlyReader{Reader: leftConn})` 完成后，不再继续读取 `leftConn` 的数据。这是因为 `Relay` 方法目的是在 `leftConn` 和 `rightConn` 之间双向复制数据，当其中一个连接关闭或者出现错误时，函数会返回；反之 `rightConn.SetReadDeadline(time.Now())` 的操作也是如此


####    go-tun2socks 的实现
[go-tun2socks](https://github.com/eycorsican/go-tun2socks/blob/master/proxy/socks/tcp.go) 的实现如下：
```golang
type duplexConn interface {
	net.Conn
	CloseRead() error       // 单独关闭读取
	CloseWrite() error      // 单独关闭写入
}

func (h *tcpHandler) relay(lhs, rhs net.Conn) {
	upCh := make(chan struct{})

	cls := func(dir direction, interrupt bool) {
		lhsDConn, lhsOk := lhs.(duplexConn)
		rhsDConn, rhsOk := rhs.(duplexConn)
		if !interrupt && lhsOk && rhsOk {
			switch dir {
			case dirUplink:
				lhsDConn.CloseRead()
				rhsDConn.CloseWrite()
			case dirDownlink:
				lhsDConn.CloseWrite()
				rhsDConn.CloseRead()
			default:
				panic("unexpected direction")
			}
		} else {
			lhs.Close()
			rhs.Close()
		}
	}

	// Uplink
	go func() {
		var err error
		_, err = io.Copy(rhs, lhs)
		if err != nil {
			cls(dirUplink, true) // interrupt the conn if the error is not nil (not EOF)
		} else {
			cls(dirUplink, false) // half close uplink direction of the TCP conn if possible
		}
		upCh <- struct{}{}
	}()

	// Downlink
	var err error
	_, err = io.Copy(lhs, rhs)
	if err != nil {
		cls(dirDownlink, true)
	} else {
		cls(dirDownlink, false)
	}

	<-upCh // Wait for uplink done.
}
```

####   cloudflare/cloudflared 的实现
[cloudflared](https://github.com/cloudflare/cloudflared/blob/be64362fdb2a2da481f8e0414f75de3db2ccdf32/stream/stream.go#L43)

```golang
// Pipe copies copy data to & from provided io.ReadWriters.
func Pipe(tunnelConn, originConn io.ReadWriter, log *zerolog.Logger) {
	status := newBiStreamStatus()

	go unidirectionalStream(tunnelConn, originConn, "origin->tunnel", status, log)
	go unidirectionalStream(originConn, tunnelConn, "tunnel->origin", status, log)

	// If one side is done, we are done.
	status.waitAnyDone()
}

func unidirectionalStream(dst io.Writer, src io.Reader, dir string, status *bidirectionalStreamStatus, log *zerolog.Logger) {
	defer func() {
		// The bidirectional streaming spawns 2 goroutines to stream each direction.
		// If any ends, the callstack returns, meaning the Tunnel request/stream (depending on http2 vs quic) will
		// close. In such case, if the other direction did not stop (due to application level stopping, e.g., if a
		// server/origin listens forever until closure), it may read/write from the underlying ReadWriter (backed by
		// the Edge<->cloudflared transport) in an unexpected state.
		// Because of this, we set this recover() logic.
		if r := recover(); r != nil {
			if status.isAnyDone() {
				// We handle such unexpected errors only when we detect that one side of the streaming is done.
				log.Debug().Msgf("Gracefully handled error %v in Streaming for %s, error %s", r, dir, debug.Stack())
			} else {
				// Otherwise, this is unexpected, but we prevent the program from crashing anyway.
				log.Warn().Msgf("Gracefully handled unexpected error %v in Streaming for %s, error %s", r, dir, debug.Stack())

				tags := make(map[string]string)
				tags["root"] = "websocket.stream"
				tags["dir"] = dir
				switch rval := r.(type) {
				case error:
					raven.CaptureError(rval, tags)
				default:
					rvalStr := fmt.Sprint(rval)
					raven.CaptureMessage(rvalStr, tags)
				}
			}
		}
	}()

	_, err := copyData(dst, src, dir)
	if err != nil {
		log.Debug().Msgf("%s copy: %v", dir, err)
	}
	status.markUniStreamDone()
}

func copyData(dst io.Writer, src io.Reader, dir string) (written int64, err error) {
	if debugCopy {
		// copyBuffer is based on stdio Copy implementation but shows copied data
		copyBuffer := func(dst io.Writer, src io.Reader, dir string) (written int64, err error) {
			var buf []byte
			size := 32 * 1024
			buf = make([]byte, size)
			for {
				t := time.Now()
				nr, er := src.Read(buf)
				if nr > 0 {
					fmt.Println(dir, t.UnixNano(), "\n"+hex.Dump(buf[0:nr]))
					nw, ew := dst.Write(buf[0:nr])
					if nw < 0 || nr < nw {
						nw = 0
						if ew == nil {
							ew = errors.New("invalid write")
						}
					}
					written += int64(nw)
					if ew != nil {
						err = ew
						break
					}
					if nr != nw {
						err = io.ErrShortWrite
						break
					}
				}
				if er != nil {
					if er != io.EOF {
						err = er
					}
					break
				}
			}
			return written, err
		}
		return copyBuffer(dst, src, dir)
	} else {
		return cfio.Copy(dst, src)
	}
}

const defaultBufferSize = 16 * 1024

var bufferPool = sync.Pool{
	New: func() interface{} {
		return make([]byte, defaultBufferSize)
	},
}

// cfio.Copy 实现
func Copy(dst io.Writer, src io.Reader) (written int64, err error) {
	_, okWriteTo := src.(io.WriterTo)
	_, okReadFrom := dst.(io.ReaderFrom)
	var buffer []byte = nil

	if !(okWriteTo || okReadFrom) {
		buffer = bufferPool.Get().([]byte)
		defer bufferPool.Put(buffer)
	}

	return io.CopyBuffer(dst, src, buffer)
}
```



##  0x04    透明代理的场景
先描述下笔者项目的场景，如下：

![arch-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/netstack/netstack-1.png)

-  `LEFT`：左侧是一个 gvisor 构建的 [TCPConn](https://github.com/google/gvisor/blob/master/pkg/tcpip/adapters/gonet/gonet.go#L230)，实现了 `net.Conn`[接口](https://pkg.go.dev/net#Conn)
-  `RIGHT`：右侧是一个 `net.Conn`（普通代理）



##  0x0 参考
- [Why copyBuffer implements while loop](https://stackoverflow.com/questions/59014085/why-copybuffer-implements-while-loop)
- [Fix: tcp relay #219](https://github.com/xjasonlyu/tun2socks/pull/219)
- [TCP Half-Close: a cool feature that is now broken](https://www.excentis.com/blog/tcp-half-close-a-cool-feature-that-is-now-broken/)
- [SSH port forwarding with Go](https://sosedoff.com/2015/05/25/ssh-port-forwarding-with-go.html)
- [关于 tcp 流量的阻塞问题 #100](https://github.com/eycorsican/go-tun2socks/issues/100)