---
layout: post
title: GoIM 源码分析（零）：总览
subtitle: 分析基于 golang 的高并发的聊天服务器实现
date: 2020-07-17
author: pandaychen
catalog: true
tags:
  - GOIM
---

## 0x00 前言

[GoIM](https://github.com/Terry-Mao/goim) 是一款基于长连接的 IM 服务。准备业余时间分析下其实现基础框架及优化思路（先前有部分阅读过，但未总结成文，网上的分析文章也非常多）。
首先，在分析项目源码前，我们思考下 B 站的弹幕场景（本质上是一个 IM，弹幕是实时消息中能被用户看到的内容，还有一部分是系统消息，用来主动触发业务逻辑或行为等）：

1.  打开一个 B 站 [视频](https://www.bilibili.com/video/BV1YK4y1s7UA?spm_id_from=333.851.b_7265706f7274466972737431.8)
2.  发送弹幕，弹幕在视频中显示
3.  服务端收到弹幕，向所有当前在此视频（房间）内的在线的用户广播这条弹幕
4.  其他用户也看到了这条弹幕
5.  弹幕需要落地存储
6.  当另外的新用户打开这个视频时，系统需要把所有该视频的弹幕推送给这个用户

GoIM 就实现了这样的功能（重点关注**实时的消息如何到达在线的用户端**）。带着场景，再结合分析源码，会更容易理解。此外，关于直播系统更多的特性，可以参考[此文](https://ruby-china.org/topics/39574)

#### 特性

- 轻量级
- 高性能
- 纯 Golang 实现
- 支持单个、多个、单房间以及广播消息推送
- 支持单个 Key 多个订阅者（可限制订阅者最大人数）
- 心跳支持（应用心跳和 tcp、keepalive）
- 支持安全验证（未授权用户不能订阅）
- 多协议支持（websocket，tcp）
- 可拓扑的架构（job、logic 模块可动态无限扩展）
- 基于 Kafka 做异步消息推送

## 0x01 GoIM 架构图

总结下 GoIM：

- 清晰的模块划分：Comet、Logic、Job 等
- gRPC、WebSocket、TcpServer 等
- 消息队列采用 Kafka
- 服务发现采用 B 站开源的 discovery，不过可以修改为 [支持 Etcd 的实现](https://github.com/Terry-Mao/goim/issues/251)
- 分层设计 / 定时器优化 / 对象（内存）分配优化

goim 官方的架构图如下，按照模块及数据流一层层分析是极佳的思路，图中标注了开文部分的数据流向示意：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/goim/goim1-arch.png)

[这里](https://juejin.im/post/5cbb9e68e51d456e51614aab) 提供了更加详细的数据流：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/goim/goim2-arch.png)

子模块的数据流向：
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/goim/goim3-arch.png)

## 0x02 GoIM 的扩展性及 HA

[goim via nats](https://github.com/tsingson/ex-goim)，这个项目使用 golang 的 [`NATS`](https://github.com/nats-io/nats-server) 高性能队列替换了 `Kafka` 的接口。另外，[goim 的业务集成 (分享会小结与 QA)](https://juejin.im/post/5cf27f8ee51d45775e33f50c) 这篇文章也对 goim 的系统集成与拓展提出了一些不错的思路，可以用作参考。

#### 各子模块的扩展

1、Comet 模块 <br>
Comet 模块属于接入层，需要扩容时直接开启多个 Comet。修改配置文件中的 `base` 节点下的 `server.id` 修改成不同值（注意一定要保证不同的 comet 进程值唯一），前端接入可以使用 LVS 或 DNS 来转发

2、Logic 模块 <br>
Logic 模块也是无状态的逻辑层，亦可按需扩容。使用 Nginx Upstream 来扩展 HTTP 接口，内部 RPC 部分，可以使用 LVS 四层转发

3、Kafka 中间件 <br>
Kafka 可以使用多 broker，或者多 partition 来扩展队列

4、Router 模块 <br>
Router 属于有状态节点，Logic 可以使用一致性 hash 配置节点，增加多个 router 节点（目前还不支持动态扩容），提前预估好在线和压力情况

5、Job 模块 <br>
Job 根据 Kafka 的 Partition 来扩展多 Job 工作方式，可以按照 Kafka 的消费者组来实现高可用

## 0x03 参考

- [GOIM 项目主页](https://goim.io/)
- [GOIM 手册](https://goim.io/tutorials/)
- [goim 源码剖析](https://laohanlinux.github.io/2016/12/22/goim%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/)
- [一套简洁的即时通信 (IM) 系统](https://alexstocks.github.io/html/im.html)
- [golang 从简单的即时聊天来看架构演变 (simple-chatroom)](https://github.com/LinkinStars/simple-chatroom)
- [goim 的业务集成 (分享会小结与 QA)](https://juejin.im/post/5cf27f8ee51d45775e33f50c)
- [GoIM 文档](https://raw.githubusercontent.com/Terry-Mao/goim/master/README_cn.md)
