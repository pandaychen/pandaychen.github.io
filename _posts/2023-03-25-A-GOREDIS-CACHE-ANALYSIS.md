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


####	Item结构
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


##	基础方法

####	set方法
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

####	getBytes方法
```golang
func (cd *Cache) getBytes(ctx context.Context, key string, skipLocalCache bool) ([]byte, error) {
	if !skipLocalCache && cd.opt.LocalCache != nil {
		b, ok := cd.opt.LocalCache.Get(key)
		if ok {
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

####	Cache.Set方法



## 0x02 参考
[go-redis/cache](https://github.com/go-redis/cache)