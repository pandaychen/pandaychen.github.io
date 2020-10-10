---
layout:     post
title:      Kratos 源码分析：CGI 框架 BM （一）
subtitle:   分析基于 gin 改造的框架 blademaster：基础
date:       2020-10-01
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---


##  0x00    前言
[blademaster](https://github.com/go-kratos/kratos/tree/master/pkg/net/http/blademaster) 是 Kratos 提供的 HTTP 框架，参考 [gin](https://github.com/gin-gonic/gin) 实现，但剥离了 gin 中的若干代码。其官方介绍是：基于 gin 二次开发，具有快速、灵活的特点，可以方便的开发中间件处理通用或特殊逻辑，基础库默认实现了 log&&trace 等。

##  0x01    Web 框架的特点
就开发经验而言，一个 CGI 框架可能需要满足如下特性（便于开发上手和功能扩展）：

-   多路由（Multiplexer Router）支持
-   方便封装自己的中间件
-   性能
-   方便嵌入业务逻辑
-   请求参数校验，CGI 签名等框架的安全特性
-   核心指标监控及稳定

而 gin 框架就是满足如上特性的一款十分优秀的 Web 框架，blademaster 正是由 gin 精简而成。

##  0x02    blademaster 设计思路
本小节源自 blademaster 的 [官方文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster.md)。<br><br>

在像微服务这样的分布式架构中，经常会有一些需求需要你调用多个服务，但是还需要确保服务的安全性、统一化每次的请求日志或者追踪用户完整的行为等等。要实现这些功能，你可能需要在所有服务中都设置一些相同的属性，虽然这个可以通过一些明确的接入文档来描述或者准入规范来界定，但是这么做的话还是有可能会有一些问题：

1. 你很难让每一个服务都实现上述功能。因为对于开发者而言，他们应当注重的是实现功能。很多项目的开发者经常在一些日常开发中遗漏了这些关键点，经常有人会忘记去打日志或者去记录调用链。但是对于一些大流量的互联网服务而言，一个线上服务一旦发生故障时，即使故障时间很小，其影响面会非常大。一旦有人在关键路径上忘记路记录日志，那么故障的排除成本会非常高，那样会导致影响面进一步扩大。
2. 事实上实现之前叙述的这些功能的成本也非常高。比如说对于鉴权（Identify）这个功能，你要是去一个服务一个服务地去实现，那样的成本也是非常高的。如果说把这个确保认证的责任分担在每个开发者身上，那样其实也会增加大家遗忘或者忽略的概率。

为了解决这样的问题，你可能需要一个框架来帮助你实现这些功能。比如说帮你在一些关键路径的请求上配置必要的鉴权或超时策略。那样服务间的调用会被多层中间件所过滤并检查，确保整体服务的稳定性。

####    blademaster 的设计目标

*   性能优异，不应该掺杂太多业务逻辑的成分
*   方便开发使用，开发对接的成本应该尽可能地小
*   后续鉴权、认证等业务逻辑的模块应该可以通过业务模块的开发接入该框架内
*   默认配置已经是 production ready 的配置，减少开发与线上环境的差异性

####    blademaster 的概览

*   参考 gin 设计整套 HTTP 框架，去除 gin 中不需要的部分逻辑
*   内置一些必要的中间件，便于业务方可以直接上手使用

####    blademaster 架构简介
一般 http 的主要处理流程如下：
![img](https://wx1.sbimg.cn/2020/08/17/3sFae.png)

blademaster 的主要流程也是如此：
![img](https://wx2.sbimg.cn/2020/08/17/3sVgO.png)

从上图来看，blademaster 由几个非常精简的内部模块组成：

*   `Router`：用于根据请求的路径分发请求
*   [`Context`](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/context.go#L35)：包含了一个完整的请求信息（封装了 `net.http` 的 `http.Request` 和 `http.ResponseWriter`），一个 `Context` 理解为一次 HTTP 请求的生命周期
*   `Handler` 则负责处理传入的 `Context`，`Handlers` 为一个列表，一个串一个地执行

所有的 `middlerware` 均以 `Handler` 的形式存在，这样可以保证 blademaster 自身足够精简且扩展性足够强。

![img](https://wx1.sbimg.cn/2020/08/17/3syp6.png)
上图描述了 blademaster 处理请求的模式及路径，大部分的逻辑都被封装在了各种 `Handler` 中。一般而言，<font color="#dd0000"> 业务逻辑作为最后一个 `Handler`</font>

正常情况下每个 `Handler` 按照顺序一个一个串行地执行下去，但是 `Handler` 中也可以中断整个处理流程，直接输出 `Response`。这种模式常被用于校验登陆、过载保护（如熔断、限流等机制）的 `middleware` 中：一旦发现请求不合法，直接响应拒绝。

请求处理的流程中也可以使用 `Render` 来辅助渲染 `Response`，比如对于不同的请求需要响应不同的数据格式 `JSON`、`XML`，此时可以使用不同的 `Render` 来简化逻辑。


####    代码组织
blademaster 的主要代码组织及功能简介如下：
*   [server.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go)：核心 server 结构定义
*   [routergroup.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/routergroup.go)：HTTP 路由操作实现
*   [tree.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/tree.go)：CGI 目录树的算法实现
*   [trace.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/trace.go)：加入链路追踪的逻辑
*   [recovery.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/recovery.go)：core 处理中间件，简化版
*   [ratelimit.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/ratelimit.go)：限流器中间件，采用 BBR 限流
*   [prometheus.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/prometheus.go)：Prometheus-httpserver
*   [perf.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/perf.go)：性能统计中间件
*   [metrics.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/metrics.go)：metrics 定义
*   [metadata.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/metadata.go)：cgi 的元数据定义
*   [logger.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/logger.go)：日志中间件，记录核心日志
*   [cors.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/cors.go)：跨域组件

此外，blademaster 还提供了 HTTP 的 client 封装：位于此 [client.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/client.go)，客户端默认加载了熔断器 breaker。


##  0x03    分析流程
本系列准备按照如下流程来分析整个 bm 框架的实现：
1.  核心数据结构：基于 `http.Server` 的包装及改造，`Engine`、`Context`、`Route` 等
2.  HTTP 从 Request 处理到 Response 完成的整体流程：当请求到达时，gin 如何处理请求；在处理完成后，如何将响应数据返回给请求方
3.  路由处理：Radix 树
4.  拦截器（中间件）：洋葱模式
5.  `binding` 及参数校验
6.  监听请求：gin 是如何开启服务及监听请求的
7.  Web 框架收集监控的指标（Metrics）
8.  HTTP-Tracing
9.	bm 框架提供的客户端封装

##  0x04    从 net.http 开始

####	http.Handler 接口

回顾下 golang 的 `net.http` 包中的核心 `handler Handler` 接口，这里的 `Handler` 是 `interface{}`，其中包含（定义）了 `ServeHTTP(ResponseWriter, *Request)` 方法：
```golang
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}

type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

从此来看，`Handler` 就是封装的入口，bm 中是由 `Engine` 类实现了此 `interface{}`，即实现了 [`ServeHTTP` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go)，这里 `ServeHTTP` 方法传递的两个参数，一个是 `Request`，一个是 `ResponseWriter`，`Engine` 中的 `ServeHTTP` 方法实现需要对这两个对象进行 Read 或者 Write 操作。<br>

```golang
type Engine struct {
	......
}

// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	// 注意：每一次 http 请求，都会调用 ServeHTTP 方法
	c := engine.pool.Get().(*Context)
	c.Request = req
	c.Writer = w
	c.reset()

	engine.handleContext(c)
	engine.pool.Put(c)
}
```

####	Context 结构
在 `net.http` 的实现中，上面这两个对象 `Request` 和 `ResponseWriter` 往往是需要同时存在的，为了避免很多函数都需要写这两个参数，我们不如封装一个结构来把这两个对象放在里面，于是就引出了 [`Context` 结构](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/context.go)<br>

一个 `Context` 本质就是对一次 HTTP 的请求响应过程的封装。

```golang
// Context is the most important part. It allows us to pass variables between
// middleware, manage the flow, validate the JSON of a request and render a
// JSON response for example.
type Context struct {
	context.Context

	Request *http.Request
	Writer  http.ResponseWriter
	......
}
```

接下来，bm 框架的实现就围绕着 `Context` 及数据处理流程来进行了。

##	0x05	Bm 框架的核心结构

####	Engine



####	Context
[`Context` 结构](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/context.go#L35) 定义如下，从结构不难看出，`Context` 主要完成了如下工作：
-	超时传递及控制，HTTP 请求周期中的 KeyValue 管理
-	HTTP 请求的处理
-	流程控制（拦截器链在 HTTP 请求中的应用、`abort` 等）

```golang
// Context is the most important part. It allows us to pass variables between
// middleware, manage the flow, validate the JSON of a request and render a
// JSON response for example.
type Context struct {
	context.Context

	Request *http.Request
	Writer  http.ResponseWriter

	// flow control
	index    int8
	handlers []HandlerFunc

	// Keys is a key/value pair exclusively for the context of each request.
	Keys map[string]interface{}
	// This mutex protect Keys map
	keysMutex sync.RWMutex

	Error error

	method string
	engine *Engine
	RoutePath string
	Params Params
}
```

##  0x05    Bm 框架的指标 Metrics
blademaster 默认提供了如下几个维度的 metrics[监控采集数据](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/metrics.go)：
-   `_metricServerReqDur `：`NewHistogramVec` 类型，含义是服务端处理请求的延迟范围（区间）：`http server requests duration(ms)`，统计维度为 `path/caller/method`，单位为 `ms`
-	`_metricServerReqCodeTotal`：

####	Metrics 的一个细节
bm 在初始化时默认注册了 [`metrics` 路由](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go#L164)：
```golang
// NewServer returns a new blank Engine instance without any middleware attached.
func NewServer(conf *ServerConfig) *Engine {
	......
	engine.RouterGroup.engine = engine
	// NOTE add prometheus monitor location
	engine.addRoute("GET", "/metrics", monitor())
	engine.addRoute("GET", "/metadata", engine.metadata())
	......
	return engine
}
```

`monitor()` 的 [实现如下](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/prometheus.go)：
```golang
func monitor() HandlerFunc {
	return func(c *Context) {
		h := promhttp.Handler()
		h.ServeHTTP(c.Writer, c.Request)
	}
}
```

注意到 `monitor()` 中 `return` 的是一个方法，该方法的参数为 `Context`，这个 `Context` 在哪里初始化传入以及传入何种数据呢？再看 [`engine.addRoute` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go#L164)：<br>

看到 `prelude`：
```golang
func (engine *Engine) addRoute(method, path string, handlers ...HandlerFunc) {
	if path[0] != '/' {
		panic("blademaster: path must begin with'/'")
	}
	if method == "" {
		panic("blademaster: HTTP method can not be empty")
	}
	if len(handlers) == 0 {
		panic("blademaster: there must be at least one handler")
	}
	if _, ok := engine.metastore[path]; !ok {
		engine.metastore[path] = make(map[string]interface{})
	}
	engine.metastore[path]["method"] = method
	root := engine.trees.get(method)
	if root == nil {
		root = new(node)
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}

	prelude := func(c *Context) {
		c.method = method
		c.RoutePath = path
	}
	handlers = append([]HandlerFunc{prelude}, handlers...)
	root.addRoute(path, handlers)
}
```

##  0x06    核心结构
本篇先从 [server.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go) 入手，分析下 blademaster 的核心数据结构：




##  0x07    参考
-   [blademaster 官方文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster.md)
-   [blademaster](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster-mid.md)
-   [go 语言框架 gin 的中文文档](https://github.com/skyhee/gin-doc-cn)
