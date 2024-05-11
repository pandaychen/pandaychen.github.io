---
layout:     post
title:      ego/gocache：一个缓存管理通用工具库的实现
subtitle:
date:       2024-04-01
author:     pandaychen
catalog:    true
tags:
    - Cache
---


##  0x00    前言
[gocache](https://github.com/eko/gocache) 是一个可扩展的缓存库（非缓存实现）

-	作为上层封装了众多开源 cache 库 / redis 的实现
-	缓存语义的包装，比如 chain cache，loadable cache 等
-	支持 cache 类通用指标的采集

版本分析基于 [lib/v4.1.6](https://github.com/eko/gocache/releases)

##	0x01	公共接口

####	Store adapters：存储适配器
该 `interface` 表示可以在存储上执行的所有操作，并且每个操作都调用客户端库中的必要方法（屏蔽了各类库的底层接口）：

```GO
type StoreInterface interface {
	Get(key interface{}) (interface{}, error)
	Set(key interface{}, value interface{}, options *Options) error
	Delete(key interface{}) error
	Invalidate(options InvalidateOptions) error
	Clear() error
	GetType() string
}
```

支持的 store adapters 如下：

* [Memory (bigcache)](https://github.com/allegro/bigcache) (allegro/bigcache)
* [Memory (ristretto)](https://github.com/dgraph-io/ristretto) (dgraph-io/ristretto)
* [Memory (go-cache)](https://github.com/patrickmn/go-cache) (patrickmn/go-cache)
* [Memcache](https://github.com/bradfitz/gomemcache) (bradfitz/memcache)
* [Redis](https://github.com/go-redis/redis) (go-redis/redis)
* [Redis (rueidis)](https://github.com/redis/rueidis) (redis/rueidis)
* [Freecache](https://github.com/coocood/freecache) (coocood/freecache)
* [Pegasus](https://pegasus.apache.org/) ([apache/incubator-pegasus](https://github.com/apache/incubator-pegasus)) [benchmark](https://pegasus.apache.org/overview/benchmark/)
* [Hazelcast](https://github.com/hazelcast/hazelcast-go-client) (hazelcast-go-client/hazelcast)

####	Cache adapters
`CacheInterface` 定义了对缓存项执行的所有必要操作: 设置、获取、删除、使数据无效、清除所有缓存等方法

```GO
type CacheInterface interface {
	Get(key interface{}) (interface{}, error)
	Set(key, object interface{}, options *store.Options) error
	Delete(key interface{}) error
	Invalidate(options store.InvalidateOptions) error
	Clear() error

	// 获取当前缓存类型是什么
	GetType() string
}
```

####	`CacheInterface` 的实例化

-	`Cache`：允许操作来自给定存储的数据的基本缓存
-	`Chain`：允许链接多个缓存的特殊缓存适配器（比如利用内存缓存 + redis 构造一个分层缓存）
-	`Loadable`： 特殊的缓存适配器，可以设置回调函数，以便在数据过期 OR 无效时自动将其重新加载到缓存中
-	`Metric`：特殊的缓存适配器，允许存储有关缓存数据的指标: 设置、获取、无效、成功或失败的项目数


####	基础使用

1、以初始化 Memcache 存储为例：

```GO
memcacheStore := store.NewMemcache(
	memcache.New("10.0.0.1:11211", "10.0.0.2:11211", "10.0.0.3:11212"),
	&store.Options{
		Expiration: 10*time.Second,
	},
)

// 然后将初始化的存储传递给缓存对象构造函数
cacheManager := cache.New(memcacheStore)
err := cacheManager.Set("my-key", []byte("my-value"), &cache.Options{
	Expiration: 15*time.Second, // Override default value of 10 seconds defined in the store
})
if err != nil {
    panic(err)
}

value := cacheManager.Get("my-key")

cacheManager.Delete("my-key")

cacheManager.Clear() // Clears the entire cache, in case you want to flush all cache
```

2、Chain：链式缓存，比如下例中的内存缓存加 redis 的组合，在该模式下，如果 ristretto 的缓存中没有数据，而 redis 有数据，那么数据也会被设置回 ristretto 内存缓存中

```GO
func main() {
        // Initialize Ristretto cache and Redis client
        ristrettoCache, err := ristretto.NewCache(&ristretto.Config{NumCounters: 1000, MaxCost: 100, BufferItems: 64})
        if err != nil {
                panic(err)
        }

        redisClient := redis.NewClient(&redis.Options{Addr: "x.x.x.x:6379", Password: "xxx"})

        // Initialize stores
        ristrettoStore := ristretto_store.NewRistretto(ristrettoCache)
        redisStore := redis_store.NewRedis(redisClient, store.WithExpiration(500*time.Second))

        // Initialize chained cache
        cacheManager := cache.NewChain[any](
                cache.New[any](ristrettoStore),
                cache.New[any](redisStore),
        )

        fmt.Println(cacheManager.Get(context.TODO(), "xxx"))
        // ... Then, do what you want with your cache

        time.Sleep(10*time.Second)

        fmt.Println(ristrettoCache.Get("xxx"))
}
```

3、Loadable 模式

TODO

##	0x01	实现分析

####	Chain 模式
结构定义如下，`setChannel` 用于异步处理要 set 的 kv 数据
```GO
type chainKeyValue[T any] struct {
	key       any
	value     T
	ttl       time.Duration
	storeType *string
}

// ChainCache represents the configuration needed by a cache aggregator
type ChainCache[T any] struct {
	caches     []SetterCacheInterface[T]
	setChannel chan *chainKeyValue[T]
}
```

1、查询及异步设置

```GO
// Get returns the object stored in cache if it exists
func (c *ChainCache[T]) Get(ctx context.Context, key any) (T, error) {
	var object T
	var err error
	var ttl time.Duration

	for _, cache := range c.caches {
		storeType := cache.GetCodec().GetStore().GetType()
		object, ttl, err = cache.GetWithTTL(ctx, key)
		if err == nil {
			// Set the value back until this cache layer
			// 如果查询到了，那么会异步设置
			c.setChannel <- &chainKeyValue[T]{key, object, ttl, &storeType}
			return object, nil
		}
	}

	return object, err
}

// setter sets a value in available caches, until a given cache layer
func (c *ChainCache[T]) setter() {
	for item := range c.setChannel {
		for _, cache := range c.caches {
			//不设置获取到的这个层次，只设置其他层次
			if item.storeType != nil && *item.storeType == cache.GetCodec().GetStore().GetType() {
				break
			}

			cache.Set(context.Background(), item.key, item.value, store.WithExpiration(item.ttl))
		}
	}
}
```


##  0x02    参考
-   [I wrote Gocache: a complete and extensible Go cache library](https://vincent.composieux.fr/article/i-wrote-gocache-a-complete-and-extensible-go-cache-library)