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
本文将描述如何实现一个具备安全认证的WebConsole，基于GolangSSH库实现。WebConsole的核心实现是打通了WebSocket+SSH的输入输出流，非常适合于轻便运维的场景。

##  Web-console数据流
一个具备远程登陆的功能的Web-Console，其数据流向大概如下：<br>
User<--->Browser<--->WebSocket<--->SSH<--->(TTY)RemoteServer

## 认证
作为一个SSH登陆系统，认证是及其重要的一环，我们将上面的数据流扩展下：<br>
(少图)

##  实现方案
1.  CGI+WEB，采用开源的框架[Gin](https://github.com/gin-gonic/gin)实现
2.  [Websocket](https://github.com/gorilla/websocket)
3.  [SSH](https://godoc.org/golang.org/x/crypto/ssh)
4.  认证我们采用临时(一次性)Token兑换真实Token的方式，这种方式简单易理解


##  后记
在整个系统中，最关键的点是怎样防止用户的身份被伪造。
