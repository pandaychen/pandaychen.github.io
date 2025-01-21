---
layout:     post
title:      使用 Golang 实现 SSH 和 SSHD（一）
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
2.  SSH-Proxy 代理（基于中间人会话劫持 + 转发控制与审计）
3.  SSH honeypot

##  0x01    SSH 架构
SSH 的基础架构图如下，进行项目开发之前，首先需要对 SSH 协议（分层）有个直观的认识：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/1029-ssh.png)

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
以 [sshd.go](https://gist.github.com/jpillora/b480fde82bff51a06238) 为例，简单看下如何构建一个 SSHD 服务端。<br>
一般来说，SSHD 的开发流程如下图所示：
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/sshd-dev-flow.png)

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
    // DisdCardRequest essentially discard those request that does not want a reply
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

注意上面代码中对`pty-req`以及`window-change`的处理都包含了`SetWinsize`这个方法，前者用于初始化登录设置，后者用于登录后发生窗口改变的监听，二者缺一不可。

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
交互式 `tty` 是打开一个终端 `tty`，然后在 `tty` 上通过键盘输入命令，回车执行，那么数据交互的过程就是打通 `tty` 和 `bash` 的通道。<br>
首先，使用 `exec.Command` 打开一个 `bash`， 然后将 `bash` 与 SSH Channel 对接, 从而实现和 `bash` 的远程交互。注意：这里可以使用定制化 `tty`（比如 `git` 也可通过 ssh 连接来交互）。比如，gliderlabs 的 [sshd 库](https://github.com/gliderlabs/ssh/blob/master/_examples/ssh-pty/pty.go) 代码，对外提供了 `pty` 接口，该库使用 `github.com/kr/pty` 包来实现 `sshd` 的 `tty` 交互功能。

PS：这里如果直接将 `bash` 的输入和输出直接对接 `terminal`，这是错误的操作， 因为 `bash` 没有运行在 `tty` 中，这里需要一个模拟 `tty` 来运行 `bash`。<br>

`tty` 和 `bash` 的关系如下图所示，这样就方便理解了。
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/tty-bash.png)

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

以上就实现了一个简单的交互式的 SSH Server。

##  0x03    构建 SSH 客户端
SSH 客户端常见的实现有交互式 tty-bash 和命令执行，其中命令执行又分为交互式 command 和一般的 bash 命令，比如 `gdb`、`top` 等就属于交互式命令，而 `ls` 就属于一般的 bash 执行。

####    交互模式
此模式为最常用的场景，客户端连接上服务器并获取了 tty-bash：
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
        session, err := client.NewSession()
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
        err = session.Shell()
        err = session.Wait()
}
```

####    远程命令模式
远程命令模式分为两种：<br>
1、一般的命令执行，执行完即返回结果 <br>
```golang
func main{
    //...session 的生成代码见上
    session.Stdout = os.Stdout
    session.Stderr = os.Stderr
    err = session.Run(command)
    if err != nil {
        fmt.Printf("\n[ERROR]Command [%s] run error(%s)", command, err.Error())
    } else {
        //fmt.Println(b.String())
        return "", nil
    }
    //...
}
```
2、交互式指令，如 `top`、`gdb` 等等，与上面最大的不同，是需要 `tty` 的支持（需要交互式输入输出的环境）
```golang
func main{
    //...session 的生成代码见上
    fd := int(os.Stdin.Fd())
    oldState, err := terminal.MakeRaw(fd)
    if err != nil {
        panic(err)
    }
    defer terminal.Restore(fd, oldState)

    session.Stdout = os.Stdout
    session.Stderr = os.Stdout
    session.Stdin = os.Stdin

    termWidth, termHeight, err := terminal.GetSize(fd)
    if err != nil {
        return "", err
    }
    modes := ssh.TerminalModes{
        ssh.ECHO:          1,
        ssh.TTY_OP_ISPEED: 14400,
        ssh.TTY_OP_OSPEED: 14400,
    }

    if err := session.RequestPty("xterm", termHeight, termWidth, modes); err != nil {
        return "", err
    }

    err = session.Run(command)
    if err != nil {
        return "", err
    }
    //...
}
```

####    sshAgent 机制
![ssh-agent](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/ssh_agent.gif)

[sshAgent](https://en.wikipedia.org/wiki/Ssh-agent) 严格的说是一个客户端认证机制，客户端通过和 `SSH_AUTH_SOCK` 这个环境变量指向的 Unix Domain Socket（本质上是 Sshd 建立的）通信，以获取其存储的私钥（实际也是保存在内存中，用户不可见），以此作为 ssh 的客户端验证方式：
```golang
func SSHAgent() ssh.AuthMethod {
	if sshAgent, err := net.Dial("unix", os.Getenv("SSH_AUTH_SOCK")); err == nil {
		return ssh.PublicKeysCallback(agent.NewClient(sshAgent).Signers)
	}
	return nil
}

func main(){
    //...
    sshConfig := &ssh.ClientConfig{
        User: "login_name",
        Auth: []ssh.AuthMethod{
            SSHAgent()
        },
    }
    //...
}
```

在此之前，需要在机器上开启 `ssh-agent` 功能及通过 `ssh-add` 向其中添加私钥：
```bash
[root@VM_0_7_centos ~]# eval `ssh-agent -s`
Agent pid 26303
[root@VM_0_7_centos ~]# ssh-add ~/.ssh/id_rsa
Identity added: /root/.ssh/id_rsa (/root/.ssh/id_rsa)
Certificate added: /root/.ssh/id_rsa-cert.pub ()
[root@VM_0_7_centos ~]# env|grep SSH_AUTH_SOCK
SSH_AUTH_SOCK=/tmp/ssh-IcFeG5r3XuQA/agent.26302
```

####    proxycommand机制介绍
Proxycommand是SSH的代理机制，如下图所示：
![proxycommand](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ssh/ssh-proxycommand.jpg)

通常有两种方式：

1、通过SSH原生方式<br>
ProxyCommand配置为：

```bash
ssh -W HostC:PortC -l USER -i PRIVATE_KEY -p PortB HostB
```

本方法原理是通过ssh命令自提供的代理机制，在机器A上建立与B的SSH连接（B机器与C机器建立连接在先），最终A连接C完成；A上ssh_config配置文件内容如下：
```bash
Host B
    HostName %h
    User xxx
    Port 12345
    IdentityFile ~/.ssh/id_dsa

Host C
    HostName %h
    User xxx 
    Port 22
    IdentityFile ~/.ssh/id_rsa
    ProxyCommand ssh -W %h:%p B
```

2、通过第三方软件连接<br>
常用的有`nc`、`corkscrew`等等，以`nc`为例，配置为：
```bash
nc -X 5 -x HostB:PortB HostC PortC  #-X 指定与代理服务器的通信协议 SOCKS4/SOCKS5/HTTPS
```

原理是利用`nc`在机器A上使用`nc`命令与代理服务器建立连接，该代理连接的B机器与C机器的SSH建立连接，最终A连接C完成；A上ssh_config配置文件内容如下：

```bash
Host C
    HostName %h
    User xxxx 
    Port 22
    IdentityFile ~/.ssh/id_rsa
    ProxyCommand nc -X 5 -x B:12345 %h %p
```

####    proxycommand的实战
通过corkscrew可以很轻松的搭建ssh-proxy，笔者修改了原始的corkscrew，支持多个代理跳转，[项目](https://github.com/pandaychen/corkscrew)，一种可能的配置方式如下（`~/.ssh/config`）：
```bash
ForwardAgent yes
StrictHostKeyChecking no
Port 22
ProxyCommand bash -c "/bin/corkscrew %r@%h"
```

上面是个非常有用的配置，`%r`、`%h`用来代替最终命令中的登录用户和登录ip（即动态的配置）；当然，`/bin/corkscrew`也可以换成其他的工具，在项目中是自己实现的proxycommand工具，加多了一些如权限控制，免密登录的特性。

PS：新版本的ssh客户端还支持`Match`指令，该指令可以决定是否触发子配置（类似PreCheck预先检查）：
```bash
Match exec "/bin/sometools check %h %r"
ForwardAgent yes
StrictHostKeyChecking no
Port 22
ProxyCommand bash -c "/bin/sometools proxy %r@%h"
```


####    SSH 客户端的多路复用机制
OpenSSH（5.6版本上）有一项特性叫作SSH 多路复用（SSH multiplexing），也称长连接保持（ControlPersist）。当使用SSH多路复用连接同一个服务器的SSH会话（一般是：登录用户#ip#port）时将会共享同一条TCP连接，这样消耗在TCP建立连接的时间就只会发生在第一次连接的时候（后续的直接复用ssh channel）

修改SSH客户端配置文件`~/.ssh/config`，增加下述配置
```text
Host *
    Compression yes
    ServerAliveInterval 30
    ServerAliveCountMax 360
    ControlMaster auto
    ControlPath ~/.ssh/socket/%h-%p-%r  
    ControlPersist yes
    #ControlPersist 4h  
```

配置说明如下：
-   `Host`：指定生效的 Host 范围
-   `ControlMaster`：设置为 `auto` 时，SSH 将尝试使用主连接，若对应的主连接不存在，将创建新连接作为主连接
-   `ControlPath`：设置存储复用的主连接的 UNIX 套接字文件，`%h`代表远程主机名、`%p`代表远程端口号、`%r`代表登录用户
-   `ControlPersist`：表示在创建首个连接（即主连接）的会话退出后，该连接是否仍然在后台保留，继续供其他复用该连接的会话使用

按照上述配置，SSH每连上一个Server都会自动在`~/.ssh/socket/`下创建一个Socket文件，**下次用相同用户名、端口、主机名进行连接就会自动复用**

SSH多路复用的原理如下：
1.  首次SSH连接到服务器时，OpenSSH将启动主连接
2.  OpenSSH创建一个UNIX 套接字（即Control socket）维持与远程服务器的连接
3.  当下一次SSH连接到服务器时（相同用户#IP#Port），OpenSSH将会使用已建立的控制套接字与服务器通信，而不再是重新建立新的TCP连接
4.  主连接保持的时间根据用户的配置决定，超时后SSH客户端将会关闭这个连接

####    scp客户端
基于 SSH 的文件传输有 `scp` 和 `sftp` 两种方式，区别如下：

-   `scp` 基于命令的标准 IO 实现，本地 `scp` 命令（客户端）会使用 SSH 连接协议，打开一个 `session`，通过 `SSH_MSG_CHANNEL_REQUEST` 的 `exec` 在服务端执行 `scp` 命令，服务端 `scp` 会读取文件，按照 `scp` 协议将文件写入标准输出，这个标准输出通过 SSH Channel 传递到本地的 `scp` 这个进程中，本地 `scp` 按照协议协议标准输出，并写入本地文件，即可完成文件下载；文件上传的过程类似；`scp`仅支持有限的指令，因为是借助于`scp`工具实现
-   `sftp` 是基于 SSH 连接协议的子系统（`subsystem`）实现的，对应的是 `SSH_MSG_CHANNEL_REQUEST` 的 `subsystem`，支持交互式的命令行界面及指令列表

下面详细介绍下scp服务端的原理：使用`scp`工具的客户端在连接上远端server的 `sshd` 后又启动了一个 `scp` 进程，利用 stdin 和 stdout/stderr 进行数据通信，也就是 `scp` 在本地客户端使用**必须依赖于目标服务器有 `scp` 存在**，不然会报错（下图）

![without-scp-error](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/openssh/scp/scp_binary_removed.png)

-   当ssh连接到服务端后，首先在远程服务端运行 `scp -t` 来告知 `scp` 运行模式是**从标准输入读入数据写入至指定文件**
-   `-t` 参数接收一个参数是将文件保存在哪里，即使用方式是 scp -t {target} 。如果目标文件不存在则会创建，但目标文件的父目录不存在则会报错；如果目标文件是一个目录，应当指定 `-d` 参数；如果需要输入详细日志指定 `-v` 参数，如果不指定 `-v` 参数那么所有的交互都是利用 stdin/stdout 完成的，指定了 `-v` 参数则会在 stderr 中输出调试日志。`scp` 的本身实现是如果调用时加了 `-v` 参数那么在启动远程 `scp` 服务时也加上 `-v` 参数

scp的交互协议如下描述，golang实现可以参考[Production-ready Secure Copy Protocol (SCP) implementation in Golang](https://github.com/povsister/scp)

1、请求（local->remote）：向远端发送消息并期待远端回复，消息的发送使用 stdin、回复使用 stdout，有如下case：

-   连接建立时不会向 stdin 写入内容，但远端会认为收到了一条内容而在 stdout 中回复；如果直接返回了错误（例如文件不存在）那么整个 scp 连接可以认为没有建立成功，应当直接报错退出
-   要创建文件时发送 `C{perm} {size} {filename}\n {文件内容}\0`，参考[`handleSend`](https://github.com/povsister/scp/blob/master/protocol.go#L214)
-   要进入目录时发送 `D{perm} 0 {filename}\n`，参考[`handleSend`](https://github.com/povsister/scp/blob/master/protocol.go#L242)
-   要退出目录时发送 `E\n`，参考[`handleSend`](https://github.com/povsister/scp/blob/master/protocol.go#L253)
-   要设定文件时间发送 `T...\n`，协议golang实现参考[`sendTimestamp`](https://github.com/povsister/scp/blob/master/protocol.go#L264)
-   权限 `perm` 是 `4` 位数字字符，与 Linux 权限位相同，常见的目录为 `0755` 文件为 `0644`

这里以`handleSend`方法来分析下文件内容的发送逻辑，分别代表着创建文件和写入内容，二者不可分割，但中间需要有一次检查 stdout 输出（`checkResponse`方法）。如果写入目标是文件那么文件名没有意义，而如果写入目标是目录那么该文件名会是要写入的内容的文件名。文件内容就是原始的文件内容，字节数应当和创建时给定的 `size` 相同，全都发送完后使用 `0x00` 代表结束然后检查 stdout。如果 `size` 和实际写入的字符数不同或是文件结束没有输入 `0x00` 都会出错

```GO
func handleSend(j sendJob, stream *sessionStream) {
	switch j.Type {
	case file:
		// close if required
		if j.close {
			if rc, ok := j.Reader.(io.ReadCloser); ok {
				defer rc.Close()
			}
		}
		// set timestamp for the next coming file
		if j.AccessTime != nil && j.ModifiedTime != nil {
			sendTimestamp(j, stream)
		}
		// send signal
		_, err := fmt.Fprintf(stream.In, "C0%o %d %s\n", j.Perm, j.Size, j.Destination)
		if err != nil {
			panicf("error sending signal C: %s", err)
		}
		checkResponse(stream)
		// send file
		_, err = io.Copy(stream.In, j.Reader)
		if err != nil {
			panicf("error sending file %q: %s", j.Destination, err)
		}
		_, err = fmt.Fprint(stream.In, "\x00")
		if err != nil {
			panicf("error finishing file %q: %s", j.Destination, err)
		}
		checkResponse(stream)

	case directory:
		if j.AccessTime != nil && j.ModifiedTime != nil {
			sendTimestamp(j, stream)
		}
		// size is always 0 for directory
		_, err := fmt.Fprintf(stream.In, "D0%o 0 %s\n", j.Perm, j.Destination)
		if err != nil {
			panicf("error sending signal D: %s", err)
		}
		checkResponse(stream)

	case exit:
		_, err := fmt.Fprintf(stream.In, "E\n")
		if err != nil {
			panicf("error sending signal E: %s", err)
		}
		checkResponse(stream)
	default:
		panicf("programmer error: unknown transferType %q", j.Type)
	}
}
```


2、回复（remote->local）：回复的规范是固定的，参考[`checkResponse`](https://github.com/povsister/scp/blob/master/protocol.go#L754)

-   第一个字节是 `0x00` / `0x01` / `0x02`，`0x00` 代表正确，`0x01`/`0x02` 代表有异常发生；其中`0x01` 代表错误，`0x02` 代表有严重错误，但因为有严重错误会导致进程直接被结束因此实际上客户端不可能收到 `0x02` 的返回
-   如果第一个字节不是 `0x00`（也就是说发生了错误），那么紧接着会有若干字符出现写明错误原因，这些字符都是 `ASCII` 字符，以 `\n`结尾

以目录的进入/退出为例，想要创建一个如下结构的目录，那么依次发送如下命令字：

```BASH
---- A
  ---- a
  ---- b
---- B
  ---- c
  ---- d
```

-   `D0755 0 A\n`：创建 `A` 目录，权限为 `755`，进入 `A` 目录（后续的文件都是在 `A` 目录中）
-   `C0644 1 a\n a\0` 在 `A` 目录中创建 `a` 文件，权限为 `644`，大小为 `1` 字节，内容为 `a`
-   `C0644 1 b\n b\0` 在 `A` 目录中创建 `b` 文件，权限为 `644`，大小为 `1` 字节，内容为 `b`
-   `E\n` 退出 `A` 目录
-   `D0755 0 B\n` 创建 `B` 目录，权限为 `755`，进入 `B` 目录
-   `C0644 2 c\n hh\0` 在 `B` 目录中创建 `c` 文件，权限为 `644`，大小为 `2` 字节，内容为 `hh`
-   `C0644 2 d\n XX\0` 在 `B` 目录中创建 `d` 文件，权限为 `644`，大小为 `2` 字节，内容设定为 `jj`
-   `E\n` 退出 `B` 目录

小结下：
-   对于上传文件场景：客户端按照`handleSend`的实现逻辑进行，需要按照scp协议规范，通过stdin向remote发送组装好的命令字、数据等
-   对于下载文件场景：客户端需要按照scp协议的规范对stdin的协议格式进行解析，然后执行相关的操作，参考[`copyFromRemote`](https://github.com/povsister/scp/blob/master/transfer.go#L449)的实现

1、scp客户端，通过`ssh.Session`配合在服务端运行指令`/usr/bin/scp -qrt $destdir`实现文件上传功能，代码如下：

```GO
func main() {
    client, err := ssh.Dial("tcp", "127.0.0.1:22", clientConfig)
    if err != nil {
        panic("Failed to dial: " + err.Error())
    }
    session, err := client.NewSession()
    if err != nil {
        panic("Failed to create session: " + err.Error())
    }
    defer session.Close()
    go func() {
        //数据上传逻辑实现
        w, _ := session.StdinPipe()
        defer w.Close()
        content := "123456789\n"
        fmt.Fprintln(w, "C0644", len(content), "testfile")
        fmt.Fprint(w, content)
        fmt.Fprint(w, "\x00") // 传输以\x00結束
    }()
    if err := session.Run("/usr/bin/scp -qrt ./"); err != nil {
        panic("Failed to run: " + err.Error())
    }
}
```

2、不依赖命令行的实现

参考[go-scp：Simple Golang scp client](https://github.com/bramvdbogaerde/go-scp)

##	0x04	其他应用
基于 golang-SSH 库还有很多有趣的应用，这里列举几个：<br>
-   [SSH Tron](https://github.com/zachlatta/sshtron)：此包实现了一个在线多人贪吃蛇服务
-   [Chat over SSH](https://github.com/shazow/ssh-chat)：基于 ssh 的聊天工具
-   [Easy SSH servers in Golang](https://github.com/gliderlabs/ssh)：一个通用的 sshd 框架实现
-   [Awesome SSH - Curated list of awesome lists](https://project-awesome.org/moul/awesome-ssh)：Awesome SSH Lists


##	0x05	几个细节问题

####    客户端的 hostkey 认证
在客户端 `ssh.Dail` 中的 `HostKeyCallback`，为客户端验证 hostkey 的回调接口，通常都设置为 `ssh.InsecureIgnoreHostKey()`，但是这样做是不安全的（SSH 中间人攻击），参考 [Man-in-the-Middle Attack](https://www.ssh.com/attack/man-in-the-middle)：
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

不过，在现网中是通过在交互式的 session 发送 `SendRequest` 来实现 keepalive，如下面的客户端代码：
```golang
func Client() {
        //....
        session, _ := client.NewSession()
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
        go func() {
                for {
                        time.Sleep(time.Second * time.Duration(30))
                        // 在当前的 session 中发送 keepalive
                        session.SendRequest("keepalive@openssh.com", true, nil)
                }
        }()
        session.Shell()
        session.Wait()
        //...
}
```

有个细节是，SSHD 服务端必须对 `keepalive@openssh.com` 这个 `Request` 有 reply 行为，不然的话，客户端会阻塞在 `session.SendRequest` 方法上；SSHD 服务端需要添加如下 [代码](https://github.com/pandaychen/golang_in_action/blob/master/sshserver/sshserver.go#L167)：
```golang
go func() {
    for req := range requests {
        switch req.Type {
        //......
        case "keepalive@openssh.com":
            req.Reply(true, nil)
        }
    }
}()
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
-   [Writing a replacement to OpenSSH using Go (1/2)](https://scalingo.com/blog/writing-a-replacement-to-openssh-using-go-12.html)
-   [Writing a replacement to OpenSSH using Go (2/2)](https://scalingo.com/blog/writing-a-replacement-to-openssh-using-go-22.html)
-   [Simple SSH Harvester in Go](https://parsiya.net/blog/2017-12-29-simple-ssh-harvester-in-go/)
-   [Go ssh 交互式执行命令](https://mritd.me/2018/11/09/go-interactive-shell/)
-   [SSH Client connection in Golang](https://blog.ralch.com/tutorial/golang-ssh-connection/)
-   [Writing an SSH server in Go](https://blog.gopheracademy.com/advent-2015/ssh-server-in-go/)
-   [SSH Agent Explained](https://smallstep.com/blog/ssh-agent-explained/)
-   [SSH Agent Forwarding 原理](https://blog.csdn.net/sdcxyz/article/details/41487897)
-   [ssh命令之ProxyCommand选项](https://dslztx.github.io/blog/2017/05/19/ssh%E5%91%BD%E4%BB%A4%E4%B9%8BProxyCommand%E9%80%89%E9%A1%B9/)
-   [SSH port forwarding with Go](https://eli.thegreenplace.net/2022/ssh-port-forwarding-with-go/)
-   [scp 原理](https://blog.singee.me/2021/01/02/d9e5fe31d708454fb99869a4c9d78f24/)