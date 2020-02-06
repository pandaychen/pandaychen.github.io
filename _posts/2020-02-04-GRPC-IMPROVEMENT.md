---
layout:     post
title:      gRPC应用篇之自带组件
subtitle:   
date:       2020-02-04
author:     pandaychen
header-img: 
catalog: true
category:   false
tags:
    - gRPC
---

##	0x01	Metadata应用
gRPC的[Metadata机制](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md)提供了一种类似HTTP-Header的机制（理解为RPC方法的Header）。它的应用场景是：对于每一次的RPC调用中，都可能会有一些有用的数据，而这些数据就可以通过metadata来传递。metadata是以key-value的形式存储数据的，其中key是`string`类型，而value是`[]string`类型。metadata使得client和server能够为对方提供关于本次调用的一些信息，就像一次http请求的RequestHeader和ResponseHeader一样。http中header的生命周周期是一次http请求，那么metadata的生命周期就是一次RPC调用。

#### 实战
例如，实现每次RPC调用后，都采集服务端的系统（如CPU、内存）数据，如下代码所示：
```golang
func (s *Server) stats() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		resp, err = handler(ctx, req)
		//采集系统数据
		if cpustat.Usage != 0 {
			//每次客户端RPC请求，服务端都会计算cpu使用率，gRPC客户端在Pick的DoneInfo中获取此值
			trailer := metadata.Pairs([]string{nmd.CPUUsage, strconv.FormatInt(int64(cpustat.Usage), 10)}...)
			//每次rpc请求时，放在tailer，上报至discovery
			grpc.SetTrailer(ctx, trailer)
		}
		return
	}
}
```

##  0x02    拦截器链（Interceptor Chain）的封装
以gRPC的服务端拦截器实现为例，封装一个实用的拦截器链。该链满足如下特性：定义一个链，`ChainUnaryServer(one, two, three)`按照`one`、`two`、`three`的顺序执行。

#### 封装Server的结构
```golang
//grpc-server核心结构
type Server struct {
    ...
	server   *grpc.Server	//
	handlers []grpc.UnaryServerInterceptor	//封装拦截器数组
}
```

#### 修改grpc.UnaryInterceptor传入的参数UnaryServerInterceptor，完成链式功能
先看下`grpc.UnaryInterceptor`这个方法的[定义](https://godoc.org/google.golang.org/grpc#UnaryInterceptor)，该方法的唯一参数是`grpc.UnaryServerInterceptor`
```GOLANG
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption
```
而`grpc.UnaryServerInterceptor`的[定义](https://godoc.org/google.golang.org/grpc#UnaryServerInterceptor)是一个方法原型，这是我们需要实现的（链式interceptor）
```GOLANG
type UnaryServerInterceptor 
func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```
实现链式拦截器的功能：
```GOLANG
// 拦截器链实现
func (s *Server) interceptor(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	var (
		i     int
		chain grpc.UnaryHandler
	)

	n := len(s.handlers)
	if n == 0 {
		return handler(ctx, req)
	}

	chain = func(ic context.Context, ir interface{}) (interface{}, error) {
		if i == n-1 {
			return handler(ic, ir)
		}
		i++
		return s.handlers[i](ic, ir, args, chain)
	}

	return s.handlers[0](ctx, req, args, chain)
}
```

####   完成grpc.UnaryInterceptor初始化
接下来，将上一步实现的`UnaryServerInterceptor`，当做`grpc.UnaryInterceptor()`方法的参数传入，这样，就使用我们自定义的链式拦截器了。
```GOLANG
// NewServer returns a new blank Server instance with a default server interceptor.
// 初始化Server
func NewServer(conf *ServerConfig, opt ...grpc.ServerOption) (s *Server) {
	if conf == nil {
		if !flag.Parsed() {
			fmt.Fprint(os.Stderr, "[warden] please call flag.Parse() before Init warden server, some configure may not effect\n")
		}
		conf = parseDSN(_grpcDSN)
	} else {
		fmt.Fprintf(os.Stderr, "[warden] config is Deprecated, argument will be ignored. please use -grpc flag or GRPC env to configure warden server.\n")
	}
	s = new(Server)
	if err := s.SetConfig(conf); err != nil {
		panic(errors.Errorf("warden: set config failed!err: %s", err.Error()))
	}
	keepParam := grpc.KeepaliveParams(keepalive.ServerParameters{
		MaxConnectionIdle:     time.Duration(s.conf.IdleTimeout),
		MaxConnectionAgeGrace: time.Duration(s.conf.ForceCloseWait),
		Time:                  time.Duration(s.conf.KeepAliveInterval),
		Timeout:               time.Duration(s.conf.KeepAliveTimeout),
		MaxConnectionAge:      time.Duration(s.conf.MaxLifeTime),
	})
	opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))  //
	s.server = grpc.NewServer(opt...)
	s.Use(s.recovery(), s.handle(), serverLogging(conf.LogFlag), s.stats(), s.validate())
	s.Use(ratelimiter.New(nil).Limit())
	return
}
```

####	定义向链式拦截器数组添加拦截器实现的方法Use
// Use attachs a global inteceptor to the server.
// For example, this is the right place for a rate limiter or error management inteceptor.

//设置一个全局的拦截器，其实就是修改s自带的拦截器数组
func (s *Server) Use(handlers ...grpc.UnaryServerInterceptor) *Server {
	finalSize := len(s.handlers) + len(handlers)
	if finalSize >= int(_abortIndex) {
		panic("warden: server use too many handlers")
	}
	mergedHandlers := make([]grpc.UnaryServerInterceptor, finalSize)
	copy(mergedHandlers, s.handlers)
	copy(mergedHandlers[len(s.handlers):], handlers)
	s.handlers = mergedHandlers		//更新自带的handlers，数组
	return s
}

####	使用拦截器数组

```GOLANG
// NewServer returns a new blank Server instance with a default server interceptor.
// 初始化Server
func NewServer(conf *ServerConfig, opt ...grpc.ServerOption) (s *Server) {
	...
	s = new(Server)
	if err := s.SetConfig(conf); err != nil {
		panic(errors.Errorf("warden: set config failed!err: %s", err.Error()))
	}
	keepParam := grpc.KeepaliveParams(keepalive.ServerParameters{
		MaxConnectionIdle:     time.Duration(s.conf.IdleTimeout),
		MaxConnectionAgeGrace: time.Duration(s.conf.ForceCloseWait),
		Time:                  time.Duration(s.conf.KeepAliveInterval),
		Timeout:               time.Duration(s.conf.KeepAliveTimeout),
		MaxConnectionAge:      time.Duration(s.conf.MaxLifeTime),
	})
	opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))  //
	s.server = grpc.NewServer(opt...)
	s.Use(s.recovery(), s.handle(), serverLogging(conf.LogFlag), s.stats(), s.validate())
	s.Use(ratelimiter.New(nil).Limit())
	...
	return
}
```


##	0x04	参考