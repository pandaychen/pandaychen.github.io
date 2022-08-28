---
layout:     post
title:      Kratos 源码分析：Naming 解析（下）
subtitle:   分析 Warden 中对 Naming 的调用及实例应用
date:       2020-07-12
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---


##  0x00    前言
上一篇文章：[Kratos 源码分析：Naming 解析（上）](https://pandaychen.github.io/2020/06/13/KRATOS-NAMING/)，分析了 Kratos 的 Naming 实现机制。<br>
从宏观上来看，Naming 的实现就是统一（提供）通用服务注册中心的接口，使得调用方可以屏蔽不同注册中心的接口差异。<br>
本篇文章继续分析下 Warden 框架是如何将 Naming 接口与 gPRC 接口封装在一起的。warden 的 [服务发现模块](https://github.com/go-kratos/kratos/tree/master/pkg/net/rpc/warden/resolver)，用于从底层的注册中心中获取 Server 节点列表并返回给 gRPC。


##    0x01	warden Resolver 简介
Warden 的 resolver 封装主要目的有两点：
1.	实现（实例化）`gRPC.resolver` 的 `Builder` 和 `Resolver` 这两个 `interface{}`
2.	调用 Kratos 的 Naming 提供的接口拿到后端节点列表，通过 `ClientConn` 提供的接口 `UpdateState` 及 `NewAddress` 通知 gRPC 内部完成 Resolver 的逻辑

以上两点就是 warden.Resolver 需要实现的功能，代码在 [这里](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/resolver/resolver.go)


前面文章 [gRPC 源码分析之 Resolver 篇](https://pandaychen.github.io/2019/11/11/GRPC-RESOLVER-DEEP-ANALYSIS/) 分析了 gRPC 的 Resolver 的实现，这里再简单回顾下。
gRPC 暴露了 [服务发现的接口](https://github.com/grpc/grpc-go/blob/master/resolver/resolver.go)：`resolver.Builder` 和 `resolver.ClientConn` 和 `resolver.Resolver`，为了详细分析下 Kratos 对 Resolver 的封装，这里还是贴下 gRPC 的主要结构定义：

```golang
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

基于上面的定义，warden 的 [`resolver.Builder`](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/resolver/resolver.go#L48) 需要实现 gRPC 的 `resolver.Builder` 方法；
[`resolver.Resolver`](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/resolver/resolver.go#L90) 需要实现上面的 `Resolver` 方法。

##	0x02	Warden 的 Resolver 实现
Warden 中封装了 gRPC 中关于 Resovler 的 [接口](https://godoc.org/google.golang.org/grpc/resolver)，从而库的开发者不需要再和 gRPC 接口打交道。

Kratos 的 resolver 包实现 [代码在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/resolver/resolver.go)。它主要提供了如下对外部的接口：
-	`func Register(b naming.Builder)`：完成向 gRPC 的 resovler 注册，不存在才注册
-	`func Set(b naming.Builder)`：和 `Register` 类似，覆盖注册
-	gRPC 中的 [`resolver.Build`](https://github.com/grpc/grpc-go/blob/master/resolver/resolver.go#L225) 封装，需要实现如下的方法：
	-	`Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)`
	-	`Scheme() string`
-	gRPC 中的 [`resolver.Resolver`](https://github.com/grpc/grpc-go/blob/master/resolver/resolver.go#L241) 封装，需要实现如下几个方法：
	-	`ResolveNow(ResolveNowOptions)`
	-	`Close()`

另外，在 Resolver 中需要开启单独的 goroutine，实现模型中的 Watcher 机制，然后将 Watcher 返回的结果，调用 gRPC 的 [接口方法](https://godoc.org/google.golang.org/grpc/resolver#ClientConn) 通知到 gRPC 的内部逻辑，如 `UpdateState` 或 `NewAddress`。

总结下上面的流程，如下图所示：
![image](https://wx2.sbimg.cn/2020/08/14/30KlY.png)

下一小节开始，分析下 `Resolver` 代码实现逻辑。


##	0x03	全局注册
在客户端实例化代码中，gRPC 要求必须指定要使用的 Resolver 名字，这里提供了 `Register` 和 `Set` 接口供外部调用：
```golang
// Register register resolver builder if nil.
func Register(b naming.Builder) {
	mu.Lock()
	defer mu.Unlock()
	if resolver.Get(b.Scheme()) == nil {
		resolver.Register(&Builder{b})
	}
}

// Set override any registered builder
func Set(b naming.Builder) {
	mu.Lock()
	defer mu.Unlock()
	resolver.Register(&Builder{b})
}
```

##	0x04	resolver.Builder 封装
`resolver.Build` 简单的封装了 `naming.Builder`，即可以调用 `naming.Builder` 的所有方法：
```golang
// Builder is also a resolver builder.
// It's build() function always returns itself.
type Builder struct {
	naming.Builder
}
```

`resolver.Builder` 实现的 `Build` 方法代码如下，按照如下几步：
1.	创建 `resolver.Resolver` 并初始化
2.	`resolver.Resolver` 的成员 `nr   naming.Resolver` 的初始化是调用 `naming.Builder.Build` 方法进行的，注意在这个方法中，启动了 `go app.watch(appid)` 的监听逻辑，这个 `appid` 就是下面代码中的 `str[0]`
3.	启动独立的工作协程 `go r.updateproc()` 来实现 `Fetch` 及调用 gRPC 接口通知内部的工作

```golang
// Build returns itself for Resolver, because it's both a builder and a resolver.
func (b *Builder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	var zone = env.Zone
	ss := int64(50)
	clusters := map[string]struct{}{}
	str := strings.SplitN(target.Endpoint, "?", 2)
	if len(str) == 0 {
		return nil, errors.Errorf("warden resolver: parse target.Endpoint(%s) failed!err:=endpoint is empty", target.Endpoint)
	} else if len(str) == 2 {
		m, err := url.ParseQuery(str[1])
		if err == nil {
			for _, c := range m[naming.MetaCluster] {
				clusters[c] = struct{}{}
			}
			zones := m[naming.MetaZone]
			if len(zones) > 0 {
				zone = zones[0]
			}
			if sub, ok := m["subset"]; ok {
				if t, err := strconv.ParseInt(sub[0], 10, 64); err == nil {
					ss = t
				}

			}
		}
	}
	r := &Resolver{
		nr:   b.Builder.Build(str[0], naming.Filter(Scheme, clusters), naming.ScheduleNode(zone), naming.Subset(int(ss))),
		cc:   cc,
		quit: make(chan struct{}, 1),
		zone: zone,
	}
	go r.updateproc()
	return r, nil
}
```

##	0x05	resolver.Resolver 封装
`resolver.Resolver` 结构如下，可以看到它封装了 `resolver.ClientConn`，这是用来调用 gRPC 通知方法；同时也封装了 `naming.Resolver`，这是用来调用 `naming.Resolver` 实例化的方法。这里就体现了设计模式的好处了，封装的是通用结构，具体调用哪个实例的方法由传入的实例化对象决定。

```golang
// Resolver watches for the updates on the specified target.
// Updates include address updates and service config updates.
type Resolver struct {
	nr   naming.Resolver			//nr：etcd、zooker、discovery 的实例化成员
	cc   resolver.ClientConn		//gRPC 通知的接口
	quit chan struct{}

	clusters   map[string]struct{}
	zone       string
	subsetSize int64
}
```

再看下 Resovler 实现的方法：
`Close` 方法：退出解析，当 `r.quit` 这个 channel 被触发时，调用 `nr.Close()` 方法通知 Naming 退出
```golang
// Close is a noop for Resolver.
func (r *Resolver) Close() {
	select {
	case r.quit <- struct{}{}:
		r.nr.Close()
	default:
	}
}
```
`nr.Close` 方法如下：
```golang
// Close close resolver.
func (r *Resolve) Close() error {
	r.e.mutex.Lock()
	if app, ok := r.e.apps[r.id]; ok && len(app.resolver) != 0 {
		delete(app.resolver, r)
	}
	r.e.mutex.Unlock()
	return nil
}
```

`updateproc` 方法，以独立的 goroutine 执行 Wacher 逻辑：
1.	调用 `naming.Resolver` 实现的 Watch() 方法获取一个接收通知的 channel-event
2.	以 `for...select` 模型监听 `event`、`quit` 等 channel，如果 `event` 有通知，通过 `naming.Resolver` 的 `Fetch()` 方法拿到结果
3.	上一步获取结果，调用 gRPC 的通知方法通知上层，完成一次 `watcher()` 工作，周而复始

```golang
func (r *Resolver) updateproc() {
	event := r.nr.Watch()
	for {
		select {
		case <-r.quit:
			return
		case _, ok := <-event:
			if !ok {
				return
			}
		}
		if ins, ok := r.nr.Fetch(context.Background()); ok {
			instances, _ := ins.Instances[r.zone]
			if len(instances) == 0 {
				for _, value := range ins.Instances {
					instances = append(instances, value...)
				}
			}
			r.newAddress(instances)
		}
	}
}
```

`newAddress` 的作用，主要是将 `instances []*naming.Instance` 中获取的后端实例，封装为 `resolver.Address` 列表，通过 `grpc.ClientConn` 的接口 `r.cc.NewAddress(addrs)`，发送给 gRPC 上层进行更新：
从测试来看，每一次后端 Backend 节点有变化（增加或减少），都会触发 gRPC 返回当前全部存活的 Backend 列表。另外，`resolver.Address{}` 的 [定义](https://godoc.org/google.golang.org/grpc/resolver#Address) 中，提供了 `Metadata`，让用户可以保存更多需要的字段信息。在这里保存的 `Metadata` 数据，在 `gRPC.balancer` 的相关接口会返回给用户。<br>

在 Warden 的 `Metadata` 中，`Weight` 为权重，`Color` 为蓝绿标识，用于发布时使用（试想这样一种场景，刚部署的服务器，希望按照一定的策略或比率将在线流量导入），权重主要在负载均衡的算法中使用，这个在另外的文章进行分析。
```json
{
    "Weight": "uint64(weight)",
    "Color": "ins.Metadata[naming.MetaColor]"
}
```

`newAddress` 的代码如下：

```golang
func (r *Resolver) newAddress(instances []*naming.Instance) {
	if len(instances) <= 0 {
		return
	}
	addrs := make([]resolver.Address, 0, len(instances))
	for _, ins := range instances {
		var weight int64
		if weight, _ = strconv.ParseInt(ins.Metadata[naming.MetaWeight], 10, 64); weight <= 0 {
			weight = 10
		}
		var rpc string
		for _, a := range ins.Addrs {
			u, err := url.Parse(a)
			if err == nil && u.Scheme == Scheme {
				rpc = u.Host
			}
		}
		addr := resolver.Address{
			Addr:       rpc,// Addr is the server address on which a connection will be established.
			Type:       resolver.Backend,
			ServerName: ins.AppID,	// ServerName is the name of this address.
			Metadata:   wmeta.MD{Weight: uint64(weight), Color: ins.Metadata[naming.MetaColor]},
		}
		addrs = append(addrs, addr)
	}
	log.Info("resolver: finally get %d instances", len(addrs))
	// 调用 grpc 的接口 NewAddress 更新 addrs
	r.cc.NewAddress(addrs)
}
```

##	0x06	Naming 应用
这里介绍使用 Etcd 来实现调用 Naming 包的例子。包含服务注册和服务发现两个部分：

####	服务注册（服务端）
客户端使用了 Etcd 进行服务发现，服务端启动后必须将自己的服务信息注册到 Etcd<br>

相对服务发现来讲，服务注册较为简单，[`naming/etcd/etcd.go`](https://github.com/go-kratos/kratos/blob/master/pkg/naming/etcd/etcd.go) 内的代码实现了 `naming/naming.go` 内的 [`Register` 接口](https://github.com/go-kratos/kratos/blob/master/pkg/naming/etcd/etcd.go#L162)，服务端启动时可以参考下面代码进行注册：
注意：这里的 `appID` 是共同服务的前缀，依靠 `Hostname` 做唯一的服务区分标识：
```golang
func runServer(addr, svcid string) *warden.Server {
	server := warden.NewServer(&warden.ServerConfig{
		// 服务端每个请求的默认超时时间
		Timeout: xtime.Duration(time.Second),
	})

	//start warden service registry
	config := &clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: time.Second * 5,
		DialOptions: []grpc.DialOption{grpc.WithBlock()},
	}
	etcd_builder, err := etcd.New(config)

	if err != nil {
		fmt.Println(err)
		return nil
	}

	//kratos-etcd-key: ratos_etcd/app1/h1
	var localaddr []string
	localaddr = append(localaddr, fmt.Sprintf("grpc://%s", addr))
	_, err = etcd_builder.Register(context.Background(), &naming.Instance{
		AppID:    "app1",
		Hostname: svcid,
		Zone:     "zone01",
		Addrs:    localaddr,
	})

	server.Use(middleware()).Use(middleware()).Use(stats())
	pb.RegisterGreeterServer(server.Server(), &helloServer{addr: addr})
	go func() {
		err := server.Run(addr)
		if err != nil {
			panic("run server failed!" + err.Error())
		}
	}()
	return server
}
```

####	服务发现（客户端）
使用 `etcd` 进行服务发现的方式，需要在业务的 `NewClient` 前进行注册并启动内置的 Resolver，客户端的参考代码如下：<br>
注意，在 `init` 中实现上面的注册逻辑，在 `client.Dial` 方法中传入参数 `etcd://default/"+AppID` 告诉 gRPC，使用 Resolver 的名字

```golang
import (
	"context"
	"github.com/go-kratos/kratos/pkg/naming/etcd"
	"github.com/go-kratos/kratos/pkg/net/rpc/warden"
	"github.com/go-kratos/kratos/pkg/net/rpc/warden/resolver"
	"google.golang.org/grpc"
)

// AppID your appid, ensure unique.
const AppID = "demo.service" // NOTE: example

func init(){
	// NOTE: 注意这段代码，表示要使用 etcd 进行服务发现 , 其他事项参考 discovery 的说明
    // NOTE: 在启动应用时，可以通过 flag(-etcd.endpoints) 或者 环境配置 (ETCD_ENDPOINTS) 指定 etcd 节点
    // NOTE: 如果需要自己指定配置时 需要同时设置 DialTimeout 与 DialOptions: []grpc.DialOption{grpc.WithBlock()}
	resolver.Register(etcd.Builder(nil))
}

// NewClient new member grpc client
func NewClient(cfg *warden.ClientConfig, opts ...grpc.DialOption) (DemoClient, error) {
	client := warden.NewClient(cfg, opts...)
  	// 这里使用 etcd scheme
	conn, err := client.Dial(context.Background(), "etcd://default/"+AppID)
	if err != nil {
		return nil, err
	}
	// 注意替换这里：
	// NewDemoClient 方法是在 "api" 目录下代码生成的
	// 对应 proto 文件内自定义的 service 名字，请使用正确方法名替换
	return NewDemoClient(conn), nil
}
```

> 注意：`resolver.Register` 是全局行为，建议放在包加载阶段或 main 方法开始时执行，该方法执行后会在 gRPC 内注册构造方法

`target` 是 `etcd://default/${appid}`，当 gRPC 内进行解析后会得到 `scheme`=`etcd` 和 `appid`，然后进行以下逻辑：

1. `warden/resolver.Builder` 会通过 `scheme` 获取到 `naming/etcd.Builder` 对象（靠 `resolver.Register` 注册过的）
2. 拿到 `naming/etcd.Builder` 后执行 `Build(appid)` 构造 `naming/etcd.Discovery`
3. `naming/etcd.Build` 对象基于 `appid` 就知道要获取哪个服务的实例信息


##	0x07	总结
本文是 Kratos 的 Naming 机制分析的第二篇，通过这两篇文章，对 Kratos 的服务发现逻辑有了一个比较清晰的理解。

##	0x08	参考
-	[warden-resolver-resolver.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/resolver/resolver.go)