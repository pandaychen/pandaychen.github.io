---
layout:     post
title:      Golang-time/rate 限速算法实现分析
subtitle:   分析 Golang 标准库提供的令牌桶限流器
date:       2020-04-05
author:     pandaychen
header-img: img/super-mario.jpg
catalog: true
category:   false
tags:
    - 限流
    - Golang
---

##  0x00    前言
这篇文章来分析下标准库 [time/rate](https://github.com/golang/time/blob/master/rate/rate.go) 的使用及实现细节，此库同样基于 Token Bucket 实现了限流。

从上一篇文章 [JuJu-Ratelimit 限速算法实现分析](https://pandaychen.github.io/2020/04/02/JUJU-RATELIMIT-ANALYSIS/) 的总结，令牌桶的实现本质就是利用了 <font color="#dd0000">Token 数可以和时间跨度相互转化</font> 的原理。需要有如下关键信息：

-	生产 Token 令牌的速率：一秒钟可以产生多少 Token（生产一个 Token 需要多长时间单位），记为 $$p$$（`1s`）
-	Token 令牌桶的大小 $Bucket_{size}$

基于上面这两个基础信息，容易得到：
1. 生成 $N$ 个新的 Token 一共需要的时间单位：$\frac{N}{p}*1s$
2. 给定一段时长 $$Duration$$，这段时间一共可以生成多少个 Token，$\frac{Duration}{1s}*p$


##  0x02  参考
-	[time/rate 源码](https://github.com/golang/time/blob/master/rate/rate.go)
-	[Golang 标准库限流器 time/rate 实现剖析](https://www.cyhone.com/articles/analisys-of-golang-rate/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权