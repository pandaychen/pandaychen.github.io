---
layout:     post
title:      gRPC 源码分析之 DnsResolver 篇
subtitle:   如何使用内置的 DNS 负载均衡器
date:       2019-11-12
author:     pandaychen
header-img:
catalog: true
tags:
    - gRPC
    - 负载均衡
---

##  0x00	介绍

关于 `gRPC Naming` 机制，官方有比较详细的 [文档](https://github.com/grpc/grpc/blob/master/doc/naming.md) 介绍。

Resolver（解析器）在 `gRPC` 中完成了这样一个过程，它对来自服务注册中心的数据（`Push` 或者 `Pull` 两种方式），做出响应，将得到的结果数据通知 `gRPC` 内置的负载均衡器 `Balancer`。在实现上通常划分为 `Resolver` 和 `Watcher` 两个模块。官方提供了一个基于 `DNS` 的实现，这篇文章来分析下这个 `DnsResolver` 的实现逻辑。

`gRPC` 支持 `DNS` 作为默认的 `Name Resolver`，如果配置的域名指向多个合法的 `DNS` 记录（如 `A`、`TXT` 等），则使用 `DnsResolver` 的 `gRPC` 请求将在多个 `IP` 之间轮转。客户端的调用形式如下：

```golang
conn, err := grpc.Dial(
        "dns:///test-service-domain:8080",
        grpc.WithBalancerName(roundrobin.Name),
        grpc.WithInsecure())
```

如果去除 `dns:///`, 仅仅是域名加端口的形式，则只会请求同一个实例。只有当该实例 Shutdown 或者下线后才会切换到另一个实例。

```golang
conn, err := grpc.Dial(
        "test-service-domain:8081",
        grpc.WithBalancerName(roundrobin.Name),
```

加了 `dns:///` 之后，客户端的调用逻辑有了怎样的变化？本篇博客就来解答这个问题。

##	0x01	官方文档

`gRPC` 支持 `DNS` 作为默认 `Naming` 系统，同时也提供了实现 `Naming` 系统乃至 `LoadBalance` 功能的用户侧接口。所以，第三方注册中心，如 `Etcd`、`Consul`、`Zookeeper` 都可以作为非常优秀的 `gRPC` 负载均衡实现。
`gRPC Name Resolution` 常用如下格式，scheme 表示要使用的 `Naming` 方式。目前常用的 schemes 有（`DNS` 是内置的方案）：

```text
scheme://authority/endpoint_name

dns (例: dns://8.8.8.8/www.qq.com)
ipv4 (IPv4 地址 例: ipv4:///110.12.92.1:443)
ipv6 (IPv6 地址 例: ipv6:///2607:f8b0:400a:801::1001)
```

##  0x02	resolver.go

``` golang
// Package resolver defines APIs for name resolution in gRPC.
// All APIs in this package are experimental.
package resolver

var (
	// m is a map from scheme to resolver builder.
	// m 是 scheme 和 resolver 构建器的关系映射，scheme 可以是自定义的，如 etcd、consul 等 string
	m = make(map[string]Builder)
	// defaultScheme is the default scheme to use.
	// defaultScheme 是默认使用的 scheme, 这里主要是为了方便测试, 因为有些测试依赖于 target 并未被解决并直接返回 endpoint 给拨号客户端。
	defaultScheme = "passthrough"
)

// TODO(bar) install dns resolver in init(){}.

// Register registers the resolver builder to the resolver map.
// b.Scheme will be used as the scheme registered with this builder.
// 注册各种 scheme 对应的 resolver 构建器，注册自定义的 resolver
func Register(b Builder) {
	m[b.Scheme()] = b
}

// Get returns the resolver builder registered with the given scheme.
// If no builder is register with the scheme, the default scheme will
// be used.
// If the default scheme is not modified, "dns" will be the default
// scheme, and the preinstalled dns resolver will be used.
// If the default scheme is modified, and a resolver is registered with
// the scheme, that resolver will be returned.
// If the default scheme is modified, and no resolver is registered with
// the scheme, nil will be returned.
// 根据 scheme 查找对应的 resolver 构建器，未找到则返回默认构建器
func Get(scheme string) Builder {
	if b, ok := m[scheme]; ok {
		return b
	}
	if b, ok := m[defaultScheme]; ok {
		return b
	}
	return nil
}

// SetDefaultScheme sets the default scheme that will be used.
// The default default scheme is "dns".
// 重置默认构建器, gRPC 默认的构建器是 dns
func SetDefaultScheme(scheme string) {
	defaultScheme = scheme
}

// AddressType indicates the address type returned by name resolution.
// 定义解析器返回的地址类型
type AddressType uint8

const (
	// Backend indicates the address is for a backend server.
	Backend AddressType = iota		// 后台服务器地址
	// GRPCLB indicates the address is for a grpclb load balancer.
	GRPCLB			// 负载均衡器地址
)

// Address represents a server the client connects to.
// This is the EXPERIMENTAL API and may be changed or extended in the future.
// 注意：表示客户端访问的服务器地址 -- 和服务注册信息关联较大
type Address struct {
	// Addr is the server address on which a connection will be established.
	// 用于构建 connection 的服务器地址
	Addr string
	// Type is the type of this address.
	// 地址类型
	Type AddressType
	// ServerName is the name of this address.
	// It's the name of the grpc load balancer, which will be used for authentication.
	// 地址名。当用于 authentication 时，它是 grpc 负载均衡器的名称
	ServerName string
	// Metadata is the information associated with Addr, which may be used
	// to make load balancing decision.
	// 地址关联元数据信息，被用于做负载均衡决策，一般关联一个 map
	Metadata interface{}
}

// BuildOption includes additional information for the builder to create
// the resolver.
// 构建器创建解析器附属信息
type BuildOption struct {
}

// ClientConn contains the callbacks for resolver to notify any updates
// to the gRPC ClientConn.
// gRPC 的客户端核心概念：ClientConn，是一个方法的集合，是解析器 Resolver 通知 ClientConn 解析结果的回调接口
type ClientConn interface {
	// NewAddress is called by resolver to notify ClientConn a new list
	// of resolved addresses.
	// The address list should be the complete list of resolved addresses.
	// 通知一个新地址列表
	NewAddress(addresses []Address)
	// NewServiceConfig is called by resolver to notify ClientConn a new
	// service config. The service config should be provided as a json string.
	// 通知一个新服务配置（json 字符串）
	NewServiceConfig(serviceConfig string)
}

// Target represents a target for gRPC, as specified in:
// https://github.com/grpc/grpc/blob/master/doc/naming.md.
// 对应的就是该详细设计的名称语法（如上面的 dns://8.8.8.8/www.qq.com）
type Target struct {
	Scheme    string
	Authority string
	Endpoint  string
}

// Builder creates a resolver that will be used to watch name resolution updates.
// 构建器接口，方便实现新的构建器。被用于监控名称解析更新
type Builder interface {
	// Build creates a new resolver for the given target.
	//
	// gRPC dial calls Build synchronously, and fails if the returned error is
	// not nil.
	// 构建自定义解析器，必须实现 Build 方法，比如下面一个小节分析的 dnsBuilder，就实现了这两个方法，
	// 一般在 Build() 方法中，会包含实现 watcher 的逻辑
	Build(target Target, cc ClientConn, opts BuildOption) (Resolver, error)
	// Scheme returns the scheme supported by this resolver.
	// Scheme is defined at https://github.com/grpc/grpc/blob/master/doc/naming.md.
	// 指定构建的是那种 Scheme 类型的解析器，就是客户端调用的 scheme 名字
	Scheme() string
}

// ResolveNowOption includes additional information for ResolveNow.
// 执行立即解析附属信息
type ResolveNowOption struct{}

// Resolver watches for the updates on the specified target.
// Updates include address updates and service config updates.
// 如果要实现自定义的解析器，必须实现下面这两个方法（解析器监视指定目标上的更新。更新内容包括已解决的地址列表和一个服务配置）
type Resolver interface {
	// ResolveNow will be called by gRPC to try to resolve the target name again.
	// It's just a hint, resolver can ignore this if it's not necessary.
	ResolveNow(ResolveNowOption)		// 通过 channel 方式，唤醒 select，强制执行立即解析
	// Close closes the resolver.
	Close()		// 关闭解析器
}

// UnregisterForTesting removes the resolver builder with the given scheme from the
// resolver map.
// This function is for testing only.
func UnregisterForTesting(scheme string) {
	delete(m, scheme)
}
```

##  0x03	dns_resolver.go
[dns_resolver.go](https://github.com/grpc/grpc-go/blob/master/resolver/dns/dns_resolver.go) 是 gRPC 提供的 DNS 解析器实现，这里的 `DNS` 解析器和先前我在项目中实现的 `Consul`、`Etcd` 的解析器相比，最大的不同，是 `DNS` 必须以轮询方式去请求获取 `Endpoint` 的地址列表。下面的代码中的 `watcher()` 方法有所体现：
``` go
// Package dns implements a dns resolver to be installed as the default resolver
// in grpc.
func init() {
	resolver.Register(NewBuilder())
}

const (
	defaultPort = "443"
	defaultFreq = time.Minute * 30	 // 默认每 30 分钟解析一次
	golang      = "GO"
	// In DNS, service config is encoded in a TXT record via the mechanism
	// described in RFC-1464 using the attribute name grpc_config.
	// 使用 dns TXT 记录 grpc_config 属性发布服务配置，TXT 记录必须满足下面的格式（均衡器）
	txtAttribute = "grpc_config="
)

var errMissingAddr = errors.New("missing address")

// NewBuilder creates a dnsBuilder which is used to factory DNS resolvers.
func NewBuilder() resolver.Builder {
	return &dnsBuilder{freq: defaultFreq}
}

// dns 解析器构建结构，freq 为轮询 DNS 权威服务器的周期
type dnsBuilder struct {
	// frequency of polling the DNS server.
	freq time.Duration
}

// Build creates and starts a DNS resolver that watches the name resolution of the target.
// 要构建自己的负载均衡器，必须实现 Build 方法，该方法返回一个 resolver.Resolver，通常该方法中会开启一个子协程，用来 watch（or pull）服务地址的变化
// 这里先埋一个问题，resolver 的 Builder 是在哪里调用的？
func (b *dnsBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOption) (resolver.Resolver, error) {
	host, port, err := parseTarget(target.Endpoint)
	if err != nil {
		return nil, err
	}

	// IP address.--IP 地址
	if net.ParseIP(host) != nil {
		host, _ = formatIP(host)
		addr := []resolver.Address{{Addr: host + ":" + port}}
		i := &ipResolver{		//ipResolver
			cc: cc,
			ip: addr,
			rn: make(chan struct{}, 1),
			q:  make(chan struct{}),
		}
		cc.NewAddress(addr)
		go i.watcher()
		return i, nil
	}

	// DNS address (non-IP). -- 域名方式（与 IP 是区分开的）
	ctx, cancel := context.WithCancel(context.Background())
	d := &dnsResolver{
		freq:   b.freq,
		host:   host,
		port:   port,
		ctx:    ctx,
		cancel: cancel,
		cc:     cc,
		t:      time.NewTimer(0),
		rn:     make(chan struct{}, 1),
	}

	d.wg.Add(1)
	go d.watcher()
	return d, nil
}

// Scheme returns the naming scheme of this resolver builder, which is "dns".
// 默认的 scheme
func (b *dnsBuilder) Scheme() string {
	return "dns"
}

// ipResolver watches for the name resolution update for an IP address.
type ipResolver struct {
	cc resolver.ClientConn
	ip []resolver.Address
	// rn channel is used by ResolveNow() to force an immediate resolution of the target.
	rn chan struct{}
	q  chan struct{}
}

// ResolveNow resend the address it stores, no resolution is needed.
func (i *ipResolver) ResolveNow(opt resolver.ResolveNowOption) {
	select {
	case i.rn <- struct{}{}:
	default:
	}
}

// Close closes the ipResolver.
func (i *ipResolver) Close() {
	close(i.q)
}

func (i *ipResolver) watcher() {
	for {
		select {
		case <-i.rn:
			i.cc.NewAddress(i.ip)
		case <-i.q:
			return
		}
	}
}

// dnsResolver watches for the name resolution update for a non-IP target.
// dnsResolver，监视 P 目标的名称解析更新，并通过 channel 通知上层 gRPC 的接口实时更新, 实现 grpc-lb 需要实现此结构
type dnsResolver struct {
	freq   time.Duration		// 解析周期（PULL）
	host   string				// 待解析的域名（如 www.qq.com）
	port   string

	// 上下文
	ctx    context.Context
	// 上下文取消函数，被用于取消解析过程
	cancel context.CancelFunc
	cc     resolver.ClientConn		//cc（resolver.ClientConn） is a interface{}，待通知的客户端连接
	// rn channel is used by ResolveNow() to force an immediate resolution of the target.
	// rn 该 channel 被用于强制触发 ResolveNow() 的立即执行
	rn chan struct{}

	// 定时器，被用于控制解析频率
	t  *time.Timer
	// wg is used to enforce Close() to return after the watcher() goroutine has finished.
	// Otherwise, data race will be possible. [Race Example] in dns_resolver_test we
	// replace the real lookup functions with mocked ones to facilitate testing.
	// If Close() doesn't wait for watcher() goroutine finishes, race detector sometimes
	// will warns lookup (READ the lookup function pointers) inside watcher() goroutine
	// has data race with replaceNetFunc (WRITE the lookup function pointers).
	// 被用于等待 watcher() 协程执行完毕后再强制 Close()，否则会出现 watcher() 协程和 replaceNetFunc 函数间数据竞争问题
	wg sync.WaitGroup
}

// ResolveNow invoke an immediate resolution of the target that this dnsResolver watches.
// 往 rn 发送执行立即解析信号（这里多说两句：这个 channel 用法是 golang 中很常见的技巧，注意看 watcher() 中的 select 分支，有一个 case <-d.rn: 条件，
// 当调用 ResolveNow 时，会立即触发执行 DNS 解析，这里的 default 是防止 channel 写入阻塞）
func (d *dnsResolver) ResolveNow(opt resolver.ResolveNowOption) {
	select {
	case d.rn <- struct{}{}:
	default:
	}
}

// Close closes the dnsResolver.
// 关闭解析器
func (d *dnsResolver) Close() {
	// 取消解析过程，取消成功后 ctx 会收到 Done() 信号
	d.cancel()

	// 等待所有协程退出（等待 watcher() 协程执行完毕）
	d.wg.Wait()

	// 停止定时器，不再发送时钟信号
	d.t.Stop()
}

// gRPC-lb 功能的第三核心实现：watcher（一般以一个单独的子 routine 创建）
func (d *dnsResolver) watcher() {
	defer d.wg.Done()		// 协程退出， 通知 watcher() 协程执行完毕
	for {
		select {
		case <-d.ctx.Done():
			return
		case <-d.t.C:		// 定时器
		case <-d.rn:		// 阻塞等待时钟信号和立即执行信号
		}
		result, sc := d.lookup()	// 主动调用 DNS 解析流程，start DNS resovle
		// Next lookup should happen after an interval defined by d.freq.
		// 重置定时器
		d.t.Reset(d.freq)
		d.cc.NewServiceConfig(string(sc))	// 发送服务配置通知
		d.cc.NewAddress(result)				// 发送地址列表通知 gRPC 更新（后端 endpoint 列表），NewAddress 已经被 UpdateState 方法代替了
	}
}


// 查找 SRV 记录，用于查询到的均衡器地址列表
func (d *dnsResolver) lookupSRV() []resolver.Address {
	var newAddrs []resolver.Address
	//1. 先根据 SRV 记录，获取 grpclb 的记录（获取所有负载均衡节点名称）
	_, srvs, err := lookupSRV(d.ctx, "grpclb", "tcp", d.host)
	if err != nil {
		grpclog.Infof("grpc: failed dns SRV record lookup due to %v.\n", err)
		return nil
	}

	// 2. 根据上一步获取的均衡节点名称，查找对应的负载均衡器地址列表（A 记录）
	for _, s := range srvs {
		lbAddrs, err := lookupHost(d.ctx, s.Target)
		if err != nil {
			grpclog.Warningf("grpc: failed load banlacer address dns lookup due to %v.\n", err)
			continue
		}
		for _, a := range lbAddrs {
			a, ok := formatIP(a)
			if !ok {
				grpclog.Errorf("grpc: failed IP parsing due to %v.\n", err)
				continue
			}
			addr := a + ":" + strconv.Itoa(int(s.Port))
			newAddrs = append(newAddrs, resolver.Address{Addr: addr, Type: resolver.GRPCLB, ServerName: s.Target})
		}
	}
	return newAddrs
}

//	查找 TXT 解析记录，返回查询到的服务配置（注意必须以此为前缀：txtAttribute = "grpc_config="）
func (d *dnsResolver) lookupTXT() string {
	ss, err := lookupTXT(d.ctx, d.host)
	if err != nil {
		grpclog.Warningf("grpc: failed dns TXT record lookup due to %v.\n", err)
		return ""
	}
	var res string
	for _, s := range ss {
		res += s
	}

	// TXT record must have "grpc_config=" attribute in order to be used as service config.
	if !strings.HasPrefix(res, txtAttribute) {
		grpclog.Warningf("grpc: TXT record %v missing %v attribute", res, txtAttribute)
		return ""
	}
	return strings.TrimPrefix(res, txtAttribute)
}

// 查找 DNS 的 A 记录，用于返回查询到的后台服务器地址列表
func (d *dnsResolver) lookupHost() []resolver.Address {
	var newAddrs []resolver.Address
	// 调用 net 的函数 lookupHost 发起 DNS 查询
	addrs, err := lookupHost(d.ctx, d.host)
	if err != nil {
		grpclog.Warningf("grpc: failed dns A record lookup due to %v.\n", err)
		return nil
	}
	for _, a := range addrs {
		//PS：注意可能解析出有 IPV6 的地址
		a, ok := formatIP(a)
		if !ok {
			grpclog.Errorf("grpc: failed IP parsing due to %v.\n", err)
			continue
		}
		addr := a + ":" + d.port
		newAddrs = append(newAddrs, resolver.Address{Addr: addr})
	}
	return newAddrs
}

// DNS 解析与获取结果，返回地址列表（服务器地址和负载均衡器地址）和服务配置
func (d *dnsResolver) lookup() ([]resolver.Address, string) {
	var newAddrs []resolver.Address
	// 查找均衡器地址列表
	newAddrs = d.lookupSRV()
	// Support fallback to non-balancer address.
	newAddrs = append(newAddrs, d.lookupHost()...)	// 追加后台服务器地址列表
	sc := d.lookupTXT()
	return newAddrs, canaryingSC(sc)
}

// formatIP returns ok = false if addr is not a valid textual representation of an IP address.
// If addr is an IPv4 address, return the addr and ok = true.
// If addr is an IPv6 address, return the addr enclosed in square brackets and ok = true.
func formatIP(addr string) (addrIP string, ok bool) {
	ip := net.ParseIP(addr)
	if ip == nil {
		return "", false
	}
	if ip.To4() != nil {
		return addr, true
	}
	return "[" + addr + "]", true
}

// parseTarget takes the user input target string, returns formatted host and port info.
// If target doesn't specify a port, set the port to be the defaultPort.
// If target is in IPv6 format and host-name is enclosed in sqarue brackets, brackets
// are strippd when setting the host.
// examples:
// target: "www.google.com" returns host: "www.google.com", port: "443"
// target: "ipv4-host:80" returns host: "ipv4-host", port: "80"
// target: "[ipv6-host]" returns host: "ipv6-host", port: "443"
// target: ":80" returns host: "localhost", port: "80"
// target: ":" returns host: "localhost", port: "443"
func parseTarget(target string) (host, port string, err error) {
	if target == "" {
		return "","", errMissingAddr
	}
	if ip := net.ParseIP(target); ip != nil {
		// target is an IPv4 or IPv6(without brackets) address
		return target, defaultPort, nil
	}
	if host, port, err = net.SplitHostPort(target); err == nil {
		// target has port, i.e ipv4-host:port, [ipv6-host]:port, host-name:port
		if host == "" {
			// Keep consistent with net.Dial(): If the host is empty, as in ":80", the local system is assumed.
			host = "localhost"
		}
		if port == "" {
			// If the port field is empty(target ends with colon), e.g. "[::1]:", defaultPort is used.
			port = defaultPort
		}
		return host, port, nil
	}
	if host, port, err = net.SplitHostPort(target + ":" + defaultPort); err == nil {
		// target doesn't have port
		return host, port, nil
	}
	return "","", fmt.Errorf("invalid target address %v, error info: %v", target, err)
}

```

##	0x05	DnsResolver 的应用
在项目中，`DnsResolver` 与 [CoreDNS](https://github.com/coredns/coredns) 搭配是一个不错的选择，不过需要注意的是，解析 `DNS` 的时间，`DnsResolver` 中默认是 `30` 分钟，个人感觉可以优化下。

笔者的项目部署在 TKE 上，使用 CoreDNS 作为服务发现媒介，gRPC 使用 `DnsResolver` 作为解析器：

集群的 CoreDNS 部署情况，通过 Deployment 方式部署，分配的集群内 IP 地址为：`172.16.0.3` 和 `172.16.1.2`：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/dnsresolver/coredns-1.png)

通过 `nslookup` 查询下服务名字（服务必须以 Headless-Service 方式部署）的解析情况：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/dnsresolver/coredns-ns-lookup.png)

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/dnsresolver/coredns-ns-lookup2.png)

gRPC resovler 的连接池建立情况：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2019/dnsresolver/coredns-grpc-tcp-long-conn.png)


##	0x06	总结
至此，`gRPC` 默认的 `DNS` 解析器主要源码就分析完成了。不过，由于 `DNS` 本身无法感知后端健康状态的问题，所以在实战中如何剔除掉不健康的后端，是使用 `DNS` 作为负载均衡手段时需要考虑的问题；另外 `DNS` 有 `TTL` 这个特性的存在，在多层次 `DNS` 架构中，可能也会成为服务治理的一个难题。

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权