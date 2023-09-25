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

##  0x02 tun And 转发代理原理

1.  从 Tun 读取 IP 数据包，写进 `gVisor` 网络栈
2.  `gVisor` 网络栈上注册 `HandlePacket` 函数用于处理 TCP/UDP/ICMP 等，将 `gonet.Conn` 和 `gonet.PacketConn` 传给 proto 里实现的 `gonet.Handler` 处理
3.  双向流 copy


## 0x03 TUN gvisor 开发基础

####  一般流程

####  netstack 开发示例

服务端代码参考：[tun_tcp_echo](https://github.com/google/netstack/blob/master/tcpip/sample/tun_tcp_connect/main.go)
客户端代码参考：[tun_tcp_connect](https://github.com/google/netstack/blob/master/tcpip/sample/tun_tcp_connect/main.go)


####  gvisor 库开发示例
相比于 netstack，gvisor 稍微有些不同，如下：

服务端代码参考：[tun_tcp_echo](https://github.com/google/gvisor/blob/master/pkg/tcpip/sample/tun_tcp_echo/main.go)
客户端代码参考：[tun_tcp_connect](https://github.com/google/gvisor/blob/master/pkg/tcpip/sample/tun_tcp_connect/main.go)

##  0x04  配置基础
以 [tun2socks](https://github.com/xjasonlyu/tun2socks/wiki/Examples#linux) 为例


####  关闭

1、关闭虚拟网卡`tun0`

```BASH
ip link set tun0 down #将tun0网卡设为下线状态，停止其网络连接
```

2、删除虚拟网卡`tun0`

```BASH
ip link delete tun0 #该命令将删除tun0网卡，彻底关闭其网络连接。请注意，执行该命令后，tun0网卡的配置信息将被永久删除，无法恢复
```

####  删除指定的路由

下述静态路由，通过`route del -net 192.168.10.0 netmask 255.255.255.0 dev eth0`指令进行删除：

```BASH
192.168.10.0    0.0.0.0         255.255.255.0   U     100    0        0 eth0 
```

##  0x05  TUN with DNS

[](https://dreamacro.github.io/clash/)

##  0x0 参考
-   [网络编程学习：vpn 实现](https://www.jianshu.com/p/e74374a9c473)
-   [Listener and Dialer using a TUN and netstack.](https://github.com/costinm/tungate)
-   [tungate.go](https://github.com/costinm/tungate/blob/main/gvisor/cmd/tungate.go)
-   [在 Linux 上开 clash tun 模式 clash 是怎么劫持 dns 的](https://www.v2ex.com/t/880652)