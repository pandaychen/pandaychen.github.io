---
layout:     post
title:      微服务中的缓存（四）：Cache 与 Redis
subtitle:   Cache 选型与使用策略梳理
date:       2021-12-22
author:     pandaychen
header-img:	img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - Cache
    - 缓存
---

##  0x00    前言


##  0x01    本地缓存
设计思想#
在项目中，我们经常会用到 Go 缓存库比如说 patrickmn/go-cache库。但很多缓存库其实都是用一个简单的 Map 来存放数据，这些库在使用的时候，当并发低，数据量少的时候是没有问题的，但是在数据量比较大并发比较高的时候会延长 GC 时间，增加内存分配次数。

比如，我们使用一个简单的例子：

Copy
func main() {
	a := make(map[string]string, 1e9) 
	for i := 0; i < 10; i++ {
		runtime.GC()
	} 
	runtime.KeepAlive(a)
}
在这个例子中，预分配了大小是10亿（1e9) 的 map，然后我们通过 gctrace 输出一下 GC 情况：


##  0x05  参考
-	[如何打造高性能的 Go 缓存库](https://www.cnblogs.com/luozhiyun/p/14869125.html)



##  0x00    
[go-cache](https://github.com/patrickmn/go-cache)





接下来，分析下[bk-iam](https://github.com/TencentBlueKing/bk-iam/blob/master/pkg/cache/redis/redis.go)以及[gopkg](https://github.com/TencentBlueKing/gopkg/tree/master/cache)中的缓存实现：

-	`gopkg.Cache`：基于go-cache，增加了若干特性

##  0x01    Redis缓存
[Cache](https://github.com/TencentBlueKing/bk-iam/blob/master/pkg/cache/redis/redis.go)这里的RedisCache是一个三层结构，如下:


```GO
// RetrieveFunc ...
type RetrieveFunc func(key gopkgcache.Key) (interface{}, error)

// Cache is a cache implements
type Cache struct {
	name              string
	keyPrefix         string
	codec             *cache.Cache
	cli               *redis.Client
	defaultExpiration time.Duration
	G                 singleflight.Group
}
```

注意结构中的`codec`成员，是来自于`"github.com/go-redis/cache/v8"`实现的`cache.Cache`结构，具体分析建此文[go-redis/cache 库分析与使用](https://pandaychen.github.io/2023/03/25/A-GOREDIS-CACHE-ANALYSIS/)。在`Cache.Get`、`Cache.Set`等方法中，都是优先调用`c.codec`提供的方法处理，如下：


####	三层结构
```GOLANG
// Set execute `set`
func (c *Cache) Set(key gopkgcache.Key, value interface{}, duration time.Duration) error {
	if duration == time.Duration(0) {
		duration = c.defaultExpiration
	}

	k := c.genKey(key.Key())
	return c.codec.Set(&cache.Item{
		Key:   k,
		Value: value,
		TTL:   duration,
	})
}

// Get execute `get`
func (c *Cache) Get(key gopkgcache.Key, value interface{}) error {
	k := c.genKey(key.Key())
	return c.codec.Get(context.TODO(), k, value)
}
```


####    方法

```GOLANG
// GetInto will retrieve the data from cache and unmarshal into the obj
func (c *Cache) GetInto(key gopkgcache.Key, obj interface{}, retrieveFunc RetrieveFunc) (err error) {
	// 1. get from cache, hit, return
	err = c.Get(key, obj)
	if err == nil {
		return nil
	}

	// 2. if missing
	// 2.1 check the guard
	// 2.2 do retrieve
	data, err, _ := c.G.Do(key.Key(), func() (interface{}, error) {
		return retrieveFunc(key)
	})
	// 2.3 do retrieve fail, make guard and return
	if err != nil {
		// if retrieve fail, should wait for few seconds for the missing-retrieve
		// c.makeGuard(key)
		return
	}

	// 3. set to cache
	errNotImportant := c.Set(key, data, 0)
	if errNotImportant != nil {
		log.Errorf("set to redis fail, key=%s, err=%s", key.Key(), errNotImportant)
	}

	// 注意, 这里基础类型无法通过 *obj = value 来赋值
	// 所以利用从缓存再次反序列化给对应指针赋值(相当于底层msgpack.unmarshal帮做了转换再次反序列化给对应指针赋值
	return c.copyTo(data, obj)
}
```


##  0x02    cleaner.CacheCleaner
一个异步的，提供清理cache-key的方法：
```GO
// CacheCleaner ...
type CacheCleaner struct {
	name   string
	ctx    context.Context
	buffer chan cache.Key

	deleter CacheDeleter
}
```

操作：
```golang
// Run ...
func (r *CacheCleaner) Run() {
	log.Infof("running a cache cleaner: %s", r.name)
	var err error
	for {
		select {
		case <-r.ctx.Done():
			return
		case d := <-r.buffer:
			err = r.deleter.Execute(d)
			if err != nil {
				log.Errorf("delete cache key=%s fail: %s", d.Key(), err)

				// report to sentry
				util.ReportToSentry(
					"cache error: delete key fail",
					map[string]interface{}{
						"key":   d.Key(),
						"error": err.Error(),
					},
				)
			}
		}
	}
}

// Delete ...
func (r *CacheCleaner) Delete(key cache.Key) {
	r.buffer <- key
}

// BatchDelete ...
func (r *CacheCleaner) BatchDelete(keys []cache.Key) {
	// TODO: support batch delete in pipeline or tx?
	for _, key := range keys {
		r.buffer <- key
	}
}
```

##  0x03    gocache.Cache
```golang

```


##  0x04    memory.Cache




```TEXT
cache007.bifrostidctest.safe.db.:50007> ttl sam:00:iambiz:2005000692#pandaychen
(integer) 76s	
```


##	参考
-	[一次错误使用 go-cache 导致出现的线上问题](https://www.cnblogs.com/457220157-FTD/p/14707868.html)