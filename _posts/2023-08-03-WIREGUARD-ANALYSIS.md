---
layout:     post
title:      Wireguard 实现原理与分析
subtitle:   一个基于 golang 实现的 vpn-tunnel 分析
date:       2023-08-03
author:     pandaychen
catalog:    true
tags:
    - network
    - wireguard
---


##  Ox00    前言
WireGuard（简称 wg）是一种快速、现代、安全的 VPN 协议，基于 golang 的开源地址 [在此](https://git.zx2c4.com/wireguard-go)


##  0x01   工作原理
WireGuard 以 UDP 实现，但是运行在 IP 层（即 ip-over-udp）。每个 Peer 都会生成一个 `wg0` 虚拟网卡，同时服务端会在物理网卡上监听 UDP `51820` 端口。应用程序的包发送到内核以后，如果地址是虚拟专用网内部的，那么就会交给 `wg0` 设备，WireGuard 就会把这个 IP 包封装成 WireGuard 的包，然后在 UDP 中发送出去，对方的 Peer 的内核收到这个 UDP 包后再反向操作，解包成为 IP 包，然后交给对应的应用程序。 WireGuard 实现的虚拟网卡就像 `eth0` 一样，可以使用标准的 Linux 工具操作，像是 `ip`, `ifconfig` 之类的命令。所以 WireGuard 也就不用实现 QoS 之类的功能，毕竟其他工具已经实现了


####    基础结构
wireguard-go 的基础结构包括了公私钥、网络接口、UDP 连接、会话等。

-   公私钥：wireguard-go 使用 `Curve25519` 椭圆曲线加密算法生成公私钥对，用于加密和解密数据包
-   网络接口：wireguard-go 创建一个虚拟网络接口，该接口可以像物理网络接口一样接收和发送数据包
-   UDP 连接：wireguard-go 使用 UDP 协议进行数据包传输，它通过 UDP 连接实现数据包的发送和接收
-   会话：wireguard-go 使用会话来管理网络连接，会话包括了公私钥、网络接口、UDP 连接等信息

####    数据包处理
wireguard-go 的数据包处理包括了加密解密、封装解封装等过程。

-   加密解密：wireguard-go 使用 `ChaCha20` 加密算法和 `Poly1305` 消息认证码算法对数据包进行加密和解密。加密过程中，数据包首先被封装到一个新的数据包中，并加入了一些元数据，然后使用 `ChaCha20` 算法进行加密，最后使用 `Poly1305` 算法生成消息认证码。解密过程中，首先使用 `Poly1305` 算法验证消息认证码，然后使用 `ChaCha20` 算法进行解密，最后解封装数据包
-   封装解封装：wireguard-go 使用 WireGuard 协议对数据包进行封装和解封装。封装过程中，数据包被封装到一个新的数据包中，并加入了一些元数据，例如发送方公钥、接收方公钥、时间戳等。解封装过程中，首先根据接收方公钥和发送方公钥计算出会话密钥，然后使用会话密钥对数据包进行解密和解封装

####    网络连接管理
wireguard-go 的网络连接管理包括了会话管理、路由管理、握手协议等。

-   会话管理：wireguard-go 使用会话来管理网络连接，会话包括了公私钥、网络接口、UDP 连接等信息。会话可以通过公钥来唯一标识一个网络连接，可以添加、删除、更新等操作
-   路由管理：wireguard-go 使用内核路由表来管理网络路由，可以添加、删除、更新等操作。路由表中的路由信息包括了目标 IP 地址、子网掩码、网关等信息
-   握手协议：wireguard-go 使用 Noise 协议进行握手，该协议可以实现安全的密钥交换和会话建立。在握手过程中，发送方和接收方会交换公钥、时间戳、随机数等信息，然后根据这些信息计算出会话密钥


##  0x02    wireguard组网
参考[安装 Wireguard 并组建中心辐射型网络](https://naiv.fun/Ops/53.html)



####    路由方式
[WireGuard基本原理](https://cshihong.github.io/2020/10/11/WireGuard%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/)


##  0x02    代码
wireguard-go的核心协议栈也是基于gvisor[实现的](https://github.com/WireGuard/wireguard-go/blob/master/go.mod#L10)

-   tun 相关接口代码：[tun.go](https://git.zx2c4.com/wireguard-go/tree/tun/netstack/tun.go)


##  0x0 参考
-   [wireguard 基本原理和最佳实践](https://www.ctyun.cn/developer/article/400185847689285)
-   [Routing & Network Namespace Integration](https://www.wireguard.com/netns/)
-   [WireGuard is not only for VPN](https://amod-kadam.medium.com/wireguard-is-not-only-for-vpn-9394605f458f)
-   [Quick Start](https://www.wireguard.com/quickstart/)
-   [WireGuard 教程：使用 DNS-SD 进行 NAT-to-NAT 穿透](https://icloudnative.io/posts/wireguard-endpoint-discovery-nat-traversal/)
-   [为什么 wireguard-go 高尚而 boringtun 孬种](https://zhuanlan.zhihu.com/p/548053546)
-   [WireGuard 教程：WireGuard 的工作原理](https://icloudnative.io/posts/wireguard-docs-theory/)
-   [Nebula: Open Source Overlay Networking](https://nebula.defined.net/docs/)
-   [WireGuard 到底好在哪？](https://zhuanlan.zhihu.com/p/404402933)
-   [吞吐优化札记](https://zhuanlan.zhihu.com/p/515881346)
-   [WireGuard VPN tunnel between 2 network islands](https://www.wut.de/e-55www-27-apus-000.php)
-   [WireGuard 教程：使用 DNS-SD 进行 NAT-to-NAT 穿透](https://icloudnative.io/posts/wireguard-endpoint-discovery-nat-traversal/)
-   [WireGuard 基本原理](https://cshihong.github.io/2020/10/11/WireGuard%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/)
-   [WIREGUARD SITE TO SITE CONFIGURATION](https://www.procustodibus.com/blog/2020/12/wireguard-site-to-site-config/)
-   [MULTI-HOP WIREGUARD](https://www.procustodibus.com/blog/2022/06/multi-hop-wireguard/)
-   [安装 Wireguard 并组建中心辐射型网络](https://naiv.fun/Ops/53.html)