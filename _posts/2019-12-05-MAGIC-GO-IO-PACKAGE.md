---
layout:     post
title:      神奇的Golang-IO包（计划）
subtitle:   
date:       2020-01-01
author:     pandaychen
header-img: 
catalog: true
tags:
    - Golang
---

##	0x01	Golang的io包介绍
在 golang 中，标准库中的package设计的基本原则是职责内聚。通常开发者可以使用自定义Wrapper的方式来封装标准库package中的interface接口，亦或在此基础上扩展，添加自定义的功能。
但是有一点，返回值必须封装的方法保持一致。

##  0x02    神奇的io.Copy

###	io.Copy的好处
`io.Copy()`相较于`ioutil.ReadAll()`之类，最大的不同是它采用了一个定长的缓冲区做中转，复制过程中，内存的消耗量是较为固定的。如果你的代理的网络状况不佳，就会发现`io.Copy()`比`ioutil.ReadAll()`的好处了。

###	io.Copy与Tcp的开发结合

做过服务端开发的同学一定都写过代理 Proxy，代理的本质，是转发两个相同方向路径上的stream（数据流）。例如，一个`A-->B-->C`的代理模式，B作为代理，需要完成下面两件事情：
1.	读取从`A--->B`的数据，转发到`B--->C`
2.	读取从`C--->B`的数据，转发到`B--->A`

在golang中，只需要`io.Copy()`就能轻而易举的完成上面的事情，其[实现代码](https://golang.org/src/io/io.go?s=12796:12856#L353)如下所示：
```go
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

###	io.Copy与Http的开发结合
在Web开发中，如果我们要实现下载一个文件（并保存在本地），有哪些高效的方式呢？<br>
第一种方式：`http.Get()`+`ioutil.WriteFile()`，将下载内容直接写到文件中

```go
func DownloadFile() error {
    url :="http://xxx/somebigfile"
    resp ,err := http.Get(url)
    if err != nil {
        fmt.Fprint(os.Stderr ,"get url error" , err)
    }

    defer resp.Body.Close()

    data ,err := ioutil.ReadAll(resp.Body)
    if err != nil {
        panic(err)
    }

	return ioutil.WriteFile("/tmp/xxx_file", data, 0755)
}
```
But，第一种方式的问题在于，如果是大文件，会出现内存不足的问题，因为它是需要先把请求内容全部读取到内存中，然后再写入到文件中的。优化的方案就是使用`io.Copy()`，它是将源复制到目标，并且是按默认的缓冲区`32k`循环操作的，不会将内容一次性全写入内存中

第二种方式：使用`io.Copy()`
```go
func DownloadFile() {
    url :="http://xxx/somebigfile"
    resp ,err := http.Get(url)
    if err != nil {
        fmt.Fprint(os.Stderr ,"get url error" , err)
    }
    defer resp.Body.Close()    
    out, err := os.Create("/tmp/xxx_file")
    wt :=bufio.NewWriter(out)
   
    defer out.Close()
    
    n, err :=io.Copy(wt, resp.Body)
    if err != nil {
        panic(err)
    }
    wt.Flush()
}
```
同理，复制大文件也可以用 `io.copy()` 这个，防止产生内存溢出。


###	使用io.Copy实现SSH代理
利用`io.Copy()`实现SSH代理的代码如下，只需要2个`io.Copy()`就简单的将两条连接无缝衔接起来了，非常直观。
```go

type Endpoint struct {
	Host string
	Port int
}

func (endpoint *Endpoint) String() string {
	return fmt.Sprintf("%s:%d", endpoint.Host, endpoint.Port)
}

type SSHtunnel struct {
	Local  *Endpoint
	Server *Endpoint
	Remote *Endpoint

	Config *ssh.ClientConfig
}

func (tunnel *SSHtunnel) Start() error {
	listener, err := net.Listen("tcp", tunnel.Local.String())
	if err != nil {
		return err
	}
	defer listener.Close()

	for {
		conn, err := listener.Accept()
		if err != nil {
			return err
		}
		go tunnel.forward(conn)
	}
}

func (tunnel *SSHtunnel) forward(localConn net.Conn) {
	serverConn, err := ssh.Dial("tcp", tunnel.Server.String(), tunnel.Config)
	if err != nil {
		fmt.Printf("Server dial error: %s\n", err)
		return
	}

	remoteConn, err := serverConn.Dial("tcp", tunnel.Remote.String())
	if err != nil {
		fmt.Printf("Remote dial error: %s\n", err)
		return
	}

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
}

func main() {
	localEndpoint := &Endpoint{
		Host: "localhost",
		Port: 9000,
	}

	serverEndpoint := &Endpoint{
		Host: "some-real-ssh-listening-port",
		Port: 22,
	}

	remoteEndpoint := &Endpoint{
		Host: "www.baidu.com",
		Port: 80,
	}

	sshConfig := &ssh.ClientConfig{
		User: "root",
		Auth: []ssh.AuthMethod{
			ssh.Password("real-password"),
		},
		HostKeyCallback: func(hostname string, remote net.Addr, key ssh.PublicKey) error {
			// Always accept key.
			return nil
		},
	}

	tunnel := &SSHtunnel{
		Config: sshConfig,
		Local:  localEndpoint,
		Server: serverEndpoint,
		Remote: remoteEndpoint,
	}

	tunnel.Start()
}
```

##  0x03	golang  io.Pipe的妙用



##	0x04	一些IO优化的tips
1.	关于io.Copy的优化
在前面讨论过，`io.Copy`此方法，从src拷贝数据到dst，中间会借助字节数组作为buf。可以使用`io.CopyBuffer` 替代 `io.Copy`，并使用sync.Pool缓存buf, 以减少临时对象的申请和释放。
```GOLANG
io.CopyBuffer(dst Writer, src Reader, buf []byte)
```

##	0x05	参考
-	[Go编程技巧--io.Reader/Writer](https://segmentfault.com/a/1190000013358594)