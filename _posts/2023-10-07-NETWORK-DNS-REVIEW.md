---
layout:     post
title:      重拾 Linux 网络（六）：DNS 劫持与代理
subtitle:   DNS 劫持代理 review 与 CoreDNS 分析
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
先回顾一下 DNS 解析的基础流程，如下图：



##  0x02    Kubernetes 的 DNS
温馨提示：特别注意域名结尾是否有一个点号`.`

1 当`ndots`小于`options ndots`

`options ndots`的值默认是`1`，在K8S中为`5`，参考下面的例子，在一个namespace `demo-ns`中有两个`SVC`，分别为`demo-svc1`和`demo-svc2`，其`/etc/resolv.conf`内容大致如下：

```bash
nameserver 10.32.0.10
search demo-ns.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

当在`demo-svc1`中直接请求域名`demo-svc2`，此时`ndots`为`1`（小于配置中的`5`），因此会触发配置中的`search`规则，即第一个解析的域名是`demo-svc2.demo-ns.svc.cluster.local`，当解析不出来的时候继续下面的`demo-svc2.svc.cluster.local`、`demo-svc2.cluster.local`，最后才是直接去解析`demo-svc2.`；注意此规则适用于任何一个域名，如在pod中去访问一个外部域名如`github.io`时也会依次进行上述查询。

2  当ndots大于等于options ndots

在`demo-svc1`中直接请求域名`demo-svc2.demo-ns.svc.cluster.local`，此时的`ndots`为`4`，依然会触发上面的`search`规则。而请求域名`demo-svc2.demo-ns.svc.cluster.local.`时，`ndots`为`5`，因此不会触发`search`规则，直接去解析`demo-svc2.demo-ns.svc.cluster.local.`这个域名并返回结果；又如请求更长的pod域名`pod-1.demo-svc2.demo-ns.svc.cluster.local.`，此时的`ndots`为`6`，亦不会触发`search`规则，会直接查询域名并返回解析

####    小结
-   同命名空间（namespace）内的服务直接通过$service_name进行互相访问而不需要使用全域名（FQDN），此时DNS解析速度最快
-   跨命名空间（namespace）的服务，可以通过$service_name.$namespace_name进行互相访问，此时DNS解析第一次查询失败，第二次才会匹配到正确的域名
-   所有的服务之间通过全域名（FQDN）$service_name.$namespace_name.svc.$cluster_name.访问的时候DNS解析的速度最快
-   在K8S集群内访问大部分的常见外网域名（ndots小于5）都会触发search规则，因此在访问外部域名的时候可以使用FQDN，即在域名的结尾配置一个点号`.`

####    DNS 解析流程



##  0x02


##  0x0 adguard 的 DNS


##  DNS 常用库 1：miekg/dns


##  DNS 常用库 2：coredns


##  透明代理中的 DNS

####    DNS：FAKE-IP



##  0x0 参考
-   [Examples made with Go DNS](https://github.com/miekg/exdns)
-   [CoreDNS 篇 10 - 分流与重定向](https://tinychen.com/20221120-dns-13-coredns-10-dnsredir-and-alternate/)
-   [CoreDNS 篇 9-kubernetes 插件](https://tinychen.com/20221107-dns-12-coredns-09-kubernetes/)
-   [adguard-dns：概览](https://adguard-dns.io/kb/zh-CN/private-dns/overview/)
-   [抓包就明白 CoreDNS 域名解析](https://juejin.cn/post/7053824699268071432)
-   [Public DNS resolver that protects you from ad trackers](https://github.com/AdguardTeam/AdGuardDNS)
-   [回顾与展望 AdGuard DNS](https://adguard-dns.io/zh_cn/blog/reimagining-adguard-dns.html)
-   [DNS-over-QUIC](https://adguard-dns.io/zh_cn/blog/dns-over-quic-official-standard.html)