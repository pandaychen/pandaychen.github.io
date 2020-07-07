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

在项目，用此库来缓存那些不经常改动且有大量读请求的场景。

##  0x01  CCache 的使用


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

##  参考
-   [ccache Doc](https://github.com/karlseguin/ccache/blob/master/readme.md)
-   [go 语言高性能缓存组件 ccache 分析](https://juejin.im/post/5cb3e226e51d456e853f8119)