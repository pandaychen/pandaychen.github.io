---
layout:     post
title:      Cache 设计、使用与优化
subtitle:   如何在设计高效的缓存使用策略
date:       2020-02-22
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Cache
    - 缓存
---

##  0x00    前言
Cache 使用第一法则是：任何 Cache 都需要有自动过期（失效）的机制。

##  0x01    Cache 属性

####    分类
Cache 根据使用可分为远程 Cache （Redis/Memcache）和本地 Cache 两种。

####    过期策略
通常 Cache 的过期淘汰策略有如下几种（以 Redis 和 GCache 为例）：
-	Simple：普通缓存策略，随机淘汰
-	LRU：Least Recently Used，优先淘汰最近最少使用的内容，是一种最近最少使用策略。无论是否过期，根据元素最后一次被使用的时间戳，清除最远使用时间戳的元素释放空间。策略算法主要比较元素最近一次被 get 使用时间。在热点数据场景下较适用，优先保证热点数据的有效性。
-	LFU：Least Frequently Used，优先淘汰访问次数最少的内容，是一种最近最少使用策略。无论是否过期，根据元素的被使用次数判断，清除使用次数较少的元素释放空间。策略算法主要比较元素的 hitCount（命中次数）。在保证高频数据有效性场景下，可选择这类策略。
-	ARC：Adaptive Replacement Cache，ARC 介于 LRU 和 LFU 之间，借助 LRU 和 LFU 基本思想实现，以获得可用缓存的最佳使用。
-	TTL：Time To Live，定时删除

##  0x02    Cache 惊群效应
高并发场景下缓存主要存在以下问题：缓存击穿、缓存穿透及缓存雪崩三种。

####	缓存击穿
在高并发的情况下，大量并发请求同时查询缓存 Cache 中同一个 key 时，此时 key 正好失效，这样会导致同一时间请求都会去查询数据库，会造成数据库 DB 请求量过大（存在 DB 压垮的风险）

惊群效应问题有时被称为缓存击穿，穿透或者雪崩效果。从本质上讲，就像是使系统不堪重负的大量请求。

缓存雪崩：缓存在同一时刻全部失效，造成瞬时 DB 请求量大、压力骤增，引起雪崩。缓存雪崩通常因为缓存服务器宕机、缓存的 key 设置了相同的过期时间等引起。

缓存击穿：一个存在的 key，在缓存过期的一刻，同时有大量的请求，这些请求都会击穿到 DB ，造成瞬时 DB 请求量大、压力骤增。

缓存穿透：查询一个不存在的数据，因为不存在则不会写到缓存中，所以每次都会去请求 DB，如果瞬间流量过大，穿透到 DB，导致宕机。



##  0x03  健壮的 Cache 机制
针对缓存的脆弱性，需要一些针对缓存的保护机制，常用的有 Singleflight，熔断保护，限流等

####    Singleflight 机制
最初看到此方案是在 [groupcache 的实现](https://github.com/golang/groupcache/tree/master/singleflight) 中，它的使用场景是，在多个并发请求触发的回调操作里，只有第一个回调方法被执行，其余请求（当然需要落在第一个回调方法的时间窗口内）阻塞等待第一个被执行的那个回调操作完成后，直接取其结果，以此保证同一时刻只有一个回调方法在执行，以达到防止缓存击穿。Singleflight 常用于下面的场景：
1、缓存失效时的保护性更新 <br>
```golang
if (/* 缓存失效 */) {
    fn = func() (interface{}, error) {
        // 缓存更新逻辑
    }
    data, err = g.Do(cacheKey, fn)
}
```
2、防止突增的接口请求对后端服务造成瞬时高负载<br>
```golang
fn = func() (interface{}, error) {
    // 发送请求到后端服务，并获取结果
}
data, err = g.Do(ApiWithParams, fn)
```

singleflight 的使用方式如下，下面 100 个 goroutine，只有 1 个能进入，其他 goroutine 都只能阻塞等待结果：
```golang
func main() {
        var singleSetCache singleflight.Group
        var wg sync.WaitGroup

        getAndSetCache := func(requestID int, cacheKey string) (string, error) {
                log.Printf("request %v start to get and set cache...", requestID)
                value, _, _ := singleSetCache.Do(cacheKey, func() (ret interface{}, err error) { //do 的入参 key，可以直接使用缓存的 key，这样同一个缓存，只有一个协程会去读 DB
                        log.Printf("request %v is setting cache...", requestID)
                        time.Sleep(3 * time.Second)
                        log.Printf("request %v set cache success!", requestID)
                        return "VALUE", nil
                })
                return value.(string), nil
        }

        cacheKey := "cacheKey"
        for i := 0; i < 100; i++ { // 模拟多个协程同时请求
                wg.Add(1)
                go func(requestID int) {
                        defer wg.Done()
                        value, _ := getAndSetCache(requestID, cacheKey)
                        log.Printf("request %v get value: %v", requestID, value)
                }(i)
        }
        wg.Wait()
}
```

然后简单看下 singleflight 的 [实现原理](https://github.com/golang/sync/blob/master/singleflight/singleflight.go)，其核心就是利用 mutex 和 sync.WaitGroup 机制来实现多 goroutine 的并发控制策略：
`call` 结构用来表示一个正在执行或已完成的函数调用：
```golang
// call is an in-flight or completed singleflight.Do call
type call struct {
	wg sync.WaitGroup

	// These fields are written once before the WaitGroup is done
	// and are only read after the WaitGroup is done.
	val interface{}		// 用来存储最终任务执行的结果
	err error

	// forgotten indicates whether Forget was called with this call's key
	// while the call was still in flight.
	forgotten bool

	// These fields are read and written with the singleflight
	// mutex held before the WaitGroup is done, and are read but
	// not written after the WaitGroup is done.
	dups  int
	chans []chan<- Result
}
```

Group 可以看做是任务的分类，使用 `map[string]*call` 存储任务：
```golang
// Group represents a class of work and forms a namespace in
// which units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}
```

再来看看核心的外部接口 `Do` 方法，再次方法中，`g.m` 的读写被 `g.mu` 互斥锁保护，`fn` 的返回结果存储在 `call.val`、`call.err` 中，并发的 goroutine 通过 `sync.WaitGroup` 实现等待 `fn` 执行结束：
```golang
// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
// The return value shared indicates whether v was given to multiple callers.
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}

	// 所有的重复的任务都会阻塞在此
	if c, ok := g.m[key]; ok {
		c.dups++
		g.mu.Unlock()
		// 其他协程全部都阻塞在 wg.Wait 上，等待唯一的工作协程执行完成后退出后才继续下去
		c.wg.Wait()

		if e, ok := c.err.(*panicError); ok {
			panic(e)
		} else if c.err == errGoexit {
			runtime.Goexit()
		}
		// 从 c 中获取任务执行结果
		return c.val, c.err, true
	}

	// 只有唯一一个任务可以获得执行
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	// 执行用户的方法
	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}

// 真正唯一执行业务逻辑
// doCall handles the single call for a key.
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	normalReturn := false
	recovered := false

	// use double-defer to distinguish panic from runtime.Goexit,
	// more details see https://golang.org/cl/134395
	defer func() {
		// the given function invoked runtime.Goexit
		if !normalReturn && !recovered {
			c.err = errGoexit
		}

		c.wg.Done()
		g.mu.Lock()
		defer g.mu.Unlock()
		if !c.forgotten {
			delete(g.m, key)
		}

		if e, ok := c.err.(*panicError); ok {
			// In order to prevent the waiting channels from being blocked forever,
			// needs to ensure that this panic cannot be recovered.
			if len(c.chans) > 0 {
				go panic(e)
				select {} // Keep this goroutine around so that it will appear in the crash dump.
			} else {
				panic(e)
			}
		} else if c.err == errGoexit {
			// Already in the process of goexit, no need to call again
		} else {
			// Normal return
			for _, ch := range c.chans {
				ch <- Result{c.val, c.err, c.dups> 0}
			}
		}
	}()

	func() {
		defer func() {
			if !normalReturn {
				// Ideally, we would wait to take a stack trace until we've determined
				// whether this is a panic or a runtime.Goexit.
				//
				// Unfortunately, the only way we can distinguish the two is to see
				// whether the recover stopped the goroutine from terminating, and by
				// the time we know that, the part of the stack trace relevant to the
				// panic has been discarded.
				if r := recover(); r != nil {
					c.err = newPanicError(r)
				}
			}
		}()

		c.val, c.err = fn()
		normalReturn = true
	}()

	if !normalReturn {
		recovered = true
	}
}
```

##  0x04    Cache 优秀开源项目
-   [go-cache](https://github.com/patrickmn/go-cache)：An in-memory key:value store/cache (similar to Memcached) library for Go, suitable for single-machine applications. https://patrickmn.com/projects/go-cache/
-   [bigcache](https://github.com/allegro/bigcache)：非常精妙的 cache 实现，参见 https://github.com/allegro/bigcache
-   [ccache](https://github.com/karlseguin/ccache)：A golang LRU Cache for high concurrency

##  0x05  参考
-   [设计实现高性能本地内存缓存](https://blog.joway.io/posts/modern-memory-cache/)
-   [如何设计并实现一个线程安全的 Map ？](https://halfrost.com/go_map_chapter_two/)
-   [Writing a very fast cache service with millions of entries in Go](https://allegro.tech/2016/03/writing-fast-cache-service-in-go.html)
-	[动手写分布式缓存 - GeeCache 第六天 防止缓存击穿](https://geektutu.com/post/geecache-day6.html)