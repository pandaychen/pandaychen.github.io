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


##	0x02	使用 Websocket 协议登录容器
在 [一个安全的 Web-Console 的实现思路](https://pandaychen.github.io/2019/10/29/A-SIMPLE-WEBCONSOLE-OF-SSH/)、[一种基于 TTY-based 的 kubernetes console 实现思路](https://pandaychen.github.io/2022/06/01/A-KUBERNETES-CONSOLE/) 中介绍了使用 websocket 登录容器的一些思路方法，由于 `kubectl exec` 命令行是实时交互的，输入和输出实时发生，所以 kubernetes 使用 SPDY/Websocket 等双向实时通信的协议，来传递输入 / 输出内容，数据流路径为：

```text
Kubectl <---(双向实时协议 SPDY/Websocket)---> Kube-Api-Server <---(双向实时协议)---> 节点 kubelet
```

旧版本 `kubectl` 使用 SPDY 协议连接 kubernetes 的 API-server，新版本已经被 HTTP2 所替代，参考 [SPDY is deprecated. Switch to HTTP/2](https://github.com/kubernetes/kubernetes/issues/7452)，kubernetes 的 API-server 同时支持 SPDY/Websocket 协议

比如通常使用如下代码连接容器，就是使用的 SPDY 协议的方式：
```go
exec, err := remotecommand.NewSPDYExecutor(config, method, url)
if err != nil {
   return err
}

// 这里开始，将输入输出，进行实时传递（Stream）
return exec.Stream(remotecommand.StreamOptions{
   Stdin:             stdin,
   Stdout:            stdout,
   Stderr:            stderr,
   Tty:               tty,
   TerminalSizeQueue: terminalSizeQueue,
})
```

下面描述下如何使用 websocket 来构建容器登录，采用的架构如下：

![gw](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/websocket/websocket-to-kubernetes.png)

####	former-CONN（Websocket）
这里有细节需要注意，客户端使用 Websocket 去连接登录网关，然后网关再通过 websocket 去连接容器，前向后向都是 websocket 协议，看起来直接协议转发就好了？答案是否定的。以客户端（用户）敲下 `ls` 命令到容器，然后容器 list 文件列表的场景为例，如果直接发送 `ls` 内容，那么肯定是不通的，因为 kubernetes 根本不认这种输入，kubernetes 认为 websocket 的报文内容是归属有频道的，不然一条 cmd 命令行执行后，用户无法判断响应的内容是 stdout？还是 stderr？

由于 kubernetes 的 `exec` 在使用 Websocket 协议时是有扩展的，并且扩展规则是 kubernetes 自己设置的规则。kubernetes 对 websocket 的约定规则如下：

| 第一字节值 | 内容含义 |
| :----:|:----:|
| 0| 标准输入 |
| 1| 标准输出 |
| 2| 标准错误 |
| 3| 服务端异常信息 |
| 4| terminal 窗口大小调整 resize|

小结下，当 WebSocket 建立连接后，发送数据时需要将缓冲的第一个字节定义为 stdin（`buf [0] = 0`），而接收数据时要判断 stdout（`buf [0] = 1`）与 stderr（`buf [0] = 2`）

```TEXT
＃传送 `ls` 指令，必须 buf[0] 自行填入 0 来表示
stdin.buf = [0 108 115 10]

＃接收
BUF = [1 108 115 13 10 27 91 48 109 27 91 ...]
```

####	latter-CONN（websocket）
第二个细节是在与 kubernetes websocket 频道的内容，需要使用 `base64` 进行编解码，当要发送 `ls` 命令，需要向 kubernetes 发送的内容如下：

```js
// 这样发送，kubernetes 才认为是收到 ls 命令
sendMsg := "0" + base64.Encoding("ls")
```

当从 kubernetes 收到响应数据时，网关需要先去掉第一个字节，然后再进行 `base64` 解码，这样才可以获取到真实的 stdout/stder 文本数据

```JS

// 发送内容
ws.send("0" + utf8_to_b64(data));

// 接收内容
switch(ev.data[0]) {
case '1':
case '2':
case '3':
term.write(b64_to_utf8(data));
break;
}
```

##  0x03    参考
-   [MQTT 和 Websocket 的区别是什么？](https://www.zhihu.com/question/21816631)
-   [万字长文，一篇吃透 WebSocket：概念、原理、易错常识、动手实践](http://www.52im.net/thread-3713-1-1.html)
-   [WebSocket 教程](https://www.ruanyifeng.com/blog/2017/05/websocket.html)
-   [为什么很少看到有人用 websocket？](https://www.v2ex.com/t/506933)
-   [golang 的哪个 websocket 好用？](https://www.v2ex.com/t/919140)
-	[利用 kubernetes exec 接口实现任意容器的 web-terminal](https://bbs.huaweicloud.com/blogs/281515)