---
layout: post
title: Golang httprouter 库分析
subtitle: 高效的路由框架 httprouter 分析
date: 2021-10-01
author: pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category: false
tags:
  - Golang
---

## 0x00 前言

[httprouter](https://github.com/julienschmidt/httprouter) 非常高效的一个 http 路由框架，gin 框架的路由也基于此库。使用比较简单：

```golang
func main() {
    router := httprouter.New()

    //注册路由
    router.GET("/", Index)
    router.GET("/hello/:name", Hello)

    log.Fatal(http.ListenAndServe(":8080", router))
}
```

## 0x01 基数树 Radix Tree

httprouter 基于 Radix Tree 实现，Radix Tree 的特点是拥有共同前缀的节点也共享同一个父节点。经典例子：

```javascript
Priority   Path             Handle
9          \                *<1>
3          ├s               nil
2          |├earch\         *<2>
1          |└upport\        *<3>
2          ├blog\           *<4>
1          |    └:post      nil
1          |         └\     *<5>
2          ├about-us\       *<6>
1          |        └team\  *<7>
1          └contact\        *<8>
```

对应于下面的 GET 请求，`*<num>` 是方法（handler）对应的指针，从根节点遍历到叶子节点我们就能得到完整的路由表：

```golang
GET("/", func1)
GET("/search/", func2)
GET("/support/", func3)
GET("/blog/", func4)
GET("/blog/:post/", func5)
GET("/about-us/", func6)
GET("/about-us/team/", func7)
GET("/contact/", func8)
```

## 0x02 主要数据结构

#### Path

#### Router

```golang
// Router is a http.Handler which can be used to dispatch requests to different
// handler functions via configurable routes
type Router struct {
    trees map[string]*node  //Map 存储了 HTTP 不同方法的根节点
    paramsPool sync.Pool    //WITH efficiency
    maxParams  uint16
    SaveMatchedRoutePath bool
    RedirectTrailingSlash bool
    RedirectFixedPath bool
    HandleMethodNotAllowed bool
    HandleOPTIONS bool
    GlobalOPTIONS http.Handler  //
    globalAllowed string
    NotFound http.Handler   //
    MethodNotAllowed http.Handler   //
    PanicHandler func(http.ResponseWriter, *http.Request, interface{})
}
```

`Router` 也实现了 `ServeHTTP` 方法，因此符合 `net/http.Handler` 接口，`ServeHTTP` 方法为处理请求时，查找路由树及 `Handler` 的逻辑：

```golang
// ServeHTTP makes the router implement the http.Handler interface.
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    if r.PanicHandler != nil {
        defer r.recv(w, req)
    }

    path := req.URL.Path

    // 获取本次请求路由（path）的 handle 方法
    if root := r.trees[req.Method]; root != nil {
    	if handle, ps, tsr := root.getValue(path, r.getParams); handle != nil {
            //root.getValue 查找路由树的具体方法
    		if ps != nil {
    			handle(w, req, *ps)
    			r.putParams(ps)
    		} else {
    			handle(w, req, nil)
    		}
    		return
    .....
}
```

#### Tree

## 0x03 路由注册

## 0x04 路由查找

## 0x05 查找算法

## 0x06 参考

- [Radix tree](https://en.wikipedia.org/wiki/Radix_tree)
