---
layout:     post
title:      使用 Golang 实现 SSH 和 SSHD
subtitle:   golang-ssh 库使用之（一）
date:       2019-10-20
author:     pandaychen
catalog:    true
tags:
    - Golang
    - Openssh
---

##  0x00    前言
&emsp;&emsp; [golang 的 SSH 包](https://godoc.org/golang.org/x/crypto/ssh) 提供了极为丰富的接口。基于此包可以很容易的实现 SSH 客户端、SSHD 服务端以及 SSH 代理等常用工具。

##  0x01    SSH 架构
SSH 的基础架构图如下，开发的话，首先需要对 SSH 协议（分层）有个直观的认识：
![image](https://s2.ax1x.com/2019/11/05/KzgajS.png)

-   TCP 层: 建立传输层链接, 然后进行 SSH 协议处理
-   Handshake 层: SSH 协议里面的传输层, 该层主要提供加密传输
-   Authentication 层: 用户认证层 [SSH-USERAUTH], 主要提供用户认证
-   Channel && Request: SSH 协议里面的链接层 [SSH-CONNECT], 该层主要是将多个加密隧道分成逻辑通道, 可以复用通道, 常见的类型：`session`、`x11`、`forwarded-tcpip`、`direct-tcpip`, 通道里面的 Requests 是用于接收创建 ssh channel 的请求的, 而 ssh channel 就是里面的 connection, 数据的交互基于 connection 交互



##  参考
-   [Writing a replacement to OpenSSH using Go (2/2)](https://scalingo.com/blog/writing-a-replacement-to-openssh-using-go-22.html)
-   [Simple SSH Harvester in Go](https://parsiya.net/blog/2017-12-29-simple-ssh-harvester-in-go/)