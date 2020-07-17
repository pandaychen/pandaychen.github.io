---
layout:     post
title:      GoIM 源码分析：总览
subtitle:   分析基于 golang 的高并发的聊天服务器实现
date:       2020-07-17
author:     pandaychen
catalog:    true
tags:
    - GoIM
---

##  0x00    前言
[GoIM](https://github.com/Terry-Mao/goim) 是一款基于长连接的 IM 服务。准备业余时间分析下其实现基础框架及优化思路（先前有部分阅读过，但未总结成文，网上的分析文章也非常多）。
首先，在分析项目源码前，我们思考下 B 站的弹幕场景（本质上是一个 IM）：
1.  打开一个 B 站 [视频](https://www.bilibili.com/video/BV1YK4y1s7UA?spm_id_from=333.851.b_7265706f7274466972737431.8)
2.  发送弹幕，弹幕在视频中显示
3.  服务端收到弹幕，向所有当前在此视频（房间）内的在线的用户广播这条弹幕
4.  其他用户也看到了这条弹幕

GoIM 就实现了这样的功能。带着场景，再结合分析源码，会更容易理解。

##  0x01    GoIM
总结下 GoIM：
-   清晰的模块划分：Comet、Logic、Job 等
-   gRPC、WebSocket、TcpServer 等
-   消息队列采用 Kafka
-   服务发现采用 B 站开源的 discovery，不过可以修改为 [支持 Etcd 的实现](https://github.com/Terry-Mao/goim/issues/251)
-   分层设计 / 定时器优化 / 对象优化

goim 官方的架构图如下，按照模块及数据流一层层分析是极佳的思路：
![image](https://wx1.sbimg.cn/2020/07/17/C8kMU.png)

[这里](https://juejin.im/post/5cbb9e68e51d456e51614aab) 提供了更加详细的数据流：
![image](https://wx2.sbimg.cn/2020/07/17/C80cd.png)


##  0x02    GoIM 的扩展
[goim via nats](https://github.com/tsingson/ex-goim)，这个项目使用 golang 的 [`NATS`](https://github.com/nats-io/nats-server) 高性能队列替换了 `Kafka` 的接口。另外，[goim 的业务集成 (分享会小结与 QA)](https://juejin.im/post/5cf27f8ee51d45775e33f50c) 这篇文章也对 goim 的系统集成与拓展提出了一些不错的思路，可以用作参考。


##  0x03    参考
-   [GOIM 项目主页](https://goim.io/)
-   [GOIM 手册](https://goim.io/tutorials/)
-   [goim 源码剖析](https://laohanlinux.github.io/2016/12/22/goim%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/)
-   [一套简洁的即时通信 (IM) 系统](https://alexstocks.github.io/html/im.html)
-   [golang 从简单的即时聊天来看架构演变 (simple-chatroom)](https://github.com/LinkinStars/simple-chatroom)
-   [goim 的业务集成 (分享会小结与 QA)](https://juejin.im/post/5cf27f8ee51d45775e33f50c)