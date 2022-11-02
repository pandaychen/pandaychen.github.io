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

所以一致性 hash 非常适合解决此类场景，服务器的伸缩不会导致系统中所有缓存的 Key 都重新计算，仅仅一小部分 Key 需要重新计算。hashring的结构实现了在槽位数量变化前后的增量式的重新映射，避免了全量的重新映射。

##  0x01    consistent-hash的原理

####    基础方式
1、把节点（用户存储kv的服务器节点）通过hash后，映射到一个范围是`[0，2^32]`的环上，比如，采用`ip:port`来命名一个节点，下图有N0~N2共`3`个物理节点：

![ch-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/consistenthash/consistent-hash-1.png)

2、把要缓存的kv数据也通过hash的方式映射到`[0，2^32]`环上，然后按顺时针方向查找第一个hash值大于等于数据的hash值的节点，该节点即为数据所分配到的节点

![ch-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/consistenthash/consistent-hash-2.png	)

3、可优化点：当删除某个服务器时，可能存在该服务器前和后面的直接的key数据没有均衡分布，所以通常引入虚拟节点来优化均衡性问题。因为节点越多，它们在环上的分布就越均匀，使用虚拟节点还可以降低节点之间的负载差异，理论的情况，从`n`个服务器扩容到`n+1`时，只需要重新映射 $\frac{k}{n+1}$ 的key（`k`为hashring的key总数），如下图：

![ch-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/consistenthash/consistent-hash-3.png)

此时，查找的逻辑变更为如下：

![ch-4](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/consistenthash/consistent-hash-5.png)

##  0x02    consistent-hash的一种实现：ketama算法
本小节分析下ketama的一种实现[代码](https://github.com/pandaychen/grpclb2etcd/blob/master/utils/ketama.go)

####    数据结构
ketama的管理节点定义如下：
```GO
type KetamaConsistent struct {
	sync.RWMutex
	hash       HashFunc
	replicas   int
	ringKeys   []int          //  Sorted keys（最终RING）
	hashMap    map[int]string //最终ring上节点的映射
	serviceMap map[string][]string
}
```


##  0x03    其他consistent-hash

####     Jump Consistent Hash
可以参考此文：[一致性哈希算法（三）- 跳跃一致性哈希法](https://writings.sh/post/consistent-hashing-algorithms-part-3-jump-consistent-hash)，根据[论文]()中的测试结果， Jump Consistent Hash在执行速度、内存消耗、映射均匀性上比ketama要好。


##  0x0 参考
-   [一致性 hash 算法 - consistent hashing](https://blog.csdn.net/sparkliang/article/details/5279393)
-   [一致性哈希 （Consistent Hashing）的前世今生](https://candicexiao.com/consistenthashing/)
-   [一致性哈希算法（二）- 哈希环法](https://writings.sh/post/consistent-hashing-algorithms-part-2-consistent-hash-ring)