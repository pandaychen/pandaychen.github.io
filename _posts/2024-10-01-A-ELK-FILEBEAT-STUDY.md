---
layout:     post
title:      FILEBEAT ：一款轻量级日志采集器agent的实现与分析
subtitle:   数据的搬运工
date:       2024-10-01
author:     pandaychen
header-img:
catalog: true
tags:
    - Golang
---

##  0x00    前言
下图形象的说明了filebeat的功能，主要包括两点：

1.  支持从不同数据源收集数据并转换成事件
2.  发送事件到指定的输出（支持多种输出）

![filebeat](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/filebeat/arch-1.png)

-   支持从多种[不同]()的input（上游）中接受需要收集的数据（如log input从日志文件中收集数据）
-   对收集来的数据进行加工（如多行合并，增加业务自定义字段，json encode等）
-   将加工好的数据发送到[output](https://www.elastic.co/guide/en/beats/filebeat/current/configuring-output.html)（下游）
-   Filebeat实现了ACK反馈确认机制：成功发送到output后，会将当前进度反馈给input
-   Filebeat在发送output失败后，会启动retry机制，配合ACK反馈确认机制，保证了每次消息至少发送一次的语义
-   Filebeat在发送output时，由于网络等原因发生阻塞，则在input端会减慢收集，自适应匹配output的状态

本文版本基于[v9.0.1](https://github.com/elastic/beats/releases/tag/v9.0.1)进行

####    基础组件介绍
下面是官方的架构图，两个重要组件`Input`、`Harvester`：
![gf](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/filebeat/filebeat.png)


##  0x01    源码分析：libbeat

TODO

####    基础数据结构


####    核心方法实现

##  0x02 源码分析：filebeat 

从源码视角，filebeat的架构如下：

![module](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/filebeat/filebeat-module-1.jpeg)

####    filebeat主要模块

-   `Crawler`： 负责管理和启动各个`Input`，管理所有`Input`收集数据并发送事件到libbeat的`Publisher`
-   `Input`： 负责管理和解析输入源的信息，以及为每个文件启动 `Harvester`
-   `Harvester`： 负责读取一个文件的数据，对应一个输入源，是收集数据的实际工作者（配置中一个具体的`Input`可以包含多个输入源`Harvester`）
-   `module`：简化了一些常见程序日志（比如nginx日志）收集、解析、可视化（kibana dashboard）配置项
-   `fileset`：`module`下具体的一种`Input`定义（比如nginx包括access和error log），包含
    -   输入配置
    -   es ingest node pipeline定义
    -   事件字段定义
    -   示例kibana dashboard
-   `Registrar`：接收libbeat反馈回来的ACK, 作相应的持久化，管理记录每个文件处理状态，包括偏移量、文件名等信息。当 Filebeat 启动时，会从 `Registrar` 恢复文件处理状态

####    libbeat主要模块

`Pipeline`（publisher）：负责管理缓存、`Harvester` 的信息写入以及 `Output` 的消费等，是 Filebeat 最核心的组件

-   `client`： 提供`Publish`接口让filebeat将事件发送到`Publisher`。在发送到队列之前，内部会先调用processors（包括input 内部的processors和全局processors）进行处理
-   `processor`：事件处理器，可对事件按照配置中的条件进行各种处理（比如删除事件、保留指定字段，过滤添加字段，多行合并等）
-   `queue`：事件队列，有`memqueue`（基于内存）和`spool`（基于磁盘文件）两种实现
-   `outputs`：事件的输出端，比如ES、Logstash、kafka等
-   `acker`：事件确认回调，在事件发送成功后进行回调


##  0x0 参考
-   [Filebeat 收集日志的那些事儿](https://cloud.tencent.com/developer/article/1634020)
-   [【Elastic Stack系列】第四章：源码分析(一) Filebeat篇](https://twocups.cn/index.php/2021/03/13/32/)
-   [Configure the output]()
-   [filebeat源码解析](https://cloud.tencent.com/developer/article/1367784)
-   [监控日志系列---- Filebeat原理](https://kingjcy.github.io/post/monitor/log/collect/filebeat/filebeat-principle/)
-   [elastic：使用 Linux 安全计算模式 (seccomp)](https://elastic.ac.cn/guide/en/beats/heartbeat/current/linux-seccomp.html)
-   [监控日志系列---- Filebeat](https://kingjcy.github.io/post/monitor/log/collect/filebeat/filebeat/)