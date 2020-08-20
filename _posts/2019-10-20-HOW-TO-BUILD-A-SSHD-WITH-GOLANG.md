---
layout:     post
title:      使用 Golang 实现 SSH 和 SSHD
subtitle:   golang-ssh 库使用（入门篇）
date:       2019-10-20
author:     pandaychen
catalog:    true
tags:
    - OpenSSH
---

##  0x00    前言
&emsp;&emsp; [golang 的 SSH 包](https://godoc.org/golang.org/x/crypto/ssh) 提供了极为丰富的接口。基于此包可以很容易的实现 SSH 客户端、SSHD 服务端以及 SSH 代理等常用工具。
此包给了开发者极大针对 SSH 体系的扩展能力，可以实现 SSH 客户端及服务端的多种工具及安全实践。

通过下面两篇文章可以简单入门：
-   [Writing a replacement to OpenSSH using Go (1/2)](https://scalingo.com/blog/writing-a-replacement-to-openssh-using-go-12.html)
-   [Writing a replacement to OpenSSH using Go (2/2)](https://scalingo.com/blog/writing-a-replacement-to-openssh-using-go-22.html)

笔者基于此包实现过如下的 SSH 工程：
1.  SSH 运维客户端
2.  SSH-Proxy 代理（基于中间人会话劫持 `+` 转发控制与审计）
3.  SSH honeypot

##  0x01    SSH 架构
SSH 的基础架构图如下，进行项目开发之前，首先需要对 SSH 协议（分层）有个直观的认识：
![image](https://s2.ax1x.com/2019/11/05/KzgajS.png)

-   TCP 层: 建立传输层链接, 然后进行 SSH 协议处理
-   Handshake 层: SSH 协议里面的传输层, 该层主要提供加密传输
-   Authentication 层: 用户认证层 [`SSH-USERAUTH`], 主要提供用户认证
-   `Channel` && `Request`: SSH 协议里面的链接层 [`SSH-CONNECT`], 该层主要是将多个加密隧道分成逻辑通道, 可以复用通道, 常见的类型：`session`、`x11`、`forwarded-tcpip`、`direct-tcpip`, 通道里面的 `Requests` 是用于接收创建 ssh channel 的请求的, 而 ssh channel 就是里面的 connection, 数据的交互基于 connection 交互

##  0x02    构建 SSHD

####    SSHD 认证接口
库提供了认证接口，常使用的有下面三种方式：
-   `PasswordCallback`：可以实现 `PAM`、`Ldap`、自定义认证（比如可以将用户名和 password 存在 db 中，如此可以灵活实现后端认证的接口）
-   `PublicKeyCallback`：可以实现公钥认证、证书认证的接口
-   `KeyboardInteractiveCallback`：keyboard 方式，类似于 Challenge/Response 模式，Create challenges that the user has to answer interactively

这里以常见的用户名 `+` password 认证方式举例：
逻辑核心是实现 `PasswordCallback` 回调函数, 该函数的核心是返回一个 `ssh.Permissions` 对象, 其中保存了用户认证通过的信息（证书的额外信息：`CriticalOptions` 和 `Extensions`）：
```golang
type Permissions struct {
    // CriticalOptions indicate restrictions to the default
    // permissions, and are typically used in conjunction with
    // user certificates. The standard for SSH certificates
    // defines "force-command" (only allow the given command to
    // execute) and "source-address" (only allow connections from
    // the given address). The SSH package currently only enforces
    // the "source-address" critical option. It is up to server
    // implementations to enforce other critical options, such as
    // "force-command", by checking them after the SSH handshake
    // is successful. In general, SSH servers should reject
    // connections that specify critical options that are unknown
    // or not supported.
    CriticalOptions map[string]string

    // Extensions are extra functionality that the server may
    // offer on authenticated connections. Lack of support for an
    // extension does not preclude authenticating a user. Common
    // extensions are "permit-agent-forwarding",
    // "permit-X11-forwarding". The Go SSH library currently does
    // not act on any extension, and it is up to server
    // implementations to honor them. Extensions can be used to
    // pass data from the authentication callbacks to the server
    // application layer.
    Extensions map[string]string
}
```

实现口令认证的回调代码如下，如果想实现堡垒机的统一认证功能，可以实现更为负载的逻辑：）

```golang
config := &ssh.ServerConfig{
	PasswordCallback: func(c ssh.ConnMetadata, pass []byte) (*ssh.Permissions, error) {
		if c.User() == "root" && string(pass) == "xxxxxx" {
			return nil, nil
		}
		return nil, fmt.Errorf("password rejected for %v", c.User())
	},
}
```

####    SSHD 开发的完整流程
我们以 [sshd.go](https://gist.github.com/jpillora/b480fde82bff51a06238) 为例，简单看下如何构建一个 SSHD 服务端。<br>
一般来说，SSHD 的开发流程如下图所示：
![img](https://wx2.sbimg.cn/2020/08/19/3aCl7.png)

1、SSH-Server 配置认证回调及其他配置
注意配置中的 `NoClientAuth: true`，即任何客户端都可以连，一般不建议开启。此外，回调方法 `func` 中的 [`ConnMetadata`](https://godoc.org/golang.org/x/crypto/ssh#ConnMetadata) 结构保存了 ssh 连接的相关属性信息，可以用于会话追踪及 Debug 等。<br>

从 SSH 库暴露给用户的接口来看，这里可以很容易的实现统一认证的代码，非常灵活，现网中比较常用的有远程验证及 `PAM` 方式。

```golang
config := &ssh.ServerConfig{
    //Define a function to run when a client attempts a password login
    PasswordCallback: func(c ssh.ConnMetadata, pass []byte) (*ssh.Permissions, error) {
        // Should use constant-time compare (or better, salt+hash) in a production setting.
        if c.User() == "foo" && string(pass) == "bar" {
            return nil, nil
        }
        return nil, fmt.Errorf("password rejected for %q", c.User())
    },
    // You may also explicitly allow anonymous client authentication, though anon bash
    // sessions may not be a wise idea
    // NoClientAuth: true,
   	ServerVersion: "SSH-2.0-OWN-SERVER",
}
```

`ConnMetadata` 的结构如下：
```golang
type ConnMetadata interface {
    // User returns the user ID for this connection.
    User() string

    // SessionID returns the session hash, also denoted by H.
    SessionID() []byte

    // ClientVersion returns the client's version string as hashed
    // into the session ID.
    ClientVersion() []byte

    // ServerVersion returns the server's version string as hashed
    // into the session ID.
    ServerVersion() []byte

    // RemoteAddr returns the remote address for this connection.
    RemoteAddr() net.Addr

    // LocalAddr returns the local address for this connection.
    LocalAddr() net.Addr
}
```

2、配置 SSH-Server 的私钥
此秘钥用于 SSH 交互双方进行 Diffie-hellman 秘钥交换算法：
```golang
// You can generate a keypair with 'ssh-keygen -t rsa'
	privateBytes, err := ioutil.ReadFile("~/.ssh/id_rsa")
	if err != nil {
		log.Fatal("Failed to load private key")
	}

	private, err := ssh.ParsePrivateKey(privateBytes)
	if err != nil {
		log.Fatal("Failed to parse private key")
	}

	config.AddHostKey(private)
```

3、监听 TCP 服务
通过 `ssh.NewServerConn` 来升级 TCP 为 SSH 连接, SSH 库提供的这个函数根据之前的配置完成如下事情:
1. 传输层 [`SSH-TRANS`]
2. 认证层 [`SSH-USERAUTH`]
3. 返回链接层的通道: incoming Channels

这里的 `go ssh.DiscardRequests(reqs)` 是做什么用处？

```golang
listener, err := net.Listen("tcp", "127.0.0.1:2200")
if err != nil {
    log.Fatalf("Failed to listen on 2200 (%s)", err)
}

// Accept all connections
for {
    tcpConn, err := listener.Accept()
    if err != nil {
        log.Printf("Failed to accept incoming connection (%s)", err)
        continue
    }
    // Before use, a handshake must be performed on the incoming net.Conn.
    sshConn, chans, reqs, err := ssh.NewServerConn(tcpConn, config)
    if err != nil {
        log.Printf("Failed to handshake (%s)", err)
        continue
    }

    log.Printf("New SSH connection from %s (%s)", sshConn.RemoteAddr(), sshConn.ClientVersion())
    // Discard all global out-of-band Requests
    go ssh.DiscardRequests(reqs)
    // Accept all channels
    go handleChannels(chans)
}
```

在 `handleChannels` 方法开启单独的 Goroutine（不阻塞） 来处理这些连接的 Channel

4、建立 SSH 通道（SSH 链接层）
一个连接可能包含和很多不同的 channel，`handleChannel` 针对每个 channel 都单独处理：
```golang
func handleChannels(chans <-chan ssh.NewChannel) {
	// Service the incoming Channel channel in go routine
	for newChannel := range chans {
		go handleChannel(newChannel)
	}
}
```

5、处理 SSH 连接上的每个 Channel
`handleChannel` 方法是整个 ssh 协议交互实现的核心逻辑，在处理 ssh channel 的数据之前, 需要先获取 channel 的类型, 因为不同类型的 channel 里面是不同类型的数据，对应不同的处理逻辑。<br>
`Channel` 的类型可以参见 [RFC 文档： The Secure Shell (SSH) Protocol Assigned Numbers](https://tools.ietf.org/html/rfc4250#section-4.9.1)。<br>

例子里仅实现了常用的交互式 tty 的服务端，主要针对 `session` 类型 `Channel` 的处理，大概分为几个步骤：
1.  建立 SSH Channel（会话交互）
2.  实现数据交互
3.  实现指令交互（必要的 `Request` 及对应的处理方法）

```golang
func handleChannel(newChannel ssh.NewChannel) {
	// Since we're handling a shell, we expect a
	// channel type of "session". The also describes
	// "x11", "direct-tcpip" and "forwarded-tcpip"
	// channel types.
	if t := newChannel.ChannelType(); t != "session" {
        // 仅处理 session 类型的 channel，其他的 channel 暂时不处理
		newChannel.Reject(ssh.UnknownChannelType, fmt.Sprintf("unknown channel type: %s", t))
		return
	}

	// At this point, we have the opportunity to reject the client's
	// request for another logical connection
	connection, requests, err := newChannel.Accept()
	if err != nil {
		log.Printf("Could not accept channel (%s)", err)
		return
	}

	// Fire up bash for this session
	bash := exec.Command("bash")

	// Prepare teardown function
	close := func() {
		connection.Close()
		_, err := bash.Process.Wait()
		if err != nil {
			log.Printf("Failed to exit bash (%s)", err)
		}
		log.Printf("Session closed")
	}

	// Allocate a terminal for this channel
	log.Print("Creating pty...")
	bashf, err := pty.Start(bash)
	if err != nil {
		log.Printf("Could not start pty (%s)", err)
		close()
		return
	}

	//pipe session to bash and visa-versa
	var once sync.Once
	go func() {
		io.Copy(connection, bashf)
		once.Do(close)
	}()
	go func() {
		io.Copy(bashf, connection)
		once.Do(close)
	}()

	// Sessions have out-of-band requests such as "shell", "pty-req" and "env"
	go func() {
		for req := range requests {
			switch req.Type {
			case "shell":
				// We only accept the default shell
				// (i.e. no command in the Payload)
				if len(req.Payload) == 0 {
					req.Reply(true, nil)
				}
			case "pty-req":
				termLen := req.Payload[3]
				w, h := parseDims(req.Payload[termLen+4:])
				SetWinsize(bashf.Fd(), w, h)
				// Responding true (OK) here will let the client
				// know we have a pty ready for input
				req.Reply(true, nil)
			case "window-change":
				w, h := parseDims(req.Payload)
				SetWinsize(bashf.Fd(), w, h)
			}
		}
	}()
}
```

####    建立 SSH Channel
`ssh.NewChannel` 提供了 `Accept` 方法, 该方法 [原型如下](https://godoc.org/golang.org/x/crypto/ssh#NewChannel)：
```golang
// Accept accepts the channel creation request. It returns the Channel
// and a Go channel containing SSH requests. The Go channel must be
// serviced otherwise the Channel will hang.
Accept() (Channel, <-chan *Request, error)
```

`Accept()` 返回两个 Queue，前者用于数据交换 （架构图里面的 connection）， 后者用于控制指令交互（架构图中的 requests）：
```golang
// At this point, we have the opportunity to reject the client's
// request for another logical connection
connection, requests, err := newChannel.Accept()
if err != nil {
    log.Printf("Could not accept channel (%v)", err)
    return
}
```

####    实现数据交互
我们知道，交互式 `tty` 是打开一个终端 `tty`，然后在 `tty` 上通过键盘输入命令，回车执行，那么数据交互的过程就是打通 `tty` 和 `bash` 的通道。<br>
首先，使用 `exec.Command` 打开一个 `bash`， 然后将 `bash` 与 SSH Channel 对接, 从而实现和 `bash` 的远程交互。注意：这里可以使用定制化 `tty`（比如 `git` 也可通过 ssh 连接来交互）。比如，gliderlabs 的 [sshd 库](https://github.com/gliderlabs/ssh/blob/master/_examples/ssh-pty/pty.go) 代码，对外提供了 `pty` 接口，该库使用 `github.com/kr/pty` 包来实现 `sshd` 的 `tty` 交互功能。

PS：这里如果直接将 `bash` 的输入和输出直接对接 `terminal`，这是错误的操作， 因为 `bash` 没有运行在 `tty` 中，这里需要一个模拟 `tty` 来运行 `bash`。<br>

`tty` 和 `bash` 的关系如下图所示，这样就方便理解了。
![img](https://wx2.sbimg.cn/2020/08/20/3Nm3N.jpg)

最后需要将 `bash` 的管道（输入和输出）和 `Connection` 的管道对接，即完成了 SSHD 的 `tty` 实现逻辑。
```golang
// Fire up bash for this session
bash := exec.Command("bash")

// Prepare teardown function
// close 方法用于关闭 ssh channel 和退出 bash 程序
close := func() {
    connection.Close()
    _, err := bash.Process.Wait()
    if err != nil {
        log.Printf("Failed to exit bash (%s)", err)
    }
    log.Printf("Session closed")
}

// Allocate a terminal for this channel
bashf, err := pty.Start(bash)
if err != nil {
    log.Printf("Could not start pty (%s)", err)
    close()
    return
}

//pipe session to bash and visa-versa
var once sync.Once
go func() {
    io.Copy(connection, bashf)
    once.Do(close)
}()
go func() {
    io.Copy(bashf, connection)
    // 使用 Once 保证 close 函数 (connection 资源释放) 只被调用一次
    once.Do(close)
}()
```

####    实现指令交互
这里主要处理以下几种类型的 Request:
-   `shell/exec/subsystem`: channel request type for shell
    -   shell：启动的一个程序或者 `shell`
    -   exec：启动的是用户的默认 `shell`
    -   subsystem：启动一个子程序来执行 connection 里面的命令（比如 `sftp`）
-   `pty-req`: 准备一个 `pty` 等待输入
-   `window-change`: 监听 `tty` 窗口改变事件，及时更新 tty size

```golang
// Sessions have out-of-band requests such as "shell", "pty-req" and "env"
go func() {
    for req := range requests {
        switch req.Type {
        case "shell":
            // We only accept the default shell
            // (i.e. no command in the Payload)
            if len(req.Payload) == 0 {
                req.Reply(true, nil)
            }
        case "pty-req":
            termLen := req.Payload[3]
            w, h := parseDims(req.Payload[termLen+4:])
            SetWinsize(bashf.Fd(), w, h)
            // Responding true (OK) here will let the client
            // know we have a pty ready for input
            req.Reply(true, nil)
        case "window-change":
            w, h := parseDims(req.Payload)
            SetWinsize(bashf.Fd(), w, h)
        }
    }
}()
```

以上，我们就实现了一个简单的交互式的 SSH Server。

##  0x03    构建 SSH 客户端
SSH 常见的方式有交互式和命令执行两种：
####    交互模式
```golang
func main() {
        ce := func(err error, msg string) {
                if err != nil {
                        log.Fatalf("%s error: %v", msg, err)
                }
        }

        client, err := ssh.Dial("tcp", "localhost:1234",
                &ssh.ClientConfig{
                        User: "root",
                        Auth: []ssh.AuthMethod{ssh.Password("xxx")},
                })
        ce(err, "dial")
        session, err := client.NewSession()
        ce(err, "new session")
        defer session.Close()
        session.Stdout = os.Stdout
        session.Stderr = os.Stderr
        session.Stdin = os.Stdin
        modes := ssh.TerminalModes{
                ssh.ECHO:          1,
                ssh.ECHOCTL:       0,
                ssh.TTY_OP_ISPEED: 14400,
                ssh.TTY_OP_OSPEED: 14400,
        }
        termFD := int(os.Stdin.Fd())
        w, h, _ := terminal.GetSize(termFD)
        termState, _ := terminal.MakeRaw(termFD)
        defer terminal.Restore(termFD, termState)
        err = session.RequestPty("xterm-256color", h, w, modes)
        ce(err, "request pty")
        err = session.Shell()
        ce(err, "start shell")
        err = session.Wait()
        ce(err, "return")
}
```

####    远程命令模式


##	0x04	其他应用
基于 golang-SSH 库还有很多有趣的应用，这里列举几个：
[SSH Tron](https://github.com/zachlatta/sshtron) 基于此包实现了一个在线多人贪吃蛇服务


##	0x05	几个细节问题

####    客户端的 hostkey 认证
在客户端 `ssh.Dail` 中的 `HostKeyCallback`，为客户端验证 hostkey 的回调接口，通常我们都设置为 `ssh.InsecureIgnoreHostKey()`，但是这样做是不安全的（SSH 中间人攻击），参考 [Man-in-the-Middle Attack](https://www.ssh.com/attack/man-in-the-middle)：
```golang
client, err := ssh.Dial("tcp", "127.0.0.1:22", &ssh.ClientConfig{
    User: "root",
    Auth: []ssh.AuthMethod{ssh.Password("xxxxx")},
    HostKeyCallback: ssh.InsecureIgnoreHostKey(),
})
```
更安全的做法，是在这里检查上一次登录中保存的机器指纹数据（或者系统采集的机器指纹数据）与登录时的指纹进行比对，相同才放行，`HostKeyCallback` 的原型如下：
```golang
HostKeyCallback: func(hostname string, remote net.Addr, key ssh.PublicKey) error {
    // 实现机器指纹验证逻辑
    return nil
},
```

这里给出一个使用 knownhosts 做 hostkey 指纹校验的例子：
```golang

import (
    kh "golang.org/x/crypto/ssh/knownhosts"
)

func main() {
    hostKeyCallback, err := kh.New("/root/.ssh/known_hosts")
    if err != nil {
            log.Fatal("could not create hostkeycallback function:", err)
    }

    config := &ssh.ClientConfig{
            User: "root",
            Auth: []ssh.AuthMethod{
                    // Add in password check here for moar security.
                    ssh.Password("xxxxx"),
            },
            HostKeyCallback: hostKeyCallback,
    }
    // Connect to the remote server and perform the SSH handshake.
    client, err := ssh.Dial("tcp", address+":"+port, config)
    if err != nil {
            log.Fatalf("unable to connect: %v", err)
    }
    defer client.Close()
    ss, err := client.NewSession()
    if err != nil {
            log.Fatal("unable to create SSH session:", err)
    }
    defer ss.Close()
    ss.Stdout = os.Stdout
    ss.Run(command)
    fmt.Println("done")
}
```

####    客户端 keepalive 机制
如何实现 SSH 客户端的 keepalive 机制？答案是使用 `SendRequest` 定时向服务端发送心跳包，Stack Overflow 上给出了 [解决方案](https://stackoverflow.com/questions/31554196/ssh-connection-timeout)：
```golang
func SSHDialTimeout(network, addr string, config *ssh.ClientConfig, timeout time.Duration) (*ssh.Client, error) {
    conn, err := net.DialTimeout(network, addr, timeout)
    if err != nil {
        return nil, err
    }

    timeoutConn := &Conn{conn, timeout, timeout}
    c, chans, reqs, err := ssh.NewClientConn(timeoutConn, addr, config)
    if err != nil {
        return nil, err
    }
    client := ssh.NewClient(c, chans, reqs)

    // this sends keepalive packets every 2 seconds
    // there's no useful response from these, so we can just abort if there's an error
    go func() {
        t := time.NewTicker(2 * time.Second)
        defer t.Stop()
        for range t.C {
            _, _, err := client.Conn.SendRequest("keepalive@golang.org", true, nil)
            if err != nil {
                return
            }
        }
    }()
    return client, nil
}
```

####    客户端 SetEnv 问题
SSH 客户端可以使用 [`Setenv` 方法](https://godoc.org/golang.org/x/crypto/ssh#Session.Setenv) 在 ssh 会话中设置环境变量，其原理也是通过 `ssh.SendRequest` 方法，但是这里需要注意的是：<br>
在 SSH 服务端的配置 `/etc/ssh/sshd_config` 中需要加上这样的配置，如下：
```bash
AcceptEnv ENV_NAME
```
这样，客户端就可以设置名字为 `ENV_NAME` 的环境变量了。

```golang
func (s *Session) Setenv(name, value string) error {
	msg := setenvRequest{
		Name:  name,
		Value: value,
	}
	ok, err := s.ch.SendRequest("env", true, Marshal(&msg))
	if err == nil && !ok {
		err = errors.New("ssh: setenv failed")
	}
	return err
}
```

##  0x06    总结
本文介绍了使用 golang-SSH 包构造 SSHD、SSH 的一般方法，利用此包还可以完成其他一些有趣的项目。

##  0x07    参考
-   [Writing a replacement to OpenSSH using Go (2/2)](https://scalingo.com/blog/writing-a-replacement-to-openssh-using-go-22.html)
-   [Simple SSH Harvester in Go](https://parsiya.net/blog/2017-12-29-simple-ssh-harvester-in-go/)
-   [Go ssh 交互式执行命令](https://mritd.me/2018/11/09/go-interactive-shell/)
-   [SSH Client connection in Golang](https://blog.ralch.com/tutorial/golang-ssh-connection/)
-   [Writing an SSH server in Go](https://blog.gopheracademy.com/advent-2015/ssh-server-in-go/)