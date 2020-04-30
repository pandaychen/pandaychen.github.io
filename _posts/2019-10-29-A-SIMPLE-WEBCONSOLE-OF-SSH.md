---
layout:     post
title:      一个安全的Web-Console的实现思路
subtitle:   使用xterm.js+Go-gin实现Web-Console的SSH登录
date:       2019-10-29
author:     pandaychen
catalog:    true
tags:
    - WebConsole
---

##  基础
本文将描述如何实现一个具备安全认证的WebConsole，基于Golang-SSH库实现。WebConsole的核心实现是打通了WebSocket`+`SSH的输入输出流，非常适合于轻便运维的场景。WebSocket基于TCP传输协议，并复用HTTP的握手通道。

##  Web-console数据流
一个具备远程登陆的功能的Web-Console，其数据流向大概如下：<br>
User<--->Browser<--->WebSocket<--->SSH<--->(TTY)RemoteServer

## 认证
作为一个SSH登陆系统，认证是及其重要的一环，我们将上面的数据流扩展下：<br>
(少一张图)

##  实现组件

1.  CGI+WEB，采用开源的框架[Gin](https://github.com/gin-gonic/gin)实现
2.  [Websocket](https://github.com/gorilla/websocket)
3.  [SSH](https://godoc.org/golang.org/x/crypto/ssh)
4.  认证我们采用临时（一次性）Token兑换真实Token（如SSH证书/秘钥/口令等）的方式，这种方式简单易理解，当然了也可以使用OAuth2/OpenID这种标准协议

##  基本实现流程
1.  用户A申请某一台机器的登录权限，后台服务返回一个一次性token构造的url给用户，如 `https://1.2.3.4/cgi-bin/webconsole/check?token=token1`（这里假设以GET方式请求）；

2.  在IOA认证（为了获取合法用户的身份信息）的前提下，用户使用浏览器访问上述登录url, 后台服务先校验用户cookies及HTTP签名，然后再校验token1是否合法（使用次数+有效时间）；

3.  当前一步的认证通过后，后台服务返回websocket的地址，再加上一次性token2，如 `ws://1.2.3.4/cgi-bin/webconsole/login?token=token2`

4.  当Websocket请求到达后端，服务校验token2，校验通过后，后台将HTTP请求升级为WebSocket协议, 后续的数据交换则遵照WebSocket协议（这里就得到一个和浏览器数据交换的连接通道）；

4.  后台服务使用token2换取真实票据后, 与远端Server建立一个SSH Channel。然后后台将终端的大小等信息通过SSH Channel请求远程主机创建PTY,请求启动默认Shell；

5.  后台服务通过Socket连接通道获取用户（键盘）输入, 再通过SSH Channel将输入传给PTY, PTY将这些数据交给远程主机处理后按照前面指定的终端标准输出到SSH Channel中；

6.  后台服务从SSH Channel中拿到按照终端大小的标准输出后又通过Socket连接将输出返回给浏览器，至此一个Web Terminal建立成功。


##  一些实现代码细节

对tcp连接的升级，这是Golang非常常见的用法

### 升级TCP连接为SSH连接
``` go
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

### 升级HTTP连接为Web Socket
定义常量，web socket升级器:
``` go
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

###  WebTerminal数据流
![image](https://s2.ax1x.com/2019/11/06/MP6Zfs.png)

### SSH的Client/Channel/Request
下图直观展示了SSH的架构：
![image](https://s2.ax1x.com/2019/11/05/KzgajS.png)
-   Client: 
实现了SSH抽象的客户端

-   Channel和Request:
[SSH Channel的实现](https://github.com/golang/crypto/blob/master/ssh/connection.go#L50)
[SSH Requests的实现](https://github.com/golang/crypto/blob/57b3e21c3d5606066a87e63cfe07ec6b9f0db000/ssh/messages.go)
[SSH Conection的实现](https://github.com/golang/crypto/blob/master/ssh/connection.go)
这二者是SSH协议里面的链接层, 该层主要是将多个加密隧道分成逻辑通道，通道可以复用
常见通道类型有：`session`、`x11`、`forwarded-tcpip`、`direct-tcpip`。通道里面的Requests是用于接收创建SSH Channel的请求的，而SSH Channel就是里面的Connection, 数据的交互基于Connection完成。<br>

### 构建非交互式SSH客户端
现在看下如何创建一个非交互式的SSH客户端，作为WebConsole和用户的交互模块

1.通过`ssh.Dial()`创建一个SSH客户端连接
``` go
sshclient,err = ssh.Dial("tcp", addr, clientConfig)
```
2.通过SSH客户端创建 SSH Channel, 并请求一个pty伪终端, 并开启用户的默认Shell
``` go
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

### Remote Server与Browser实时数据交换
现在为止建立了两个`IO`通道，一个是WebSocket，另外一个是SSH Channel。由于需要双向转发数据，这里新建`2`个`groutine`, `groutine1`不停的从WebSocket通道里读取用户的输入, 并通过SSH Channel传给远程主机。`groutine2`负责将远程主机的数据（主要是终端屏显数据）传递给浏览器。
-   groutine1
``` go
go func() {
    for {
        // p为用户输入
        _, p, err := ws.ReadMessage()
        if err != nil {
            return
        }
        // 获取用户的输入，写入Channel
        _, err = this.channel.Write(p)
        if err != nil {
            return
        }
    }
}()
```
-   groutine2:
``` go
go func() {
    br := bufio.NewReader(this.channel)
    buf := []byte{}
    t := time.NewTimer(time.Microsecond * 100)
    defer t.Stop()
    // 构建一个信道, 一端将数据远程主机的数据写入, 一段读取数据写入ws
    r := make(chan rune)

    // 另起一个协程, 一个死循环不断的读取SSH Channel的数据, 并传给r-chan直到连接断开
    go func() {
        defer this.Client.Close()
        defer this.Session.Close()

        for {
            x, size, err := br.ReadRune()
            if err != nil {
                log.Println(err)
                ws.WriteMessage(1, []byte("\033[31m已经关闭连接!\033[0m"))
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
        // 每隔100微秒, 只要buf的长度不为0就将数据写入ws, 并重置时间和buf
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
        // 前面已经将SSH Channel里读取的数据写入创建的通道r, 这里读取数据, 不断增加buf的长度, 在设定的 100 microsecond后由上面判定长度是否返送数据
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

##  登录效果验证
直接在浏览器中输入已认证的url，成功通过WebConsole连上远端的SSH服务器，大功告成<br>
![image](https://s2.ax1x.com/2019/11/01/KHvxHS.png)


##  后记
1.  在整个系统中，最关键的点是怎样防止用户的身份被伪造，直观点，就是在第2步中，后台服务如何确定，当前的接口调用方就是用户A。另外，我们如何解决共享权限的场景，假设A申请了某台机器的登录权限，假设A授权B也可以使用该票据登录，那么我们的系统的认证如何完成呢？这个是很有趣的问题，待后面在工作中慢慢思考和实现吧。
2.  此外，作为SSH连接代理的服务（本文中以`CGI`服务承担）的稳定性也很重要，因为WebConsole的所有流量都会经由SSH连接代理转发，TCP连接也由代理维持，一旦代理故障，所有的WebConsole连接都会断开，所以可用性的设计也是非常重要的一环。

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权