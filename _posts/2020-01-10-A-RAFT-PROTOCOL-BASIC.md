---
layout: post
title: Raft 协议分析与实战（理论篇）
subtitle: Raft：一种更容易理解的共识算法
date: 2020-01-10
author: pandaychen
catalog: true
tags:
  - 分布式理论
  - Raft
  - Etcd
---

## 0x00 前言

本文是对于 Raft 协议原理的一些总结。原文 [在此](https://raft.github.io/raft.pdf)

> Raft implements consensus by first electing a distinguished leader, then giving the leader complete responsibility for managing the replicated log. The leader accepts log entries from clients, replicates them on other servers, and tells servers when it is safe to apply log entries to their state machines. A leader can fail or become disconnected from the other servers, in which case a new leader is elected.


共识算法就是保证一个集群的多台机器协同工作，在遇到请求时，数据能够保持一致。即使遇到机器宕机，整个系统仍然能够对外保持服务的可用性。

####  CAP定理

[CAP 定理基础](https://pandaychen.github.io/2019/10/20/ETCD-BEST-PRACTISE/#0x02-cap-%E5%AE%9A%E7%90%86%E5%9F%BA%E7%A1%80)

CAP ： **一个分布式系统不可能同时满足一致性 （C： Consistency）、可用性（A：Availability）和分区容错性（P：Partition tolerance）这三个基本需求，最多只能同时满足其中的两项**


分布式存储架构中，设计共识算法要考虑在一个复制集群中，所有节点按照确认的顺序处理命令，最终结果是在客户端看来，一个复制集群表现的像一个单状态机一样一致。为了保证多个节点顺序一致，需要处理如下问题：

- 非拜占庭错误，包含网络延迟，分区，丢包，重复和乱序
- 只要多数节点（quorum）正常，集群就能对外提供稳定的服务。错误的节点可以在稍后恢复后重新加入集群服务
- 不依赖系统时钟，实现修改的顺序一致性
- 性能，多数节点达成一致即可，不能因为少量的慢节点拖累整个集群的性能
- 脑裂问题

![distribute](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/distribute-storage-architecture.png)

####  Raft 要解决什么样的问题
由于分布式系统引入了多个节点，节点规模越大，宕机、网络时延、网络分区就会成为常态，任何一个问题都可能导致节点之间的数据不一致，Raft 是用来解决分布式场景下的一致性问题的共识算法。

举例来说有`5`台机器组成的集群，需要保证机器之间数据同步，有如下问题需要考虑：

- 客户端进行写入的时候，是直接写入这`5`台机器再返回吗？这样如果某台机器比较慢，势必会影响性能
- 那如果只写入一台机器，新的数据要如何同步到其他机器呢？服务应该在什么时候给客户端返回来保证数据的强一致性呢？
- 多个客户端同时写入的时候，如何保证同步的时候操作执行的顺序是一致的呢？或者说如何保证数据不会乱呢？
- 当客户端读取数据的时候会不会存在读到老数据的可能？应该以一种什么样的方式去读才能读取到最新的数据呢？

这些问题都是分布式系统经常会面临的问题。Raft 一致性算法可以保证，即使在网络丢包，延迟，乱序，或者宕机，部分网络隔离的情况下，集群中机器能保证数据的强一致性。

####  Raft的核心逻辑

Raft 的核心逻辑是：先选举出 Leader，Leader 完全负责 Replicated-Log 的管理。Leader 负责接受所有客户端更新请求，然后复制到 Follower 节点，并在安全的时候执行这些请求。若 Leader 故障，Followes 会重新选举出新的 Leader。Raft 是满足 CAP 定理中的 C 和 P 特性，属于强一致性的分布式协议。通俗而言，**Raft的核心就是leader发出日志同步请求，follower接收并同步日志，最终保证整个集群的日志一致性**。

Raft 的三个核心问题：

1.  Leader Election：领导人选举（选主），主节点宕机后必须选择一个节点成为新的主节点
2.  Log Replication：日志复制（备份），主节点必须接受客户端指令日志，并强制复制到其他节点
3.  Safty：安全性，复制集群状态机必须一致。相同索引的日志指令一致，顺序一致

Raft 的优点如下：

1.  高可用：Raft 协议中，$N$ 个节点，系统容忍 $\frac{N}{2}$ 个节点的故障，通常 $N==5$。选举和日志同步都只需要大多数的节点 $\frac{N}{2}+1$ 正常互联即可，即使 Leader 故障，在选举 Term 超时到期后，集群自动选举新 Leader（不可用时间非常小）

2.  强一致：虽然 Raft 协议中所有节点的数据非实时一致，但 Raft 算法保证 Leader 节点的数据最全，同时所有修改请求（写入 / 更新 / 删除）都由 Leader 处理，<font color="#dd0000"> 从用户角度看是指永远可以读到最新的写成功的数据，而从服务内部来看指的是所有的存活节点的 State Machine 中的数据都保持一致 </font>
3.  高可靠：Raft 算法保证了 Committed 的日志不会被修改，S0tate Matchine 只应用 Committed 的日志，所以当客户端收到请求成功即代表数据不再改变。Committed 日志在大多数节点上冗余存储，少于一半的磁盘故障数据不会丢失

4.  高性能：与必须将数据写到所有节点才能返回客户端成功的算法对比，Raft 算法只需要大多数节点成功即可，少量节点处理缓慢不会延缓整体系统运行

## 0x01 Raft 协议的状态机

#### Raft 状态机节点

在 Raft 算法中，集群的节点有以下 `3` 种状态：

- `Leader`：领导者，一个 Raft 集群里只能存在一个 `Leader`
- `Follower`：跟随者，一个客户端的修改数据请求如果发送到 `Follower` 时，会首先由 `Follower` 重定向到 `Leader` 上
- `Candidate`：选举者，当一个节点切换到此状态时，将开始进行一次新的选举

每开始一次新的选举时，称为一个任期（TERM）。每个 TERM 都有一个严格递增的整数值与之关联，这个特性后面会用到

#### 状态机切换

节点的状态切换状态机如下图所示。
![raft-state](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/raft3.jpg)

1.  Start 起始状态，各个节点刚启动的时候自动进入 `Follower` 状态
2.  Times out, starts election：`Follower` 在启动之后，将开启一个选举超时定时器，当这个定时器到期时，`Follower` 将切换到 `Candidate` 状态发起选举（每个 `Follower` 都投自己一票，成为 `Candidate`）
3.  Times out, new election：进入 `Candidate` 状态之后就开始选举，但是如果在下一次选举超时之前，还没有选出一个新的 `Leader`，那么还会保持在 `Candidate` 状态重新开始一次新选举
4.  Receives votes from majority of servers：当 `Candidate` 状态的节点，收到了超过半数的节点的选票，那么其将切换状态成为新的 `Leader`，同时向其他节点广播，其他节点成为 `Follower`
5.  Discovers current leader or new term：处于 `Candidate` 状态的节点，如果收到了来自 `Leader` 的消息，或者更高任期号（Term）的消息，表示集群中已经有 `Leader` 了，将切换回到 `Follower` 状态
6.  Discovers server with higher term：`Leader` 状态下如果收到来自更高任期号的消息，将切换到 `Follower` 状态。这种情况大多数发生在有网络分区的状态下（网络分区又恢复）

Raft 节点间通过 RPC 请求来互相通信，主要有以下两类 RPC 请求：

1.  Request-Vote RPC：用于 `Candidate` 状态的节点进行选举用途
2.  Append-Entries RPC：由 `Leader` 节点向其他节点复制日志数据以及同步心跳数据

#### 任期 Term

#### 选举定时器

在 Raft 中有两个 Timeout 定时器控制着选主 Election 的进行：

1.  选举超时时间（Election Timeout）：意思是 `Follower` 要等待成为 `Candidate` 的时间（要成为 `Candidate` 后才可以投票选举），通常 Election Timeout 定时器为 `150ms~300ms`，这个时间结束之后 `Follower` 变成 `Candidate` 开始选举过程。首先是自己对自己投票（`+1`），然后向其他节点请求投票（选票），如果接收节点在收到投票请求时还没有参与过投票，那么它会响应本次请求，并把票投给这个请求投票的 `Candidate`，然后重置自身的 Election Timeout 定时器，一旦一个 `Candidate` 拥有所有节点中的大多数投票，则此节点变成一个 `Leader`
2.  心跳超时时间（Heartbeat Timeout）：当一个节点从 `Candidate` 变为 `Leader` 时，此节点开始向其他 `Follower` 发送 Append Entries，这些消息发送的频率是通过 Heartbeat Timeout 配置，`Follower` 会响应每条的 Append Entry，整个选举会一直进行直到 `Follower` 停止接受 heartbeat 并且变成 `Candidate` 开始下一轮选举（即假设此时 `Leader` 故障或者丢失了，`Follower` 检测到心跳超时（在此期间没有收到 `Leader` 发送的 Append Entry）后，再等待自身选举超时后发起新一轮选举）

## 0x02 Leader Election

当 `Leader` 节点由于异常（宕机、网络故障等）无法继续提供服务时，可以认为它结束了本轮任期（`TERM=N`），需要开始新一轮的选举（Election），而新的 `Leader` 当然要从 `Follower` 中产生，开始新一轮的任期（`TERM=N+1`）。（从 `Follower` 节点的角度来看，当它接收不到 `Leader` 节点的心跳即心跳超时时间触发时，可认为 Leader 已经丢失，便进入新一轮的选举流程）

选举的过程如下图所示：
![raft-election](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/raft-term.png)

一个 `Follower` 的竞选过程如下：

1.  节点状态由 `Follower` 变为 `Candidate`，同时设置当前的 Term 任期
2.  `Candidate` 节点给自己投 `1` 票，同时向其他所有节点发送 <font color="#dd0000"> 拉票请求 </font>（RequestVote RPC）
3.  `Candidate` 节点等待投票结果，确认下一步的去向：
    - case1：本 Node 赢得选举，节点状态变为 `Leader`
    - case2：有其他 Node 赢得选举，节点状态变为 `Follower`
    - case3：第一轮选举未产生结果，节点状态保持为 `Candidate`

1、case1：本 Node（自己）赢得选举 <br>
只有当一个 `Candidate` 得到 <font color="#dd0000"> 超过半数选票时才能赢得选举 </font>，每一个节点按照先到先得的方式，最多投票给一位 `Candidate`。在 `Candidate` 赢得选举后，自己变为 `Leader`，同时向所有节点发送心跳信息以使其他节点变为 `Follower`，本次选举结束，开始下一任任期

2、case2：其他 Node 赢得选举 <br>
在等待投票结果的过程中，如果 `Candidate` 收到其他节点发送的心跳信息（即 AppendEntries RPC），<font color="#dd0000"> 并检查此心跳信息中的任期不比自己小（这一点很重要）</font>，则自己变为 `Follower`，听从新上任的 `Leader` 的指挥

3、本轮选举未产生结果 <br>
极端情况下，如当有多个 `Candidate` 同时竞选时，由于每个人先为自己投一票，导致没有任何一个人的选票数量过半。当这种情况出现时，每位 `Candidate` 都开始准备下一任竞选：将 `TERM+=1`，同时再次发送拉票请求。为了防止出现长时间选不出新 `Leader` 的情况，Raft 采用了两个方法来尽可能规避：

- `Follower` 认为 `Leader` 不可用的超时时间（Election Timeout，即选举超时时间）是随机值，防止了所有的 `Follower` 都在同一时刻发现 `Leader` 不可用的情况，从而让先发现的 `Follower` 顺利成为 `Candidate`，继而完成剩下的选举过程并当选
- 即使出现多个 `Candidate` 同时竞选的情况，再发送拉票请求时，也有一段随机的延迟，来确保各个 `Candidate` 不是同时发送拉票请求

## 0x03 Log Replication

## 0x04 参考

- [1620 秒入门 Raft](https://zhuanlan.zhihu.com/p/27910576)
- [分布式一致性与共识算法](https://www.infoq.cn/article/urx2mw3funeb7bdstqxk)
- [Raft 协议详解](https://zhuanlan.zhihu.com/p/27207160)
- [Raft 算法动态演示](http://www.kailing.pub/raft/index.html)
- [Raft 动画演示中文版：代码仓库](https://github.com/klboke/raft-animation)
- [高可用分布式存储 etcd 的实现原理](https://draveness.me/etcd-introduction/)
- [Raft 算法原理](https://www.codedump.info/post/20180921-raft/#%E9%9B%86%E7%BE%A4%E6%88%90%E5%91%98%E5%8F%98%E6%9B%B4)
- [About C implementation of the Raft Consensus protocol, BSD licensed](https://github.com/willemt/raft)
- [烧脑系列：手撕 hashicorp/raft 算法，超长文](https://mp.weixin.qq.com/s?__biz=MzAxMTA4Njc0OQ==&mid=2651442047&idx=4&sn=08af250a96ad311bc3edfe13636c73fa&chksm=80bb158db7cc9c9bee41780ef3b947eff1b91b0b0fb72037218977f58a07d3b11941ba1445e4&scene=178#rd)
- [Etcd Raft库的工程化实现](https://www.codedump.info/post/20210515-raft/)
