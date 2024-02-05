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
前文介绍了TUN技术在透明代理中的使用，TUN技术可应用于多种互联网场景：

- 透明代理技术
- 加速器
- 虚拟专用网络（VPN）
- 跨平台（OS）网络互联

##  0x01  基础知识

####  iobased OR fdbased

启动 TUN 网卡的方式如何选型？fdbased 和 iobased 的区别主要体现在数据的传输方式上

当使用 fdbased 方式启动 TUN 网卡时，TUN 网卡会被创建为一个虚拟的网络接口，数据通过文件系统进行读写，即数据传输通过文件的方式进行。这种方式的优点是简单易用，不需要特殊的设备或驱动程序，适用于小型应用程序。但是，由于数据传输需要经过文件系统，因此可能会影响数据传输的速度

当使用 iobased 方式启动 TUN 网卡时，TUN 网卡会被创建为一个虚拟的网络接口，数据通过输入/输出方式进行读写，即数据直接传输到设备或驱动程序中。这种方式的优点是数据传输速度快，适用于大型应用程序。但是，由于需要特殊的设备或驱动程序支持，因此相对来说比较复杂


#### TUN 转发代理原理

1.  从 Tun（虚拟网卡接口）读取 IP 数据包，写进 `gVisor` 网络栈
2.  `gVisor` 网络栈上注册 `HandlePacket` 函数用于处理 TCP/UDP/ICMP 等，将 `gonet.Conn` 和 `gonet.PacketConn` 传给 proto 里实现的 `gonet.Handler` 处理
3.  双向流 copy


##  0x03  TUN基础：VPN互联

## 0x03 TUN gvisor 开发基础

####  一般流程

####  netstack 开发示例

服务端代码参考：[tun_tcp_echo](https://github.com/google/netstack/blob/master/tcpip/sample/tun_tcp_connect/main.go)

客户端代码参考：[tun_tcp_connect](https://github.com/google/netstack/blob/master/tcpip/sample/tun_tcp_connect/main.go)


####  gvisor 库开发示例
相比于 netstack，gvisor 稍微有些不同，如下：

服务端代码参考：[tun_tcp_echo](https://github.com/google/gvisor/blob/master/pkg/tcpip/sample/tun_tcp_echo/main.go)

客户端代码参考：[tun_tcp_connect](https://github.com/google/gvisor/blob/master/pkg/tcpip/sample/tun_tcp_connect/main.go)

##  0x04 配置基础
以 [tun2socks](https://github.com/xjasonlyu/tun2socks/wiki/Examples#linux) 为例

####  TUN虚拟网卡配置
1、关闭虚拟网卡 `tun0`

```BASH
ip link set tun0 down #将 tun0 网卡设为下线状态，停止其网络连接
```

2、删除虚拟网卡 `tun0`

```BASH
ip link delete tun0 #该命令将删除 tun0 网卡，彻底关闭其网络连接。请注意，执行该命令后，tun0 网卡的配置信息将被永久删除，无法恢复
```

####  路由配置

1、删除指定的路由，下述静态路由，通过 `route del -net 192.168.10.0 netmask 255.255.255.0 dev eth0` 指令进行删除：

```BASH
192.168.10.0    0.0.0.0         255.255.255.0   U     100    0        0 eth0
```


####  内核参数配置
1、内核参数 `rp_filter` 可能引发的问题

`rp_filter` 是 Linux 的安全功能，`rp_filter` 会在计算路由决策的时候，计算包的反向路由，也就是将包的源地址和目的地址对调再查找路由表。由本机或者其他设备流向 `clash0` 的 IP Packet 一般不会有问题，但是当 `rp_filter` 检查 `clash0` 流出的包时，由于这些包不会带有 `fwmark`，检查反向路由时不会走刚才定义的策略路由，导致 `rp_filter` 检查失败，包被丢弃；通常解决方法时关闭 clash0 NIC 的 rp_filter 功能，如下：
```bash
sysctl -w net.ipv4.conf.clash0.rp_filter=0
sysctl -w net.ipv4.conf.all.rp_filter=0
```

一般而言，将 `all.rp_filter` 设置为 `0` 是必须的；将 `all.rp_filter` 设置为 `0` 并不会将所有其他网卡的 `rp_filter` 一并关闭。此时其他网卡的 `rp_filter` 由它们各自的 `rp_filter` 控制


##  0x05  一些细节
构建基于 TUN 的包处理项目一定要注意防止路由回环，什么是路由回环呢？简言之，一个 packet 从 TUN 读出来后再写入 TUN，下次读还会将自己刚写入的 packet 读出来，如果设置默认路由是 tun 网卡，会导致死循环。这篇文章：[记录 Tun 透明代理的多种实现方式，以及如何避免 routing loop](https://chaochaogege.com/2021/08/01/57/)给出了解决LOOP的常用方法，可以结合自己的项目场景使用

笔者项目中使用了如下几种方式：

#### bind before connect
bind之后connect，routing 不会起作用，这样就能解决设置默认网关后导致的 routing loop

通过调试 leaf,我已经能够十分确认上面这句话的正确性

If I bind an interface before to connect, Does that mean the connect for outgoing traffic will use that interface I bind without follow the routing decision?

@nuclear yes, if you bind() to an interface before connect()'ing, the connection will go out through that interface. That is the whole point.

listen之前需要bind，决定listen到哪个网卡。
如果作为client去connect，在调用connect时bind会隐式发生。你也可以主动bind before connect，绕过路由选择，强迫出流量使用某个network interface

https://stackoverflow.com/a/4297381/7529562


为需要直连的ip设置单独的路由(删除掉默认路由)
同学，我又来了[无辜笑]。我看seal的全局模式可以转发整个系统的流量，最近自己在搞一个类似的透明代理工具，有个问题想请教下

如果自己设置默认路由全部流量都发到tun设备，自己read出来后交给userspace网络栈处理，再将处理后的数据包写入tun，之后的read会不会重新read出来自己刚写入的包呢
⁣
⁣我理解会，内核根据默认路由会重新送给tun，但那样死循环了呀
⁣
⁣但我看seal的全局模式transparent proxy从未遇到过这问题，这是怎么做到的呢

回复： read出来不会直接写入tun，是重新封包发给VPN server

vpn server是单独加了一条路由，而且发送包不是直接写tun包，是直接通过tcp/udp发出，socket可以绑定指定网卡发送

ip route del default
ip route add default dev wg0
ip route add 163.172.161.0/32 via 192.168.1.1 dev eth0
The Classic Solutions

为直连ip设置单独路由（不删除默认路由，只覆盖）
默认路由的作用是没有匹配到时走default，通过设置 0.0.0.0/1，让这条路由总是先于 default 命中。
再对要直连的 ip（这里是 163.172.161.0） 设置单独的路由，不需要删除原来的默认路由

ip route add 0.0.0.0/1 dev wg0
ip route add 128.0.0.0/1 dev wg0
ip route add 163.172.161.0/32 via 192.168.1.1 dev eth0

这种也叫做 0/1 128/1 trick。

但这trick有局限搜0/1

推荐阅读

Overriding The Default Route

##  0x05  TUN with DNS


##  0x0 参考
-   [网络编程学习：vpn 实现](https://www.jianshu.com/p/e74374a9c473)
-   [Listener and Dialer using a TUN and netstack.](https://github.com/costinm/tungate)
-   [tungate.go](https://github.com/costinm/tungate/blob/main/gvisor/cmd/tungate.go)
-   [在 Linux 上开 clash tun 模式 clash 是怎么劫持 dns 的](https://www.v2ex.com/t/880652)
-   [How does a socket know which network interface controller to use?](https://stackoverflow.com/questions/4297356/how-does-a-socket-know-which-network-interface-controller-to-use/4297381#4297381)
-   [基于TUN/TAP实现多个局域网设备之间的通讯](https://blog.csdn.net/qq_63445283/article/details/123779498)
-   [iOS network extension packet parsing](https://stackoverflow.com/questions/69260852/ios-network-extension-packet-parsing/69487795#69487795)
-   [网络协议之:haproxy的Proxy Protocol代理协议](https://developer.aliyun.com/article/938138)