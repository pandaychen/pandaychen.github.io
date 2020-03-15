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
Redis 是 Linux C 项目中单线程开发模型的典范，其内置了多种高性能的数据结构，如 sds 动态字符串、skiplist 调表，zset，hashset 等。今天这篇文章是想分享下项目中使用 Redis 的一些技巧，结合 [go-redis](https://github.com/go-redis/redis) 这个客户端库。

##  0x01    Pipeline 加速


##  0x02    Pool 连接池使用


##  0x03    Set：集合操作
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

##	0x04	参考
-   [聊聊GO-REDIS的一些高级用法](http://vearne.cc/archives/1113)