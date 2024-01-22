---
layout:     post
title:      微服务中的缓存（四）：内存缓存使用的一些技巧
subtitle:	记录项目中遇到的问题及解决方案
date:       2022-10-03
author:     pandaychen
header-img:	img/golang-horse-fly.png
catalog:    true
tags:
    - Cache
    - 缓存
---


##  0x00    前言
本文梳理下笔者在项目开发中使用过一些内存缓存的技巧

##	0x01	双缓冲：double buffering
有一个场景是，存在某一本地文件配置（修改少），程序初始化时把文件内容读取到本地内存里（假设为 map1），由于程序逻辑需要频繁且高性能的读取 map1（尽量不加锁），在这种背景下如何实现安全的修改文件自动同步到 `map` 里（且不加锁）？有两种思路：

1.	分段锁
2.	使用双缓冲（double buffering）技术，方法是在后台修改一个副本的 `map`，当修改完成后，将其原子性地替换为当前活动的 `map`。这样在修改期间，程序可以继续高性能（原子）读取当前活动的 map，修改完成后，将当前活动的 `map` 指向已经完成新一轮加载读取的 map（ping-pong map）

参考代码如下，`MapHolder` 包含 `2` 个 `map` 和一个 atomic 变量 `idx`。`LoadFile` 方法读取文件内容并将其加载到一个新的 `map` 中，然后原子性地替换当前活动的 `map`。`Get` 方法根据当前活动的 `map` 来获取键值对；如此实现允许在不加锁的情况下实现对 `map` 的高性能访问。但注意该实现仅适用于读密集型场景，如果需要同时进行大量的写操作则不建议使用此方法

```GO
type MapHolder struct {
	maps [2]*map[string]string
	idx  int32
}

func NewMapHolder() *MapHolder {
	m1 := make(map[string]string)
	m2 := make(map[string]string)
	return &MapHolder{maps: [2]*map[string]string{&m1, &m2}}
}

func (h *MapHolder) LoadFile(filename string) error {
	file, err := os.Open(filename)
	if err != nil {
		return err
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	newMap := make(map[string]string)
	for scanner.Scan() {
		parts := strings.Split(scanner.Text(), ":")
		if len(parts) == 2 {
			newMap[parts[0]] = parts[1]
		}
	}

	// 原子性地替换活动 map
	idx := atomic.LoadInt32(&h.idx)
	atomic.StorePointer((*unsafe.Pointer)(unsafe.Pointer(&h.maps[idx^1])), unsafe.Pointer(&newMap))
	atomic.StoreInt32(&h.idx, idx^1)

	return scanner.Err()
}

func (h *MapHolder) Get(key string) (string, bool) {
	idx := atomic.LoadInt32(&h.idx)     // 原子获取当前活动的 index
	m := h.maps[idx]
	value, ok := (*m)[key]
	return value, ok
}

func main() {
	holder := NewMapHolder()
	err := holder.LoadFile("file.txt")
	if err != nil {
		fmt.Println("Error loading file:", err)
		return
	}

	go func() {
		for {
			time.Sleep(1 * time.Second)
			err := holder.LoadFile("file.txt")
			if err != nil {
				fmt.Println("Error reloading file:", err)
			}
		}
	}()

	for {
		time.Sleep(100 * time.Millisecond)
		value, ok := holder.Get("a")
		if ok {
			fmt.Println("Value of a:", value)
		} else {
			fmt.Println("Key a not found")
		}
	}
}
```

##	0x02	go-cache 的使用
前面文章介绍了不少实用的内存 hashtable 结构实现，不过如果项目 ** 对性能要求不高 **，可以考虑使用 [go-cache](https://github.com/patrickmn/go-cache)，该项目的特点就是足够简单，存取不需要序列化 / 反序列化，支持 Value 过期，对应 `expired` 的逻辑就是启动个 ticker 定时器，定时全量扫描 map 进行回收，如下：

```GO
func (j *janitor) Run(c *cache) {
	ticker := time.NewTicker(j.Interval)
	for {
		select {
		case <-ticker.C:
			//
			c.DeleteExpired()
		case <-j.stop:
			ticker.Stop()
			return
		}
	}
}

// Delete all expired items from the cache.
func (c *cache) DeleteExpired() {
	var evictedItems []keyAndValue
	now := time.Now().UnixNano()
	c.mu.Lock()
	// 锁表：先扫描哪些需要被删除的 kv 并收集
	for k, v := range c.items {
		// "Inlining" of expired
		if v.Expiration > 0 && now > v.Expiration {
			ov, evicted := c.delete(k)
			if evicted {
				evictedItems = append(evictedItems, keyAndValue{k, ov})
			}
		}
	}
	// 全部处理完再解锁
	c.mu.Unlock()
	for _, v := range evictedItems {
		c.onEvicted(v.key, v.value)
	}
}
```

使用 go-cache 要注意下面几点：
1.	加锁力度过粗，项目需要衡量好并发后再使用；高 QPS 的接口尽量不要去直接 `Set` 数据, 如果必须 `Set` 考虑采用异步操作
2.	定时清理逻辑中，如果耗时过长，又整个 cache 加锁，可能会导致其他的 goroutine 阻塞等待在 lock 上无法写入，参考 [一次错误使用 go-cache 导致出现的线上问题](https://www.cnblogs.com/457220157-FTD/p/14707868.html) 此文的坑
3.	关于 `map[string]interface{}` 存储的 value，有可能会改变；如果存的是 slice/map 或者指针等，当取出使用的时候，修改值，会导致缓存中的原始值变化；此外，如果是非线程安全的 value，还 ** 必须考虑读写加锁以实现并发操作安全 **，看下面的例子
4.	尽量存放那些相对不怎么变化的数据, 适用于所有的 local cache（包括 map, `sync.map`）
5.	监控 go-cache 里面 key 的数量, 如果过多时, 需要及时调整参数
6.	go-cache 的过期检查时间要设置相对较小, 也不能过小

```go
// 一个修改了 value 指针的例子
func main() {
        c := cache.New(5*time.Minute, 10*time.Minute)
        var tmap = make(map[int]int)
        c.Set("foo", tmap, cache.DefaultExpiration)
        foo, found := c.Get("foo")
        if found {
                fmt.Println(foo)
        }
        foom, _ := foo.(map[int]int)
		// 如果是并发操作，还需要考虑并发安全
        foom[1] = 1
        foo, found = c.Get("foo")
        if found {
                fmt.Println(foo)
        }
}
```

##	0x03	一个典型的缓存使用：bk-iam
本小节，分析下 [bk-iam](https://github.com/TencentBlueKing/bk-iam/blob/master/pkg/cacheimpls/init.go) 项目给出的典型的缓存应用，包含下面 `4` 种用法：

1.	`gocache.Cache`
2.	`memory.Cache`
3.	`redis.Cache`
4.	`cleaner.CacheCleaner`

####	`memory.Cache`

####	`gopkg.Cache`
基于 go-cache 实现，增加了若干特性

####	`redis.Cache`：提供缓存 + 外部数据接口
[`redis.Cache`](https://github.com/TencentBlueKing/bk-iam/blob/master/pkg/cache/redis/redis.go) 是一个封装了 `go-redis/cache` 的共享缓存实现:

结构如下：
```go
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

// NewCache create a cache instance
func NewCache(name string, expiration time.Duration) *Cache {
	cli := GetDefaultRedisClient()

	// key format = iam:{version}:{cache_name}:{real_key}
	keyPrefix := fmt.Sprintf("iam:%s:%s", CacheVersion, name)

	// 初始化的时候没有传入 localcache 实现
	codec := cache.New(&cache.Options{
		Redis: cli,
	})

	return &Cache{
		name:              name,
		keyPrefix:         keyPrefix,
		codec:             codec,
		cli:               cli,
		defaultExpiration: expiration,
	}
}
```

注意结构中的 `codec` 成员，是来自于 `"github.com/go-redis/cache/v8"` 实现的 `cache.Cache` 结构，具体分析参考此文 [go-redis/cache 库分析与使用](https://pandaychen.github.io/2023/03/25/A-GOREDIS-CACHE-ANALYSIS/)。在 `Cache.Get`、`Cache.Set` 等方法中，都是优先调用 `c.codec` 提供的方法处理（代码如下）

另外还有一个细节是，`codec` 初始化时，并未传入 localcache 实现，所以从 [实现](https://github.com/go-redis/cache/blob/v8/cache.go#L161
) 看，`codec` 就仅仅退化为 redis 缓存了

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

// Delete execute `del`
func (c *Cache) Delete(key gopkgcache.Key) (err error) {
	k := c.genKey(key.Key())

	ctx := context.TODO()

	_, err = c.cli.Del(ctx, k).Result()
	return err
}
```

该结构提供的所有方法可以参考 [代码](https://github.com/TencentBlueKing/bk-iam/blob/master/pkg/cache/redis/redis.go)

此外，结构还提供了 `GetInto` 方法，并且用 singleflight 进行了封装，意义在于当缓存失效时，从远端（如数据库 / 接口等）获取数据并设置缓存，不过这里为啥 `c.Set` 逻辑不一齐放置在 singleflight 里面？


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
	// 所以利用从缓存再次反序列化给对应指针赋值 (相当于底层 msgpack.unmarshal 帮做了转换再次反序列化给对应指针赋值
	return c.copyTo(data, obj)
}
```

####	`cleaner.CacheCleaner`
`cleaner.CacheCleaner` 提供了一种异步的，提供清理 cache-key 的方法

-	通过 `buffer` 提供异步删除的 key 通道
-	开发者需要使用 `CacheDeleter` 接口实现 `Execute` 方法，实现删除 key 的方法

```GO
// CacheDeleter ...
type CacheDeleter interface {
	Execute(key cache.Key) error
}

// CacheCleaner ...
type CacheCleaner struct {
	name   string
	ctx    context.Context
	buffer chan cache.Key	// 待删除 key 的异步队列
	deleter CacheDeleter	// 删除 key 的方法
}
```

操作方式也比较直观，经典的模型，调用开发者实现 `Execute` 执行删除逻辑，如下：

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

具体例子可以参考 [`systemCacheDeleter`](https://github.com/TencentBlueKing/bk-iam/blob/master/pkg/cacheimpls/deleter.go#L89)，比如可以支持异步删除多个其他类型的缓存：

```go
func (d systemCacheDeleter) Execute(key cache.Key) (err error) {
	err = multierr.Combine(
		SystemCache.Delete(key),
		LocalSystemClientsCache.Delete(key),
	)
	return
}
```

##	0x03	参考
-	[应用双缓冲技术完美解决资源数据优雅无损的热加载问题](http://blog.codeg.cn/2016/01/27/double-buffering/)
-	[一次错误使用 go-cache 导致出现的线上问题](https://www.cnblogs.com/457220157-FTD/p/14707868.html)
-	[go-cache](https://github.com/patrickmn/go-cache)
-	[如何打造高性能的 Go 缓存库](https://www.cnblogs.com/luozhiyun/p/14869125.html)
-	[cacheimpls](https://github.com/TencentBlueKing/bk-iam/tree/master/pkg/cacheimpls)