---
layout:     post
title:      一个安全的 Web-Console 的实现思路
subtitle:   使用 xterm.js+Go-gin 实现 Web-Console 的 SSH 登录
date:       2019-10-29
author:     pandaychen
header-img: img/super-mario.jpg
catalog:    true
tags:
    - WebConsole
---

##  0x00    基础
本文将描述如何实现一个具备安全认证的 WebConsole，基于 [Golang-SSH 库](https://godoc.org/golang.org/x/crypto/ssh) 实现。WebConsole 的核心实现是打通了 WebSocket`+`SSH 的输入输出流，使得用户直接使用浏览器就可以运行 SSH 终端，非常适合于轻便运维的场景。WebSocket 基于 TCP 传输协议，并复用 HTTP 的握手通道，关于 WebSocket 和 Golang 的开发可以参见：[How to Use Websockets in Golang: Best Tools and Step-by-Step Guide](https://yalantis.com/blog/how-to-build-websockets-in-go/)。

##  0x01    WebConsole 数据流
一个具备远程登陆的功能的 Web-Console，其数据流向大概如下：<br>
User`<--->`Browser`<--->`WebSocket`<--->`SSH`<--->`(TTY)RemoteServer

####    数据流
中间的 Proxy 代理层，负责将 websocket 流转换为 SSH 流（核心是输入和输出的转发）：

![image](https://s2.ax1x.com/2019/11/06/MP6Zfs.png)

![image](https://wx1.sbimg.cn/2020/08/04/oMP4j.png)


##  0x02    实现
作为一个 SSH 登陆系统，认证是及其重要的一环，我们将上面的数据流扩展下，加入必要的身份 / 票据认证：<br>
(少一张图)

####  组件

1.  CGI+WEB，采用开源的框架 [Gin](https://github.com/gin-gonic/gin) 实现
2.  [Websocket](https://github.com/gorilla/websocket)
3.  [SSH](https://godoc.org/golang.org/x/crypto/ssh)
4.  认证我们采用临时（一次性）Token 兑换真实 Token（如 SSH 证书 / 秘钥 / 口令等）的方式，这种方式简单易理解，当然了也可以使用 OAuth2/OpenID 这种标准协议

####  基本实现流程
1.  用户 A 申请某一台机器的登录权限，后台服务返回一个一次性 token 构造的 url 给用户，如 `https://1.2.3.4/cgi-bin/webconsole/check?token=token1`（这里假设以 GET 方式请求）；

2.  在 IOA 认证（为了获取合法用户的身份信息）的前提下，用户使用浏览器访问上述登录 url, 后台服务先校验用户 cookies 及 HTTP 签名，然后再校验 token1 是否合法（使用次数 + 有效时间）；

3.  当前一步的认证通过后，后台服务返回 websocket 的地址，再加上一次性 token2，如 `ws://1.2.3.4/cgi-bin/webconsole/login?token=token2`

4.  当 Websocket 请求到达后端，服务校验 token2，校验通过后，后台将 HTTP 请求升级为 WebSocket 协议, 后续的数据交换则遵照 WebSocket 协议（这里就得到一个和浏览器数据交换的连接通道）；

4.  后台服务使用 token2 换取真实票据后, 与远端 Server 建立一个 SSH Channel。然后后台将终端的大小等信息通过 SSH Channel 请求远程主机创建 PTY, 请求启动默认 Shell；

5.  后台服务通过 Socket 连接通道获取用户（键盘）输入, 再通过 SSH Channel 将输入传给 PTY, PTY 将这些数据交给远程主机处理后按照前面指定的终端标准输出到 SSH Channel 中；

6.  后台服务从 SSH Channel 中拿到按照终端大小的标准输出后又通过 Socket 连接将输出返回给浏览器，至此一个 Web Terminal 建立成功。


##  0x03    一些代码细节


####    升级 TCP 连接为 SSH 连接
对 Tcp 连接进行升级，这是 Golang 中非常常见的做法：
```golang
tcpConn, err := listener.Accept()
if err != nil {
    log.Printf("failed to accept incoming connection (%s)", err)
    continue
}
// Before use, a handshake must be performed on the incoming net.Conn.
sshConn, chans, reqs, err := ssh.NewServerConn(tcpConn, sshServerConfig)
if err != nil {
    log.Printf("failed to handshake (%s)", err)
    continue
}
```

####    升级 HTTP 连接为 Web Socket
定义常量，web socket 升级器:
```golang
var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}

websock_conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
if err != nil {
    c.Error(err)
    return
}
```


####    SSH 的层次结构 Client/Channel/Request
下图直观展示了 SSH 的架构：

![image](https://s2.ax1x.com/2019/11/05/KzgajS.png)

-   Client: 实现了 SSH 抽象的客户端
-   Channel 和 Request:
    -   [SSH Channel 的实现](https://github.com/golang/crypto/blob/master/ssh/connection.go#L50)
    -   [SSH Requests 的实现](https://github.com/golang/crypto/blob/master/ssh/messages.go)
    -   [SSH Conection 的实现](https://github.com/golang/crypto/blob/master/ssh/connection.go)

这二者是 SSH 协议里面的链接层, 该层主要是将多个加密隧道分成逻辑通道，通道可以复用
常见通道类型有：`session`、`x11`、`forwarded-tcpip`、`direct-tcpip`。通道里面的 `Requests` 是用于接收创建 SSH `Channel` 的请求的，而 SSH `Channel` 就是里面的 `Connection`, 数据的交互是基于 `Connection` 完成。<br>

####    构建非交互式 SSH 客户端
现在看下如何创建一个非交互式的 SSH 客户端，作为 WebConsole 和用户的交互模块：

1、通过 `ssh.Dial()` 创建一个 SSH 客户端连接
```golang
sshclient,err = ssh.Dial("tcp", addr, clientConfig)
```

2、通过 SSH 客户端创建 SSH Channel, 并请求一个 pty 伪终端, 并开启用户的默认 Shell
```golang
channel, inRequests, err := sshclient.OpenChannel("session", nil)
if err != nil {
    log.Println(err)
    return nil
}
ok, err := channel.SendRequest("pty-req", true, ssh.Marshal(&req))
if !ok || err != nil {
    log.Println(err)
    return nil
}

ok, err = channel.SendRequest("shell", true, nil)
if !ok || err != nil {
    log.Println(err)
    return nil
}
```

####    Remote Server 与 Browser 实时数据交换
现在为止建立了两个 `IO` 通道，一个是 `WebSocket` 通道，另外一个是 SSH `Channel`。由于需要双向转发数据，这里新建 `2` 个 `groutine`, `groutine1` 不停的从 WebSocket 通道里读取用户的输入, 并通过 SSH Channel 传给远程主机。`groutine2` 负责将远程主机的数据（主要是终端屏显数据）传递给浏览器。
1、`groutine1` 主要完成从 `Websocket` 中读取数据，通过 ssh 的 `Channel` 发送到目标服务器
```golang
go func() {
    for {
        // p 为用户输入
        _, p, err := ws.ReadMessage()
        if err != nil {
            return
        }
        // 获取用户的输入，写入 Channel
        _, err = this.channel.Write(p)
        if err != nil {
            return
        }
    }
}()
```

2、`groutine2` 主要完成从 SSH `Channel` 中读取数据，写入 `Websocket`，这样用户在浏览器上可以看到操作回显了：
```golang
go func() {
    br := bufio.NewReader(this.channel)
    buf := []byte{}
    t := time.NewTimer(time.Microsecond * 100)
    defer t.Stop()
    // 构建一个信道, 一端将数据远程主机的数据写入, 一段读取数据写入 ws
    r := make(chan rune)

    // 另起一个协程, 一个死循环不断的读取 SSH Channel 的数据, 并传给 r-chan 直到连接断开
    go func() {
        defer this.Client.Close()
        defer this.Session.Close()

        for {
            x, size, err := br.ReadRune()
            if err != nil {
                log.Println(err)
                ws.WriteMessage(1, []byte("\033[31m 已经关闭连接!\033[0m"))
                ws.Close()
                return
            }
            if size > 0 {
                r <- x
            }
        }
    }()

    // 主循环
    for {
        select {
        // 每隔 100 微秒, 只要 buf 的长度不为 0 就将数据写入 ws, 并重置时间和 buf
        case <-t.C:
            if len(buf) != 0 {
                err := ws.WriteMessage(websocket.TextMessage, buf)
                buf = []byte{}
                if err != nil {
                    log.Println(err)
                    return
                }
            }
            t.Reset(time.Microsecond * 100)
        // 前面已经将 SSH Channel 里读取的数据写入创建的通道 r, 这里读取数据, 不断增加 buf 的长度, 在设定的 100 microsecond 后由上面判定长度是否返送数据
        case d := <-r:
            if d != utf8.RuneError {
                p := make([]byte, utf8.RuneLen(d))
                utf8.EncodeRune(p, d)
                buf = append(buf, p...)
            } else {
                buf = append(buf, []byte("@")...)
            }
        }
    }
}()
```

##  0x04    登录效果验证
直接在浏览器中输入已认证的 `url`，成功通过 `WebConsole` 连上远端的 SSH 服务器，大功告成 <br>
![image](https://s2.ax1x.com/2019/11/01/KHvxHS.png)


##  0x05    总结
1.  在整个系统中，最关键的点是怎样防止用户的身份被伪造，直观点，就是在第 2 步中，后台服务如何确定，当前的接口调用方就是用户 A。另外，我们如何解决共享权限的场景，假设 A 申请了某台机器的登录权限，假设 A 授权 B 也可以使用该票据登录，那么我们的系统的认证如何完成呢？这个是很有趣的问题，待后面在工作中慢慢思考和实现吧。
2.  此外，作为 SSH 连接代理的服务（本文中以 `CGI` 服务承担）的稳定性也很重要，因为 WebConsole 的所有流量都会经由 SSH 连接代理转发，TCP 连接也由代理维持，一旦代理故障，所有的 WebConsole 连接都会断开，所以可用性的设计也是非常重要的一环。
3.  整个 Web 页面需要前置认证机制，比如接入 github 的 Oauth、Onelogin 等等


转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权