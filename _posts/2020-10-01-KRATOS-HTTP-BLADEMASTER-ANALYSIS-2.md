---
layout:     post
title:      Kratos 源码分析：CGI 框架 BM （二）
subtitle:   分析 blademaster 中的拦截器实现及设计
date:       2020-10-01
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
熟悉 gRPC 的开发者对拦截器 interceptor（中间件）不会陌生，`gin` 框架也提供了这种能力。看下面的 helloworld 例子：
```golang
func main() {
    r := gin.Default()

    r.GET("/ping", func(c *gin.Context) {
        c.String(http.StatusOK, "%s", "hello world")
    })

    if err := r.Run("0.0.0.0:8080"); err != nil {
        log.Fatalln(err)
    }
}
```
此程序默认启用了 `2` 个中间件, 分别是 `Logger()` 和 `Recovery()`

##  0x01    拦截器的执行顺序


##  参考
-   [明白了，原来 Go web 框架中的中间件都是这样实现的](https://colobu.com/2019/08/21/decorator-pattern-pipeline-pattern-and-go-web-middlewares/)