---
layout:     post
title:      WebSocket：原理与应用
subtitle:   WebSocket 在 golang 中的应用介绍
date:       2022-08-18
author:     pandaychen
catalog:    true
tags:
    - websocket
---


##  0x00    前言
本文汇总下WebSocket的应用及概念。笔者接触过的websocket有如下场景：
-   GoIm的[websocket Server](https://github.com/Terry-Mao/goim/blob/master/internal/comet/server_websocket.go)
-   支持webconsole登录到SSH/docker的转发层
-   长连接服务端

##  0x01   Websocket 基础及用法

WebSocket 是一种网络传输协议，可在单个 TCP 连接上进行全双工通信，位于 OSI 模型的应用层。WebSocket 使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就可以创建持久性的连接，并进行双向数据传输。

注意，Websocket包含了握手及消息传输两个步骤，详细的基础知识请参考此文：[万字长文，一篇吃透WebSocket：概念、原理、易错常识、动手实践](http://www.52im.net/thread-3713-1-1.html)

####    协议格式


####    基础例子
使用go实现服务端、客户端websocket通信的代码如下：

1、server端实现[server.go](https://github.com/pandaychen/golang_in_action/blob/master/websocket/server.go)

2、client端实现[client.go](https://github.com/pandaychen/golang_in_action/blob/master/websocket/client.go)


##  参考
-   [MQTT和Websocket的区别是什么？](https://www.zhihu.com/question/21816631)
-   [万字长文，一篇吃透WebSocket：概念、原理、易错常识、动手实践](http://www.52im.net/thread-3713-1-1.html)
-   [WebSocket 教程](https://www.ruanyifeng.com/blog/2017/05/websocket.html)