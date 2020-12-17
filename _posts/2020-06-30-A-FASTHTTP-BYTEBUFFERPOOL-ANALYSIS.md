---
layout:     post
title:      Fasthttp 高性能 HttpServer 最佳实践之二：bytebufferpool
subtitle:   分析 Fasthttp 的 byte 对象池实现
date:       2020-06-30
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - Fasthttp
    - Pool
---


##  0x00    前言
`sync.Pool` 是 golang 提供的对象池实现，主要用于提高对象的复用效率，减少 GC。而 fasthttp 也提供了类似的实现 [bytebufferpool](https://github.com/valyala/bytebufferpool)，bytebufferpool 维护了一个数据对象池，比如当接收到 `http.Request` 的时候，就会从该对象池中获取一个数据对象填充，用完再归还。


##  0x01    数据结构
核心结构 [Pool](https://github.com/valyala/bytebufferpool/blob/master/pool.go#L25) 定义如下：
```golang
// Pool represents byte buffer pool.
//
// Distinct pools may be used for distinct types of byte buffers.
// Properly determined byte buffer types with their own pools may help reducing
// memory waste.
type Pool struct {
	calls       [steps]uint64
	calibrating uint64

	defaultSize uint64
	maxSize     uint64

	pool sync.Pool      //pool sync.Pool 的封装
}
```

##  参考
-   [Anti-memory-waste byte buffer pool：bytebufferpool](https://github.com/valyala/bytebufferpool)
