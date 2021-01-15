---
layout:     post
title:      golang-LRU 缓存设计与实现
subtitle:   分析 一款高性能的本地缓存开源组件 CCache
date:       2020-07-02
author:     pandaychen
catalog:    true
tags:
    - 缓存
---


##  0x00    开篇
&emsp;&emsp; [ccache](https://github.com/karlseguin/ccache) 是笔者在项目中使用过的一款 Local-Cache 的高性能组件，作为常用的性能优化手段，选择此库是因为它有如下的特点：
1.  Value 是 `interface{}` 类型，支持结构化存储
2.  支持 LRU 算法的 Key 淘汰机制
3.  支持多级 Cache，如 LayeredCache 和 SecondaryCache
4.  采用分段锁及其他一些优化策略来降低锁冲突（Competition）

在项目，用此库来缓存那些不经常改动且有大量读请求的数据场景。

##  0x01  CCache 的使用
使用方法见 [文档](https://github.com/karlseguin/ccache/blob/master/readme.md)，需要注意的一点是，当使用 `Get()` 方法获取值时，需要使用 `Expired()` 方法来判断设置的 key 是否到期，到期后需要调用 `cache.Delete()` 方法进行删除。

##  0x02  CCache 的优化

####  分段锁
第一次接触 Golang 的这个概念，是在 [goim](https://github.com/Terry-Mao/goim/blob/master/internal/comet/bucket.go) 的代码中：

注意看 `Server` 的 `buckets` 成员，就是一个典型的分段锁的实现（只在每个 `Bucket` 中绑定一个读写锁 `sync.RWMutex`），用来保护并发读写的安全性。在 CCache 中也是 [类似做法](https://github.com/karlseguin/ccache/blob/master/cache.go)，将一个 HashTable 根据 key 拆分成多个 HashTable，每个 HashTable 对应一个锁，锁粒度更细，冲突的概率也就相应的降低了。

goim 中的分段锁定义如下：
```golang
// Server is comet server.
type Server struct {
	c         *conf.Config
	round     *Round    // accept round store
	buckets   []*Bucket // subkey bucket
	bucketIdx uint32

	serverID  string
	rpcClient logic.LogicClient
}

// Bucket is a channel holder.
type Bucket struct {
	c     *conf.Bucket
	cLock sync.RWMutex        // protect the channels for chs
	chs   map[string]*Channel // map sub key to a channel
	// room
	rooms       map[string]*Room // bucket room channels
	routines    []chan *grpc.BroadcastRoomReq
	routinesNum uint64

	ipCnts map[string]int32
}
```

在 CCache 中，通过 FNV 算法，根据 Key 来获取到具体存储到哪个 Bucket 中：
```golang
func (c *Cache) bucket(key string) *bucket {
	h := fnv.New32a()
	h.Write([]byte(key))
	return c.buckets[h.Sum32()&c.bucketMask]
}
```

##	0x03	LRU 的实现

####	实现原理


####	代码
[gc 方法](https://github.com/karlseguin/ccache/blob/master/cache.go#L293) 实现了 LRU 删除的逻辑，就是自双链表 `c.list` 的尾部遍历，按照配置 `itemsToPrune` 的个数为上限，进行删除（为空时退出）：
```golang
func (c *Cache) gc() int {
	dropped := 0
	element := c.list.Back()
	for i := 0; i < c.itemsToPrune; i++ {
		if element == nil {
			return dropped
		}
		// 取 prev
		prev := element.Prev()
		item := element.Value.(*Item)
		if c.tracking == false || atomic.LoadInt32(&item.refCount) == 0 {
			// 删除 bucket 中的 key
			c.bucket(item.key).delete(item.key)
			c.size -= item.size
			// 从 list 中 remove
			c.list.Remove(element)
			if c.onDelete != nil {
				c.onDelete(item)
			}
			dropped += 1
			item.promotions = -2
		}
		// 继续遍历
		element = prev
	}
	return dropped
}
```

[worker 方法](https://github.com/karlseguin/ccache/blob/master/cache.go#L216)，
```golang
func (c *Cache) worker() {
	defer close(c.control)
	dropped := 0
	for {
		select {
		case item, ok := <-c.promotables:
			if ok == false {
				goto drain
			}
			if c.doPromote(item) && c.size > c.maxSize {
				dropped += c.gc()
			}
		case item := <-c.deletables:
			c.doDelete(item)
		case control := <-c.control:
			switch msg := control.(type) {
			case getDropped:
				msg.res <- dropped
				dropped = 0
			case setMaxSize:
				c.maxSize = msg.size
				if c.size > c.maxSize {
					dropped += c.gc()
				}
			case clear:
				for _, bucket := range c.buckets {
					bucket.clear()
				}
				c.size = 0
				c.list = list.New()
				msg.done <- struct{}{}
			}
		}
	}

drain:
	for {
		select {
		case item := <-c.deletables:
			c.doDelete(item)
		default:
			close(c.deletables)
			return
		}
	}
}
```

##  参考
-   [ccache Doc](https://github.com/karlseguin/ccache/blob/master/readme.md)
-   [go 语言高性能缓存组件 ccache 分析](https://juejin.im/post/5cb3e226e51d456e853f8119)