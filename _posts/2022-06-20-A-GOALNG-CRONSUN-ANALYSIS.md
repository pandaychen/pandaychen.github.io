---
layout: post
title: Cronsun：任务计划统一集中管理系统设计与分析
subtitle: 分析一款典型的分布式任务调度系统
date: 2022-06-20
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - Crontab
---

##  0x00    开篇
cronsun 是一个分布式任务系统，单个节点和 Linux 机器上的 `crontab` 类似，目的是解决多台 Linux 机器上 crontab 任务管理不方便的问题，同时提供任务高可用的支持（当某个节点死机的时候可以自动调度到正常的节点执行），此外，有页面及邮件支持。

本文分析下其实现的思路，之前已有相关文章：
1.  [Golang CRON 库 Crontab 的使用与设计](https://pandaychen.github.io/2021/10/05/A-GOLANG-CRONTAB-V3-BASIC-INTRO/)
2.  [基于 CRON 库扩展的分布式 Crontab 的实现](https://pandaychen.github.io/2022/01/16/A-GOLANG-CRONTAB-V3-ANALYSIS/)

##	0x05	总结

```text
                                      [web]
                                        |
                           --------------------------
 (add/del/update/exec jobs)|                        |(query job exec result)
                         [etcd]                 [mongodb]
                           |                        ^
                  --------------------              |
                  |        |         |              |
               [node.1]  [node.2]  [node.n]         |
   (job exec fail)|        |         |              |
[send mail]<-----------------------------------------(job exec result)
```

笔者阅读过的分布式crontab实现，几乎都是与该架构类似；
1.  [gocron](https://github.com/ouqiang/gocron)：定时任务管理系统，文档见[此](https://github.com/ouqiang/gocron/wiki)

##	0x02 组件介绍
-	MongoDB
-	Etcd

##	核心代码分析


##	总结
本项目


##  0x06	参考
-	[分布式任务系统 cronsun](http://bos.itdks.com/786da33844604637be5479c3a16af11e.pdf)