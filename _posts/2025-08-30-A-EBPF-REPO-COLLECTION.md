---
layout:     post
title:      eBPF awesome
subtitle:   
date:       2025-08-30
author:     pandaychen
catalog:    true
tags:
    - eBPF
---

##  0x00  前言
本文记录一下eBPF相关的项目


##  0x01   工具
1、bpftrace

##  0x02    主机&&容器安全
1、Falco：使用eBPF检测主机和容器中的恶意行为

##  0x03    可观测
1、beyla：一个使用eBPF的应用指标采集工具，基于eBPF实现了内核层面http/https和应用层面go-rpc等指标的自动采集

2、Hubble：和Cilium配套的监控系统，使用eBPF的K8s网络、服务、安全性可观测的平台。

3、pixie - 利用eBPF实现对Kubernetes的可观测性。其功能包括协议跟踪、用户应用性能profile，以及支持分布式bpftrace部署
x s
##  0x04    网络

1、Cilium：一个依赖于BPF和XDP技术的项目，提供了基于eBPF程序动态生成的内核内高速网络和容器安全策略执行

2、Open vSwitch（OvS）及其相关项目Open Virtual Network（OVN，一种开源网络虚拟化解决方案）正在考虑在各个层面使用eBPF

3、Katran - Meta开源的基于XDP的L4负载均衡

4、Project Calico: 知名的K8s网络插件，也是用了eBPF提供一个低延迟，高吞吐的数据面

##  0x04    总结

##  0x05    参考
-   [eBPF入门指南：小白也能轻松掌握](https://juejin.cn/post/7353085048856985654)
