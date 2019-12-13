---
layout:     post
title:      gRPC源码分析之Balancer篇
subtitle:   gRPC客户端平衡器分析
date:       2019-11-22
author:     pandaychen
header-img: 
catalog: true
tags:
    - gRPC
    - Loadbalance
---

##  前言
本篇文章详细分析下[gRPC-Banlancer](https://github.com/grpc/grpc-go/tree/master/balancer)的实现。<br>
首先，提几个问题，带着这几个问题，再看源码会收货更多：
-   resolver获取到的后端列表，在哪里进行初始化连接的？
-   resolver和balancer的交互是怎样的？
-   balancer提供给用户的接口在哪里生效？
-   picker如何实现？

##  0x00	gRPC的平衡器Balancer
首先，再看这张gRPC客户端负载均衡实现的官方架构图：
![image](https://raw.githubusercontent.com/grpc/proposal/master/L9_graphics/bar_after.png)
从图中，可以看到Balancer平衡器位于架构的最右方，内置一个Picker模块，Balancer主要完成下面这几个功能：

-   与Resolver通信（维持通信机制），接收Resolver通知的服务端列表更新，维护Connection Pool及每个连接的状态
-   对上一步获取的服务端列表，调用`newSubConn`异步建立长连接（每个后端一个长连接），同时，监控连接的状态，及时更新Connection Pool
-	创建Picker，Picker执行的算法就是真正的LB逻辑，当客户端使用`conn`初始化RPC方法时，通过Picker选择一个存活的连接，返回给客户端，然后调用UpdatePicker更新LB算法的内置状态，为下一次调用做准备
-   Balancer是gRPC负载均衡最核心的模块

下面我们从Resovler更新列表开始，逐步揭开Balancer神秘的面纱。

##  0x01    Balancer和Resolver的交互

通过前面文章[gRPC源码分析之Resolver篇](https://pandaychen.github.io/2019/11/20/GRPC-RESOLVER-DEEP-ANALYSIS/)和[gRPC源码分析-DnsResovler篇](https://pandaychen.github.io/2019/07/11/GRPC-BALANCER-DNSRESOVLER-ANALYSIS/)这两篇的分析，我们了解到通过Resolver（名字服务解析或List-watch）得到服务端IP列表后，最终会调用`ClientConn.updateResolverState`接口，这个接口开始涉及Balancer的逻辑，为何需要由Resolver通知Balancer呢？因为（通过Resolver）获取到服务端IP列表后，需要根据负载均衡算法的初始化要求，对后端服务IP列表做一些初始化工作（如consistent-hash、设置初始化权重等）

### resolver.ClientConn
`dnsResolver.watcher`（或是自己实现的Resolver）在解析到IP地址以及获取服务配置后分别调用`ClientConn.NewAddress`和`ClientConn.NewServiceConfig`两个接口通知上层应用，`ccResolverWrapper`实现了ClientConn接口。

举例来说，在DnsResolver中，DnsResovler`-->`Balancer的调用链如下：
``` go
dnsResovler.watcher --> ccResolverWrapper.NewAddress --> ClientConn.updateResolverState 
-->  cc.balancerWrapper.updateClientConnState

dnsResovler.watcher --> ccResolverWrapper.NewServiceConfig --> ClientConn.updateResolverState
-->  cc.balancerWrapper.updateClientConnState
``` 
可以看到Resolver在解析地址以及后续监听到地址变化后，最终调用了`ClientConn.updateResolverState`更新状态，如上面最左边的流程所示。

### ClientConn.updateResolverState
分析`ClientConn.updateResolverState`：
```go
func (cc *ClientConn) updateResolverState(s resolver.State) error {
    cc.mu.Lock()
    defer cc.mu.Unlock()
    // Check if the ClientConn is already closed. Some fields (e.g.
    // balancerWrapper) are set to nil when closing the ClientConn, and could
    // cause nil pointer panic if we don't have this check.
    if cc.conns == nil {
        return nil
    }

    // 如果禁用服务配置或者服务配置为空，使用默认服务配置
    if cc.dopts.disableServiceConfig || s.ServiceConfig == nil {
        if cc.dopts.defaultServiceConfig != nil && cc.sc == nil {
            cc.applyServiceConfig(cc.dopts.defaultServiceConfig)
        }
    } else if sc, ok := s.ServiceConfig.(*ServiceConfig); ok {
        cc.applyServiceConfig(sc)
    }

    /*
    *1. 优先使用服务配置的负载均衡策略
    *2. 如果返回的解析地址中有一个是resolver.GRPCLB类型，使用grpclb
    */
    var balCfg serviceconfig.LoadBalancingConfig
    if cc.dopts.balancerBuilder == nil {
        // Only look at balancer types and switch balancer if balancer dial
        // option is not set.
        var newBalancerName string
        //优先使用服务配置的负载均衡策略
        if cc.sc != nil && cc.sc.lbConfig != nil {
            newBalancerName = cc.sc.lbConfig.name
            balCfg = cc.sc.lbConfig.cfg
        } else {
            //如果返回的解析地址中有一个是resolver.GRPCLB类型，使用grpclb
            var isGRPCLB bool
            for _, a := range s.Addresses {
                if a.Type == resolver.GRPCLB {
                    isGRPCLB = true
                    break
                }
            }
            if isGRPCLB {
                newBalancerName = grpclbName
            } else if cc.sc != nil && cc.sc.LB != nil {
                newBalancerName = *cc.sc.LB
            } else {
                newBalancerName = PickFirstBalancerName
            }
        }
        //切换负载均衡策略
        cc.switchBalancer(newBalancerName)
    } else if cc.balancerWrapper == nil {
        // Balancer dial option was set, and this is the first time handling
        // resolved addresses. Build a balancer with dopts.balancerBuilder.
        cc.curBalancerName = cc.dopts.balancerBuilder.Name()
        cc.balancerWrapper = newCCBalancerWrapper(cc, cc.dopts.balancerBuilder, cc.balancerBuildOpts)
    }
    //调用负载均衡策略更新连接状态
    cc.balancerWrapper.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
    return nil
}
```

####    grpc.addrConn结构与gRPC连接状态切换
一个Balancer需要维持与N个后端之间的连接，这里，`grpc.ClientConn` 就表示一个客户端实例与服务端之间的连接，主要包含如下数据结构：
```go
grpc.connectivityStateManager(grpc.ClientConn.csMgr)    //总体的连接状态
```
状态类型为`connectivity.State`，有如下几种状态：

-   Idle
-   Connecting
-   Ready
-   TransientFailure
-   Shutdown
`grpc.ClientConn` 包含了多个 `grpc.addrConn`（每个 `grpc.addrConn` 表示客户端到一个服务端的一条连接），每个 `grpc.addrConn` 也有自己的连接转态（关于 `grpc.addrConn.state` 的转态切换可参考文档：[gRPC Connectivity Semantics and API](http://yangxikun.com/golang/2019/10/19/golang-grpc-client-side-lb.html)）。简单来说，规则如下：
1.  当至少有一个 `grpc.addrConn.state = Ready`，则 `grpc.ClientConn.csMgr.state = Ready`
2.  当至少有一个 `grpc.addrConn.state = Connecting`，则 `grpc.ClientConn.csMgr.state = Connecting`
3.  否则 `grpc.ClientConn.csMgr.state = TransientFailure`
默认客户端与某一个服务端（IP`+`PORT）只会建立一条连接（HTTP/2），所有 RPC 调用都会复用这条连接。关于为何只建立一条连接可以看下这个 [Use multiple connections to avoid the server’s SETTINGS_MAX_CONCURRENT_STREAMS](https://github.com/grpc/grpc/issues/11704)

```go
// addrConn is a network connection to a given address.
type addrConn struct {
	ctx    context.Context
	cancel context.CancelFunc

	cc     *ClientConn
	dopts  dialOptions
	acbw   balancer.SubConn
	scopts balancer.NewSubConnOptions

	// transport is set when there's a viable transport (note: ac state may not be READY as LB channel
	// health checking may require server to report healthy to set ac to READY), and is reset
	// to nil when the current transport should no longer be used to create a stream (e.g. after GoAway
	// is received, transport is closed, ac has been torn down).
	transport transport.ClientTransport // The current transport.

	mu      sync.Mutex
	curAddr resolver.Address   // The current address.
	addrs   []resolver.Address // All addresses that the resolver resolved to.

    // Use updateConnectivityState for updating addrConn's connectivity state.
    // 单个连接的状态
	state connectivity.State

	backoffIdx   int // Needs to be stateful for resetConnectBackoff.
	resetBackoff chan struct{}

	channelzID int64 // channelz unique identification number.
	czData     *channelzData
}
```

##  0x02    Balancer主要流程分析

### Balancer的设计模式
gRPC提供了Balancer和V2Balancer，grpclb只实现了V2Balancer接口。gPRC接口的设计原则遵循：接口隔离`+`使用Builder和Wrapper设计模式

### newCCBalancerWrapper
balancer和resovler的设计模式比较类似，通过`newCCBalancerWrapper`创建`ccBalancerWrapper`，保存为`cc.balancerWrapper`
```go
func newCCBalancerWrapper(cc *ClientConn, b balancer.Builder, bopts balancer.BuildOptions) *ccBalancerWrapper {
    // 初始化
    ccb := &ccBalancerWrapper{
        cc:               cc,
        //subConn变更通道
        stateChangeQueue: newSCStateUpdateBuffer(),
        //ClientConn变更通道
        ccUpdateCh:       make(chan *balancer.ClientConnState, 1),
        done:             make(chan struct{}),
        subConns:         make(map[*acBalancerWrapper]struct{}),
    }
    // 进入监听模式
    go ccb.watcher()
    // 创建balancer
    ccb.balancer = b.Build(ccb, bopts)
    return ccb
}
```

### ClientConn.switchBalancer
在`updateResolverState`中`maybeApplyDefaultServiceConfig`-->`applyServiceConfigAndBalancer`调用`switchBalancer`方法，即在解析到服务IP和service config后会触发`switchBalancer`
```go
func (cc *ClientConn) switchBalancer(name string) {
    // 如果前后balancer一样
    if strings.EqualFold(cc.curBalancerName, name) {
        return
    }

    // 优先使用参数指定的balancerBuilder
    grpclog.Infof("ClientConn switching balancer to %q", name)
    if cc.dopts.balancerBuilder != nil {
        grpclog.Infoln("ignoring balancer switching: Balancer DialOption used instead")
        return
    }
    // 关闭原先的balancerWrapper
    if cc.balancerWrapper != nil {
        cc.balancerWrapper.close()
    }

    // 如果无法获取指定balancer默认使用PickFirst策略
    builder := balancer.Get(name)
    if builder == nil {
        grpclog.Infof("failed to get balancer builder for: %v, using pick_first instead", name)
        builder = newPickfirstBuilder()
    }

    // 创建ccBalancerWrapper
    cc.curBalancerName = builder.Name()
    cc.balancerWrapper = newCCBalancerWrapper(cc, builder, cc.balancerBuildOpts)
}
```

### ccBalancerWrapper.watcher
**这个watcher的作用是什么?**
在创建ccBalancerWrapper时，会启用独立的groutine执行watcher函数，因为只有watcher会调用balancer接口，所以balancer可以做到无锁更新。
```go
func (ccb *ccBalancerWrapper) watcher() {
    for {
        select {
        //由ccBalancerWrapper.handleSubConnStateChange触发
        case t := <-ccb.stateChangeQueue.get():
            ccb.stateChangeQueue.load()
            select {
            case <-ccb.done:
                ccb.balancer.Close()
                return
            default:
            }
            // 通过balancer.V2Balancer更新subConn状态
            if ub, ok := ccb.balancer.(balancer.V2Balancer); ok {
                ub.UpdateSubConnState(t.sc, balancer.SubConnState{ConnectivityState: t.state})
            } else {
                ccb.balancer.HandleSubConnStateChange(t.sc, t.state)
            }
        //由ccBalancerWrapper.updateClientConnState写入触发
        case s := <-ccb.ccUpdateCh:
            select {
            case <-ccb.done:
                ccb.balancer.Close()
                return
            default:
            }
            // 通过balancer.V2Balancer更新ClientConn状态（）
            if ub, ok := ccb.balancer.(balancer.V2Balancer); ok {
                ub.UpdateClientConnState(*s)
            } else {
                ccb.balancer.HandleResolvedAddrs(s.ResolverState.Addresses, nil)
            }
        case <-ccb.done:
        }

        select {
        case <-ccb.done:
            ccb.balancer.Close()
            ccb.mu.Lock()
            scs := ccb.subConns
            ccb.subConns = nil
            ccb.mu.Unlock()
            for acbw := range scs {
                ccb.cc.removeAddrConn(acbw.getAddrConn(), errConnDrain)
            }
            ccb.UpdateBalancerState(connectivity.Connecting, nil)
            return
        default:
        }
        //触发解析事件？？
        ccb.cc.firstResolveEvent.Fire()
    }
}
```
分析完这段代码，**不难回答上面问题的答案**：
grpc.ccBalancer.Wrapper 会启动一个 goroutine，负责监听服务地址更新事件和连接状态变化事件：
-   服务地址更新事件会触发负载均衡组件更新连接池
-   连接状态变化事件会触发负载均衡组件更新连接池中连接的状态，以及更新picker


### Balancer注册和创建
现在我们看下，`balancer.newCCBalancerWrapper`中创建balancer的过程，`ccb.balancer = b.Build(ccb, bopts)`。下面是包初始化函数init()
```go
//grpclb.go
func init() {
    balancer.Register(newLBBuilder())
}

func (b *lbBuilder) Build(cc balancer.ClientConn, opt balancer.BuildOptions) balancer.Balancer {
    // This generates a manual resolver builder with a random scheme. This
    // scheme will be used to dial to remote LB, so we can send filtered address
    // updates to remote LB ClientConn using this manual resolver.
    //lbManualResolver主要用于grpclb内部使用，一个特殊的resolver,后续分析
    scheme := "grpclb_internal_" + strconv.FormatInt(time.Now().UnixNano(), 36)
    r := &lbManualResolver{scheme: scheme, ccb: cc}

    lb := &lbBalancer{
        //实现缓存功能
        cc:              newLBCacheClientConn(cc),
        target:          opt.Target.Endpoint,
        opt:             opt,
        fallbackTimeout: b.fallbackTimeout,
        doneCh:          make(chan struct{}),

        manualResolver: r,
        subConns:       make(map[resolver.Address]balancer.SubConn),
        scStates:       make(map[balancer.SubConn]connectivity.State),
        //初始化picker
        picker:         &errPicker{err: balancer.ErrNoSubConnAvailable},
        clientStats:    newRPCStats(),
        backoff:        defaultBackoffConfig, // TODO: make backoff configurable.
    }

    return lb
}
```

### lbBalancer.UpdateClientConnState
这里watcher的作用是，监听resolver通知的后端IP列表的更新，来更新`ccBalancerWrapper`的连接缓存
```go
ClientConn.updateResolverState -> ccBalancerWrapper.updateClientConnState -> ccBalancerWrapper.watcher -> lbBalancer.UpdateClientConnState
```

在解析到服务地址后最终是调用`lbBalancer.UpdateClientConnState`接口：
```go
func (lb *lbBalancer) UpdateClientConnState(ccs balancer.ClientConnState) {
    if grpclog.V(2) {
        grpclog.Infof("lbBalancer: UpdateClientConnState: %+v", ccs)
    }
    //处理balancer配置
    gc, _ := ccs.BalancerConfig.(*grpclbServiceConfig)
    lb.handleServiceConfig(gc)

    addrs := ccs.ResolverState.Addresses
    if len(addrs) <= 0 {
        return
    }

    //过滤出server地址和balancer地址
    var remoteBalancerAddrs, backendAddrs []resolver.Address
    for _, a := range addrs {
        if a.Type == resolver.GRPCLB {
            a.Type = resolver.Backend
            remoteBalancerAddrs = append(remoteBalancerAddrs, a)
        } else {
            backendAddrs = append(backendAddrs, a)
        }
    }

    //lb.ccRemoteLB == nil表示还没有和balancer server建立连接
    if lb.ccRemoteLB == nil {
        if len(remoteBalancerAddrs) <= 0 {
            grpclog.Errorf("grpclb: no remote balancer address is available, should never happen")
            return
        }
        // First time receiving resolved addresses, create a cc to remote
        // balancers.
        //和第一个balancer server建立连接
        lb.dialRemoteLB(remoteBalancerAddrs[0].ServerName)
        // Start the fallback goroutine.
        //如果在lb.fallbackTimeout无法和balancer server建立连接是使用兜底逻辑
        go lb.fallbackToBackendsAfter(lb.fallbackTimeout)
    }

    // cc to remote balancers uses lb.manualResolver. Send the updated remote
    // balancer addresses to it through manualResolver.
    lb.manualResolver.UpdateState(resolver.State{Addresses: remoteBalancerAddrs})

    lb.mu.Lock()
    lb.resolvedBackendAddrs = backendAddrs
    if lb.inFallback {
        // This means we received a new list of resolved backends, and we are
        // still in fallback mode. Need to update the list of backends we are
        // using to the new list of backends.
        lb.refreshSubConns(lb.resolvedBackendAddrs, true, lb.usePickFirst)
    }
    lb.mu.Unlock()
}
```
至此，我们已经分析完grpclb策略和loadbalancer建立连接的逻辑。grpclb会通过gRPCstream向loadbalancer请求服务地址列表

### lbBalancer.dialRemoteLB
```go
func (lb *lbBalancer) dialRemoteLB(remoteLBName string) {
    var dopts []grpc.DialOption
    // 省略其他参数构建
    // 这里最重要的两个参数是指定了使用PickFirst负载均衡策略以及使用manualResolver
    // 这里internal.WithResolverBuilder = withResolverBuilder
    // withResolverBuilder用于在参数中指定resolver
    // Explicitly set pickfirst as the balancer.
    dopts = append(dopts, grpc.WithBalancerName(grpc.PickFirstBalancerName))
    wrb := internal.WithResolverBuilder.(func(resolver.Builder) grpc.DialOption)
    dopts = append(dopts, wrb(lb.manualResolver))

    // Enable Keepalive for grpclb client.
    dopts = append(dopts, grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                20 * time.Second,
        Timeout:             10 * time.Second,
        PermitWithoutStream: true,
    }))

    // DialContext using manualResolver.Scheme, which is a random scheme
    // generated when init grpclb. The target scheme here is not important.
    //
    // The grpc dial target will be used by the creds (ALTS) as the authority,
    // so it has to be set to remoteLBName that comes from resolver.
    // 创建和load balancer server的ClientConn
    cc, err := grpc.DialContext(context.Background(), remoteLBName, dopts...)
    if err != nil {
        grpclog.Fatalf("failed to dial: %v", err)
    }
    lb.ccRemoteLB = cc
    // 进入监听模式
    go lb.watchRemoteBalancer()
}
```

### lbBalancer.watchRemoteBalancer
这里面进一步通过`callRemoteBalancer`函数和load balancer server建立连接，发起请求和处理获取的服务地址
```go
func (lb *lbBalancer) watchRemoteBalancer() {
    var retryCount int
    for {
        //和load balancer server建立连接，发起请求和处理获取的服务地址
        doBackoff, err := lb.callRemoteBalancer()
        select {
        case <-lb.doneCh:
            return
        default:
            if err != nil {
                if err == errServerTerminatedConnection {
                    grpclog.Info(err)
                } else {
                    grpclog.Warning(err)
                }
            }
        }
        // 失败重新触发解析
        // Trigger a re-resolve when the stream errors.
        lb.cc.cc.ResolveNow(resolver.ResolveNowOption{})

        lb.mu.Lock()
        lb.remoteBalancerConnected = false
        lb.fullServerList = nil
        // Enter fallback when connection to remote balancer is lost, and the
        // aggregated state is not Ready.
        if !lb.inFallback && lb.state != connectivity.Ready {
            // Entering fallback.
            lb.refreshSubConns(lb.resolvedBackendAddrs, true, lb.usePickFirst)
        }
        lb.mu.Unlock()
        //执行退避
        if !doBackoff {
            retryCount = 0
            continue
        }

        timer := time.NewTimer(lb.backoff.Backoff(retryCount))
        select {
        case <-timer.C:
        case <-lb.doneCh:
            timer.Stop()
            return
        }
        retryCount++
    }
}
```
### lbBalancer.processServerList
处理从load balancer获取到的server地址
```go
func (lb *lbBalancer) processServerList(l *lbpb.ServerList) {
    if grpclog.V(2) {
        grpclog.Infof("lbBalancer: processing server list: %+v", l)
    }
    lb.mu.Lock()
    defer lb.mu.Unlock()

    // Set serverListReceived to true so fallback will not take effect if it has
    // not hit timeout.
    lb.serverListReceived = true

    // If the new server list == old server list, do nothing.
    //前后IP地址完全一致，无需处理
    if reflect.DeepEqual(lb.fullServerList, l.Servers) {
        if grpclog.V(2) {
            grpclog.Infof("lbBalancer: new serverlist same as the previous one, ignoring")
        }
        return
    }
    lb.fullServerList = l.Servers

    //backendAddrs存储了从load balancer返回的有效服务地址
    var backendAddrs []resolver.Address
    for i, s := range l.Servers {
        //服务器已废弃
        if s.Drop {
            continue
        }

        //构建metadata主要是保留LoadBalanceToken
        md := metadata.Pairs(lbTokeyKey, s.LoadBalanceToken)
        ip := net.IP(s.IpAddress)
        ipStr := ip.String()
        if ip.To4() == nil {
            // Add square brackets to ipv6 addresses, otherwise net.Dial() and
            // net.SplitHostPort() will return too many colons error.
            ipStr = fmt.Sprintf("[%s]", ipStr)
        }
        addr := resolver.Address{
            Addr:     fmt.Sprintf("%s:%d", ipStr, s.Port),
            Metadata: &md,
        }
        if grpclog.V(2) {
            grpclog.Infof("lbBalancer: server list entry[%d]: ipStr:|%s|, port:|%d|, load balancer token:|%v|",
                i, ipStr, s.Port, s.LoadBalanceToken)
        }
        backendAddrs = append(backendAddrs, addr)
    }

    // Call refreshSubConns to create/remove SubConns.  If we are in fallback,
    // this is also exiting fallback.
    //更新子连接
    lb.refeshSubConns(backendAddrs, false, lb.usePickFirst)
}
```

### lbBalancer.refreshSubConns
这里删除废弃的服务器连接，为新增的服务器创建连接
```go
func (lb *lbBalancer) refreshSubConns(backendAddrs []resolver.Address, fallback bool, pickFirst bool) {
    opts := balancer.NewSubConnOptions{}
    if !fallback {
        opts.CredsBundle = lb.grpclbBackendCreds
    }

    lb.backendAddrs = backendAddrs
    lb.backendAddrsWithoutMetadata = nil

    fallbackModeChanged := lb.inFallback != fallback
    lb.inFallback = fallback

    //判断选取策略是否变化，获取第一个可用连接
    balancingPolicyChanged := lb.usePickFirst != pickFirst
    oldUsePickFirst := lb.usePickFirst
    lb.usePickFirst = pickFirst

    if fallbackModeChanged || balancingPolicyChanged {
        // Remove all SubConns when switching balancing policy or switching
        // fallback mode.
        //
        // For fallback mode switching with pickfirst, we want to recreate the
        // SubConn because the creds could be different.
        // 如果策略发生变化，删除所有的已经建立的连接
        for a, sc := range lb.subConns {
            if oldUsePickFirst {
                // If old SubConn were created for pickfirst, bypass cache and
                // remove directly.
                lb.cc.cc.RemoveSubConn(sc)
            } else {
                lb.cc.RemoveSubConn(sc)
            }
            delete(lb.subConns, a)
        }
    }

    //获取第一个可用连接
    if lb.usePickFirst {
        var sc balancer.SubConn
        for _, sc = range lb.subConns {
            break
        }
        //更新server地址并触发连接
        if sc != nil {
            sc.UpdateAddresses(backendAddrs)
            sc.Connect()
            return
        }
        // 创建SubConn并触发连接
        // This bypasses the cc wrapper with SubConn cache.
        // 这里最终调用了ClientConn.newAddrConn创建连接
        sc, err := lb.cc.cc.NewSubConn(backendAddrs, opts)
        if err != nil {
            grpclog.Warningf("grpclb: failed to create new SubConn: %v", err)
            return
        }
        sc.Connect()
        lb.subConns[backendAddrs[0]] = sc
        lb.scStates[sc] = connectivity.Idle
        return
    }

    // addrsSet is the set converted from backendAddrsWithoutMetadata, it's used to quick
    // lookup for an address.
    addrsSet := make(map[resolver.Address]struct{})
    // Create new SubConns.
    for _, addr := range backendAddrs {
        addrWithoutMD := addr
        addrWithoutMD.Metadata = nil
        addrsSet[addrWithoutMD] = struct{}{}
        lb.backendAddrsWithoutMetadata = append(lb.backendAddrsWithoutMetadata, addrWithoutMD)
        // 这里为新增的server创建SubConn,在lb.subConns找不到说明是新增服务器
        if _, ok := lb.subConns[addrWithoutMD]; !ok {
            // Use addrWithMD to create the SubConn.
            sc, err := lb.cc.NewSubConn([]resolver.Address{addr}, opts)
            if err != nil {
                grpclog.Warningf("grpclb: failed to create new SubConn: %v", err)
                continue
            }
            lb.subConns[addrWithoutMD] = sc // Use the addr without MD as key for the map.
            if _, ok := lb.scStates[sc]; !ok {
                // Only set state of new sc to IDLE. The state could already be
                // READY for cached SubConns.
                lb.scStates[sc] = connectivity.Idle
            }
            sc.Connect()
        }
    }

    // 删除已经废弃的服务连接
    for a, sc := range lb.subConns {
        // a was removed by resolver.
        if _, ok := addrsSet[a]; !ok {
            lb.cc.RemoveSubConn(sc)
            delete(lb.subConns, a)
            // Keep the state of this sc in b.scStates until sc's state becomes Shutdown.
            // The entry will be deleted in HandleSubConnStateChange.
        }
    }

    // Regenerate and update picker after refreshing subconns because with
    // cache, even if SubConn was newed/removed, there might be no state
    // changes (the subconn will be kept in cache, not actually
    // newed/removed).
    // 跟新状态和picker，forceRegeneratePicker和resetDrop都是true，会触发Picker更新
    lb.updateStateAndPicker(true, true)
}
```
### lbBalancer.updateStateAndPicker
```go
func (lb *lbBalancer) updateStateAndPicker(forceRegeneratePicker bool, resetDrop bool) {
    oldAggrState := lb.state
    // 聚合SubConn状态
    /*
    * 1. 只要有一个SubConn是connectivity.Ready状态，lb.state=connectivity.Ready
    * 2. 否则只要有一个SubConn是connectivity.Connecting，lb.state=connectivity.Connecting
    * 3. 否则lb.state=connectivity.TransientFailure
    */
    lb.state = lb.aggregateSubConnStates()
    // Regenerate picker when one of the following happens:
    //  - caller wants to regenerate
    //  - the aggregated state changed
    // 重新生成picker
    if forceRegeneratePicker || (lb.state != oldAggrState) {
        lb.regeneratePicker(resetDrop)
    }

      //更新lb状态和picker
    lb.cc.UpdateBalancerState(lb.state lb.picker)
}
```

### ccBalancerWrapper.UpdateBalancerState
```go
func (ccb *ccBalancerWrapper) UpdateBalancerState(s connectivity.State, p balancer.Picker) {
    ccb.mu.Lock()
    defer ccb.mu.Unlock()
    if ccb.subConns == nil {
        return
    }
    // Update picker before updating state.  Even though the ordering here does
    // not matter, it can lead to multiple calls of Pick in the common start-up
    // case where we wait for ready and then perform an RPC.  If the picker is
    // updated later, we could call the "connecting" picker when the state is
    // updated, and then call the "ready" picker after the picker gets updated.
    // 跟新ClientConn的picker
    ccb.cc.blockingpicker.updatePicker(p)
    //更新状态
    ccb.cc.csMgr.updaeState(s)
}
```

### lbBalancer.regeneratePicker
```go
func (lb *lbBalancer) regeneratePicker(resetDrop bool) {
    //lb.state == connectivity.TransientFailure
    //lb.state == connectivity.Connecting
    //这两种状态其实都是没有可用的SubConn，创建errPicker
    if lb.state == connectivity.TransientFailure {
        lb.picker = &errPicker{err: balancer.ErrTransientFailure}
        return
    }

    if lb.state == connectivity.Connecting {
        lb.picker = &errPicker{err: balancer.ErrNoSubConnAvailable}
        return
    }

    //这里lb.state==connectivity.Ready
    var readySCs []balancer.SubConn
    //在usePickFirst情况下，第一个肯定是connectivity.Ready状态
    if lb.usePickFirst {
        for _, sc := range lb.subConns {
            readySCs = append(readySCs, sc)
            break
        }
    } else {
        //查找connectivity.Ready状态的连接
        for _, a := range lb.backendAddrsWithoutMetadata {
            if sc, ok := lb.subConns[a]; ok {
                if st, ok := lb.scStates[sc]; ok && st == connectivity.Ready {
                    readySCs = append(readySCs, sc)
                }
            }
        }
    }

    //无可用的SubConn
    if len(readySCs) <= 0 {
        // If there's no ready SubConns, always re-pick. This is to avoid drops
        // unless at least one SubConn is ready. Otherwise we may drop more
        // often than want because of drops + re-picks(which become re-drops).
        //
        // This doesn't seem to be necessary after the connecting check above.
        // Kept for safety.
        lb.picker = &errPicker{err: balancer.ErrNoSubConnAvailable}
        return
    }
    //触发兜底逻辑，使用RoundRobin算法
    if lb.inFallback {
        lb.picker = newRRPicker(readySCs)
        return
    }
    //如果是需要重置，创建新的Picker
    if resetDrop {
        lb.picker = newLBPicker(lb.fullServerList, readySCs, lb.clientStats)
        return
    }
    //原来的Picker不是lbPicker，创建新Picker
    prevLBPicker, ok := lb.picker.(*lbPicker)
    if !ok {
        lb.picker = newLBPicker(lb.fullServerList, readySCs, lb.clientStats)
        return
    }
    //更新SubConn
    prevLBPicker.updateReadySCs(readySCs)
}
```

##  总结

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权