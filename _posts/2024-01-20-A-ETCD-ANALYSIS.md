---
layout:     post
title:      Etcd 工程解析（一）
subtitle:   整体模块拆解
date:       2024-01-20
author:     pandaychen
catalog:    true
tags:
    - Etcd
---


##  0x00    前言
前文 [Raft 协议分析与实战（理论篇）](https://pandaychen.github.io/2020/01/10/A-RAFT-PROTOCOL-BASIC/) 大致介绍了 Raft 算法的理论基础，本系列分析下 Etcd 的工程化实现，主要包括：

-   主要模块拆分
-   存储的实现
-   Raft 库的实现
-   Etcd-Server 实现

本文主要基于 V3 版本分析，版本号 [3.1.10](https://github.com/pandaychen/etcd-3.1.10-codedump)

####    Etcd 的整体架构

![arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/etcd/code/arch.png)

##  0x01    模块拆分
下图展示了 etcd 如何处理一个客户端请求涉及到的模块和流程。包括如下几个核心模块：

-   etcd server：对外接受客户端的请求，请求 etcd 代码中的 etcd server 目录，其中还有一个 raft.go 的模块与 etcd raft 库进行通信。etcd server 中与存储相关的模块是 applierV3，这里封装了 V3 版本的数据存储， WAL，用于写数据日志，etcd 启动时会根据这部分内容进行恢复
-   etcd raft：etcd 的 raft 库。除了与本节点的 etcd server 通信之外，还与集群中的其他 etcd server 进行交互一致性数据同步的工作 aDD

图源自 [Etcd 存储的实现](https://www.codedump.info/post/20181125-etcd-server/)

##  0x02    Etcd 存储实现

##  0x0 参考
-   [Etcd Raft 库的工程化实现](https://www.codedump.info/post/20210515-raft/)
-   [ETCD 连载](https://www.lixueduan.com/categories/etcd/)
-   [etcd 实现 - 全流程分析](https://www.jianshu.com/p/2614fdb5d1c3)
-   [Etcd 存储的实现](https://www.codedump.info/post/20181125-etcd-server/)
-   [etcd Raft 库解析](https://www.codedump.info/post/20180922-etcd-raft/)
-   [Kubernetes 控制平面组件：etcd](https://burningmyself.gitee.io/kubernetes/k8s_etcd/)
-   [etcd 教程 (四)---etcd 架构及其实现简单分析](https://www.lixueduan.com/posts/etcd/04-etcd-architecture/)