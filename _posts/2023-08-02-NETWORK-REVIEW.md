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
本文回顾下网络的基础知识，主要是网卡、虚拟网卡，以及非常重要的 tap/tun 开发模式及应用构建相关。

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

1、tun 技术的应用场景

那么 TUN 机制可以来解决哪些场景下的问题呢？可以编写一个用户态程序，拿到 TUN 设备句柄，进行收发包操作：向其写入序列化的 `IP` 数据包，从中读取数据并还原成 `IP` 数据包进行处理，必要时需要取出其 payload 继续解析为相应传输层协议。在实际应用中，通常使用 TUN 技术的是 VPN 和透明代理，然而这两类程序在对待 TUN 中传递的 IP 数据包时通常有不同的行为，如下：
-       VPN：通常做网络层的封装；如将拿到的 IP 包进行加密和封装，然后通过某个连接传输到另一个网络中，在解封装和解密后，将 IP 包发送到该网络。在这个过程中，对 IP 包本身的修改是非常小的，不会涉及到整体结构的变动，通常仅会修改一下源 IP 和目标 IP ，做一下 NAT
-       透明代理： 通常是传输层的代理，在从 TUN 设备拿到 IP 包后，需要继续解析其 payload，还原出 TCP 或者 UDP 结构，然后加密和封装传输层 (TCP 或 UDP) 的 payload。网络层的 IP 和传输层的端口信息通常会作为该连接的元数据进行处理，使用额外的加密和封装手段，通常会修改五元组数据以确保数据能够在路由表中合理的转发

简单来说，VPN 不需要解析 IP 包的 payload，而透明代理需要解析出传输层信息并处理（特别是像 TCP 这样复杂的协议，对其处理更是需要非常小心）

2、使用 tun 技术来构建透明代理

对于透明代理的实现，通常有两种模式：在用户态实现网络栈，或者直接利用操作系统网络栈实现。用户态网络栈通常参考已有的开源实现（自己造肯定不现实），常用的有如下几个：

-       lwIP 是一个轻量级的 TCP/IP 栈实现（C），在占用超少内存的情况下，实现了完整的 TCP，被广泛应用到嵌入式设备中
-       gVisor 中的实现（go），gVisor 项目目的是为容器提供自己的应用程序内核

3、透明代理的开发流程（核心）

基本思路就是，用户态网络栈就是要不断通过协议解析，从 IPv4 数据包中不断解析出 TCP 流中的载荷数据；将传输层载荷通过不断的协议封装，拿到最终的 IPv4 数据包。核心包含如下：

-       从 TUN 往外读：从 TUN 设备所对应的句柄中读出了一段字节序列，便是需要处理的 IP 数据包（一般是 IPv4 协议，不过还是需要先根据字节序列的第一个字节进行判断）；如果判断为 IPv4 包，就将整个字节序列扔到 IPv4 的 Packet Parser 实现中，还原出 IPv4 数据包结构。根据 IPv4 Header 中的 protocol 字段，判断 payload 应该使用哪个上层协议解析；一般仅需要处理 ICMP、TCP、UDP 这三种协议，拿 TCP 为例，只需要将 IPv4 的 payload 扔到 TCP 的 Parser 中，即可取出我们想要的传输层载荷
-       向 TUN 写数据：写的过程其实就是读的过程反过来，拿到的是某个传输层协议的 payload，就拿 UDP 为例，根据该数据报的元信息，构建出完整的 UDP Header，然后将 payload 内容拼接进去；接下来构建 IPv4 Header，然后将 UDP 报文拼接进 IPv4 payload 中。在拿到 IPv4 数据包后，即可序列化为字节序列，写入 TUN 句柄即可（各层的 checksum 需要重新计算）
-       实际使用中需要考虑更多的 case，包括但不限于分片、丢包、重传、流量控制等等，TCP 作为一个极其复杂的传输层协议，有巨多情况需要考虑，很明显用上面的基本思路是非常繁琐并且难以使用的；上述用户态网络栈都提供了非常友好且直接的接口，可以直接创建一个 TCP/IP 网络栈实例，拿到两个句柄，一端负责读取和写入网络层 IP 数据包，另一端负责接收和写入传输层载荷，中间的复杂转换关系和特殊情况都被内部屏蔽掉了

4、操作系统网络栈
根据我们的需求，实际就是在 IPv4 和 TCP payload 之间进行转换，而操作系统的网络栈正好就有这个功能，我们无法简单的直接使用操作系统的网络栈代码，但是可以想办法复用操作系统网络栈提供的功能。TUN 在网络层已经打开了一个口子，还需要在传输层也打开一个口子，其实可以利用操作系统提供的 socket。

我们使用操作系统提供的 Socket 创建一个传输层的 Listener，将某个 IPv4 数据包的目标 IP 和目标端口修改为我们监听的 IP 和端口，然后通过 TUN 将该 IPv4 数据包注入到操作系统的网络栈中，操作系统就会自动的进行相应的解析，并将所需要的传输层 payload 通过前面创建的 Socket 发送给 Listener，由此便利用操作系统网络栈完成了 “往外读” 的操作。

对于 “向里写” 的操作，只需要向刚刚创建的传输层连接句柄写入即可，操作系统的网络栈同样会进行相应的封包，最后形成 IPv4 数据包。很明显，需要考虑反向的数据包，当向传输层连接的句柄中写入数据、操作系统的网络栈封包时，源 IP 和源端口会被视为新的目标 IP 和目标端口，因为我们需要使返回的 IPv4 数据包能够被 TUN 接口捕获到，在上面步骤中就不能只修改目标 IP 和目标端口，同时还要修改源 IP 和源端口，源 IP 应该限制为 TUN 网段中的 IP。

工作流程
在利用操作系统网络栈时，通常是以下步骤，这里拿 TCP 协议举例。

在我们的例子中， TUN 网络的配置为 198.10.0.1/16，主机 IP 为 198.10.0.1，代理客户端监听 198.10.0.1:1313，App 想要访问 google.com:80，自定义的 DNS 服务返回 google.com 的 Fake IP 198.10.2.2。

1. Proxy 创建 TCP Socket Listener

这里首先要在系统网络栈的传输层开个口子，创建一个 TCP Socket Listener，监听 198.10.0.1:1313

2. 某 App 发起连接

当某需要代理的 App 发起连接，访问 google.com:80，我们通过自定义的 DNS 服务返回一个 Fake IP (198.10.2.2)，使流量被路由到 TUN 设备上。

当然这里也可以不使用 Fake IP 方式来捕获流量，通过配置路由规则或者流量重定向也可以将流量导向 TUN 设备，不过 Fake IP 是最常用的方法，所以这里以此举例。

App
Kernel
TCP connect(198.10.2.2:80)
198.10.0.1:34567 -> 198.10.2.2:80
App
Kernel
3. 将 TUN 读取到的 IPv4 解析为 TCP 载荷数据

TUN 设备捕获到流量，也就是 IPv4 数据包，在读取出来后，需要利用系统网络栈解析出 TCP 载荷数据。

这一步，需要将读取到的 IPv4 数据包进行修改，也就是我们上面说的 源 IP、源端口，目标 IP 和目标端口，还有相应的 checksum 也需要重新计算。修改的目的是让 IPv4 数据包通过 TUN 注入到操作系统网络栈后，能够被正确路由并通过一开始监听的 TCP Socket 将最里层的 TCP payload 返还给我们。

Proxy
Kernel
TUN read IPv4
1
198.10.0.1:34567 -> 198.10.2.2:80
Modify src_ip:src_port, dst_ip:dst_port
2
198.10.2.2:80 -> 198.10.0.1:1313
TUN write IPv4
3
System NetStack process
TCP Socket read
198.10.0.1:1313
4
Got TCP payload
And peer_addr as src_ip:src_port
Proxy
Kernel
这里为了方便，直接将源 IP 和源端口设置为初始的目标 IP 和目标端口，在实际编程时，有更多的设置策略，也就是 NAT 策略。

4. 代理客户端请求代理服务器

此时代理客户端已经拿到了请求的真实 TCP 载荷，并且可以通过获取 TCP 连接的 peer 信息得到在第 3 步修改的源 IP 和源端口，通过这些信息可以通过查 NAT 表得到 App 真正想要访问的 IP 和 端口（甚至可以通过查 DNS 请求记录拿到域名信息），因此代理客户端可以根据自己的协议进行加密和封装等操作，然后发送给代理服务端，由代理服务端进行真实的请求操作。

Proxy Client
Proxy Server
Google
Wrap request
with connection meta info
Wrapped Request
1
Request
2
Response
3
Wrapped Response
4
Unwrap response
Proxy Client
Proxy Server
Google
5. 将返回数据封包回 IPv4 并写入 TUN

通过代理客户端与代理服务端、代理服务端与谷歌的通信，拿到谷歌真正的返回数据，现在需要重新封装回 IPv4 数据包，还是利用系统网络栈：将数据写入 TCP Socket (198.10.0.1:1313) 中，便可以在 TUN 侧拿到封装好的 IPv4，就是这么轻松。

Proxy
Kernel
TCP Socket write payload
198.10.0.1:1313
1
System NetStack process
TUN read IPv4
2
198.10.0.1:1313 -> 198.10.2.2:80
Restore src_ip:src_port, dst_ip:dst_port
3
198.10.2.2:80 -> 198.10.0.1:34567
TUN write IPv4
4
Proxy
Kernel
6. App 拿到返回数据

App
Kernel
198.10.0.1:34567 <- 198.10.2.2:80
TCP read payload
1
App
Kernel
上面的过程便是利用操作系统网络栈完成 IPv4 到 TCP 载荷数据及其反方向转变的过程。通过这种办法，可以充分利用操作系统的实现，都是饱经检验，质量可靠，且满足各种复杂情况。但是也有缺点，数据需要拷贝多次，增加了性能损耗和延迟。

NAT 策略
我这里想说的 NAT 策略不是指常说的那四种 NAT 类型，当然你可以去实现不同的 NAT 类型来满足各种各样的需求，但那是更深入的话题，不在本文讨论。

在刚刚的流程的第 3 步中，你应该发现对源 IP 和源端口的修改是有限制的，我们需要将 IP 限定为 TUN 网段，从而使返回的数据包可以重新被 TUN 设备捕获。但是这种限制是非常宽松的，在我们的例子对 TUN 设备网段的配置中，你有 2^16 个 IP 可供选择，每一个 IP 又有 2^16 个端口可供选择。

但是如果你仔细观察，你会发现上面的例子并没有充分利用这些资源，我们仅仅是将 Fake IP 作为源 IP、真实目标端口作为源端口，而这个 IP 的其他端口都被闲置了。同时我也在其他人写的某些程序中发现，他们仅选择一个 IP 设置为源 IP，通过合理的分配该 IP 的端口作为源端口，在这种情况下， TUN 网段中其余的 IP 资源就被浪费了。

以上两种 NAT 策略在个人电脑上没啥问题，但是如果代理客户端运行在网关上，网络中访问的 IP 数量超过网段中 IP 数量上限，或者 hash(ip:port) 数量超过端口总数 (2^16)，就会难以继续分配 NAT 项。因此我们应该专门编写一个 NAT 管理组件，合理分配 IP 和端口资源，争取做到利用最大化。

防止环路
抛开事实不谈，如果我们想要代理全部流量，就是要通过路由规则将所有流量导向我们的 TUN 设备，这是很直观且朴素的想法，就像下面的命令一样单纯：

1
sudo route add -net 0.0.0.0/0 dev tun0
如果你真的这么写，你就会发现你上不了网了。这是因为出现了环路。

如果稍微思考一下，你就会发现，虽然我们想要代理所有流量，但是代理客户端与代理服务端的流量却是需要跳过的，如果用上面的路由，就会导致代理客户端发出的流量经过路由然后从 TUN 重新回到了代理客户端，这是一个死环，没有流量可以走出去。流量只近不出，来回转圈，你的文件打开数爆炸，操作系统不再给你分配更多的句柄，数据来回拷贝，你的 CPU 风扇猛转，电脑开始变卡。

这是我们不想看到的，需要采取一些措施避免环路的产生。在实践中有不少方法可以避免这种情况的发生，例如通过合理的配置路由规则，使连接代理服务器的流量可以顺利匹配到外部网络接口。只不过这种方法不够灵活，如果代理服务器 IP 发生变化则需要及时改变路由规则，非常麻烦，所以我们接下来介绍其他的方法。

Fake IP
Fake IP 就是我们上面例子中用到的方法，这是一种限制进入流量的方法。基本思路是自己实现一个 DNS 服务器，对用户的查询返回一个假的 IP 地址，我们可以将返回的 IP 地址限制为 TUN 设备的网络段，这样应用发起的流量其实便是发给 TUN 网络的流量，自然的被路由匹配，而无需像前面那样路由全部的流量，其余的流量包括代理客户端发起的请求便不会被路由，可以保证不产生环路。

当代理客户端需要知道应用真正想要请求的地址时，就通过一些接口向自己实现的 DNS 服务器进行反向查询即可。

策略路由
通过前面的分析，可以发现产生环路是因为代理客户端本身发出的流量被系统路由到 TUN 设备导致的，因此我们可以想办法让代理客户端本身发起的流量不走 TUN 而是从真实的物理网络接口出去。

在 (类)Unix 系统中，可以对代理客户端的流量打上 fwmark 防火墙标记，然后通过策略路由使带有标记的流量走单独的路由表出去，从而绕过全局的流量捕获。

cgroup

cgroup 是 Linux 内核的功能，可以用来限制、隔离进程的资源，其中 net_cls 子系统可以限制网络的访问。在网络控制层面，可以通过 class ID 确定流量是否属于某个 cgroup，因此可以对来自特定 cgroup 的流量打上 fwmark，使其能够被策略路由控制。

我们可以创建一个用于绕过代理的 cgroup ，对该 cgroup 下进程的流量使用默认的路由规则，而不在该 cgroup 的其余进程的流量都要路由到 TUN 设备进行代理。

一些其他的知识
TUN 与 TAP 的区别
TAP 在 2 层，读取和写入的数据需要是以太帧结构

TUN 在 3 层，读取和写入的数据需要是 IP 数据包结构

IP 等配置
在给网卡配置 IP 时，其实是修改内核网络栈中的某些参数，而不是修改网卡。虽然网卡也会有一些可供修改的配置项，但一般情况是通过其他方法进行修改的 (驱动程序)。

物理网卡与虚拟网卡的区别
物理网卡会有 DMA 功能，在启用 DMA 时网卡和网络栈 (内存中的缓冲区) 的通讯由 DMA 控制器管理，因此性能更高延迟也更低。

如何创建 TUN 设备
在 Linux 下一切皆文件，/dev/net/tun 是特殊的字符 (char) 设备文件，通过打开这个文件获得一个文件句柄，然后通过 ioctl() 系统调用对其进行配置。在这里可以选择打开 TUN 设备还是 TAP 设备，可以设置设备名称。

详见：Network device allocation

与 BPF 的关系
BPF 是一种高级数据包过滤器，可以附加到现有的网络接口，但其本身不提供虚拟网络接口。 TUN/TAP 驱动程序提供虚拟网络接口，可以将 BPF 附加到该接口。

扩展阅读
https://www.kernel.org/doc/html/latest/networking/tuntap.html
https://github.com/xjasonlyu/tun2socks
https://github.com/eycorsican/go-tun2socks
https://github.com/gfreezy/seeker
https://github.com/shadowsocks/shadowsocks-rust
https://www.wintun.net/

////


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

如何向 tun/tap 发送 / 读取数据呢？一般有如下方法：

1. 应用程序可以通过标准的 Socket API 向 Tun/Tap 接口发送 IP 数据包，就好像对一个真实的网卡进行操作一样。除了应用程序以外，操作系统也会根据 TCP/IP 协议栈的处理向 Tun/Tap 接口发送 IP 数据包或者以太网数据包，例如 ARP 或者 ICMP 数据包。Tun/Tap 驱动程序会将 Tun/Tap 接口收到的数据包原样写入到 `/dev/net/tun` 字符设备上，处理 Tun/Tap 数据的应用程序如 VPN 程序可以从该设备上读取到数据包，以进行相应处理

2. 应用程序也可以通过 `/dev/net/tun` 字符设备写入数据包，这种情况下该字符设备上写入的数据包会被发送到 Tun/Tap 虚拟接口上，进入操作系统的 TCP/IP 协议栈进行相应处理，就像从物理网卡进入操作系统的数据一样

3. 在内核网络协议栈中处理该数据包时，根据目标 IP 和路由表判断，需要从该网络接口设备发出时，与该设备绑定的用户态进程的文件描述符将 `read` 系统调用将读到数据

4. 用户态进程调用 `write` 系统调用向与该设备绑定的文件描述符写数据时，该设备会将写入的数据流当做网络数据包发送到内核网络协议栈中


还有一个问题，内核通过什么信息将数据报文转发到某个 tun 虚拟网卡上呢？

内核将数据报文转发到某个 tun 虚拟网卡上，主要是根据路由表中的路由规则和目的 IP 地址进行匹配。当内核接收到一个数据报文时，它会首先检查该数据报文的目的 IP 地址，然后查找路由表中与该目的 IP 地址匹配的路由规则。如果找到了匹配的路由规则，内核就会根据该路由规则指定的下一跳地址或接口将数据报文转发出去。如果路由规则中指定了 tun 虚拟网卡作为下一跳接口，内核就会将数据报文转发到该虚拟网卡上。此时，数据报文就会被封装成一个新的数据包，并通过 tun 虚拟网卡发送到用户空间的应用程序中进行处理。

需要注意的是，为了使内核能够正确地将数据报文转发到 tun 虚拟网卡上，应用程序需要正确地配置虚拟网卡的 IP 地址和子网掩码，并将该虚拟网卡添加到路由表中

####    通过 VPN 收发包深刻理解 tun 工作机制（数据转发原理）
openvpn 基于 tun/tap 虚拟网卡驱动了隧道包封装。openvpn 启动后，tap 网卡及路由配置如下：

![tap0](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tap-flow/tap0.png)

![route](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tap-flow/route.png)


1、虚拟网卡和物理网卡的通信

从上面的路由表可知，网段是 `10.8.0` 的 IP 数据包都会被发到 tap0 虚拟网卡，此网卡样被系统内核的网络设备管理系统管理；站在管理系统的视角来看，虚拟网卡和实际网卡一样，都是网络设备。网络设备收到的数据，都会发到上层协议栈处理，而协议栈发出的数据，同样也会发动对应的网络设备。协议栈根据目的 IP（路由表）决定将数据包路由到哪一个网卡，那么虚拟网卡下层的设备是什么？

因为数据包肯定是从物理网卡出去的，所以虚拟网卡发出去的数据包最终要发到物理网卡。而基于 tun/tap 的虚拟网卡，其实现是通过用户态程序来将数据包转交。而内核态和用户态的数据交互，在 Linux 下面的方式，比如通过 Socket 或者文件系统管理的文件交互或者设备文件。Linux 下面的设备文件分为块设备、网络设备、字符设备，块设备主要是存储类型的设备；网络设备使用 socket 访问；而字符设备就是以字节流形式通讯的 I/O 设备。这里 tun/tap 驱动程序选择的是以字符设备（字符设备文件）作为传输通道。

参考下图（红线）：

![tap](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tap-flow/tap2.png)

图中虚拟网卡将数据发送到字符设备文件，然后用户程序读取到文件内容后，进行处理，处理完成以后将其发往协议栈，最后经由物理网卡发送出去（当然经过用户程序处理完成后的包的目的 ip 要匹配合适的路由才行）。小结下，虚拟网卡既负责实现物理网卡的功能，又负责实现和字符设备的交互。所以 tun/tap 驱动程序实际上分成了两块：网卡驱动和字符设备驱动。


2、网络传输包格式
openvpn 系统内是直接使用局域网 IP 通信的，局域网 IP 数据包不能直接在公网上路由，需要封装为公网 IP 才可以传输，原始数据包如下：

![raw](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tap-flow/packet1.png)

上面的数据包经过协议栈（路由表）后自然会转发到虚拟网卡 tap0，再由 tap0 将其发送到 openvpn，然后 openvpn 内部将该完整的数据包进行加密，再封装上公网 IP 包头，数据包示意图如下：

![raw](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tap-flow/packet2.png)

openvpn 如何知道内网和公网 IP 的映射关系？这点在 openvpn 客户端建立连接的时候就已经确认了。因为建立连接的时候 openvpn 会指定客户端的内网 IP，并且由于能够收到客户端的握手信息，所以能够从包中提取到客户端实际的公网 IP


3、接收机制

接收机制如下图所示，蓝线是公网 IP，红线是解密后的内网 IP 包。经过两次协议栈的路由，最终将数据发到用户程序

![recv](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tap-flow/vpn-recv.png)

4、发送机制

发送机制如下图所示，红线是内网 IP，蓝线是加密过后的公网 IP 包

![send](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tap-flow/vpn-send.png)

5、服务器端的转发机制

![trans](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tap-flow/vpn-transmit.png)

服务端转发的时候并不需要经过 tap 虚拟网卡，因为数据包首先会到 openvpn，openvpn 内部会得知该数据包是要转发到另外一台局域网客户端的，所以根据建立连接时的公网 IP 与内网 IP 映射关系，将公网 IP 包的目的 IP 改变即可。所以蓝线部分直接就将数据包发送出去了（可直接将服务器端的 tap0 虚拟网卡关闭，一段时间后客户端之间仍然可以正常通信，所以在转发机制这边并不需要走虚拟网卡）


##  0x04    tap/tun 机制的应用场景

####    1、使用 Tun/Tap 创建点对点隧道

![tun](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tun-app-1.png)

如上图使用 tun 机制构建点对点的隧道。左侧主机应用程序发送到 Tun 虚拟设备上的 IP 数据包被 VPN 程序通过字符设备接收，然后再通过一个 TCP or UDP 隧道发送到右侧 VPN 服务器上，VPN 服务器将隧道负载中的原始 IP 数据包写入字符设备，这些 IP 包就会出现在右侧的 Tun 虚拟设备上，最后通过操作系统协议栈和 socket 接口发送到右侧的应用程序上（在网络中传输的报肯定是正常的 IP 包，VPN 的包要被封装起来，在 TUN 应用中去解包）

如果采用 tap 隧道实现，那么隧道的负载将是以太数据帧而不是 IP 数据包，而且还会传递 ARP 等广播数据包：

![tun](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/linux-tap-tunnel.png)


####    2、使用 Tun/Tap 隧道绕过防火墙

![tunnel](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/linux-access-internet-via-tunnel.png)

结合路由规则和 iptables 规则，可以将 VPN 服务器端的主机作为连接外部网络的网关，以绕过防火墙对客户端的一些外部网络访问限制。如下图所示，防火墙规则允许客户端访问主机 IP2，而禁止访问其他 Internet 上的节点。通过采用 Tun 隧道，从防火墙角度只能看到被封装后的数据包，因此防火墙认为客户端只是在访问 IP2，会对数据进行放行。而 VPN 服务端在解包得到真实的访问目的后，会通过路由规则和 iptables 规则将请求转发到真正的访问目的地上，然后再将真实目的地的响应 IP 数据包封装进隧道后原路返回给客户端，从而达到绕过防火墙限制的目的


####    3、使用 Tap 隧道桥接两个远程站点

使用 tap 建立二层隧道将两个远程站点桥接起来，组成一个局域网。对于两边站点中的主机来说，访问对方站点的主机和本地站点的主机的方式没有区别，都处于一个局域网 `192.168.0.0/24` 中

详细配置参考此文：[Linux Tun/Tap 介绍](https://www.zhaohuabing.com/post/2020-02-24-linux-taptun/)

![bridge](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/linux-bridge-tunnel.png)


####    4、隧道通信
tap/tun 最常见的应用也就是用于隧道通信，如 VPN，包括 tunnel 和应用层的 IPsec 等。比较有名的是下面两个：
1.      [openvpn](https://openvpn.net/)
2.      [VTun](https://vtun.sourceforge.net/)

此外，还有一些基于 golang 实现的 VPN 项目，如下：
-       [vtun](https://github.com/net-byte/vtun)：原理是利用 TUN 设备（虚拟网卡）接收数据，在客户端和服务端之间进行数据加密转发


大部分的 VPN 实现都是将客户端虚拟网卡（tun）的流量转发到 Linux 服务器虚拟网卡, 然后将服务器虚拟网卡流量再转发到真实的物理网卡上，传输层可以使用 UDP or TCP, 然后对流量进行加密

####    5、透明代理
目前很多开源项目都是使用 tun 来实现透明代理，非常方便，一般的原理如下：


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
-   [云原生虚拟网络 tun/tap & veth-pair](https://www.luozhiyun.com/archives/684)