---
layout:     post
title:      微服务基础之多级服务缓存（Cache）
subtitle:
date:       2020-04-10
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 缓存
---


##	0x00 前言

在微服务架构中，后台组件开启热数据缓存，可以有效的降低 Mysql 等后端存储的读负载。在笔者项目后台服务构建中，一般会有本地缓存（Local-cache）---> Redis（高可用缓存服务）--->Mysql，这种层级的缓存逻辑。


##	0x01	缓存更新模式

不管如何更新缓存，这里有一个前提是：缓存必须有过期时间（TTL），就像在 Redis 中添加的 key-value 对，一定要加 expire key 操作，这是防止脏缓存的保底策略。

缓存机制是牺牲强制一致性换来高性能的一种手段之一。一般而言，缓存更新有以下几种模式（在项目中用的最多的是 Cache Aside 模式）：

-	Cache Aside 模式
Client 读取 Cache 数据：未命中 cache，Client 读数据库，读取成功后 Client 将数据更新到 cache
Client 读取 Cache 数据：读取到了返回。
Client 更新 Cache 数据：先更新数据库，更新数据库成功后让 Cache 失效（删除 Cache 后再更新 Cache）

-	Read/Write Through 模式
Read Through ：Client 读取 Cache 服务，未命中，Cache 服务读取数据库，并且自己更新数据，对 Client 保持透明
Write Through ：Client 更新数据的时候，命中 Cache 则更新 Cache，Cache 服务自身去更新数据库。未命中 Cache，直接去更新数据库

-	Write Behind Caching 模式
更新数据时，只更新 Cache。Cache 服务异步批量去更新数据库，比如按照时间周期，定期更新一次，等等


##	0x02	缓存架构构建


##	0x03	缓存穿透问题


##	0x04 缓存和 Mysql 的配合使用


##	0x05 合理的 Ratelimit 及 Breaker 策略及异步化
限流和熔断机制，在访问缓存系统及 Mysql 也是需要具备的，防止高并发下访问 Redis 或 Mysql 集群引发的失败及雪崩问题。在现网中，遇到过针对 inndoDB 的数据库表（约 40W 行数据），并发 2k + 个请求 update 数据库，由于默认的超时时间（innodb_lock_wait_timeout = 50s）较短，从而出现大量的错误：`ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction`。通常这种错误有这几种处理方法：
1.	增加默认的 innodb_lock_wait_timeout 时间
2.	异步化，采用 MQ（比如 Kafka 等）来异步更新 Redis 或者 Mysql（要考虑 Kafka 的消费延迟及 DB 操作延迟）
3.	服务接口加入限流和熔断机制（较为暴力）

##	0x	参考
-	[缓存更新的套路](https://coolshell.cn/articles/17416.html)
-	[分布式之数据库和缓存双写一致性方案解析](https://www.cnblogs.com/rjzheng/p/9041659.html?hmsr=joyk.com&utm_source=joyk.com&utm_medium=referral)