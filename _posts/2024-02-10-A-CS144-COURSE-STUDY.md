---
layout:     post
title:      CS144：一个轻量级 TCP 重组器的实现与分析
subtitle:
date:       2024-02-10
author:     pandaychen
catalog:    true
tags:
    - network
---


##  0x00    前言
CS144 课程提供了一个用户态 TCP 协议的简单实践，先回顾下 TCP 的若干特点，为了最大限度的保证传输可靠：

1、可靠性保证

-   校验和，TCP 每个报文都有校验和字段，防止数据丢失或出错
-   序列化和确认号，保证每个序号的字节都交付，解决丢失、重复等问题
-   超时重传，对于超时未能确认的报文，TCP 会重传这些包，确保数据达到对端
-   拥塞控制等等

2、全双工

TCP 协议通信的双方既可以发送数据，又可以接受数据，双方拥有独立的序号、窗口等信息。一个 TCP 连接既可以是 sender 也可以是 receiver，同时连接拥有两个字节流，一个输出流，被 sender 控制，另一个是输入流，由 receiver 管理

3、字节流

TCP 数据传输是基于流的，意味着 TCP 传输数据是没有边界和大小限制的，可以传输无限量的字节。但是 TCP 报文大小是有限制的，这主要取决于滑动窗口大小、路径最大传输单元 MTU 等因素。TCP 数据写、读、传入都是基于字节流，因此常常会有字节流乱序发生，因此 TCP 需要重组

####    sponge 协议
CS144 给出了一个简化 TCP 版本（sponge），它的特点如下

-   Sponge 协议建立在 UDP/IP（基于 TUN/TAP 技术） 之上
-   Sponge 协议是一种简易版 TCP 协议，和 TCP 协议一样有滑动窗口、重传、校验和等功能，但是一些复杂的特性（如：紧急指针、拥塞控制、Options）暂不支持

####    cs144-Lab
LAB0-LAB4 是循序渐进的


##  0x02   核定代码分析

```TEXT
libsponge/
├── byte_stream.cc          // ByteStream(数据流) 实现文件
├── byte_stream.hh          // ByteStream 头文件
├── stream_reassembler.cc   // StreamReassembler(数据流重组器) 实现文件
├── stream_reassembler.hh   // StreamReassembler 头文件
├── tcp_connection.cc       // TCPConnection(TCP 连接) 实现文件
├── tcp_connection.hh       // TCPConnection 头文件
├── tcp_receiver.cc         // TCPReceiver(TCP 接收者) 实现文件
├── tcp_receiver.hh         // TCPReceiver 头文件
├── tcp_sender.cc           // TCPSender(TCP 发送者) 实现文件
├── tcp_sender.hh           // TCPSender 头文件
├── wrapping_integers.cc    // WrappingIntegers(包装 32 位 seqno、ackno) 实现文件
└── wrapping_integers.hh    // WrappingIntegers 头文件
```

##  0x03    LAB0
实现一个读写字节流 [`ByteSteam`]()，用来作为存放给用户调用获取数据的有限长度缓冲区 buffer

##  0x04    LAB1
`StreamReassembler` 实现，作为 `ByteSteam` 的上游，实现 sponge-TCP 协议流重组的功能

##  0x05    LAB2
`TCPReceiver` 的实现，`TCPReceiver` 包含了一个 `StreamReassembler` 实现，它主要解决如下问题：

-   从哪里接收 TCP 分段数据
-   重组数据（调用 `StreamReassembler`），缓存数据
-   重组后的数据放在哪里（写入 `ByteSteam`），等待上层读取

##  0x06    LAB3
`TCPSender` 实现，仅包含 outbound 的 `ByteSteam`，但实际相对于 `TCPReceiver` 要复杂，需要支持：
-   根据 `TCPSender` 当前的状态对可发送窗口进行填充，发包
-   `TCPSender` 需要根据对方通知的窗口大小和 `ackno` 来确认对方当前收到的字节流进度
-   需支持超时重传机制，根据时间变化（RTO），定时重传那些还没有 `ack` 的报文

##  0x07    LAB4
`TCPConnection` 的实现，包含如下步骤：

-   发起连接
-   写入数据，发送数据包
-   接收数据包
-   关闭连接
-   丰富超时重传机制


##  0x01    参考
-   [谈谈用户态 TCP 协议实现](https://zhuanlan.zhihu.com/p/412758694)
-   [CS144 Lab2](https://doraemonzzz.com/2021/12/27/2021-12-27-CS144-Lab2/)
-   [CS144 Lab2 翻译](https://doraemonzzz.com/2021/12/27/2021-12-27-CS144-Lab2%E7%BF%BB%E8%AF%91/)
-   [CS144 Labs 总结](https://carlclone.github.io/labs/cs144/)
-   [CS144-Lab2-TCPReceiver](https://www.cnblogs.com/lawliet12/p/17066709.html)