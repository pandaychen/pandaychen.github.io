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

### Metadata实战场景
例如，实现每次RPC调用后，都采集服务端的系统（如CPU、内存）数据，如下代码所示：
```golang
func (s *Server) stats() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		resp, err = handler(ctx, req)
		//采集系统数据
		if cpustat.Usage != 0 {
			//每次客户端RPC请求，服务端都会计算cpu使用率，gRPC客户端在Pick的DoneInfo中获取此值
			trailer := gmd.Pairs([]string{nmd.CPUUsage, strconv.FormatInt(int64(cpustat.Usage), 10)}...)
			//每次rpc请求时，放在tailer，上报至discovery
			grpc.SetTrailer(ctx, trailer)
		}
		return
	}
}
```

##  0x02    

##	0x04	参考