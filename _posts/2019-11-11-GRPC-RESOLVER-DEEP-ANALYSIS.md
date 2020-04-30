---
layout:     post
title:      gRPC源码分析之Resolver篇
subtitle:   gRPC客户端解析器实现分析
date:       2019-11-11
author:     pandaychen
catalog:    true
tags:
    - 负载均衡
---

##	前言
gRPC 负载均衡是针对每次请求，而不是连接，这样可以保证服务端负载的均衡性，所有gRPC负载均衡算法实现都在客户端。本系列文章对 gRPC 的负载均衡框架做深入的分析。

##  gRPC的解析器Resolver
Resolver，直观上就联想到/etc/resolv.conf，配置域名解析规则，和DNS服务器交互，获取解析结果。<br>
本篇文章详细分析下[gRPC-Resolver](https://godoc.org/google.golang.org/grpc/resolver)的实现，[上一篇文章](https://pandaychen.github.io/2019/07/11/GRPC-SERVICE-DISCOVERY/)介绍了gRPC负载均衡的基础概念。<br>
首先，来看一张关于gRPC客户端负载均衡实现的官方架构图（截止目前最新的架构）：
![image](https://raw.githubusercontent.com/grpc/proposal/master/L9_graphics/bar_after.png)
从图中，可以看到Resolver解析器位于架构的最左方，它主要完成下面这几个功能：

-   服务发现的实现
-   和注册中心（`Etcd`、`CoreDNS`、`Consul`等）通信，实时获取服务器的列表（或者处理变更信息）
-   将上步获取的数据（更新的结果），及时发送给Balancer，用以更新Connection Pool

##  Resolver的应用
这里先提供两个典型的实现，`DNSResolver`和`EtcdResolver`：
-   [DNSResolver](https://pandaychen.github.io/2019/07/11/GRPC-BALANCER-DNSRESOVLER-ANALYSIS/)，gRPC官方提供的实现，以定时轮询方式访问域名服务器来获取服务器更新
-   [EtcdResolver](https://etcd.io/docs/v3.3.12/dev-guide/grpc_naming/)，Etcd文档提供的示例，以List-Watcher方式实现的Resolver

##  resolver.go中主要结构分析
本小节，来分析下[resolver.go](https://godoc.org/google.golang.org/grpc/resolver)的主要结构。最早gRPC提供了[`Naming`](https://godoc.org/google.golang.org/grpc/naming)包，用来完成解析的功能，不过其功能很有限，现在已经deprecated了，现在一般用resolver包来完成。

### Address结构
Address结构中的`Addr`字段一般包含ip和端口信息，`Metadata`一般放入服务器的额外信息，比如权重、总连接数等等信息，用于负载均衡算法的判定：
```go
// Address represents a server the client connects to.
// This is the EXPERIMENTAL API and may be changed or extended in the future.
type Address struct {
	// Addr is the server address on which a connection will be established.
	Addr string

	// ServerName is the name of this address.
	// If non-empty, the ServerName is used as the transport certification authority for
	// the address, instead of the hostname from the Dial target string. In most cases,
	// this should not be set.
	//
	// If Type is GRPCLB, ServerName should be the name of the remote load
	// balancer, not the name of the backend.
	//
	// WARNING: ServerName must only be populated with trusted values. It
	// is insecure to populate it with data from untrusted inputs since untrusted
	// values could be used to bypass the authority checks performed by TLS.
	ServerName string

	// Attributes contains arbitrary data about this address intended for
	// consumption by the load balancing policy.
	Attributes *attributes.Attributes

	// Type is the type of this address.
	//
	// Deprecated: use Attributes instead.
	Type AddressType

	// Metadata is the information associated with Addr, which may be used
	// to make load balancing decision.
	//
	// Deprecated: use Attributes instead.
	Metadata interface{}
}
```

### Builder
官方文档的这句：`Builder creates a resolver that will be used to watch name resolution updates`，大致意思是：当向gRPC注册（解析器）服务发现时，实际上注册的是`Builder`，一般在`Build`中会开启单独的groutine，进行List-watcher逻辑。<br>
Build()参数中的`cc ClientConn`，提供了`Builder`和`ClientConn`交互的纽带，可以调用`cc.UpdateState(resolver.State{Addresses: addrList})`来向`ClientConn`即时发送服务器列表的更新。<br>
**<font color="#dd0000">这里先预埋一个问题，我们实现的`resolver.Builder()`在哪个gRPC阶段被调用？</font>**

```go
type Builder interface {
	// Build creates a new resolver for the given target.
	//
	// gRPC dial calls Build synchronously, and fails if the returned error is
    // not nil.
    // 创建Resolver，当resolver发现服务列表更新，需要通过ClientConn接口通知上层
	Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)
	// Scheme returns the scheme supported by this resolver.
	// Scheme is defined at https://github.com/grpc/grpc/blob/master/doc/naming.md.
	Scheme() string
}
```

### Resolver
```go
// Resolver watches for the updates on the specified target.
// Updates include address updates and service config updates.
type Resolver interface {
	// ResolveNow will be called by gRPC to try to resolve the target name
	// again. It's just a hint, resolver can ignore this if it's not necessary.
	//
    // It could be called multiple times concurrently.
    // 当有连接被出现异常时，会触发该方法，因为这时候可能是有服务实例挂了，需要立即实现一次服务发现
	ResolveNow(ResolveNowOptions)
	// Close closes the resolver.
	Close()
}
```

### ClientConn
resolver中的`ClientConn`结构提供了resolver通知`ClientConn`更新服务端列表的回调方法。日常封装自己的resolver时，建议使用一个成员变量来接收`resolver.ClientConn`，这里注册中心用的是`Etcd`：
```go
type EtcdResolver struct {
    ...
	cc            resolver.ClientConn       //用来接收在resolver.Build()中的第二个参数，由外部传入
	EtcdCli       *clientv3.Client
    ...
}
```
再看ClientConn的结构：
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
    // 服务列表和服务配置更新回调接口，目前都使用这个通知gRPC上层，包含了下面两个已废弃接口的功能
	UpdateState(State)
	// ReportError notifies the ClientConn that the Resolver encountered an
	// error.  The ClientConn will notify the load balancer and begin calling
	// ResolveNow on the Resolver with exponential backoff.
	ReportError(error)
	// NewAddress is called by resolver to notify ClientConn a new list
	// of resolved addresses.
	// The address list should be the complete list of resolved addresses.
	//
	// Deprecated: Use UpdateState instead. -- 已废弃（服务列表更新通知接口）
	NewAddress(addresses []Address)
	// NewServiceConfig is called by resolver to notify ClientConn a new
	// service config. The service config should be provided as a json string.
	//
	// Deprecated: Use UpdateState instead. -- 已废弃（服务配置更新通知接口）
	NewServiceConfig(serviceConfig string)
	// ParseServiceConfig parses the provided service config and returns an
	// object that provides the parsed config.
	ParseServiceConfig(serviceConfigJSON string) *serviceconfig.ParseResult
}
```

### Scheme
通过Scheme来标识gRPC使用的解析器名字，比如，`Etcd`的scheme就可以写成`etcdv3:///"`，`CoreDNS`的scheme可写成`dns:///8.8.8.8:53`
```go
// string. e.g. target string "unknown_scheme://authority/endpoint" will be parsed into
// &Target{Scheme: resolver.GetDefaultScheme(), Endpoint: "unknown_scheme://authority/endpoint"}.
type Target struct {
	Scheme    string
	Authority string
	Endpoint  string
}
```
### 小结
现在我们对resolver做下小结：<br>
其中`Builder`接口用来创建`Resolver`，我们可以提供自己的服务发现实现逻辑，然后将其注册到gRPC中，其中通过`scheme`来标识，而`Resolver`接口则是提供服务发现功能。当`Resolver`发现服务列表发生变更时，会通过`ClientConn`回调接口通知上层。
下一小节，我们来看下Resolver在gRPC客户端流程中实例的调用链。

##  Resolver的调用链
本小节，来分析下Resolver的调用链是什么。<br>

在实际项目中，一般gRPC`+`Etcd实现的客户端的负载均衡调用的代码实现如下（Etcd也有个官方的简单实现[Using etcd discovery with go-grpc](https://etcd.io/docs/v3.3.12/dev-guide/grpc_naming/)）：
-	通用的实现方式

```go
resolver := NewXXXResolver(service_name,registy_endpoints)      //传入要watcher的key和etcd集群地址
balancer := grpc.RoundRobin(resolver)                           //调用负载均衡器初始化resolver
ctx, cancel := context.WithTimeout(context.Background(), GRPC_CONNECT_TIMEOUT*time.Second)
//Dial/DialContext开启balancer+resolver的功能
conn, err = grpc.DialContext(ctx, registy_endpoints, grpc.WithTransportCredentials(creds), grpc.WithBalancer(balancer), grpc.WithBlock())
```

-	Etcd文档的实现，大同小异

```go
import (
	"go.etcd.io/etcd/clientv3"
	etcdnaming "go.etcd.io/etcd/clientv3/naming"
	"google.golang.org/grpc"
)
cli, cerr := clientv3.NewFromURL("http://localhost:2379")
r := &etcdnaming.GRPCResolver{Client: cli}
b := grpc.RoundRobin(r)
conn, gerr := grpc.Dial("my-service", grpc.WithBalancer(b), grpc.WithBlock(), ...)
```

### 调用链视图
grpc.RoundRobin-->Dial/DialContext-->newCCResolverWrapper-->调用resolver.Build()--->调用Build()实现的Watcher()--->完成并返回状态
![image]()

### grpc.RoundRobin
[grpc.RoundRobin](https://github.com/grpc/grpc-go/blob/master/balancer.go#L124)是一个`grpc.Balancer`
它的原型如下，可见传入的参数是`naming.Resolver`，返回值是`grpc.Balancer`，ps：从描述上看，此方法也即将`Deprecated`，更新后的方法[balancer/roundrobin](https://godoc.org/google.golang.org/grpc/balancer/roundrobin)，这是一个很规范的实现gRPC `Picker`的模板，后面文章会详细分析。
```go
// Deprecated: please use package balancer/roundrobin. May be removed in a future 1.x release.
func RoundRobin(r naming.Resolver) Balancer {
	return &roundRobin{r: r}
}
```

### DialContext
这里我们就从`DialContext`入手，看下gRPC对Resolver的处理过程。当我们使用Dial或者DialContext接口创建gRPC的客户端连接时，首先会解析参数target（Etcd集群地址），然后创建对应的resolver：
``` go
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
	cc := &ClientConn{
		target:            target,
		csMgr:             &connectivityStateManager{},
		conns:             make(map[*addrConn]struct{}),        //初始化ClientConn的连接存储Map
		dopts:             defaultDialOptions(),
		blockingpicker:    newPickerWrapper(),
		czData:            new(channelzData),
		firstResolveEvent: grpcsync.NewEvent(),
	}
    ......
    for _, opt := range opts {
        //解析参数
		opt.apply(&cc.dopts)
	}
    ......
    
	// resolverBuilder，用于解析target为目标服务列表
	// 如果没有指定resolverBuilder
	if cc.dopts.resolverBuilder == nil {
		// 解析target，根据target的scheme获取对应的resolver
		cc.parsedTarget = parseTarget(cc.target)							//解析我们自定义实现的resolver
 		cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)		//重要：在下面分析
		// 如果scheme没有注册对应的resolver
		if cc.dopts.resolverBuilder == nil {
            // 使用默认的resolver
			cc.parsedTarget = resolver.Target{
				Endpoint: target, // 这时候参数target就是endpoint，passthrough的实现就是直接返回endpoint，即不使用服务发现功能，参数Dial传进来的地址就是grpc server的地址
			}
            // 获取默认的resolver，也就是passthrough
			cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
		}
	} else {
        // 如果Dial的option中手动指定了需要使用的resolver，那么endpoint也是target
        // 实例中指定了我们自己实现的NewXXXResolver，target为Etcd集群的地址
        /*
            // &Target{Scheme: resolver.GetDefaultScheme(), Endpoint: "unknown_scheme://authority/endpoint"}.
            type Target struct {
                Scheme    string
                Authority string
                Endpoint  string    //
            }
        */
		cc.parsedTarget = resolver.Target{Endpoint: target}
	}
    
	......
    
	// newCCResolverWrapper方法内调用builder的Build接口创建resolver
	rWrapper, err := newCCResolverWrapper(cc)
	if err != nil {
		return nil, fmt.Errorf("failed to build resolver: %v", err)
	}

	cc.mu.Lock()
	cc.resolverWrapper = rWrapper
	cc.mu.Unlock()
    
 	......

	return cc, nil
}
```

### parseTarget
回想下resolver包中[`Build()`](https://godoc.org/google.golang.org/grpc/resolver#Get)中的`Scheme()`方法，一般实现中自己设定一个解析器标识:
```go
type Builder interface {
    // Build creates a new resolver for the given target.
    //
    // gRPC dial calls Build synchronously, and fails if the returned error is not nil.
    Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)
    // Scheme returns the scheme supported by this resolver.
    // Scheme is defined at https://github.com/grpc/grpc/blob/master/doc/naming.md.
    Scheme() string
}
```

在项目中，需要设定一个解析器标识scheme，例如，解析器名称设置为`etcdv3resolver`，那么在`Dial`传入`etcdv3resolver:///`，就在`parseTarget`这里做解析：
```go
func (r EtcdNewResolver) Scheme() string {
	return "etcdv3resolver"
}
```

如果已经实现自定义的resolver，那么传入`Dial`的scheme会在这里做解析：
```go
// 有效的target：scheme://authority/endpoint
func parseTarget(target string) (ret resolver.Target) {
	var ok bool
	ret.Scheme, ret.Endpoint, ok = split2(target, "://")	//传入etcdv3resolver:///，Scheme就是etcdv3resolver
	if !ok {
        // 如果没有scheme，则整个target作为endpoint
		return resolver.Target{Endpoint: target}
	}
    // 如果指定了sheme，那么必须有`/`，分割authorigy和endpoint
    // 当不需要指定authority，比如使用dnsResolver时:`dns:///www.demo.com`
	ret.Authority, ret.Endpoint, ok = split2(ret.Endpoint, "/")
	if !ok {
		return resolver.Target{Endpoint: target}
	}
	return ret
}
```

###	resolver.Get(cc.parsedTarget.Scheme)
这里会根据解析得到的解析器的名称，去resolver包中`m map[string]Builder`这个全局map中查询对应的`Builder`，然后返回给`cc.dopts.resolverBuilder`，这样，我们自定义的`resolver.Builder`就成功的和`ClientConn`关联上了。
```go
//resolver中的全局MAP
m = make(map[string]Builder)
// Get returns the resolver builder registered with the given scheme.
// If no builder is register with the scheme, nil will be returned.
func Get(scheme string) Builder {
	if b, ok := m[scheme]; ok {
		return b
	}
	return nil
}
```

### newCCResolverWrapper
这里回答前面预埋的一个问题，我们自己构建resolver中的`Build()`方法，最终是在哪里被调用的？答案就是`newCCResolverWrapper`。代码如下：
```go
func newCCResolverWrapper(cc *ClientConn) (*ccResolverWrapper, error) {
	// 在DialContext方法中，已经初始化了resolverBuilder
    rb := cc.dopts.resolverBuilder
	if rb == nil {
		return nil, fmt.Errorf("could not get resolver for scheme: %q", cc.parsedTarget.Scheme)
	}

 	// ccResolverWrapper实现resolver.ClientConn接口，用于提供服务列表变更之后的通知回调接口
	ccr := &ccResolverWrapper{
		cc:     cc,     //本客户端的ClientConn
		addrCh: make(chan []resolver.Address, 1),   //用于通知channel
		scCh:   make(chan string, 1),                //用于通知channel
	}

	var err error
    // 创建resolver，resolver创建之后，需要立即执行服务发现逻辑，然后将发现的服务列表通过resolver.ClientConn回调接口通知上层
    
	// 非常重要：这里的Build也就是我们在自己的resolver.go中实现的Build()方法，传入的三个参数。在我们实现中，Build中创建和启动了Watcher
	 // 在gRPC的DNSresolver实现里，调用dnsBuilder.Build函数创建dnsResolver
	ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, resolver.BuildOption{DisableServiceConfig: cc.dopts.disableServiceConfig})
	if err != nil {
		return nil, err
	}
	return ccr, nil
}
```

###	Resovler与Balancer的交互


##  总结
至此，对gRPC-Resolver的流程分析就基本完成了。

##	参考
-	[package resolver](https://godoc.org/google.golang.org/grpc/resolver)
-	[package grpc](https://godoc.org/google.golang.org/grpc)
-	[package roundrobin](https://godoc.org/google.golang.org/grpc/balancer/roundrobin)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
