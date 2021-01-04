---
layout:     post
title:      Raft 协议分析与实战（理论篇）
subtitle:   Raft：一种更容易理解的共识算法
date:       2020-01-10
author:     pandaychen
catalog:    true
tags:
    - 分布式理论
    - Raft
    - Etcd
---

##  0x00    前言
本文是对于 Raft 协议原理的一些总结。原文 [在此](https://raft.github.io/raft.pdf)

> Raft implements consensus by first electing a distinguished leader, then giving the leader complete responsibility for managing the replicated log. The leader accepts log entries from clients, replicates them on other servers, and tells servers when it is safe to apply log entries to their state machines. A leader can fail or become disconnected from the other servers, in which case a new leader is elected.

Raft 的核心逻辑是：先选举出 Leader，Leader 完全负责 Replicated-Log 的管理。Leader 负责接受所有客户端更新请求，然后复制到 Follower 节点，并在安全的时候执行这些请求。若 Leader 故障，Followes 会重新选举出新的 Leader。Raft 是满足 CAP 定理中的 C 和 P 特性，属于强一致性的分布式协议。

Raft 的两个核心问题：
1.  Leader Election：领导人选举
2.  Log Replication：日志复制（备份）

Raft 的优点如下：
1.  高可用：Raft 协议中，$N$ 个节点，系统容忍 $\frac{N}{2}$ 个节点的故障，通常 $N==5$。选举和日志同步都只需要大多数的节点 $\frac{N}{2}+1$ 正常互联即可，即使 Leader 故障，在选举 Term 超时到期后，集群自动选举新 Leader（不可用时间非常小）

2.  强一致：虽然 Raft 协议中所有节点的数据非实时一致，但 Raft 算法保证 Leader 节点的数据最全，同时所有修改请求（写入 / 更新 / 删除）都由 Leader 处理，<font color="#dd0000"> 从用户角度看是指永远可以读到最新的写成功的数据，而从服务内部来看指的是所有的存活节点的 State Machine 中的数据都保持一致 </font>
3.  高可靠：Raft 算法保证了 Committed 的日志不会被修改，S0tate Matchine 只应用 Committed 的日志，所以当客户端收到请求成功即代表数据不再改变。Committed 日志在大多数节点上冗余存储，少于一半的磁盘故障数据不会丢失

4.  高性能：与必须将数据写到所有节点才能返回客户端成功的算法对比，Raft 算法只需要大多数节点成功即可，少量节点处理缓慢不会延缓整体系统运行

##  0x01    Raft 协议的状态机

####    Raft 状态机节点

在 Raft 算法中，集群的节点有以下 `3` 种状态：
-   `Leader`：领导者，一个 Raft 集群里只能存在一个 `Leader`
-   `Follower`：跟随者，一个客户端的修改数据请求如果发送到 `Follower` 时，会首先由 `Follower` 重定向到 `Leader` 上
-   `Candidate`：选举者，当一个节点切换到此状态时，将开始进行一次新的选举

每开始一次新的选举时，称为一个任期（TERM）。每个 TERM 都有一个严格递增的整数值与之关联，这个特性后面会用到


####    状态机切换

节点的状态切换状态机如下图所示。
![raft-state](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/raft3.jpg)

1.  Start 起始状态，各个节点刚启动的时候自动进入 `Follower` 状态
2.  Times out, starts election：`Follower` 在启动之后，将开启一个选举超时定时器，当这个定时器到期时，`Follower` 将切换到 `Candidate` 状态发起选举（每个 `Follower` 都投自己一票，成为 `Candidate`）
3.  Times out, new election：进入 `Candidate` 状态之后就开始选举，但是如果在下一次选举超时之前，还没有选出一个新的 `Leader`，那么还会保持在 `Candidate` 状态重新开始一次新选举
4.  Receives votes from majority of servers：当 `Candidate` 状态的节点，收到了超过半数的节点的选票，那么其将切换状态成为新的 `Leader`，同时向其他节点广播，其他节点成为 `Follower`
4.  Discovers current leader or new term：处于 `Candidate` 状态的节点，如果收到了来自 `Leader` 的消息，或者更高任期号（Term）的消息，表示集群中已经有 `Leader` 了，将切换回到 `Follower` 状态
5.  Discovers server with higher term：`Leader` 状态下如果收到来自更高任期号的消息，将切换到 `Follower` 状态。这种情况大多数发生在有网络分区的状态下（网络分区又恢复）

Raft 节点间通过 RPC 请求来互相通信，主要有以下两类 RPC 请求：
1.  Request-Vote RPC：用于 `Candidate` 状态的节点进行选举用途
2.  Append-Entries RPC：由 `Leader` 节点向其他节点复制日志数据以及同步心跳数据

####    任期 Term

####    选举定时器
在 Raft 中有两个 Timeout 定时器控制着选主 Election 的进行：
1.  选举超时时间（Election Timeout）：意思是 `Follower` 要等待成为 `Candidate` 的时间（要成为 `Candidate` 后才可以投票选举），通常 Election Timeout 定时器为 `150ms~300ms`，这个时间结束之后 `Follower` 变成 `Candidate` 开始选举过程。首先是自己对自己投票（`+1`），然后向其他节点请求投票（选票），如果接收节点在收到投票请求时还没有参与过投票，那么它会响应本次请求，并把票投给这个请求投票的 `Candidate`，然后重置自身的 Election Timeout 定时器，一旦一个 `Candidate` 拥有所有节点中的大多数投票，则此节点变成一个 `Leader`
2.  心跳超时时间（Heartbeat Timeout）：当一个节点从 `Candidate` 变为 `Leader` 时，此节点开始向其他 `Follower` 发送 Append Entries，这些消息发送的频率是通过 Heartbeat Timeout 配置，`Follower` 会响应每条的 Append Entry，整个选举会一直进行直到 `Follower` 停止接受 heartbeat 并且变成 `Candidate` 开始下一轮选举（即假设此时 `Leader` 故障或者丢失了，`Follower` 检测到心跳超时（在此期间没有收到 `Leader` 发送的 Append Entry）后，再等待自身选举超时后发起新一轮选举）

##  0x02    Leader Election

##  0x03    Log Replication

##  0x04    参考
-   [分布式一致性与共识算法](https://www.infoq.cn/article/urx2mw3funeb7bdstqxk)
-   [Raft 协议详解](https://zhuanlan.zhihu.com/p/27207160)
-   [Raft 算法动态演示](http://www.kailing.pub/raft/index.html)
-   [Raft 动画演示中文版：代码仓库](https://github.com/klboke/raft-animation)
-   [高可用分布式存储 etcd 的实现原理](https://draveness.me/etcd-introduction/)
-   [Raft 算法原理](https://www.codedump.info/post/20180921-raft/#%E9%9B%86%E7%BE%A4%E6%88%90%E5%91%98%E5%8F%98%E6%9B%B4)
-   [About C implementation of the Raft Consensus protocol, BSD licensed](https://github.com/willemt/raft)
-   [烧脑系列：手撕 hashicorp/raft 算法，超长文](https://mp.weixin.qq.com/s?__biz=MzAxMTA4Njc0OQ==&mid=2651442047&idx=4&sn=08af250a96ad311bc3edfe13636c73fa&chksm=80bb158db7cc9c9bee41780ef3b947eff1b91b0b0fb72037218977f58a07d3b11941ba1445e4&scene=178#rd)