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
[go-redis/cache](https://github.com/go-redis/cache) 是一个小而精悍的项目，实现了本地缓存配合 redis（远端缓存）的高性能 cache，可借鉴的地方两点（以 `V8` 版本分析）：

1.  缓存CRUD的操作语义（本地 / 远程）
2.  singleflight 机制的应用
3.	如果value比较大, 可以考虑启用[压缩](https://github.com/go-redis/cache/blob/v8/cache.go#L350)
4.	如果缓存数据量比较大, 并且对性能有要求的场景, 可以尝试[使用](https://github.com/go-redis/cache/blob/v8/cache.go#L332)msgpack代替json


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

可以传入本地缓存实现 `LocalCache` 以及远程缓存实现 `rediser`，项目提供了默认的 `LocalCache`[实现](https://github.com/go-redis/cache/blob/v8/local.go)

```golang
type rediser interface {
	Set(ctx context.Context, key string, value interface{}, ttl time.Duration) *redis.StatusCmd
	SetXX(ctx context.Context, key string, value interface{}, ttl time.Duration) *redis.BoolCmd
	SetNX(ctx context.Context, key string, value interface{}, ttl time.Duration) *redis.BoolCmd

	Get(ctx context.Context, key string) *redis.StringCmd
	Del(ctx context.Context, keys ...string) *redis.IntCmd
}


type LocalCache interface {
	Set(key string, data []byte)
	Get(key string) ([]byte, bool)
	Del(key string)
}
```

注意，如果[初始化](https://github.com/go-redis/cache/blob/v8/cache.go#L125)的时候没有传入`LocalCache`相关配置的话，那么这里就退化为Redis的KV操作了，参考[此项目](https://github.com/TencentBlueKing/bk-iam/blob/master/pkg/cache/redis/redis.go#L81)


####	Item 结构
```golang
type Item struct {
	Ctx context.Context

	Key   string
	Value interface{}

	// TTL is the cache expiration time.
	// Default TTL is 1 hour.
	TTL time.Duration

	// Do returns value to be cached.
	Do func(*Item) (interface{}, error)

	// SetXX only sets the key if it already exists.
	SetXX bool

	// SetNX only sets the key if it does not already exist.
	SetNX bool

	// SkipLocalCache skips local cache as if it is not set.
	SkipLocalCache bool
}
```

`Item` 结构中，`Do` 方法的作用是对 `Item` 执行用户传入的方法，并以此为 `Value`
```GOLANG
func (item *Item) value() (interface{}, error) {
	if item.Do != nil {
		return item.Do(item)
	}
	if item.Value != nil {
		return item.Value, nil
	}
	return nil, nil
}
```


##	0x02	基础方法

####	set 方法
`set` 方法描述了缓存的 Set 方法的语义，即：
1.	先获取缓存值
2.	设置本地缓存（不 care 是否成功），项目中本地缓存的 TTL 是个固定值
3.	设置 Redis 缓存及 TTL


```golang
func (cd *Cache) set(item *Item) ([]byte, bool, error) {
	value, err := item.value()
	if err != nil {
		return nil, false, err
	}

	b, err := cd.Marshal(value)
	if err != nil {
		return nil, false, err
	}

	if cd.opt.LocalCache != nil && !item.SkipLocalCache {
		cd.opt.LocalCache.Set(item.Key, b)
	}

	if cd.opt.Redis == nil {
		if cd.opt.LocalCache == nil {
			return b, true, errRedisLocalCacheNil
		}
		return b, true, nil
	}

	ttl := item.ttl()
	if ttl == 0 {
		return b, true, nil
	}

	if item.SetXX {
		return b, true, cd.opt.Redis.SetXX(item.Context(), item.Key, b, ttl).Err()
	}
	if item.SetNX {
		return b, true, cd.opt.Redis.SetNX(item.Context(), item.Key, b, ttl).Err()
	}
	return b, true, cd.opt.Redis.Set(item.Context(), item.Key, b, ttl).Err()
}
```

####	getBytes 方法
`getBytes` 方法描述了缓存的 Get 方法的语义，即：
1.	如果本地缓存有效，那么先查本地缓存，查询命中直接返回
2.	如 `1` 未命中，则查询 Redis；同时标记 cacheMiss
3.	如果 Redis 查询命中，那么返回前优先设置本地缓存

```golang
func (cd *Cache) getBytes(ctx context.Context, key string, skipLocalCache bool) ([]byte, error) {
	if !skipLocalCache && cd.opt.LocalCache != nil {
		b, ok := cd.opt.LocalCache.Get(key)
		if ok {
			// 本地命中
			return b, nil
		}
	}

	if cd.opt.Redis == nil {
		if cd.opt.LocalCache == nil {
			return nil, errRedisLocalCacheNil
		}
		return nil, ErrCacheMiss
	}

	b, err := cd.opt.Redis.Get(ctx, key).Bytes()
	if err != nil {
		if cd.opt.StatsEnabled {
			atomic.AddUint64(&cd.misses, 1)
		}
		if err == redis.Nil {
			return nil, ErrCacheMiss
		}
		return nil, err
	}

	if cd.opt.StatsEnabled {
		atomic.AddUint64(&cd.hits, 1)
	}

	if !skipLocalCache && cd.opt.LocalCache != nil {
		cd.opt.LocalCache.Set(key, b)
	}
	return b, nil
}
```

####	getSetItemBytesOnce 方法
`getSetItemBytesOnce` 方法提供了基于 `singleflight` 查询优化的语义，如下：
1.	优先在本地缓存查询，存在则返回
2.	当本地缓存 miss 时，以 singleflight 方式访问 Redis，有两种情况：
	-	当 Redis 查询**命中时**（调用 `getBytes` 方法），直接返回
	-	当 Redis 查询**未命中时**，调用 `set` 方法设置缓存，设置完成返回（succ/failed）
3.	返回 singleflight 的结果

通过singleflight机制保证，当本地Cache Miss时，并发的请求不会大量透传到Redis，从而保障Redis的可用性

```golang
func (cd *Cache) getSetItemBytesOnce(item *Item) (b []byte, cached bool, err error) {
	if cd.opt.LocalCache != nil {
		b, ok := cd.opt.LocalCache.Get(item.Key)
		if ok {
			return b, true, nil
		}
	}

	v, err, _ := cd.group.Do(item.Key, func() (interface{}, error) {
		b, err := cd.getBytes(item.Context(), item.Key, item.SkipLocalCache)
		if err == nil {
			cached = true
			return b, nil
		}

		b, ok, err := cd.set(item)
		if ok {
			return b, nil
		}
		return nil, err
	})
	if err != nil {
		return nil, false, err
	}
	return v.([]byte), cached, nil
}
```


####	Del方法
`Delete`的逻辑就比较简单了：

-	如果指定了`LocalCache`，就删除本地缓存的key
-	再删除Redis的Key

```GO
func (cd *Cache) Delete(ctx context.Context, key string) error {
	if cd.opt.LocalCache != nil {
		cd.opt.LocalCache.Del(key)
	}

	if cd.opt.Redis == nil {
		if cd.opt.LocalCache == nil {
			return errRedisLocalCacheNil
		}
		return nil
	}

	_, err := cd.opt.Redis.Del(ctx, key).Result()
	return err
}
```


## 0x03 参考
[go-redis/cache](https://github.com/go-redis/cache)