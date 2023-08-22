---
layout:     post
title:      Transparent Proxy All In One
subtitle:
date:       2023-08-18
author:     pandaychen
catalog:    true
tags:
    - network
---


##  Ox00    前言
本文汇总下笔者在调研透明代理的一些分享（Linux下的全流量代理网关）

##  0x01    tproxy VS tun

前文分别介绍了 tun/tproxy 如何实现透明代理技术，那么它们实现透明代理的区别是什么？主要实现方式和应用场景有所不同：

-   TProxy 是一种 Linux 内核模块，可以在 Linux 内核层面拦截网络数据并进行处理，从而实现透明代理。TProxy 可以在不改变源 IP 地址和端口的情况下，将数据包重定向到代理服务器进行处理。TProxy 适用于 HTTP、HTTPS 等基于 TCP 协议的应用，但不支持 UDP 协议
-   TUN 是一种虚拟网络设备，可以将网络数据包通过用户空间程序进行处理，然后再发送到网络中。通过创建 TUN 设备，可以将网络数据包从内核层面转移到用户空间，从而实现透明代理。TUN 可以处理任何协议的数据包，包括 TCP、UDP 等。但是，TUN 需要在用户空间编写代理程序，相对比较复杂。

因此，TProxy 适用于基于 TCP 协议的应用，使用相对简单；而 TUN 适用于任何协议的应用，但需要在用户空间编写代理程序，使用相对复杂


从实现上来说，tproxy

##  0x02    iptables 构建透明代理



##  0x02    v2ray
[v2ray-core](https://github.com/v2fly/v2ray-core)

##  0x03    Clash.Meta && clash-plus-pro [Clash 二次开发]

鉴于 clash premium 不开源（开源版本不支持 tun 虚拟网卡功能），有如下二次开发版本：

-   [Clash.Meta](https://github.com/MetaCubeX/Clash.Meta/tree/Meta)这是 [clash](https://github.com/Dreamacro/clash) 的二次开发版本
-   [clash-plus-pro](https://github.com/yaling888/clash)


####    Clash.Meta
功能强大且复杂：

1、代理模块

-   支持出站传输协议 VLESS Reality, Vision 流控
-   支持出站传输协议 Trojan XTLS
-   支持出站传输协议 Hysteria
-   支持出站传输协议 TUIC
-   支持出站传输协议 ShadowTLS
-   支持 PASS（跳过）规则
-   主动健康检测 urltest/fallback（基于 tcp 握手，限定时间内多次失败会主动触发健康检测使用节点）
-   支持策略组正则筛选
-   允许 Provider 请求通过代理更新
-   Proxy-Providers 支持直接使用 V2rayN 等客户端的普通订阅
-   Relay 代理链支持 UDP over TCP
-   TCP 连接并发


2、规则模块

-   支持规则 GEOSITE
-   支持入站类型规则 IN-TYPE
-   支持规则集 RULE-SET
-   支持规则 SRC-PORT 和 DST-PORT 的多端口条件
-   支持规则对 TCP / UDP 分别管控
-   支持 Network 规则，支持匹配网络类型 (TCP / UDP)
-   支持逻辑判断规则 (NOT / OR / AND)
-   支持子规则集
-   支持所有规则的源 IPCIDR 条件，只需附加到末尾即可
-   支持 GEODATA MODE 切换，mmdb / dat
-   支持切换 GEODATA LOADER 模式切换 , 普通 / 小内存模式
-   支持 GeoSite 按需加载
-   支持使用 geox-url 自定义的 GEOIP / GEOSITE 数据库下载地址

3、DNS 模块

-   支持 Sniffer 域名嗅探器
-   支持 Fallback-Filter 使用 Geosite
-   恢复 Redir-Host 远程解析
-   支持使用代理解析 ip
-   支持使用 Policy 分流 DNS
-   支持 DNS over HTTP/3
-   支持 DNS over QUIC

4、TUN 模块

-   支持 macOS、Linux 和 Windows
-   内置 iptables, 无需手动配置
-   内置 Wintun 驱动程序
-   支持 gvisor / system / lwip 堆栈


可以使用工具 [tpclash](https://github.com/mritd/tpclash) 来部署 clash，clash配置文档[参考](https://wiki.metacubex.one/config/)

配置文件如下：

```bash
# 需要开启 TUN 配置
tun:
  enable: true
  stack: system
  dns-hijack:
    - any:53
  #   - 8.8.8.8:53
  #   - tcp://8.8.8.8:53
  auto-route: true
  auto-redir: true
  auto-detect-interface: true

# 开启 DNS 配置, 且使用 fake-ip 模式
dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  default-nameserver:
    - 183.60.82.98
  nameserver:
    - 183.60.82.98

proxies:
  - name: "http"
    type: http
    server: x.x.x.x
    port: 8080

rules:  #这里默认只使用一个代理
  - MATCH,http
```

##  0x04    Gost

##  0x05    tun2socks


##  0x06    seeker
[seeker](https://github.com/gfreezy/seeker)是基于rust实现的，通过使用 tun 来实现透明代理，实现了类似 surge 增强模式与网关模式

####    seeker的实现原理
seeker 参考了 Surge for Mac 的实现原理，基本如下：

seeker 会在本地启动一个 DNS server，并自动将本机 DNS 修改为 seeker 的 DNS 服务器地址

seeker 会创建一个 TUN 设备，并将 IP 设置为 10.0.0.1，系统路由表设置 10.0.0.0/16 网段都路由到 TUN 设备

有应用请求 DNS 的时候， seeker 会为这个域名返回 10.0.0.0/16 网段内一个唯一的 IP

seeker 从 TUN 接受到 IP 包后，会在内部组装成 TCP/UDP 数据

seeker 会根据规则和网络连接的 uid 判断走代理还是直连

如果需要走代理，将 TCP/UDP 数据转发到 SS 服务器/ socks5 代理，从代理接受到数据后，在返回给应用；如果直连，则本地建立直接将数据发送到目标地址


##  0x07    surge（MAC）


##  0x  参考
-   [透明代理入门](https://xtls.github.io/document/level-2/transparent_proxy/transparent_proxy.html#iptables-%E5%AE%9E%E7%8E%B0%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86)
-   [Clash.Meta Docs](https://wiki.metacubex.one/)
-   [Meta Kernel](https://github.com/MetaCubeX/Clash.Meta/tree/Meta)
-   [我花了一周准备，想和你分享 Clash 所有特性运用到极致之后的体验](https://www.v2ex.com/t/948499)
-   [kungfu：Flexible DNS hijacking and proxy tool.](https://github.com/yinheli/kungfu)
-   [tpclash：Transparent proxy tool for Clash](https://github.com/mritd/tpclash)
-   [clash 配置自定义规则](https://tomorrow505.xyz/clash%E9%85%8D%E7%BD%AE%E8%87%AA%E5%AE%9A%E4%B9%89%E8%A7%84%E5%88%99/)
-   [透明代理入门](https://xtls.github.io/document/level-2/transparent_proxy/transparent_proxy.html#iptables-%E5%AE%9E%E7%8E%B0%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86)
-   [GO Simple Tunnel](https://gost.run/)
-   [Surge 官方中文指引：理解 Surge 原理](https://manual.nssurge.com/book/understanding-surge/cn/)