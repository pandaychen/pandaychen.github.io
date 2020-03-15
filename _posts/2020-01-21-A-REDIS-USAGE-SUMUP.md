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
&emsp;&emsp; 在大批量的操作 Redis（写入 / 读取）时，采用 pipeline 是个非常好的优化思路，可以提高在单个 Redis 连接操作的数据吞吐量，从而提升整体性能。
通过 pipeline 方式将 client 端命令（打包后）一起发出，Redis 服务端会处理完多条命令后，将结果一起打包返回 client，从而节省大量的网络延迟开销。这里需要注意两点：
1.  服务端的模式（主从 Sentinel、集群 Cluster、Proxy）是否支持 pipeline 操作？
2.  项目中使用的 Redis 客户端库，是否支持 pipeline 操作？
3.  一次 pipeline.Exec() 操作打包多少条 Redis 命令？

这里有趣的问题是第 3 个，从下面这个问题中，大概可以找到答案：
[How many commands could redis-py pipeline have?](https://stackoverflow.com/questions/27007530/how-many-commands-could-redis-py-pipeline-have)

>Not sure that there is a maximum, but I don't think that you want to reach it in case there is.
>In most cases, limiting the size of the pipeline to 100-1000 operations gives the best results. But, you can do a little benchmark >research that includes typical requests that you send. Pipelining requests is generally good, but keep in mind that the responses >are kept in Redis memory until all pipeline requests are served, and your client waits for the long reply of all requests.
>You should try to find the sweet spot of concurrent connections, pipelined requests and your Redis memory.


##  0x02    Pool 连接池使用
&emsp;&emsp; 连接池的好处，在于避免每次Redis操作时，新建TCP连接的开销，在要求高性能的场景推荐开启。个人建议，使用的Redis客户端库，包含的连接池至少满足下面的特性：
1.  连接健康检查
2.  自动关闭空连接

下面这张表格总结了常见的Golang的Redis客户端库的连接池特性，笔者在项目中使用的连接池是基于[go-redis](https://github.com/go-redis/redis/tree/master/internal/pool)的。

| 连接池功能 | go-redis | radix.V2 | redigo|
| :-----:| :----: | :----: | :----: |
| 实现数据结构 | slice | channel | linklist|
| 连接健康检查 | No | Yes | Yes|
| 自动关闭空闲连接| Yes| No| Yes|
| 连接的Max生存时间| Yes| No| Yes|


##  0x03    Set 集合操作
Set 作为 CS 中最基础的数据结构，使用非常广泛，在 Redis 中 Set，在高性能查找中扮演着极为重要的角色。举例来说，SInter(Store) 和 SUnion(Store) 及 SDiff(STORE) 这几个操作，取（多个）集合的交集、并集及差集并直接存储。在实际项目中，如计算多个子集合间的交集、并集和差集，非常方便。

-   `sinter key [key ...]` 返回一个集合的全部成员，该集合是所有给定集合的交集
-   `sunion key [key ...]` 返回一个集合的全部成员，该集合是所有给定集合的并集
-   `sdiff key [key ...]`  返回所有给定 key 与第一个 key 的差集
-   `sinterstore destination key [key ...]` 将 交集 数据存储到某个对象中
-   `sunionstore destination key [key ...]` 将 并集 数据存储到某个对象中
-   `sdiffstore destination key [key  ...]` 将 差集 数据存储到某个对象中

举例看下：

1、计算 Set5 和 Set1~4 之间并集的交集
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

2、复制一个 Set
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

##  Hash 使用

##	0x04	参考
-   [聊聊 GO-REDIS 的一些高级用法](http://vearne.cc/archives/1113)