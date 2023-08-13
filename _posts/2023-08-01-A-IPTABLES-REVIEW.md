---
layout:     post
title:      重拾 Linux 网络（一）：iptables
subtitle:
date:       2023-08-01
author:     pandaychen
catalog:    true
tags:
    - network
    - iptables
---


##  0x00    前言
最新笔者在研究全流量代理，本文回顾一下 iptables 的原理，先回顾下 netfilter/iptables 的基础概念：

Netfilter 模块，在网络层的五个位置（也就是 iptables 四表五链中的五链）注册了一些钩子函数，用来截取数据包；把数据包的信息拿出来匹配各个链位置在对应表中的规则，匹配之后再进行相应的处理

![netfilter](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/netfilter-1.png)

![netfilter](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/iptables-2.png)

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


| 名称 | 功能（位置） | 包含链 | 典型场景（作用） |
|  -------- |-------- |-------- |-------- |
|filter 表 | 负责过滤功能，防火墙 | 包含三个规则链：`INPUT`，`FORWARD`，`OUTPUT`||
|nat（Network Address Translation）表 | 用于网络地址转换（IP、端口）| 包含三个规则链：PREROUTING，POSTROUTING，OUTPUT||
|mangle 表 | 拆解报文，作出修改，封装报文 | 包含五个规则链：PREROUTING，POSTROUTING，INPUT，OUTPUT，FORWARD||
|raw 表 | 关闭 nat 表上启用的连接追踪机制，确定是否对该数据包进行状态跟踪 | 包含两条规则链：OUTPUT，PREROUTING||

注意：`raw` 表只使用在 `PREROUTING` 链和 `OUTPUT` 链上，因为其优先级最高，从而可以对收到的数据包在系统进行 `ip_conntrack`（连接跟踪）前进行处理。一但用户使用了 `raw` 表, 在某个链上，`raw` 表处理完后，将跳过 `NAT` 表和 `ip_conntrack` 处理，即不再做地址转换和数据包的链接跟踪处理了。`RAW` 表可以应用在那些不需要做 `nat` 的情况下，以提高性能


注意每一个链对应的表都是不完全一样的，表和链之间是多对多的对应关系。每个链中的表，按照下面的优先顺序进行查找匹配：

```BASH
raw>mangle>nat>filter
```

####    链的作用

当数据报文进入链之后，首先匹配第一条规则，如果第一条规则通过则访问，如果不匹配，则接着向下匹配，如果链中的所有规则都不匹配，那么就按照链的默认规则处理数据报文的动作，如下：

![link]()

| 名称 | 功能（位置） | 表优先级顺序 | 典型场景（作用） |
|  -------- |-------- |-------- |-------- |
| PREROUTING | 数据包进入路由之前 | raw、mangle、nat|  |
| INPUT | 数据包进入路由之前 | 	mangle、nat、filter |  |
| OUTPUT | 原地址为本机，向外发送 | raw、mangle、Nat、filter |  |
| FORWARD | 实现转发 | mangle、filter|  |
| POSTROUTING | 发送到网卡之前 |mangle、nat |  |


####    数据流向经过的表

如前图所示，请求报文流入本地要经过的链如下：

-   请求报文要进入本机的某个应用程序，首先会到达 iptables 防火墙的 `PREROUTING` 链，然后又 `PREROUTING` 链转发到 `INPUT` 链，最后转发到所在的应用程序上，即`PREROUTING—>INPUT—>PROCESS`

-   请求报文从本机流出要经过的链：请求报文读取完应用程序要从本机流出，首先要经过 iptables 的 `OUTPUT` 链，然后转发到 `POSTROUTING` 链，最后从本机成功流出，`PROCESS—>OUTPUT—>POSTROUTING`

-   请求报文经过本机向其他主机转发时要经过的链：请求报文要经过本机向其他的主机进行换发时，首先进入 A 主机的 PREROUTING 链，此时不会被转发到 INPUT 链，因为不是发给本机的请求报文，此时会通过 FORWARD 链进行转发，然后从 A 主机的 POSTROUTING 链流出，最后到达 B 主机的 PREROUTING 链；即`PREROUTING—>FORWARD—>POSTROUTING`


####   链与表的关系

iptables 的链就是因为在访问该链的时候会按照每个链对应的表依次进行查询匹配执行的操作。如 `PREROUTING` 链对应的表关系是（`raw->mangle->nat`），每个表按照优先级顺序进行链接，此外，每个表中还可能有多个规则，因此最后看起来就像链一样，如下图：

![relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/iptables/relation-1.png)


当数据包抵达防火墙时，如上图所示，将依次应用 `raw`、`mangle`、`nat` 和 `filter` 表中对应链内的规则。

##  0x0 典型配置

1.  当一个数据包进入网卡时，它首先进入 PREROUTING 链，内核根据数据包目的 IP 判断是否需要转送出去
2.  如果数据包就是进入本机的，它就会沿着图向下移动，到达 INPUT 链。数据包到了 INPUT 链后，任何进程都会收到它。本机上运行的程序可以发送数据包，这些数据包会经过 OUTPUT 链，然后到达 POSTROUTING 链输出
3.  如果数据包是要转发出去的，且内核允许转发，数据包就会如图所示向右移动，经过 FORWARD 链，然后到达 POSTROUTING 链输出

那么，规则是如何命中这些报文的呢？

####    iptables 规则命中

1）规则概念
规则（rules）其实就是网络管理员预定义的条件，规则一般的定义为 “如果数据包头符合这样的条件，就这样处理这个数据包”。规则存储在内核空间的信息 包过滤表中，这些规则分别指定了源地址、目的地址、传输协议（如 TCP、UDP、ICMP）和服务类型（如 HTTP、FTP 和 SMTP）等。
当数据包与规则匹配时，iptables 就根据规则所定义的方法来处理这些数据包，如放行 (accept), 拒绝(reject) 和丢弃 (drop) 等。配置防火墙的主要工作是添加, 修改和删除等规则。
其中：
匹配（match）：符合指定的条件，比如指定的 IP 地址和端口。
丢弃（drop）：当一个包到达时，简单地丢弃，不做其它任何处理。
接受（accept）：和丢弃相反，接受这个包，让这个包通过。
拒绝（reject）：和丢弃相似，但它还会向发送这个包的源主机发送错误消息。这个错误消息可以指定，也可以自动产生。
目标（target）：指定的动作，说明如何处理一个包，比如：丢弃，接受，或拒绝。
跳转（jump）：和目标类似，不过它指定的不是一个具体的动作，而是另一个链，表示要跳转到那个链上。
规则（rule）：一个或多个匹配及其对应的目标。

重点关注的代理相关的处理动作：

1、ACCEPT<br>
将封包放行，进行完此处理动作后，将不再比对其它规则，直接跳往下一个规则炼（nat:postrouting）。

2、REJECT<br>
拦阻该封包，并传送封包通知对方，可以传送的封包有几个选择：ICMP port-unreachable、ICMP echo-reply 或是 tcp-reset（这个封包会要求对方关闭联机），进后，将不再比对其它规则，直接 中断过滤程序。 范例如下：

 `iptables -A FORWARD -p TCP --dport 22 -j REJECT --reject-with tcp-reset`

3、DROP<br>
丢弃封包不予处理，进行完此处理动作后，将不再比对其它规则，直接中断过滤程序。

4、REDIRECT<br>
将封包重新导向到另一个端口（PNAT），进行完此处理动作后，将 会继续比对其它规则。 这个功能可以用来实作通透式 porxy 或用来保护
web 服务器。例如：

iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080

5、MASQUERADE<br>
改写封包来源 IP 为防火墙 NIC IP，可以指定 port 对应的范围，进行完此处理动作后，直接跳往下一个规则炼（mangle:postrouting）。这个功能与 SNAT 略有不同，当进行 IP 伪装时，不需指定要伪装成哪个 IP，IP 会从网卡直接读取，当使用拨接连线时，IP 通常是由 ISP 公司的 DHCP 服务器指派的，这个时候 MASQUERADE 特别有用。范例如下：

iptables -t nat -A POSTROUTING -p TCP -j MASQUERADE --to-ports 1024-31000

6、LOG<br>
将封包相关讯息纪录在 /var/log 中，详细位置请查阅 /etc/syslog.conf 组态档，进行完此处理动作后，将会继续比对其它规则。例如：

iptables -A INPUT -p tcp -j LOG --log-prefix "INPUT packets"

7、SNAT<br>
改写封包来源 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将直接跳往下一个规则炼（mangle:postrouting）。范例如下：

iptables -t nat -A POSTROUTING -p tcp-o eth0 -j SNAT --to-source 194.236.50.155-194.236.50.160:1024-32000

8、DNAT<br>
改写封包目的地 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将会直接跳往下一个规则炼（filter:input 或 filter:forward）。范例如下：

iptables -t nat -A PREROUTING -p tcp -d 15.45.23.67 --dport 80 -j DNAT --to-destination 192.168.1.1-192.168.1.10:80-100

9、MIRROR<br>
镜像封包，也就是将来源 IP 与目的地 IP 对调后，将封包送回，进行完此处理动作后，将会中断过滤程序。

10、QUEUE<br>
中断过滤程序，将封包放入队列，交给其它程序处理。透过自行开发的处理程序，可以进行其它应用，例如：计算联机费用....... 等。

11、RETURN<br>
结束在目前规则炼中的过滤程序，返回主规则炼继续过滤，如果把自订规则炼看成是一个子程序，那么这个动作，就相当于提早结束子程序并返回到主程序中。

12、MARK<br>
将封包标上某个代号，以便提供作为后续过滤的条件判断依据，进行完此处理动作后，将会继续比对其它规则，举例如下：





##  0x0 iptables 与 tun 网卡


##  0x0 iptables 应用收发包流程

##  0x0 NAT 工作原理


##  0x  透明代理配置


##  0x01 数据包的旅程


##  参考
-   [深入浅出带你理解 iptables 原理](https://zhuanlan.zhihu.com/p/547257686)
-   [iptables 的四表五链与 NAT 工作原理 _](https://tinychen.com/20200414-iptables-principle-introduction/)