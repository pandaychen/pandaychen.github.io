---
layout:     post
title:      Kratos 源码分析（一）：Lazy Load Container
subtitle:   分析 Kratos 的懒对象容器及应用场景
date:       2020-06-06
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00 前言
&emsp;&emsp; 所谓懒加载（Lazy）就是延时加载，延迟加载。至于为什么要用懒加载呢，就是当我们要访问的数据量过大时，明显用缓存不太合适，因为内存容量有限，为了减少并发量及系统资源的消耗，我们让数据在需要的时候才进行加载。这就是懒加载。

在数据、策略等需要 "分组（组中以 Name 区分）" 的情况下，非常适合使用该方式实现存储（取），这里存储的值以 interface{} 表示，可以是一个方法，或是一个结构体等；

##  0x01    核心代码
`Group` 定义如下，注意 `Group` 的成员里 `new` 是一个方法成员，该方法的返回值为 `interface{}`：
```golang
// Package group provides a sample lazy load container.
// The group only creating a new object not until the object is needed by user.
// And it will cache all the objects to reduce the creation of object.

// Group is a lazy load container.
type Group struct {
	new  func() interface{} // 存储的对象
	objs sync.Map           //syncMAP
	sync.RWMutex
}
```

Group 只提供了如下对外的方法：
1.  `NewGroup` 方法：初始化一个 `Group`，传入方法参数 `new func() interface{}`
2.  `Get` 方法：以 `key` 在 `sync.Map` 中搜索，如果不存在则创建并存储，创建调用的方法就是 `NewGroup` 中传入的参数 `new`
3.  `Reset` 方法：传入方法参数 `new func() interface{}`，更新 `new` 成员
4.  `Clear` 方法，遍历 `objs sync.Map`，删除所有的 `key`

```golang
// NewGroup news a group container.
func NewGroup(new func() interface{}) *Group {
	if new == nil {
		panic("container.group: can't assign a nil to the new function")
	}
	return &Group{
		new: new, //
	}
}

// Get gets the object by the given key.
// 以 key 在 sync.Map 中搜索，如果不存在则创建
func (g *Group) Get(key string) interface{} {
	g.RLock()
	new := g.new
	g.RUnlock()
	obj, ok := g.objs.Load(key) // 查询 key
	if !ok {
		obj = new()
		g.objs.Store(key, obj)
	}
	return obj
}

// Reset resets the new function and deletes all existing objects.
func (g *Group) Reset(new func() interface{}) {
	if new == nil {
		panic("container.group: can't assign a nil to the new function")
	}
	g.Lock()
	g.new = new
	g.Unlock()
	g.Clear()
}

// Clear deletes all objects.
func (g *Group) Clear() {
	g.objs.Range(func(key, value interface{}) bool {
		g.objs.Delete(key)
		return true
	})
}
```

##  0x02    Group 应用
Kraots 框架在限流器和熔断器中，都采用了 `Group` 方法进行了配置。这里以限流器的实现，观察下 `Group` 的应用模式。

####	Group 初始化及二次封装
在 [BBR 限流算法的实现](https://github.com/go-kratos/kratos/blob/master/pkg/ratelimit/bbr/bbr.go) 中，使用了 `Group` 做限流策略封装：

```golang
// RateLimiter bbr middleware.
type RateLimiter struct {
	group   *bbr.Group
	logTime int64
}

// New return a ratelimit middleware.
func New(conf *bbr.Config) (s *RateLimiter) {
	return &RateLimiter{
		group:   bbr.NewGroup(conf),
		logTime: time.Now().UnixNano(),
	}
}
```

其中 `bbr` 包的 `Group` 类型，封装了最底层的 `Group`：
```golang
// Group represents a class of BBRLimiter and forms a namespace in which units of BBRLimiter.
type Group struct {
	group *group.Group		// 封装了原始的 Group
}
```

`bbr` 包的 `NewGroup` 方法，封装了原始的 `Group` 的 `NewGroup` 方法，传入的 `new` 方法为 `bbr.newLimiter`

```golang
// NewGroup new a limiter group container, if conf nil use default conf.
func NewGroup(conf *Config) *Group {
	if conf == nil {
		conf = defaultConf
	}
	group := group.NewGroup(func() interface{} {
		return newLimiter(conf)		//Group 中传入 new 为  newLimiter(conf)
	})

	// 返回 group 对象
	return &Group{
		group: group,
	}
}

// Get get a limiter by a specified key, if limiter not exists then make a new one.
func (g *Group) Get(key string) limit.Limiter {
	limiter := g.group.Get(key)
	return limiter.(limit.Limiter)
}
```

这样，在 `bbr` 包的 `Get` 方法中，就通过 Lazy Load 方式拿到了一份限流器的配置。


这里，贴一下 `bbr.newLimiter` 方法的代码，注意它返回的类型是 `limit.Limiter`，这里也体现了 golang 的特色：`interface{}` 可以包容万物。
```golang
func newLimiter(conf *Config) limit.Limiter {
	if conf == nil {
		conf = defaultConf
	}
	size := conf.WinBucket
	bucketDuration := conf.Window / time.Duration(conf.WinBucket)
	passStat := metric.NewRollingCounter(metric.RollingCounterOpts{Size: size, BucketDuration: bucketDuration})
	rtStat := metric.NewRollingCounter(metric.RollingCounterOpts{Size: size, BucketDuration: bucketDuration})
	cpu := func() int64 {
		return atomic.LoadInt64(&cpu)
	}
	limiter := &BBR{
		cpu:             cpu,
		conf:            conf,
		passStat:        passStat,
		rtStat:          rtStat,
		winBucketPerSec: int64(time.Second) / (int64(conf.Window) / int64(conf.WinBucket)),
	}
	return limiter
}
```

####	使用 Group

如下 warden-Server 端的 [限流器 Interceptor 代码](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/ratelimiter/ratelimiter.go) 中，通过 `limiter := b.group.Get(uri)`，这里以 `uri` 为 `key`，从 `Group` 中获取限流器的配置，然后执行限流判定：

```golang
// Limit is a server interceptor that detects and rejects overloaded traffic.
func (b *RateLimiter) Limit() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		uri := args.FullMethod			// 取 RPC 的方法
		limiter := b.group.Get(uri)
		// 执行限流判定
		done, err := limiter.Allow(ctx)
		if err != nil {
			// 需要限流
			_metricServerBBR.Inc(uri)
			return
		}
		// 不限流
		defer func() {
			done(limit.DoneInfo{Op: limit.Success})
			b.printStats(uri, limiter)
		}()
		// 调用 rpc 方法
		resp, err = handler(ctx, req)
		return
	}
}
```

##  0x03    参考
-   [Group 的实现代码](https://github.com/go-kratos/kratos/blob/master/pkg/container/group/group.go)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权