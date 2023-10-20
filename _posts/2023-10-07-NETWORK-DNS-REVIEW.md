---
layout:     post
title:      重拾 Linux 网络（六）：DNS 劫持与代理
subtitle:   DNS 劫持代理回顾与 CoreDNS
date:       2023-10-07
author:     pandaychen
catalog:    true
tags:
    - network
    - DNS
---


##  Ox00    前言
本文是项目中的 DNS 相关开发工作记录

1.  如何合适的实现 DNS 查询缓存？
2.  DOH、DOT 实现
3.  如何优雅的实现一个 DNS 代理
4.  透明网关中的 DNS 机制、DNS 分流机制等

##  0x01    基础
先回顾一下 DNS 的基础知识


##  0x02    Kubernetes 的DNS 


####    DNS解析流程



##  0x02    


##  0x0 adguard的DNS


##  DNS常用库1：miekg/dns


##  DNS常用库2：coredns


##  透明代理中的DNS

####    DNS：FAKE-IP



##  0x0 参考
-   [Examples made with Go DNS](https://github.com/miekg/exdns)
-   [CoreDNS篇10-分流与重定向](https://tinychen.com/20221120-dns-13-coredns-10-dnsredir-and-alternate/)
-   [CoreDNS篇9-kubernetes插件](https://tinychen.com/20221107-dns-12-coredns-09-kubernetes/)
-   [adguard-dns：概览](https://adguard-dns.io/kb/zh-CN/private-dns/overview/)
-   [抓包就明白CoreDNS域名解析](https://juejin.cn/post/7053824699268071432)
-   [Public DNS resolver that protects you from ad trackers](https://github.com/AdguardTeam/AdGuardDNS)
-   [回顾与展望 AdGuard DNS](https://adguard-dns.io/zh_cn/blog/reimagining-adguard-dns.html)
-   [DNS-over-QUIC](https://adguard-dns.io/zh_cn/blog/dns-over-quic-official-standard.html)