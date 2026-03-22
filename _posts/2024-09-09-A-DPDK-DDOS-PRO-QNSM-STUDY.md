---
layout:     post
title:      QNSM 实现：原理&&分析
subtitle:
date:       2024-09-09
author:     pandaychen
header-img:
catalog: true
tags:
    - DDoS
    - DPDK
---

##  0x00    前言
QNSM是一个旁路部署的全流量，实时，高性能网络安全监控引擎，基于DPDK开发，集成了DDOS检测和IDPS模块。

####    DDOS检测
DDOS检测功能包括:

-   全流量检测，可以部署在IDC环境，支持SYN，ACK，RST，FIN，SYNACK，ICMP，UDP FLOOD以及反射攻击（DNS/NTP/SSDP反射...）
-   实时多维度聚合数据
-   流采样数据，提供攻击事件未检出后的fall back机制
-   随时停启的聚合数据输出
-   数据以json格式输出，便于数据分析
-   针对UDP反射攻击，提供DFI/DPI机制（MEMCACHE，TFTP，CHARGEN，CLDAP，QOTD...）
-   事件过程中dump攻击数据包
-   支持IPv4和IPv6

##  0x01    原理 && 应用


##  0x03    参考
-   [DPDK文档：哈希库](https://dpdk-docs.readthedocs.io/en/latest/prog_guide/hash_lib.html?spm=a2c6h.12873639.article-detail.7.6fd92ea3ZAbHlH)
-   [qnsm](https://github.com/iqiyi/qnsm)
-   [qnsm guide](https://github.com/iqiyi/qnsm/blob/master/doc/guide.md)
-   [爱奇艺网络流量分析引擎 QNSM 及其应用](https://www.infoq.cn/article/EeLZSUYzo7rbjaxk6Tau)
-   [爱奇艺开源的高性能网络安全监控引擎](https://cloud.tencent.com/developer/article/2153196)