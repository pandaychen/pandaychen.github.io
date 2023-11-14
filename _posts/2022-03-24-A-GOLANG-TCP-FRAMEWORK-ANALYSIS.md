---
layout: post
title: Golang 网络编程（二）：轻量级 TCP 框架实现分析
subtitle: 分析典型的 TCP 框架：getty && xtcp
date: 2022-03-24
author: pandaychen
catalog: true
tags:
  - Golang
  - 网络编程
---


##  0x00    前言
本文分析两个典型的 tcp-framework 实现：

[xtcp](https://github.com/xfxdev/xtcp/tree/master) 是一个轻量级的 tcp 框架，支持用户自定义如下属性：
* how to define the protocol format
* how to create server and client
* how to custom the logger
* how to handle event
* how to stop the server and client

[getty](https://github.com/AlexStocks/getty)：a netty like asynchronous network I/O library based on tcp/udp/websocket; a bidirectional RPC framework based on JSON/Protobuf; a microservice framework based on zookeeper/etcd

##  0x01  核心结构封装

####  [Server](https://github.com/xfxdev/xtcp/blob/master/xserver.go#L10)
```golang
// Server used for running a tcp server.
type Server struct {
	Opts    *Options
	stopped chan struct{}
	wg      sync.WaitGroup
	mu      sync.Mutex
	once    sync.Once
	lis     net.Listener
	conns   map[*Conn]bool
}
```

`conns   map[*Conn]bool`


####  Conn：单个连接
```go
// A Conn represents the server side of an tcp connection.
type Conn struct {
	sync.Mutex
	Opts        *Options
	RawConn     net.Conn
	UserData    interface{}
	sendBufList chan []byte
	closed      chan struct{}
	state       int32
	wg          sync.WaitGroup
	once        sync.Once
	SendDropped uint32
	sendBytes   uint64
	recvBytes   uint64
	dropped     uint32
}
```


##	0x0	getty 库：介绍



##  参考
- [example](https://github.com/xfxdev/xtcp/blob/master/_example/main.go)
- [getty 开发日志](https://alexstocks.github.io/html/getty.html)
- [Go 网络库 getty 的那些事](https://cloud.tencent.com/developer/article/1877651)
- [dubbo-go 的开发、设计与功能介绍](https://dubbogo.github.io/zh-cn/docs/md/arch/dubbo-go-design-implement-and-featrues.html)