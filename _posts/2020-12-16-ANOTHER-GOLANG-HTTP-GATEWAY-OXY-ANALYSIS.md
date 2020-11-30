---
layout:     post
title:      HTTP(s) 代理中间件库：Oxy（未完成）
subtitle:   分析基于 Golang 的 HTTP(s) 代理中间件库 Oxy：Part One
date:       2020-12-16
author:     pandaychen
header-img: img/super-mario.jpg
catalog: true
category:   false
tags:
	- 反向代理
  - 网关
  - Oxy
---

##  0x00    前言
[Oxy：Go middlewares for HTTP servers & proxies](https://github.com/vulcand/oxy) 是一个 HTTP(s) 代理开发（中间件）库，另一个负载均衡网关项目 [vulcand](https://github.com/vulcand/vulcand) 也基于此库来实现反向代理（Reverse Proxy Engine）功能。它的 [主要组件如下](https://github.com/vulcand/oxy/blob/master/README.md)，框架还是围绕着原生 `net/http` 库实现的：
* [Buffer](https://pkg.go.dev/github.com/vulcand/oxy/buffer) retries and buffers requests and responses
* [Stream](https://pkg.go.dev/github.com/vulcand/oxy/stream) passes-through requests, supports chunked encoding with configurable flush interval
* [Forward](https://pkg.go.dev/github.com/vulcand/oxy/forward) forwards requests to remote location and rewrites headers
* [Roundrobin](https://pkg.go.dev/github.com/vulcand/oxy/roundrobin) is a round-robin load balancer
* [Circuit Breaker](https://pkg.go.dev/github.com/vulcand/oxy/cbreaker) Hystrix-style circuit breaker
* [Connlimit](https://pkg.go.dev/github.com/vulcand/oxy/connlimit) Simultaneous connections limiter
* [Ratelimit](https://pkg.go.dev/github.com/vulcand/oxy/ratelimit) Rate limiter (based on tokenbucket algo)
* [Trace](https://pkg.go.dev/github.com/vulcand/oxy/trace) Structured request and response logger

同类型的其他 package 还有：
-	[elazarl/goproxy](https://github.com/elazarl/goproxy)
-	[net/http/httputil]()

无论是 `gin` Web 框架，还是 HTTP(s) 代理，都离不开对 `http.Handler` 的封装、对 HTTP 路由、Header、参数及其他 HTTP 关键参数的重写及改造，而标准库 `net/http` 只是提供了最基础的 CS 架构下的 Request-Response 模型：
> Every handler is http.Handler, so writing and plugging in a middleware is easy

####	使用介绍
从官方提供的使用例子，先了解下 Oxy 的基础功能：
1、Simple [reverse proxy](https://github.com/vulcand/oxy/blob/master/forward/fwd.go)，简单的反向代理，将 `8080` 的 HTTP 请求发送给 `http://localhost:63450` <br>
```golang
import (
  "net/http"
  "github.com/vulcand/oxy/forward"
  "github.com/vulcand/oxy/testutils"
)

// Forwards incoming requests to whatever location URL points to, adds proper forwarding headers
fwd, _ := forward.New()

redirect := http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
    // let us forward this request to another server
    req.URL = testutils.ParseURI("http://localhost:63450")
    // 将上层的 Request 和 ResponseWriter 传递给 fwd
		fwd.ServeHTTP(w, req)
})

// that's it! our reverse proxy is ready!
s := &http.Server{
	Addr:           ":8080",
	Handler:        redirect,
}
s.ListenAndServe()
```

2、给反向代理的多后端增加负载均衡策略 <br>
```golang
import (
        "github.com/vulcand/oxy/forward"
        "github.com/vulcand/oxy/roundrobin"
        "github.com/vulcand/oxy/testutils"
        "net/http"
)

func main() {
        // Forwards incoming requests to whatever location URL points to, adds proper forwarding headers
        fwd, _ := forward.New()
        lb, _ := roundrobin.New(fwd)

        url1 := testutils.ParseURI("http://localhost:63450")
        url2 := testutils.ParseURI("http://localhost:63451")

        lb.UpsertServer(url1)
        lb.UpsertServer(url2)

        s := &http.Server{
                Addr:    ":8080",
                Handler: lb,
        }
        s.ListenAndServe()
}
```

3、额外处理重试及错误 <br>
```golang
import (
  "net/http"
  "github.com/vulcand/oxy/forward"
  "github.com/vulcand/oxy/buffer"
  "github.com/vulcand/oxy/roundrobin"
)

// Forwards incoming requests to whatever location URL points to, adds proper forwarding headers

fwd, _ := forward.New()
lb, _ := roundrobin.New(fwd)

// buffer will read the request body and will replay the request again in case if forward returned status
// corresponding to nework error (e.g. Gateway Timeout)
buffer, _ := buffer.New(lb, buffer.Retry(`IsNetworkError() && Attempts() < 2`))

lb.UpsertServer(url1)
lb.UpsertServer(url2)

// that's it! our reverse proxy is ready!
s := &http.Server{
	Addr:           ":8080",
	Handler:        buffer,
}
s.ListenAndServe()
```


##   参考
-   [Vulcand Gateway Doc](https://vulcand.github.io/quickstart.html)
-   [Vulcand - Programmatic load balancer backed by Etcd](https://github.com/vulcand/vulcand)
-	  [goproxy：An HTTP proxy library for Go](https://github.com/elazarl/goproxy)