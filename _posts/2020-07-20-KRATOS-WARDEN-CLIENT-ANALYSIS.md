---
layout:     post
title:      Kratos 源码分析：Warden 之 gRPC-Client 封装
subtitle:   分析 Warden 的 Client 端封装
date:       2020-07-20
author:     pandaychen
header-img:	img/golang-tools-fun.png
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
Kratos 的 Warden 框架 [client.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go) 封装了 gRPC 的客户端启动的核心逻辑。
客户端封装的重点在下面几个方面：
-   客户端启动 & 配置流程
-   客户端拦截器链的 "安装" 顺序
-   `tracer`、`metrics` 以及 `breaker` 与 `grpc.Client` 的结合
-	客户端的 `Naming` 和 gRPC 框架结合
-   客户端调用的 LoadBalance 算法

上面列举的条目中，注意，Warden 的 Server 端默认开启了 `limiter`、而客户端默认开启的是 `breaker`、当然也可以加入 `limiter`

##  0x01    Client 端分析
`warden.Client` 结构如下，默认封装了熔断器 `breaker *breaker.Group` 及拦截器数组 `handlers []grpc.UnaryClientInterceptor`：
```golang
// Client is the framework's client side instance, it contains the ctx, opt and interceptors.
// Create an instance of Client, by using NewClient().
type Client struct {
	conf    *ClientConfig
	breaker *breaker.Group
	mutex   sync.RWMutex

	opts     []grpc.DialOption
	handlers []grpc.UnaryClientInterceptor
}
```

####    客户端初始化
Warden 提供了外部调用的方法 `NewClient`，用来创建 `warden.Client` 结构，注意默认开启的 LB 算法是 `P2C`，初始化主要为了设置 Client 端的 options：
```golang
// NewClient returns a new blank Client instance with a default client interceptor.
// opt can be used to add grpc dial options.
func NewClient(conf *ClientConfig, opt ...grpc.DialOption) *Client {
	c := new(Client)
	if err := c.SetConfig(conf); err != nil {
		panic(err)
	}
	c.UseOpt(grpc.WithBalancerName(p2c.Name))
	c.UseOpt(opt...)
	return c
}
```

####	客户端 Dial/DialTLS
客户端拦截器的初始化，发起连接等操作都集成在了 [`client.dial()` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go#L273) 中，主要流程如下：
1.	判断是否以阻塞方式调用，`grpc.WithBlock()`
2.	检查是否需要客户端的 keepalive 配置
3.	初始化客户端的拦截器链，按照执行顺序为，`c.recovery()`--->`clientLogging(dialOptions...)`--->`c.handlers()`，注意 `c.handlers()` 必须放在最后，为什么？
4.	解析传入配置，获取 `Dial` 的 scheme 地址（一般有直连方式、DNS 和注册中心这几种）
5.	发起真正的 `Dial` 操作

```golang
// Dial creates a client connection to the given target.
// Target format is scheme://authority/endpoint?query_arg=value
// example: discovery://default/account.account.service?cluster=shfy01&cluster=shfy02
func (c *Client) Dial(ctx context.Context, target string, opts ...grpc.DialOption) (conn *grpc.ClientConn, err error) {
	opts = append(opts, grpc.WithInsecure())
	return c.dial(ctx, target, opts...)
}

func (c *Client) dial(ctx context.Context, target string, opts ...grpc.DialOption) (conn *grpc.ClientConn, err error) {
	dialOptions := c.cloneOpts()
	if !c.conf.NonBlock {
		// 是否以阻塞方式调用，常用于服务发现模式
		dialOptions = append(dialOptions, grpc.WithBlock())
	}
	// 客户端的 keepalive 配置
	dialOptions = append(dialOptions, grpc.WithKeepaliveParams(keepalive.ClientParameters{
		Time:                time.Duration(c.conf.KeepAliveInterval),
		Timeout:             time.Duration(c.conf.KeepAliveTimeout),
		PermitWithoutStream: !c.conf.KeepAliveWithoutStream,
	}))
	dialOptions = append(dialOptions, opts...)

	// 初始化默认拦截器
	var handlers []grpc.UnaryClientInterceptor
	handlers = append(handlers, c.recovery())
	handlers = append(handlers, clientLogging(dialOptions...))
	handlers = append(handlers, c.handlers...)
	// NOTE: c.handle must be a last interceptor.
	handlers = append(handlers, c.handle())

	// 拦截器链传入
	dialOptions = append(dialOptions, grpc.WithUnaryInterceptor(chainUnaryClient(handlers)))
	c.mutex.RLock()
	conf := c.conf
	c.mutex.RUnlock()
	if conf.Dial > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, time.Duration(conf.Dial))
		defer cancel()
	}
	if u, e := url.Parse(target); e == nil {
		v := u.Query()
		for _, c := range c.conf.Clusters {
			v.Add(naming.MetaCluster, c)
		}
		if c.conf.Zone != "" {
			v.Add(naming.MetaZone, c.conf.Zone)
		}
		if v.Get("subset") == "" && c.conf.Subset > 0 {
			v.Add("subset", strconv.FormatInt(int64(c.conf.Subset), 10))
		}
		u.RawQuery = v.Encode()
		// 比较_grpcTarget 中的 appid 是否等于 u.path 中的 appid，并替换成 mock 的地址
		for _, t := range _grpcTarget {
			strs := strings.SplitN(t, "=", 2)
			if len(strs) == 2 && ("/"+strs[0]) == u.Path {
				u.Path = "/" + strs[1]
				u.Scheme = "passthrough"
				u.RawQuery = ""
				break
			}
		}
		target = u.String()
	}

	//Do real dial action
	if conn, err = grpc.DialContext(ctx, target, dialOptions...); err != nil {
		fmt.Fprintf(os.Stderr, "warden client: dial %s error %v!", target, err)
	}
	err = errors.WithStack(err)
	return
}
```

Warden 客户端拦截器链的实现代码如下，在 [Kratos 源码分析：gRPC-Warden 拦截器（链）及实现](https://pandaychen.github.io/2020/05/30/KRATOS-INTERCEPTOR-ANALYSIS/) 一文中已经分析过，主要关注 < font color="#dd0000"> 拦截器链的执行顺序以及 context 在拦截器链中的传递 </font>，注释中也给出了：

> Execution is done in left-to-right order, including passing of context.
> For example ChainUnaryClient(one, two, three) will execute one before two before three.

```golang
// chainUnaryClient creates a single interceptor out of a chain of many interceptors.
//
// Execution is done in left-to-right order, including passing of context.
// For example ChainUnaryClient(one, two, three) will execute one before two before three.
func chainUnaryClient(handlers []grpc.UnaryClientInterceptor) grpc.UnaryClientInterceptor {
	n := len(handlers)
	if n == 0 {
		return func(ctx context.Context, method string, req, reply interface{},
			cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
			return invoker(ctx, method, req, reply, cc, opts...)
		}
	}

	return func(ctx context.Context, method string, req, reply interface{},
		cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		var (
			i            int
			chainHandler grpc.UnaryInvoker
		)
		chainHandler = func(ictx context.Context, imethod string, ireq, ireply interface{}, ic *grpc.ClientConn, iopts ...grpc.CallOption) error {
			if i == n-1 {
				return invoker(ictx, imethod, ireq, ireply, ic, iopts...)
			}
			// 从 i 为 0 开始，及从拦截器数组的左边遍历到右边
			i++
			return handlers[i](ictx, imethod, ireq, ireply, ic, chainHandler, iopts...)
		}

		return handlers[0](ctx, method, req, reply, cc, chainHandler, opts...)
	}
}
```

##	0x02	客户端拦截器分析

![warden-client-interceptor-chain]()

####	对外接口 Use
Warden 的客户端对外暴露了 [`Use()` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go#L249)，用于添加自定义的客户端拦截器。
`Use` 方法将参数中的 `handlers` 拦截器添加到 `c.handlers` 数组的末尾位置：
```golang
// Use attachs a global inteceptor to the Client.
// For example, this is the right place for a circuit breaker or error management inteceptor.
func (c *Client) Use(handlers ...grpc.UnaryClientInterceptor) *Client {
	finalSize := len(c.handlers) + len(handlers)
	if finalSize >= int(_abortIndex) {
		panic("warden: client use too many handlers")
	}
	mergedHandlers := make([]grpc.UnaryClientInterceptor, finalSize)
	copy(mergedHandlers, c.handlers)
	copy(mergedHandlers[len(c.handlers):], handlers)
	c.handlers = mergedHandlers
	return c
}
```

####	Client.handle 默认拦截器
在 Warden 的客户端中，将 OpenTracing 和熔断器 Breaker 默认封装在 `Client.handle()` 中：
```golang
// handle returns a new unary client interceptor for OpenTracing\Logging\LinkTimeout.
func (c *Client) handle() grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) (err error) {
		var (
			ok     bool
			t      trace.Trace
			gmd    metadata.MD
			conf   *ClientConfig
			cancel context.CancelFunc
			addr   string
			p      peer.Peer
		)
		var ec ecode.Codes = ecode.OK
		// apm tracing
		if t, ok = trace.FromContext(ctx); ok {
			t = t.Fork("", method)
			defer t.Finish(&err)
		}

		// setup metadata
		gmd = baseMetadata()
		trace.Inject(t, trace.GRPCFormat, gmd)
		c.mutex.RLock()
		if conf, ok = c.conf.Method[method]; !ok {
			conf = c.conf
		}
		c.mutex.RUnlock()
		brk := c.breaker.Get(method)
		if err = brk.Allow(); err != nil {
			_metricClientReqCodeTotal.Inc(method, "breaker")
			return
		}
		defer onBreaker(brk, &err)
		var timeOpt *TimeoutCallOption
		for _, opt := range opts {
			var tok bool
			timeOpt, tok = opt.(*TimeoutCallOption)
			if tok {
				break
			}
		}
		if timeOpt != nil && timeOpt.Timeout > 0 {
			ctx, cancel = context.WithTimeout(nmd.WithContext(ctx), timeOpt.Timeout)
		} else {
			_, ctx, cancel = conf.Timeout.Shrink(ctx)
		}

		defer cancel()
		nmd.Range(ctx,
			func(key string, value interface{}) {
				if valstr, ok := value.(string); ok {
					gmd[key] = []string{valstr}
				}
			},
			nmd.IsOutgoingKey)
		// merge with old matadata if exists
		if oldmd, ok := metadata.FromOutgoingContext(ctx); ok {
			gmd = metadata.Join(gmd, oldmd)
		}
		ctx = metadata.NewOutgoingContext(ctx, gmd)

		opts = append(opts, grpc.Peer(&p))
		if err = invoker(ctx, method, req, reply, cc, opts...); err != nil {
			gst, _ := gstatus.FromError(err)
			ec = status.ToEcode(gst)
			err = errors.WithMessage(ec, gst.Message())
		}
		if p.Addr != nil {
			addr = p.Addr.String()
		}
		if t != nil {
			t.SetTag(trace.String(trace.TagAddress, addr), trace.String(trace.TagComment, ""))
		}
		return
	}
}
```


##  0x03	参考
-   [client.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go)