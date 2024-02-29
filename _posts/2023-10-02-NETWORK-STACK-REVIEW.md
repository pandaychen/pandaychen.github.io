---
layout:     post
title:      重拾 Linux 网络（五）：TCP/IP 协议栈回顾
subtitle:   工作中遇到的那些协议栈相关的知识点汇总（基于 google-netstack 的讨论）
date:       2023-10-02
author:     pandaychen
catalog:    true
tags:
    - network
    - 协议栈
---


##  0x00    前言
本文汇总下笔者在近期工作中遇到的与 TCP/IP 协议栈相关的知识点汇总

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

##  0x03    传输层 - TCP

####    TCP options

1、tcp timestamp option 在三次握手中有什么作用？

TCP Timestamp Option（时间戳选项）在三次握手中的作用是提供一个可靠的方法来测量网络延迟和计算往返时间（`RTT`）。在三次握手中，发送方在 `SYN` 包中添加一个时间戳选项，接收方在 `SYN-ACK` 包中将该选项返回给发送方。发送方可以使用这个时间戳来计算 `RTT`，并根据计算结果调整数据包的发送时间，从而优化网络性能。此外，时间戳选项还可以防止一些网络攻击，如 TCP 序列号预测攻击。可以通过 `sysctl -w net.ipv4.tcp_timestamps=0` 禁用 TCP 时间戳选项，禁用 TCP 时间戳选项可以减少网络带宽和 CPU 资源的消耗，因为 TCP 时间戳选项需要在每个数据包中添加额外的数据，好处是提升安全性

####    拥塞控制算法


####    TCP 滑动窗口

####    SEQ && ACK
-   序列号（`SEQ`）：在建立连接时由内核生成的随机数作为其初始值，通过 `SYN` 报文传给接收端主机，每发送一次数据，就累加一次该数据字节数的大小，用来解决网络包乱序问题。累加方式为：序列号 = 上一次发送的序列号 + `len(data)`，若上一次发送的报文是 `SYN`/`FIN` 报文，则改为上一次发送的序列号 + `1`
-   确认号（`ACK`）：指下一次期望收到的数据的序列号，发送端收到接收方发来的 `ACK` 确认报文以后，可认为在这个序号以前的数据都已经被正常接收；用来解决丢包的问题。确认号 = 上一次收到的报文中的序列号 + `len(data)`。若收到的是 SYN/FIN 报文，则改为上一次收到的报文中的序列号 + `1`

在 TCP 重组中，依赖于这两个关键参数

####    TCP - 流重组
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


##  0x04    传输层 - UDP


##  0x05    应用层


##  0x06    杂项


##  0x07 参考
-   [流量控制 - 滑动窗口](https://wiki.brewlin.com/wiki/github/net-protocol/3.%E4%BC%A0%E8%BE%93%E5%B1%82/tcp/5.%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6%E7%9A%84%E5%AE%9E%E7%8E%B0-%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3/)
-   [An Implementation and Analysis of a Kernel Network Stack in Go with the CSP Style](https://arxiv.org/abs/1603.05636)
-   [SampleCaptures](https://wiki.wireshark.org/SampleCaptures#tcp)
-   [tcp retransmission](https://www.cloudshark.org/captures/64c49f52f75e)
-   [TCP 的那些事儿（上）](https://coolshell.cn/articles/11564.html)
-   [TCP 重组原理及实现](https://www.cnblogs.com/realjimmy/p/12933690.html)
-   [4.2 TCP 重传、滑动窗口、流量控制、拥塞控制](https://xiaolincoding.com/network/3_tcp/tcp_feature.html#%E9%87%8D%E4%BC%A0%E6%9C%BA%E5%88%B6)
-   [CS144 计算机网络 Lab1](https://kiprey.github.io/2021/11/cs144-lab1/)
-   [Comp 443 program 2: TCP Stream Reassembler](http://pld.cs.luc.edu/courses/443/spr10/tcp_reassembler.html)
-   [TCP Datagram Reassembly](https://pypcapkit.jarryshaw.me/en/v0.15.4/reassembly/tcp.html)
-   [CS144-minnow](https://github.com/cs144/minnow)
-   [【计算机网络】Stanford CS144 Lab Assignments 学习笔记](https://www.cnblogs.com/kangyupl/p/stanford_cs144_labs.html)