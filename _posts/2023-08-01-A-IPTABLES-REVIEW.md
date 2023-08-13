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

![netfilter]()

##  0x01    iptables 的四表五链

四表五链，链是很重要的概念，每个链都包含了若干个表的元素（这里的元素代表若干条 iptables 规则），所有同一类的规则构成了 iptables 的表。注意每一个链对应的表都是不完全一样的，表和链之间是多对多的对应关系

![](blog_img/mitm/iptables/iptables-2.png)


链代表位置：
-   `PREROUTING`：在对数据包作路由选择之前，应用此链中的规则
-   `INPUT`：当接收到防火墙本机地址的数据包（入站）时，应用此链中的规则
-   `FORWARD`：当接收到需要通过防火墙发送给其他地址的数据包（转发）时，应用此链中的规则
-   `OUTPUT`：当防火墙本机向外发送数据包（出站）时，应用此链中的规则
-   `POSTROUTING`：在对数据包作路由选择之后，应用此链中的规则

注意：`INPUT`、`OUTPUT` 链更多的应用在 ** 主机防火墙 ** 中，即主要针对服务器本机进出数据的安全控制；而 `FORWARD`、`PREROUTING`、`POSTROUTING` 链更多的应用在 ** 网络防火墙 ** 中，特别是防火墙服务器作为网关使用时的情况

表代表了存储的规则（链式）：数据包到了该链处，会去对应表中查询设置的规则，然后决定是否放行、丢弃、转发还是修改等等操作

-   `raw`
-   `mangle`
-   `nat`
-   `filter`


| 名称 | 功能（位置） | 包含链 | 典型场景（作用） |
|  -------- |-------- |-------- |-------- |
|filter 表 | 负责过滤功能，防火墙 |包含三个规则链：INPUT，FORWARD，OUTPUT||
|nat（Network Address Translation）表 | 用于网络地址转换（IP、端口）|包含三个规则链：PREROUTING，POSTROUTING，OUTPUT||
|mangle 表 | 拆解报文，作出修改，封装报文 |包含五个规则链：PREROUTING，POSTROUTING，INPUT，OUTPUT，FORWARD||
|raw 表 | 关闭 nat 表上启用的连接追踪机制，确定是否对该数据包进行状态跟踪 |包含两条规则链：OUTPUT，PREROUTING||


注意每一个链对应的表都是不完全一样的，表和链之间是多对多的对应关系。每个链中的表，按照下面的优先顺序进行查找匹配：

```BASH
raw>mangle>nat>filter
```

####    链的作用



| 名称 | 功能（位置） | 表优先级顺序 | 典型场景（作用） |
|  -------- |-------- |-------- |-------- |
| PREROUTING | 数据包进入路由之前 | |  |
| INPUT | 数据包进入路由之前 | |  |
| OUTPUT | 原地址为本机，向外发送 | |  |
| FORWARD | 实现转发 | |  |
| POSTROUTING | 发送到网卡之前 | |  |


####   链与表的关系

iptables 的链就是因为在访问该链的时候会按照每个链对应的表依次进行查询匹配执行的操作。如 `PREROUTING` 链对应的表关系是（`raw->mangle->nat`），每个表按照优先级顺序进行链接，此外，每个表中还可能有多个规则，因此最后看起来就像链一样，如下图：

![relation]()


当数据包抵达防火墙时，如上图所示，将依次应用raw、mangle、nat和filter表中对应链内的规则。

##  0x0 典型配置




##  0x  透明代理配置



##  参考
-   [深入浅出带你理解 iptables 原理](https://zhuanlan.zhihu.com/p/547257686)
-   [iptables 的四表五链与 NAT 工作原理 _](https://tinychen.com/20200414-iptables-principle-introduction/)