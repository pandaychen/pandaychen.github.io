---
layout:     post
title:      基于redis的分布式限频库分析：ratelimit
subtitle:   
date:       2022-08-18
author:     pandaychen
catalog:    true
tags:
    - Redis
    - 限流
    - 分布式
---


##  0x00    前言
项目中要对接口调用进行并发限制，所以需要分布式的限流中间件，本文分析下基于Redis实现的分布式限流器，[ratelimit](https://github.com/vearne/ratelimit)


##  参考
-   [基于redis的分布式限频库](https://github.com/vearne/ratelimit/blob/master/README_zh.md)