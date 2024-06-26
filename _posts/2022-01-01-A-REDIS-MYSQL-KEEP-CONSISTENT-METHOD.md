---
layout:     post
title:      微服务中的缓存（三）：缓存一致性原理 && 实践
subtitle:   一道美团的面试题：Redis 与 MySQL 双写一致性如何保证？
date:       2022-01-01
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 缓存
    - Redis
    - MySQL
    - Cache
    - Go-Zero
---


##  0x00    前言
前两篇文章介绍了微服务中的缓存常用概念及一致性的问题解决，本文以 go-zero 项目为例，看下其针对于多级缓存的实现分析。
-   [微服务中的缓存（一）：Cache 使用与优化](https://pandaychen.github.io/2020/02/22/A-CACHE-STUDY/)
-   [微服务中的缓存（二）：微服务基础之多级服务缓存（Cache）](https://pandaychen.github.io/2020/04/10/MICROSVC-CACHE-INTRO/)

##  0x01    问题
本文汇总下微服务的缓存使用原理，再结合go-zero框架分析下如何优雅的针对一致性问题的实践。

##  0x02    再回首：经典的 Cache 模式（读写策略）
本小节以标准的 Cache&&DB 回顾下经典读写策略，如有用户表，属性如下：
-   `userId`：用户 id
-   `phone`：用户电话
-  `avtoar`：用户头像 url

在 Cache 中使用 `phone` 作为 key，存储 `avtoar`。问题如下，当用户修改 `avtoar` 时该如何操作 DB 以及 Cache？

####    Cache Aside 模式

关于 DB 和 Cache 的操作，**优先更新 DB、其次操作 Cache 的方式**，又分为两种：
-   先更新 DB 数据，再更新 Cache 数据（不推荐）
-   **先更新 DB 数据，再删除 Cache 数据（推荐）**

为什么推荐后一种方式呢？首先变更 DB 数据库和变更 Cache 缓存是两个独立操作，而这里并没有对操作做任何的并发控制。那么当两（多）个线程并发更新操作的时候，就会因为写入顺序的不同造成数据不一致。所以更推荐后一种方式（Cache Aside）：

-   更新数据时不更新 Cache，而是直接删除缓存 Cache
-   而后续的请求发现 Cache Miss，再触发查询 DB 数据库，此时再将结果 load 到缓存 Cache

Cache Aside 策略数据以数据库中的数据为准，缓存中的数据是按需加载的，分为读策略和写策略两种。但是可见的问题也就出现了：频繁的读写操作会导致 Cache 反复地替换，导致缓存命中率降低。当然如果在业务中对命中率有监控报警时，考虑以下优化策略：

1.  更新数据时同时更新缓存 Cache，但是在更新缓存前加个分布式锁（后抢到锁的后更新，先抢到的先更新Cache），如此解决并发修改 Cache 的问题。同时在后续读请求中时读到最新的缓存，解决了不一致的问题
2.  更新数据时同时更新缓存 Cache，但是给 Cache 缓存一个较短的 TTL，让其快速的过期

Cache Aside 模式的流程如下图：
![cache-aside](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/cache/go-zero/CacheAside.png)

####    Write Through 模式
Write Through 核心原则是用户只与缓存Cache打交道，由缓存组件和DB通信，写入或者读取数据。先查询写入数据key是否命中缓存Cache，如果命中则更新缓存Cache，同时缓存组件同步数据至DB；不存在，则触发 Write Miss。适用场景是本地进程缓存组件

Write Through 模式的流程如下：

![Write-Through](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/cache/go-zero/cache-WriteThrough.png)

而针对上述 Write Miss 有两种方式：
1.  Write Allocate机制：写时直接分配 Cache line
2.  No-write allocate机制：写时不写入缓存，直接写入DB后Return

在 Write Through 模式中，一般采取 No-write allocate机制。因为无论哪种，最终数据都会持久化到DB中，省去一步缓存的写入，提升写性能。而缓存由 Read Through 写入缓存

Write Through 模式的缺点是：由于写数据时缓存和数据库DB同步，写入性能可能会受影响（存储介质的速度差几个数量级）

####    Write Back 模式
Write back 模式就是在写数据时只更新该 Cache Line 对应的数据，并把该行标记为 Dirty。在读数据时或是在缓存满时换出（缓存替换策略）时，将 Dirty 写入存储。需要注意的是：在 Write Miss 情况下，采取的是 Write Allocate，即写入存储同时写入缓存，这样在之后的写请求只需要更新缓存

![write-back](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/cache/go-zero/cache-WriteBack.png)

上述三者中，使用最广泛的还是Cache Aside模式。

##  0x03    微服务中的Redis：旁路缓存
在应用中，需要增加对Redis的操作代码，包含：
1.  读取缓存Cache
2.  读取数据库Db
3.  更新缓存

####    Redis缓存的问题与应对
-   缓存的容量限制与淘汰：通过监控解决
-   上游并发请求对Redis冲击，防止缓存击穿：[singleflight机制](https://pandaychen.github.io/2020/02/22/A-CACHE-STUDY/#singleflight-%E6%9C%BA%E5%88%B6)、go-zero的[ShardCalls机制](https://talkgo.org/t/topic/1505)（类似）
-   Redis缓存与后端存储的数据一致性问题

关于一致性实践，项目中遵循这几点：
1.  一切数据应该以DB数据库为准，对缓存的更新应该滞后于DB数据库更新
2.  缓存写入应该交给读请求来完成
3.  写DB数据库请求尽可能保证数据一致性
4.  Redis缓存必须要有TTL机制，TTL的时长可视业务场景设置长或者短，防止误删除/更新导致的Redis脏缓存
5.  对缓存的命中率做统计（Cache Miss过高就需要重新设计Cache机制了）


##  0x04    Cache模式的现网应用
针对笔者项目中，通常按照下面的逻辑来使用 Cache：
-   针对不会更新或少更新的数据，通常采用独立的旁路逻辑，定期把 Mysql-DB 中的数据全量缓存到 Redis，读取的时候优先走 Redis 读，CacheMiss 之后会触发一次 DB 读取


##  0x05    缓存的一致性：到底使用何种更新顺序？
为了方便理解，这里缓存代指 Redis，数据库代指 Mysql

1、场景 1：先删缓存再更新数据库 <br>


2、场景 2：先更新数据库再删除缓存 <br>


3、场景 3：并发读写 <br>


4、场景 4：重试机制 <br>


##  0x05  go-Zero 的缓存层设计分析
本小节摘自原文 [微服务缓存原理与最佳实践](https://talkgo.org/t/topic/1505)，接口的设计原则是调用者只需要负责业务逻辑，缓存写入和删除对调用方透明

代码[见此](https://github.com/zeromicro/go-zero/blob/master/core/stores/sqlc/cachedsql.go#L75)，封装基于两点：
-   将业务逻辑和缓存操作分离，给外部提供写入逻辑的hook
-   缓存操作支持流量冲击，缓存策略的优化

####    Exec实现：写过程
写请求，使用的就是之前缓存策略中的 Cache Aside模式，即先写数据库，再删除缓存
```GO
_, err := m.Exec(func(conn sqlx.SqlConn) (result sql.Result, err error) {
  execSQL := fmt.Sprintf("update your_table set %s where 1=1", m.table, AuthRows)
  return conn.Exec(execSQL, data.RangeId, data.AuthContentId)
}, keys...)

func (cc CachedConn) Exec(exec ExecFn, keys ...string) (sql.Result, error) {
 res, err := exec(cc.db)
 if err != nil {
  return nil, err
 }

 if err := cc.DelCache(keys...); err != nil {
  return nil, err
 }

 return res, nil
}
```


####    QueryRow实现：读过程
`QueryRow`对应缓存策略中的：Read Through，即查询业务逻辑用 `func(conn sqlx.SqlConn, v interface{})` 进行封装。用户无需考虑缓存写入，只需要传入需要写入的 `cacheKey`即可，同时把查询结果 `res` 返回

调用方代码：
```golang
// res: query result
// cacheKey: redis key
err := m.QueryRow(&res, cacheKey, func(conn sqlx.SqlConn, v interface{}) error {
  querySQL := `select * from your_table where campus_id = ? and student_id = ?`
  return conn.QueryRow(v, querySQL, campusId, studentId)
})
```

```GOLANG
func (c cacheNode) QueryRow(v interface{}, key string, query func(conn sqlx.SqlConn, v interface{}) error) error {
 cacheVal := func(v interface{}) error {
  return c.SetCache(key, v)
 }
 // 1. cache hit -> return
  // 2. cache miss -> err
 if err := c.doGetCache(key, v); err != nil {
    // 2.1 err defalut val {*}
  if err == errPlaceholder {
   return c.errNotFound
  } else if err != c.errNotFound {
   return err
  }
  // 2.2 cache miss -> query db
    // 2.2.1 query db return err {NotFound} -> return err defalut val「see 2.1」
  if err = query(c.db, v); err == c.errNotFound {
   if err = c.setCacheWithNotFound(key); err != nil {
    logx.Error(err)
   }

   return c.errNotFound
  } else if err != nil {
   c.stat.IncrementDbFails()
   return err
  }
  // 2.3 query db success -> set val to cache
  if err = cacheVal(v); err != nil {
   logx.Error(err)
   return err
  }
 }
 // 1.1 cache hit -> IncrementHit
 c.stat.IncrementHit()

 return nil
}
```

`queryRow`的实现思路如下图：
![queryRow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/cache/go-zero/ReadThrough-app.png)


##  0x06   go-zero 的缓存机制

####    DB 层缓存机制（持久层）
go-zero 中的机制如下：
-   对缓存是只做删除操作，不做更新，一旦 DB 里数据被修改，会直接删除对应的缓存，而不更新
-   更新的操作仅仅发生在读取时，按照 Primary-Key 更新缓存

####    业务层缓存机制（业务视角）



##  0x07    go-zero 的 DB 缓存机制：Mysql 的行缓存
[GoZero 文档](https://go-zero.dev/cn/docs/blog/cache/cache) 给出了其 `QueryRowIndex`，基于 Mysql 行缓存的实现方式，** 核心思路是把 MYSQL 的主键 / uniqueKey 映射到 Redis 的 key-value 结构 **，具体的步骤如下：

1.  若没有查询条件到 Primary-Key 映射的缓存
    -   通过查询条件到 Mysql-DB 去查询行记录，然后做 `2` 件事情
        -   a）把 Primary-Key 到行记录的缓存写到 Redis 里
        -   b）把查询条件到 Primary-Key 的映射保存到 Redis 里，由框架的 `Take` 方法自动完成
    -   可能的过期顺序，又分为两种情况：
        -   a）查询条件到 Primary-Key 的映射缓存未过期
            -   Primary-Key 到行记录的缓存未过期
                -   直接返回缓存行记录
            -   Primary-Key 到行记录的缓存已过期
                -   通过 Primary-Key 到 Mysql-DB 获取行记录，并写入缓存
                    -   此时存在的问题是，查询条件到 Primary-Key 的缓存可能已经快要过期了，短时间内的查询又会触发一次数据库查询
                    -   要避免这个问题，可以让后面 cache 的 ttl 比前面 cache 的 ttl 略长一些（如 `5s`）
        -   b）查询条件到 Primary-Key 的映射缓存已过期，不管 Primary-Key 到行记录的缓存是否过期
            -   查询条件到 Primary-Key 的映射会被重新获取，获取过程中会自动写入新的 Primary 到行记录的缓存，这样 `2` 种缓存的过期时间都是刚刚设置

2.  有查询条件到 Primary-Key 映射的缓存
    -   没有 Primary-Key 到行记录的缓存
        -   a）通过 Primary-Key 到 Mysql-DB 查询行记录，并写入缓存
    -   有 Primary-Key 到行记录的缓存
        -   a）直接返回缓存结果


##  0x08    总结
一般推荐使用**更新数据库 + 删除缓存**的方案。如果根据需要，如热点数据较多，可以使用**更新数据库 + 更新缓存**策略。 在**更新数据库 + 删除缓存**的方案中，推荐使用推荐用**先更新数据库，再删除缓存**策略，因为先删除缓存可能会导致大量请求落到数据库，而且延迟双删的时间很难评估

##  0x09 参考
-   [go-zero 微服务实战系列（六、缓存的一致性如何保证）](https://learnku.com/articles/69446)
-   [美团二面：Redis 与 MySQL 双写一致性如何保证？](https://juejin.cn/post/6964531365643550751)
-   [go-zero 缓存设计之持久层缓存](https://go-zero.dev/cn/docs/blog/cache/redis-cache/)
-   [go-zero 微服务实战系列（五、缓存代码怎么写）](https://segmentfault.com/a/1190000042018127)
-   [go-zero 微服务实战系列（六、缓存的一致性如何保证）](https://learnku.com/articles/69446)
-   [【原创】分布式之数据库和缓存双写一致性方案解析](https://www.cnblogs.com/rjzheng/p/9041659.html)
-   [DB 缓存机制](https://go-zero.dev/cn/docs/blog/cache/cache)
-   [微服务缓存原理与最佳实践](https://talkgo.org/t/topic/1505/4)