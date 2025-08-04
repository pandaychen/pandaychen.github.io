---
layout:     post
title:      重拾 Linux 网络（五）：TCP/IP 协议栈回顾
subtitle:   工作中遇到的那些协议栈相关的知识点汇总（基于 google-netstack 的应用开发）
date:       2023-10-02
author:     pandaychen
catalog:    true
tags:
    - network
    - 协议栈
---


##  0x00    前言
本文汇总下笔者在近期工作中遇到的与 TCP/IP 协议栈相关的知识点汇总

####    Linux内核网络数据收发流程
1、Linux网络接收数据包流程如下，在Linux内核中当数据包到达网卡的时候，通过`DMA`方式将数据映射到内存，然后硬中断通知CPU有数据到来，调用硬中断处理函数，之后交给软中断去处理。通过`ksoftirq`调用软中断处理函数，收包的软中断处理函数是`net_rx_action`函数，主要将Ring Buffer缓冲区数据做成`sk_buff`送给上层协议栈进行处理，之后数据包经过层层解包最终将数据放入套接字缓冲区中，CPU通过将数据拷贝给应用程序

![recv](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/how_kernel_recv_data.png)

2、Linux网络发送数据包流程如下，首先，应用层应用程序通过CPU将数据拷贝到套接字缓冲区，然后数据包经过层层处理封装好数据包，经过软中断处理函数，通过建立`DMA`映射，将数据放到发送缓冲区中，最后经过一个物理网卡发送出去

![send](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/how_kernel_send_data.png)

####    DMA
DMA（Direct Memory Access）它可以在CPU不参与的情况下，完成外部硬件设备和存储器之间或者存储器和存储器之间的高速数据传输，数据可以直接通过DMA进行快速拷贝，节省 CPU 的资源去做其他工作

##  0x01    链路层

-   目的 mac 地址：`6` 字节物理地址
-   源 mac 地址：`6` 字节物理地址
-   数据包协议类型： 为 `0x8000` 时为 IPv4 协议包，为 `0x8060` 时，后面为 ARP 协议包
-   数据包：网卡输送能力上限 MTU(`1500` 字节), 对网络层 ip 协议对封装

```TEXT
0               1               2               3               4               5               6
0 1 2 3 4 5 6 7 8 1 2 3 4 5 6 7 8 1 2 3 4 5 6 7 8 1 2 3 4 5 6 7 8 1 2 3 4 5 6 7 8 1 2 3 4 5 6 7 8
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          DESTINATION      MAC    6 字节目的 mac 地址                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          ORIGINALSRC      MAC    6 字节源 mac 地址                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       2 字节 网络层协议类型      |     46 - 1500 字节（ip 包头 + 传输层包头 + 应用层数据）           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

##  0x02    网络层

##  0x03    传输层：TCP

####    TCP协议格式

![tcp-proto]()

####    TCP的状态机

![tcp-state-machine]()

####    TCP options

1、tcp timestamp option 在三次握手中有什么作用？

TCP Timestamp Option（时间戳选项）在三次握手中的作用是提供一个可靠的方法来测量网络延迟和计算往返时间（`RTT`）。在三次握手中，发送方在 `SYN` 包中添加一个时间戳选项，接收方在 `SYN-ACK` 包中将该选项返回给发送方。发送方可以使用这个时间戳来计算 `RTT`，并根据计算结果调整数据包的发送时间，从而优化网络性能。此外，时间戳选项还可以防止一些网络攻击，如 TCP 序列号预测攻击。可以通过 `sysctl -w net.ipv4.tcp_timestamps=0` 禁用 TCP 时间戳选项，禁用 TCP 时间戳选项可以减少网络带宽和 CPU 资源的消耗，因为 TCP 时间戳选项需要在每个数据包中添加额外的数据，好处是提升安全性
####    TCP options：SACK


####    TCP options：D-SACK

####    SEQ && ACK
-   序列号（`SEQ`）：在建立连接时由内核生成的随机数作为其初始值，通过 `SYN` 报文传给接收端主机，每发送一次数据，就累加一次该数据字节数的大小，用来解决网络包乱序问题。累加方式为序列号等于上一次发送的序列号加上`len(data)`，若上一次发送的报文是 `SYN`/`FIN` 报文，则调整为上一次发送的序列号加一
-   确认号（`ACK`）：指下一次期望收到的数据的序列号，发送端收到接收方发来的 `ACK` 确认报文以后，可认为在这个序号以前的数据都已经被正常接收；用来解决丢包的问题。确认号等于上一次收到的报文中的序列号再加上 `len(data)`。若收到的是 SYN/FIN 报文，则等于上一次收到的报文中的序列号加一

在 TCP 重组中，依赖于这两个关键参数

####    MTU And MSS
-   MTU: Maxitum Transmission Unit 最大传输单元
-   MSS: Maxitum Segment Size 最大分段大小

##  0x04    TCP：高级话题

####    TCP 滑动窗口机制
基于滑动窗口的流控机制，是TCP避免发送方的数据填满接收方的缓存

####    TCP重传机制

1、超时重传机制

2、快速重传机制

3、SACK 方法

4、Duplicate SACK：重复收到数据的问题

####    拥塞控制算法（Congestion Handling）
Congestion Handling是TCP协议设计者针对整个网络的考量，即TCP不是一个自私的协议，当拥塞发生的时候，要做自我牺牲，就像交通阻塞一样，每个车都应该把路让出来，避免加重拥塞。Congestion Handling主要是如下四个算法：

-   慢启动（Slow Start）
-   拥塞避免（Congestion Avoidance）
-   拥塞发生
-   快速恢复（Fast Recovery）

TCP拥塞控制目的就是避免发送方的数据填满整个网络。为了在发送方调节所要发送数据的量，引入新概念即拥塞窗口，拥塞窗口 `cwnd`是发送方维护的一个的状态变量，它会根据网络的拥塞程度动态变化的。前文提到过发送窗口 `swnd` 与接收窗口 `rwnd` 是约等于的关系，那么引入拥塞窗口后，此时发送窗口的值是`swnd = min(cwnd, rwnd)`，即拥塞窗口和接收窗口中的最小值。拥塞窗口 `cwnd` 变化的规则：

-   只要网络中没有出现拥塞，`cwnd` 就会增大
-   但网络中出现了拥塞，`cwnd` 就减少

问题：协议栈如何判断当前网络出现拥塞？判断方式是**只要发送方没有在规定时间内接收到 ACK 应答报文，也就是发生了超时重传，就会认为网络出现了拥塞**

接下来就分析下拥塞控制的四类算法

1、慢启动

慢启动的意思是，刚刚加入网络的连接，一点一点地提速，不要一上来就像那些特权车一样霸道地把路占满，规则为**当发送方每收到一个 ACK，拥塞窗口 cwnd 的大小就会加 1，是指数性的增长**，慢启动的算法如下（如果网速很快的话，ACK也会返回得快，RTT也会短，那么慢启动就一点也不慢）：

1.  连接建好的开始先初始化`cwnd = 1`，表明可以传一个`MSS`大小的数据
2.  每当收到一个ACK，`cwnd++`，呈线性上升
3.  每当过了一个RTT，`cwnd = cwnd*2`，呈指数上升
4.  慢启动何时结束？状态变量慢启动门限ssthresh（slow start threshold）是一个上限，当`cwnd >= ssthresh`时，就会结束慢启动模式，进入拥塞避免算法

-   当 `cwnd < ssthresh` 时，使用慢启动算法。
-   当 `cwnd >= ssthresh` 时，就会使用拥塞避免算法

![slow-start]()

```BASH
# kernel 5.4.119
[root@VM-X-X-tencentos X-X]#  ss -nli |grep cwnd
         cubic cwnd:10
         cubic cwnd:10
         cubic cwnd:10
```


小结下，TCP拥塞控制的各个阶段特点：

| 阶段 | 触发条件 | cwnd变化规则| 作用 |
| :-----| :---- | :---- | :----|
| 慢启动 | 连接建立或超时重传后 | 每收到1个ACK，cwnd增加1 MSS，呈指数增长 | 快速探测可用带宽 |
| 拥塞避免 | cwnd >= ssthresh | 每经过1 RTT（可能包含多个ACK），cwnd增加1 MSS，呈线性增长 | 避免突破网络容量阈值 |
| 快速重传 | 收到3个重复ACK | cwnd = max(cwnd/2, 2 MSS) + 3 MSS | 快速恢复单包丢失 |
| 快速恢复 | 快速重传后 | 每收到重复ACK，cwnd增加1 MSS | 维持传输速率避免中断 |
|超时重传|RTO超时（严重拥塞）|cwnd重置为1 MSS|彻底降速缓解拥塞|

####    TCP：流重组
`SEQ` 是为了保证 TCP 数据包的按顺序传输来设计的，可以有效的实现 TCP 数据的完整传输，特别是在数据传送过程中出现错误的时候可以有效的进行错误修正。在 TCP 会话的重新组合过程中需要按照数据包的序列号对接收到的数据包进行排序

1、BSD 的实现思路 <br>

![BSD](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/protocol/tcpip/tcp-assembly-1.png)
以 SSH 连接的服务端（被动接收）出发，实现涉及两个队列，队列 A 存放顺序到来的数据包，队列 B 存放失序到来的数据包；假设队列 A 最后的缓存数据记录为 `seq=100`/`len=100`，下一个到达的数据包可能如下：
-   case1：顺序到来的数据包，数据包 `2` 的序列号为 `seq2 = seq1+len1`，由此数据包的 seq 可知，这个报文预期后续报文，将此报文追加到正常报文队列
-   case2：重复数据包，如数据包 `3`/`4`/`5`，都包含在队列 A 已经缓存的数据包 buffer 之中，应该被丢弃
-   case3：重叠数据包，如数据包 `6` 的前部分 `seq=150`、`len=50` 与已缓存的数据包 buffer 重叠，而后部分 `len=50` 则是新数据，此时应该对这个报文作如下处理
    -   计算重复字节数：`(seq1+len1)-seq2=100+100-150=50`，即这个报文段前 `50` 个字节是重复的，需要丢弃
    -   截取报文段新数据，即只保留字节序号段 `200~249`
    -   重新设置这个报文段的 `seq=200`
    -   重新设置这个报文段的数据长度 `len2=50`
    -   将重新设置后报文段加入顺序队列 A
-   case4：提前到达的报文，如数据包 `7`：`seq2>seq1+len1`，是提前到来的报文，此时应该将这个报文放置到失序报文队列 B 缓存起来，以备后续重组使用

这样直到客户端断开此 TCP 连接（`FIN` OR `RST`），此时将正常报文队列和失序报文队列中的数据合并起来，完成重组。取出正常报文队列最后一个报文的 `seq` 和 `len`，在失序报文队列中查找属于它的后续报文，该报文是否可以作为正常报文队列的后续报文，至此重组完成

2、cs144的tcp assembler<br>


3、gvisor 的实现<br>
参考 [代码](https://github.com/google/gvisor/tree/master/pkg/tcpip/transport/tcp)

####    gopacket 中的 tcp 重组实现

##  0x0    传输层：UDP

##  0x0    应用层

##  0x0    杂项

##  0x0 参考
-   [流量控制 - 滑动窗口](https://wiki.brewlin.com/wiki/github/net-protocol/3.%E4%BC%A0%E8%BE%93%E5%B1%82/tcp/5.%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6%E7%9A%84%E5%AE%9E%E7%8E%B0-%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3/)
-   [An Implementation and Analysis of a Kernel Network Stack in Go with the CSP Style](https://arxiv.org/abs/1603.05636)
-   [SampleCaptures](https://wiki.wireshark.org/SampleCaptures#tcp)
-   [tcp retransmission](https://www.cloudshark.org/captures/64c49f52f75e)
-   [TCP 的那些事儿（上）](https://coolshell.cn/articles/11564.html)
-   [TCP 的那些事儿（下）](https://coolshell.cn/articles/11609.html)
-   [TCP 重组原理及实现](https://www.cnblogs.com/realjimmy/p/12933690.html)
-   [4.2 TCP 重传、滑动窗口、流量控制、拥塞控制](https://xiaolincoding.com/network/3_tcp/tcp_feature.html#%E9%87%8D%E4%BC%A0%E6%9C%BA%E5%88%B6)
-   [CS144 计算机网络 Lab1](https://kiprey.github.io/2021/11/cs144-lab1/)
-   [Comp 443 program 2: TCP Stream Reassembler](http://pld.cs.luc.edu/courses/443/spr10/tcp_reassembler.html)
-   [TCP Datagram Reassembly](https://pypcapkit.jarryshaw.me/en/v0.15.4/reassembly/tcp.html)
-   [CS144-minnow](https://github.com/cs144/minnow)
-   [【计算机网络】Stanford CS144 Lab Assignments 学习笔记](https://www.cnblogs.com/kangyupl/p/stanford_cs144_labs.html)
-   [The IP Fragmentation and Re-assembly Algorithm](https://www.cs.emory.edu/~cheung/Courses/455/Syllabus/4b-internet/IP-protocol5.html)
-   [25 张图，一万字，拆解 Linux 网络包发送过程](https://mp.weixin.qq.com/s?__biz=Mzg3ODUxNzM0MA==&mid=2247484532&idx=2&sn=2a934f8d6ae87b53b62671972987388d&scene=21#wechat_redirect)
-   [图解Linux网络包接收过程](https://cloud.tencent.com/developer/article/1966873)
-   [Linux是怎么从网络上接收数据包的](https://mp.weixin.qq.com/s/gVYdRWCoQwpFtsKXMyIHRg)
-   [4.2 TCP 重传、滑动窗口、流量控制、拥塞控制](https://xiaolincoding.com/network/3_tcp/tcp_feature.html#%E6%8B%A5%E5%A1%9E%E6%8E%A7%E5%88%B6)