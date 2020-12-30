---
layout:     post
title:      Etcd 最佳实（踩）践（坑）
subtitle:   Etcd 日常踩坑集锦
date:       2019-10-20
author:     pandaychen
catalog:    true
tags:
    - Etcd
---

##	0x00    Etcd 最佳实践

此篇文章是本人在学习和使用 Etcd 中，遇到的问题和一些使用心得的总结，避免重复踩坑。<br>
最近阅读的一篇文章 [三年之久的 etcd 3 数据不一致 bug 分析](http://dockerone.com/article/10077)，非常好，推荐看下。

##  0x01    介绍

Etcd 是一个基于 Raft 协议实现的高可用的 KV 存储系统，具备如下几个优点：

-   简单：Golang 实现，可直接二进制部署
-   安全：支持 SSL 证书认证，数据传输加密
-   快速：在保证强一致性的同时，读写性能还算优秀
-   可靠：采用 Raft 算法实现分布式系统数据的高可用性和强一致性

##  0x02    CAP 定理基础

####    CAP 定理三要素
分布式系统的三个指标，即 CAP 定理的三要素：

-   一致性（Consistency）
-   可用性（Availability）
-   分区容忍性（Partition Tolerance）

CAP 定理指的是，这三个要素中最多只能同时实现两点，不可能三者兼顾。因此在进行分布式架构设计时，必须做出取舍。而对于分布式数据系统，分区容忍性是基本要求，否则就失去了价值。因此设计分布式数据系统，就是在一致性和可用性之间取一个平衡。而对于大多数 Web 应用，其实并不需要强一致性，因此牺牲一致性而换取高可用性，是目前多数分布式数据库产品的方向。

当然，牺牲一致性，并不是完全不管数据的一致性，否则数据是混乱的，那么系统可用性再高分布式再好也没有了价值。牺牲一致性，只是不再要求关系型数据库中的强一致性，而是只要系统能达到最终一致性即可，考虑到客户体验，这个最终一致的时间窗口，要尽可能的对用户透明，也就是需要保障 "用户感知到的一致性"。通常是通过数据的多份异步复制来实现系统的高可用和数据的最终一致性的，"用户感知到的一致性" 的时间窗口则取决于数据复制到一致状态的时间。

下面这个例子，可以解释为啥 C、A 和 P 这三者为什么不能同时满足：

假设有两台机器 A、B，两者之间互相同步保持数据的一致性。因为网络抖动是客观存在的事实，现在 B 由于网络原因不能与 A 通信（Network Partition），假设某个客户端向 A 写入数据，现在有两种选择：

1.  A 拒绝写入，这样能保证与 B 的一致性，但是牺牲了可用性
2.  A 允许写入，但是这样就不能保证与 B 的一致性了

Network Partition 是必然的，网络非常可能出现问题（断线、超时），因此 CAP 理论一般只能 AP 或 CP，而 CA 一般较难实现。

-   CP: 要实现一致性，则需要一定的一致性算法，一般是基于多数派表决的，如 Paxos 和 Raft
-   AP: 要实现可用性，则要有一定的策略决议到底用哪个数据，并且数据一般要进行冗余备份（replication），比如 Redis 集群中使用的 [Gossip 协议](https://en.wikipedia.org/wiki/Gossip_protocol)

当然，在上面的例子中，A 可以先允许写入，等 B 的网络恢复以后再同步至 B（根据 CAP 原理这样不能保证强一致性了，但是可以考虑实现最终一致性）

####    最终一致性（Eventually Consistent）

对于一致性，可以分为从客户端和服务端两个不同的视角。从客户端来看，一致性主要指的是多并发访问时更新过的数据如何获取的问题。从服务端来看，则是更新如何复制分布到整个系统，以保证数据最终一致。一致性是因为有并发读写才有的问题，因此在理解一致性的问题时，一定要注意结合考虑并发读写的场景。

从客户端角度，多进程并发访问时，更新过的数据在不同进程如何获取的不同策略，决定了不同的一致性。对于关系型数据库，要求更新过的数据能被后续的访问都能看到，这是强一致性。如果能容忍后续的部分或者全部访问不到，则是弱一致性。如果经过一段时间后要求能访问到更新后的数据，则是最终一致性。

####    经典的 CAP 图
![image](https://s2.ax1x.com/2019/11/08/MZr0WF.png)

##  0x03    Etcd 原理
Etcd 官方提供了一个动态演示集群原理的项目 <br>
[play.etcd.io](https://github.com/etcd-io/etcdlabs)

官方的 API 文档在这里：[Etcd-API](https://etcd.io/docs/v3.2.17/learning/api/)

##  0x04    物理节点

####    物理节点监控
物理节点的监控 + 进程监控 <br>
对于 `N` 个物理节点的 Etcd 集群，挂掉 $\frac{N-1}{2}$ 台，剩余节点组成的集群可以正常工作。

####    各个物理节点的 NTP 同步
Etcd 中，存在租约的概念，租约过期后，Key 就会被删除。假设我们三台机器组成了 Etcd 集群，那么如果其中某台机器的 `NTP` 误差较大的话，是存在风险的，可能会导致设置的 Lease 时间和预期不符。所以必要的 `NTP` 同步是需要的

##  0x05    客户端

####    版本差异
1.	Etcd 有 V2 和 V3 版本，数据是不互通的，所以勿用 V3 的 API 去操作 V2 版本 API 写入的数据，反之亦然
2.	在 V2 版本中，每一个 key 都进行一次 Etcd 的 `Set` 操作，这个操作是加锁的，所以，在读写的情况下会耗时很多在锁上面。V3 版本中，是先聚合，再每 128 个操作进行一次事务性执行。V3 较 V2 性能提升明显。

####    更安全的客户端
在很多环境中我们启动 Etcd 都是通过配置 TLS 方式进行的（比如 Kubernetes）, 所以在连接 Etcd 的时候需要使用 TLS 方式连接：
```golang
tlsInfo := transport.TLSInfo{
    CertFile:      "etcd-v3.3.12-linux-amd64/etcd.pem",
    KeyFile:       "etcd-v3.3.12-linux-amd64/etcd-key.pem",
    TrustedCAFile: "etcd-v3.3.12-linux-amd64/ca.pem",
}
tlsConfig, err := tlsInfo.ClientConfig()
if err != nil {
    log.Fatal(err)
}
config := clientv3.Config{
    Endpoints: []string{"127.0.0.1:2379"},
    TLS:       tlsConfig,
}
client, e := clientv3.New(config)
if e != nil {
    log.Fatal(e.Error())
}
defer client.Close()
```

当然也可以使用 User+Password 的方式来创建，看这里的 [clientv3.Client 结构](https://godoc.org/github.com/coreos/etcd/clientv3#Client)
```golang
type Client struct {
    Cluster
    KV
    Lease
    Watcher
    Auth
    Maintenance
    // Username is a user name for authentication.
    Username string
    // Password is a password for authentication.
    Password string
    // contains filtered or unexported fields
}
```

此外，由于 EtcdV3 的客户端是 gRPC 实现的，所以也提供了 gRPC 拦截器的初始化：
```golang
cli, err := clientv3.New(clientv3.Config{
    Endpoints: endpoints,
    DialOptions: []grpc.DialOption{
        grpc.WithUnaryInterceptor(grpcprom.UnaryClientInterceptor),
        grpc.WithStreamInterceptor(grpcprom.StreamClientInterceptor),
    },
})
```

##  0x06    MVCC
MVCC(Multiversion concurrency control 多版本并发控制)，Etcd 在内存中维护了一个 BTREE(B 树) 纯内存索引，它是有序的。(回想起 MYSQL 的索引也是 BTREE 实现的，极大的提升查找效率)<br>
在这个 BTREE 中，整个 KV 存储大概就是这样：
```golang
type treeIndex struct {
    sync.RWMutex
    tree *btree.BTree
}
```
当存储大量的 Key-Value 时，因为用户的 Value 一般比较大，全部放在内存 BTREE 里内存耗费过大，所以 Etcd 将用户 Value 保存在磁盘中。

Etcd 在事件模型（Watch 机制）上与 ZooKeeper 完全不同，每次数据变化都会通知，并且通知里携带有变化后的数据内容，其基础就是自带 MVCC 的 Bboltdb 存储引擎。

####	MVCC 要点

1.	每个 tx 事务有唯一事务 ID，在 Etcd 中叫做 main ID，全局递增不重复
2.	一个 tx 可以包含多个修改操作（put 和 delete），每一个操作叫做一个 revision（修订），共享同一个 main ID
3.	一个 tx 内连续的多个修改操作会被从 0 递增编号，这个编号叫做 sub ID
4.	每个 Revision 由（main ID，sub ID）唯一标识

####    关于 Version/Revision/ModRevison 的概念与区别
从 MVCC 引出 `Version`/`Revision`/`ModRevison` 这三个重要概念：<br>

-	`Revision` 表示改动序号（ID），每次 KV 的变化，leader 节点都会修改 `Revision` 值，因此，这个值在 cluster 内是全局唯一的，而且是递增的。

-	`ModRevison` 记录了某个 key 最近修改时的 Revision，即它是与 key 关联的。
-   `Version` 表示 KV 的版本号，初始值为 1，每次修改 KV 对应的 `Version` 都会加 1，也就是说它是作用在 KV 之内的。

使用参数 `--write-out` 可以格式化（json/fields ...）输出详细的信息，包括 `Revision`、`ModRevison`、`Version`，此外，还包括 `LeaseID`

如：
```javascript
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

####    Revision 和 CreateVsion 的区别

##  0x07    数据压缩

####    定期压缩（compac）、碎片整理（defrag）

对于 Etcd 这种多版本的 kv 存储系统而言，每一次成功修改数据的原子操作，都会被记录到新的版本中，每一个历史版本的数据都会被完整保存下来。由于 Etcd 本身是磁盘存储，随着数据量的增大，不可避免的会出现两个问题：一是数据体积增大、二是磁盘碎片增多；随着修改次数的增多，存储的数据量会越来越大，这对 Etcd 集群的性能和稳定性都会带来很大的影响。<br>

因此，在大量使用的 Etcd 的实际生产场景中，需要考虑优化 Etcd 集群的配置，定期做 compact 和 defrag，且对每个节点的 defrag 时间需要错开，不能同时进行。Etcd 提供了如下参数来帮助我们实现自动压缩和碎片整理：

```bash
1. --auto-compaction-retention
```

```bash
2.--max-request-bytes
```

```bash
3.--quota-backend-bytes
```
此参数 Etcd-db 数据大小，默认是 2G, 当数据达到 2G 的时候就不允许写入，必须对历史数据进行压缩才能继续写入。在启动的时候就应该提前确定大小，官方推荐是 8G

##  0x08    租约机制

####    Lease 机制（Heartbeat）
EtcdV3 中提供了自动续期的函数 [Lease.KeepAlive](https://github.com/Etcd-io/Etcd/blob/master/clientv3/lease.go#L108)，可以实现自动定时的续约某个租约（绑定到某个 KEY）。关于 Lease 的 TTL 时间设置大小，是有个双刃剑的问题：
1.  如果 LeaseID 过长，某台应用服务故障（服务不可用），导致 Lease 突然中断且 Etcd 不能及时感知到服务下线，那么来自客户端的请求很有可能继续发送到故障的服务，从而导致调用失败；
2.  如果 LeaseID 过短，网络的突然抖动，导致 Key 在 Lease 未成功续期而被 Etcd 移除，这就导致应用服务是正常的，但是在 Etcd 中，该（应用服务）因为 Lease 的 TTL 过期导致节点（KEY）已经不存在了。
3.  KeepAlive 和 Put 一样，如果在执行之前 Lease 就已经过期了，那么需要重新分配 Lease。Etcd 并没有提供 API 来实现原子的 Put with Lease。


解决的方法，我这里提供两点思路：
1.  使用定时器 timer 代替 Lease 的 Keepalive 功能，定时 PUT-KEY-with-Lease，续期时间建议设置为 LeaseTTL 的 1/3
2.  监听 Lease 的 channel，如果发现 channel 被动关闭了，重新申请 Lease 然后再执行 KeepAlive 方法进行续期（当然了，也要先判断 Key 是不是丢失了）

总结一下：<br>
Etcd 的 lease 可以用来做心跳，监控模块存活状态。Lease 的存活时间决定了发现服务异常的及时性，太长会较晚才能发现服务异常，较短容易受网络波动的影响。另外一种解决的思路是结合心跳和探测。把 Lease 设置为一个较小的值，在发现心跳消失的时候，做网络探测，确实不通了，再判定为不可用。需要注意的是，上报心跳的进程要对 lease not found 这种情况做处理，重新生成一个 lease 进行上报。

以下上 GODOC 中对 KeepAlive 方法的说明：
```bash
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

##  0x09    Watcher（监听器）

Etcd 提供了 watcher，来监控集群 kv 的变化。这个在开发 gRPC 服务发现的 ClientConn 实时更新接口时，必不可少。但是 Watch 返回的 WatchChan 有可能在运行过程中失败而关闭，此时 WatchResponse.Canceled 会被置为 true，WatchResponse.Err() 也会返回具体的错误信息。所以在 range WatchChan 的时候，每一次循环都要检查 WatchResponse.Canceled，在关闭的时候重新发起 Watch 或报错。


##  0x0A    参考文档
-   [Godoc - package clientv3](https://godoc.org/github.com/Etcd-io/Etcd/clientv3)
-   [etcd/Documentation/op-guide/maintenance.md](https://github.com/Etcd-io/Etcd/blob/master/Documentation/op-guide/maintenance.md)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
