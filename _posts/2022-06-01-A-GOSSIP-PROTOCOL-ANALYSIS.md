---
layout:     post
title:      CAP 分析 && 实战：Gossip 协议
subtitle:   AP 协议：gossip && hashicorp/memberlist 分析
date:       2022-06-01
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - GOSSIP
    - CAP
---

##  0x00    前言
前文 []() 中，了解到 CAP 定理中，满足 AP 的典型协议就是 Gossip，本文就介绍下该协议。

Gossip protocol 也叫 Epidemic Protocol （流行病协议），是一种 **去中心化、容错并保证最终一致性的协议**；其基本思想是通过不断的和集群中的节点 Gossip 交换信息，经过 `O(log(N))` 个回合, Gossip 协议即可将信息传递到所有的节点（其中 `N` 表示节点的个数）。典型的实现有这么几个：
-   Redis 集群间通信
-   Consul 集群间各节点交换数据

Gossip 协议的目的是解决状态在集群中的传播和状态一致性的保证两个问题

![gossip](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/cap/gossip/gossip-protocol-1.png)

上图中的每个节点均为运行着 Gossip 协议的 Agent（收发 Gossip 协议消息），包括服务器节点与普通节点。最终一致指的是经过一段时间后系统的状态达成一致。

![GIF](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/cap/gossip/gossip-protocol-2.gif)


从算法上的解释如下：Gossip 协议本质上是一个并行的图广度优先遍历算法。假设 `A` 得到某些信息，更新了自身的信息，`A` 需要将信息告诉 `B`、`C` 等，然后 `B`、`C` 告诉其他的 `D`、`E`、`F`、`G`，一直遍历。如果节点 `B` 收到 `A` 的消息，发现自己早就知道这个消息就直接忽略，从而可以防止图重复遍历

##  0x01    Gossip 协议基础
Gossip 协议的特点是：
1、可扩展（Scalable）<br>
gossip 协议是可扩展的，每个节点仅发送固定数量的消息。在数据传送的时候，节点并不会等待消息的 ack，所以消息传送失败也没有关系，因为可以通过其他节点将消息传递给之前传送失败的节点

2、容错（Fault-tolerance）<br>
网络中任何节点的重启或者宕机都不会影响 gossip 协议的运行

3、健壮（Robust）<br>
gossip 协议是去中心化的协议，所以集群中的所有节点都是对等的，没有特殊的节点，所以任何节点出现问题都不会阻止其他节点继续发送消息。任何节点都可以随时 Join/Leave，而不会影响系统的整体服务质量 QOS

4、最终一致性（Convergent consistency）<br>
Gossip 协议实现信息指数级的快速传播，因此在有新信息需要传播时，消息可以快速地发送到全局节点，在有限的时间内能够做到所有节点都拥有最新的数据

####    Gossip 通信过程
Gossip 协议分为 Push-based 和 Pull-based 两种模式，具体工作流程如下：

1、Push-based 模式 <br>

![]()

-   网络中的某个节点随机选择 `N` 个节点作为数据接收对象
-   该节点向其选中的 `N` 个节点传输相应数据
-   接收到数据的节点对数据进行存储
-   接收到数据的节点再从第一步开始周期性执行

如此经过一段时间之后，整个集群都收到了此条 Gossip 消息，从而达到最终一致。另外注意每次消息传递都会选择尚未发送过的节点进行传播（单向的），如 `A->B`, 则当 `B` 进行传播的时候，不会再发送给 `A`

2、Pull-based 模式 <br>

![]()

-   集群内的所有节点，随机选择其它 `k` 个节点询问有没有新数据
-   接收到请求的节点，返回新数据


####    通信模式
在 Gossip 协议下，网络中两个节点之间有 `3` 种通信方式:
1.  Push: 节点 `A` 将数据 `(key,value,version)` 及对应的版本号推送给 `B` 节点，`B` 节点更新 `A` 中比自己新的数据
2.  Pull：`A` 仅将数据 key, version 推送给 `B`，`B` 将本地比 `A` 新的数据 `(Key, value, version)` 推送给 `A`，`A` 更新本地
3.  Push/Pull：比 `Pull` 方式多了一步，`A` 再将本地比 `B` 新的数据推送给 `B`，`B` 则更新本地

如果把两个节点数据同步一次定义计为一个周期，则在一个周期内，Push 需通信 `1` 次，Pull 需 `2` 次，Push/Pull 则需 `3` 次。虽然消息数增加了，但从效果上来讲，Push/Pull 最好（快），理论上一个周期内可以使两个节点完全一致

####    Gossip 的缺点
1、消息延迟 <br>
由于 Gossip 协议中，节点只会随机向少数几个节点发送消息，消息最终是通过多个轮次的散播而到达全网的，因此使用 Gossip 协议会造成不可避免的消息延迟。不适合用在对实时性要求较高的场景下

2、消息存在冗余 <br>
Gossip 协议规定，节点会定期随机选择周围节点发送消息，而收到消息的节点也会重复该步骤，因此就不可避免的存在消息重复发送给同一节点的情况，造成了消息的冗余，同时也增加了收到消息的节点的处理压力

##  0x02    Consul 中的 gossip 分析
Consul 主要使用 Gossip 协议实现如下功能：
1.  集群成员管理
2.  消息广播

Consul 的 Agent 之间通过 Gossip 协议进行状态检查，通过节点之间互 `ping` 而减轻了 Server 的节点的压力。如果有节点宕机，任意与其保持连接的节点发现即可通过 Gossip 广播给整个集群。当该故障机的节点重启后重新加入集群，一段时间后，它的状态也能够通过 Gossip 协议与其他的节点达成一致，这体现出 Gossip 协议具有的天然的分布式容错的特点

####    两种Gossip池：LAN && WAN 池
1、LAN 池<br>
Consul 中的每个数据中心有一个 LAN 池，它包含了该数据中心的所有成员，包括 clients 和 servers。

2、WAN 池<br>
WAN 池是全局唯一的，所有数据中心的 Server 都应该加入到 WAN 池中，由 WAN 池提供的成员关系信息允许 server 做一些跨数据中心的请求

####    gossip库的实现： hashicorp/memberlist
`hashicorp/memberlist`基于 Gossip 协议，实现了集群内节点发现、节点失效探测、节点故障转移、节点状态同步等。



##  0x03    hashicorp/memberlist测试
用memberlist库实现集群节点发现的功能，样例代码[参考]()，下面输出演示了一个`3`节点的集群，手动停止一个节点后的变化情况（最终变成`2`个节点）

```text
-------------start--------------
Member: VM_120_245_centos-8002 192.168.10.1
Member: VM_120_245_centos-8003 192.168.10.1
Member: VM_120_245_centos-8001 192.168.10.1
-------------end--------------
2022/10/05 01:01:43 [DEBUG] memberlist: Stream connection from=9.134.120.245:52117
2022/10/05 01:01:44 [DEBUG] memberlist: Initiating push/pull sync with: VM_120_245_centos-8001 192.168.10.1:8001
-------------start--------------
Member: VM_120_245_centos-8001 192.168.10.1
Member: VM_120_245_centos-8003 192.168.10.1
Member: VM_120_245_centos-8002 192.168.10.1
-------------end--------------
-------------start--------------
Member: VM_120_245_centos-8002 192.168.10.1
Member: VM_120_245_centos-8001 192.168.10.1
Member: VM_120_245_centos-8003 192.168.10.1
-------------end--------------
-------------start--------------
Member: VM_120_245_centos-8003 192.168.10.1
Member: VM_120_245_centos-8002 192.168.10.1
Member: VM_120_245_centos-8001 192.168.10.1
-------------end--------------
-------------start--------------
Member: VM_120_245_centos-8002 192.168.10.1
Member: VM_120_245_centos-8003 192.168.10.1
Member: VM_120_245_centos-8001 192.168.10.1
-------------end--------------
-------------start--------------
Member: VM_120_245_centos-8001 192.168.10.1
Member: VM_120_245_centos-8003 192.168.10.1
Member: VM_120_245_centos-8002 192.168.10.1
-------------end--------------
2022/10/05 01:01:58 [DEBUG] memberlist: Stream connection from=9.134.120.245:52122
2022/10/05 01:01:59 [DEBUG] memberlist: Initiating push/pull sync with: VM_120_245_centos-8001 192.168.10.1:8001
-------------start--------------
Member: VM_120_245_centos-8001 192.168.10.1
Member: VM_120_245_centos-8003 192.168.10.1
Member: VM_120_245_centos-8002 192.168.10.1
-------------end--------------
-------------start--------------
Member: VM_120_245_centos-8001 192.168.10.1
Member: VM_120_245_centos-8003 192.168.10.1
Member: VM_120_245_centos-8002 192.168.10.1
-------------end--------------
2022/10/05 01:02:04 [DEBUG] memberlist: Failed UDP ping: VM_120_245_centos-8003 (timeout reached)
2022/10/05 01:02:04 [INFO] memberlist: Suspect VM_120_245_centos-8003 has failed, no acks received
-------------start--------------
Member: VM_120_245_centos-8001 192.168.10.1
Member: VM_120_245_centos-8002 192.168.10.1
Member: VM_120_245_centos-8003 192.168.10.1
-------------end--------------
2022/10/05 01:02:06 [DEBUG] memberlist: Failed UDP ping: VM_120_245_centos-8003 (timeout reached)
2022/10/05 01:02:06 [INFO] memberlist: Suspect VM_120_245_centos-8003 has failed, no acks received
2022/10/05 01:02:07 [INFO] memberlist: Marking VM_120_245_centos-8003 as failed, suspect timeout reached (1 peer confirmations)
2022/10/05 01:02:08 [DEBUG] memberlist: Failed UDP ping: VM_120_245_centos-8003 (timeout reached)
2022/10/05 01:02:08 [INFO] memberlist: Suspect VM_120_245_centos-8003 has failed, no acks received
-------------start--------------
Member: VM_120_245_centos-8001 192.168.10.1
Member: VM_120_245_centos-8002 192.168.10.1
-------------end--------------
-------------start--------------
Member: VM_120_245_centos-8002 192.168.10.1
Member: VM_120_245_centos-8001 192.168.10.1
```

##  0x04  参考
-   [P2P 网络核心技术：Gossip 协议](https://zhuanlan.zhihu.com/p/41228196)
-   [](https://zhuanlan.zhihu.com/p/369178156)
-   [聊聊 Gossip 的一个实现]()