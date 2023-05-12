---
layout: post
title: Golang 网络编程（二）：xtcp - 轻量级TCP框架实现分析
subtitle: 分析一款典型的 TCP 框架
date: 2022-03-24
author: pandaychen
catalog: true
tags:
  - Golang
  - 网络编程
---


##  0x00    前言
[xtcp](https://github.com/xfxdev/xtcp/tree/master)是一个轻量级的tcp框架，支持用户自定义如下属性：
* how to define the protocol format
* how to create server and client
* how to custom the logger
* how to handle event
* how to stop the server and client

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

##  参考
- [example](https://github.com/xfxdev/xtcp/blob/master/_example/main.go)
