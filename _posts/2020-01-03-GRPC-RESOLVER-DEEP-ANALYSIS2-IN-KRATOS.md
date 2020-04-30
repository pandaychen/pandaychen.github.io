---
layout:     post
title:      gRPC 应用篇之 Resolver 接口封装
subtitle:   如何封装 gRPC 的 Resolver（Kratos）
date:       2020-01-01
author:     pandaychen
header-img:
catalog: true
tags:
	- gRPC
	- 负载均衡
---

> 本篇文章分析下，Kratos 中如何封装 gRPC 中服务发现的接口，已经在项目中是如何使用的。

## 0x00	再看 gRPC 的 Resolver（解析器）接口

###	1.1 Resolver 暴露的三个接口

前文说过，gRPC 内置的服务治理功能，对开发者暴露了服务发现的 `interface{}`，`resolver.Builder` 和 `resolver.ClientConn` 和 `resolver.Resolver`，[相关代码](https://github.com/grpc/grpc-go/blob/master/resolver/resolver.go)。开发者在实例化这三个接口之后，就可以实现从指定的 scheme 中获取服务列表，通知 balancer 并与这些服务端建立 RPC 长连接。

1.  resolver.Builder
2.	resolver.ClientConn
3.	resolver.Resolver

-	resolver.Builder
`Builder` 用于 gRPC 内部创建 `Resolver` 接口的实现，但注意内部声明的 `Build()` 方法将接口 `ClientConn` 作为参数传入了，在前文的分析中，我们了解到 `ClientConn`[结库](https://github.com/grpc/grpc-go/blob/master/clientconn.go#L459) 是非常重要的结构，其成员 `conns map[*addrConn]struct{}` 中维护了所有从注册中心获取到的服务端列表。
```go
// Builder creates a resolver that will be used to watch name resolution updates.
type Builder interface {
	// Build creates a new resolver for the given target.
	//
	// gRPC dial calls Build synchronously, and fails if the returned error is
	// not nil.
	Build(target Target, cc ClientConn, opts BuildOption) (Resolver, error)
	// Scheme returns the scheme supported by this resolver.
	// Scheme is defined at https://github.com/grpc/grpc/blob/master/doc/naming.md.
	Scheme() string
}
```
-	[resolver.ClientConn](https://github.com/grpc/grpc-go/blob/master/clientconn.go#L459)
`ClientConn` 接口中，`UpdateState` 方法需要传入 `State` 结构，`NewAddress` 方法需要传入 `Address` 结构，看代码可以发现其中包含了 `Addresses []Address // Resolved addresses for the target`，可以看出是需要将服务发现得到的 `Address` 对象列表告诉 `ClientConn` 的对象。
```go
// ClientConn contains the callbacks for resolver to notify any updates
// to the gRPC ClientConn.
//
// This interface is to be implemented by gRPC. Users should not need a
// brand new implementation of this interface. For the situations like
// testing, the new implementation should embed this interface. This allows
// gRPC to add new methods to this interface.
type ClientConn interface {
	// UpdateState updates the state of the ClientConn appropriately.
	UpdateState(State)
	// NewAddress is called by resolver to notify ClientConn a new list
	// of resolved addresses.
	// The address list should be the complete list of resolved addresses.
	//
	// Deprecated: Use UpdateState instead.
	NewAddress(addresses []Address)
	// NewServiceConfig is called by resolver to notify ClientConn a new
	// service config. The service config should be provided as a json string.
	//
	// Deprecated: Use UpdateState instead.
	NewServiceConfig(serviceConfig string)
}
```
-	resolver.Resolver
`Resolver` 提供了 `ResolveNow` 用于被 gRPC 尝试重新进行服务发现
```go
// Resolver watches for the updates on the specified target.
// Updates include address updates and service config updates.
type Resolver interface {
	// ResolveNow will be called by gRPC to try to resolve the target name
	// again. It's just a hint, resolver can ignore this if it's not necessary.
	//
	// It could be called multiple times concurrently.
	ResolveNow(ResolveNowOption)
	// Close closes the resolver.
	Close()
}
```

###	1.2 梳理 Resolver 过程
通过这三个接口，再次梳理下 gRPC 的服务发现实现逻辑

1.	通过 `Builder.Build()` 进行 `Reslover` 的创建，在 `Build()` 的过程中将服务发现的地址信息丢给 `ClientConn` 用于内部连接创建（通过 `ClientConn.UpdateState()` 实现）等逻辑；

2.	当 `client` 在 `Dial` 时会根据 `target` 解析的 `scheme` 获取对应的 `Builder`，[代码位置](https://github.com/grpc/grpc-go/blob/master/clientconn.go#L242)
```go
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
	...
	...
	// Determine the resolver to use.
	cc.parsedTarget = parseTarget(cc.target)
	grpclog.Infof("parsed scheme: %q", cc.parsedTarget.Scheme)
	resolverBuilder := cc.getResolver(cc.parsedTarget.Scheme)		// 通过 scheme(名字) 获取对应的 resolver
	if resolverBuilder == nil {
		// If resolver builder is still nil, the parsed target's scheme is
		// not registered. Fallback to default resolver and set Endpoint to
		// the original target.
		grpclog.Infof("scheme %q not registered, fallback to default scheme", cc.parsedTarget.Scheme)
		cc.parsedTarget = resolver.Target{
			Scheme:   resolver.GetDefaultScheme(),
			Endpoint: target,
		}
		resolverBuilder = cc.getResolver(cc.parsedTarget.Scheme)
		if resolverBuilder == nil {
			return nil, fmt.Errorf("could not get resolver for default scheme: %q", cc.parsedTarget.Scheme)
		}
	}
	...
	...
	// Build the resolver.
	rWrapper, err := newCCResolverWrapper(cc, resolverBuilder)		// 通过 gRPC 提供的 Wrapper，应用我们实现的 resolver 逻辑
	if err != nil {
		return nil, fmt.Errorf("failed to build resolver: %v", err)
	}
	cc.mu.Lock()
	cc.resolverWrapper = rWrapper
	cc.mu.Unlock()
	...
	...
}
```

3.	当 `Dial` 成功会创建出结构体 `ClientConn` 的对象 [官方代码位置](https://github.com/grpc/grpc-go/blob/master/clientconn.go#L447)(注意不是上面的 `ClientConn` 接口)，可以看到结构体 `ClientConn` 内的成员 `resolverWrapper` 又实现了接口 `ClientConn` 的方法 [官方代码位置](https://github.com/grpc/grpc-go/blob/master/resolver_conn_wrapper.go)
```go
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
	// 初始化 CC
	cc := &ClientConn{
		target:            target,
		csMgr:             &connectivityStateManager{},
		conns:             make(map[*addrConn]struct{}),
		dopts:             defaultDialOptions(),
		blockingpicker:    newPickerWrapper(),
		czData:            new(channelzData),
		firstResolveEvent: grpcsync.NewEvent(),
	}
	...
	...

	...
	...
	return cc, nil
}
```
4.	当 `resolverWrapper` 被初始化时就会调用 `Build` 方法 [官方代码位置](https://github.com/grpc/grpc-go/blob/master/resolver_conn_wrapper.go#L89)，其中参数为接口 `ClientConn` 传入的是 `ccResolverWrapper`
```go
// newCCResolverWrapper uses the resolver.Builder to build a Resolver and
// returns a ccResolverWrapper object which wraps the newly built resolver.
func newCCResolverWrapper(cc *ClientConn, rb resolver.Builder) (*ccResolverWrapper, error) {
	ccr := &ccResolverWrapper{
		cc:   cc,
		done: grpcsync.NewEvent(),
	}

	var credsClone credentials.TransportCredentials
	if creds := cc.dopts.copts.TransportCredentials; creds != nil {
		credsClone = creds.Clone()
	}
	rbo := resolver.BuildOptions{
		DisableServiceConfig: cc.dopts.disableServiceConfig,
		DialCreds:            credsClone,
		CredsBundle:          cc.dopts.copts.CredsBundle,
		Dialer:               cc.dopts.copts.Dialer,
	}

	var err error
	// We need to hold the lock here while we assign to the ccr.resolver field
	// to guard against a data race caused by the following code path,
	// rb.Build-->ccr.ReportError-->ccr.poll-->ccr.resolveNow, would end up
	// accessing ccr.resolver which is being assigned here.
	ccr.resolverMu.Lock()
	defer ccr.resolverMu.Unlock()
	ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, rbo)
	if err != nil {
		return nil, err
	}
	return ccr, nil
}
```
5.	当用户基于 `Builder` 的实现进行 `UpdateState` 调用时，则会触发结构体 `ClientConn` 的 `updateResolverState` 方法 [官方代码位置](https://github.com/grpc/grpc-go/blob/master/resolver_conn_wrapper.go#L109)，`updateResolverState` 则会对传入的 `Address` 进行初始化等逻辑 [官方代码位置](https://github.com/grpc/grpc-go/blob/master/clientconn.go#L553)
```go
func (cc *ClientConn) updateResolverState(s resolver.State, err error) error {
	defer cc.firstResolveEvent.Fire()
	cc.mu.Lock()
	// Check if the ClientConn is already closed. Some fields (e.g.
	// balancerWrapper) are set to nil when closing the ClientConn, and could
	// cause nil pointer panic if we don't have this check.
	if cc.conns == nil {
		cc.mu.Unlock()
		return nil
	}

	if err != nil {
		// May need to apply the initial service config in case the resolver
		// doesn't support service configs, or doesn't provide a service config
		// with the new addresses.
		cc.maybeApplyDefaultServiceConfig(nil)

		if cc.balancerWrapper != nil {
			cc.balancerWrapper.resolverError(err)
		}

		// No addresses are valid with err set; return early.
		cc.mu.Unlock()
		return balancer.ErrBadResolverState
	}

	var ret error
	if cc.dopts.disableServiceConfig || s.ServiceConfig == nil {
		cc.maybeApplyDefaultServiceConfig(s.Addresses)
		// TODO: do we need to apply a failing LB policy if there is no
		// default, per the error handling design?
	} else {
		if sc, ok := s.ServiceConfig.Config.(*ServiceConfig); s.ServiceConfig.Err == nil && ok {
			cc.applyServiceConfigAndBalancer(sc, s.Addresses)
		} else {
			ret = balancer.ErrBadResolverState
			if cc.balancerWrapper == nil {
				var err error
				if s.ServiceConfig.Err != nil {
					err = status.Errorf(codes.Unavailable, "error parsing service config: %v", s.ServiceConfig.Err)
				} else {
					err = status.Errorf(codes.Unavailable, "illegal service config type: %T", s.ServiceConfig.Config)
				}
				cc.blockingpicker.updatePicker(base.NewErrPicker(err))
				cc.csMgr.updateState(connectivity.TransientFailure)
				cc.mu.Unlock()
				return ret
			}
		}
	}

	var balCfg serviceconfig.LoadBalancingConfig
	if cc.dopts.balancerBuilder == nil && cc.sc != nil && cc.sc.lbConfig != nil {
		balCfg = cc.sc.lbConfig.cfg
	}

	cbn := cc.curBalancerName
	bw := cc.balancerWrapper
	cc.mu.Unlock()
	if cbn != grpclbName {
		// Filter any grpclb addresses since we don't have the grpclb balancer.
		for i := 0; i <len(s.Addresses); {
			if s.Addresses[i].Type == resolver.GRPCLB {
				copy(s.Addresses[i:], s.Addresses[i+1:])
				s.Addresses = s.Addresses[:len(s.Addresses)-1]
				continue
			}
			i++
		}
	}
	uccsErr := bw.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
	if ret == nil {
		ret = uccsErr // prefer ErrBadResolver state since any other error is
		// currently meaningless to the caller.
	}
	return ret
}
```

6.	至此整个服务发现过程就结束了。从中也可以看出 gRPC 官方提供的三个接口还是很灵活的，但也正因为灵活要实现稍微麻烦一些，而 `Address`[官方代码位置](https://github.com/grpc/grpc-go/blob/master/resolver/resolver.go#L79) 如果直接被业务拿来用于服务节点信息的描述结构则显得有些过于简单。

所以 `warden` 包装了 gRPC 的整个服务发现实现逻辑，代码分别位于 `pkg/naming/naming.go` 和 `warden/resolver/resolver.go`，其中：

* `naming.go` 内定义了用于描述业务实例的 `Instance` 结构、用于服务注册的 `Registry` 接口、用于服务发现的 `Resolver` 接口
* `resolver.go` 内实现了 gRPC 官方的 `resolver.Builder` 和 `resolver.Resolver` 接口，但也暴露了 `naming.go` 内的 `naming.Builder` 和 `naming.Resolver` 接口

## 0x01	gRPC 嵌入 Register 实现
（未完待续）

##	0x02	kratos 对 Resolver 的二次封装
（未完待续）

##	0x03	参考
-	[package resolver](https://godoc.org/google.golang.org/grpc/resolver)
-	[Kratos-warden-resolver](https://github.com/bilibili/kratos/blob/master/doc/wiki-cn/warden-resolver.md)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权