---
layout:     post
title:      一个基于 golang 的轻量级 IM 项目分析：gochat
subtitle:   分析一款实时通信 IM 项目 gochat
date:       2021-06-30
author:     pandaychen
catalog:    true
tags:
    - IM
    - Golang
---

##  0x00    开篇
[gochat](https://github.com/LockGit/gochat) 是一款基于 golang 实现轻量级的 im 系统。技术上各层之间通过 RPC 通讯，使用 Redis 作为消息存储与投递的队列，模块间基于 etcd 的服务发现。其架构和 goim 很相像。本文简单了解下下面两个功能的实现：
-   点对点的消息发送（用户对用户）
-   点对面的消息发送（用户在房间里广播消息）

涉及到业务流程：
-   用户注册 && 用户登录
-   用户发送消息
-   用户退出登录

gochat 的架构图如下：
![gochat](https://raw.githubusercontent.com/LockGit/gochat/master/architecture/gochat.png)

层级结构如下：
-   对外接口层，主要包含 API 模块和 connect 模块，API 模块主要给用户提供发送消息（指定人、广播等）的接口，connect 模块主要用于长连接的消息推送（注意：用户主动发送消息是不直接通过 connect 模块的，该模块仅为了生成长连接推送消息用）
-   内部业务逻辑：
-   消息队列：
-   内部业务 Task：独立工作进程

[核心模块](https://github.com/pandaychen/gochat-note/blob/master/main.go#L27)：
-   api 模块：若干外部接口，以CGI方式提供
-   connect 模块：支持 tcp/websocket的长连接服务端，用来处理上行/下行用户数据
-   logic 模块：处理业务逻辑，分发用户的 msg
-   task 模块：消息队列的消费端，用来异步处理非实时场景下的业务逻辑


##  0x01    核心数据结构
核心数据结构，[代码](https://github.com/LockGit/gochat/tree/master/connect) 如下：

####    Server
```GO
type Server struct {
	Buckets   []*Bucket     // 包含一个 bucket 数组
	Options   ServerOptions
	bucketIdx uint32
	operator  Operator
}
```

####   Bucket
一个 `Bucket` 包括了如下重要成员：
-   管理多个 `Room`
-   管理多个 `Channel`

```GO
type Bucket struct {
	cLock         sync.RWMutex     // protect the channels for chs
	chs           map[int]*Channel // map sub key to a channel      // 包含一个映射 Channel 结构的 map
	bucketOptions BucketOptions
	rooms         map[int]*Room // bucket room channels             // 包含一个映射 Room 结构的 map
	routines      []chan *proto.PushRoomMsgRequest          // 包含一个 chan *PushRoomMsgRequest 的数组（用于向 channel 异步放入 msg）
	routinesNum   uint64
	broadcast     chan []byte
}
```


####    Room
[`Room`](https://github.com/LockGit/gochat/blob/master/connect/room.go#L17) 代表某个房间，房间必然有 `N` 个用户（长连接）
```GOLANG
type Room struct {
	Id          int
	OnlineCount int // room online user count
	rLock       sync.RWMutex
	drop        bool // make room is live
	next        *Channel        //room 用来管理 channel
}
```


####    Channel
`Channel` 代表房间内的长连接，在同一个 `Room` 内，**多个用户的长连接会话构建成一个 doubleLinkList（双向链表）**
```go
//in fact, Channel it's a user Connect session
type Channel struct {
	Room      *Room         // 属于哪个 room
	Next      *Channel      //linklist
	Prev      *Channel      //linklist
	broadcast chan *proto.Msg
	userId    int
	conn      *websocket.Conn
	connTcp   *net.TCPConn
}
```

![double](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/double.png)

注意：`Server`/`Room`/`Channel`/`Bucket` 存在相互引用的关系，所以这四个结构定义在同一层级比较合适

####    各个子模块的关系

各个数据结构之前的关系如下图所示，需要注意的是 channel（会话）本质上是属于 `Bucket`
![struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/gochat-struct.png)

##	0x02	系统架构
笔者整理了下 gochat 的模块调用架构：
![architecture](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/gochat.png)

##	0x03	数据流程
全局时序图如下，来自官方文档，本小节详细梳理下各个子模块的工作过程

![timing](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/timing.png)

####	用户注册 / 登录流程

![registry](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/flow_registry_and_login.png)

####	用户点对点发送消息流程

注意，CONNECT模块是TCP/websocket服务端（用来支持长连接），API是CGI层，LOGIC是RPC实现的服务端

![msg-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/flow(C2C).png)

####    用户发送信息流程
![send-message](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gochat/send-broadcast-message-flow.png)

####	用户注销流程

特别注意的是，在用户从 room 中退出时（此时用户还在 room 里）还需要广播此用户退出的消息给 room 内的所有用户（参考用户发送消息流程）

![logout](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/flow_Client_logout.png)

##	0x04	 核心代码分析

代码分析见[gochat-note](https://github.com/pandaychen/gochat-note)，主要是理清了数据走向之后代码就比较简单易懂了，摘录几个要点：

-	几个模块的作用，各司其职
-	Etcd服务发现（客户端）
-	用户到用户的消息发送/接受流程
-	用户向房间广播消息的流程
-	Server（CONNECT）->->Bucket->Room->Channel的结构关联
-	消息队列，项目中的redis可以扩展为其他的队列类型
-	除了CONNECT模块，其他的模块都是无状态，可以并发扩展的（得益于服务发现能力）
-	服务端模块可以考虑加入metrics指标数据监控

####	CONNECT模块
本项目CONNECT模块主要的功能是：
1.	作为长连接的数据传输
2.	客户端携带认证票据`authtoken`到CONNECT模块，然后认证成功在CONNECT模块上创建长连接、关联`bucket`、`room`关系
	-	websocket模块仅处理创建长连接
	-	tcp模块除了创建长连接之外，还额外加入了[`OpRoomSend`](https://github.com/pandaychen/gochat-note/blob/master/connect/server_tcp.go#L188)广播数据


####	LOGIC模块


####	TASK 模块
TASK模块的整体流程如下图：
![task-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/gochat-taskmodule-workflow.png)

##  0x05	参考
-	[gochat](https://github.com/LockGit/gochat)
-	[gochat 源码解析](https://blog.csdn.net/zhanglehes/article/details/115676339)