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
[getty](https://github.com/AlexStocks/getty)一个类似 Netty 的异步网络 I/O 库，是一个很典型的tcpframe，Getty 基于分层设计，主要分为数据交互层、业务控制层、网络层，同时还提供非常易于扩展的监控接口，对外暴露的网络库使用接口。初步分析可以从其[echo-server](https://github.com/AlexStocks/getty/tree/master/examples/echo/tcp-echo/server)例子入手，先窥探下getty构建一个简单的服务端，（至少）需要用户实现哪些功能？

####	协议
[echo](https://github.com/AlexStocks/getty/blob/master/examples/echo/tcp-echo/server/app/echo.go)协议，这个似乎没啥好说的，需要用户自行设置TLV的协议

```GO
type EchoPkgHeader struct {
	Magic uint32
	LogID uint32 // log id

	Sequence  uint32 // request/response sequence
	ServiceID uint32 // service id

	Command uint32 // operation command code
	Code    int32  // error code

	Len uint16 // body length
	_   uint16
	_   int32 // reserved, maybe used as package md5 checksum
}

type EchoPackage struct {
	H EchoPkgHeader		//Unmarshal 和 Marshal方法
	B string
}
```

####	echo-server
从[`initServer`](https://github.com/AlexStocks/getty/blob/master/examples/echo/tcp-echo/server/app/server.go#L126)切入，最核心的设置部分为`server.RunEventLoop(newSession)`，其中最重要的两个方法：

-	`session.SetPkgHandler(echoPkgHandler)`
-	`session.SetEventListener(echoMsgHandler)`

```GOLANG
func newSession(session getty.Session) error {
	var (
		ok      bool
		tcpConn *net.TCPConn
	)

	if conf.GettySessionParam.CompressEncoding {
		session.SetCompressType(getty.CompressZip)
	}

	if tcpConn, ok = session.Conn().(*net.TCPConn); !ok {
		panic("bad connection type")
	}

	tcpConn.SetNoDelay(conf.GettySessionParam.TcpNoDelay)
	tcpConn.SetKeepAlive(conf.GettySessionParam.TcpKeepAlive)
	if conf.GettySessionParam.TcpKeepAlive {
		tcpConn.SetKeepAlivePeriod(conf.GettySessionParam.keepAlivePeriod)
	}
	tcpConn.SetReadBuffer(conf.GettySessionParam.TcpRBufSize)
	tcpConn.SetWriteBuffer(conf.GettySessionParam.TcpWBufSize)

	session.SetName(conf.GettySessionParam.SessionName)
	session.SetMaxMsgLen(conf.GettySessionParam.MaxMsgLen)
	session.SetPkgHandler(echoPkgHandler)				//设置pkg处理方法
	session.SetEventListener(echoMsgHandler)			//设置事件钩子
	session.SetReadTimeout(conf.GettySessionParam.tcpReadTimeout)
	session.SetWriteTimeout(conf.GettySessionParam.tcpWriteTimeout)
	session.SetCronPeriod((int)(conf.sessionTimeout.Nanoseconds() / 1e6))
	session.SetWaitTime(conf.GettySessionParam.waitTimeout)
	// session.SetTaskPool(taskPool)

	return nil
}

func initServer() {
	var (
		addr     string
		portList []string
		server   getty.Server
	)

	portList = conf.Ports
	if len(portList) == 0 {
		panic("portList is nil")
	}
	for _, port := range portList {
		addr = gxnet.HostAddress2(conf.Host, port)
		serverOpts := []getty.ServerOption{getty.WithLocalAddress(addr)}
		serverOpts = append(serverOpts, getty.WithServerTaskPool(taskPool))
		server = getty.NewTCPServer(serverOpts...)
		// run server
		server.RunEventLoop(newSession)		//核心入口
		log.Debug("server bind addr ok")
		serverList = append(serverList, server)
	}
}
```

1、`session.SetPkgHandler(echoPkgHandler)`

`echoPkgHandler`是[`EchoPackageHandler`](https://github.com/AlexStocks/getty/blob/master/examples/echo/tcp-echo/server/app/readwriter.go)，实现了`Read`/`Write`方法，从例子看，需要用户自行实现序列化`Read`以及反序列化`Write` 这两个方法

```GO
func (h *EchoPackageHandler) Read(ss getty.Session, data []byte) (interface{}, int, error) {
	var (
		err error
		len int
		pkg EchoPackage		//用户需要自定义的协议
		buf *bytes.Buffer
	)

	buf = bytes.NewBuffer(data)
	len, err = pkg.Unmarshal(buf)
	if err != nil {
		if err == ErrNotEnoughStream {
			return nil, 0, nil
		}

		return nil, 0, err
	}

	return &pkg, len, nil
}

func (h *EchoPackageHandler) Write(ss getty.Session, pkg interface{}) ([]byte, error) {
	var (
		ok        bool
		err       error
		startTime time.Time
		echoPkg   *EchoPackage
		buf       *bytes.Buffer
	)

	startTime = time.Now()
	if echoPkg, ok = pkg.(*EchoPackage); !ok {
		//bad package
		return nil, errors.New("invalid echo package!")
	}

	buf, err = echoPkg.Marshal()
	if err != nil {
		return nil, err
	}

	log.Debug("WriteEchoPkgTimeMs = %s", time.Since(startTime).String())

	return buf.Bytes(), nil
}
```

2、`session.SetEventListener(echoMsgHandler)`

感觉这个是比较核心的设置，主要是设置一次会话Session的

```golang
// EventListener is used to process pkg that received from remote session
type EventListener interface {
	// OnOpen invoked when session opened
	// If the return error is not nil, @Session will be closed.
	OnOpen(Session) error

	// OnClose invoked when session closed.
	OnClose(Session)

	// OnError invoked when got error.
	OnError(Session, error)

	// OnCron invoked periodically, its period can be set by (Session)SetCronPeriod
	OnCron(Session)

	// OnMessage invoked when getty received a package. Pls attention that do not handle long time
	// logic processing in this func. You'd better set the package's maximum length.
	// If the message's length is greater than it, u should should return err in
	// Reader{Read} and getty will close this connection soon.
	//
	// If ur logic processing in this func will take a long time, u should start a goroutine
	// pool(like working thread pool in cpp) to handle the processing asynchronously. Or u
	// can do the logic processing in other asynchronous way.
	// !!!In short, ur OnMessage callback func should return asap.
	//
	// If this is a udp event listener, the second parameter type is UDPContext.
	OnMessage(Session, interface{})
}
```

现在看`echo-server`的实现，首先是`EchoMessageHandler`的定义：

```GO
type EchoMessageHandler struct {
	handlers map[uint32]PackageHandler		//定义自定义协议命令字的处理方法

	rwlock     sync.RWMutex
	sessionMap map[getty.Session]*clientEchoSession	
}

func newEchoMessageHandler() *EchoMessageHandler {
	handlers := make(map[uint32]PackageHandler)
	handlers[heartbeatCmd] = hbHandler		//处理方法1
	handlers[echoCmd] = msgHandler			//处理方法2

	return &EchoMessageHandler{sessionMap: make(map[getty.Session]*clientEchoSession), handlers: handlers}
}
```

再看各个方法的实现：
```go
func (h *EchoMessageHandler) OnOpen(session getty.Session) error {
	var err error

	h.rwlock.RLock()
	if conf.SessionNumber <= len(h.sessionMap) {
		err = errTooManySessions
	}
	h.rwlock.RUnlock()
	if err != nil {
		return err
	}

	log.Info("got session:%s", session.Stat())
	h.rwlock.Lock()
	h.sessionMap[session] = &clientEchoSession{session: session}
	h.rwlock.Unlock()
	return nil
}

func (h *EchoMessageHandler) OnError(session getty.Session, err error) {
	log.Info("session %s got error %v, will be closed.", session.Stat(), err)
	h.rwlock.Lock()
	delete(h.sessionMap, session)
	h.rwlock.Unlock()
}

func (h *EchoMessageHandler) OnClose(session getty.Session) {
	log.Info("session %s is closing......", session.Stat())
	h.rwlock.Lock()
	delete(h.sessionMap, session)
	h.rwlock.Unlock()
}

func (h *EchoMessageHandler) OnMessage(session getty.Session, pkg interface{}) {
	p, ok := pkg.(*EchoPackage)
	if !ok {
		//illegal package
		return
	}

	handler, ok := h.handlers[p.H.Command]
	if !ok {
		log.Error("illegal command %d", p.H.Command)
		return
	}
	err := handler.Handle(session, p)
	if err != nil {
		h.rwlock.Lock()
		if _, ok := h.sessionMap[session]; ok {
			h.sessionMap[session].reqNum++
		}
		h.rwlock.Unlock()
	}
}

func (h *EchoMessageHandler) OnCron(session getty.Session) {
	var (
		flag   bool
		active time.Time
	)
	h.rwlock.RLock()
	if _, ok := h.sessionMap[session]; ok {
		active = session.GetActive()
		if conf.sessionTimeout.Nanoseconds() < time.Since(active).Nanoseconds() {
			flag = true
			log.Warn("session %s timeout %s, reqNum %d",
				session.Stat(), time.Since(active).String(), h.sessionMap[session].reqNum)
		}
	}
	h.rwlock.RUnlock()
	if flag {
		h.rwlock.Lock()
		delete(h.sessionMap, session)
		h.rwlock.Unlock()
		session.Close()
	}
}
```

####	echo-client



##  0x0	参考
- [Go 语言网络库 getty 的那些事](https://developer.aliyun.com/article/791609)
- [example](https://github.com/xfxdev/xtcp/blob/master/_example/main.go)
- [getty 开发日志](https://alexstocks.github.io/html/getty.html)
- [Go 网络库 getty 的那些事](https://cloud.tencent.com/developer/article/1877651)
- [dubbo-go 的开发、设计与功能介绍](https://dubbogo.github.io/zh-cn/docs/md/arch/dubbo-go-design-implement-and-featrues.html)