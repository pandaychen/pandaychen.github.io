---
layout:     post
title:      Etcd 最佳实(踩)践(坑)
subtitle:   Etcd 日常踩坑集锦
date:       2019-10-20
author:     pandaychen
catalog:    true
tags:
    - Etcd
    - MVCC
---

##	Etcd 最佳实践

此篇文章是本人在学习和使用Etcd中，遇到的问题和一些使用心得的总结，避免重复踩坑。：）<br>
（PS：当然了，大部分的情况，都是使用者的问题）<br>

###	Introduce

### 物理节点监控
物理节点的监控+进程监控<br>
对于N个物理节点的Etcd集群，挂掉(N-1)/2台，剩余节点组成的集群可以正常工作。

### 各个物理节点的时间同步（NTP）

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


