---
layout:     post
title:      重拾 Linux 网络（四）：NAT
subtitle:   NAT 与 内网穿透
date:       2023-09-02
author:     pandaychen
catalog:    true
tags:
    - network
    - NAT
---


##  0x00    前言
NAT（Network Address Translation，网络地址转换），也叫做网络掩蔽或者 IP 掩蔽。NAT 是一种网络地址翻译技术，主要是将内部的私有 IP 地址（private IP）转换成可以在公网使用的公网 IP（public IP）

本文介绍下笔者在项目中解决 UDP 透明代理的 NAT 实现及相关背景知识，以及内网穿透的原理及应用

NAT 分为两类：
-   基础 NAT：网络地址转换（Network Address Translation）
-   NAPT：网络地址端口转换（Network Address Port Translation）

####    基础 NAT
基础 NAT 仅对网络地址进行转换，要求对每一个当前连接都要对应一个公网 IP 地址，所以需要有一个公网 IP 池；基础 NAT 内部有一张 NAT 表以记录对应 IP 映射关系

####    NAPT
NAPT 需要对网络地址和端口进行转换，这种类型允许多台主机共用一个公网 IP 地址，NAPT 内部同样有一张 NAT 表，并标注了端口，以记录对应关系，如下：

| 内网 ip | 外网 ip |
| :-----:| :----: |
| 192.168.1.1:1025 |1.2.3.4:1025 |
| 192.168.1.1:3333 |1.2.3.5:10000 |
| 192.168.1.12:7788 |1.2.3.6:32556|

本文主要讨论 NAPT 相关内容

##  0x01    NAPT

NAPT 分类如下：

-   完全锥型
-   受限锥型
-   端口受限锥
-   对称型

![type](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nat/nat-type.png)

####    完全锥型（full cone nat）
从同一个内网地址端口 `192.168.1.1:7777` 发起的请求都由 NAT 转换成公网地址端口 `1.2.3.4:10000`；反方向，`192.168.1.1:7777` 可以收到任意外部主机发到 `1.2.3.4:10000` 的数据包（注意，不关心目标的地址和端口）

![full-cone-nat](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nat/full-cone-nat.png)

####    受限锥型（restricted  cone nat）
受限锥型也称地址受限锥型，在完全锥型的基础上，对 ip 地址进行了限制；从同一个内网地址端口 `192.168.1.1:7777` 发起的请求都由 NAT 转换成公网地址端口 `1.2.3.4:10000`，其访问的服务器为 `8.8.8.8:123`，只有当 `192.168.1.1:7777` 向 `8.8.8.8:123` 发送一个报文后，`192.168.1.1:7777` 才可以收到 `8.8.8.8` 发回 `1.2.3.4:10000` 的报文

![restrict](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nat/restrict-nat.png)

####    端口受限锥型（port restricted cone nat）
在受限锥型的基础上，对端口也进行了限制；从同一个内网地址端口 `192.168.1.1:7777` 发起的请求都由 NAT 转换成公网地址端口 `1.2.3.4:10000`，其访问的服务器为 `8.8.8.8:123`，只有当 `192.168.1.1:7777` 向 `8.8.8.8:123` 发送一个报文后，`192.168.1.1:7777` 才可以收到 `8.8.8.8:123` 发回 `1.2.3.4:10000` 的报文

![restrict-port](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nat/restrict-port-nat.png)

####    对称型（symmetric nat）
在对称型 NAT 中，只有来自于同一个内网地址端口 、且针对同一目标地址端口的请求才被 NAT 转换至同一个公网地址端口，否则的话，NAT 将为之分配一个新的公网地址端口。内网地址端口 `192.168.1.1:7777` 发起请求到 `8.8.8.8:123`，由 NAT 转换成公网地址端口 `1.2.3.4:10000`，随后内网地址端口 `192.168.1.1:7777` 又发起请求到 `9.9.9.9:456`，NAT 将分配新的公网地址端口 `1.2.3.4:10001`

小结下：在 锥型 NAT 中，映射关系和目标地址端口无关，而在对称型 NAT 中则与目标地址端口有关。** 锥型 NAT 正因为其于目标地址端口无关，所以网络拓扑是圆锥型的 **

![symmetric-nat](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nat/symmetric-nat.png)

####    锥型 NAT（cone-nat）
![cone-nat](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nat/cone-nat.png)


##  0x02 NAT 的工作流程

1. 发送数据

当一个 TCP/UDP 的请求 (`192.168.1.1:7777` => `8.8.8.8:123`) 到达 NAT 网关 (`1.2.3.4`)，由 NAT 网关修改报文的源地址和源端口（包含 checksum 等），随后再发往目标服务器，计 `192.168.1.1:7777` => `1.2.3.4:10000` => `8.8.8.8:123`

2. 接收数据

随后 `8.8.8.8:123` 返回响应数据到 `1.2.3.4:10000`，NAT 查询映射表，修改目的地址和目的端口以及相应 checksum，再将数据返回给真实的请求方客户端，`8.8.8.8:123` => `1.2.3.4:10000` => `192.168.1.1:7777`

3. 其他协议

不同协议的工作特性不同，其和 TCP/UDP 协议的处理方式不同；比如 ICMP 协议工作在 IP 层（无端口），NAT 以 ICMP 报文中的 identifier 作为标记，以此来判断这个报文是内网哪台主机发出的

4. 映射老化时间

建立了 NAT 映射关系后，这些映射何时失效？不同协议有不同的失效机制，比如 TCP 的通信在收到 RST 过后就会删除映射关系，或 TCP 在某个超时时间后也会自动失效，而 ICMP 在收到 ICMP 响应后就会删除映射关系，当然超时后也会自动失效

##  0x03    NAT 类型检测
探测 NAT 的类型是 NAT 穿透中的第一步，可以通过客户端和两个服务器端的交互来探测 NAT 的工作类型，以下是来源于 STUN [协议](https://tools.ietf.org/html/rfc3489) 的探测流程图

![check](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nat/nat-checking.png)

![check](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nat/udp-nat-checking.png)

1.  客户端使用同一个内网地址端口分别向主服务器（server1）和协助服务器 (server2) 发起 UDP 请求，主服务器获取到客户端出口地址端口后，返回给客户端，客户端对比自己本地地址和出口地址是否一致，如果是则表示处于 Open Internet 中
2.  协助服务器同样也获取到了客户端出口地址端口，将该信息转发给主服务器，同样将该信息返回给客户端，客户端对比两个出口地址端口 (主服务器返回与协助服务器返回的) 是否一致，如果是则表示处于 Symmetric NAT 中
3.  客户端再使用不同的内网地址端口分别向主服务器和协助服务器发起 UDP 请求，主服务器和协助服务器都可以获得一个新的客户端出口地址端口，协助服务器将客户端出口地址端口转发给主服务器
4.  主服务器向协助服务器获取到的客户端出口地址端口发送 UDP 数据，客户端如果可以收到数据，则表示处于 Full-Cone NAT 中
5.  主服务器使用另一个端口，向主服务器获取到的客户端出口地址端口发送 UDP 数据，如果客户端收到数据，则表示处于 Restricted NAT 中，否则处于 Restricted-Port NAT 中

不过，上述只是理论可行的方法，实际网络往往都更加复杂，导致无法准确的探测 NAT 类型

##  0x04    UDP 打洞
NAT 穿透的思想在于：如何复用 NAT 中的映射关系。在锥型 NAT 中，同一个内网地址端口访问不同的目标只会建立一条映射关系，所以可复用，而对称型 NAT 不行。同时由于 TCP 工作比较复杂，在 NAT 穿透中存在一些局限性，所以在实际场景中 UDP 穿透应用更广泛，本小节先描述下 UDP 穿透的原理和流程：
这里以 Restricted-Port NAT 类型作为例子，因为其使用得最为广泛，同时权限也是最为严格的

![udp](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/nat/udp-frp.png)

参考上图，有 PC1，Router1，PC2，Router2，Server 五台设备；公网服务器（Server）用于获取客户端实际的出口地址端口，UDP 穿透的流程如下：
1.  PC（`192.168.1.1:7777`） 发送 UDP 请求到 Server（`9.9.9.9:1024`），此时 Server 可以获取到 PC1 的出口地址端口 (也就是 Router1 的出口地址端口) `1.2.3.4:10000`，同时 Router1 添加一条映射 `192.168.1.1:7777` <=> `1.2.3.4:10000` <=> `9.9.9.9:1024`
2.  PC2（`192.168.2.1:8888`） 同样发送 UDP 请求到 Server，Router2 添加一条映射 `192.168.2.1:8888` <=> `5.6.7.8:20000` <=> `9.9.9.9:1024`
3.  Server 将 PC2 的出口地址端口 (`5.6.7.8:20000`) 发送给 PC1
4.  Server 将 PC1 的出口地址端口 (`1.2.3.4:10000`) 发送给 PC2
5.  PC1 使用相同的内网地址端口 (`192.168.1.1:7777`) 发送 UDP 请求到 PC2 的出口地址端口 (Router2 `5.6.7.8:20000`)，此时 Router1 添加一条映射 `192.168.1.1:7777` <=> `1.2.3.4:10000` <=> `5.6.7.8:20000`，与此同时 Router2 没有关于 `1.2.3.4:10000` 的映射，这个请求将被 Router2 丢弃
6.  PC2 使用相同的内网地址端口 (`192.168.2.1:8888`) 发送 UDP 请求到 PC1 的出口地址端口 (Router1 `1.2.3.4:10000`)，此时 Router2 添加一条映射 `192.168.2.1:8888` <=> `5.6.7.8:20000` <=> `1.2.3.4:10000`，与此同时 Router1 有一条关于 `5.6.7.8:20000` 的映射 (上一步中添加的)，Router1 将报文转发给 PC1（`192.168.1.1:7777`）
7. 在 Router1 和 Router2 都有了对方的映射关系，此时 PC1 和 PC2 通过 UDP 穿透建立通信


##  0x05    穿透项目：frp
[frp](https://github.com/fatedier/frp/blob/dev/README_zh.md) 是一个专注于内网穿透的高性能的反向代理应用，支持 TCP、UDP、HTTP、HTTPS 等多种协议，且支持 P2P 通信。可以将内网服务以安全、便捷的方式通过具有公网 IP 节点的中转暴露到公网

####    frp 工作原理
frp 主要由两个组件组成：客户端（frpc）、服务端（frps）；通常情况下，服务端部署在具有公网 IP 地址的机器上，而客户端部署在需要穿透的内网服务所在的机器上；由于内网服务缺乏公网 IP 地址，因此无法直接被非局域网内的用户访问。用户通过访问服务端的 frps，frp 负责根据请求的端口或其他信息将请求路由到相应的内网机器，从而实现穿透通信

![frp-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/frp/frp-flow.png)

1.  首先，frpc 启动之后，连接 frps，并且发送一个 login 请求，之后保持住这个长连接
2.  frps 收到请求之后，会建立一个 listener 监听来自公网的请求
3.  当 frps 接受到请求之后，会在本地看是否有可用的连接（连接池），如果没有，就下发一个 `msg.StartWorkConn` 并且等待来自 frpc 的请求
4.  frpc 收到之后，对 frps 发起请求，请求的最开始会指定这个连接是去向哪个 proxy
5.  frps 收到来自 frpc 的连接之后，就把新建立的连接与来自公网的连接进行流量互转；如果请求断开了，那么就把另一端的请求也断开

## 0x06 NAT 开发相关
对于 NAT 网关服务，实现一个稳定的 NAT 会话表也是核心的问题



##  0x07    总结

最后，回顾下 NAT 的类型及检测方法：

| 类型 | NAT 说明 | 检测方法 |
| :-----:| :----: | :----: |
| Symmetric NAT（对称 NAT） | 在 Symmetric NAT 中，NAT 设备会为每个内部 IP 地址和端口号分配一个唯一的公共 IP 地址和端口号，但是对于不同的目的 IP 地址和端口号，NAT 设备会为其分配不同的公共 IP 地址和端口号。这意味着，对于同一个内部主机，向不同的目的主机发送数据包时，NAT 设备会将其映射到不同的内部 IP 地址和端口号 | 使用 STUN 协议，向外部服务器发送 UDP 数据包，如果返回的数据包中包含了本地 IP 地址和端口号，并且数据包的源 IP 地址和端口号与之前发送的数据包不同，则表示是 Symmetric NAT |
|  |  |  |


##  0x08 参考
-   [NAT 原理以及 UDP 穿透](https://paper.seebug.org/1561/)
-   [进阶必读：代理协议 UDP 全方位透彻解析](https://zhuanlan.zhihu.com/p/518088166)
-   [一口气搞明白有点奇怪的 Socks 5 协议以及 HTTP 代理](https://www.txthinking.com/talks/articles/socks5-and-http-proxy.article)
-   [Socks5 udp 代理](https://www.jianshu.com/p/cf88c619ee5c)
-   [frp 官方文档](https://gofrp.org/zh-cn/docs/)