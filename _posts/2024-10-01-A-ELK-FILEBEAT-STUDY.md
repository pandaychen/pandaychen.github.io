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
-   将加工好的数据发送到output（下游）
-   Filebeat实现了ACK反馈确认机制：成功发送到output后，会将当前进度反馈给input
-   Filebeat在发送output失败后，会启动retry机制，配合ACK反馈确认机制，保证了每次消息至少发送一次的语义
-   Filebeat在发送output时，由于网络等原因发生阻塞，则在input端会减慢收集，自适应匹配output的状态


####    filebeat基础组件介绍
下面是官方的架构图，两个重要组件`Input`、`Harvester`：
![gf](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/filebeat/filebeat.png)


##  0x01 libbeat


##  0x02    源码分析
从源码视角，filebeat的架构如下：

![module](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/filebeat/filebeat-module-1.jpeg)


##  0x03 参考
-   [Filebeat 收集日志的那些事儿](https://cloud.tencent.com/developer/article/1634020)
-   [【Elastic Stack系列】第四章：源码分析(一) Filebeat篇](https://twocups.cn/index.php/2021/03/13/32/)
-   [Configure the output]()
-   [filebeat源码解析](https://cloud.tencent.com/developer/article/1367784)
-   [监控日志系列---- Filebeat原理](https://kingjcy.github.io/post/monitor/log/collect/filebeat/filebeat-principle/)