---
layout:     post
title:      Golang IO copy 系列方法探究
subtitle:	golang-IO 包使用经验（二）
date:       2020-01-01
author:     pandaychen
header-img:
catalog: true
tags:
    - Golang
---

##  0x00    前言
前文 [神奇的 Golang-IO 包](https://pandaychen.github.io/2020/01/01/MAGIC-GO-IO-PACKAGE/)，介绍了 `io.Copy()`、`io.Pipe()` 等方法的使用，本文再对近期项目中笔者使用的 io 方法做一次深入总结。

##  0x01    再看文件复制
使用 golang 完成文件复制，通常有下面几种方法：

1、 使用 ioutils 库的 `ReadFile` 及 `WriteFile`，一次性将文件读取到内存，大文件场景不合适 <br>
```golang
func CopyFile(){
    //..
    input, err := ioutil.ReadFile(sourceFile)
    if err != nil {
            fmt.Println(err)
            return
    }

    err = ioutil.WriteFile(destinationFile, input, 0644)
    if err != nil {
            fmt.Println("Error creating", destinationFile)
            fmt.Println(err)
            return
    }
    //...
}
```

2、自己实现循环读写，自定义 buffer 大小 <br>
```golang
func CopyFile(...) (error){
    //...
    buf := make([]byte, BUFFERSIZE)
    for {
            n, err := source.Read(buf)
            if err != nil && err != io.EOF {
                    return err
            }
            if n == 0 {
                    break
            }

            if _, err := destination.Write(buf[:n]); err != nil {
                    return err
            }
    }
}
```

3、使用 io 库的 `io.Copy()` 或者 `io.CopyBuffer()`，后者支持设置 BufferSize，这个是比较推荐的方法 <br>
```golang
func CopyFile(src, dst string) (int64, error) {
        source, err := os.Open(src)
        if err != nil {
                return 0, err
        }
        defer source.Close()

        destination, err := os.Create(dst)
        if err != nil {
                return 0, err
        }
        defer destination.Close()
        // 注意，这里 destination 是 os.File 类型
        nBytes, err := io.Copy(destination, source)
        return nBytes, err
}
```

####    文件复制场景使用 io.Copy 遇到的坑
注意到，在文件复制的场景下，`copyBuffer` 的参数 `dst Writer, src Reader` 都由 `os.File` 结构提供实现，当前系统的 golang 版本是 `go version go1.16.6 linux/amd64`，对应的 `io.Copy` 底层实现方法 `copyBuffer` 如下：

```golang
// copyBuffer is the actual implementation of Copy and CopyBuffer.
// if buf is nil, one is allocated.
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	if buf == nil {
        //BufferSize 默认 32K
		size := 32 * 1024
		if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
			if l.N < 1 {
				size = 1
			} else {
				size = int(l.N)
			}
		}
		buf = make([]byte, size)
	}
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
		if er != nil {
			if er != EOF {
				err = er
			}
			break
		}
	}
	return written, err
}
```

看下 `go version go1.16.6 linux/amd64` 版本，`file` 提供了 `ReadFrom` 的实现，而旧版本未提供，所以就导致新版本会直接进入下面这段逻辑（`ReadFrom` 下面是普通的实现，这里是特殊实现，也是 go 的一大特色）：

```golang
// Similarly, if the writer has a ReadFrom method, use it to do the copy.
if rt, ok := dst.(ReaderFrom); ok {
    return rt.ReadFrom(src)
}
```

笔者特意找了两台不同版本的机器，确认了下上述问题的确存在：

在 `1.15` 版本以前 `os.File` 没有实现 `WriterTo` 和 `ReaderFrom` 方法，但在 `1.15` 版本中，由于 `os.File` 实现了 `ReaderFrom` 接口，导致进入了这一段逻辑，** 同时相关的 buffer 设置直接失效了 **。
```bash
[root@VM_120_245_centos /usr/local/go]#  grep ReadFrom * -rn|grep file|grep func
src/os/file.go:148:func (f *File) ReadFrom(r io.Reader) (n int64, err error) {
src/os/file.go:159:func genericReadFrom(f *File, r io.Reader) (int64, error) {
```

对比下 `go version go1.11 linux/amd64` 版本的代码：
```bash
[root@VM-153-178-tlinux /usr/local/go]# grep ReadFrom * -rn|grep file|grep func
```

####    跟踪 file 的 ReadFrom 方法
```golang
// ReadFrom implements io.ReaderFrom.
func (f *File) ReadFrom(r io.Reader) (n int64, err error) {
        if err := f.checkValid("write"); err != nil {
                return 0, err
        }
        // 依赖 File readFrom 方法的返回 handled
        n, handled, e := f.readFrom(r)
        if !handled {
                return genericReadFrom(f, r) // without wrapping
        }
        return n, f.wrapErr("write", e)
}

func genericReadFrom(f *File, r io.Reader) (int64, error) {
        return io.Copy(onlyWriter{f}, r)
}

type onlyWriter struct {
        io.Writer
}
```

注意上面那个 `readFrom` 方法，在 `readfrom_stub.go` 文件中发现，对于非 linux 系统，这里的返回是 `false`，就直接退化为 `io.Copy(onlyWriter{f}, r)` 方法了
```golang
// +build !linux

func (f *File) readFrom(r io.Reader) (n int64, handled bool, err error) {
        return 0, false, nil
}
```

而对于 Linux 系统，`readFrom` 方法实现如下：
```golang
var pollCopyFileRange = poll.CopyFileRange

func (f *File) readFrom(r io.Reader) (written int64, handled bool, err error) {
        // copy_file_range(2) does not support destinations opened with
        // O_APPEND, so don't even try.
        if f.appendMode {
                return 0, false, nil
        }

        remain := int64(1 << 62)

        lr, ok := r.(*io.LimitedReader)
        if ok {
                remain, r = lr.N, lr.R
                if remain <= 0 {
                        return 0, true, nil
                }
        }

        src, ok := r.(*File)
        if !ok {
                return 0, false, nil
        }
        if src.checkValid("ReadFrom") != nil {
                // Avoid returning the error as we report handled as false,
                // leave further error handling as the responsibility of the caller.
                return 0, false, nil
        }

        written, handled, err = pollCopyFileRange(&f.pfd, &src.pfd, remain)
        if lr != nil {
                lr.N -= written
        }
        return written, handled, NewSyscallError("copy_file_range", err)
}
```
从上面这段代码可知，底层调用的是 `pollCopyFileRange` 方法，最终调用了是 linux 内核（`4.5` 版本）的 `copy_file_range`[函数](https://man7.org/linux/man-pages/man2/copy_file_range.2.html) 完成，猜想版本肯定对 copy 做了优化。主要基于如下几点：
-   内核态完成 copy，避免数据从内核态 copy 到用户态，再 copy 到内核态
-   NFS 场景下的 `server-side-copy` 优化

####    如何规避这个问题呢？
通过强行转换 `io.CopyBuffer` 的第一个参数，避免使用 `os.File` 类型传入，见此 [代码](https://github.com/pandaychen/golang_in_action/blob/master/io/copy_without_optimise.go)。不过该方式放弃了 `pollCopyFileRange` 的内核 copy 优化，具体场景下可能还需要进一步衡量得失：
-   支持自定义支持 bufferSize
-   在 nfs 没有开启 server side copy 特性的场景，使用 `copy_file_range` 性能偏差

##  0x02    再看 io 包
本小节再梳理下在 golang 中，src 拷贝到 dst 的各种方法。假设 dst 实现了 `Write` 方法，src 实现了 `Read` 方法；另外只要 dst、src 实现了方法就可以（如 `os.File`、`net.Conn` 等）；通俗点说：
-   dst 是 `Writer` 接口的实例：只要能实现 `Writer` 接口功能，就可以做 dst
-   src 是 `Reader` 接口的实例：只要能实现 `Reader` 接口功能，就可以做 src

####    方法 1：io.Copy(dst Writer, src Reader)
[官方](https://cs.opensource.google/go/go/+/refs/tags/go1.18.4:src/io/io.go;l=384)，不停的从 src 中读取数据，然后写入到 dst，直到遇到 `EOF` 或者读取 / 写入发送错误时停止；不过需要注意的一点是读到 `EOF` 不会返回错误；常用于实现数据流转发
```golang
// Copy copies from src to dst until either EOF is reached
// on src or an error occurs. It returns the number of bytes
// copied and the first error encountered while copying, if any.
//
// A successful Copy returns err == nil, not err == EOF.
// Because Copy is defined to read from src until EOF, it does
// not treat an EOF from Read as an error to be reported.
//
// If src implements the WriterTo interface,
// the copy is implemented by calling src.WriteTo(dst).
// Otherwise, if dst implements the ReaderFrom interface,
// the copy is implemented by calling dst.ReadFrom(src).
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}
```

####   方法 2：io.CopyBuffer(dst Writer, src Reader, buf []byte)
与 `Copy` 相同，不同点在于中间 buffer 可传入且 buffer 大小不能为 `0`
```golang
// CopyBuffer is identical to Copy except that it stages through the
// provided buffer (if one is required) rather than allocating a
// temporary one. If buf is nil, one is allocated; otherwise if it has
// zero length, CopyBuffer panics.
//
// If either src implements WriterTo or dst implements ReaderFrom,
// buf will not be used to perform the copy.
func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	if buf != nil && len(buf) == 0 {
		panic("empty buffer in CopyBuffer")
	}
	return copyBuffer(dst, src, buf)
}
```

####    方法 3：io.CopyN(dst Writer, src Reader, n int64)
从 src 中拷贝 `n` bytes 的和数据到 dst，注意，只 copy `n` 长度的数据，完成即退出，技巧是使用 `io.LimitReader`：
```golang
// CopyN copies n bytes (or until an error) from src to dst.
// It returns the number of bytes copied and the earliest
// error encountered while copying.
// On return, written == n if and only if err == nil.
//
// If dst implements the ReaderFrom interface,
// the copy is implemented using it.
func CopyN(dst Writer, src Reader, n int64) (written int64, err error) {
	written, err = Copy(dst, LimitReader(src, n))
	if written == n {
		return n, nil
	}
	if written < n && err == nil {
		// src stopped early; must have been EOF.
		err = EOF
	}
	return
}
```

从笔者测试来看，`CopyN` 并不能给 `Copy` 增加限流的功能（如 SSH 代理），其设置的总数 `n`，在读取时会不断的被更新（直到 `0`），此时，就直接退出了。

####    为何退出？
看下 `LimitReader` 提供的 `Read`[方法](https://cs.opensource.google/go/go/+/refs/tags/go1.18.4:src/io/io.go;drc=2d655fb15a50036547a6bf8f77608aab9fb31368;bpv=1;bpt=1;l=465)，每次 `Read` 结束都会更新
`l.N -= int64(n)`，当 `l.N<=0` 时就返回 `EOF` 了

```golang
// A LimitedReader reads from R but limits the amount of
// data returned to just N bytes. Each call to Read
// updates N to reflect the new amount remaining.
// Read returns EOF when N <= 0 or when the underlying R returns EOF.
type LimitedReader struct {
	R Reader // underlying reader
	N int64  // max bytes remaining
}

func (l *LimitedReader) Read(p []byte) (n int, err error) {
	if l.N <= 0 {
		return 0, EOF
	}
	if int64(len(p)) > l.N {
		p = p[0:l.N]
	}
	n, err = l.R.Read(p)
	l.N -= int64(n)     // 读取时会不停的更新 N 的值，直到 0
	return
}
```

##  0x03    io.Copy 如何实现限流
[SSH 代理](https://pandaychen.github.io/2020/01/01/MAGIC-GO-IO-PACKAGE/#%E4%BD%BF%E7%94%A8-iocopy-%E5%AE%9E%E7%8E%B0-ssh-%E4%BB%A3%E7%90%86) 的实现中，如何对双向连接复制流进行流控呢？

```golang
copyConn := func(writer, reader net.Conn) {
    defer writer.Close()
    defer reader.Close()

    _, err := io.Copy(writer, reader)
    if err != nil {
        fmt.Printf("io.Copy error: %s", err)
    }
}

go copyConn(localConn, remoteConn)
go copyConn(remoteConn, localConn)
```

放在下一篇博客中介绍。

##      0x04    io.MultiReader 的用法
`io.MultiReader` 接收一系列实现了 `io.Reader` 接口的对象，并返回一个新的 `io.Reader` 对象。返回的 `io.Reader` 会依次从输入的 `io.Reader` 对象中读取数据，当一个读取完毕后，会继续从下一个读取。即 `io.MultiReader` 将多个 `io.Reader` 对象合并为一个，使它们看起来像一个连续的数据流

典型的使用场景是合并多个文件：需要将多个文件合并为一个时，可以使用 io.MultiReader 将这些文件的 io.Reader 合并，然后一次性读取所有文件的内容

```GO

func main() {
	// 打开文件
	file, err := os.Open("file.txt")
	if err != nil {
		fmt.Printf("Error opening file: %v\n", err)
		return
	}
	defer file.Close()

	// 创建一个包含 header 信息的 strings.Reader
	header := strings.NewReader("Header information\n")

	// 使用 io.MultiReader 将 header 和文件内容合并
	reader := io.MultiReader(header, file)

	// 使用 io.Copy 将合并后的内容输出到控制台（os.Stdout）
	if _, err := io.Copy(os.Stdout, reader); err != nil {
		fmt.Printf("Error copying data to stdout: %v\n", err)
	}
}
```

在下文中，还可以看到 `io.MultiReader` 的其他使用场景

##      0x05    实战：martian 项目的 io 用法汇总
在阅读 martian[项目](https://github.com/google/martian) 中，遇到若干非常典型的 io 用法，这里汇总下。

1、HTTPS-MITM 的核心实现

这里主要关注如下函数调用路径，关联 [文件](https://github.com/google/martian/blob/master/proxy.go)，即 `p.handleLoop`->`p.handle`->`p.handleConnectRequest`，且当 MITM 开启下的处理过程

这里 `handleLoop` 的主要功能是把 TCP 层连接 `net.Conn` 转换为 http 连接（包括长连接的场景），调用到 `brw := bufio.NewReadWriter(bufio.NewReader(conn), bufio.NewWriter(conn))` 方法，它是一个缓冲读写器，用于从 `conn` 连接中读取数据和向 `conn` 连接写入数据，可以方便地在同一个连接上进行缓冲的读取和写入操作

- `bufio.NewReader(conn)`：此函数创建了一个新的缓冲读取器（bufio.Reader），它将从 conn 连接中读取数据。缓冲读取器可以提高 I/O 效率，因为它会一次性从底层连接中读取较大块的数据，并将其存储在内部缓冲区中，以便以后按需读取
-  `bufio.NewWriter(conn)`: 此函数创建了一个新的缓冲写入器（bufio.Writer），它将向 conn 连接写入数据。缓冲写入器可以提高 I/O 效率，因为它会将要写入的数据先存储在内部缓冲区中，然后在缓冲区满时或调用 Flush() 函数时一次性写入底层连接


```GO
// 异步处理单个 mitm 连接
func (p *Proxy) handleLoop(conn net.Conn) {
	p.connsMu.Lock()
	p.conns.Add(1)
	p.connsMu.Unlock()
	defer p.conns.Done()
	defer conn.Close()
	if p.Closing() {
		// 被动关闭
		return
	}

	brw := bufio.NewReadWriter(bufio.NewReader(conn), bufio.NewWriter(conn))

	// 本质上 brw 是对 conn 的优化式包装

	// 构建新 session，同时将 conn 关联到这个 session
	s, err := newSession(conn, brw)
	if err != nil {
		log.Errorf("martian: failed to create session: %v", err)
		return
	}

	ctx, err := withSession(s)
	if err != nil {
		log.Errorf("martian: failed to create context: %v", err)
		return
	}

	/*
		这个 for 循环用于在代理服务器和客户端之间建立会话，并且持续监听和处理来自客户端的请求。如果代理服务器正在关闭，循环会退出并关闭连接。在每次循环中，会设置一个读取和写入操作的截止时间，以确保在超时的情况下不会一直等待。如果在处理请求时出现了可以关闭连接的错误，循环会退出并关闭连接。
	*/

	// 考虑长连接的 case，循环处理
	for {
		deadline := time.Now().Add(p.timeout)
		conn.SetDeadline(deadline)

		if err := p.handle(ctx, conn, brw); isCloseable(err) {
			log.Debugf("martian: closing connection: %v", conn.RemoteAddr())
			return
		}
	}
}
```

接着，在 `handle` 方法中，调用 `readRequest` 方法尝试以异步的方式读取一个 HTTP 请求（回想下 HTTP-CONNECT 代理第一个请求是明文的 `CONNECT` 请求），在 `readReqeust` 中，使用 `http.ReadRequest(brw.Reader)` 方法读取一个 HTTP 请求


```GO
// handle ：处理 HTTPS-mitm 代理逻辑
// conn 用于写数据
// brw：用于读数据
func (p *Proxy) handle(ctx *Context, conn net.Conn, brw *bufio.ReadWriter) error {
	log.Debugf("martian: waiting for request: %v", conn.RemoteAddr())

	// 读取请求（readRequest 的用法），这里是在
	req, err := p.readRequest(ctx, conn, brw)
	if err != nil {
		return err
	}
	defer req.Body.Close() //

	session := ctx.Session()
	ctx, err = withSession(session)
	if err != nil {
		log.Errorf("martian: failed to build new context: %v", err)
		return err
	}

	link(req, ctx)
	defer unlink(req)

	if tsconn, ok := conn.(*trafficshape.Conn); ok {
		wrconn := tsconn.GetWrappedConn()
		if sconn, ok := wrconn.(*tls.Conn); ok {
			session.MarkSecure()

			cs := sconn.ConnectionState()
			req.TLS = &cs
		}
	}

	if tconn, ok := conn.(*tls.Conn); ok {
		//mitm https      //MarkSecure	tls 会命中此处逻辑
		session.MarkSecure()

		cs := tconn.ConnectionState()
		req.TLS = &cs
	}
	fmt.Println("for handle..", conn.RemoteAddr(), req.URL.Scheme, req.URL.Host, req.Host, req.Method)
	/*
		for handle.. 9.x.x.x:45360  www.baidu.com:443 www.baidu.com:443 CONNECT
		for handle.. 9.x.x.x:45360   www.baidu.com GET	// 注意是同一个 tcp 连接完成的

		在 CONNECT https 场景中，p.handle 会被调用两次
	*/
	req.URL.Scheme = "http"
	if session.IsSecure() {
		log.Infof("martian: forcing HTTPS inside secure session")
		req.URL.Scheme = "https"
	}

	req.RemoteAddr = conn.RemoteAddr().String()
	if req.URL.Host == "" {
		req.URL.Host = req.Host
	}

	if req.Method == "CONNECT" {
		// phase 1：curl https://www.baidu.com
		return p.handleConnectRequest(ctx, req, session, brw, conn)
	}

	// phase 2：curl http://www.baidu.com
	// Not a CONNECT request
	if err := p.reqmod.ModifyRequest(req); err != nil {
		log.Errorf("martian: error modifying request: %v", err)
		proxyutil.Warning(req.Header, err)
	}
	if session.Hijacked() {
		return nil
	}

	// perform the HTTP roundtrip
	// 执行原始客户端 https 访问

	// 如果是 get 下载 / post 上传大文件场景，这里的性能如何？
	res, err := p.roundTrip(ctx, req)
	if err != nil {
		log.Errorf("martian: failed to round trip: %v", err)
		res = proxyutil.NewResponse(502, nil, req)
		proxyutil.Warning(res.Header, err)
	}
	defer res.Body.Close()

	// set request to original request manually, res.Request may be changed in transport.
	// see https://github.com/google/martian/issues/298
	res.Request = req

	if err := p.resmod.ModifyResponse(res); err != nil {
		log.Errorf("martian: error modifying response: %v", err)
		proxyutil.Warning(res.Header, err)
	}
	if session.Hijacked() {
		log.Infof("martian: connection hijacked by response modifier")
		return nil
	}

	var closing error
	if req.Close || res.Close || p.Closing() {
		log.Debugf("martian: received close request: %v", req.RemoteAddr)
		res.Close = true
		closing = errClose
	}

        // ......

	// 将数据发送回客户端
	// 这里其实不太喜欢 martian 的实现
	err = res.Write(brw)
	if err != nil {
		log.Errorf("martian: got error while writing response back to client: %v", err)
		if _, ok := err.(*trafficshape.ErrForceClose); ok {
			closing = errClose
		}
	}
	err = brw.Flush()
	if err != nil {
		log.Errorf("martian: got error while flushing response back to client: %v", err)
		if _, ok := err.(*trafficshape.ErrForceClose); ok {
			closing = errClose
		}
	}
	return closing
}

// readRequest：异步处理请求读取
func (p *Proxy) readRequest(ctx *Context, conn net.Conn, brw *bufio.ReadWriter) (*http.Request, error) {
	var req *http.Request
	reqc := make(chan *http.Request, 1)
	errc := make(chan error, 1)
	go func() {
		// 按照 http 协议读取 brw
		/*
			从一个 bufio.ReadWriter（brw）中读取并解析一个 HTTP 请求。
			http.ReadRequest 是 Go 语言中 net/http 包的一个函数，它接收一个实现了 io.Reader 接口的参数，用于从中读取数据。在这个例子中，参数是 brw.Reader，即 bufio.ReadWriter 的读取部分（brw 是一个指向 bufio.ReadWriter 类型的指针）。
			函数返回两个值：r 和 err。r 是一个指向 http.Request 类型的指针，表示成功解析的 HTTP 请求对象；err 是一个错误对象，用于表示在读取或解析请求过程中遇到的任何错误。如果没有错误发生，err 将为 nil，否则你需要检查和处理错误。
		*/
		r, err := http.ReadRequest(brw.Reader)
		if err != nil {
			errc <- err
			return
		}
		reqc <- r
	}()
	select {
	case err := <-errc:
		if isCloseable(err) {
			log.Debugf("martian: connection closed prematurely: %v", err)
		} else {
			log.Errorf("martian: failed to read request: %v", err)
		}

		// TODO: TCPConn.WriteClose() to avoid sending an RST to the client.

		return nil, errClose
	case req = <-reqc:
	case <-p.closing:
		return nil, errClose
	}

	return req, nil
}
```


最后再看下 `handleConnectRequest` 方法，这里有个关于 `io.MultiReader` 的用法，大致流程是：

1. 在开启 mitm 模式下，先从 `brw` 中读取 `1` 个字节（`CONECT` 请求后，如果是 https 协议，那么这里就是 tls 握手协议的第一个 clienthello 报文）

```GO
b := make([]byte, 1)
// 只读 1 个字节
if _, err := brw.Read(b); err != nil {
        log.Errorf("martian: error peeking message through CONNECT tunnel to determine type: %v", err)
}
```

2、保存 `br.Reader` 剩下的数据

```GO
// Drain all of the rest of the buffered data.
buf := make([]byte, brw.Reader.Buffered())
brw.Read(buf)
```

3、确认是 tls 协议处理，使用 `io.MultiReader(bytes.NewReader(b), bytes.NewReader(buf), conn)` 把这三个 `io.Reader` 按照顺序组装成一个，这是很有意思的用法，由于已经使用 `brw` 对 `conn` 读出了一些数据，这里的目的是把 `conn` 还原成先前的样子（第一个 clienthello 报文）

```GO
tlsconn := tls.Server(&peekedConn{conn, io.MultiReader(bytes.NewReader(b), bytes.NewReader(buf), conn)}, p.mitm.TLSForHost(req.Host) /*tls config*/)
```

4、在组装的 tlsconn 上进行 tls 握手
```GO
if err := tlsconn.Handshake(); err != nil {
        p.mitm.HandshakeErrorCallback(req, err)
        return err
}
```

5、握手完成，将 `brw` 重定向到 tlsconn 上进行读写

```GO
var nconn net.Conn
nconn = tlsconn
// If the original connection is a traffic shaped connection, wrap the tls
// connection inside a traffic shaped connection too.
if ptsconn, ok := conn.(*trafficshape.Conn); ok {
        nconn = ptsconn.Listener.GetTrafficShapedConn(tlsconn)
}

brw.Writer.Reset(nconn)
brw.Reader.Reset(nconn)

// 处理 http 请求（这里已经是明文了）
p.handle(ctx, nconn, brw)
```

```GO
// handleConnectRequest：处理 HTTP CONNECT 逻辑
func (p *Proxy) handleConnectRequest(ctx *Context, req *http.Request, session *Session, brw *bufio.ReadWriter, conn net.Conn) error {
	if err := p.reqmod.ModifyRequest(req); err != nil {
		log.Errorf("martian: error modifying CONNECT request: %v", err)
		proxyutil.Warning(req.Header, err)
	}
	if session.Hijacked() {
		log.Debugf("martian: connection hijacked by request modifier")
		return nil
	}

	if p.mitm != nil {
		// 开启 MITM 处理
		log.Debugf("martian: attempting MITM for connection: %s / %s", req.Host, req.URL.String())

		// 构造 200 OK 回复 response，tunnel 建立
		res := proxyutil.NewResponse(200, nil, req)

		if err := p.resmod.ModifyResponse(res); err != nil {
			log.Errorf("martian: error modifying CONNECT response: %v", err)
			proxyutil.Warning(res.Header, err)
		}
		if session.Hijacked() {
			log.Infof("martian: connection hijacked by response modifier")
			return nil
		}

		/*
			将一个 http.Response（res）对象写入到 bufio.ReadWriter（brw）中，并将缓冲区的数据刷新到底层的写入器。

			if err := res.Write(brw); err != nil { ... }：这一部分使用 http.Response 的 Write 方法将响应对象 res 序列化为 HTTP 响应格式并写入到 bufio.ReadWriter（brw）的缓冲区中。如果在写入过程中遇到任何错误，将记录错误日志。

			if err := brw.Flush(); err != nil { ...}：在将响应写入到 bufio.ReadWriter 的缓冲区之后，这一部分调用 Flush 方法将缓冲区的数据刷新到底层的写入器（通常是一个网络连接）。这样，HTTP 响应才会被发送回客户端。如果在刷新过程中遇到任何错误，将记录错误日志。

			总之，这段代码的主要作用是将 http.Response 对象序列化并发送回客户端，同时处理可能遇到的任何错误。
		*/
		if err := res.Write(brw); err != nil {
			log.Errorf("martian: got error while writing response back to client: %v", err)
		}
		if err := brw.Flush(); err != nil {
			log.Errorf("martian: got error while flushing response back to client: %v", err)
		}

		log.Debugf("martian: completed MITM for connection: %s", req.Host)

		// 按照正常的协议流程来走，等待接收客户端的 TLS（packet），如果第一个字节是 22（十六进制 16）

		b := make([]byte, 1)
		// 只读 1 个字节
		if _, err := brw.Read(b); err != nil {
			log.Errorf("martian: error peeking message through CONNECT tunnel to determine type: %v", err)
		}

		// Drain all of the rest of the buffered data.
		buf := make([]byte, brw.Reader.Buffered())
		brw.Read(buf)

		// 22 is the TLS handshake.
		// https://tools.ietf.org/html/rfc5246#section-6.2.1
		if b[0] == 22 {
			// Prepend the previously read data to be read again by
			// http.ReadRequest.

			//martain 用 CONNECT 中的域名 做代理

			// 构建 faketls 配置

			// 在指定的 net.Conn 上构建 tls conn
			// 读走 brw，写走 conn，brw 读取效率更高一些
			// 这里是将客户端的 TCPconn 升级为 tlsconn

			// 注意这里 MultiReader 的用法 io.MultiReader(bytes.NewReader(b), bytes.NewReader(buf), conn)
			// 是将 3 个 io.Reader 对象，按顺序合并成一个
			tlsconn := tls.Server(&peekedConn{conn, io.MultiReader(bytes.NewReader(b), bytes.NewReader(buf), conn)}, p.mitm.TLSForHost(req.Host) /*tls config*/)

			if err := tlsconn.Handshake(); err != nil {
				p.mitm.HandshakeErrorCallback(req, err)
				return err
			}

			// if H2 protocol
			if tlsconn.ConnectionState().NegotiatedProtocol == "h2" {
				// http2
				return p.mitm.H2Config().Proxy(p.closing, tlsconn, req.URL)
			}

			var nconn net.Conn
			nconn = tlsconn
			// If the original connection is a traffic shaped connection, wrap the tls
			// connection inside a traffic shaped connection too.
			if ptsconn, ok := conn.(*trafficshape.Conn); ok {
				nconn = ptsconn.Listener.GetTrafficShapedConn(tlsconn)
			}
			/*
				这段代码中的 Reset 是 bufio.Writer 和 bufio.Reader 的方法，它们分别用于重置 brw.Writer 和 brw.Reader 的底层连接为新的 net.Conn 对象（在这个例子中是 nconn，即 tlsconn）。
				brw.Writer.Reset(nconn)：这一行代码将 brw.Writer 的底层连接重置为新的 net.Conn 对象 nconn。这意味着后续的写入操作将直接发送到 nconn。
				brw.Reader.Reset(nconn)：这一行代码将 brw.Reader 的底层连接重置为新的 net.Conn 对象 nconn。这意味着后续的读取操作将直接从 nconn 读取数据。
				这段代码的目的是将 bufio.ReadWriter 的底层连接切换到新的 net.Conn 对象，通常用于在底层连接发生变化时（例如，从普通连接升级到 TLS 连接）更新缓冲读写器。这样，在后续的读写操作中，brw 会使用新的底层连接进行数据传输。
			*/
			brw.Writer.Reset(nconn)
			brw.Reader.Reset(nconn)
			return p.handle(ctx, nconn, brw)
		}

		// Prepend the previously read data to be read again by http.ReadRequest.
		brw.Reader.Reset(io.MultiReader(bytes.NewReader(b), bytes.NewReader(buf), conn))
		return p.handle(ctx, conn, brw)
	}

	// 未启用 MITM 的处理
	log.Debugf("martian: attempting to establish CONNECT tunnel: %s", req.URL.Host)
	res, cconn, cerr := p.connect(req)
	if cerr != nil {
		log.Errorf("martian: failed to CONNECT: %v", cerr)
		res = proxyutil.NewResponse(502, nil, req)
		proxyutil.Warning(res.Header, cerr)

		if err := p.resmod.ModifyResponse(res); err != nil {
			log.Errorf("martian: error modifying CONNECT response: %v", err)
			proxyutil.Warning(res.Header, err)
		}
		if session.Hijacked() {
			log.Infof("martian: connection hijacked by response modifier")
			return nil
		}

		if err := res.Write(brw); err != nil {
			log.Errorf("martian: got error while writing response back to client: %v", err)
		}
		err := brw.Flush()
		if err != nil {
			log.Errorf("martian: got error while flushing response back to client: %v", err)
		}
		return err
	}
	defer res.Body.Close()
	defer cconn.Close()

	if err := p.resmod.ModifyResponse(res); err != nil {
		log.Errorf("martian: error modifying CONNECT response: %v", err)
		proxyutil.Warning(res.Header, err)
	}
	if session.Hijacked() {
		log.Infof("martian: connection hijacked by response modifier")
		return nil
	}

	res.ContentLength = -1
	if err := res.Write(brw); err != nil {
		log.Errorf("martian: got error while writing response back to client: %v", err)
	}
	if err := brw.Flush(); err != nil {
		log.Errorf("martian: got error while flushing response back to client: %v", err)
	}

	cbw := bufio.NewWriter(cconn)
	cbr := bufio.NewReader(cconn)
	defer cbw.Flush()

	copySync := func(w io.Writer, r io.Reader, donec chan<- bool) {
		if _, err := io.Copy(w, r); err != nil && err != io.EOF {
			log.Errorf("martian: failed to copy CONNECT tunnel: %v", err)
		}

		log.Debugf("martian: CONNECT tunnel finished copying")
		donec <- true
	}

	donec := make(chan bool, 2)
	go copySync(cbw, brw, donec)
	go copySync(brw, cbr, donec)

	log.Debugf("martian: established CONNECT tunnel, proxying traffic")
	<-donec
	<-donec
	log.Debugf("martian: closed CONNECT tunnel")

	return errClose
}
```

##  0x06    参考
-   [为什么要避免在 Go 中使用 ioutil.ReadAll？](https://developer.aliyun.com/article/893062)
-   [golang-io](https://pkg.go.dev/io)