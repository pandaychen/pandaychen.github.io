---
layout:     post
title:      GoIM 源码分析（一）：Comet
subtitle:   分析 GoIM 对外服务模块 Comet
date:       2020-08-01
author:     pandaychen
catalog:    true
tags:
    - GOIM
---


##  0x00	前言
本篇文章，分析下 GoIM 的 Comet 模块。
1.	Comet 主方法
2.	Comet 主要数据结构
3.	Comet 管理服务（与 Job 模块通信）

Comet模块的位置：
![COMET](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/goim/goim2-arch.png)

##	0x01	Comet 模块
Comet 模块为用户代理 Server（主要提供对外服务），用于客户端的连接，根据情况可部署多个 Comet-Server（扩展）。Comet 模块支持 Tcp, Http, WebSocket, TLS WebSocket 等多种服务。

##  0x02   主逻辑入口
Comet 的入口 [main.go](https://github.com/Terry-Mao/goim/blob/master/cmd/comet/main.go)，主要完成初始化及服务器的启动工作及注册：
这里提几点可借鉴的经验：
1.  使用随机数必须运行 `rand.Seed(time.Now().UTC().UnixNano())`，一般放在初始化时
2.  使用 `runtime.GOMAXPROCS(runtime.NumCPU())` 设置合适的 core 数是个好习惯，K8S 环境常用 `automaxprocess` 设置
3.  启动了 N 个监听端口（配置文件）
    -   对外端口有：`InitTCP`，`InitWebsocket`
    -   对内端口（管理）：`grpc.New(conf.Conf.RPCServer, srv)`
4.  服务的注册使用 `register` 完成，用于客户端服务发现
5.  Server 端优雅退出
6.  Comet 维护了一个 [白名单](https://github.com/Terry-Mao/goim/blob/master/internal/comet/whitelist.go) 机制

```golang
func main() {
    ...
	rand.Seed(time.Now().UTC().UnixNano())
	runtime.GOMAXPROCS(runtime.NumCPU())
	println(conf.Conf.Debug)
	log.Infof("goim-comet [version: %s env: %+v] start", ver, conf.Conf.Env)
	// register discovery
	dis := naming.New(conf.Conf.Discovery)
	resolver.Register(dis)
	// new comet server
	srv := comet.NewServer(conf.Conf)
	if err := comet.InitWhitelist(conf.Conf.Whitelist); err != nil {
		panic(err)
	}
	if err := comet.InitTCP(srv, conf.Conf.TCP.Bind, runtime.NumCPU()); err != nil {
		panic(err)
	}
	if err := comet.InitWebsocket(srv, conf.Conf.Websocket.Bind, runtime.NumCPU()); err != nil {
		panic(err)
	}
	if conf.Conf.Websocket.TLSOpen {
		if err := comet.InitWebsocketWithTLS(srv, conf.Conf.Websocket.TLSBind, conf.Conf.Websocket.CertFile, conf.Conf.Websocket.PrivateFile, runtime.NumCPU()); err != nil {
			panic(err)
		}
	}
	// new grpc server
	rpcSrv := grpc.New(conf.Conf.RPCServer, srv)
	cancel := register(dis, srv)
	// signal
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT)
	for {
		s := <-c
		log.Infof("goim-comet get a signal %s", s.String())
		switch s {
		case syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT:
			if cancel != nil {
				cancel()
			}
			rpcSrv.GracefulStop()
			srv.Close()
			log.Infof("goim-comet [version: %s] exit", ver)
			log.Flush()
			return
		case syscall.SIGHUP:
		default:
			return
		}
	}
}
```

##  0x03	Comet 结构
[comet 结构定义](https://github.com/Terry-Mao/goim/tree/master/internal/comet)，Comet 对应的结构是项目中最复杂的，下面先简单列举下：

[`Server`](https://github.com/Terry-Mao/goim/blob/master/internal/comet/server.go#L54)：
```golang
// Server is comet server.
type Server struct {
	c         *conf.Config
	round     *Round    // accept round store
	buckets   []*Bucket // subkey bucket
	bucketIdx uint32

	serverID  string
	rpcClient logic.LogicClient
}
```

[`Round`](https://github.com/Terry-Mao/goim/blob/master/internal/comet/round.go#L21)：
```golang
// Round userd for connection round-robin get a reader/writer/timer for split big lock.
type Round struct {
	readers []bytes.Pool
	writers []bytes.Pool
	timers  []time.Timer
	options RoundOptions
}
```

[`Bucket`](https://github.com/Terry-Mao/goim/blob/master/internal/comet/bucket.go#L12)：
```golang
// Bucket is a channel holder.
type Bucket struct {
	c     *conf.Bucket
	cLock sync.RWMutex        // protect the channels for chs
	chs   map[string]*Channel // map sub key to a channel
	// room
	rooms       map[string]*Room // bucket room channels
	routines    []chan *grpc.BroadcastRoomReq
	routinesNum uint64

	ipCnts map[string]int32
}
```


##  参考
-	[消息推送架构-Based-GOIM](https://yeqown.github.io/2020/04/02/%E6%B6%88%E6%81%AF%E6%8E%A8%E9%80%81%E6%9E%B6%E6%9E%84-based-GOIM/)
-	[goim 架构与定制](https://tsingson.github.io/tech/goim-go-01/)
-	[goim 及分布式工程解构](https://github.com/talkgo/night/issues/363)
-	[goim via nats](https://github.com/tsingson/ex-goim/releases)
-	[goim源码剖析](https://laohanlinux.github.io/2016/12/22/goim%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/)
-	[goim v2.0](https://github.com/Terry-Mao/goim/blob/master/README_cn.md)
-	[分享 直播 -- 弹幕系统简介](https://ruby-china.org/topics/39574)
-	[对于router服务存在的疑问](https://github.com/Terry-Mao/goim/issues/33)
-	[高并发实时弹幕系统的实战之路](https://zhuanlan.zhihu.com/p/22016939)
-	[goim 中的 data flow 数据流转及优化思考](https://tsingson.github.io/tech/goim-go-04/)