---
layout:     post
title:      Redis 应用梳理篇（一）
subtitle:   如何在实战项目中灵活使用 Redis 及排坑
date:       2020-01-21
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Redis
---

##	0x00	前言
&emsp;&emsp; Redis 是 Linux C 项目中单线程开发模型的典范，其内置了多种高性能的数据结构，如 sds 动态字符串、skiplist 跳表，zset，hashset 等。今天这篇文章是想分享下项目中使用 Redis 的一些技巧，结合 [go-redis](https://github.com/go-redis/redis) 这个客户端库。

##  0x01    Pipeline 加速
&emsp;&emsp; 在大批量的操作 Redis（写入 / 读取）时，采用 `pipeline` 是个非常好的优化思路，可以提高在单个 Redis 连接操作的数据吞吐量，从而提升整体性能。
通过 `pipeline` 方式将 client 端命令（打包后）一起发出，Redis 服务端会处理完多条命令后，将结果一起打包返回 client，从而节省大量的网络延迟开销。这里需要注意两点：
1.  服务端的模式（主从 Sentinel、集群 Cluster、Proxy）是否支持 `pipeline` 操作？
2.  项目中使用的 Redis 客户端库，是否支持 pipeline 操作？
3.  一次 `pipeline.Exec()` 操作打包多少条 Redis 命令？

这里有趣的问题是第 3 个，从下面这个问题中，大概可以找到答案：
[How many commands could redis-py pipeline have?](https://stackoverflow.com/questions/27007530/how-many-commands-could-redis-py-pipeline-have)

>Not sure that there is a maximum, but I don't think that you want to reach it in case there is.
>In most cases, limiting the size of the pipeline to 100-1000 operations gives the best results. But, you can do a little benchmark >research that includes typical requests that you send. Pipelining requests is generally good, but keep in mind that the responses >are kept in Redis memory until all pipeline requests are served, and your client waits for the long reply of all requests.
>You should try to find the sweet spot of concurrent connections, pipelined requests and your Redis memory.


##  0x02    Pool 连接池使用
&emsp;&emsp; 连接池的好处，在于避免每次 Redis 操作时，新建 TCP 连接的开销，在要求高性能的场景推荐开启。个人建议，使用的 Redis 客户端库，包含的连接池至少满足下面的特性：
1.  连接健康检查
2.  自动关闭空连接

下面这张表格总结了常见的 Golang 的 Redis 客户端库的连接池特性，笔者在项目中使用的连接池是基于 [go-redis](https://github.com/go-redis/redis/tree/master/internal/pool) 的。

| 连接池功能 | go-redis | radix.V2 | redigo|
| :-----:| :----: | :----: | :----: |
| 实现数据结构 | slice | channel | linklist|
| 连接健康检查 | No | Yes | Yes|
| 自动关闭空闲连接 | Yes| No| Yes|
| 连接的 Max 生存时间 | Yes| No| Yes|

关于 go-redis 连接池的实现可见此文 [Go-Redis 连接池（Pool）源码分析](https://pandaychen.github.io/2020/02/22/A-REDIS-POOL-ANALYSIS/)

####  健康度检查
使用 go-redis 初始化的时候，建议开启如下配置：
```golang
func InitRedisClient(config *config.Config) error {
        redisCfg := config.RedisConfig
        if redisCfg.PoolSize == 0 {
                redisCfg.PoolSize = 5
        }
        redisClient = redis.NewClient(&redis.Options{
                RedisAlive:   0, // 健康度标识
                DB:           redisCfg.Db,
                Addr:         redisCfg.Addr,
                Password:     redisCfg.Password,
                PoolSize:     redisCfg.PoolSize,    // 连接池大小
                MinIdleConns: redisCfg.MinIdleConns,
                IdleTimeout:  time.Duration(redisCfg.IdleTimeout) * time.Second,
        })
        _, err := redisClient.Ping(context.TODO()).Result()
        if err != nil {
                return err
        }
        // 是否支持特殊指令
        enableSpecialCommand = redisCfg.EnableSpecialCommand

        go StartRedisMonitor()
        return nil
}

//异步监控
func (c *Client) StartRedisMonitor() {
        ticker := time.NewTicker(pingInterval)
        for range ticker.C {
                if _, err := lim.store.Ping(context.TODO().Result()); err == nil {
                  atomic.StoreUint32(&c.redisAlive, 1)
                } else {
                  //  健康度检查失败
                  atomic.StoreUint32(&c.redisAlive, 0)
                }
        }
}
```

##  0x03    Redis 常用操作

####  Set 集合
Set 作为 CS 中最基础的数据结构，使用非常广泛，Redis 的 Set 在高性能查找中扮演着极为重要的角色。举例来说，`SInter(Store)` 和 `SUnion(Store)` 及 `SDiff(STORE)` 这几个操作，取（多个）集合的交集、并集及差集并直接存储。在实际项目中，如计算多个子集合间的交集、并集和差集，非常方便。

-   `sinter key [key ...]` 返回一个集合的全部成员，该集合是所有给定集合的交集
-   `sunion key [key ...]` 返回一个集合的全部成员，该集合是所有给定集合的并集
-   `sdiff key [key ...]`  返回所有给定 key 与第一个 key 的差集
-   `sinterstore destination key [key ...]` 将 交集 数据存储到某个对象中
-   `sunionstore destination key [key ...]` 将 并集 数据存储到某个对象中
-   `sdiffstore destination key [key  ...]` 将 差集 数据存储到某个对象中

举例看下：

1、计算 `Set5` 和 `Set1~4` 之间并集的交集 <br>
```bash
127.0.0.1:6379> sadd set1 a b c d e f g
(integer) 7
127.0.0.1:6379>
127.0.0.1:6379> sadd set2 a b c d e f g h i j k l m n
(integer) 14
127.0.0.1:6379> sadd set3 a c d e f g h i j
(integer) 9
127.0.0.1:6379> sadd set4 a b c e f g h i j m n
(integer) 11
127.0.0.1:6379>
127.0.0.1:6379> sadd set5 b c d g h j q z
(integer) 8
127.0.0.1:6379>
127.0.0.1:6379> SUNIONSTORE tempset set1 set2 set3 set4
(integer) 26
127.0.0.1:6379>
127.0.0.1:6379> SINTER tempset set5
1) "h"
2) "g"
3) "j"
4) "d"
5) "c"
6) "b"
```

2、复制一个 Set<br>
```bash
127.0.0.1:6379> SADD set1 a b c
(integer) 3

127.0.0.1:6379> SUNIONSTORE set2 set1 temp
(integer) 3
127.0.0.1:6379> smembers set1
1) "b"
2) "a"
3) "c"
127.0.0.1:6379> smembers set2
1) "a"
2) "b"
3) "c"
127.0.0.1:6379> smembers temp       //temp 并不存在的 key
(empty list or set)
```

####  Hash
Hash 是一个 `string` 类型的 field（字段） 和 value（值） 的映射表，特别适合用于存储对象，参考如下操作：

```bash
127.0.0.1:6379>   HMSET runoobkey name "redis tutorial" description "redis basic commands for caching" likes 20 visitors 23000
OK
127.0.0.1:6379> HGETALL runoobkey
1) "name"
2) "redis tutorial"
3) "description"
4) "redis basic commands for caching"
5) "likes"
6) "20"
7) "visitors"
8) "23000"
127.0.0.1:6379> HGETALL runoobkey
1) "name"
2) "redis tutorial"
3) "description"
4) "redis basic commands for caching"
5) "likes"
6) "20"
7) "visitors"
8) "23000"
127.0.0.1:6379> HEXISTS name
(error) ERR wrong number of arguments for 'hexists' command
127.0.0.1:6379> HEXISTS runoobkey name
(integer) 1
127.0.0.1:6379> HGET runoobkey name
"redis tutorial"
127.0.0.1:6379> HGET runoobkey name
"redis tutorial"
```

####  ZSET：有序不重复集合
ZSET即有序集合，通常用来实现延时队列或者排行榜（如销量 / 积分排行、成绩排行等等）。ZSET 通常包含 `3` 个 关键字操作：
- `key` （与 redis 通常操作的 key value 中的 key 一致）
- `score` （**用于排序的分数**，该分数是有序集合的关键，可以是双精度或者是整数）
- `member` （指传入的 obj，与 key value 中的 value 一致）

几个细节：
1.  `ZADD`，如果操作的 `key` 中已经有了相同的 `member`，则更新该 `member` 的 `score` 值，并会重排序
2.  `ZRANGE/ZREVRANGE`，按照 `score` 排序并返回，注意需要传入 `score` 的 [start,end] 区间
3.  ZSET 每个元素都会关联一个 `float64` 类型的分数，Redis 是通过分数来为集合中的成员进行从小到大的排序
4.  ZSET 的成员 `member` 是唯一的, 但分数 `score` 却允许重复
5.  ZSET 是通过哈希表实现的，所以添加 / 删除 / 查找的复杂度都是 `O(1)`
6.  从使用意义来看，`ZSET` 中只有 `score` 值非常重要，`value` 值没有特别的意义，只需要保证它是唯一的就可以

```golang
127.0.0.1:6379>  zadd key1 1 a 2 b 3 c
(integer) 3
127.0.0.1:6379> zrange key1 0 -1
1) "a"
2) "b"
3) "c"
127.0.0.1:6379> zrevrange key1 0 -1
1) "c"
2) "b"
3) "a"
```

####  【WARNING】慎用的操作
在项目实践中，也遇到了一些性能问题，如下：
-   慢操作：`SMEMBERS` 命令是从一个 SET 结构获取集合，但这个集合数据量已经很大，当这个命令被频繁的调用，会极大的占用网络 IO，当网络被占满时，其余的操作也会受到影响；尽量使用 Redis 提供的操作来操作集合运算，如 `SISMEMBER` 等
-   危险操作，如 `KEYS  *` 等，不要在现网中使用

##  0x04  Redis 的应用场景
本小节梳理下，项目中使用 Redis 的场景。

####  消息队列
主要是利用 Redis 的 List 数据结构，实现 Broker 机制，又细分为三种模式：
1、List<br>

- 使用生产者：`LPUSH`；消费者：`RBPOP`或`RPOP`模拟队列

2、Stream<br>
3、PUB/SUB（订阅）<br>

####  延迟队列
1、实现一思路 <br>
- 使用 `ZSET`+ 定时轮询的方式实现延时队列机制，任务集合记为 `taskGroupKey`
- 生成任务以 **当前时间戳** 与 **延时时间** 相加后得到任务真正的触发时间，记为 `time1`，任务的 uuid 即为 `taskid`，当前时间戳记为 `curTime`
- 使用 `ZADD taskGroupKey time1 taskid` 将任务写入 ZSET
- 主逻辑不断以轮询方式 `ZRANGE taskGroupKey curTime MAXTIME withscores` 获取 `[curTime,MAXTIME)` 之间的任务，记为已经到期的延时任务（集）
- 处理延时任务，处理完成后删除即可
- 保存当前时间戳`curTime`，作为下一次轮询时的`ZRANGE`指令的范围起点


####  滑动窗口
1、通过Redis构建滑动窗口并实现计数器限流<br>




##	0x05	参考
-   [Using pipelining to speedup Redis queries](https://redis.io/topics/pipelining)
-   [聊聊 GO-REDIS 的一些高级用法](http://vearne.cc/archives/1113)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权