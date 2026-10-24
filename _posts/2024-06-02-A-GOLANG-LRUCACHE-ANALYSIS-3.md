---
layout: post
title: golang 缓存设计与实现（三）
subtitle: 受限资源环境下的 cache 设计小结：LRU淘汰与自过期
date: 2024-06-02
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 缓存
---

##  0x00    开篇

笔者的采集器运行在业务机器上，对 CPU、内存等资源的占用需要严格控制。在这种受限资源环境下，cache 的设计需要兼顾以下几点：

1. **LRU 淘汰**：在内存有限的条件下，只保留最近/最常使用的数据，防止 cache 无限膨胀
2. **TTL 自过期**：缓存数据存在时效性，过期数据应该被自动清理，避免返回陈旧结果
3. **低资源占用**：不能引入过多的后台 goroutine、锁竞争或额外内存开销
4. **并发安全**：采集器内部多个模块可能并发读写 cache

本文分析对比以下四个 Go 语言 cache 实现，从数据结构、淘汰机制、TTL 支持、内存控制、并发模型等维度进行拆解：

| 库 | 定位 |
|---|---|
| [hashicorp/golang-lru](https://github.com/hashicorp/golang-lru) | 经典 LRU + 可选 TTL，HashiCorp 出品 |
| [karlseguin/ccache](https://github.com/karlseguin/ccache) | 面向高并发的近似 LRU，分桶 + 异步 worker |
| [k8s.io/utils/lru](https://github.com/kubernetes/utils/tree/master/lru) | Kubernetes 工具包中的轻量 LRU |
| [k8s.io/apimachinery/pkg/util/cache](https://github.com/kubernetes/apimachinery/tree/master/pkg/util/cache) | Kubernetes 内部使用的 LRU+TTL / 纯 TTL 缓存 |

##	0x01	hashicorp/golang-lru

[hashicorp/golang-lru](https://github.com/hashicorp/golang-lru)（v2）提供了两种主要实现：

- `simplelru.LRU`：非线程安全的基础 LRU
- `lru.Cache`：线程安全的 LRU 包装
- `expirable.LRU`：线程安全的 LRU + TTL 过期

####    核心数据结构

v2 版本使用了泛型，内部链表为自定义实现（非标准库 `container/list`），避免了 `interface{}` 装箱开销：

```golang
// 源码：https://github.com/hashicorp/golang-lru/blob/main/simplelru/lru.go
type LRU[K comparable, V any] struct {
    size      int
    evictList *internal.LruList[K, V]
    items     map[K]*internal.Entry[K, V]
    onEvict   EvictCallback[K, V]
}
```

内部链表节点 `Entry` 定义如下（[源码](https://github.com/hashicorp/golang-lru/blob/main/internal/list.go)）：

```golang
type Entry[K comparable, V any] struct {
    next, prev   *Entry[K, V]
    list         *LruList[K, V]
    Key          K
    Value        V
    ExpiresAt    time.Time
    ExpireBucket uint8
}
```

这是一个带哨兵节点（sentinel `root`）的环形双向链表，`ExpiresAt` 和 `ExpireBucket` 字段在 expirable 变体中使用。

####    LRU 淘汰机制

经典的 `map + doubly linked list` 方案，时间复杂度 O(1)：

- **Add**：key 已存在则 `MoveToFront`（更新 value），不存在则 `PushFront`；如果条目数超过 `size`，调用 `removeOldest()` 从链表尾部删除
- **Get**：命中后 `MoveToFront` 更新 LRU 顺序
- **淘汰**：`removeOldest()` 取 `evictList.Back()`（最久未使用），从 map 和链表中移除，触发 `onEvict` 回调

```golang
// 源码：https://github.com/hashicorp/golang-lru/blob/main/simplelru/lru.go
func (c *LRU[K, V]) Add(key K, value V) (evicted bool) {
    if ent, ok := c.items[key]; ok {
        c.evictList.MoveToFront(ent)
        ent.Value = value
        return false
    }
    ent := c.evictList.PushFront(key, value)
    c.items[key] = ent

    evict := c.evictList.Length() > c.size
    if evict {
        c.removeOldest()
    }
    return evict
}
```

####    TTL 过期机制（expirable 变体）

[expirable.LRU](https://github.com/hashicorp/golang-lru/blob/main/expirable/expirable_lru.go) 在基础 LRU 之上增加了 TTL 支持，核心结构如下：

```golang
// 源码：https://github.com/hashicorp/golang-lru/blob/main/expirable/expirable_lru.go
type LRU[K comparable, V any] struct {
    size      int
    evictList *internal.LruList[K, V]
    items     map[K]*internal.Entry[K, V]
    onEvict   EvictCallback[K, V]

    mu              sync.Mutex
    ttl             time.Duration
    done            chan struct{}
    buckets         []bucket[K, V]
    nextCleanupBucket uint8
    cleanupStopped  bool
}

type bucket[K comparable, V any] struct {
    entries     map[K]*internal.Entry[K, V]
    newestEntry time.Time
}
```

TTL 过期采用 **分桶轮询** 策略（`numBuckets = 100`），具体流程：

1. **写入时分桶**：`Add` 操作将 entry 的 `ExpiresAt` 设为 `now.Add(ttl)`，并将 entry 放入当前最新的 bucket（`addToBucket`）
2. **后台 goroutine 轮询**：以 `ttl/100` 为间隔触发 ticker，每次调用 `deleteExpired()` 清理一个 bucket 中的所有 entry
3. **读取时懒检查**：`Get`/`Peek` 检查 `time.Now().After(ent.ExpiresAt)`，过期则返回 miss（但不主动删除，留给后台清理）
4. **生命周期管理**：必须调用 `Close()` 停止后台 goroutine；可通过 `Restart()` 重启

```golang
// 源码：https://github.com/hashicorp/golang-lru/blob/main/expirable/expirable_lru.go
func (c *LRU[K, V]) deleteExpired() {
    c.mu.Lock()
    done := c.done
    bucketIdx := c.nextCleanupBucket
    timeToExpire := time.Until(c.buckets[bucketIdx].newestEntry)
    if timeToExpire > 0 {
        c.mu.Unlock()
        select {
        case <-time.After(timeToExpire):
        case <-done:
            return
        }
        c.mu.Lock()
        // ...
    }
    for _, ent := range c.buckets[bucketIdx].entries {
        c.removeElement(ent)
    }
    c.nextCleanupBucket = (c.nextCleanupBucket + 1) % numBuckets
    c.mu.Unlock()
}
```

分桶轮询的思路：将过期时间近似分为 `100` 个时间段，每个 ticker 周期清理一个桶，避免一次性遍历所有条目。桶的 `newestEntry` 记录该桶内最晚过期时间，清理时会先等待该时间点到来再批量清理

todo：和现网项目的思路有点类似，但是有个问题是`select ....`中的`case <-time.After(timeToExpire)`，极端情况下每个bucket都会阻塞一会吗？

####    内存控制

仅支持 **条目数上限**（`size` 参数），不支持按字节大小控制。`size == 0` 表示无上限。

####    并发安全

| 变体 | 锁类型 | Get 路径 |
|---|---|---|
| `simplelru.LRU` | 无锁（非线程安全） | - |
| `lru.Cache` | `sync.RWMutex` | **写锁**（`MoveToFront` 修改链表） |
| `expirable.LRU` | `sync.Mutex` | **互斥锁** |

注意 `lru.Cache` 的 `Get` 使用的是 `Lock()` 而非 `RLock()`，因为需要修改链表顺序。只有 `Peek`、`Contains` 等不修改 LRU 顺序的方法才使用 `RLock()`。`expirable.LRU` 则统一使用 `sync.Mutex`，所有操作完全互斥

####    性能特征

- 读写均为 O(1)（map 查找 + 链表操作）
- 单全局锁，高并发下存在竞争瓶颈
- expirable 变体额外有一个后台 ticker goroutine，资源开销较小
- 自定义链表比 `container/list` 减少了 `interface{}` 装箱和类型断言开销

##	0x02	karlseguin/ccache

[karlseguin/ccache](https://github.com/karlseguin/ccache)（v3）是一个面向高并发场景设计的 LRU cache，其核心设计理念是通过 **分桶 + 异步 worker** 将热路径上的锁竞争降到最低。

####    核心数据结构

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/cache.go
type Cache[T any] struct {
    *Configuration[T]
    control
    list            *List[T]
    size            int64
    pruneTargetSize int64
    buckets         []*bucket[T]
    bucketMask      uint32
    deletables      chan *Item[T]
    promotables     chan *Item[T]
}
```

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/item.go
type Item[T any] struct {
    key        string
    group      string
    promotions int32
    refCount   int32
    expires    int64
    size       int64
    value      T
    next       *Item[T]
    prev       *Item[T]
    inList     bool
}
```

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/bucket.go
type bucket[T any] struct {
    sync.RWMutex
    lookup map[string]*Item[T]
}
```

架构要点：
- **分桶存储**：key 经过 FNV-32a hash 后映射到某个 bucket（默认 `16` 个桶，`bucketMask = len-1`），每个 bucket 独立持有 `sync.RWMutex`
- **全局 LRU 链表**：自定义双向链表 `List[T]`，Head 端为 MRU（最近使用），Tail 端为 LRU（待淘汰）
- **异步 worker**：一个独立 goroutine 消费 `promotables` / `deletables` channel，是唯一修改链表和 `size` 的角色

####    近似 LRU 机制

ccache 的 LRU 是 **近似的**（approximate LRU），这是其高并发性能的关键设计：

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/cache.go
func (c *Cache[T]) Get(key string) *Item[T] {
    item := c.bucket(key).get(key)
    if item == nil {
        return nil
    }
    if !item.Expired() {
        select {
        case c.promotables <- item:
        default:
        }
    }
    return item
}
```

`Get` 的热路径只涉及：
1. bucket 级 `RLock` 查找 map
2. **非阻塞** 地将 item 发送到 `promotables` channel（channel 满则丢弃）

worker 收到 promote 请求后的处理逻辑：

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/cache.go
func (c *Cache[T]) doPromote(item *Item[T]) bool {
    if item.promotions == -2 {
        return false   // 已删除
    }
    if item.inList {
        if item.shouldPromote(c.getsPerPromote) {
            c.list.MoveToFront(item)
            item.promotions = 0
        }
        return false
    }
    c.size += item.size
    c.list.Insert(item)
    return true
}
```

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/item.go
func (i *Item[T]) shouldPromote(getsPerPromote int32) bool {
    i.promotions += 1
    return i.promotions == getsPerPromote
}
```

关键设计：
- **`getsPerPromote`**（默认 `3`）：item 每被 Get N 次才真正执行一次 `MoveToFront`，减少链表操作频率
- **channel 满时丢弃 promotion**：`select { case c.promotables <- item: default: }` 保证 Get 不阻塞，代价是 LRU 顺序不完全精确
- **链表操作无锁**：只有 worker goroutine 修改链表，无需加锁

####    GC 淘汰机制

当 `size > maxSize` 时触发 GC：

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/cache.go
func (c *Cache[T]) gc() int {
    dropped := 0
    item := c.list.Tail
    prunedSize := int64(0)
    sizeToPrune := c.size - c.pruneTargetSize

    for prunedSize < sizeToPrune {
        if item == nil {
            return dropped
        }
        prev := item.prev
        if !c.tracking || atomic.LoadInt32(&item.refCount) == 0 {
            c.bucket(item.key).delete(item.key)
            itemSize := item.size
            c.size -= itemSize
            prunedSize += itemSize
            c.list.Remove(item)
            if c.onDelete != nil {
                c.onDelete(item)
            }
            dropped += 1
            item.promotions = -2
        }
        item = prev
    }
    return dropped
}
```

GC 从 Tail（最久未使用）开始，批量淘汰直到 `size` 降到 `pruneTargetSize` 以下。`pruneTargetSize = maxSize - maxSize * percentToPrune / 100`（默认淘汰 `10%`），即每次 GC 不只淘汰一个 item，而是批量回收一定比例，减少 GC 频率。

####    TTL 支持

每个 item 有独立的 TTL：

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/cache.go
func (c *Cache[T]) set(key string, value T, duration time.Duration, track bool) *Item[T] {
    item, existing := c.bucket(key).set(key, value, duration, track)
    if existing != nil {
        c.deletables <- existing
    }
    c.promotables <- item
    return item
}
```

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/item.go
func (i *Item[T]) Expired() bool {
    expires := atomic.LoadInt64(&i.expires)
    return expires < time.Now().UnixNano()
}
```

注意：**`Get` 可能返回已过期的 item**，调用者需要自行检查 `item.Expired()`。ccache 没有后台定时清理 goroutine，过期 item 的回收依赖 LRU 淘汰路径（GC 从 Tail 端移除时自然清理了最久未使用的 item，其中可能包含过期的）。

####    内存大小控制：Sized 接口

ccache 是四个库中 **唯一支持自定义权重** 的实现：

```golang
// 源码：https://github.com/karlseguin/ccache/blob/master/item.go
type Sized interface {
    Size() int64
}

func newItem[T any](key string, value T, expires int64, track bool) *Item[T] {
    size := int64(1)
    if sized, ok := (interface{})(value).(Sized); ok {
        size = sized.Size()
    }
    item := &Item[T]{
        key:        key,
        value:      value,
        promotions: 0,
        size:       size,
        expires:    expires,
    }
    // ...
    return item
}
```

如果 value 实现了 `Sized` 接口，item 的 `size` 就是 `value.Size()` 的返回值；否则默认为 `1`（等价于按条目数计数）。`Cache.size` 是所有 item `size` 的累加，与 `maxSize` 比较来决定是否触发 GC。

这意味着在受限资源环境下，可以通过实现 `Sized` 接口来按字节控制 cache 内存占用：

```golang
type CachedData struct {
    Data []byte
}

func (c *CachedData) Size() int64 {
    return int64(len(c.Data))
}

cache := ccache.New(ccache.Configure[*CachedData]().MaxSize(50 * 1024 * 1024)) // 50MB
```

####    并发模型

```
┌──────────────┐    ┌──────────────┐
│  Goroutine A │    │  Goroutine B │
│   Get(key1)  │    │   Set(key2)  │
└──────┬───────┘    └──────┬───────┘
       │ RLock              │ Lock
       ▼                    ▼
┌──────────────┐    ┌──────────────┐
│   bucket[3]  │    │   bucket[7]  │   ← 不同桶无竞争
└──────┬───────┘    └──────┬───────┘
       │                    │
       │  chan promotables  │  chan promotables
       └────────┬───────────┘
                ▼
        ┌──────────────┐
        │    worker     │  ← 唯一修改链表的角色
        │  goroutine    │
        └──────────────┘
```

- **读路径**：bucket 级 `RLock` + 非阻塞 channel send，不同 bucket 之间完全无竞争
- **写路径**：bucket 级 `Lock` + channel send
- **链表操作**：全部由 worker goroutine 串行处理，无锁

这种设计使 ccache 在高并发读多写少的场景下表现优异。

##  0x03    k8s.io/utils/lru

[k8s.io/utils/lru](https://github.com/kubernetes/utils/tree/master/lru) 是 Kubernetes 工具包中的一个轻量 LRU 实现，内部 fork 自 Google 的 [groupcache/lru](https://github.com/golang/groupcache/tree/master/lru)。

####    核心数据结构

外层线程安全包装（[源码](https://github.com/kubernetes/utils/blob/master/lru/lru.go)）：

```golang
type Cache struct {
    cache *groupcache.Cache
    lock  sync.RWMutex
}
```

内部 LRU 实现（[源码](https://github.com/kubernetes/utils/blob/master/internal/third_party/forked/golang/golang-lru/lru.go)）：

```golang
type Cache struct {
    MaxEntries int
    OnEvicted  func(key Key, value interface{})
    ll         *list.List
    cache      map[interface{}]*list.Element
}

type entry struct {
    key   Key
    value interface{}
}
```

使用标准库 `container/list` 作为双向链表，`Key` 是 `interface{}` 类型。

####    LRU 淘汰机制

与 hashicorp 的基础 LRU 类似，经典 `map + list`：

```golang
// 源码：https://github.com/kubernetes/utils/blob/master/internal/third_party/forked/golang/golang-lru/lru.go
func (c *Cache) Add(key Key, value interface{}) {
    if c.cache == nil {
        c.cache = make(map[interface{}]*list.Element)
        c.ll = list.New()
    }
    if ee, ok := c.cache[key]; ok {
        c.ll.MoveToFront(ee)
        ee.Value.(*entry).value = value
        return
    }
    ele := c.ll.PushFront(&entry{key, value})
    c.cache[key] = ele
    if c.MaxEntries != 0 && c.ll.Len() > c.MaxEntries {
        c.RemoveOldest()
    }
}
```

####    TTL 支持

**不支持**。这是一个纯 LRU 淘汰的缓存，没有任何时间过期机制。

####    内存控制

仅支持 **条目数上限**（`MaxEntries`），不支持按字节大小控制。`MaxEntries == 0` 表示无上限（需要调用者自行管理淘汰）。

####    并发安全

外层用 `sync.RWMutex` 包装，但实际上 **`Get` 也使用写锁**：

```golang
// 源码：https://github.com/kubernetes/utils/blob/master/lru/lru.go
func (c *Cache) Get(key Key) (value interface{}, ok bool) {
    c.lock.Lock()
    defer c.lock.Unlock()
    return c.cache.Get(key)
}
```

因为内部 `Get` 会调用 `MoveToFront` 修改链表结构。只有 `Len()` 使用 `RLock`。

####    性能特征

- 读写 O(1)
- **极简轻量**：无后台 goroutine，无 TTL 逻辑，代码量极小
- 使用 `container/list` 和 `interface{}`，每次读写涉及类型断言，相比泛型实现有一定开销
- 单全局锁，高并发场景下存在竞争
- API 极简：`New`、`Add`、`Get`、`Remove`、`RemoveOldest`、`Len`、`Clear`

####    适用场景

Kubernetes 内部大量使用此库做简单的热点缓存（如 informer cache key 映射等）。适合对 cache 需求简单、不需要 TTL、并发压力不大的场景。

##  0x04    k8s.io/apimachinery/pkg/util/cache

[k8s.io/apimachinery/pkg/util/cache](https://github.com/kubernetes/apimachinery/tree/master/pkg/util/cache) 包含两个不同定位的缓存实现。

####    LRUExpireCache：LRU + TTL

[源码](https://github.com/kubernetes/apimachinery/blob/master/pkg/util/cache/lruexpirecache.go)

这是 Kubernetes 内部使用最广泛的缓存实现之一，同时支持 LRU 淘汰和 per-key TTL。

**核心数据结构**：

```golang
type LRUExpireCache struct {
    clock        Clock
    lock         sync.Mutex
    maxSize      int
    evictionList list.List
    entries      map[interface{}]*list.Element
}

type cacheEntry struct {
    key        interface{}
    value      interface{}
    expireTime time.Time
}
```

**Add 操作**：

```golang
// 源码：https://github.com/kubernetes/apimachinery/blob/master/pkg/util/cache/lruexpirecache.go
func (c *LRUExpireCache) Add(key interface{}, value interface{}, ttl time.Duration) {
    c.lock.Lock()
    defer c.lock.Unlock()

    oldElement, ok := c.entries[key]
    if ok {
        c.evictionList.MoveToFront(oldElement)
        oldElement.Value.(*cacheEntry).value = value
        oldElement.Value.(*cacheEntry).expireTime = c.clock.Now().Add(ttl)
        return
    }

    if c.evictionList.Len() >= c.maxSize {
        toEvict := c.evictionList.Back()
        c.evictionList.Remove(toEvict)
        delete(c.entries, toEvict.Value.(*cacheEntry).key)
    }

    entry := &cacheEntry{
        key:        key,
        value:      value,
        expireTime: c.clock.Now().Add(ttl),
    }
    element := c.evictionList.PushFront(entry)
    c.entries[key] = element
}
```

**Get 操作——惰性过期删除**：

```golang
// 源码：https://github.com/kubernetes/apimachinery/blob/master/pkg/util/cache/lruexpirecache.go
func (c *LRUExpireCache) Get(key interface{}) (interface{}, bool) {
    c.lock.Lock()
    defer c.lock.Unlock()

    element, ok := c.entries[key]
    if !ok {
        return nil, false
    }

    if c.clock.Now().After(element.Value.(*cacheEntry).expireTime) {
        c.evictionList.Remove(element)
        delete(c.entries, key)
        return nil, false
    }

    c.evictionList.MoveToFront(element)
    return element.Value.(*cacheEntry).value, true
}
```

与 hashicorp expirable 不同，`LRUExpireCache` 在 `Get` 时发现过期后会 **主动删除**（从 list 和 map 中移除），而不是仅仅返回 miss。这意味着不需要后台 goroutine 清理，过期回收完全惰性，资源占用更低。

**特点总结**：
- 同时支持 LRU 淘汰（`maxSize` 条目上限）和 per-key TTL
- `Get` 时惰性删除过期条目，无后台 goroutine
- `sync.Mutex` 全局互斥
- 支持 `Clock` 接口注入，便于单元测试
- `Keys()` 方法遍历时过滤已过期 key
- `maxSize <= 0` 会 panic
- 不支持按字节大小控制

####    Expiring：纯 TTL 过期（无 LRU）

[源码](https://github.com/kubernetes/apimachinery/blob/master/pkg/util/cache/expiring.go)

`Expiring` 是一个纯粹基于 TTL 的缓存，**不支持 LRU 淘汰**，**没有容量上限**。

**核心数据结构**：

```golang
type Expiring struct {
    AllowExpiredGet bool
    clock           clock.Clock
    mu              sync.RWMutex
    cache           map[interface{}]entry
    generation      uint64
    heap            expiringHeap
}

type entry struct {
    val        interface{}
    expiry     time.Time
    generation uint64
}

type expiringHeapEntry struct {
    key        interface{}
    expiry     time.Time
    generation uint64
}
```

**关键机制——generation 防误删**：

每次 `Set` 调用递增全局 `generation`，并同时写入 map entry 和 heap entry。当 GC 从 heap 中 pop 出过期条目时，会比较 heap entry 的 `generation` 与 map 中当前 entry 的 `generation`：

```golang
// 源码：https://github.com/kubernetes/apimachinery/blob/master/pkg/util/cache/expiring.go
func (c *Expiring) Set(key interface{}, val interface{}, ttl time.Duration) {
    now := c.clock.Now()
    expiry := now.Add(ttl)

    c.mu.Lock()
    defer c.mu.Unlock()

    c.generation++
    c.cache[key] = entry{
        val:        val,
        expiry:     expiry,
        generation: c.generation,
    }
    c.gc(now)
    heap.Push(&c.heap, &expiringHeapEntry{
        key:        key,
        expiry:     expiry,
        generation: c.generation,
    })
}

func (c *Expiring) del(key interface{}, generation uint64) {
    e, ok := c.cache[key]
    if !ok {
        return
    }
    if generation != 0 && generation != e.generation {
        return  // generation 不匹配，说明 key 已被更新，跳过删除
    }
    delete(c.cache, key)
}
```

这解决了一个经典问题：同一个 key 被多次 `Set`（续期）后，旧的 heap entry 过期时不应删除新的 map entry。通过 `generation` 匹配来保证只删除版本一致的条目。

**GC 触发时机**：

```golang
// 源码：https://github.com/kubernetes/apimachinery/blob/master/pkg/util/cache/expiring.go
func (c *Expiring) gc(now time.Time) {
    for {
        if len(c.heap) == 0 || now.Before(c.heap[0].expiry) {
            return
        }
        cleanup := heap.Pop(&c.heap).(*expiringHeapEntry)
        c.del(cleanup.key, cleanup.generation)
    }
}
```

GC 在每次 `Set` 时内联执行，从 min-heap 顶部 pop 所有已过期条目。`Get` 仅做时间检查，不触发清理。

**并发安全**：

`Expiring` 是四个实现中唯一在 `Get` 路径上使用 **真正的读锁** 的：

```golang
// 源码：https://github.com/kubernetes/apimachinery/blob/master/pkg/util/cache/expiring.go
func (c *Expiring) Get(key interface{}) (val interface{}, ok bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    e, ok := c.cache[key]
    if !ok {
        return nil, false
    }
    if !c.AllowExpiredGet && !c.clock.Now().Before(e.expiry) {
        return nil, false
    }
    return e.val, true
}
```

因为 `Get` 不修改任何数据结构（不做 MoveToFront，不删除过期条目），所以可以用 `RLock` 实现真正的读并发。这在读多写少场景下性能优势明显。

**性能特征**：
- `Get`：O(1) map 查找 + 时间比较，读锁并发
- `Set`：O(log n)（`heap.Push`）+ GC 弹出 k 个过期项 O(k log n)
- 堆中可能残留旧条目（同 key 多次 Set 不会删除旧 heap entry），内存占用会比实际条目数多
- **无容量上限**：在受限资源环境下需要注意，如果 TTL 设置不当或写入速率过高，内存会持续增长

####    两者对比

| 维度 | LRUExpireCache | Expiring |
|---|---|---|
| 淘汰策略 | LRU + TTL | 仅 TTL |
| 容量上限 | `maxSize` 条目数 | **无上限** |
| 过期清理 | `Get` 时惰性删除 | `Set` 时 min-heap GC |
| Get 路径锁 | `Mutex`（互斥） | `RWMutex.RLock`（读共享） |
| 后台 goroutine | 无 | 无 |
| Set 复杂度 | O(1) | O(log n) |

##  0x05    四者对比总结

| 维度 | hashicorp/golang-lru (expirable) | karlseguin/ccache | k8s.io/utils/lru | k8s apimachinery LRUExpireCache | k8s apimachinery Expiring |
|---|---|---|---|---|---|
| **淘汰策略** | LRU + TTL | 近似 LRU + TTL | 纯 LRU | LRU + TTL | 纯 TTL |
| **容量上限** | 条目数 | 权重之和（可自定义） | 条目数 | 条目数 | **无上限** |
| **内存字节控制** | 不支持 | 支持（`Sized` 接口） | 不支持 | 不支持 | 不支持 |
| **TTL 支持** | 全局统一 TTL | per-item TTL | 不支持 | per-key TTL | per-key TTL |
| **过期清理方式** | 后台 goroutine 分桶轮询 + 读时懒检查 | 无主动清理（依赖 LRU GC） | - | Get 时惰性删除 | Set 时 min-heap GC |
| **Get 路径锁** | `Mutex` | bucket 级 `RLock` | `Mutex`（名义 RWMutex） | `Mutex` | `RLock`（读共享） |
| **后台 goroutine** | 1 个 ticker | 1 个 worker | 无 | 无 | 无 |
| **泛型支持** | Go 1.18+ 泛型 | Go 1.18+ 泛型 | 无（`interface{}`） | 无（`interface{}`） | 无（`interface{}`） |
| **链表实现** | 自定义 LruList | 自定义 List | `container/list` | `container/list` | 无链表（仅 heap） |
| **Get 复杂度** | O(1) | O(1) | O(1) | O(1) | O(1) |
| **Set/Add 复杂度** | O(1) | O(1) | O(1) | O(1) | O(log n) |
| **代码量** | 中等 | 较多 | 极少 | 少 | 少 |

##  0x06    受限资源环境下的选型建议

针对采集器在业务机器上运行、资源严格受限的场景，选型建议如下：

####    场景一：需要 LRU + TTL，并发不高

推荐 **`k8s.io/apimachinery/pkg/util/cache.LRUExpireCache`**：
- 同时具备 LRU 淘汰和 per-key TTL，满足大多数采集器缓存需求
- 无后台 goroutine，资源占用最低
- Get 时惰性删除过期条目，不会有清理延迟导致的内存堆积
- 实现简洁，行为可预测

如果需要泛型和更丰富的 API（`Peek`、`Contains`、`Resize` 等），可以选择 **`hashicorp/golang-lru/expirable`**，但需要注意管理 `Close()` 的调用来停止后台 goroutine。

####    场景二：高并发读，需要极低的锁竞争

推荐 **`karlseguin/ccache`**：
- 分桶 + 异步 worker 的设计在高并发读场景下锁竞争极小
- 支持 `Sized` 接口按字节控制内存占用
- 近似 LRU 在大多数场景下足够
- 需要注意：`Get` 可能返回已过期 item，调用者需自行检查

####    场景三：简单场景，只需 LRU

推荐 **`k8s.io/utils/lru`**：
- 极简轻量，零依赖（除 Kubernetes utils 内部）
- 无后台 goroutine，无 TTL 开销
- 适合缓存生命周期短、由 LRU 淘汰就足够的场景

####    场景四：读多写少，仅需 TTL 过期

推荐 **`k8s.io/apimachinery/pkg/util/cache.Expiring`**：
- `Get` 使用真正的读锁（`RLock`），读并发性能最好
- 适合 token 缓存、DNS 解析缓存等纯时效性数据
- **注意无容量上限**，必须确保 TTL 设置合理且写入速率可控，否则内存会持续增长

####    关于内存字节控制

四个库中，只有 **ccache** 通过 `Sized` 接口支持按字节（或自定义权重）控制缓存大小。其余三个均只支持条目数上限。如果严格的内存字节控制是硬性需求，ccache 是唯一直接支持的选项。对于其他库，可以通过估算单条目平均大小、合理设置条目数上限来间接控制内存占用。

##  0x07	参考
-	[hashicorp/golang-lru](https://github.com/hashicorp/golang-lru)
-   [hashicorp/golang-lru expirable 源码](https://github.com/hashicorp/golang-lru/blob/main/expirable/expirable_lru.go)
-   [hashicorp/golang-lru simplelru 源码](https://github.com/hashicorp/golang-lru/blob/main/simplelru/lru.go)
-   [hashicorp/golang-lru internal list 源码](https://github.com/hashicorp/golang-lru/blob/main/internal/list.go)
- [karlseguin/ccache](https://github.com/karlseguin/ccache)
-   [karlseguin/ccache cache.go 源码](https://github.com/karlseguin/ccache/blob/master/cache.go)
-   [karlseguin/ccache item.go 源码](https://github.com/karlseguin/ccache/blob/master/item.go)
- [k8s.io/utils/lru](https://github.com/kubernetes/utils/tree/master/lru)
-   [k8s.io/utils/lru 源码](https://github.com/kubernetes/utils/blob/master/lru/lru.go)
-   [k8s.io/utils 内部 forked golang-lru 源码](https://github.com/kubernetes/utils/blob/master/internal/third_party/forked/golang/golang-lru/lru.go)
- [k8s.io/apimachinery/pkg/util/cache](https://github.com/kubernetes/apimachinery/tree/master/pkg/util/cache)
-   [LRUExpireCache 源码](https://github.com/kubernetes/apimachinery/blob/master/pkg/util/cache/lruexpirecache.go)
-   [Expiring 源码](https://github.com/kubernetes/apimachinery/blob/master/pkg/util/cache/expiring.go)
