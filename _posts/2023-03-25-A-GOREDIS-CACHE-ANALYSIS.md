---
layout: post
title:  go-redis/cache 库分析与使用
subtitle: 
date: 2023-03-25
author: pandaychen
catalog: true
tags:
  - Golang
  - Redis
---

## 0x00 前言
[go-redis/cache](https://github.com/go-redis/cache)是一个小而精悍的项目，实现了本地缓存配合redis（远端缓存）的高性能cache，可借鉴的地方两点（以`V8`版本分析）：

1.  缓存的操作语义（本地/远程）
2.  singleflight机制的应用


##  0x01  结构

####  Options
```go
type Options struct {
	Redis        rediser
	LocalCache   LocalCache
	StatsEnabled bool
	Marshal      MarshalFunc    //（反）序列化方法
	Unmarshal    UnmarshalFunc
}
```

可以传入本地缓存实现`LocalCache`以及远程缓存实现`rediser`，项目提供了默认的`LocalCache`[实现](https://github.com/go-redis/cache/blob/v8/local.go)



## 0x02 参考
[go-redis/cache](https://github.com/go-redis/cache)