---
layout:     post
title:      透明代理汇总：Transparent Proxy All In One
subtitle:   记录项目预研 / 开发的若干细节
date:       2023-08-18
author:     pandaychen
catalog:    true
tags:
    - network
---


##  0x00    前言
本文汇总下笔者在调研透明代理（Linux 下的全流量代理网关）的一些技术学习与分享

##  0x01    代理的技术方案比较

####  TProxy
TProxy 是一种 Linux 内核模块，可以在 Linux 内核层面拦截网络数据并进行处理，从而实现透明代理。TProxy 可以在不改变源 IP 地址和端口的情况下，将数据包重定向到代理服务器（通常是监听在本机`lo`网卡上的进程）进行处理。TProxy 适用于 HTTP、HTTPS 等基于 TCP 协议的应用，但不支持 UDP 协议，TProxy 主要通过修改 iptables 和路由规则实现

####  Tun：虚拟网卡技术
TUN 是一种虚拟网络设备，可以将网络数据包通过用户空间程序进行处理，然后再发送到网络中。通过创建 TUN 设备，可以将网络数据包从内核层面转移到用户空间，从而实现透明代理。TUN 可以处理任何协议的数据包，包括 TCP/UDP 等。但是，TUN 需要在用户空间编写代理程序，相对比较复杂


####  NFQUEUE ：网络过滤器队列技术
NFQUEUE 是一种 iptables 和 ip6tables 的目标（an iptables and ip6tables target），将网络包处理决定委托给用户态软件。作为一种在 Linux 内核中实现的机制，允许用户空间程序与内核空间的网络堆栈进行交互。nfqueue 的主要目的是在用户空间中对网络数据包进行处理和过滤，然后决定是否允许数据包通过或者丢弃；nfqueue 的工作原理如下：

- 注册：用户空间程序通过 `libnetfilter_queue` 库注册一个队列，该队列与内核中的一个特定 nfqueue 实例相关联。程序可以指定队列的编号和回调函数
- 数据包捕获：内核中的 iptables（网络过滤器）或者 nftables（新一代网络过滤器）根据预先定义的规则将数据包发送到指定的 nfqueue 实例。这些规则可以基于数据包的来源、目的、协议等属性进行匹配
- 用户空间处理：当数据包进入 nfqueue 时，内核将它们发送到关联的用户空间程序。程序可以对数据包进行任意处理，例如修改、丢弃或者允许通过
- 决策：用户空间程序根据自己的逻辑做出决策，将对数据包的处理结果（允许通过、丢弃等）返回给内核。内核根据这个决策来处理数据包
- 注销：当用户空间程序不再需要处理数据包时，它可以通过 `libnetfilter_queue` 库注销队列


####  iptables REDIRECT 重定向
iptables redirect 是一种基于 iptables 的透明代理技术。它通过更改数据包的目标地址，将数据包重定向到本地的代理服务器。代理服务器收到数据包后，需要进行 NAT（网络地址转换）操作，才能找到原始目标地址。这种方法会导致一定程度的性能损失

####  对比
- nfqueue 提供了一种灵活的机制，允许用户空间程序对网络数据包进行深度检查和处理。这对于实现防火墙、入侵检测系统、网络监控工具等应用非常有用。
- TProxy 适用于基于 TCP 协议的应用，使用相对简单
- TUN 适用于任何协议的应用，但需要在用户空间编写代理程序，使用相对复杂


##  0x02    Review：Tproxy 构建透明代理
TProxy（Transparent Proxy）是内核支持的一种透明代理方式，于 Linux `2.6.28` 引入。不同于 NAT 修改数据包目的地址实现重定向，**TProxy 仅替换数据包的 skb 原本持有的 socket**，不需要修改数据包标头，TPROXY 是一个 iptables 扩展的名称。

####  使用方式

1.  由 `--on-port`/`--on-ip` 指定重定向目的地
2.  由于没有修改数据包目的地址，在 `PREROUTING` 之后的路由选择仍会因为目的地址不是本机而走到 `FORWARD` 链。所以需要 ** 策略路由（ip rule）** 来引导数据包进入 `INPUT` 链

```BASH
ip rule add fwmark 0x233 table 100
ip route add local default dev lo table 100

iptables -t mangle -A PREROUTING -p udp -j TPROXY --on-ip 127.0.0.1 --on-port 10000 --tproxy-mark 0x233
iptables -t mangle -A PREROUTING -p tcp -j TPROXY --on-ip 127.0.0.1 --on-port 10000 --tproxy-mark 0x233
```

用监听在 `:10000` 的 socket 替换数据包原 socket，同时打上 `0x233` 标记。设置策略路由，让所有带有 `0x233` 标记的数据包使用 `100` 号路由表。在 `100` 号表中设定默认路由走 `lo` 本地回环设备。而从本地回环设备发出的数据包都会被视作发向本机，也就避免了被转发出去

####  典型配置举例

1、clash（开源版）配合 tproxy

具体配置可以参考：[a-clash-tproxy-gateway.md](https://gist.github.com/phlinhng/38a141862de775b10c613f7f2c6ade99)

核心 `3` 步：

1）开启转发配置，在网关机器上打开 ipv4 转发

```BASH
echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf && sysctl -p
```

2）设置 tproxy 配置，将本地路由放通，不走 clash

```BASH
# ROUTE RULES
ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100

# CREATE TABLE
iptables -t mangle -N clash

# RETURN LOCAL AND LANS
iptables -t mangle -A clash -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A clash -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A clash -d 192.168.50.0/16 -j RETURN
iptables -t mangle -A clash -d 192.168.9.0/16 -j RETURN

iptables -t mangle -A clash -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A clash -d 240.0.0.0/4 -j RETURN

# FORWARD ALL
iptables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 1
iptables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 1

# REDIRECT
iptables -t mangle -A PREROUTING -j clash
```

3）开启 clash，clash 配置文件中需要打开 tproxy 配置

```TEXT
# Transparent proxy server port for Linux (TProxy TCP and TProxy UDP)
tproxy-port: 7893
```

4）作为网关服务器的额外配置

最后在局域网的 `DHCP` 服务器设置网关为该机器 IP 即可，也可以在希望走代理机器的网关设置为该机器 IP

##  0x03    v2ray
[v2ray-core](https://github.com/v2fly/v2ray-core)

####  非 tun 模式
同样使用 tproxy 进行，完整的配置过程，可以参考此文 [透明代理入门](https://xtls.github.io/document/level-2/transparent_proxy/transparent_proxy.html#iptables-%E5%AE%9E%E7%8E%B0%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86)

####  tun 模式


##  0x04    Clash.Meta && clash-plus-pro [Clash 二次开发]

鉴于 clash premium 不开源（开源版本不支持 tun 虚拟网卡功能），有如下二次开发版本：

-   [Clash.Meta](https://github.com/MetaCubeX/Clash.Meta/tree/Meta) 这是 [clash](https://github.com/Dreamacro/clash) 的二次开发版本
-   [clash-plus-pro](https://github.com/yaling888/clash)
- [clashr：My Own Fork of Clash - Support RuleProviders、SSR(Add "chacha20" and "none" ciphers) 、 TCP/UDP Tunnel、MTProxy Inbound、MixEC(RESTful Api+Socks5+MTProxy) Inbound and Tun(from sing-tun or @yaling888 or @comzyh) Inbound](https://github.com/wwqgtxx/clashr)
- [clash](https://github.com/comzyh/clash)：一个 tun 模式的实现


####    Clash.Meta
功能强大且复杂：

1、代理模块

-   支持出站传输协议 VLESS Reality/Vision 流控
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


可以使用工具 [tpclash](https://github.com/mritd/tpclash) 来部署 clash，clash 配置文档 [参考](https://wiki.metacubex.one/config/)

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

##  0x05    Gost

##  0x06    tun2socks
Tun2socks 的原理是通过创建 TUN/TAP 设备，将本地应用程序的网络流量发送到用户空间进行处理，并通过代理服务器将数据包转发到目标服务器。通过设置转发规则，可以实现对不同应用程序的网络流量进行不同的处理，其核心过程如下：

![tun2socks.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tun2socks.png)

- TUN/TAP 设备：tun2socks 通过创建一个 TUN/TAP 设备，将本地应用程序的网络流量发送到该设备
- 网络协议栈：Tun2socks 将 TUN/TAP 设备与本地网络协议栈连接起来，使得网络流量可以在用户空间进行处理。当本地应用程序发送数据包时，数据包首先会被发送到 TUN/TAP 设备，然后通过网络协议栈进行处理
- 代理服务器：Tun2socks 可以将网络流量发送到指定的代理服务器，代理服务器可以是 HTTP 代理、SOCKS 代理或 Shadowsocks 代理等。当 Tun2socks 将数据包发送到代理服务器时，会对数据包进行加密或解密，确保数据的安全性
- 转发规则：Tun2socks 可以根据用户的需求设置转发规则，将指定的应用程序的网络流量转发到指定的代理服务器。如可以将浏览器的网络流量转发到 HTTP 代理服务器，将其他应用程序的网络流量转发到 Shadowsocks 代理服务器


##  0x07    seeker
[seeker](https://github.com/gfreezy/seeker) 是基于 rust 实现的，通过使用 tun 来实现透明代理，实现了类似 surge 增强模式与网关模式

####    实现原理
seeker 参考了 Surge 的实现原理，使用了 fake-ip 模式，基本如下：

1.  seeker 会在本地启动一个 DNS server，并自动将本机 DNS 修改为 seeker 的 DNS 服务器地址

2. seeker 会创建一个 TUN 设备，并将 IP 设置为 `10.0.0.1`，系统路由表设置 `10.0.0.0/16 ` 网段都路由到 TUN 设备

3.  有应用请求 DNS 的时候， seeker 会为这个域名返回 `10.0.0.0/16` 网段内一个唯一的 IP

4.  seeker 从 TUN 接受到 IP 包后，会在内部组装成 TCP/UDP 数据

5.  seeker 会根据规则和网络连接的 `uid` 判断走代理还是直连

6.  如果需要走代理，将 TCP/UDP 数据转发到 `SS` 服务器 / `socks5` 代理，从代理接收到数据后，再返回给应用；如果直连，则本地建立直接将数据发送到目标地址

##  0x08    surge（MAC）


##  0x09  参考
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
-   [深入理解 Linux TProxy](https://rook1e.com/p/linux-tproxy/)
- [用 iptables/tproxy 做透明代理](https://zhuanlan.zhihu.com/p/191601221)
- [clash：Add TCP TPROXY support](https://github.com/Dreamacro/clash/pull/1049)
- [Clash 作为网关的透明代理](https://www.wogong.net/blog/2020/11/clash-transparent-proxy)
- [Clash 作为网关的透明代理](https://www.wogong.net/blog/2023/07/clash-gateway)
- [记录 Tun 透明代理的多种实现方式，以及如何避免 routing loop](https://chaochaogege.com/2021/08/01/57/)
- [Mellow is a rule-based global transparent proxy client for Windows, macOS and Linux. Also a Proxifier alternative.](https://github.com/mellow-io/mellow)
- [透明代理入门](https://xtls.github.io/document/level-2/transparent_proxy/transparent_proxy.html#iptables-nftables)
- [go-nfqueue](https://github.com/florianl/go-nfqueue)
- [OpenGFW](https://github.com/apernet/OpenGFW)