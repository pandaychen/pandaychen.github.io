---
layout:     post
title:      Kratos 源码分析：CGI 框架 BM （三）
subtitle:   分析 CGI 框架 blademaster：路由
date:       2020-10-03
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
本篇文章看下 bm 框架的路由注册及搭配拦截器的路由调用的实现细节。先看下路由是如何注册的。
bm 框架的默认生成 [模板](https://github.com/bilibili/kratos-demo/blob/master/internal/server/http/server.go)：

1.  bm 默认创建的 `Engine` 对象及启动
2.  `initRouter` 初始化路由的方法及路由（`Group`）的创建
3.  bm 的 handler 方法 `ping` 和 `howToStart` 的定义（ `handler` 方法默认传入 bm 的 `Context` 对象）

```golang
//bm 默认创建的 engine 及启动代码
engine = bm.DefaultServer(hc.Server)
initRouter(engine)
if err := engine.Start(); err != nil {
    panic(err)
}

func initRouter(e *bm.Engine) {
	e.Ping(ping) // engine 自带的 "/ping" 接口，用于负载均衡检测服务健康状态
	g := e.Group("/kratos-demo") // e.Group 创建一组 "/kratos-demo" 起始的路由组
	{
		g.GET("/start", howToStart) // g.GET 创建一个 "kratos-demo/start" 的路由，使用 GET 方式请求，默认处理 Handler 为 howToStart 方法
		g.POST("start", howToStart) // g.POST 创建一个 "kratos-demo/start" 的路由，使用 POST 方式请求，默认处理 Handler 为 howToStart 方法
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

此外，先预埋几个问题：
1.  路由注册的处理过程是如何？
2.  中间件（拦截器）是如何与路由注册发生交集的？
3.  常用到的 `c.Next()` 方法的作用是什么？
4.  路由存储的原理是什么？

##  0x01    bm 路由树结构

`Engine` 结构中包含了两个重要成员：
先看下 `RouterGroup` 的结构，`RouterGroup` 提供了所有操作 bm 路由的 [方法集合](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/routergroup.go)：
```golang
type Engine struct {
	RouterGroup             // 保存路由的中间件，提供对外操作接口
    ......
    trees     methodTrees   // 存储路由 Tree
}
```

####    路由结构 RouterGroup
`RouterGroup` 是对路由树的包装，所有的路由规则最终都通过 `RouterGroup` 提供的方法来进行管理。Engine 结构体继承了 RouterGroup ，所以 Engine 直接具备了 RouterGroup 所有的路由管理功能。
```golang
// RouterGroup is used internally to configure router, a RouterGroup is associated with a prefix
// and an array of handlers (middleware).
type RouterGroup struct {
	Handlers   []HandlerFunc
	basePath   string
	engine     *Engine
	root       bool
	baseConfig *MethodConfig
}
```
`RouterGroup` 对外提供了如下的路由控制方法：
-   `GET`:


####    路由树
bm 的路由树的结构定义如下, `methodTrees` 和 `Engine` 一一对应，`methodTree` 代表了一棵树的入口，`node` 表示 `methodTree` 树中的节点。<br>
对 `node` 结构而言，其成员 `children []*node` 表示所有的孩子都放在这个数组里面, 利用 `indices`, `priority` 变相实现了一棵树。<br>
```golang
type methodTrees []methodTree

type methodTree struct {
	method string
	root   *node
}

type node struct {
	path      string
	indices   string
	children  []*node
	handlers  []HandlerFunc     // 该成员的中间件数组
	priority  uint32
	nType     nodeType
	maxParams uint8
	wildChild bool
}
```


##	0x02	bm 框架的路由原理

####    注册路由预处理
如前一节所示，通过下面的代码实现路由注册或者影响到路由的调用，换句话说，下面这些针对路由操作, 最终都会在反应到 bm 的路由树。

1、一般注册方式
```golang
router.POST("/somePost", func(context *bm.Context) {
    ......
})
```

2、使用中间件方式
```golang
router.Use(Logger())
```

3、使用 `bm.Group` 方式构建组
```golang
v1 := router.Group("v1")
{
    v1.POST("login", func(context *gin.Context) {
        ......
    })
}
```

####    具体实现
就上面的三种方式来说明下如何操作路由 `RouterGroup`：
1、一般注册方式，我们以 [`RouterGroup.POST()` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/routergroup.go#L108) 来分析，此方法会调用 `group.handle()` 方法：
```golang
// POST is a shortcut for router.Handle("POST", path, handle).
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle("POST", relativePath, handlers...)
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    absolutePath := group.calculateAbsolutePath(relativePath)
    // 中间件合并
    handlers = group.combineHandlers(handlers)
    // 调用 addRoute 方法增加路由 httpMethod+absolutePath
    group.engine.addRoute(httpMethod, absolutePath, handlers)
    return group.returnObj()
}

//combineHandlers
func (group *RouterGroup) combineHandlers(handlerGroups ...[]HandlerFunc) []HandlerFunc {
	finalSize := len(group.Handlers)
	for _, handlers := range handlerGroups {
		finalSize += len(handlers)
	}
	if finalSize >= int(_abortIndex) {
		panic("too many handlers")
	}
	mergedHandlers := make([]HandlerFunc, finalSize)
	copy(mergedHandlers, group.Handlers)
	position := len(group.Handlers)
	for _, handlers := range handlerGroups {
		copy(mergedHandlers[position:], handlers)
		position += len(handlers)
	}
	return mergedHandlers
}
```

不仅如此，在调用 `POST`、`GET`、`HEAD` 等路由 HTTP 相关函数时, 会调用 `group.handle()` 方法。上面还列出了 `RouterGroup.combineHandlers()` 方法的实现，该方法将传入的路由处理逻辑 `handlerGroups ...[]HandlerFunc` 与 `group` 中间件的成员 `group.Handlers` 合并并返回，然后通过 `group.engine.addRoute()` 注册到路由树中。

2、以中间件方式调用
如果调用了中间件的话, 会调用 [`Use` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go#L388) 或者 `UseFunc` 方法，注意，每次调用这两个方法时都会 `rebuild404Handlers` 及 `rebuild405Handlers`
```golang
// Use attachs a global middleware to the router. ie. the middleware attached though Use() will be
// included in the handlers chain for every single request. Even 404, 405, static files...
// For example, this is the right place for a logger or error management middleware.
func (engine *Engine) Use(middleware ...Handler) IRoutes {
	engine.RouterGroup.Use(middleware...)
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}

// UseFunc attachs a global middleware to the router. ie. the middleware attached though UseFunc() will be
// included in the handlers chain for every single request. Even 404, 405, static files...
// For example, this is the right place for a logger or error management middleware.
func (engine *Engine) UseFunc(middleware ...HandlerFunc) IRoutes {
	engine.RouterGroup.UseFunc(middleware...)
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}
```

3、构建 Group 方式
如果使用了 `bm.Group` 的话, 会调用 `RouterGroup.Group()` 方法，重点关注 `combineHandlers` 及 `calculateAbsolutePath` 这两个方法（前述）：
```golang
// Group creates a new router group. You should add all the routes that have common middlwares or the same path prefix.
// For example, all the routes that use a common middlware for authorization could be grouped.
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
	return &RouterGroup{
		Handlers: group.combineHandlers(handlers),
		basePath: group.calculateAbsolutePath(relativePath),
		engine:   group.engine,
		root:     false,
	}
}

func (group *RouterGroup) calculateAbsolutePath(relativePath string) string {
    return joinPaths(group.basePath, relativePath)
}

func joinPaths(absolutePath, relativePath string) string {
	if relativePath == "" {
		return absolutePath
	}

	finalPath := path.Join(absolutePath, relativePath)
	appendSlash := lastChar(relativePath) == '/' && lastChar(finalPath) != '/'
	if appendSlash {
		return finalPath + "/"
	}
	return finalPath
}
```

####    egnine.addRoute 方法
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

####    小结
这里小结下，bm 框架在调用中间件的时候, 是将某个路由的 `handler` 处理函数和中间件的处理函数合并后，放在了 `Handlers` 的数组中（路由的 `handler` 处理函数在最后位置）；在调用 `Group` 的时候, 是将路由的 `path` 上面拼上 `Group` 的值。举例来说：`GroupPath=/kratos-demo`，`path=/start`, 组合后注册的路由会变成 `/kratos-demo/start`<br>
真正的注册方法是 [`RouterGroup.handle()` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/routergroup.go#L74)，它完成了三件小事：
1.  获取完成的路由信息 `absolutePath`
2.  合并自身与系统的拦截器数组
3.  调用 `group.engine.addRoute(httpMethod, absolutePath, handlers)` 将预处理阶段的结果注册到 `bm.Engine` 的 路由树 `trees` 上。树节点的关键信息为 `httpMethod` 及 `absolutePath`

##  0x05    参考
-   [blademaster 官方文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster.md)
-   [blademaster](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster-mid.md)
-   [go 语言框架 gin 的中文文档](https://github.com/skyhee/gin-doc-cn)
-   [blademaster 官方文档：快速入门](https://github.com/go-kratos/kratos/blob/master/docs/blademaster-quickstart.md)
-   [gin 框架之路由前缀树初始化分析](https://www.infoq.cn/article/8Tg1alapeyfcAKF6zwGh)