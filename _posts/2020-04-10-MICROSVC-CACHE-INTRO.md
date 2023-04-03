---
layout:     post
title:      微服务中的缓存（二）：微服务基础之多级服务缓存（Cache）
subtitle:   微服务应用中缓存的使用策略、问题及解决
date:       2020-04-10
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 缓存
    - Cache
---


##	0x00 前言

&emsp;&emsp;在微服务架构中，后台组件开启热数据缓存，可以有效的降低 Mysql 等后端存储的读负载。在笔者项目后台服务构建中，一般会有**本地缓存（Local-cache）---> Redis（高可用缓存服务）--->Mysql（高可用存储集群）**，这种层级的缓存逻辑。

![image](https://s1.ax1x.com/2020/04/17/JeS9bt.png)

##	0x01	缓存更新模式

不管如何更新缓存，这里有一个前提是：**缓存必须有过期时间（TTL）**，就像在 Redis 中添加的 key-value 对，一定要加 expire key 操作，这是防止脏缓存的保底策略。

缓存机制是牺牲强制一致性换来高性能的一种手段之一。一般而言，缓存更新有以下几种模式（在项目中用的最多的是 Cache Aside 模式）：

1.	Cache Aside 模式
    -   Client 读取 Cache 数据：未命中 Cache，Client 读数据库，读取成功后 Client 将数据更新到 Cache
    -   Client 读取 Cache 数据：读取到了返回。
    -   Client 更新 Cache 数据：先更新数据库，更新数据库成功后让 Cache 失效（删除 Cache 后再更新 Cache）

2.	Read/Write Through 模式
    -   Read Through ：Client 读取 Cache 服务，未命中，Cache 服务读取数据库，并且自己更新数据，对 Client 保持透明
    -   Write Through ：Client 更新数据的时候，命中 Cache 则更新 Cache，Cache 服务自身去更新数据库。未命中 Cache，直接去更新数据库

3.	Write Behind Caching 模式
    -   更新数据时，只更新 Cache。Cache 服务异步批量去更新数据库，比如按照时间周期，定期更新一次，等等


##	0x02	缓存架构构建


##	0x03	缓存的问题
缓存的问题，通常有缓存穿透，缓存击穿和缓存雪崩这几种：

-   缓存穿透（cache penetration）：表示程序要访问的缓存key不在缓存key的取值范围里
-   缓存击穿（hotspot invalid）：表示缓存数据失效了。和cache penetration的区别是缓存的key还是存在的，只是这个key的数据失效了
-   缓存雪崩（cache avalanche）：表示大量缓存都失效了，像雪崩一样

##	0x04 	缓存和 Mysql 的配合


##	0x05 合理的 Ratelimit 及 Breaker 策略及异步化
&emsp;&emsp;限流和熔断机制，在访问缓存系统及 Mysql 也是需要具备的，防止高并发下访问 Redis 或 Mysql 集群引发的失败及雪崩问题。在现网中，遇到过针对 inndoDB 的数据库表（约 40W 行数据），并发 2k + 个请求 update 数据库，由于默认的超时时间（`innodb_lock_wait_timeout = 50s`）较短，从而出现大量的错误：`ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction`。通常，优化此种场景，有这么几种方法：
1.	增加默认的 innodb_lock_wait_timeout 时间
2.	异步化，采用 MQ（比如 Kafka 等）来异步更新 Redis 或者 Mysql（要考虑 Kafka 的消费延迟及 DB 操作延迟）
3.	在操作 Mysql 或 Redis 的接口加入限流和熔断机制

##	0x06	参考
-	[缓存更新的套路](https://coolshell.cn/articles/17416.html)
-	[分布式之数据库和缓存双写一致性方案解析](https://www.cnblogs.com/rjzheng/p/9041659.html?hmsr=joyk.com&utm_source=joyk.com&utm_medium=referral)
-   [微服务缓存原理与最佳实践](https://talkgo.org/t/topic/1505)
-   [go-zero缓存设计之业务层缓存](https://www.bookstack.cn/read/go-zero-1.3-zh/buiness-cache.md)
-   [go-zero缓存设计之持久层缓存](https://www.bookstack.cn/read/go-zero-1.3-zh/redis-cache.md)
-   [DB缓存机制](https://www.bookstack.cn/read/go-zero-1.3-zh/sql-cache.md)
-   [防止缓存击穿之进程内共享调用](https://www.bookstack.cn/read/go-zero-1.3-zh/sharedcalls.md)
-   [主从DB与cache一致性](https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=404308725&idx=1&sn=1a25ce76dd1956014ceb8a011855268e&scene=21#wechat_redirect)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权