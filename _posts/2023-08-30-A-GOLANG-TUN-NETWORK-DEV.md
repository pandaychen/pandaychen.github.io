---
layout: post
title: Golang 网络编程（三）：tun 网络编程
subtitle: golang tun 网络编程应用以及 gvisor 协议栈开发介绍
date: 2023-08-30
author: pandaychen
catalog: true
tags:
  - Golang
  - 网络编程
  - Tun
  - gvisor
---


##  0x00    前言


##  0x01  基础知识

####  iobased or fdbased

启动 tun 网卡的方式如何选型？fdbased 和 iobased 的区别主要体现在数据的传输方式上

当使用 fdbased 方式启动 tun 网卡时，tun 网卡会被创建为一个虚拟的网络接口，数据通过文件系统进行读写，即数据传输通过文件的方式进行。这种方式的优点是简单易用，不需要特殊的设备或驱动程序，适用于小型应用程序。但是，由于数据传输需要经过文件系统，因此可能会影响数据传输的速度

当使用 iobased 方式启动 tun 网卡时，tun 网卡会被创建为一个虚拟的网络接口，数据通过输入 / 输出方式进行读写，即数据直接传输到设备或驱动程序中。这种方式的优点是数据传输速度快，适用于大型应用程序。但是，由于需要特殊的设备或驱动程序支持，因此相对来说比较复杂

小结一下，当需要处理大量数据或需要更快的速度时，建议使用 iobased 方式启动 tun 网卡；当数据量较小或需要更简单的管理和备份时，建议使用 fdbased 方式启动 tun 网卡


## 0x02 TUN gvisor 开发基础

####  一般流程


####  使用 gvisor 简化

1.  从 Tun 读取 IP 数据包，写进 `gVisor` 网络栈
2.  `gVisor` 网络栈上注册 `HandlePacket` 函数用于处理 TCP/UDP/ICMP 等，将 `gonet.Conn` 和 `gonet.PacketConn` 传给 proto 里实现的 `gonet.Handler` 处理



##  0x03  TUN with DNS

[](https://dreamacro.github.io/clash/)

##  参考
-   [网络编程学习：vpn 实现](https://www.jianshu.com/p/e74374a9c473)
-   [Listener and Dialer using a TUN and netstack.](https://github.com/costinm/tungate)
-   [tungate.go](https://github.com/costinm/tungate/blob/main/gvisor/cmd/tungate.go)
-   [在 Linux 上开 clash tun 模式 clash 是怎么劫持 dns 的](https://www.v2ex.com/t/880652)