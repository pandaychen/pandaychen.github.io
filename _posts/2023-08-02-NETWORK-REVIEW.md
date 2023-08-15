---
layout:     post
title:      重拾 Linux 网络（二）：网卡 / 虚拟网卡、tap/tun 那些事
subtitle:
date:       2023-08-02
author:     pandaychen
catalog:    true
tags:
    - network
    - tap
    - tun
---


##  0x00    前言
本文回顾下网络的那些事

##  0x01    物理网卡 && 虚拟网卡

![nic-vnic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nic-vnic.gif)

####    物理网卡
物理网卡设备的工作流程如下：

![nic-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nic-work-flow.png)

所有物理网卡收到的包会交给内核的 Network Stack 处理，然后通过 Socket API 通知给用户程序。

####    虚拟网卡
相比于物理网卡负责内核网络协议栈和外界网络之间的数据传输，** 虚拟网卡的两端则是内核网络协议栈和用户空间（用户的二进制程序）**，它负责在内核网络协议栈和用户空间的程序之间传递数据，工作流程如下图：
-   发送到虚拟网卡的数据来自于用户空间，然后被内核读取到网络协议栈中
-   内核写入虚拟网卡准备通过该网卡发送的数据，目的地是用户空间


![vnic-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/vnic-work-flow.png)

还记得前文描述的 `tun2socks` 这种软件么，就是走了虚拟网卡与应用分流的逻辑

tun/tap 设备与物理网卡的区别，如下图所示:

-   对于硬件网络设备而言，一端连接的是物理网络，一端连接的是网络协议栈
-   对于 tun/tap 设备而言，一端连接的是应用程序（通过 字符设备文件 `/net/dev/tun`），一端连接的是网络协议栈

![diff](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nic-vnic-1.png)


####    物理网卡与虚拟网卡的关系


####    tun 与 tproxy 的区别
作为能同时代理 TCP 和 UDP 的模式，tproxy 和 tun 有哪些区别呢？参考 [此文](https://lancellc.gitbook.io/clash/start-clash/clash-udp-tproxy-support)：

```text
What's different between TProxy mode and TUN mode?
There's no big difference in user-side.
In developer side, Tproxy is a proxy, although it is not perceived by the application. And TUN is a gatway, application knows there is a gateway, and traffic must pass through it.
```

通过上文描述大概可知，tproxy 是一种代理模式（应用不能轻易感知到），而 tun 是一个类似于网关的东西，所有的数据流都需要经过它；思考下，[重拾 Linux 网络（一）：iptables](https://pandaychen.github.io/2023/08/01/A-IPTABLES-REVIEW/) 文中介绍了 tproxy 配合 sockv5 客户端的模式，sockv5 客户端是在 tproxy 之后的，而 tun 一般部署在入口处，紧接着网卡的位置

最后的问题是，tun 和 tproxy 能混用吗？

另外还有一种说法是，对于 `TCP` ，tun 和 tproxy 速度差不多。但是对于 `UDP`， tproxy 速度只有 tun 一半，[来源](https://v2ex.com/t/849307)


##  0x02    tun/tap 基础知识

用网络虚拟化的角度看，tun/tap 设备属于虚拟网卡（简称 vNIC），一端连着网络协议栈，另一端连着用户态程序。tun/tap 设备可以将 TCP/IP 协议栈处理好的网络包发送给任何一个使用 tun/tap 驱动的进程，由用户进程处理后发送给物理链路中，这种特性很适合于数据压缩、加密等等。OpenVPN、Vtun、flannel 等都是基于此实现隧道封装的。

tun/tap 设备是利用 Linux 设备文件实现内核态和用户态的数据交互，设备驱动是内核态和用户态的一个接口，访问设备文件则会调用设备驱动的相应例程。

tun 设备和 tap 设备的原理相同，区别在于:
-   tun 设备的 `/dev/tunX` 文件收到的是 `IP` 包，只能工作在 L3，无法与物理网卡桥接
-   tap 设备的 `/dev/tapX` 文件收到的是链路层包，可以与物理网卡桥接
-   TAP 等同于一个以太网设备，它操作第二层数据包如以太网数据帧
-   TUN 模拟了网络层设备，操作第三层数据包比如 `IP` 数据封包
-   操作系统通过 TUN/TAP 设备向绑定该设备的用户空间的程序发送数据，反之，用户空间的程序也可以像操作硬件网络设备那样，通过 TUN/TAP 设备发送数据。在后种情况下，TUN/TAP 设备向操作系统的网络栈投递（注入）数据包，从而模拟从外部接受数据的过程

下图描述了 Tap/Tun 的工作原理：

![tun](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/linux-tuntap.png)

####    核心知识：tun 开发相关
tap/tun 是设备是用户空间软件和系统网络栈之间的一个通道，只是它们工作的层级不一样。TUN 是内核提供的三层虚拟网络设备，由软件实现来替代真实的硬件，相当于在系统网络栈的 `L3`（网络层）位置开了一个口子，将符合条件（路由匹配命中）的三层数据包交由相应的用户空间软件来处理，用户空间软件也可以通过 TUN 设备向系统网络栈注入数据包。

那么 TUN 机制可以来解决哪些场景下的问题呢？我们可以编写一个用户态程序，拿到 TUN 设备句柄，进行收发包操作：向其写入序列化的 `IP` 数据包，从中读取数据并还原成 `IP` 数据包进行处理，必要时需要取出其 payload 继续解析为相应传输层协议。



还有若干细节，可以参考此文：[使用 TUN 的模式](https://zu1k.com/posts/coding/tun-mode/)
####    tun 收包

使用 [water](https://github.com/songgao/water) 来实现收包，代码如下：

```GO
import (
        "log"
        "github.com/songgao/water"
)

func main() {
        ifce, err := water.New(water.Config{
                DeviceType: water.TUN,
        })
        if err != nil {
                log.Fatal(err)
        }

        log.Printf("Interface Name: %s\n", ifce.Name())

        packet := make([]byte, 2000)
        for {
                n, err := ifce.Read(packet)
                if err != nil {
                        log.Fatal(err)
                }
                log.Printf("Packet Received: % x\n", packet[:n])
        }
}
```

运行上述程序，需要 up 虚拟网卡，如下命令（默认开启的虚拟网卡为 `tun0`），然后另外开启终端 `ping 10.1.0.20`，程序输出：

```bash
ifconfig tun0 10.1.0.10 up
```

```text
2023/08/15 11:40:15 Interface Name: tun0
2023/08/15 11:40:22 Packet Received: 60 00 00 00 00 08 3a ff fe 80 00 00 00 00 00 00 aa e4 09 b6 7d d1 00 8d ff 02 00 00 00 00 00 00 00 00 00 00 00 00 00 02 85 00 4a 3e 00 00 00 00
```

####    相关命令

1、虚拟网卡操作相关

```bash
# add tun
ip tuntap add tun0 mode tun
# del tun
ip tuntap del tun0 mode tun
```

2、向虚拟网卡发送数据

```bash
ping 10.0.0.10 -I tun0
curl --interface tun0 -O target_url
```



####    tun：透明代理的转发原理
1.  配置网络路由：在操作系统中配置路由规则，将目标 IP 地址与 TUN 设备关联起来。这样，当有数据包要发送到目标 IP 地址时，操作系统会将其路由到 TUN 设备
2.  拦截数据包：TUN 设备会拦截经过物理网卡的数据包，并将其传递给用户空间的代理程序
3.  代理程序处理：** 代理程序会接收到拦截的数据包，并根据需要进行处理，例如修改数据包的目标 IP 地址或端口等 **
4.  发送数据包：代理程序将处理后的数据包发送回操作系统
5.  路由转发：** 操作系统根据路由规则，将代理程序发送的数据包路由到正确的目标 IP 地址 **
6.  发送到物理网卡：最后操作系统将数据包发送到物理网卡，以便将其发送到网络中

通过上述过程，TUN 透明代理可以在不需要修改客户端配置的情况下，将数据包从物理网卡转发到代理程序进行处理，并将处理后的数据包发送回操作系统，最终发送到网络中。这样，代理程序可以实现对数据包的修改、过滤或其他操作，从而实现透明代理的功能



##  0x03    tap/tun 收发包机制

本小节内容主要参考：[tun/tap 虚拟网卡收发机制解析](https://blog.csdn.net/zhou307/article/details/102806500)

tap 类型的虚拟网卡是 tun/tap 驱动程序实现的，tap 表示虚拟的是以太网设备，tun 表示虚拟的是点对点设备，这两种设备针对网络包实施不同的封装。前文介绍的 `tun2socks` 基于 tun 虚拟网卡驱动了隧道包封装。本小节关注下虚拟网卡和物理网卡的转发原理是什么（数据包最终肯定要从物理网卡才能流出）以及通信包是如何封装的

##  0x04    tap/tun 的应用场景


##  0x0  参考
-   [理解 Linux 虚拟网卡设备 tun/tap 的一切](https://www.junmajinlong.com/virtual/network/all_about_tun_tap/)
-   [A simple TUN/TAP library written in native Go.](https://github.com/songgao/water)
-   [tun/tap 虚拟网卡收发机制解析](https://blog.csdn.net/zhou307/article/details/102806500)
-   [Linux Tun/Tap 介绍](https://www.zhaohuabing.com/post/2020-02-24-linux-taptun/)
-   [Linux 中的 Tun/Tap 介绍](https://blog.haohtml.com/archives/31687)
-   [seeker](https://github.com/gfreezy/seeker)
-   [Surge 官方中文指引：理解 Surge 原理](https://manual.nssurge.com/book/understanding-surge/cn/)
-   [CentOS 下的 TUN 与 TAP 应用](https://liqiang.io/post/tap-and-tun-application-in-cetnoss1)
-   [使用 TUN 的模式](https://zu1k.com/posts/coding/tun-mode/)
-   [A simple VPN written in Go.](https://github.com/net-byte/vtun)