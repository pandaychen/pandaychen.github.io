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
本文汇总下 WebSocket 的应用及概念。笔者接触过的 websocket 有如下场景：
-   GoIm 的 [websocket Server](https://github.com/Terry-Mao/goim/blob/master/internal/comet/server_websocket.go)
-   支持 webconsole 登录到 SSH/docker 的转发层
-   长连接服务端（消息推送）
-	利用 websocket 构建正向代理

##  0x01   Websocket 基础及用法

WebSocket 是一种网络传输协议，可在单个 TCP 连接上进行全双工通信，位于 OSI 模型的应用层。WebSocket 使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就可以创建持久性的连接，并进行双向数据传输。

注意，Websocket 包含了握手及消息传输两个步骤，详细的基础知识请参考此文：[万字长文，一篇吃透 WebSocket：概念、原理、易错常识、动手实践](http://www.52im.net/thread-3713-1-1.html)

1. 用户打开 Web 浏览器
2. Web 浏览器（客户端）与 Web 服务端建立连接
3. Web 浏览器（客户端）能定时收发 Web 服务端数据，Web 服务端也能定时收发 Web 浏览器数据

####    协议格式
1、请求消息体

```text
# 请求头部分
# [请求方式] [资源路径] [版本]
GET /xxx HTTP/1.1
# 主机
Host: server.example.com
# 协议升级
Upgrade: websocket
# 连接状态
Connection: Upgrade
# websocket 客户端生成的随机字符
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
# websocket 协议的子协议，自定义字符，可以理解为频道
Sec-WebSocket-Protocol: chat, superchat
# websocket 协议的版本是 13
Sec-WebSocket-Version: 13
```

2、响应消息体

```text
# 响应头部分
# [版本] [状态码]
HTTP/1.1 101 Switching Protocols    #101 Switching Protocols 是 HTTP 协议状态码
# 协议升级
Upgrade: websocket
# 连接状态
Connection: Upgrade
# WebSocket 服务端根据 Sec-WebSocket-Key 生成的随机字符
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=  #Sec-WebSocket-Accept 这个则是经过服务器确认，并且加密过后的 Sec-WebSocket-Key
# WebSocket 协议的子协议，自定义字符，可以理解为频道
Sec-WebSocket-Protocol: chat
```

特别注意：`Upgrade` 字段仅限 HTTP/1.1 版本，不适合 HTTP/2.0 版本

####    持久连接
-   websocket 提供了一种服务端主动 push 数据给客户端的方式（基于 HTTP），相对于传统的 ajax 长连接 / long poll 技术
-   心跳重连：可以通过服务端实现 Pings / Pongs 方式实现心跳，即通过服务端向浏览器（客户端）发送 ping `0x9` 消息，浏览器会自动返回 pong `0xA` 消息


####    基础例子
使用 go 实现服务端、客户端 websocket 通信的代码如下：

1、server 端实现 [server.go](https://github.com/pandaychen/golang_in_action/blob/master/websocket/server.go)

```GO
func main() {
	//websocket 的升级接口
	upgrader := websocket.Upgrader{}

	http.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
		// 通过 upgrader 将 http 连接升级为 websocket 连接
		connect, err := upgrader.Upgrade(writer, request, nil)
		if nil != err {
			log.Println(err)
			return
		}

		defer connect.Close()

		// 定时向客户端发送数据
		go tickWriter(connect)

		// 启动数据读取循环，读取客户端发送来的数据
		for {
			// 从 websocket 中读取数据
			//messageType 消息类型，websocket 标准
			//messageData 消息数据
			messageType, messageData, err := connect.ReadMessage()
			if nil != err {
				log.Println(err)
				break
			}
			switch messageType {
			case websocket.TextMessage: // 文本数据
				fmt.Println(string(messageData))
			case websocket.BinaryMessage: // 二进制数据
				fmt.Println(messageData)
			case websocket.CloseMessage: // 关闭
			case websocket.PingMessage: //Ping
			case websocket.PongMessage: //Pong
			default:

			}
		}
	})

    // ....
}

func tickWriter(connect *websocket.Conn) {
	for {
		// 向客户端发送类型为文本的数据
		err := connect.WriteMessage(websocket.TextMessage, []byte("from server to client"))
		if nil != err {
			log.Println(err)
			break
		}

		time.Sleep(time.Second)
	}
}
```

2、client 端实现 [client.go](https://github.com/pandaychen/golang_in_action/blob/master/websocket/client.go)：双向读写

```go
func main() {

	dialer := websocket.Dialer{}
	// 向服务器发送连接请求，websocket 统一使用 ws://
	connect, _, err := dialer.Dial("ws://127.0.0.1:60000/", nil)
	if nil != err {
		log.Println(err)
		return
	}

	defer connect.Close()

	// 定时向客户端发送数据
	go tickWriter(connect)

	// 启动数据读取循环，读取客户端发送来的数据
	for {
		// 从 websocket 中读取数据
		//messageType 消息类型，websocket 标准
		//messageData 消息数据
		messageType, messageData, err := connect.ReadMessage()
		if nil != err {
			log.Println(err)
			break
		}
		switch messageType {
		case websocket.TextMessage: // 文本数据
			fmt.Println(string(messageData))
		case websocket.BinaryMessage: // 二进制数据
			fmt.Println(messageData)
		case websocket.CloseMessage: // 关闭
		case websocket.PingMessage: //Ping
		case websocket.PongMessage: //Pong
		default:

		}
	}
}

func tickWriter(connect *websocket.Conn) {
	for {
		// 向客户端发送类型为文本的数据
		err := connect.WriteMessage(websocket.TextMessage, []byte("from client to server"))
		if nil != err {
			log.Println(err)
			break
		}

		time.Sleep(time.Second)
	}
}
```

####    MITM with ws/wss
参考项目 [go-mitmproxy](https://github.com/lqqyt2423/go-mitmproxy) 的实现

1、ws 协议

2、wss 协议


####    websocket 的局限
参考 [为什么很少看到有人用 websocket？](https://www.v2ex.com/t/506933) 此文

-   websocket 是长连接，受网络限制比较大，需要处理好重连问题，大部分不重要的业务，使用 ws 不如使用 http 轮训来的简单
-   websocket 长连接的用户收到消息是个 PUSH 操作，http 轮训用户收消息是 PULL 操作；PUSH 都存在单生产推多消费，为广播模型，怎么处理好连接，保障每个消费推且只推一次？PULL 模式下消费方想要你就来生产方拉一下，拉几次，消息就准确的送达几次，不存在多消费和连接处理的问题，缺点当然就是消息推送的不及时，优点非常明显，简单易实现

####    websocket 的开发库
-   [websocket](https://github.com/gorilla/websocket)：稳定
-   [nbio](https://github.com/lesismal/nbio)：About Pure Go 1000k+ connections solution, support tls/http1.x/websocket and basically compatible with net/http, with high-performance and low memory cost, non-blocking, event-driven, easy-to-use

##  0x02    参考
-   [MQTT 和 Websocket 的区别是什么？](https://www.zhihu.com/question/21816631)
-   [万字长文，一篇吃透 WebSocket：概念、原理、易错常识、动手实践](http://www.52im.net/thread-3713-1-1.html)
-   [WebSocket 教程](https://www.ruanyifeng.com/blog/2017/05/websocket.html)
-   [为什么很少看到有人用 websocket？](https://www.v2ex.com/t/506933)
-   [golang 的哪个 websocket 好用？](https://www.v2ex.com/t/919140)