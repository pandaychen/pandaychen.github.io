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
1.  高可用：Raft 协议中，$$N$$ 个节点，系统容忍 $$\frac{N}{2}$$ 个节点的故障，通常 $$N==5$$。选举和日志同步都只需要大多数的节点 $$\frac{N}{2}+1$$ 正常互联即可，即使 Leader 故障，在选举 Term 超时到期后，集群自动选举新 Leader（不可用时间非常小）

2.  强一致：虽然 Raft 协议中所有节点的数据非实时一致，但 Raft 算法保证 Leader 节点的数据最全，同时所有修改请求（写入 / 更新 / 删除）都由 Leader 处理，<font color="#dd0000"> 在客户端角度看是强一致性的 </font>
3.  高可靠：Raft 算法保证了 Committed 的日志不会被修改，S0tate Matchine 只应用 Committed 的日志，所以当客户端收到请求成功即代表数据不再改变。Committed 日志在大多数节点上冗余存储，少于一半的磁盘故障数据不会丢失

4.  高性能：与必须将数据写到所有节点才能返回客户端成功的算法对比，Raft 算法只需要大多数节点成功即可，少量节点处理缓慢不会延缓整体系统运行

##  0x01    Raft协议的节点

##  0x02    Leader Election

##  0x03    Log Replication

##  0x04    参考
-   [Raft 算法动态演示](http://www.kailing.pub/raft/index.html)
-   [Raft 动画演示中文版：代码仓库](https://github.com/klboke/raft-animation)
-   [高可用分布式存储 etcd 的实现原理](https://draveness.me/etcd-introduction/)
-   [Raft 算法原理](https://www.codedump.info/post/20180921-raft/#%E9%9B%86%E7%BE%A4%E6%88%90%E5%91%98%E5%8F%98%E6%9B%B4)
-   [About C implementation of the Raft Consensus protocol, BSD licensed](https://github.com/willemt/raft)
-   [烧脑系列：手撕 hashicorp/raft 算法，超长文](https://mp.weixin.qq.com/s?__biz=MzAxMTA4Njc0OQ==&mid=2651442047&idx=4&sn=08af250a96ad311bc3edfe13636c73fa&chksm=80bb158db7cc9c9bee41780ef3b947eff1b91b0b0fb72037218977f58a07d3b11941ba1445e4&scene=178#rd)