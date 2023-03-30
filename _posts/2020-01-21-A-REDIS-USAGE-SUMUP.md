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
    - 分布式
    - 限流
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


####  Pipeline 的坑
注意当`pipeClient.Exec`方法返回err时，还需要处理返回值（当`err==redis.Nil`时），看下面批量`HGet`，其他批量操作也是同理，测试代码见[pipeline.go](https://github.com/pandaychen/golang_in_action/blob/master/redis/go-redis/pipeline.go)

```GO
// PipelineGetHashField 使用pipeline命令获取多个hash key的单个字段
// keyList，需要获取的hash key列表
// field 需要获取的字段值
func PipelineGetHashField(keyList []string,filed string) []string {
    pipeClient :=client.Pipeline()
    for _, key := range keyList {
        pipeClient.HGet(key, filed)
    }
    res, err := pipeClient.Exec()
    if err != nil {
        if err != redis.Nil {
            logrus.WithField("key_list", keyList).Errorf("get from redis error:%s", err.Error())
        }
        // 注意这里如果某一次获取时出错（常见的redis.Nil），返回的err即不为空
        // 如果需要处理redis.Nil为默认值，此处不能直接return
    }
    valList := make([]string, 0, len(keyList))
    for index, cmdRes := range res {
        var val string
        // 此处断言类型为在for循环内执行的命令返回的类型,上面HGet返回的即为*redis.StringCmd类型
        // 处理方式和直接调用同样处理即可
        cmd, ok := cmdRes.(*redis.StringCmd)
        if ok {
            val,err = cmd.Result()
            if err != nil {
                logrus.WithField("key",keyList[index]).Errorf("get key error:%s",err.Error())
            }
        }
        valList = append(valList, val)
    }
    return valList
}
```


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


####  连接池的大小的合理性
在 go-redis 中，在高并发的场景下，如果连接池的 size 设置过小，可能会导致 `connection pool timeout` 的 [报错](https://github.com/go-redis/redis/issues/195)。一般通过如下策略优化：
- 优化耗时过久的操作
- 调大连接池的大小
- 合理设置 `PoolTimeout`、`IdleTimeout`、`ReadTimeout`、`MinIdleConns` 以及 `WriteTimeout` 等 [参数](https://github.com/go-redis/redis/blob/master/options.go#L31) 的值

如：
```golang
 Cluster = redis.NewClient(&redis.ClusterOptions{
        Addrs:        addresses,
        PoolSize:     512,
        PoolTimeout:  2 * time.Minute,
        IdleTimeout:  10 * time.Minute,
        ReadTimeout:  2 * time.Minute,
        WriteTimeout: 1 * time.Minute,
    })
```

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

// 异步监控
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
ZSET 即有序集合，通常用来实现延时队列或者排行榜（如销量 / 积分排行、成绩排行等等）。ZSET 通常包含 `3` 个 关键字操作：
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
7.  `ZREMRANGEBYSCORE key startScore endScore` 会删除 `[startScore,endScore]` 区间的数据（包含左右区间）

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
127.0.0.1:6379> ZREMRANGEBYSCORE key1 1 3
(integer) 3
```

####  【WARNING】慎用的操作
在项目实践中，也遇到了一些性能问题，如下：
-   慢操作：`SMEMBERS` 命令是从一个 SET 结构获取集合，但这个集合数据量已经很大，当这个命令被频繁的调用，会极大的占用网络 IO，当网络被占满时，其余的操作也会受到影响；尽量使用 Redis 提供的操作来操作集合运算，如 `SISMEMBER` 等
-   危险操作，如 `KEYS  *` 等，不要在现网中使用

##  0x04  Redis 的应用场景
本小节梳理下，项目中使用 Redis 的场景。

####  消息队列
主要是利用 Redis 的 List 数据结构，实现 Broker 机制，又细分为三种模式：
1、List：使用生产者：`LPUSH`；消费者：`RBPOP` 或 `RPOP` 模拟队列<br>
2、Stream<br>
3、PUB/SUB（订阅）<br>

基于 go-redis 的 pub/sub[测试代码](https://github.com/pandaychen/golang_in_action/tree/master/redis/go-redis/subpub) 在此。基于 Redis 来实现消息的发布与订阅，优点是比较轻量级（适合数据量不大且容忍数据丢失的场景），缺点是消息不能堆积，一旦消费者节点没有消费消息，消息将会丢失。订阅的模型如下：

![subpub](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/redis/subpub-1.png)

在项目中，一个可实践的场景是借助 Redis 的订阅 / 发布机制和 WebSocket 长连接结合，实现轻量级的订阅发布和消息实时推送功能。

####  延迟队列
1、实现一思路 <br>
- 使用 `ZSET`+ 定时轮询的方式实现延时队列机制，任务集合记为 `taskGroupKey`
- 生成任务以 **当前时间戳** 与 **延时时间** 相加后得到任务真正的触发时间，记为 `time1`，任务的 uuid 即为 `taskid`，当前时间戳记为 `curTime`
- 使用 `ZADD taskGroupKey time1 taskid` 将任务写入 ZSET
- 主逻辑不断以轮询方式 `ZRANGE taskGroupKey curTime MAXTIME withscores` 获取 `[curTime,MAXTIME)` 之间的任务，记为已经到期的延时任务（集）
- 处理延时任务，处理完成后删除即可
- 保存当前时间戳 `curTime`，作为下一次轮询时的 `ZRANGE` 指令的范围起点


####  滑动窗口

######  场景 1：通过 Redis 构建滑动窗口并实现计数器限流

比如，系统要限定用户的某个行为在指定的时间里只能允许发生 `N` 次，采用滑动窗口的方式实现。利用 `ZSET` 可实现滑动窗口的机制。如下图所示，使用 ZSET 记录所有用户的访问历史数据，每个 `key` 表示不同的用户，用 `score` 标记时间范围的刻度，`value` 这里无意义，仅用于唯一性考虑。该流程大致流程如下：
![redis-rolling](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/redis/redis-rolling-window1.png)

1.  当用户 A 访问时，向 ZSET 中添加 `ZADD usernameA ${curTime} ${uuid}`，其中 `score` 值 `$curtime` 为当前时间戳，`value` 值 `${uuid}` 为保证唯一性的字符串（或者改用毫秒时间戳减少 `size`）
2.  针对用户 `usernameA`，限流只需要计算指定的时间区间的总数 `ZCOUNT usernameA ${startTime} ${endTime}`，将此值与限流值 `limiter` 比较即可；通过 `score` 值圈出时间窗口，实现了滑动窗口的效果
3.  对于 `score` 窗口之外的数据，会有占用内存过大的风险，有两个方法优化：
  - 异步清理 `score` 过期的数据（滑动窗口外的数据）
  - 若某个时间段检查用户并并未触发限流（滑动窗口内的行为是空记录），那么可以直接删掉 ZSET 的此 `key` 以减少内存占用
4.  该方案的 Redis 操作都是针对同一个 `key` 的，使用 `pipeline` 可以显著提升存取效率
5.  该方案的缺点是要记录时间窗口内所有的行为记录，如果这个并发量很大，会消耗大量的内存

通常分布式场景使用 lua 脚本实现，如下：
```lua
--KEYS[1]: 限流对应的 key
--ARGV[1]: 一分钟之前的时间戳（假设按照 1 分钟为滑动窗口的区间）
--ARGV[2]: 此时此刻的时间戳（score 值）
--ARGV[3]: 允许通过的最大数量
--ARGV[4]:member 名称（随机生成）
redis.call('zremrangeByScore', KEYS[1], 0, ARGV[1])
local res = redis.call('zcard', KEYS[1])
if (res == nil) or (res < tonumber(ARGV[3])) then
    redis.call('zadd', KEYS[1], ARGV[2], ARGV[4])
    return 0
else return 1 end
```

###### 场景 2：通过 `EXPIRE` 实现滑动窗口计数器限流
和上面使用 `ZSET` 不同，这里用 `EXPIRE` 来模拟窗口（注意：仅能使用秒为单位），从某个时间点开始每次请求过来请求数 `+=1`，同时判断当前时间窗口内请求数是否超过限制，超过限制则拒绝该请求，然后下个时间窗口开始时计数器清零等待请求，lua 代码如下：

```lua
-- KYES[1]: 限流器 key
-- ARGV[1]:qos, 单位时间内最多请求次数
-- ARGV[2]: 单位限流窗口时间
-- 请求最大次数, 等于 p.quota
local limit = tonumber(ARGV[1])
-- 窗口即一个单位限流周期, 这里用过期模拟窗口效果, 等于 p.permit
local window = tonumber(ARGV[2])
-- 请求次数 + 1, 获取请求总数
local current = redis.call("INCRBY",KYES[1],1)
-- 如果是第一次请求, 则设置过期时间并返回 成功
if current == 1 then
  redis.call("EXPIRE",KYES[1],window)
  return 1
-- 如果当前请求数量小于 limit 则返回 成功
elseif current < limit then
  return 1
-- 如果当前请求数量 ==limit 则返回 最后一次请求
elseif current == limit then
  return 2
-- 请求数量 > limit 则返回 失败
else
  return 0
end
```


##	0x05	参考
-   [Using pipelining to speedup Redis queries](https://redis.io/topics/pipelining)
-   [聊聊 GO-REDIS 的一些高级用法](http://vearne.cc/archives/1113)
- [Redis 设计与实现：订阅与发布](https://redisbook.readthedocs.io/en/latest/feature/pubsub.html)
- [把 Redis 当作队列来用，真的合适吗？](https://www.51cto.com/article/659208.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权