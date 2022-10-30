---
layout:     post
title:      分布式一致性 hash 算法的实现分析
subtitle:   ketama 算法分析与应用
date:       2020-08-18
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 分布式
    - consistent-hash
---

##  0x00    前言

在工作中，遇到过如下几种需要分布式一致的场景：

1.  分布式一致的缓存系统
2.  gRPC的基于一致性算法的loadbalance实现，某些业务场景下，需要尽可能使指定客户端的请求经由consistent-hash算法到达指定的服务器进行处理

####    consistent-hash 的应用场景
consistent-hash 算法主要的应用场景是：当服务是一个有状态服务等时候，需要根据特定的 key 路由到相同的目标服务机器进行处理的场景。

比如，考虑下面这种有状态的缓存服务，对 N（海量）个员工信息进行缓存，在访问时，需要对其做增删查。同时，数据量规模较大，需要存储在多台机器上（将 hash 缓存分发到多个服务器），要求是 key 的存储尽可能均衡，有几种方式：

```text
john@example.com => john
mark@example.com => mark
tesla@example.com => tesla
adam@examle.com => adam
......
```

1. 普通 hash：按服务器数量，对 key 进行 mod。问题是当某台服务器故障下线了（或者扩容新增服务器），都需要 ReHash，**此方法会导致所有系统中缓存的 key 都要重新计算一次**
    -   对后端服务器来说，将大大增加服务器的负载（因为会丢失缓存后需要重新查询建立映射）
    -   对客户端来说，由于映射方式的改变大量 mail 都被映射到新的服务器上，导致映射在这一刻不可用，并且容易导致缓存穿透问题

所以一致性 hash 非常适合解决此类场景，服务器的伸缩不会导致系统中所有缓存的 Key 都重新计算，仅仅一小部分 Key 需要重新计算

##  0x01    consistent-hash的原理

####    基础方式
1、把节点（用户存储kv的服务器节点）通过hash后，映射到一个范围是`[0，2^32]`的环上，比如，采用`ip:port`来命名一个节点，下图有N0~N2共`3`个物理节点：

![ch-1]()

2、把要缓存的kv数据也通过hash的方式映射到`[0，2^32]`环上，然后按顺时针方向查找第一个hash值大于等于数据的hash值的节点，该节点即为数据所分配到的节点

![ch-2]()

3、可优化点：当删除某个服务器时，可能存在该服务器前和后面的直接的key数据没有均衡分布，所以通常引入虚拟节点来优化均衡性问题。因为节点越多，它们在环上的分布就越均匀，使用虚拟节点还可以降低节点之间的负载差异，理论的情况，从`n`个服务器扩容到`n+1`时，只需要重新映射 $\frac{1}{n+1}$ 的key，如下图：

![ch-3]()

此时，查找的逻辑变更为如下：

![ch-4]()

##  0x02    consistent-hash的一种实现：ketama算法



##  0x0 参考
-   [一致性 hash 算法 - consistent hashing](https://blog.csdn.net/sparkliang/article/details/5279393)
-   [一致性哈希 （Consistent Hashing）的前世今生](https://candicexiao.com/consistenthashing/)