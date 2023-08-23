---
layout:     post
title:      重拾 Linux 网络（三）：透明代理中的路由策略
subtitle:
date:       2023-08-02
author:     pandaychen
catalog:    true
tags:
    - network
    - 路由
---


##  Ox00    前言
不要把路由表和 `iptables` 混淆，路由表决定如何传输数据包，而 `iptables` 决定是否传输数据包，他俩的职责不一样；最近印象较深的一句话

##  0x01    iptables/router 区别

-   `ip route`（即`route -n`），`iproute2`提供的命令

```bash
[root@VM-16-10-tencentos ~]# ip rule
0:      from all lookup local
9500:   not from all dport 53 lookup main suppress_prefixlength 0
9510:   not from all iif lo lookup 1970566510
9520:   from 0.0.0.0 iif lo uidrange 0-4294967294 lookup 1970566510
9530:   from 198.18.0.1 iif lo uidrange 0-4294967294 lookup 1970566510
32766:  from all lookup main
32767:  from all lookup default
```

```bash
[root@VM-16-10-tencentos ~]# ip route
default via 172.16.16.1 dev eth0 
172.16.16.0/20 dev eth0 proto kernel scope link src 172.16.16.10 metric 100 
198.18.0.0/16 dev utun proto kernel scope link src 198.18.0.1 
```

本小节梳理 `ip rule`，`ip route`，`iptables`` 三者之间的关系：

####    路由策略 （使用 ip rule 命令操作路由策略数据库）
1、`ip rule` 命令

```BASH
ip rule [list | add | del] SELECTOR ACTION （add 添加；del 删除； llist 列表）
SELECTOR := [from PREFIX 数据包源地址] [ to PREFIX 数据包目的地址] [ tos TOS 服务类型][ dev STRING 物理接口] [ pref NUMBER ] [fwmark MARK iptables 标签]
ACTION := [table TABLE_ID 指定所使用的路由表] [ nat ADDRESS 网络地址转换][ prohibit 丢弃该表 | reject 拒绝该包 | unreachable 丢弃该包]
[flowid CLASSID]
TABLE_ID := [local | main | default | new | NUMBER]
```

2、作用

基于策略的路由比传统路由在功能上更强大，使用更灵活，它使网络管理员不仅能够根据目的地址而且能够根据报文大小、应用或 IP 源地址等属性来选择转发路径。

linux 高级路由即基于策略的路由，比传统路由在功能上更强大，使用也更灵活，它不仅能够像传统路由一样，根据目的地址来转发数据，而且也能够根据报文大小、应用，协议或 ip 源地址来选择路由转发路径。

3、规则

在 Linux 系统启动时，内核会为路由策略数据库配置三条缺省的规则：

0 匹配任何条件 查询路由表 local(ID 255) 路由表 local 是一个特殊的路由表，包含对于本地和广播地址的高优先级控制路由。rule 0 非常特殊，不能被删除或者覆盖。
32766 匹配任何条件 查询路由表 main(ID 254) 路由表 main(ID 254) 是一个通常的表，包含所有的无策略路由。系统管理员可以删除或者使用另外的规则覆盖这条规则。
32767 匹配任何条件 查询路由表 default(ID 253) 路由表 default(ID 253) 是一个空表，它是为一些后续处理保留的。对于前面的缺省策略没有匹配到的数据包，系统使用这个策略进行处理。这个规则也可以删除。
PS：路由表和策略区别：

规则指向路由表，多个规则可以引用一个路由表，而且某些路由表可以没有策略指向它。如果系统管理员删除了指向某个路由表的所有规则，这个表就没有用了，但是仍然存在，直到里面的所有路由都被删除，它才会消失。

####    路由表 （使用 ip route 命令操作静态路由表）
1、概念

所谓路由表，指的是路由器或者其他互联网网络设备上存储的表，该表中存有到达特定网络终端的路径，在某些情况下，还有一些与这些路径相关的度量。路由器的主要工作就是为经过路由器的每个数据包寻找一条最佳的传输路径，并将该数据有效地传送到目的站点。由此可见，选择最佳路径的策略即路由算法是路由器的关键所在。为了完成这项工作，在路由器中保存着各种传输路径的相关数据——路由表（Routing Table），供路由选择时使用，表中包含的信息决定了数据转发的策略。打个比方，路由表就像我们平时使用的地图一样，标识着各种路线，路由表中保存着子网的标志信息、网上路由器的个数和下一个路由器的名字等内容。路由表根据其建立的方法，可以分为动态路由表和静态路由表。


2、路由表分类

linux 系统中，可以自定义从 1－252 个路由表，其中，linux 系统维护了 4 个路由表：

0# 表： 系统保留表
253# 表： defulte table 没特别指定的默认路由都放在改表
254# 表： main table 没指明路由表的所有路由放在该表
255# 表： locale table 保存本地接口地址，广播地址、NAT 地址 由系统维护，用户不得更改

```BASH
[root@VM-16-10-tencentos ~]# cat /etc/iproute2/rt_tables
#
# reserved values
#
255     local
254     main
253     default
0       unspec
#
# local
#
#1      inr.ruhep
```

3、常用命令

路由表的查看可有以下二种方法：

ip route list table table_number
ip route list table table_name
路由表序号和表名的对应关系在 /etc/iproute2/rt_tables 文件中，可手动编辑。路由表添加完毕即时生效，下面为实例：

ip route add default via 192.168.1.1 table 1 在一号表中添加默认路由为 192.168.1.1
ip route add 192.168.0.0/24 via 192.168.1.2 table 1 在一号表中添加一条到 192.168.0.0 网段的路由为 192.168.1.2
使用 ip route , ip rule , iptables 配置策略路由

需求：公司内网要求 192.168.0.100 以内的使用 10.0.0.1 网关上网（电信），其他 IP 使用 20.0.0.1 （网通）上网。

实现：

1、首先要在网关服务器上添加一个默认路由，当然这个指向是绝大多数的 IP 的出口网关。

ip route add default gw 20.0.0.1

2、通过 ip route 添加一个路由表

ip route add table 3 via 10.0.0.1 dev ethX (ethx 是 10.0.0.1 所在的网卡, 3 是路由表的编号)

3、添加 ip rule 规则

ip rule add fwmark 3 table 3 （fwmark 3 是标记，table 3 是路由表 3 上边。 意思就是凡事标记了 3 的数据使用 table3 路由表）

4、使用 iptables 给相应的数据打上标记

iptables -A PREROUTING -t mangle -i eth0 -s 192.168.0.1 -192.168.0.100 -j MARK --set-mark 3

因为 mangle 的处理是优先于 nat 和 fiter 表的，所以相依数据包到达之后先打上标记，之后在通过 ip rule 规则，对应的数据包使用相应的路由表进行路由，最后读取路由表信息，将数据包送出网关。


这里可以看出 Netfilter 处理网络包的先后顺序：接收网络包，先 DNAT，然后查路由策略，查路由策略指定的路由表做路由，然后 SNAT，再发出网络包。

####    策略路由与路由表
Linux 从 2.2 版本左右的内核开始，便包含了多个路由表，而不是一个！同时，还有一套规则，这套规则会告诉内核如何为每个数据包选择正确的路由表。

当你执行 ip route 时，你看到的是一个特定的路由表 main，除了 main 之外还有其他的路由表存在。路由表一般用整数来标识，也可以通过文本对其命名，这些命名都保存在文件 /etc/iproute2/rt_tables 中。默认内容如下：

$ cat /etc/iproute2/rt_tables
#
# reserved values
#
255     local
254     main
253     default
0       unspec
#
# local
#
#1      inr.ruhep
复制
Linux 系统中，可以自定义从 1－252 个路由表。Linux 系统默认维护了 4 个路由表：

0：系统保留表。
253：defulte table。没特别指定的默认路由都放在该表。
254：main table。没指明路由表的所有路由放在该表。
255：locale table。保存本地接口地址，广播地址、NAT 地址，由系统维护，用户不得更改。
这里有一个很奇怪的单词：inr.ruhep，这可能是 Alexey Kuznetsov 添加的，他负责服务质量（QoS）在Linux内核中的实现，iproute2 也是他在负责，这个单词表示“核研究/俄罗斯高能物理研究所”，是 Alexey 当时工作的地方，可能指的是他们的内部网络。当然，还有另外一种可能，有一个老式的俄罗斯计算机网络/ISP 叫做 RUHEP/Radio-MSU[1]。

路由表的查看可有以下二种方法：

$ ip route show table table_number
 
$ ip route show table table_name
复制
❝不要把路由表和 iptables 混淆，路由表决定如何传输数据包，而 iptables 决定是否传输数据包，他俩的职责不一样。

路由策略
内核是如何知道哪个数据包应该使用哪个路由表的呢？答案已经在前文给出来了，系统中有一套规则会告诉内核如何为每个数据包选择正确的路由表，这套规则就是路由策略数据库。这个数据库由 ip rule 命令来管理，如果不加任何参数，将会打印所有的路由规则：

0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default
复制
左边的数字（0, 32764, ......）表示规则的优先级：数值越小的规则，优先级越高。也就是说，数值较小的规则会被优先处理。

❝路由规则的数值范围：1 ~ 

2
23
−
1
除了优先级之外，每个规则还有一个选择器（selector）和对应的执行策略（action）。选择器会判断该规则是否适用于当前的数据包，如果适用，就执行对应的策略。最常见的执行策略就是查询一个特定的路由表（参考上一节内容）。如果该路由表包含了当前数据包的路由，那么就执行该路由；否则就会跳过当前路由表，继续匹配下一个路由规则。

在 Linux 系统启动时，内核会为路由策略数据库配置三条缺省的规则：

0：匹配任何条件，查询路由表 local (ID 255)，该表 local 是一个特殊的路由表，包含对于本地和广播地址的优先级控制路由。rule 0 非常特殊，不能被删除或者覆盖。
32766：匹配任何条件，查询路由表 main (ID 254)，该表是一个常规的表，包含所有的无策略路由。系统管理员可以删除或者使用另外的规则覆盖这条规则。
32767：匹配任何条件，查询路由表 default (ID 253)，该表是一个空表，它是后续处理保留。对于前面的策略没有匹配到的数据包，系统使用这个策略进行处理，这个规则也可以删除。
在默认情况下进行路由时，首先会根据规则 0 在本地路由表里寻找路由，如果目的地址是本网络，或是广播地址的话，在这里就可以找到合适的路由；如果路由失败，就会匹配下一个不空的规则，在这里只有 32766 规则，在这里将会在主路由表里寻找路由；如果失败，就会匹配 32767 规则，即寻找默认路由表。如果失败，路由将失败。从这里可以看出，策略性路由是往前兼容的。

##  0x02    wireguard 中的路由

本节主要内容来源[彻底理解 WireGuard 的路由策略](https://cloud.tencent.com/developer/article/2153889)

####    问题起因

在网关主机上运行wireguard，所有的流量都通过 WireGuard 路由出去，但却无法通过 `ip route`的输出中看出路由走向；下面的路由全部是走`eth0`出入的，并未看到wireguard的虚拟网卡配置：

```BASH
default via 192.168.100.254 dev eth0 proto dhcp src 192.168.100.63 metric 100 
192.168.100.0/24 dev eth0 proto kernel scope link src 192.168.100.63 
192.168.100.254 dev eth0 proto dhcp scope link src 192.168.100.63 metric 100
```

猜想可能是配置在其他的路由表中，再看


WireGuard 全局路由策略
现在回到 WireGuard，很多 WireGuard 用户会选择将本机的所有流量通过 WireGuard 对端路由，原因嘛大家都懂得😁。

配置嘛也很简单，只需将 0.0.0.0/0 添加到 AllowedIPs 里即可：

```BASH
# /etc/wireguard/wg0.conf

[Interface]
PrivateKey =  
Address = 10.0.0.2/32
# PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
# PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
# ListenPort = 51820

[Peer]
PublicKey = 
Endpoint = 192.168.100.251:51820
AllowedIPs = 0.0.0.0/0  #一般会设置为将本机的所有流量通过 WireGuard 对端路由
```

But，理论上这样就可以让所有的流量都通过对端路由了，但是如果是旧版本的 `wg-quick` 版本比较旧，一顿操作猛如虎（wg-quick up wg0）之后，你会发现事情并不是你想象的那样，甚至可能连 WireGuard 对端都连不上了。主要还是因为 WireGuard 自身的流量也通过虚拟网络接口进行路由了，这肯定是不行的。

新版本的 wg-quick 通过路由策略巧妙地解决了这个问题，我们来看看它妙在何处！

####    新版本wg-quick的解决

首先，使用 wg-quick 启动 wg0 网卡：

```bash
$ wg-quick up wg0
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.0.0.2/32 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] wg set wg0 fwmark 51820
[#] ip -4 route add 0.0.0.0/0 dev wg0 table 51820
[#] ip -4 rule add not fwmark 51820 table 51820
[#] ip -4 rule add table main suppress_prefixlength 0
[#] sysctl -q net.ipv4.conf.all.src_valid_mark=1
[#] iptables-restore -n
```


嘻嘻，看到了熟悉的路由策略，这就打印所有的路由规则看看：

$ ip rule
0:      from all lookup local
32764:  from all lookup main suppress_prefixlength 0
32765:  not from all fwmark 0xca6c lookup 51820
32766:  from all lookup main
32767:  from all lookup default
复制
好家伙，多了两条规则：

32764:  from all lookup main suppress_prefixlength 0
32765:  not from all fwmark 0xca6c lookup 51820
复制
我们来扒扒他们的底裤，揭开神秘面纱。先来灵魂三问：suppress_prefixlength 是啥？0xca6c 又是啥？数据包怎么可能 not from all？

Rule 32764
先从规则 32764 开始分析，因为它的数值比较小，会被优先匹配：

32764:  from all lookup main suppress_prefixlength 0
复制
这条规则没有使用选择器，也就是说，内核会为每一个数据包去查询 main 路由表。我们来看看 main 路由表内容是啥：

$ ip route
default via 192.168.100.254 dev eth0 proto dhcp src 192.168.100.63 metric 100 
192.168.100.0/24 dev eth0 proto kernel scope link src 192.168.100.63 
192.168.100.254 dev eth0 proto dhcp scope link src 192.168.100.63 metric 100
复制
如果真的是这样，那所有的数据包都会通过 main 路由表路由，永远不会到达 wg0。你别忘了，这条规则末尾还有一个参数：suppress_prefixlength 0，这是啥意思呢？参考  ip-rule(8) man page：

suppress_prefixlength NUMBER
    reject routing decisions that have a prefix length of NUMBER or less.
复制
这里的 prefix 也就是前缀，表示路由表中匹配的地址范围的掩码。因此，如果路由表中包含 10.2.3.4 的路由，前缀长度就是 32；如果是 10.0.0.0/8，前缀长度就是 8。

suppress 的意思是抑制，所以 suppress_prefixlength 0 的意思是：拒绝前缀长度小于或等于 0 的路由策略。

那么什么样的地址范围前缀长度才会小于等于 0？只有一种可能：0.0.0.0/0，也就是默认路由。以我的机器为例，默认路由就是：

default via 192.168.100.254 dev eth0 proto dhcp src 192.168.100.63 metric 100
复制
如果数据包匹配到了默认路由，就拒绝转发；如果是其他路由，就正常转发。

这条规则的目的很简单，管理员手动添加到 main 路由表中的路由都会正常转发，而默认路由会被忽略，继续匹配下一条规则。

Rule 32765
下一条规则就是 32765：

32765:  not from all fwmark 0xca6c lookup 51820
复制
这里的 not from all 是 ip rule 格式化的问题，有点反人类，人类更容易理解的顺序应该是这样：

32765:  from all not fwmark 0xca6c lookup 51820
复制
从前面 wg-quick up wg0 的输出来看，规则的选择器是没有添加 from 前缀（地址或者地址范围）的：

ip -4 rule add not fwmark 51820 table 51820
复制
如果规则选择器没有 from 前缀，ip rule 就会打印出 from all，所以这条规则才会是这个样子。

51820 是一个路由表，也是由 wg-quick 创建的，只包含一条路由：

$ ip route show table 51820
default dev wg0 scope link
复制
所以这条规则的效果是：匹配到该规则的所有数据包都通过 WireGuard 对端进行路由，除了 not fwmark 0xca6c。

0xca6c 只是一个防火墙标记，wg-quick 会让 wg 标记它发出的所有数据包（wg set wg0 fwmark 51820），这些数据包已经封装了其他数据包，如果这些数据包也通过 WireGuard 进行路由，就会形成一个无限路由环路。

所以 not from all fwmark 0xca6c lookup 51820 意思是说，满足条件 from all fwmark 0xca6c（WireGuard 发出的都带 fwmark 0xca6c）请忽略本条规则，继续往下走。否则，请使用 51820 路由表，通过 wg0 隧道出去。

对于 wg0 接口发包自带的 0xca6c，继续走下一条规则，也就是匹配默认的 main 路由表：

32766:  from all lookup main
复制
此时已经没有抑制器了，所有的数据包都可以自由使用 main 路由表，因此 WireGuard 对端的 Endpoint 地址会通过 eth0 接口发送出去。

完美！

❝wg-quick 创建的路由表和 fwmark 使用的是同一个数字：51820。0xca6c 是 51820 的十六进制表示。

总结
wg-quick 这种做法的巧妙之处在于，它不会扰乱你的主路由表，而是通过规则匹配新创建的路由表。断开连接时只需删除这两条路由规则，默认路由就会被重新激活。你学废了吗？

##  0x03    clash.Meta tun模式中的路由

```BASH
[root@VM-16-10-tencentos ~]# ip route
default via 172.16.16.1 dev eth0 
172.16.16.0/20 dev eth0 proto kernel scope link src 172.16.16.10 metric 100 
198.18.0.0/16 dev utun proto kernel scope link src 198.18.0.1 
```

```bash
[root@VM-16-10-tencentos ~]# ip rule
0:      from all lookup local
9500:   not from all dport 53 lookup main suppress_prefixlength 0
9510:   not from all iif lo lookup 1970566510
9520:   from 0.0.0.0 iif lo uidrange 0-4294967294 lookup 1970566510
9530:   from 198.18.0.1 iif lo uidrange 0-4294967294 lookup 1970566510
32766:  from all lookup main
32767:  from all lookup default
```


##  0x0 参考
-   [详解ip rule，ip route，iptables 三者之间的关系](https://www.toutiao.com/article/6662630661838340621/?wid=1692243899353)
-   [理解 OpenStack 高可用（HA）（3）：Neutron 分布式虚拟路由（Neutron Distributed Virtual Routing）](https://www.cnblogs.com/sammyliu/p/4713562.html)
-   [彻底理解 WireGuard 的路由策略](https://cloud.tencent.com/developer/article/2153889)
-   [在WireGuard场景中使用策略路由定义复杂路由规则](https://blog.imlk.top/posts/policy-routing-with-wg-tunnel/)
-   [Linux 上的 WireGuard 网络分析（一）](https://thiscute.world/posts/wireguard-on-linux/)