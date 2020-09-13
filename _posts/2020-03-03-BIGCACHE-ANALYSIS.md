---
layout:     post
title:      Golang 高性能 LocalCache：BigCache 设计与分析
subtitle:   如何在 Golang 构建一个高性能的本地缓存
date:       2020-03-03
author:     pandaychen
header-img: img/golang-tools-fun.png
catalog: true
category:   false
tags:
    - Cache
---


##	0x00	前言
通常在 Golang 中，缓存的实现离不开如下几种：
1.	原生 `map`
2.	`sync.Map`
3.	基于以上二者封装的复合型 `map`

本篇文章要分析的项目 [BigCache](https://github.com/allegro/bigcache) ：就是这么一种高性能复合封装 `map` 的实现。


##  0x01    为什么使用 BigCache
后台业务开发中，需要本地缓存数据的场景特别常见，而 Go 语言实现的 bigCache 本地缓存库以其快速、并发和高性能著称，它可以存储百万级的数据。bigCache 将数据保存到堆中，为了规避了 golang GC 对它的影响，map 结构中 key 和 value 中均不存储指针类型数据，底层数据结构采用 bytes 切片。

这样 GC 就变成了 map 无指针 +[]byte 结构的扫描问题了，因此性能会高出很多。

总结：
1、尝试使用 sync.Map（并发环境下）
2、分片锁，降低锁竞争


##	0x02	核心结构

2、核心的 [存储结构 [`queue.BytesQueue`](https://github.com/allegro/bigcache/blob/master/queue/bytes_queue.go)：真正的数据以二进制序列化后存储的位置。
```golang
// BytesQueue is a non-thread safe queue type of fifo based on bytes array.
// For every push operation index of entry is returned. It can be used to read the entry later
type BytesQueue struct {
	full            bool
	array           []byte
	capacity        int
	maxCapacity     int
	head            int
	tail            int
	count           int
	rightMargin     int
	headerBuffer    []byte		// 插入前做临时 buffer 所用（slice-copy）
	verbose         bool
	initialCapacity int
}
```

##  0x03    bigCache && shard 分片
针对并发而做的分片 shard 优化，已经是常用套路了。为了避免协程并发访问，单个锁成为系统的瓶颈，bigCache 采用 shards 的方式来解决：它将全部要存储的数据划分成若干个 shard 独立管理，而每一个 shard 都拥有一个锁，这样每个锁只负责一部分的数据。能够减少并发读写对 lock 的调用。

```golang
type BigCache struct {
    shards       []*cacheShard	// 分片结构
    lifeWindow   uint64 // 数据过期时间
    clock        clock  // 获取数据存储时的时间戳
    hash         Hasher // 计算请求数据 key 的 hash 值
    config       Config
    shardMask    uint64
    maxShardSize uint32 // 每个 shard 中 byte 数组的最大容量
    close        chan struct{}
}
```

`cacheShard` 结构如下：
单个 shard 的 [结构](https://github.com/allegro/bigcache/blob/master/shard.go) 如下：
```golang
type cacheShard struct {
	hashmap     map[uint64]uint32		 // 存储数据在 entries 中具体位置, key 为数据 key 的 hash，value 为位置
	entries     queue.BytesQueue		// 数据最终存储的位置
	lock        sync.RWMutex
	entryBuffer []byte
	onRemove    onRemoveCallback	 // 删除数据时提供的回调函数

	isVerbose    bool
	statsEnabled bool
	logger       Logger
	clock        clock
	lifeWindow   uint64

	hashmapStats map[uint64]uint32
	stats        Stats
}
```

我们以 `bigCache.Get` 方法为例：
1.  根据 `fnv` 算法计算出用户 key 的 `hash` 值 `hashedKey`
2.	`getShard` 方法：通过 `hashedKey` 获取数据存储到哪一个 `shard` 中
3.  调用 `shard` 的 `get` 方法获取存储的数据
```golang
func (c *BigCache) Get(key string) ([]byte, error) {
	hashedKey := c.hash.Sum64(key)
	shard := c.getShard(hashedKey)
	return shard.get(key, hashedKey)
}

//getShard 方法：通过 `hashedKey` 获取数据存储到哪一个 `shard 中
func (c *BigCache) getShard(hashedKey uint64) (shard *cacheShard) {
    return c.shards[hashedKey&c.shardMask]
}
```

与 shard 相关的一些关键方法如下：
1、`Sum64`：用于用户 key 转换，请求数据的 key 转换为实际存储在 `cacheShard.hashmap` 的 key，这里是使用 `fnv` 算法来计算的
```golang
func (f fnv64a) Sum64(key string) uint64 {
    var hash uint64 = offset64
    for i := 0; i <len(key); i++ {
        hash ^= uint64(key[i])
        hash *= prime64
    }
    return hash
}
```


####	处理用户数据（数据序列化）
在许多高性能的组件实现，针对数据部分的存储大都会将其由 `string` 类型按照一定的 pack 格式转为 `binary` 类型以减少内存占用，在 bigCache 也是类似做法，每个要插入的 key-value 由 5 部分组成，分别是时间戳 (8byte)、key 的 hash 值 (8byte)、key 的长度 (2byte)、key 的值以及 value 的值，序列化时采用 `LittleEndian` 小端序。对 timestamp 和 hash 和 key 和 value 封装在一起。这种形式的存储和 `leveldb` 的实现类似。

在 BigCache 中，数据采用 binary 方式进行序列化，由 map 保存 key-value 的索引位置，如果要进行查找和删除数据可以通过 map 找到数据保存的位置。
```golang
const (
    timestampSizeInBytes = 8  // 存放时间戳
    hashSizeInBytes      = 8  // 存放 hash 值
    keySizeInBytes       = 2  // 存放 key 的长度
    headersSizeInBytes   = timestampSizeInBytes + hashSizeInBytes + keySizeInBytes // header 的长度
)
```
最终的存储 Unit 如下图所示：
![img]()

`wrapEntry` 完成的上述 pack 的过程（通用的方法），将 key 和 value 都 pack：
```golang
func wrapEntry(timestamp uint64, hash uint64, key string, entry []byte, buffer *[]byte) []byte {
    keyLength := len(key)  // key 的长度
    blobLength := len(entry) + headersSizeInBytes + keyLength

    if blobLength > len(*buffer) {
        *buffer = make([]byte, blobLength)
    }
    blob := *buffer
  	// 数据存储采用小端序 LittleEndian
    binary.LittleEndian.PutUint64(blob, timestamp)
    binary.LittleEndian.PutUint64(blob[timestampSizeInBytes:], hash)
    binary.LittleEndian.PutUint16(blob[timestampSizeInBytes+hashSizeInBytes:], uint16(keyLength))
    copy(blob[headersSizeInBytes:], key)
    copy(blob[headersSizeInBytes+keyLength:], entry)
    return blob[:blobLength]
}
```


##	0x04	BytesQueue 结构
[`BytesQueue` 结构](https://github.com/allegro/bigcache/blob/master/queue/bytes_queue.go)，用来存储上面序列化后的数据，它使用 `[]byte` 类型来模拟队列，插入数据从 tail 位置，删除数据从 head 位置。有些像低配版本的 `bytes.Buffer`。

```golang
type BytesQueue struct {
    array           []byte
    capacity        int    // array 的容量
    maxCapacity     int    // array 可以申请的最大容量
    head            int    // 起始位置
    tail            int    // 下次可以插入 item 的位置
    count           int    // 插入的 item 数量
    rightMargin     int    // array 当前最后一个 item 的位置
    headerBuffer    []byte
    verbose         bool   // array 扩容时是否打印信息
    initialCapacity int    // BytesQueue 创建时，array 的初始容量
}
```

####    数据操作
head 和 tail 是一个相对的位置，head 不一定一直在 tail 的前面，比如随着数据的插入，tail 已处于 array 的最后面，bigCache 会尝试从 head 前面查找是否还有可以插入的位置，如果插入成功，则 head 就会在 tail 的后面。

BytesQueue 不同于队列，head 所指向的元素不一定是最早插入的元素，tail 指向的元素也不是最晚插入 BytesQueue 的，它们可能会因为扩容是发生变化。因此它们的作用不是用来判断数据的新旧程度，而是用来判断是否可以插入新的元素，判断数据是否过期是使用的元素中的 timestamp 字段。

![img]()

####    allocateAdditionalMemory 方法

```golang
func (q *BytesQueue) allocateAdditionalMemory(minimum int) {
    start := time.Now()
    if q.capacity < minimum {
        q.capacity += minimum
    }
    q.capacity = q.capacity * 2
    if q.capacity > q.maxCapacity && q.maxCapacity > 0 {
        q.capacity = q.maxCapacity
    }
    oldArray := q.array
    q.array = make([]byte, q.capacity)
    if leftMarginIndex != q.rightMargin {
        copy(q.array, oldArray[:q.rightMargin])
        if q.tail < q.head {
            emptyBlobLen := q.head - q.tail - headerEntrySize
            q.push(make([]byte, emptyBlobLen), emptyBlobLen)
            q.head = leftMarginIndex // head 指向了后插入的数据
            q.tail = q.rightMargin
        }
    }
}
```
rightMargin，用于标识队列中最后一个元素的位置，是一个绝对位置，当队列需要扩容时，会 copy 该坐标之前的所有元素。

BytesQueue 中的每个 item 都由 2 部分组成，前 4 个 byte 是数据的长度，后面是数据的值。

####	bytesQueue.copy
```golang
func (q *BytesQueue) copy(data []byte, len int) {
   q.tail += copy(q.array[q.tail:], data[:len])
}
```

####	写入数据
使用 `push` 方法将 `data` 写入 `array []byte` 写入数据：
```golang
func (q *BytesQueue) push(data []byte, len int) {
    binary.LittleEndian.PutUint32(q.headerBuffer, uint32(len))
    q.copy(q.headerBuffer, headerEntrySize)
    q.copy(data, len)
    if q.tail > q.head {
        q.rightMargin = q.tail
    }
    q.count++
}
```

##	0x05	参考
-	[本地缓存 BigCache](https://neojos.com/blog/2018/08-19-%E6%9C%AC%E5%9C%B0%E7%BC%93%E5%AD%98bigcache/)