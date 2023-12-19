---
layout:     post
title:      重拾 Linux 网络（一）：iptables
subtitle:   如何利用 iptables 构建透明代理
date:       2023-08-01
author:     pandaychen
catalog:    true
tags:
    - network
    - iptables
---


##  0x00    前言
最新笔者在研究全流量代理，本文回顾一下 iptables 的原理，先回顾下 netfilter/iptables 的基础概念：

iptables 可以参考下面若干文章：
-  [IPtables](https://www.zsythink.net/archives/category/%e8%bf%90%e7%bb%b4%e7%9b%b8%e5%85%b3/iptables)，文章合集，非常全面了
-  [重温 iptables](https://blog.gmem.cc/iptables)

Netfilter 模块，在网络层的五个位置（也就是 iptables 四表五链中的五链）注册了一些钩子函数，用来截取数据包；把数据包的信息拿出来匹配各个链位置在对应表中的规则，匹配之后再进行相应的处理

![netfilter](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/netfilter-1.png)

iptables 的位置：
![iptables](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/iptables-1.png)

本文部分章节参考了公司同事的文章，没法标注引用地址（表示感谢）

####    本文关注的 iptables 几个重点话题

1.  四表五链及配置原理
2.  NAT
3.  基于 tproxy 的透明代理（本质是 iptables 配置）
4.  iptables 与 tun 虚拟网卡的配合
5.  典型配置

##  0x01    iptables 的四表五链

四表五链，链是很重要的概念，每个链都包含了若干个表的元素（这里的元素代表若干条 iptables 规则），所有同一类的规则构成了 iptables 的表。注意每一个链对应的表都是不完全一样的，表和链之间是多对多的对应关系

![iptables](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/iptables-2.png)

![iptables](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/iptables-3.png)


链代表位置：
-   `PREROUTING`：在对数据包作路由选择之前，应用此链中的规则
-   `INPUT`：当接收到防火墙本机地址的数据包（入站）时，应用此链中的规则
-   `FORWARD`：当接收到需要通过防火墙发送给其他地址的数据包（转发）时，应用此链中的规则；当收到需要通过防火墙中转发给其他地址的数据包时，将应用此链中的规则，注意如果需要实现 `forward` 转发需要开启 Linux 内核中的 `ip_forward` 功能（`sysctl`）
-   `OUTPUT`：当防火墙本机向外发送数据包（出站）时，应用此链中的规则
-   `POSTROUTING`：在对数据包作路由选择之后，应用此链中的规则

注意：`INPUT`、`OUTPUT` 链更多的应用在 ** 主机防火墙 ** 中，即主要针对服务器本机进出数据的安全控制；而 `FORWARD`、`PREROUTING`、`POSTROUTING` 链更多的应用在 ** 网络防火墙 ** 中，特别是防火墙服务器作为网关使用时的情况

表代表了存储的规则（链式）：数据包到了该链处，会去对应表中查询设置的规则，然后决定是否放行、丢弃、转发还是修改等等操作

-   `raw`：主要用来决定是否对数据包进行状态跟踪
-   `mangle`：主要用来修改数据包的服务类型，生存周期，为数据包设置标记，实现流量整形、策略路由等
-   `nat`：网络地址转换，主要用来修改数据包的 IP 地址、端口号信息
-   `filter`：用来对数据包进行过滤，具体的规则要求决定如何处理一个数据包


| 名称 | 功能（位置） | 包含链 | 典型场景 |
|  -------- |-------- |-------- |-------- |
|`filter` 表 | 负责过滤功能，防火墙 | 包含三个规则链：`INPUT`，`FORWARD`，`OUTPUT`||
|`nat`（Network Address Translation）表 | 用于网络地址转换（IP、端口）| 包含三个规则链：`PREROUTING`，`POSTROUTING`，`OUTPUT`||
|`mangle` 表 | 拆解报文，作出修改，封装报文 | 包含五个规则链：`PREROUTING`，`POSTROUTING`，`INPUT`，`OUTPUT`，`FORWARD`||
|`raw` 表 | 关闭 `nat` 表上启用的连接追踪机制，确定是否对该数据包进行状态跟踪 | 包含两条规则链：`OUTPUT`，`PREROUTING`||

注意：`raw` 表只使用在 `PREROUTING` 链和 `OUTPUT` 链上，因为其优先级最高，从而可以对收到的数据包在系统进行 `ip_conntrack`（连接跟踪）前进行处理。一但用户使用了 `raw` 表, 在某个链上，`raw` 表处理完后，将跳过 `NAT` 表和 `ip_conntrack` 处理，即不再做地址转换和数据包的链接跟踪处理了。`RAW` 表可以应用在那些不需要做 `nat` 的情况下，以提高性能


注意每一个链对应的表都是不完全一样的，表和链之间是多对多的对应关系。每个链中的表，按照下面的优先顺序进行查找匹配：

```BASH
raw>mangle>nat>filter
```

封包会依次经过相关的链，在每个链中，会根据表的优先级，依次遍历各个表中的规则。任何表中的规则都有机会拒绝封包

####  表作用

1、`mangle` 表

用于修改封包，某些目标只能用在 `mangle` 表中，包括：
-  `TOS`：此目标设置封包的 Type Of Servier 字段，以便影响封包的处理策略，例如如何进行路由
-  `TTL`：设置封包的 Time To Live 字段
-  `MARK`：给封包添加一个标记值（数字），`iproute2` 能够识别此 `mark` 并且进行特殊的路由处理。此外基于 `MARK` 还可以进行带宽限制、基于 Class 的排队

`MARK` 特性结合策略路由，可以实现很灵活的机制，比如下面说到的透明代理技术

2、`nat` 表

应当仅仅用于网络地址转换。也就是使用以下目标：

-  DNAT：用在你有一个公共地址，别人访问此地址时，你需要将访问重定向到防火墙背后的某个服务时
-  SNAT：允许隐藏在防火墙背后的内网机器访问外部网络
-  MASQUERADE：类似 SNAT，不同之处在于每次都需要动态计算使用什么作为转换后的源地址。用在主机地址不固定的情况下
-  REDIRECT：类似于 DNAT，但是新的目的地址被锁定为接收封包的那个网卡地址，同时端口改为随机的或指定的值

上面这些目标都会改变 IP 封包的首部（源 or 目的）

3、`filter` 表

用于过滤封包，使用 `ACCEPT`、`DROP` 之类的目标

####    链的作用

当数据报文进入链之后，首先匹配第一条规则，如果第一条规则通过则访问，如果不匹配，则接着向下匹配，如果链中的所有规则都不匹配，那么就按照链的默认规则处理数据报文的动作，如下：

![link](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/iptables-link-1.png)

| 名称 | 功能（位置） | 表优先级顺序 | 典型场景（作用） |
|  -------- |-------- |-------- |-------- |
| `PREROUTING` | 数据包进入路由之前 | `raw`、`mangle`、`nat`|  |
| `INPUT` | 数据包进入路由之前 | 	`mangle`、`nat`、`filter` |  |
| `OUTPUT` | 原地址为本机，向外发送 | `raw`、`mangle`、`nat`、`filter` |  |
| `FORWARD` | 实现转发 | `mangle`、`filter`|  |
| `POSTROUTING` | 发送到网卡之前 |`mangle`、`nat` |  |


####    数据流向经过的表

如前图所示，请求报文流入本地要经过的链如下：

-   请求报文要进入本机的某个应用程序，首先会到达 iptables 防火墙的 `PREROUTING` 链，然后又 `PREROUTING` 链转发到 `INPUT` 链，最后转发到所在的应用程序上，即 `PREROUTING—>INPUT—>PROCESS`

-   请求报文从本机流出要经过的链：请求报文读取完应用程序要从本机流出，首先要经过 iptables 的 `OUTPUT` 链，然后转发到 `POSTROUTING` 链，最后从本机成功流出，`PROCESS—>OUTPUT—>POSTROUTING`

-   请求报文经过本机向其他主机转发时要经过的链：请求报文要经过本机向其他的主机进行换发时，首先进入 A 主机的 PREROUTING 链，此时不会被转发到 INPUT 链，因为不是发给本机的请求报文，此时会通过 FORWARD 链进行转发，然后从 A 主机的 POSTROUTING 链流出，最后到达 B 主机的 PREROUTING 链；即 `PREROUTING—>FORWARD—>POSTROUTING`


####   链与表的关系

iptables 的链就是因为在访问该链的时候会按照每个链对应的表依次进行查询匹配执行的操作。如 `PREROUTING` 链对应的表关系是（`raw->mangle->nat`），每个表按照优先级顺序进行链接，此外，每个表中还可能有多个规则，因此最后看起来就像链一样，如下图：

![relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/relation-1.png)


当数据包抵达防火墙时，如上图所示，将依次应用 `raw`、`mangle`、`nat` 和 `filter` 表中对应链内的规则。

再次注意，不是任何链上可以挂任何表：

-  `raw` 可以挂在 `PREROUTING`、`OUTPUT`
-  `mangle` 可以挂在任何链上
-  `nat（SNAT）` 可以挂在 `POSTROUTING`、`INPUT`
-  `nat（DNAT）` 可以挂在 `PREROUTING`、`OUTPUT`
-  `filter` 可以挂在 `FORWARD`、`INPUT`、`OUTPUT`
-  `security` 可以挂在 `FORWARD`、`INPUT`、`OUTPUT`

##  0x02 典型配置

1.  当一个数据包进入网卡时，它首先进入 `PREROUTING` 链，内核根据数据包目的 IP 判断是否需要转送出去
2.  如果数据包就是进入本机的，它就会沿着图向下移动，到达 `INPUT` 链。数据包到了 `INPUT` 链后，任何进程都会收到它。本机上运行的程序可以发送数据包，这些数据包会经过 `OUTPUT` 链，然后到达 `POSTROUTING` 链输出
3.  如果数据包是要转发出去的，且内核允许转发，数据包就会如图所示向右移动，经过 `FORWARD` 链，然后到达 `POSTROUTING` 链输出

那么，规则是如何命中这些报文的呢？


####    usage

![params](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/iptables-params.png)

####    iptables 规则命中

1）规则概念
规则其实就是网络管理员预定义的条件，规则一般的定义为如果数据包头符合这样的条件，就这样处理这个数据包。规则存储在内核空间的信息 包过滤表中，这些规则分别指定了源地址、目的地址、传输协议（如 TCP、UDP、ICMP）和服务类型（如 HTTP、FTP 和 SMTP）等。
当数据包与规则匹配时，iptables 就根据规则所定义的方法来处理这些数据包，如放行 (accept), 拒绝 (reject) 和丢弃 (drop) 等。配置防火墙的主要工作是添加, 修改和删除等规则。
其中：
-  匹配（match）：符合指定的条件，比如指定的 IP 地址和端口
-  丢弃（drop）：当一个包到达时，简单地丢弃，不做其它任何处理
-  接受（accept）：和丢弃相反，接受这个包，让这个包通过
-  拒绝（reject）：和丢弃相似，但它还会向发送这个包的源主机发送错误消息。这个错误消息可以指定，也可以自动产生
-  目标（target）：指定的动作，说明如何处理一个包，比如：丢弃，接受，或拒绝
-  跳转（jump）：和目标类似，不过它指定的不是一个具体的动作，而是另一个链，表示要跳转到那个链上
-  规则（rule）：一个或多个匹配及其对应的目标

重点关注的代理相关的处理动作：

1、ACCEPT<br>
将封包放行，进行完此处理动作后，将不再比对其它规则，直接跳往下一个规则炼（nat:postrouting）。

2、REJECT<br>
拦阻该封包，并传送封包通知对方，可以传送的封包有几个选择：ICMP port-unreachable、ICMP echo-reply 或是 tcp-reset（这个封包会要求对方关闭联机），进后，将不再比对其它规则，直接 中断过滤程序。 如下：

 `iptables -A FORWARD -p TCP --dport 22 -j REJECT --reject-with tcp-reset`

3、DROP<br>
丢弃封包不予处理，进行完此处理动作后，将不再比对其它规则，直接中断过滤程序。

4、REDIRECT<br>
将封包重新导向到另一个端口（PNAT），进行完此处理动作后，将 会继续比对其它规则。 这个功能可以用来实现透明代理

5、MASQUERADE<br>
改写封包来源 IP 为防火墙 NIC IP，可以指定 port 对应的范围，进行完此处理动作后，直接跳往下一个规则炼（mangle:postrouting）。这个功能与 SNAT 略有不同，当进行 IP 伪装时，不需指定要伪装成哪个 IP，IP 会从网卡直接读取，当使用拨接连线时，IP 通常是由 ISP 公司的 DHCP 服务器指派的，这个时候 `MASQUERADE` 特别有用。如下：
```bash
iptables -t nat -A POSTROUTING -p TCP -j MASQUERADE --to-ports 1024-31000
```


6、LOG<br>
将封包相关讯息纪录在 `/var/log` 中，进行完此处理动作后，将会继续比对其它规则。例如：
```bash
iptables -A INPUT -p tcp -j LOG --log-prefix "INPUT packets"
```

7、SNAT<br>
改写封包来源 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将直接跳往下一个规则链（mangle:postrouting）。如下：

```BASH
iptables -t nat -A POSTROUTING -p tcp-o eth0 -j SNAT --to-source 194.236.50.155-194.236.50.160:1024-32000
```

8、DNAT<br>
改写封包目的地 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将会直接跳往下一个规则炼（filter:input 或 filter:forward）。如下：
```BASH
iptables -t nat -A PREROUTING -p tcp -d 15.45.23.67 --dport 80 -j DNAT --to-destination 192.168.1.1-192.168.1.10:80-100
```



9、RETURN<br>
结束在目前规则炼中的过滤程序，返回主规则炼继续过滤，如果把自定义规则炼看成是一个子程序，那么这个动作，就相当于提早结束子程序并返回到主程序中，常见于透明代理中放通本地局域网网段流量

12、MARK<br>
将封包标上某个代号，以便提供作为后续过滤的条件判断依据，进行完此处理动作后，将会继续比对其它规则

####  小结：iptables

![flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/iptables-link-table-flow.png)

小结下，数据包从网卡要通过iptables，数据包流向如下：

1. 数据包到达服务器的网络接口、NIC
2. 进入`raw`表的 `PREROUTING` 链，该链的作用是决定数据包是否被状态跟踪
   -  进入 `mangle` 表的 `PREROUTING` 链，在此可以修改数据包，比如 TOS 等
   -  进入 `nat` 表的 `PREROUTING` 链，可以在此做DNAT
3. 决定路由，查看目标地址是交给本地主机还是转发给其它主机
4. 第一种情况是数据包要转发给其它主机（网关服务器），数据包会依次经过
   -  进入 `mangle` 表的 `FORWARD` 链
   -  进入 `filter` 表的 `FORWARD` 链，在这里可以对所有转发的数据包进行过滤
5. 
   -  进入 `mangle` 表的 `POSTROUTING` 链
   -  进入 `nat` 表的 `POSTROUTING` 链，在这里一般都是用来做 `SNAT` 
6. 数据包流出网络接口，发往网络，本分支流程结束
7. 另外一种情况，数据包的目标地址就是发给本地主机的，它会依次经过：
8. 
   -  进入 `mangle` 表的 `INPUT` 链
   -  进入 `filter` 表的 `INPUT` 链，在这里可以对流入的所有数据包进行过滤
   -  数据包交给本地主机的应用程序进行处理
10. 应用程序处理完毕后发送的数据包进行路由发送决定
11. 
   -  进入 `raw` 表的 `OUTPUT` 链
   -  进入 `mangle` 表的 `OUTPUT` 链
   -  进入 `nat` 表的 `OUTPUT` 链
   -  进入 `filter` 表的 `OUTPUT` 链
12. 
   -  进入 `mangle` 表的 `POSTROUTING` 链
   -  进入 `nat` 表的 `POSTROUTING` 链
13. 进入出去的网络接口，发送往网络，本分支流程结束


##  0x03 iptables 与 tun 网卡
笔者使用TUN网卡构建的代理程序，提供的fake-DNS模式会使用到iptables，通常有下面这一条核心配置：

```BASH
iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53
```

上面这条命令的作用是将所有入方向且目标端口为`53`的UDP（DNS请求）数据包重定向到本地的`53`端口，这样做的目的通常是为了在本地搭建一个DNS服务（DNS透明代理程序）来处理所有的DNS请求，参数解释如下：

```text
-t nat：指定要操作的表为nat表，主要用于网络地址转换
-A PREROUTING：在PREROUTING链中添加一条新的规则，PREROUTING链用于处理进入本地系统之前的数据包
-p udp：指定要匹配的数据包协议为UDP
--dport 53：指定要匹配的数据包的目标端口为53
-j REDIRECT：指定对匹配的数据包执行REDIRECT动作，REDIRECT动作会将数据包重定向到本地的一个端口
--to-ports 53：指定重定向的目标端口为53
```

##  0x04 iptables 应用收发包流程

##  0x05 NAT 工作原理

通常用 iptables 配置 SNAT 和 DNAT，如下图所示，网关进行一次 SNAT 转换，将来源 IP 换成出口公网 IP，这样发给服务器的数据服务器才能知道响应给谁。服务器收到数据后，会返回一个数据包，这里仍然会标记上来源 IP 为服务器的 IP。响应到网关的数据包，网关进行一个 DNAT 转换，将目标地址 IP 换成内网私有 IP，并将数据包发给设备

![NAT](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/nat-1.png)


####    NAT 的典型配置


####  conntrack 和 NAT


####  REDIRECT VS DNAT
`REDIRECT`和`DNAT`都是用来修改数据包的目标地址的，区别如下：

1、`REDIRECT`主要用于在本地机器上重定向数据包，当一个数据包匹配到某个`REDIRECT`规则时，iptables会将数据包的目标地址修改为本地机器的地址，并将目标端口修改为指定的端口；这样，数据包会被发送到本地机器上的另一个服务。`REDIRECT`通常用于透明代理。例如，当想要将所有HTTP流量通过本地的代理服务器进行处理时，可参考下面命令

```BASH
#这条规则会将所有目标端口为80的TCP数据包重定向到本地的3128端口
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3128
```

2、`DNAT`（Destination Network Address Translation） 用于将数据包的目标地址或端口修改为另一个地址或端口。DNAT通常用于端口转发和负载均衡等场景
与REDIRECT不同，DNAT可以将数据包转发到本地机器之外的其他机器

```BASH
#这条规则会将所有目标端口为80的TCP数据包的目标地址和端口修改为192.168.1.2:8080
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.2:8080
```


-  `REDIRECT`用于在本地机器上重定向数据包，常用于透明代理等场景
-  `DNAT`用于修改数据包的目标地址或端口，可以将数据包转发到本地机器之外的其他机器，常用于端口转发和负载均衡等场景

##  0x06  重点：使用 tproxy 实现透明代理

背景：
1.  用户通过路由器访问某些站点（用户所有对外请求的数据都要经过路由器）
2.  用户侧无感知配置（需要在用户的路由器上配置透明代理）
3.  具备自动分流（国内 vs 国外）的能力，分离国内流量和国外流量，同时将国外流量送往代理


架构图如下：
![tproxy](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/transparent-proxy-with-tproxy.png)

注意，路由器、透明代理、国内代理服务（带客户端功能），可能是部署在同台 Linux 主机上，本例采用 tproxy 配合 sockv5 客户端实现透明代理功能

####    tproxy
`Tproxy` 是 Linux 内核自带的一个模块，它可以在 ** 不改变数据包内容的前提下将数据包转发到本地的 socket 上 **。Tproxy 透明代理需要 `iptables`，`ip rule`， `ip route` 相互配合。猜测按照如下的配置来搞定透明代理：

1.  将国内代理服务（也是个 socksv5 客户端）部署在路由器上，在 `lo` 上监听一个本地端口
2.  配置 tproxy ，将需要代理的流量转发到这个端口之后，有国内的代理服务客户端加密之后发送到海外的代理服务，之后的流程就是正常的代理访问

基于 tproxy 的 golang 实现可以参考此项目：[go-tproxy：Linux Transparent Proxy library for Golang](https://github.com/KatelynHaworth/go-tproxy)

####    tproxy 的经典配置
先看下最经典的 tproxy 配置：这个配置会将使得所有去到目的网段 `1.1.1.0/24` 的 tcp 请求送去 tproxy 透明代理，再送去本地（`lo` 环回）的 `1081` 端口。tproxy 源码可以参考 [xt_TPROXY.c](https://github.com/torvalds/linux/blob/master/net/netfilter/xt_TPROXY.c)

```bash
iptables -t mangle -A PREROUTING -d 1.1.1.0/24 -p tcp -m socket -j MARK --set-mark 1
iptables -t mangle -A PREROUTING -d 1.1.1.0/24 -p tcp -j TPROXY --on-port 1081 --on-ip 127.0.0.1 --tproxy-mark 0x1/0x1
ip rule add fwmark 1 lookup 100     #数据包有标记 1，进入 100 号路由表
ip route add local 0.0.0.0/0 dev lo table 100
```

上面配置的主要含义是：

1.  在 `mangle` 表的 `PREROUTING` 链中，匹配目的 IP 地址为 `1.1.1.0/24`，协议为 TCP，并使用 `socket` 匹配器，将数据报文的标记设置为 `1`
2.  在 `mangle` 表的 `PREROUTING` 链中，匹配目的 IP 地址为 `1.1.1.0/24`，协议为 TCP，使用 TPROXY 扩展，将数据报文重定向到本地 IP 地址 `127.0.0.1` 的 `1081` 端口，并设置 TProxy 标记为 `0x1/0x1`
3.  添加一个规则，当数据报文的标记为 `1` 时，使用 `100` 号路由表进行路由
4.  在 `100` 号路由表中添加一条本地路由，将所有数据报文都路由到 `lo` 接口上

所以，上述配置的作用是实现数据报文的透明代理。当数据报文的目的 IP 地址为 `1.1.1.0/24` 时，内核会根据第一条 iptables 规则将数据报文的标记设置为 `1`，并根据第二条 iptables 规则将数据报文重定向到本地的 `1081` 端口上。然后，根据第三条规则，内核将数据报文路由到 `100` 号路由表中，并根据第四条规则将数据报文路由到 `lo` 接口上，从而实现数据报文的透明代理
现网配置如下，`mangle`链：

![mangle](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/transparentproxy/tproxy/tproxy-mangle-1.png)

`100`号路由表配置：

![router](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/transparentproxy/tproxy/tproxy-router-100.png)

ip rule规则配置：

![rule](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/transparentproxy/tproxy/tproxy-ip-rule-2.png)


####    配置原理：路由辅助
既然 Tproxy 已经搞定一切，那上面 经典配置 中 `1`，`3`，`4` 行命令的作用是？需要结合 iptables 的转发流来看：

![iptables-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/iptables-flow.jpg)

由于 tproxy 工作在 `PREROUTING` 链的 `mangle` 表，上面也说了 tproxy 只是替换了 socket 没有做其他操作，下一步经过 `nat` 表后进入路由判断。由于没有修改数据包的目标 ip，正常路由匹配后发现目标 ip 不是本机 ip 会走图中的数据转发流向，进入 `FORWARD` 链，然后进入 `POSTROUTING` 链离开本机

上面是一般报文的流程，但是需要数据进入本机才能被本机的 socket 捕获进入下一步 sockv5 客户端加密转发逻辑，这里上面的其他命令就起作用了：

-  操作 `1`：给所有要 tproxy 处理的数据打上 `mark 1`
-  操作 `3`：添加路由规则，发现有 `mark 1` 的数据包走 id 为 `100` 的路由表
-  操作 `4`：给 id 为 `100` 的路由表添加一条路由，去往 `0.0.0.0/0` （任何数据）的数据送往 本地 `lo` 接口

注意一个问题：上图也看到了 tproxy 工作在 `PREROUTING` 链的 `mangle` 表，而路由器本身发起的请求会走图中数据发出流向的方向，先走 `OUTPUT` 链再走 `POSTROUTING` 链直接离开本机。因此无法对路由本身发起的请求做透明代理（无法在路由器上直接 `CURL` 测试）

经过上述操作，数据包就会进入 `INPUT` 链，进而被之前替换好的 socket （socksv5 客户端）读取；最终回到 `INPUT` 链的原因是因为在最后修改之后，去到了 `lo` 接口被接收，需要监听在 `lo` 上的进程还需要额外的 socket 配置，否则包会被 tproxy 丢弃掉；但是这里有个疑问是，为啥这里 `lo` 上不受 iptables 影响了？

最后强调一下，**Tproxy 确实没有改变数据包内容，而是在本地把对应的 socket 换了**

####    代理程序 1：国内服务（socksv5 可客户端）
在本地监听 `1081` 端口就可以获取到从 tproxy 过来的数据了，参考项目 [go-tproxy](https://github.com/KatelynHaworth/go-tproxy)，记得一定要设置 socket 的选项。支持接收非本地端口的请求，需要在 socket 上设置 `SOL_IP`、`IP_TRANSPARENT` 选项，否则的话会被 tproxy 丢弃

```golang
func main(){
    // ...
    // 监听本地 1081 端口
    addr, _ := net.ResolveTCPAddr("tcp", "0.0.0.0:1081")
    server, err := net.ListenTCP("tcp", addr)
    if err != nil {
       //myPrintf(1, "create tcp get error %s \n", err.Error())
       return
    }

    f, err := server.File()
    if err != nil {
        //myPrintf(1, "get server file error %s \n", err.Error())
        return
    }

    if err := syscall.SetsockoptInt(int(f.Fd()), syscall.SOL_IP, syscall.IP_TRANSPARENT, 1); err != nil {
        //fmt.Printf(1, "set sock opt int error %s \n", err.Error())
        return
    }

    //...
}
```

至此，通过 tproxy 机制再加上本地的监听 socket，已经能从透明代理中拿到客户端送来的数据了，下面就是构建加密通讯的头包把目标 ip port 相关信息送到海外代理服务。可通过 `c.LocalAddr().String()` 获取目的地地址和端口，之后按照 `socks5` 协议中 代理请求协议格式拼接包头，即可

```golang
func (tproxy *Tproxy) onReceive(fd int32, c net.Conn, mainConn net.Conn) {

   h, p, err := net.SplitHostPort(c.LocalAddr().String())
   if err != nil {
      myPrintf(1, "SplitHostPort error %s \n", err)
      return
   }

   var data []byte
   ip := net.ParseIP(h)
   if ip4 := ip.To4(); ip4 != nil {
      data = append([]byte("\x05\x01\x00\x01"), []byte(ip4)...)
   } else if ip6 := ip.To16(); ip6 != nil {
      data = append([]byte("\x05\x01\x00\x04"), []byte(ip6)...)
   }

   i, _ := strconv.Atoi(p)
   port := make([]byte, 2)
   binary.BigEndian.PutUint16(port, uint16(i))
   data = append(data, port...)

   // 后续加密送往海外代理服务的逻辑
}
```

小结下，到这里已经实现了对所有客户端去往 `1.1.1.0/24` 地址的请求透明代理并发送到海外，不过还存在下面的问题：

-  一个网段需要走特殊逻辑，多个网段如何配置？
-  国内外流量分离
-  自动化的路由发现配置

####    代理程序 2：服务端

服务端这里直接获取到真实的请求，发起真实访问即可


####    高级话题：如何分离流量

原文提供了两种方案：

-  第一种方案是将所有国内 ip 段写到 iptables 中，如果目标地址是国内 ip 段就直接路由不走代理
-  第二种方案是准备一个域名名单，名单中的域名解析 `dns` 后自动加入一个 `ipset` 后续用这个 `ipset` 判断

####  方案一：基于 iptables

下载到所有国内 ip 网段，选 `CIDR` 格式，然后根据导出的网段生成 iptables 命令。文件格式如下：

```
43.224.68.0/22
```


这里特别需要注意的是，一定要把本机 ip，内网地址加到免代理列表中，否则就连不上你的路由器了，最终的 iptables 策略大概如下：

```BASH
# 单独创建一个 proxy 链用于管理代理地址 并在 PREROUTING 链中引入 proxy
iptables -t mangle -N proxy
iptables -t mangle -A PREROUTING -j proxy

# 对于去往本地地址，内网地址的 直接 return 离开 proxy 链会继续 PREROUTING 链其他处理
iptables -t mangle -A proxy -d 127.0.0.1/32 -j RETURN
iptables -t mangle -A proxy -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A proxy -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A proxy -d 10.0.0.0/8 -j RETURN

# 根据之前下载的国内 ip 网段生成的 iptables 命令，如果命中国内网段也 return 离开 proxy 链
iptables -t mangle -A proxy -d 43.224.68.0/22 -j RETURN
#..... 其余的网段
#......

# 最后余下的全部走代理，udp 也配置起来
iptables -t mangle -A proxy -p tcp -m socket -j MARK --set-mark 1
iptables -t mangle -A proxy -p tcp -j TPROXY --on-port 1081 --on-ip 127.0.0.1 --tproxy-mark 0x1/0x1
iptables -t mangle -A proxy -p udp -m socket -j MARK --set-mark 1
iptables -t mangle -A proxy -p udp -j TPROXY --on-port 1081 --on-ip 127.0.0.1 --tproxy-mark 0x1/0x1
```

这个方案最大的问题是 iptables 中匹配的条数太多，每个流量都要经过 6000 多行规则匹配效率不高

####  方案二：dns
需要用到 `dnsmasq`，需要添加 [代理配置名单](https://raw.githubusercontent.com/phusbot/dnsmasq_gfwlist_ipset/master/dnsmasq_gfwlist_ipset.conf) 到 `dnsmasq` 配置中，只要 `dns` 解析了名单中的域名，在返回域名对应 ip 同时会把加入到叫 `xxxlist` 的 `ipset` 中。通过下面命令查看列表中的 ip：

```BASH
root@X-WRT:~# ipset list xxxlist
Name: xxxlist
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 23976
References: 1
Number of entries: 867
Members:
104.244.42.194
208.115.237.76
54.88.42.242
172.217.160.97
142.250.66.46
......
```

之后的 iptables 只需判断目的 ip 是否在 `ipset` 中，`ipset` 相对规模会小一些。

```BASH
# 创建一个 ipset
ipset create xxxlist hash:ip

# 单独创建一个 proxy 链用于管理代理地址 并在 PREROUTING 链中引入 proxy
iptables -t mangle -N proxy
iptables -t mangle -A PREROUTING -j proxy

# 对于去往本地地址，内网地址的 直接 return 离开 proxy 链会继续 PREROUTING 链其他处理
iptables -t mangle -A proxy -d 127.0.0.1/32 -j RETURN
iptables -t mangle -A proxy -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A proxy -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A proxy -d 10.0.0.0/8 -j RETURN

# 没有在 xxxlist ipset 中的目的地直接 return 离开 proxy 链
iptables -t mangle -A proxy -m set ! --match-set xxxlist dst -j RETURN

# 最后余下的全部走代理，udp 也配置起来
iptables -t mangle -A proxy -p tcp -m socket -j MARK --set-mark 1
iptables -t mangle -A proxy -p tcp -j TPROXY --on-port 1081 --on-ip 127.0.0.1 --tproxy-mark 0x1/0x1
iptables -t mangle -A proxy -p udp -m socket -j MARK --set-mark 1
iptables -t mangle -A proxy -p udp -j TPROXY --on-port 1081 --on-ip 127.0.0.1 --tproxy-mark 0x1/0x1
```

小结：DNS 的方案，如果你访问的外部域名特别多导致 ipset 非常大可以考虑用第一种方案


####    小结

再次回顾下一个外部请求是如何经过 `tproxy` 透明代理转发到最终服务器的：

1.  用户请求 `curl https://www.google.com`
2.  `dnsmasq` 返回了 google 对应的 ip 并把 ip 加入 xxxlist 的 ipset
3.  `curl` 根据返回的 ip 组装了 ip 包发给了路由器，注意，用户发出的包是正常的数据包，srcip 为本机 ip，dstip 为 google 的 ip
4.  数据包进入了路由器 `iptables` 的 `PREROUTING` 链的 `mangle` 表
5.  `PREROUTING` 链把数据给了 `proxy` 链
6.  经过一番匹配，`proxy` 中的 `set` 模块在 `xxxlist` ipset 中找到了 google ip，因此这个数据包没有离开 `proxy` 链
7.  socket 模块给这个数据包打上了 `mark 1` 标签
8.  数据包进入 `tproxy`，`tproxy` 的作用是把对应的 socket 替换为绑定在 `127.0.0.1:1801` 端口上的科学上网国内客户端
9.  `tproxy` 工作完成，由于数据包有 `mark 1` 标签路由时使用了 id 为 `100` 的路由表
10. 路由表中只有一条路由（所有数据都去 `lo`），因此数据包送入 `INPUT` 链，最终被 `1801` 端口上的程序接收
11. 代理程序读取到数据包的目的地址，构建了一个 `socks5` 代理请求协议头
12. 协议头和后续数据走加密通道传输到海外代理服务端
13. 海外代理服务端请求 google 后，将返回数据加密送回
14. 国内代理服务收到响应数据后解密，发给你的 ip。回包走 `OUTPUT` 链走 `POSTROUTING` 离开路由
15. 用户收到回包，由于请求包的数据头没被修改，响应包的源地址是请求包的目标地址，`curl` 收到了包解析展现结果在屏幕上

整个透明代理过程使用了 ipset、iptables、tproxy、ip rule、ip route 以及各种加密通讯技术。

##  0x07 数据包的旅程

下面这张图描述了一个 `L3`IP 封包如何通过 iptables，有几个特别重要的点：

-  iptables 是用户空间命令，通过 netlink 和内核的 netfilter 模块进行交互，在 L3/L4 操控封包
-  iptables 和内核路由的关系：执行完 `PREROUTING` 链之后，会进行路由表的查询
-  通过 `lo` 接口的封包，不走 `PREROUTING` 的 DNAT 表
-  出站封包在 `OUTPUT` 链之前就进行了路由处理。但是如果 `OUTPUT` 进行了 DNAT，则会进行重新选路
-  入站封包，如果使用了隧道，则会经由 `PREROUTING` - `INPUT` 链逐层的解除隧道
-  出站封包，如果使用了隧道，则会经由 `OUTPUT` -`POSTROUTING` 链逐层的进行隧道封装

![flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/packet_flow10.png)

下面这张表明确了 iptables 进行路由判断的时机：

![route](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/packet_flow11.jpg)

##  0x08  一些疑问

####  透明代理：IP_TRANSPARENT选项
`IP_TRANSPARENT` 选项允许 socket 将任意非本机地址视为本机地址，进而可以绑定在非本机地址，伪装为非本机地址发送、接收数据；例如，网关（`192.168.0.1` / `1.2.3.4`）作为透明代理，劫持了客户端（`192.168.0.200`）与远端（`2.2.2.2`）的连接。代替客户端与远端连接，又伪装成远端与客户端连接：

```BASH
$ netstat -atunp
Proto Recv-Q Send-Q Local Address           Foreign Address            State       PID/Program name
tcp        0      0 1.2.3.4:37338        2.2.2.2:443            ESTABLISHED    2904/proxy
tcp        0      0 ::ffff:2.2.2.2:443  ::ffff:192.168.0.200:56418 ESTABLISHED 2904/proxy
```

####  透明代理：socket替换的原理是什么？
可以参考此文[一文吃透 Linux TProxy 透明代理](https://asphaltt.github.io/post/linux-how-tproxy-works/)\

####  `-m socket`的作用
回看tproxy透明代理第一步操作：
```bash
iptables -t mangle -A proxy -p tcp -m socket -j MARK --set-mark 1
```

这里使用到`-m socket` 来优化性能，`nf_tproxy_get_sock_v4()` 的注释中提到了这一点：

```text
/*
 * Please note that there's an overlap between what a TPROXY target
 * and a socket match will match. Normally if you have both rules the
 * "socket" match will be the first one, effectively all packets
 * belonging to established connections going through that one.
*/
```

被 TProxy 重定向过的数据包建立连接后，网络栈中有了数据包原始五元组与 socket 的映射关系。之后相同五元组的数据包在网络栈的常规处理中匹配到的 socket，也即 TPROXY 中第一次用数据包五元组匹配的 `sk = nf_tproxy_get_sock_v4(...., NF_TPROXY_LOOKUP_ESTABLISHED)` ，就是正确的（或者说已重定向过的），没必要进行后续的 socket 替换。所以用 `iptables socket` 规则分流出这一部分，提升性能

以 TCP 为例：

```BASH
iptables -t mangle -N tproxy_divert
iptables -t mangle -A tproxy_divert -j MARK --set-mark 0x233
iptables -t mangle -A tproxy_divert -j ACCEPT

iptables -t mangle -A PREROUTING -p tcp -m socket -j tproxy_divert
iptables -t mangle -A PREROUTING -p tcp -j TPROXY --on-port 10000 --on-ip 127.0.0.1 --tproxy-mark 0x233
```

####    如何理解 iptables 的转发条件：本机

iptables 有五个链：`INPUT`、`OUTPUT`、`FORWARD`、`PREROUTING`、`POSTROUTING`

其中，`PREROUTING` 和 `POSTROUTING` 链会查询路由表：
-  `PREROUTING` 链在数据包进入网络接口后，在路由选择前被处理，因此需要查询路由表以确定数据包的下一跳
-  `POSTROUTING` 链在数据包离开网络接口前，对数据包进行处理，也需要查询路由表以确定数据包的下一跳

iptables 的 `FORWARD` 链是用于处理转发（forward）数据包的链，即当一台 Linux 路由器上收到一个数据包，需要将该数据包转发到另一台主机时，该数据包就会进入 `FORWARD` 链；在 Linux 系统中，当一个数据包到达时，会根据其目的 IP 地址进行路由选择，如果目的 IP 地址不是本机的 IP 地址，则该数据包会被认为是转发数据包，进入 `FORWARD` 链。因此，可以理解为，当数据包的目的 IP 地址不是本机的 IP 地址时，该数据包就会进入 `FORWARD` 链。需要注意的是，iptables 的 `FORWARD` 链只有在 Linux 系统作为路由器时才会被触发，如果 Linux 系统只是作为普通的主机使用，那么 `FORWARD` 链将不会被触发


####  iptables与路由表
看了tproxy的原理，让我产生了一个疑问，iptables的整个流程中，哪些分支会经过路由表查询？

-  数据包到达本地网卡后，内核会进行路由表查询，判断数据包的目标IP地址是否为本地IP地址
-  如果数据包的目标IP地址是本地IP地址，内核会将数据包传递给`lo`接口
-  `lo`接口会将数据包发送到内核中进行处理。在处理过程中，内核会根据iptables规则进行匹配，找到与数据包匹配的规则
-  如果iptables规则中设置了`REDIRECT`或`DNAT`目标，内核会将数据包重定向到指定的本地端口或者目标IP地址和端口号。重定向后，数据包会经过本地网卡和路由表，然后到达目标服务器
-  如果iptables规则中设置了`MARK`目标，内核会将数据包打上标记，然后将数据包传递给下一个处理步骤
-  如果iptables规则中设置了`ACCEPT`或`DROP`目标，内核会根据目标进行相应的处理，例如接受或者丢弃数据包

这里有一个细节，当数据包的目标IP是本地IP地址时，数据包不需要通过网卡发送到物理网络上，而是直接传递给`lo`接口进行处理；当内核将数据包传递给`lo`接口时，数据包不会经过物理网络，而是直接进入内核进行处理


## 0x09  总结

再回顾下tproxy用于将数据包重定向到本地代理服务器进行处理的核心命令：
```bash
iptables -t mangle -A PREROUTING -d 1.1.1.0/24 -p tcp -j TPROXY --on-port 1081 --on-ip 127.0.0.1 --tproxy-mark 0x1/0x1
```

```text
-t mangle：指定iptables表为mangle表，mangle表可以修改数据包的头部信息
-A PREROUTING：将规则添加到PREROUTING链中，表示数据包在路由之前被处理
-d 1.1.1.0/24：指定目标IP地址为1.1.1.0/24，表示匹配目标IP地址为1.1.1.0/24的数据包
-p tcp：指定协议为TCP，表示匹配TCP协议的数据包
-j TPROXY：指定目标为TPROXY，表示将匹配的数据包重定向到TPROXY处理
--on-port 1081：指定重定向的目标端口为1081，表示将匹配的数据包重定向到本地端口1081
--on-ip 127.0.0.1：指定重定向的目标IP地址为127.0.0.1，表示将匹配的数据包重定向到本地IP地址127.0.0.1
--tproxy-mark 0x1/0x1：指定tproxy标记为0x1，表示标记需要经过tproxy处理的数据包
```

```BASH
ip rule add fwmark 1 lookup 100     #数据包有标记 1，进入 100 号路由表
ip route add local 0.0.0.0/0 dev lo table 100
```

另外，使用策略路由将`fwmark`路由到`lo`上，这样即使数据包的目的IP不是本机，内核不会把这个包送到`FORWARD`链上去（默认会被送到`FORWARD`链），这样数据包就能送到代理客户端进行后续处理了（策略路由将所有数据包的下一跳都指向 `lo`，默认`lo`接口的数据包就会直接送去本机进程）


tproxy的数据流向如下图（蓝色->绿色->红色）：
![tproxy-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/tproxy-flow.png)

##  0x0A 参考
-   [深入浅出带你理解 iptables 原理](https://zhuanlan.zhihu.com/p/547257686)
-   [iptables 的四表五链与 NAT 工作原理 _](https://tinychen.com/20200414-iptables-principle-introduction/)
-   [Transparent proxy support](https://www.kernel.org/doc/html/latest/networking/tproxy.html#making-non-local-sockets-work)
-   [重温 iptables](https://blog.gmem.cc/iptables)
-   [iptables 详解（4）：iptables 匹配条件总结之一](https://www.zsythink.net/archives/1544)
-   [tproxy（透明代理）](https://www.zhaohuabing.com/learning-linux/docs/tproxy/)
-   [go-tproxy 项目](https://github.com/KatelynHaworth/go-tproxy/blob/master/tproxy_tcp.go)
-   [Linux-iptables 防火墙、四表五链及 SNAT 与 DNAT 的原理与应用
目录](https://dianqk.blog/2021/11/20/iptables-tproxy-and-home/)
-   [iptables 网络防火墙、SNAT、DNAT 原理及端口重定向实战](https://www.cnblogs.com/struggle-1216/p/11994523.html)
-   [深入理解 Linux TProxy](https://rook1e.com/p/linux-tproxy/)
-   [iptables-extensions](https://ipset.netfilter.org/iptables-extensions.man.html#lbDW)