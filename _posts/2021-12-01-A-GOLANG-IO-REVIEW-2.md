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

##  0x02    再看io包
本小节再梳理下在golang中，src 拷贝到dst的各种方法。假设 dst 实现了`Write` 方法，src 实现了 `Read` 方法；另外只要dst、src实现了方法就可以（如`os.File`、`net.Conn`等）；通俗点说：
-   dst是`Writer`接口的实例：只要能实现`Writer`接口功能，就可以做dst
-   src是`Reader`接口的实例：只要能实现`Reader`接口功能，就可以做src

####    方法1：io.Copy(dst Writer, src Reader)
[官方](https://cs.opensource.google/go/go/+/refs/tags/go1.18.4:src/io/io.go;l=384)，不停的从 src 中读取数据，然后写入到 dst，直到遇到 `EOF`或者读取/写入发送错误时停止；不过需要注意的一点是读到`EOF`不会返回错误；常用于实现数据流转发
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

####   方法2：io.CopyBuffer(dst Writer, src Reader, buf []byte)
与`Copy`相同，不同点在于中间 buffer 可传入且 buffer 大小不能为`0`
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

####    方法3：io.CopyN(dst Writer, src Reader, n int64)
从 src 中拷贝 `n` bytes的和数据到 dst，注意，只copy `n`长度的数据，完成即退出，技巧是使用`io.LimitReader`：
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

从笔者测试来看，`CopyN`并不能给`Copy`增加限流的功能（如SSH代理），其设置的总数`n`，在读取时会不断的被更新（直到`0`），此时，就直接退出了。

####    为何退出？
看下`LimitReader`提供的`Read`[方法](https://cs.opensource.google/go/go/+/refs/tags/go1.18.4:src/io/io.go;drc=2d655fb15a50036547a6bf8f77608aab9fb31368;bpv=1;bpt=1;l=465)，每次`Read`结束都会更新
`l.N -= int64(n)`，当`l.N<=0`时就返回`EOF`了

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
	l.N -= int64(n)     //读取时会不停的更新N的值，直到0
	return
}
```

##  0x03    io.Copy如何实现限流
[SSH代理](https://pandaychen.github.io/2020/01/01/MAGIC-GO-IO-PACKAGE/#%E4%BD%BF%E7%94%A8-iocopy-%E5%AE%9E%E7%8E%B0-ssh-%E4%BB%A3%E7%90%86)的实现中，如何对双向连接复制流进行流控呢？

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

##  0x04    参考
-   [为什么要避免在 Go 中使用 ioutil.ReadAll？](https://developer.aliyun.com/article/893062)
-   [golang-io](https://pkg.go.dev/io)