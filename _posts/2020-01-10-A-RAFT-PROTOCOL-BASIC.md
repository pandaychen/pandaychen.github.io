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

1.  Leader Election：领导人选举（选主），主节点宕机后必须选择一个节点成为新的主节点（有且仅有一个`Leader`节点，如果`Leader`宕机，通过选举机制选出新的`Leader`）
2.  Log Replication：日志复制（备份），主节点必须接受客户端指令日志，并强制复制到其他节点（`Leader`从客户端接收数据更新/删除请求，然后日志复制到`Follower`节点，从而保证集群数据的一致性）
3.  Safty：安全性，复制集群状态机必须一致。相同索引的日志指令一致，顺序一致（通过安全性原则来处理一些特殊case，保证Raft算法的完备性）

所以，Raft算法核心流程可以归纳为：

- 首先选出`Leader`，`Leader`节点负责接收外部的数据更新/删除请求
- 然后日志复制到其他`Follower`节点，同时通过安全性的准则来保证整个日志复制的一致性
- 如果遇到`Leader`故障，`Followers`会重新发起选举出新的`Leader`

Raft 的优点如下：

1.  高可用：Raft 协议中，$N$ 个节点，系统容忍 $\frac{N}{2}$ 个节点的故障，通常 $N==5$。选举和日志同步都只需要大多数的节点 $\frac{N}{2}+1$ 正常互联即可，即使 `Leader` 故障，在选举 Term 超时到期后，集群自动选举新 `Leader`（不可用时间非常小）

2.  强一致：虽然 Raft 协议中所有节点的数据非实时一致，但 Raft 算法保证 `Leader` 节点的数据最全，同时所有修改请求（写入 / 更新 / 删除）都由 `Leader` 处理，<font color="#dd0000"> 从用户角度看是指永远可以读到最新的写成功的数据，而从服务内部来看指的是所有的存活节点的 State Machine 中的数据都保持一致 </font>
3.  高可靠：Raft 算法保证了 Committed 的日志不会被修改，S0tate Matchine 只应用 Committed 的日志，所以当客户端收到请求成功即代表数据不再改变。Committed 日志在大多数节点上冗余存储，少于一半的磁盘故障数据不会丢失

4.  高性能：与必须将数据写到所有节点才能返回客户端成功的算法对比，Raft 算法只需要大多数节点成功即可，少量节点处理缓慢不会延缓整体系统运行

####  复制状态机
复制状态机用于解决分布式系统中的各种容错问题，通常通过复制日志实现，每个服务器存储一个包含一系列命令的日志，其状态机按顺序执行日志中的命令。每个状态机处理日志的顺序相同，因此也能够得到相同的输出序列。

一致性算法的工作就是保证复制日志的一致性。每台服务器上的一致性模块接收来自客户端的命令，并将它们添加到其日志中。它与其他服务器上的一致性模块通信，以确保每个日志最终以相同的顺序包含相同的命令，即使有一些服务器失败。一旦命令被正确复制，每个服务器上的状态机按日志顺序处理它们，并将输出返回给客户端。这样就形成了高可用的复制状态机。

![replication-state](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/raft-replication-state-machine.png)


####  通信方式
Raft使用RPC进行通信，主要有三种类型RPC：

1.  请求投票（RequestVote）RPC， 由 `Candidate` 在选举期间发起
2.  追加条目（AppendEntries）RPC， 由 `Leader` 发起，用来复制日志和提供一种心跳机制
3.  重试RPC，当服务器没有及时的收到 RPC 的响应时，会进行重试

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
- Raft算法把时间轴划分为不同任期Term
- 每个Term开始于选举阶段，一般由选举阶段和领导阶段组成。
- 每个任期Term都有自己的编号TermId，该编号全局唯一且连续单调递增

每个任期开始都会进行领导选举（Leader Election）。如果**选举成功**，则进入维持任务Term阶段，此时`Leader`负责接收客户端请求并，负责复制日志。`Leader`和所有`Follower`都保持通信，如果`Follower`发现通信超时（网络问题、或者`Leader`宕机），会触发TermId递增并发起新的选举。如果选举成功，则进入新的任期。如果选举失败（**Hint：选举失败触发的条件是什么？**），没选出`Leader`，则当前任期很快结束，TermId递增，然后重新发起选举直到成功。

如下图，Term `N`选举成功，Term `N+1`和Term `N+2`选举失败，Term `N+3`重新选举成功

![term](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/raft-term.png)

![term](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/raft-term.png)

####  任期的意义：逻辑时钟
在Raft协议中，任期充当逻辑时钟，服务器节点可以通过任期发现一些过期的信息，比如过时的 `Leader`。任期单调递增，**服务器之间通信的时候会交换当前任期号；如果一个服务器的当前任期号比其他的小，该服务器会将自己的任期号更新为较大的那个值**。如果一个 `Candidate` 或 `Leader` 发现自己的任期号过期了，它会立即回到 `Follower` 状态。如果一个节点接收到一个包含过期的任期号的请求，它会直接拒绝这个请求

#### 选举定时器

在 Raft 中有两个 Timeout 定时器控制着选主 Election 的进行：

1.  选举超时时间（Election Timeout）：意思是 `Follower` 要等待成为 `Candidate` 的时间（要成为 `Candidate` 后才可以投票选举），通常 Election Timeout 定时器为 `150ms~300ms`，这个时间结束之后 `Follower` 变成 `Candidate` 开始选举过程。首先是自己对自己投票（`+1`），然后向其他节点请求投票（选票），如果接收节点在收到投票请求时还没有参与过投票，那么它会响应本次请求，并把票投给这个请求投票的 `Candidate`，然后重置自身的 Election Timeout 定时器，一旦一个 `Candidate` 拥有所有节点中的大多数投票，则此节点变成一个 `Leader`
2.  心跳超时时间（Heartbeat Timeout）：当一个节点从 `Candidate` 变为 `Leader` 时，此节点开始向其他 `Follower` 发送 Append Entries，这些消息发送的频率是通过 Heartbeat Timeout 配置，`Follower` 会响应每条的 Append Entry，整个选举会一直进行直到 `Follower` 停止接受 heartbeat 并且变成 `Candidate` 开始下一轮选举（即假设此时 `Leader` 故障或者丢失了，`Follower` 检测到心跳超时（在此期间没有收到 `Leader` 发送的 Append Entry）后，再等待自身选举超时后发起新一轮选举）

简言之：
- 随机超时时间： `Follower`节点每次收到`Leader`的心跳请求后，会设置一个随机的，区间位于`[150ms, 300ms)`的超时时间。如果超过超时时间，还没有收到`Leader`的下一条请求，则认为`Leader`过期/故障了
- `Leader`心跳（`+1s`）： `Leader`在当选期间，会以一定时间间隔向其他节点发送心跳请求，以维护自己的`Leader`地位

## 0x02 Leader Election

当 `Leader` 节点由于异常（宕机、网络故障等）无法继续提供服务时，可以认为它结束了本轮任期（`TERM=N`），需要开始新一轮的选举（Election），而新的 `Leader` 当然要从 `Follower` 中产生，开始新一轮的任期（`TERM=N+1`）。（从 `Follower` 节点的角度来看，当它接收不到 `Leader` 节点的心跳即心跳超时时间触发时，可认为 `Leader` 已经丢失，便进入新一轮的选举流程）  

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

3、本轮选举未产生结果（一段时间之后没有任何获胜者） <br>
极端情况下，如当有多个 `Candidate` 同时竞选时，由于每个人先为自己投一票，导致没有任何一个人的选票数量过半。当这种情况出现时，每位 `Candidate` 都开始准备下一任竞选：将 `TERM+=1`，同时再次发送拉票请求。为了防止出现长时间选不出新 `Leader` 的情况，Raft 采用了两个方法来尽可能规避该情况发生（防止第`3`种情况无限重复，使得没有`Leader`被选举出来）：

- `Follower` 认为 `Leader` 不可用的超时时间（Election Timeout，即选举超时时间）是随机值，防止了所有的 `Follower` 都在同一时刻发现 `Leader` 不可用的情况，从而让先发现的 `Follower` 顺利成为 `Candidate`，继而完成剩下的选举过程并当选（使用随机化选举超时时间，从一个固定的区间如`150-300ms`随机选择，避免同时出现多个`Candidate`）
- 即使出现多个 `Candidate` 同时竞选的情况，再发送拉票请求时，也有一段随机的延迟，来确保各个 `Candidate` 不是同时发送拉票请求（`Candidate`等待超时的时间随机：`Candidate` 在开始一次选举的时候会重置一个随机的选举超时时间，然后一直等待直到选举超时，这样减小了在新的选举中再次发生选票瓜分情况的可能性）

![vote](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/raft-vote-for-leader.png)

## 0x03 Leader Election（节点视角）

集群中每个节点只能处于`Leader`、`Follower`和`Candidate`三种状态的一种，从节点视角看选举流程如下：

1.  `Follower`（从节点）
- 节点默认是`Follower`
- 如果刚刚开始或和`Leader`通信超时时（原因下述），`Follower`会发起选举，变成`Candidate`，然后去竞选`Leader`
- 如果收到其他`Candidate`的竞选投票请求，按照先来先得&&每个任期只能投票一次的投票原则投票

2.  `Candidate`（候选者）
- `Follower`发起选举后就变为`Candidate`，会向其他节点拉选票。`Candidate`的票会投给自己，所以不会向其他节点投票
- 如果获得超过半数的投票，`Candidate`变成`Leader`，然后马上和其他节点通信，表明自己的`Leader`的地位
- 如果选举超时，重新发起选举
- 如果遇到更高任期Term的`Leader`的通信请求，转化为`Follower`

3.  `Leader`（主节点）
- 节点成为`Leader`节点后，此时可以接受客户端的数据请求，负责日志同步
- 如果遇到更高任期Term的`Candidate`的通信请求，这说明`Candidate`正在竞选`Leader`，此时之前任期的`Leader`转化为`Follower`，且完成投票；
- 如果遇到更高任期Term的`Leader`的通信请求，这说明已经选举成功新的`Leader`，此时之前任期的`Leader`转化为`Follower`

![vote-node-view](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/raft-vote-node-view.jpg)

那么，什么时候选举重新发生呢？
1.  `Leader`离线，故障
2.  `Follower`发现通信超时（收不到`Leader`的应答包）

在选举完成后，`Leader`和所有`Follower`都保持通信，如果`Follower`发现通信超时，任期ID（TermId）递增并发起新的选举。如果选举成功，则进入新的任期。如果选举失败，TermId递增，然后重新发起选举直到成功。

####  触发重新选举

上面说了，`Leader`在任期内会周期性向其他`Follower`节点发送心跳来维持地位，`Follower`如果发现心跳超时，就认为`Leader`节点宕机或不存在。随机等待一定时间后，`Follower`会发起选举，变成`Candidate`，然后去竞选`Leader`。重新选举结果有三种情况：

1.  获取超过半数投票，赢得选举
  - 当`Candidate`获得超过半数（**如何判定超过了半数？**）的投票时，代表自己赢得了选举，且转化为`Leader`。此时，它会马上向其他节点发送请求，从而确认自己的`Leader`地位，从而阻止新一轮的选举
  - 投票原则：当多个`Candidate`竞选`Leader`时
    - 一个任期内，`Follower`只会投票一次票，且投票先来显得
    - `Candidate`存储的日志至少要和`Follower`一样新（安全性准则），否则拒绝投票

2.  投票未超过半数，选举失败
  - 当`Candidate`没有获得超过半数的投票时，说明多个`Candidate`竞争投票导致过于分散，或者出现了丢包现象。此时，认为当期任期选举失败，任期TermId+1，然后发起新一轮选举；
  - 上述机制可能出现多个`Candidate`竞争投票，导致每个`Candidate`一直得不到超过半数的票，最终导致无限选举投票循环；
  - 投票分散问题解决： Raft会给每个`Candidate`在固定时间内随机确认一个超时时间（一般为`150-300ms`）。这么做可以尽量避免新的一次选举出现多个`Candidate`竞争投票的现象

3.  收到其他`Leader`通信请求
  - 如果`Candidate`收到其他声称自己是`Leader`的请求的时候，通过任期TermId来判断是否处理；
  - 如果请求的任期TermId不小于`Candidate`当前任期TermId，那么`Candidate`会承认该`Leader`的合法地位并转化为`Follower`；
  - 否则，拒绝这次请求，并继续保持`Candidate`

####  脑裂问题
脑裂问题（Brain Split），即任一任期Term内有超过一个`Leader`被选出（Raft要求在一个集群中任何时刻只能有一个`Leader`），这是非常严重的问题，会导致数据的覆盖丢失。

出现的场景，如下面的Raft集群，某个时刻出现网络分区，集群被隔开成两个网络分区，在不同的网络分区里会因为无法接收到原来的`Leader`发出的心跳而超时选主，这样就会造成多`Leader`现象。

![brain-split](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/brain-split.png)


在网络分区`1`和`2`中，出现了两个`Leader`：A和D，假设此时要更新分区`2`的值,因为分区`2`无法得到集群中的大多数节点的ACK，会导致复制失败。而网络分区`1`会成功,因为分区`1`中的节点更多,`Leader` A能得到大多数回应。

当网络恢复的时候，集群网络恢复，不再是双分区，Raft会有如下操作：
1.  任期安全性约束：`Leader` D 发现自己的Term小于`Leader` A，会自动下台（Step Down）成为`Follower`，`Leader` A保持不变依旧是集群中的主`Leader`角色（因为`Leader` A是重新选举出的，TermId至少会加`1`）
2.  分区中的所有节点会回滚自己的数据日志,并匹配新`Leader`的日志Log，然后实现同步提交更新自身的值。通知旧`Leader` 节点D也会主动匹配主`Leader`节点A的最新值，并加入到`Follower`中
3.  最终集群达到整体一致，集群存在唯一`Leader`（节点A）

小结下，领导选举流程通过若干的投票原则，保证一次选举有且仅可能最多选出一个`Leader`，从而解决了脑裂问题


##  0x04  宕机（选举完成后）

####  `Leader`宕机
1.  每当`Leader`对所有的`Follower`发出Append Entries的时候，`Follower`会有一个随机的超时时间，如果超时TTL内收到了`Leader`的请求就会重置超时TTL；如果超时后`Follower`仍然没有收到 `Leader`的心跳，`Follower`会认为 `Leader` 可能已经离线，此时第一个超时的`Follower`会发起投票，这个时候它依然会向宕机的原`Leader`发出Reuest Vote（原`Leader`不会回复）。当收到集群超过一半的节点的RequestVote reply后,此时的`Follower`会成为`Leader`

2.  若稍后宕机的`Leader`恢复正常之后，重新加入到Raft集群，初始化的角色是`Follower`（非leader）

####  `Follower`宕机
`Follower`宕机对整个集群影响不大（可用个数内），影响是`Leader`发出的Append Entries无法被收到，但是`Leader`还会继续一直发送，直到`Follower`恢复正常。Raft协议会保证发送AppendEntries request的rpc消息是幂等的：即如果`Follower`已经接受到了消息，但是`Leader`又让它再次接受相同的消息，`Follower`会直接忽略

## 0x05 Log Replication：日志同步（复制）
Raft集群建立完成后，`Leader`接收所有客户端请求，然后转化为log复制命令，发送通知其他节点完成日志复制请求。每个日志复制请求包括状态机命令 & 任期号，同时还有前一个日志的任期号和日志索引。状态机命令表示客户端请求的数据操作指令，任期号表示`Leader`的当前任期。

####  基础概念

日志定义：客户端的每一个请求都包含一条将被复制状态机执行的指令。`Leader` 把该指令作为一个新的条目追加到日志中去，然后并行的发起 AppendEntries RPC，同步到其他副本

安全复制：当有超过一半的副本复制好了该条目，便认为已经安全的复制，`Leader`接着会将条目应用到其状态机中，并将结果返回给客户端。若遇到网络故障或丢失包，`Leader` 会不断地重试 AppendEntries RPC（即使已经回复了客户端）直到所有的 `Follower` 最终都存储了所有的日志条目

日志索引：每个日志条目存储一条状态机指令和 `Leader` 收到该指令时的任期号，同时每个日志条目都有一个整数索引值来表明它在日志中的位置

日志提交：一旦创建该日志条目的 `Leader` 将它复制到过半的服务器上，该日志条目就会被提交，Raft 算法保证所有已提交的日志条目都是持久化的并且最终会被所有可用的状态机执行。不仅如此，`Leader` 日志中该日志条目之前的所有日志条目也都是已经提交的条目。`Leader`会保存当前已提交最大的日志索引，未来的所有 AppendEntries RPC 都会包含该索引，这样其他的服务器才能最终知道哪些日志条目需要被提交。`Follower` 一旦知道某个日志条目已经被提交就会将该日志条目按顺序应用到自己的本地状态机中

日志由有序编号的日志条目组成。每个日志条目包含它被创建时的任期号（每个方块中的数字），并且包含用于状态机执行的命令。如果一个条目能够被状态机安全执行，就被认为可以提交了，如下图：
![figure](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/Figure-6-Logs.png)

Leader 决定什么时候将日志条目应用到状态机是安全的；这种条目被称为是已提交的（Committed）。Raft 保证可已提交的日志条目是持久化的，并且最终会被所有可用的状态机执行。一旦被 `Leader` 创建的条目已经复制到了大多数的服务器上，这个条目就称为已提交的（上图中的`7`号条目）。`Leader` 日志中之前的条目都是已提交的，包括由之前的 `Leader` 创建的条目。

**Leader 跟踪记录它所知道的已提交的条目的最大索引值，并且这个索引值会包含在之后的 AppendEntries RPC 中（包括心跳中），为的是让其他服务器都知道这个条目已经提交。一旦一个 Follower 知道了一个日志条目已经是已提交的，它会将该条目应用至本地的状态机（按照日志顺序）**


日志同步的概念：服务器接收客户的数据更新/删除请求，这些请求会落地为命令日志。只要输入状态机的日志命令相同，状态机的执行结果就相同。Raft中是`Leader`发出日志同步请求，`Follower`接收并同步日志，最终保证整个集群的日志一致性。

![log-replication](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/raft-master-worker.jpg)

在了解日志同步前，先了解复制状态机的概念。**复制状态机（Replicated state machines）是指不同节点从相同的初始状态出发，执行相同顺序的输入指令集后，会得到相同的结束状态。**

分布式一致性算法的实现是基于复制状态机的。在Raft算法中，节点初始化后具有相同初始状态。为了提供相同的输入指令集这个条件，Raft将一个客户端请求（command）封装到一个Log Entry中。`Leader`负责将这些Log Entries 复制到所有的`Follower`节点，然后节点按照相同的顺序应用commands，从而达到最终的一致状态。

当`Leader`收到客户端的写请求，到将执行结果返回给客户端的这个过程，从`Leader`视角来看经历了以下步骤：

1.  本地追加日志信息，`Leader` 节点接收到的数据处于未提交状态（uncommitted）
2.  并行发出 AppendEntries RPC请求（`Leader` 节点会并发地向所有`Follower`节点复制数据并等待接收响应ACK）
3.  等待大多数`Follower`的回应，收到超过半数节点的成功提交回应，代表该日志被成功复制到了大多数节点中（committed）；一旦向 Client 发出数据接收 Ack 响应后，表明此时数据状态进入已提交状态，`Leader` 节点再向 `Follower` 节点发通知告知该数据状态已提交
4.  在状态机上执行entry command。 既将该日志应用到状态机，真正影响到节点状态（applied）；`Follower`开始commit自己的数据,此时Raft集群达到主节点和从节点的一致
5.  回应Client 执行结果
6.  确认`Follower`也执行了这条command。如果`Follower`崩溃、运行缓慢或者网络丢包，`Leader`将无限期地重试AppendEntries RPC，直到所有`Followers`应用了所有日志条目

从`Follower`视角来看经历了如下步骤，`Follower`收到日志复制请求的处理流程

1.  `Follower`会使用前一个日志的任期号和日志索引来对比自己收到的数据
  - 如果相同，接收复制请求，回复ok
  - 否则回拒绝复制当前日志，回复error
2.  `Leader`收到拒绝复制的回复后，继续发送节点日志复制请求，不过这次会带上更前面的一个日志任期号和索引
3.  **如此循环往复，直到找到一个共同的任期号&日志索引**。此时`Follower`从这个索引值开始复制，最终和`Leader`节点日志保持一致
4.  日志复制过程中，`Leader`会无限重试直到成功。如果超过半数的节点复制日志成功，就可以任务当前数据请求达成了共识，即日志可以commit提交了

日志复制（Log Replication）有两个特点：

- 如果在不同日志中的两个条目有着相同日志索引index和任期号termid，则所存储的命令是相同的，这点是由`Leader`来保证的
- 如果在不同日志中的两个条目有着相同日志索引index和任期号termid，则它们之间所有条目完全一样，这点是由日志复制的规则来保证的

####  Logs
Logs由顺序排列的Log Entry组成 ，每个Log Entry包含command和产生该Log Entry时的Leader Term。从上图可知，五个节点的日志并不完全一致，Raft算法为了保证高可用，并不是强一致性，而是最终一致性，`Leader`会不断尝试给`Follower`发Log Entries，直到所有节点的Log Entries都相同。
- Index：日志索引，保证唯一性
- TermId：任期Id，标识当前操作在哪个任期内
- Action：标识任期内的数据操作

在前面的流程中，`Leader`只需要日志被复制到大多数节点即可向客户端返回，而一旦向客户端返回成功消息，那么系统就必须保证log在任何异常的情况下都不会发生回滚。

####  日志复制的例子

![log-replication](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/log-replication.jpg)

从图中看，LogIndex `1-4`的日志已经完成同步，LogIndex `5`正在同步，LogIndex `6`还未开始同步，下一小节会基于官方文档完整的描述日志复制的过程

####  日志不一致
日志不一致问题：在正常情况下，`Leader`和`Follower`的日志复制能够保证整个集群的一致性，但是遇到`Leader`宕机时，`Leader`和`Follower`日志可能出现了不一致，一般而言此时`Follower`相比`Leader`缺少部分日志。

为了解决数据不一致性，Raft算法规定`Follower`强制复制`Leader`节点的日志，即`Follower`不一致日志都会被`Leader`的日志覆盖，最终`Follower`和`Leader`保持一致。

这个强制复制指的是即从前向后寻找`Follower`和`Leader`第一个公共LogIndex的位置，然后从这个位置开始，`Follower`强制复制`Leader`的日志。这里的复制动作受到下面的Raft安全性规则约束（一切以数据最终一致性为前提）

####  logs 复制时异常问题及解决
在进行一致性复制的过程中,假如出现了异常情况，raft都是如何处理的呢？
1.  写入数据（请求）到达 `Leader` 节点前，该阶段 `Leader` 挂掉不影响一致性（触发重新选举，重新选举出`Leader`）
2.  写入请求到达 `Leader` 节点，但未复制到 `Follower` 节点。该阶段 `Leader` 挂掉，数据属于未提交状态，Client 不会收到最终的 Ack 会认为超时失败、可安全发起重试
3.  写入请求到达 `Leader` 节点，成功复制到 `Follower` 所有节点，但 `Follower` 还未向 `Leader` 响应接收。若该阶段 `Leader` 挂掉，虽然数据在 `Follower` 节点处于未提交状态（uncommitted），但是数据是保持一致的。重新选出 `Leader` 后可完成数据提交
4.  写入数据到达 `Leader` 节点，成功复制到 `Follower` 的部分节点，但这部分 `Follower` 节点还未向 `Leader` 响应接收。这个阶段 `Leader` 挂掉，数据在 `Follower` 节点处于 未提交状态（uncommitted）且不一致。由于Raft 协议要求投票只能投给拥有 最新数据 的节点。所以拥有最新数据的节点会被选为 Leader，然后再 强制同步数据 到其他 Follower，保证 数据不会丢失并 最终一致。
5.  写入数据到达 `Leader` 节点，成功复制到 `Follower` 所有或多数节点，数据在 `Leader` 处于已提交状态，但在 `Follower` 处于未提交状态。若该阶段 `Leader` 挂掉，重新选出新的 `Leader` 后的处理流程和阶段 `3` 一样
6.  写入数据到达 `Leader` 节点，成功复制到 `Follower` 所有或多数节点，数据在所有节点都处于已提交状态，但还未响应 Client。这个阶段 `Leader` 挂掉，集群内部数据其实已经是 一致的，Client 重复重试基于幂等策略对一致性无影响

##  0x05  官方文档：再看logs复制的细节
本小节摘抄于原文：[5.3 日志复制](https://knowledge-sharing.gitbooks.io/raft/content/chapter5/5-3.html)

####  复制故障说明
正常操作期间，`Leader` 和 `Follower` 的日志保持一致，方格数字表示任期。所以 AppendEntries RPC 的一致性检查从来不会失败。但也有可能出现下图的故障：

![loss](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/Figure-7-Log-states.png)

当一个 `Leader` 成功当选时（最上面那行），`Follower` 的状态机可能是（a-f）中的任何情况。`Follower` 可能会缺少一些日志条目（如a、b），可能会有一些未被提交的日志条目（如c、d），或者两种情况都存在（e、f）。例如，场景 f 可能这样发生，f 对应的服务器在任期 `2` 的时候被选举为 `Leader` ，追加了一些日志条目到自己的日志中，一条都还没提交（commit）就宕机了；该服务器很快重启，在任期 `3` 重新被选为 `Leader`，又追加了一些日志条目到自己的日志中；在这些任期 `2` 和任期 `3` 中的日志都还没被提交之前，该服务器又宕机了，并且在接下来的几个任期里一直处于宕机状态。那么，新任`Leader`该如何同步日志呢？

在Raft中，`Leader` 通过强制 `Follower` 复制它的日志来解决不一致的问题。这意味着 `Follower` 中跟 `Leader` 冲突的日志条目会被 `Leader` 的日志条目覆盖。

- Case1：要使得 `Follower` 的日志跟自己一致，`Leader` 必须找到两者达成一致的最大的日志条目（索引最大），删除 `Follower` 日志中从那个点之后的所有日志条目，并且将自己从那个点之后的所有日志条目发送给 `follower`
- Case2：选举时，`Leader` 针对每一个 `Follower` 都维护了一个 `nextIndex`，表示 `Leader` 要发送给 `Follower` 的下一个日志条目的索引。当选出一个新 `Leader` 时，该 `Leader` 将所有 `nextIndex` 的值都初始化为自己最后一个日志条目的 index 加`1`（上图中的 `11`）。如果 `Follower` 的日志和 `Leader` 的不一致，那么下一次 AppendEntries RPC 中的一致性检查就会失败。在被 `Follower` 拒绝之后，`Leader` 就会减小 `nextIndex` 值并重试 AppendEntries RPC 。最终 nextIndex 会在某个位置使得 `Leader` 和 `Follower` 的日志达成一致。此时，AppendEntries RPC 就会成功，将 `Follower` 中跟 `Leader` 冲突的日志条目全部删除然后追加 `Leader` 中的日志条目（如果有需要追加的日志条目的话）。一旦 AppendEntries RPC 成功，`Follower` 的日志就和 `Leader` 一致，并且在该任期接下来的时间里保持一致

优化：如果想要的话，Raft协议可以被优化来减少被拒绝的 AppendEntries RPC 的个数。例如，当拒绝一个 AppendEntries RPC 的请求的时候，`Follower` 可以包含冲突条目的任期号和自己存储的那个任期的第一个 index。借助这些信息，`Leader` 可以跳过那个任期内所有冲突的日志条目来减小 `nextIndex`；这样就变成每个有冲突日志条目的任期需要一个 AppendEntries RPC 而不是每个条目一次。

####  举例：强行覆盖Leader日志的测试用例

##  0x06  状态机
![state-machine](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/raft-state-machine.png)

##  0x07  安全性及约束
为何要引入安全性Safty？因为当前的 Leader election 领导选举 和 Log replication 日志复制并不能保证Raft算法的安全性，在一些特殊情况下，可能导致数据不一致，所以需要引入下面安全性规则对场景进行约束，保证Raft协议的CP特性。


####  选举安全性（Election Safety）

选举安全性主要是为了规避脑裂问题。要求任一任期内最多一个`Leader`被选出。在一个集群中任何时刻只能有一个`Leader`。如果Raft分布式系统中同时有多个`Leader`的问题，被称之为脑裂（brain split）现象，这是会导致数据的覆盖丢失（非常严重）。在Raft协议中，通过`3`点（**严格的投票规则**）保证了这个属性：

- 一个`Follower`节点某一任期内最多只能投一票；且遵循先来先得的原则；而节点B的Term必须比A的新，A才能给B投票
- 只有获得超过半数的投票的节点才（有机会）会成为`Leader`
- `Candidate`存储的日志至少要和`Follower`一样新，`Follower`才可以给`Candidate`投票

####  日志append only（Leader Append-Only）
本约束要求日志只能由`Leader`添加和修改，所有的数据请求都要交给`Leader`节点处理，要求如下：

- `Leader`只能日志追加日志，不能覆盖日志
- `Leader`在某一Term的任一位置只会创建一个Log Entry，且Log Entry是Append-only
- 只有`Leader`的日志项才能被提交，`Follower`不能接收写请求和提交日志
- 只有已经提交的日志项，才能被应用到状态机中
- 选举时限制新`Leader`日志包含所有已提交日志项（这点是为了防止出现）
- 一致性检查，`Leader`在AppendEntries请求中会包含最新Log Entry的前一个log的term和index，如果`Follower`在对应的term index找不到日志，那么就会告知`Leader`日志不一致，然后开始同步自己的日志。同步时，找到日志分叉点，然后将`Leader`后续的日志复制到本地

####  日志匹配特性（Log Matching）
本约束主要是为了**保证日志的唯一性**，要求：
- 如果两个节点上的某个Log Entry的Log Index相同且term相同，那么在该index之前的所有log entry应该都是相同的
- Raft的日志机制提供两个保证，统称为Log Matching Property：
  - 不同机器的日志中如果有两个entry有相同的偏移和term号，那么它们存储相同的指令
  - 如果不同机器上的日志中有两个相同偏移和term号的日志，那么日志中这个entry之前的所有entry保持一致

####  `Leader`完备性（Leader Completeness）
本约束主要要求`Leader`必须具备最新提交日志的属性。Raft规定：只有拥有最新提交日志的`Follower`节点才有资格成为`Leader`节点。具体做法：`Candidate`竞选投票时会携带最新提交日志，`Follower`会用自己的日志和`Candidate`做比较：

- 如果`Follower`的日志比`Candidate`还新，那么拒绝这次投票
- 否则根据前面小节梳理的选举投票规则处理，这样就可以保证只有最新提交节点成为`Leader`

由于日志提交需要超过半数的节点同意，所以针对日志同步落后的`Follower`（还未同步完全部日志，导致落后于其他节点）在竞选`Leader`的时候，肯定拿不到超过半数的票，也只有那些完成同步的才有可能获取超过半数的票的`Follower`成为`Leader`。

那么，如何判断日志是新 OR 旧？判断方式是比较日志项的`TermId`和`Index`值：

- 如果`TermId`不同，选择`TermId`值最大的
- 如果`TermId`相同，选择`Index`值最大的

问题：为何Raft要保证`Leader`完备性的规则？

![leader_completeness](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/leader_completeness.jpg)

1.  假如集群中`Follower4`在LogIndex3 故障宕机，经过一段时间，任期Term3的`Leader`接收并提交了很多日志（假设LogIndex1-5已提交，LogIndex6正在复制中）
2.  此时`Follower4`恢复正常，在没有和`Leader`完成同步日志的情况下，如果`Leader`突然宕机，此时开始领导选举。再假设在Term4 `Follower4`当选`Leader`。根据日志复制的规则，其他`Follower`强制复制`Leader`的日志，那么已经提交却没完成同步的日志将会被强制覆盖掉，这会导致已提交日志被覆盖
3.  所以，通过本约束限制上一步中`Follower4`，让它不可以成为`Leader`

![leader_completeness_2.jpg](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/leader_completeness_2.jpg)


Leader完备性，意义为被选举人必须比自己知道的更多（比较term 、log index）
如果一个Log Entry在某个任期被提交（committed），那么这条日志一定会出现在所有更高term的`Leader`的日志里面。选举人必须比自己知道的更多（比较term，log index）如果日志中含有不同的任期，则选最新的任期的节点；如果最新任期一致，则选最长的日志的那个节点。

####  状态机安全性（State Machine Safety）
本约束确保当前任期日志提交的安全性，保证日志的一致性。在算法中，一个日志被复制到多数节点才算committed，**如果一个log entry在某个任期被提交（committed），那么这条日志一定会出现在所有更高term的`Leader`的日志里面**。

考虑到当前的日志复制规则：

- 当前`Follower`节点强制复制`Leader`节点
- 若以前Term日志复制超过半数节点，在面对当前任期日志的节点比较中，很明显当前任期节点更新，有资格成为`Leader`

上述两条就可能出现已有任期日志被覆盖的情况，这意味着已复制超过半数的以前任期日志被强制覆盖了，和前面提到的日志安全性矛盾。所以，Raft对日志提交有额外安全机制：**`Leader`只能提交当前任期Term的日志，旧任期Term（以前的数据）只能通过当前任期Term的数据提交来间接完成提交**。即日志提交有两个条件需要满足：

- 当前任期
- 复制结点超过半数


问题：为何Raft要保证状态机安全性原则？

![state_machine _safety.jpg](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/state_machine_safety.jpg)


1.  任期Term2：`Follower1`是`Leader`，此时LogIndex3已经复制到`Follower2`，且正在给`Follower3`复制，此时`Follower`突然宕机（`Follower3`复制失败，logIndex3并未被提交）
2.  任期Term3：假设`Leader`（`Follower1`）宕机，触发重新`Leader`选举。`Follower5`发起投票，可以得到自己、`Follower3`、`Follower4`的票（`3/5`），最终成为`Leader`；这是有可能发生的，因为前一步消息并未被多数节点确认
3.  在任期Term3内，新`Leader`（节点`5`）提交接收客户请求并提交LogIndex`3-5`，但是暂时未复制到其他节点，然后`Leader`宕机，此时又会触发重新选举
4.  任期Term4：`Leader`选举，`Follower1`发起选举，可以得到自己、`Follower2`、`Follower3`、`Follower4`的票（`4/5`），最终成为`Leader`。这也是有可能发生的
5.  此时`Follower1`将LogIndex3复制到`Follower3`，此时LogIndex3复制超过半数，接着在本地提交了LogIndex4，然后宕机
6.  任期Term4：`Leader`重选举：`Follower5`发起选举，可以得到自己、`Follower2`、`Follower3`、`Follower4`的票（`4/5`），最终成为`Leader`。注意，为何这里`Leader5`可以被选举为`Leader`呢？因为节点`5`此刻的TermId比其他几个节点（节点`2/3/4`）都大
6.  此时其他节点需要强制复制`Follower5`的日志，那么`Follower1`、`Follower2`、`Follower3`的日志被强制覆盖掉。即虽然LogIndex3被复制到了超过半数节点，但也有可能被覆盖掉
7.  因此，Raft必须限制此种情况的出现，即Raft协议在日志项提交上增加了限制：**只有当前任期且复制超过半数的日志才可以提交。即只有LogIndex4提交后，LogIndex3才会被提交**

那么，旧Term任期的数据是如何提交呢？


##  0x08 演示
可以通过Raft算法[动画演示](http://thesecretlivesofdata.com/raft/)或[官方博客](https://raft.github.io/)，对Raft算法的选举和日志复制过程有直观清晰的认知：

![flash](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/core-km/raft-flash-1.png)

## 0x09 参考
- [raft step by step入门演示](http://thesecretlivesofdata.com/raft/)
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
- [浅谈分布式一致性算法raft](https://www.cnblogs.com/wyq178/p/13899534.html)
- [raft算法中，5.4.2节一疑问？](https://www.zhihu.com/question/68287713)
- [为什么Raft不能“直接”提交之前任期的日志?](http://www.ilovecpp.com/2022/03/01/raft-prevlog-commit/)
- [5.4.2：提交之前任期的日志条目](https://knowledge-sharing.gitbooks.io/raft/content/chapter5/5-4/5-4-2.html)
- [5.3：日志复制](https://knowledge-sharing.gitbooks.io/raft/content/chapter5/5-3.html)