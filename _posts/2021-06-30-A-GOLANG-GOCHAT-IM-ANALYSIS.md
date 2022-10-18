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
-   用户注册&& 用户登录
-   用户发送消息
-   用户退出登录

gochat的架构图如下：
![gochat](https://raw.githubusercontent.com/LockGit/gochat/master/architecture/gochat.png)

层级结构如下：
-   对外接口层，主要包含API模块和connect模块，API模块主要给用户提供发送消息（指定人、广播等）的接口，connect模块主要用于长连接的消息推送（注意：用户主动发送消息是不直接通过connect模块的，该模块仅为了生成长连接推送消息用）
-   内部业务逻辑：
-   消息队列：
-   内部业务Task：独立工作进程

核心模块：
-   api模块
-   connect模块
-   logic模块：处理业务逻辑，分发用户的msg
-   task模块


##  0x01    核心数据结构
核心数据结构，[代码](https://github.com/LockGit/gochat/tree/master/connect)如下：

####    Server
```GO
type Server struct {
	Buckets   []*Bucket     //包含一个bucket数组
	Options   ServerOptions
	bucketIdx uint32
	operator  Operator
}
```

####   Bucket
一个`Bucket`包括了如下重要成员：
-   管理多个`Room`
-   管理多个`Channel`

```GO
type Bucket struct {
	cLock         sync.RWMutex     // protect the channels for chs
	chs           map[int]*Channel // map sub key to a channel      //包含一个映射Channel结构的map
	bucketOptions BucketOptions
	rooms         map[int]*Room // bucket room channels             //包含一个映射Room结构的map
	routines      []chan *proto.PushRoomMsgRequest          //包含一个chan *PushRoomMsgRequest的数组（用于向channel异步放入msg）
	routinesNum   uint64
	broadcast     chan []byte
}
```


####    Room
[`Room`](https://github.com/LockGit/gochat/blob/master/connect/room.go#L17)代表某个房间，房间必然有`N`个用户（长连接）
```GOLANG
type Room struct {
	Id          int
	OnlineCount int // room online user count
	rLock       sync.RWMutex
	drop        bool // make room is live
	next        *Channel        //room用来管理channel
}
```


####    Channel
`Channel`代表房间内的长连接，在同一个`Room`内，多个用户的长连接会话构建成一个doubleLinkList 
```go
//in fact, Channel it's a user Connect session
type Channel struct {
	Room      *Room         //属于哪个room
	Next      *Channel      //linklist
	Prev      *Channel      //linklist
	broadcast chan *proto.Msg
	userId    int
	conn      *websocket.Conn
	connTcp   *net.TCPConn
}
```

![double](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/double.png)

####    各个子模块的关系

各个数据结构之前的关系如下图所示，需要注意的是channel（会话）本质上是属于`Bucket`
![struct](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/go-chat/gochat-struct.png)

##	0x02	系统架构
笔者整理了下gochat的模块调用架构：
![architecture](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/gochat.png)

##	0x03	数据流程
全局时序图如下，来自官方文档，本小节详细梳理下各个子模块的工作过程

![timing](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/go-chat/timing.png)

####    用户发送信息流程
![send-message](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gochat/send-broadcast-message-flow.png)


##  0x04	参考
-	[gochat](https://github.com/LockGit/gochat)
-	[gochat 源码解析](https://blog.csdn.net/zhanglehes/article/details/115676339)