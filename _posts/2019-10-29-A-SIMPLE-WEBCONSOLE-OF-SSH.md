---
layout:     post
title:      一个安全的Web-Console的实现思路
subtitle:   使用xterm.js+Go-gin实现Web-Console的SSH登录
date:       2019-10-29
author:     pandaychen
catalog:    true
tags:
    - WebConsole
    - WebSocket
---

##  基础
本文将描述如何实现一个具备安全认证的WebConsole，基于GolangSSH库实现。WebConsole的核心实现是打通了WebSocket+SSH的输入输出流，非常适合于轻便运维的场景。WebSocket基于TCP传输协议，并复用HTTP的握手通道。

##  Web-console数据流
一个具备远程登陆的功能的Web-Console，其数据流向大概如下：<br>
User<--->Browser<--->WebSocket<--->SSH<--->(TTY)RemoteServer

## 认证
作为一个SSH登陆系统，认证是及其重要的一环，我们将上面的数据流扩展下：<br>
(少图)

##  实现组件

1.  CGI+WEB，采用开源的框架[Gin](https://github.com/gin-gonic/gin)实现
2.  [Websocket](https://github.com/gorilla/websocket)
3.  [SSH](https://godoc.org/golang.org/x/crypto/ssh)
4.  认证我们采用临时（一次性）Token兑换真实Token（如SSH证书/秘钥/口令等）的方式，这种方式简单易理解，当然了也可以使用Oauth2/OpenID这种标准协议

##  基本实现流程
1.  用户申请某一台机器的登录权限，后台服务返回一个一次性token构造的url给用户，如https://1.2.3.4/cgi-bin/webconsole/check?token=token1（这里假设以GET方式请求）；

2.  在IOA认证（为了获取合法用户的身份信息）的前提下，用户使用浏览器访问上述登录url, 后台服务先校验用户cookies及HTTP签名，然后再校验token1是否合法（使用次数+有效时间）；

3.  当前一步的认证通过后，后台服务返回websocket的地址，再加上一次性token2，如ws://1.2.3.4/cgi-bin/webconsole/login?token=token2

4.  当Websocket请求到达后端，服务校验token2，校验通过后，后台将HTTP请求升级为WebSocket协议, 后续的数据交换则遵照WebSocket协议（这里就得到一个和浏览器数据交换的连接通道）；

4.  后台服务使用token2换取真实票据后, 与远端Server建立一个SSH Channel。然后后台将终端的大小等信息通过SSH Channel请求远程主机创建PTY,请求启动默认Shell；

5.  后台服务通过Socket连接通道获取用户（键盘）输入, 再通过SSH Channel将输入传给PTY, PTY将这些数据交给远程主机处理后按照前面指定的终端标准输出到SSH Channel中；

6.  后台服务从SSH Channel中拿到按照终端大小的标准输出后又通过Socket连接将输出返回给浏览器，至此一个Web Terminal建立成功。


##  一些实现代码细节

对tcp连接的升级，这是Golang非常常见的用法

### 升级TCP连接为SSH连接的方法
```
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

### 升级HTTP连接为web Socket
```
定义常量, 升级器:
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



##  登录效果
直接在浏览器中已认证的url，成功通过WebConsole连上远端的SSH服务器，大功告成<br>
![image](https://s2.ax1x.com/2019/11/01/KHvxHS.png)


##

##  后记
在整个系统中，最关键的点是怎样防止用户的身份被伪造
