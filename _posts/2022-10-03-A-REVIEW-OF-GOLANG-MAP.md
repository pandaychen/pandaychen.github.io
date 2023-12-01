---
layout:     post
title:      内存缓存使用的一些技巧
subtitle:   
date:       2022-10-03
author:     pandaychen
catalog:    true
tags:
    - Cache
---


##  0x00    前言
本文梳理下笔者在项目开发中使用过一些内存缓存的技巧

##	0x01	双缓冲：double buffering
有一个场景是，存在某一本地文件配置（修改少），程序初始化时把文件内容读取到本地内存里（假设为map1），由于程序逻辑需要频繁且高性能的读取map1（尽量不加锁），在这种背景下如何实现安全的修改文件自动同步到map里（且不加锁）？有两种思路：

1.	分段锁
2.	使用双缓冲（double buffering）技术，方法是在后台修改一个副本的map，当修改完成后，将其原子性地替换为当前活动的map。这样，在修改期间，程序可以继续高性能地读取当前活动的map，修改完成后，将当前活动的map指向已经完成新一轮加载读取的map（ping-pong map）




##	0x	参考