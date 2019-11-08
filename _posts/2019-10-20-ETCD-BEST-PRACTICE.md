---
layout:     post
title:      Etcd 最佳实(踩)践(坑)
subtitle:   Etcd 日常踩坑集锦
date:       2019-10-20
author:     pandaychen
catalog:    true
tags:
    - Etcd
---

##	Etcd 最佳实践

此篇文章是本人在学习和使用Etcd中，遇到的问题和一些使用心得的总结，避免重复踩坑。：）<br>
（PS：当然了，大部分的情况，都是使用者的问题，想熟练的掌握Etcd，哪能不踩几个雷）<br>


### 介绍

Etcd 是一个基于Raft协议实现的高可用的KV存储系统，具备如下几个优点：

-   简单：Golang实现，可直接二进制部署
-   安全：支持 SSL 证书认证，数据传输加密
-   快速：在保证强一致性的同时，读写性能还算优秀
-   可靠：采用 Raft 算法实现分布式系统数据的高可用性和强一致性

### CAP定理基础 

#####    CAP定理三要素
分布式系统的三个指标，即CAP定理的三要素：

-   一致性（Consistency）
-   可用性（Availability）
-   分区容忍性（Partition Tolerance）

CAP定理指的是，这三个要素中最多只能同时实现两点，不可能三者兼顾。因此在进行分布式架构设计时，必须做出取舍。而对于分布式数据系统，分区容忍性是基本要求，否则就失去了价值。因此设计分布式数据系统，就是在一致性和可用性之间取一个平衡。而对于大多数web应用，其实并不需要强一致性，因此牺牲一致性而换取高可用性，是目前多数分布式数据库产品的方向。

当然，牺牲一致性，并不是完全不管数据的一致性，否则数据是混乱的，那么系统可用性再高分布式再好也没有了价值。牺牲一致性，只是不再要求关系型数据库中的强一致性，而是只要系统能达到最终一致性即可，考虑到客户体验，这个最终一致的时间窗口，要尽可能的对用户透明，也就是需要保障"用户感知到的一致性"。通常是通过数据的多份异步复制来实现系统的高可用和数据的最终一致性的，"用户感知到的一致性"的时间窗口则取决于数据复制到一致状态的时间。

下面这个例子，可以解释为啥C、A和P这三者为什么不能同时满足：

假设有两台机器A、B，两者之间互相同步保持数据的一致性。因为网络抖动是客观存在的事实，现在B由于网络原因不能与A通信（Network Partition），假设某个客户端向A写入数据，现在有两种选择：

1.  A拒绝写入，这样能保证与B的一致性，但是牺牲了可用性
2.  A允许写入，但是这样就不能保证与B的一致性了

Network Partition是必然的，网络非常可能出现问题（断线、超时），因此CAP理论一般只能AP或CP，而CA一般较难实现。

-   CP: 要实现一致性，则需要一定的一致性算法，一般是基于多数派表决的，如Paxos和Raft
-   AP: 要实现可用性，则要有一定的策略决议到底用哪个数据，并且数据一般要进行冗余备份（replication），比如Redis集群中使用的[Gossip协议](https://en.wikipedia.org/wiki/Gossip_protocol)

当然，在上面的例子中，A可以先允许写入，等B的网络恢复以后再同步至B（根据CAP原理这样不能保证强一致性了，但是可以考虑实现最终一致性）

#####    最终一致性（Eventually Consistent）

对于一致性，可以分为从客户端和服务端两个不同的视角。从客户端来看，一致性主要指的是多并发访问时更新过的数据如何获取的问题。从服务端来看，则是更新如何复制分布到整个系统，以保证数据最终一致。一致性是因为有并发读写才有的问题，因此在理解一致性的问题时，一定要注意结合考虑并发读写的场景。

从客户端角度，多进程并发访问时，更新过的数据在不同进程如何获取的不同策略，决定了不同的一致性。对于关系型数据库，要求更新过的数据能被后续的访问都能看到，这是强一致性。如果能容忍后续的部分或者全部访问不到，则是弱一致性。如果经过一段时间后要求能访问到更新后的数据，则是最终一致性。

#####    经典的CAP图

![image](https://s2.ax1x.com/2019/11/08/MZr0WF.png)

###	Some Funny Things
Etcd官方提供了一个动态演示集群原理的项目<br>
![play.etcd.io](https://github.com/etcd-io/etcdlabs)

官方的API文档在这里：![Etcd-API](https://etcd.io/docs/v3.2.17/learning/api/)

### 物理节点监控
物理节点的监控+进程监控<br>
对于N个物理节点的Etcd集群，挂掉(N-1)/2台，剩余节点组成的集群可以正常工作。

### 各个物理节点的时间同步（NTP）
Etcd中，存在租约的概念，租约过期后，Key就会被删除。假设我们三台机器组成了Etcd集群，那么如果其中某台机器的NTP误差较大的话，是存在风险的，可能会导致设置的Lease时间和预期不符。所以必要的NTP同步是需要的

###	版本问题
1.	Etcd有V2和V3版本，数据是不互通的，所以别用V3的API去操作V2版本API写入的数据，反之亦然
2.	在V2版本中，每一个key都进行一次Etcd的set操作，这个操作是加锁的，所以，在读写的情况下会耗时很多在锁上面。V3版本中，是先聚合，再每128个操作进行一次事务性执行。V3较V2性能提升明显。


### MVCC
MVCC( Multiversion concurrency control 多版本并发控制 )，Etcd在内存中维护了一个 BTREE(B树)纯内存索引，它是有序的。(回想起MYSQL的索引也是BTREE实现的，极大的提升查找效率)<br>
在这个BTREE中，整个KV存储大概就是这样：
```
type treeIndex struct {
    sync.RWMutex
    tree *btree.BTree
}
```
当存储大量的Key-Value时，因为用户的value一般比较大，全部放在内存btree里内存耗费过大，所以Etcd将用户value保存在磁盘中。

Etcd在事件模型（watch 机制）上与ZooKeeper完全不同，每次数据变化都会通知，并且通知里携带有变化后的数据内容，其基础就是自带 MVCC 的 bboltdb 存储引擎。

###	MVCC要点

1.	每个tx事务有唯一事务ID，在Etcd中叫做main ID，全局递增不重复
2.	一个tx可以包含多个修改操作（put和delete），每一个操作叫做一个revision（修订），共享同一个main ID
3.	一个tx内连续的多个修改操作会被从0递增编号，这个编号叫做sub ID
4.	每个Revision由（main ID，sub ID）唯一标识

###	关于Version/Revision/ModRevison的概念与区别
从MVCC引出Version/Revision/ModRevison这三个重要概念：<br>

-	Revision 表示改动序号（ID），每次 KV 的变化，leader 节点都会修改 Revision 值，因此，这个值在cluster内是全局唯一的，而且是递增的。

-	ModRevison 记录了某个 key 最近修改时的 Revision，即它是与 key 关联的。
-Version 表示 KV 的版本号，初始值为 1，每次修改 KV 对应的 version 都会加 1，也就是说它是作用在 KV 之内的。

-	使用参数 --write-out 可以格式化（json/fields ...）输出详细的信息，包括 Revision、ModRevison、Version，此外，还包括LeaseID

如：
```
Etcdctl get /a/b --prefix --write-out=fields    #

"Key" : "/a/b/key1-Iag4se1uz1"
"CreateRevision" : 3758
"ModRevision" : 3758
"Version" : 1
"Value" : "127.0.0.1:11111"
"Lease" : 6547788213836736818
"Key" : "/a/b/key2-IYg3We7uzQ"
"CreateRevision" : 3754
"ModRevision" : 3754
"Version" : 1
"Value" : "127.0.0.1:11112"
"Lease" : 6547788213836736811
"More" : false
"Count" : 15
```

###	定期压缩（compac）、碎片整理（defrag）

对于Etcd这种多版本的kv存储系统而言，每一次成功修改数据的原子操作，都会被记录到新的版本中，每一个历史版本的数据都会被完整保存下来。由于Etcd本身是磁盘存储，随着数据量的增大，不可避免的会出现两个问题：一是数据体积增大、二是磁盘碎片增多；随着修改次数的增多，存储的数据量会越来越大，这对Etcd集群的性能和稳定性都会带来很大的影响。<br>

因此，在大量使用的Etcd的实际生产场景中，需要考虑优化Etcd集群的配置，定期做compact和defrag，且对每个节点的defrag时间需要错开，不能同时进行。Etcd提供了如下参数来帮助我们实现自动压缩和碎片整理：

```
1. --auto-compaction-retention  
```

```
2.--max-request-bytes
```

```
3.--quota-backend-bytes
```
此参数Etcd-db数据大小，默认是2G,当数据达到2G的时候就不允许写入，必须对历史数据进行压缩才能继续写入。在启动的时候就应该提前确定大小，官方推荐是8G

###  Lease机制（Heartbeat）
EtcdV3中提供了自动续期的函数[Lease.KeepAlive](https://github.com/Etcd-io/Etcd/blob/master/clientv3/lease.go#L108)，可以实现自动定时的续约某个租约（绑定到某个KEY）。关于Lease的TTL时间设置大小，是有个双刃剑的问题：
1.  如果LeaseID过长，某台应用服务故障（服务不可用），导致Lease突然中断且Etcd不能及时感知到服务下线，那么来自客户端的请求很有可能继续发送到故障的服务，从而导致调用失败；
2.  如果LeaseID过短，网络的突然抖动，导致Key在Lease未成功续期而被Etcd移除，这就导致应用服务是正常的，但是在Etcd中，该（应用服务）因为Lease的TTL过期导致节点（KEY）已经不存在了。
3.  KeepAlive和Put一样，如果在执行之前Lease就已经过期了，那么需要重新分配Lease。Etcd并没有提供API来实现原子的Put with Lease。


解决的方法，我这里提供两点思路：
1.  使用定时器timer代替Lease的Keepalive功能，定时PUT-KEY-with-Lease，续期时间建议设置为LeaseTTL的1/3
2.  监听Lease的channel，如果发现channel被动关闭了，重新申请Lease然后再执行KeepAlive方法进行续期（当然了，也要先判断Key是不是丢失了）

总结一下：<br>
Etcd的lease可以用来做心跳，监控模块存活状态。Lease的存活时间决定了发现服务异常的及时性，太长会较晚才能发现服务异常，较短容易受网络波动的影响。另外一种解决的思路是结合心跳和探测。把Lease设置为一个较小的值，在发现心跳消失的时候，做网络探测，确实不通了，再判定为不可用。需要注意的是，上报心跳的进程要对lease not found这种情况做处理，重新生成一个lease进行上报。

以下上GODOC中对KeepAlive方法的说明：
```
   // KeepAlive attempts to keep the given lease alive forever. If the keepalive responses posted
    // to the channel are not consumed promptly the channel may become full. When full, the lease
    // client will continue sending keep alive requests to the Etcd server, but will drop responses
    // until there is capacity on the channel to send more responses.
    //
    // If client keep alive loop halts with an unexpected error (e.g. "Etcdserver: no leader") or
    // canceled by the caller (e.g. context.Canceled), KeepAlive returns a ErrKeepAliveHalted error
    // containing the error reason.
    //
    // The returned "LeaseKeepAliveResponse" channel closes if underlying keep
    // alive stream is interrupted in some way the client cannot handle itself;
    // given context "ctx" is canceled or timed out.
    //
    // TODO(v4.0): post errors to last keep alive message before closing
    // (see https://github.com/Etcd-io/Etcd/pull/7866)
```

###	Watcher（监听器）

Etcd提供了watcher，来监控集群kv的变化。这个在开发GRPC服务发现的ClientConn实时更新接口时，必不可少。但是Watch返回的WatchChan有可能在运行过程中失败而关闭，此时WatchResponse.Canceled会被置为true，WatchResponse.Err()也会返回具体的错误信息。所以在range WatchChan的时候，每一次循环都要检查WatchResponse.Canceled，在关闭的时候重新发起Watch或报错。


### 参考文档
-   https://godoc.org/github.com/Etcd-io/Etcd/clientv3
-   https://github.com/Etcd-io/Etcd/blob/master/Documentation/op-guide/maintenance.md
-   


