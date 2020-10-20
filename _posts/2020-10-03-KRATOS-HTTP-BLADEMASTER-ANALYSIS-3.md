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
本篇文章看下 bm 框架的路由注册及搭配拦截器的路由调用的实现细节。先看下路由是如何注册的，先看下 bm 框架的默认生成 [模板](https://github.com/bilibili/kratos-demo/blob/master/internal/server/http/server.go)：
1.  Bm 默认创建的 `engine` 及启动
2.  `initRouter` 初始化路由的方法及路由（`Group`）的创建
3.  Bm 的 handler 方法 `ping` 和 `howToStart` 的定义（ `handler` 方法默认传入 bm 的 `Context` 对象）

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

##	0x01	gin的路由原理
注册路由预处理
我们在使用gin时通过下面的代码注册路由

普通注册
router.POST("/somePost", func(context *gin.Context) {
    context.String(http.StatusOK, "some post")
})
使用中间件
router.Use(Logger())
使用Group
v1 := router.Group("v1")
{
    v1.POST("login", func(context *gin.Context) {
        context.String(http.StatusOK, "v1 login")
    })
}
这些操作, 最终都会在反应到gin的路由树上

具体实现
// routergroup.go:L72-77
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    absolutePath := group.calculateAbsolutePath(relativePath) // <---
    handlers = group.combineHandlers(handlers) // <---
    group.engine.addRoute(httpMethod, absolutePath, handlers)
    return group.returnObj()
}
在调用POST, GET, HEAD等路由HTTP相关函数时, 会调用handle函数

如果调用了中间件的话, 会调用下面函数

func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
    group.Handlers = append(group.Handlers, middleware...)
    return group.returnObj()
}
如果使用了Group的话, 会调用下面函数

func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
    return &RouterGroup{
        Handlers: group.combineHandlers(handlers),
        basePath: group.calculateAbsolutePath(relativePath),
        engine:   group.engine,
    }
}
重点关注下面两个函数:

// routergroup.go:L208-217
func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
    finalSize := len(group.Handlers) + len(handlers)
    if finalSize >= int(abortIndex) {
        panic("too many handlers")
    }
    mergedHandlers := make(HandlersChain, finalSize)
    copy(mergedHandlers, group.Handlers)
    copy(mergedHandlers[len(group.Handlers):], handlers)
    return mergedHandlers
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
在joinPaths函数里面有段代码, 很有意思, 我还以为是写错了. 主要是path.Join的用法.

finalPath := path.Join(absolutePath, relativePath)
appendSlash := lastChar(relativePath) == '/' && lastChar(finalPath) != '/'
在当路由是/user/这种情况就满足了lastChar(relativePath) == '/' && lastChar(finalPath) != '/'. 主要原因是path.Join(absolutePath, relativePath)之后, finalPath是user

综合来看, 在预处理阶段

在调用中间件的时候, 是将某个路由的handler处理函数和中间件的处理函数都放在了Handlers的数组中
在调用Group的时候, 是将路由的path上面拼上Group的值. 也就是/user/:name, 会变成v1/user:name
真正注册
// routergroup.go:L72-77
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    absolutePath := group.calculateAbsolutePath(relativePath) // <---
    handlers = group.combineHandlers(handlers) // <---
    group.engine.addRoute(httpMethod, absolutePath, handlers)
    return group.returnObj()
}
调用group.engine.addRoute(httpMethod, absolutePath, handlers)将预处理阶段的结果注册到gin Engine的trees上

gin路由树简单介绍
gin的路由树算法是一棵前缀树. 不过并不是只有一颗树, 而是每种方法(POST, GET ...)都有自己的一颗树

func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
    assert1(path[0] == '/', "path must begin with '/'")
    assert1(method != "", "HTTP method can not be empty")
    assert1(len(handlers) > 0, "there must be at least one handler")

    debugPrintRoute(method, path, handlers)
    root := engine.trees.get(method) // <-- 看这里
    if root == nil {
        root = new(node)
        engine.trees = append(engine.trees, methodTree{method: method, root: root})
    }
    root.addRoute(path, handlers)
}
gin 路由最终的样子大概是下面的样子



##  0x05    参考
-   [blademaster 官方文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster.md)
-   [blademaster](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster-mid.md)
-   [go 语言框架 gin 的中文文档](https://github.com/skyhee/gin-doc-cn)
-   [blademaster 官方文档：快速入门](https://github.com/go-kratos/kratos/blob/master/docs/blademaster-quickstart.md)
