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


##  0x05    参考
-   [blademaster 官方文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster.md)
-   [blademaster](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster-mid.md)
-   [go 语言框架 gin 的中文文档](https://github.com/skyhee/gin-doc-cn)
-   [blademaster 官方文档：快速入门](https://github.com/go-kratos/kratos/blob/master/docs/blademaster-quickstart.md)
