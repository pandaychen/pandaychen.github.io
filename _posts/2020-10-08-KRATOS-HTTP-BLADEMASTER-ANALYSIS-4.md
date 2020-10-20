---
layout:     post
title:      Kratos 源码分析：CGI 框架 BM （三）
subtitle:   分析基于 gin 改造的框架 blademaster：模块
date:       2020-10-08
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---


##  0x00    代码组织
blademaster 的代码组织及功能简介如下：
*   [server.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go)：核心 server 结构定义
*   [routergroup.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/routergroup.go)
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


##  Metrics
[Metrics](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/metrics.go)：定义了如下全局变量
-   `_metricServerReqDur`：`metric.NewHistogramVec` 类型
-   `_metricServerReqCodeTotal`：`metric.NewCounterVec` 类型
-   `_metricServerBBR`
-   `_metricClientReqDur`
-   `_metricClientReqCodeTotal`


##
启动的 [代码](https://github.com/bilibili/kratos-demo/blob/master/internal/server/http/server.go)：
```golang
import (
	"net/http"

	pb "kratos-demo/api"
	"kratos-demo/internal/model"
	"github.com/bilibili/kratos/pkg/conf/paladin"
	"github.com/bilibili/kratos/pkg/log"
	bm "github.com/bilibili/kratos/pkg/net/http/blademaster"
)

var svc pb.DemoServer

// New new a bm server.
func New(s pb.DemoServer) (engine *bm.Engine, err error) {
	var (
		cfg bm.ServerConfig
		ct paladin.TOML
	)
	if err = paladin.Get("http.toml").Unmarshal(&ct); err != nil {
		return
	}
	if err = ct.Get("Server").UnmarshalTOML(&cfg); err != nil {
		return
	}
	svc = s
	engine = bm.DefaultServer(&cfg)
	pb.RegisterDemoBMServer(engine, s)
	initRouter(engine)
	err = engine.Start()
	return
}

func initRouter(e *bm.Engine) {
	e.Ping(ping)
	g := e.Group("/kratos-demo")
	{
		g.GET("/start", howToStart)
	}
}

func ping(ctx *bm.Context) {
	if _, err := svc.Ping(ctx, nil); err != nil {
		log.Error("ping error(%v)", err)
		ctx.AbortWithStatus(http.StatusServiceUnavailable)
	}
}

// example for http request handler.
func howToStart(c *bm.Context) {
	k := &model.Kratos{
		Hello: "Golang 大法好 !!!",
	}
	c.JSON(k, nil)
}
```

##  Engine
`Engine` 在 bm 框架中主要是封装了 `net.http` 库的 `http.Server`，然后将其扩展为一个更为通用的结构，`Engine` 本质上就是一个 HTTP 服务实例，其 [结构如下](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go#L117)，同时注意到 `Engine` 中包含了路由成员 `RouterGroup`：
```golang
// Engine is the framework's instance, it contains the muxer, middleware and configuration settings.
// Create an instance of Engine, by using New() or Default()
type Engine struct {
	RouterGroup         // 路由定义

	lock sync.RWMutex
	conf *ServerConfig

	address string

	trees     methodTrees
	server    atomic.Value                      // store *http.Server
	metastore map[string]map[string]interface{} // metastore is the path as key and the metadata of this path as value, it export via /metadata

	pcLock        sync.RWMutex
	methodConfigs map[string]*MethodConfig

	injections []injection

	// If enabled, the url.RawPath will be used to find parameters.
	UseRawPath bool

	// If true, the path value will be unescaped.
	// If UseRawPath is false (by default), the UnescapePathValues effectively is true,
	// as url.Path gonna be used, which is already unescaped.
	UnescapePathValues bool

	// If enabled, the router checks if another method is allowed for the
	// current route, if the current request can not be routed.
	// If this is the case, the request is answered with 'Method Not Allowed'
	// and HTTP status code 405.
	// If no other Method is allowed, the request is delegated to the NotFound
	// handler.
	HandleMethodNotAllowed bool

	allNoRoute  []HandlerFunc
	allNoMethod []HandlerFunc
	noRoute     []HandlerFunc
	noMethod    []HandlerFunc

	pool sync.Pool
}
```

`Engine.Run` 方法，注意，这里是把 `engine` 直接当做 `http.Server` 的 `Handler` 成员传入：
```golang
// Run attaches the router to a http.Server and starts listening and serving HTTP requests.
// It is a shortcut for http.ListenAndServe(addr, router)
// Note: this method will block the calling goroutine indefinitely unless an error happens.
func (engine *Engine) Run(addr ...string) (err error) {
	address := resolveAddress(addr)
	server := &http.Server{
		Addr:    address,
		Handler: engine,
	}
	engine.server.Store(server)
	if err = server.ListenAndServe(); err != nil {
		err = errors.Wrapf(err, "addrs: %v", addr)
	}
	return
}
```

##  Context
bm 框架中的 `Context` 代表了一次 HTTP 请求和响应的完全过程。


##  RouterGroup
`RouterGroup`结构是[`IRoutes`]((https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/routergroup.go#L14))这个`interface{}`的实例化结构：
```golang
// IRoutes http router interface.
type IRoutes interface {
	UseFunc(...HandlerFunc) IRoutes
	Use(...Handler) IRoutes

	Handle(string, string, ...HandlerFunc) IRoutes
	HEAD(string, ...HandlerFunc) IRoutes
	GET(string, ...HandlerFunc) IRoutes
	POST(string, ...HandlerFunc) IRoutes
	PUT(string, ...HandlerFunc) IRoutes
	DELETE(string, ...HandlerFunc) IRoutes
}

// RouterGroup is used internally to configure router, a RouterGroup is associated with a prefix
// and an array of handlers (middleware).
type RouterGroup struct {
	Handlers   []HandlerFunc    //存放全局拦截器
	basePath   string
	engine     *Engine          //属于哪个engine？
	root       bool
	baseConfig *MethodConfig
}
```

`RouterGroup`提供了全局拦截器的注册方法：
```golang
// Use adds middleware to the group, see example code in doc.
func (group *RouterGroup) Use(middleware ...Handler) IRoutes {
	for _, m := range middleware {
		group.Handlers = append(group.Handlers, m.ServeHTTP)
	}
	return group.returnObj()
}

// UseFunc adds middleware to the group, see example code in doc.
func (group *RouterGroup) UseFunc(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}
```

####    回顾下拦截器的定义及实现
这里先回顾下拦截器的[结构定义](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go#L62)与实现：
```golang
// Handler responds to an HTTP request.
type Handler interface {
	ServeHTTP(c *Context)
}

// HandlerFunc http request handler function.
type HandlerFunc func(*Context)

// ServeHTTP calls f(ctx).
func (f HandlerFunc) ServeHTTP(c *Context) {
	f(c)
}
```

以recovery及ratelimit的实现为例，`b.Limit`直接返回了一个`func(c *Context)`，也即`bm.HandlerFunc`，和上面的结构能对应起来：
```golang
// Limit return a bm handler func.
func (b *RateLimiter) Limit() HandlerFunc {
	return func(c *Context) {
		uri := fmt.Sprintf("%s://%s%s", c.Request.URL.Scheme, c.Request.Host, c.Request.URL.Path)
		limiter := b.group.Get(uri)
		done, err := limiter.Allow(c)
		if err != nil {
			_metricServerBBR.Inc(uri, c.Request.Method)
			c.JSON(nil, err)
			c.Abort()
			return
		}
		defer func() {
			done(limit.DoneInfo{Op: limit.Success})
			b.printStats(uri, limiter)
		}()
		c.Next()
	}
}
```



##  核心流程
`engine.Start()`(https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go#L89) -->


##  0x05    参考
-   [blademaster 官方文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster.md)
-   [blademaster](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster-mid.md)
-   [go 语言框架 gin 的中文文档](https://github.com/skyhee/gin-doc-cn)
