---
layout: post
title: 基于 CRON 库扩展的分布式 Crontab 的实现
subtitle: 基于 golang 的分布式定时器任务模型的通用实现
date: 2022-01-16
author: pandaychen
catalog: true
tags:
  - Crontab
  - Golang
---

## 0x00 前言

最新项目中使用到了分布式定时器事件（场景是服务器密码定期修改密码），本篇文章小结下。

## 0x01 单机 CRON 库介绍

## 0x02 分布式的实现

为了实现 HA，满足分布式高可用的特性，且实现满足两种定时任务需求的 crontab 系统：

1.  执行周期性任务
2.  只执行一次的任务

试想一下，我们的系统应该是下面这样的，满足如下特性：
![distribute-crontab](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/dcrontab/dcron1.png)

- 存储：定时任务必须落地在满足 AP 或 CP 的分布式系统中，典型的系统如 Redis（AP）、Etcd（CP）
- 工作节点的可用性：系统在定时器触发的那一刻需要至少选出一个工作节点（worknode）来执行任务
- 负载均衡：根据任务数据和节点数据均衡分发任务（避免任务都调度到负载较高的 worknode 上）
- 平滑扩容：如果任务节点负载过大，直接启动新的服务器后部分任务会自动迁移至新服务实现无缝扩容，各个 worknode 最好是无状态的
- 故障转移：单个节点故障，系统会自动将任务自动转移至其他正常节点执行
- 任务唯一：同一个服务内同一个任务只会启动单个运行实例，不会重复执行

## 0x03 源码分析

[dcron](https://github.com/libi/dcron)正是采用 redis 实现的一个分布式定时任务库。其整体架构如下：
![dcron 架构图](https://raw.githubusercontent.com/libi/dcron/master/dcron.png)

dcron 的背景和我们遇到的场景[基本类似](https://libisky.com/post/Dcron-基于redis与一致性哈希算法的分布式定时任务库)，主要解决单机场景下的定时任务问题。

#### 一致性 hash 的使用

## 0x04 可改进的思路

## 0x05 参考

- [dcron 库](https://github.com/libi/dcron)
- [Dcron:基于 redis 与一致性哈希算法的分布式定时任务库](https://libisky.com/post/Dcron-%E5%9F%BA%E4%BA%8Eredis%E4%B8%8E%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1%E5%BA%93)
