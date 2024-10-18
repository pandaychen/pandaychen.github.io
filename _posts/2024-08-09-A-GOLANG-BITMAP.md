---
layout:     post
title:      数据结构与算法回顾（九）：bitmap
subtitle:
date:       2024-08-09
author:     pandaychen
header-img:
catalog: true
tags:
    - Golang
    - 数据结构
---

##  0x00    前言

##  0x01    原理 && 应用
bitmap比较简单，用一个 bit 来标记某个元素对应的 value，而 Key 即是该元素。由于采用bit 来存储一个数据，相对节省空间（不考虑稀疏存储的场景下）。假设要对 `0-31` 内的 `3` 个元素 `(10,17,28)` 排序，那么就可以采用 Bitmap 方法（假设这些元素没有重复）

要表示 `32` 个数，我们就只需要 `32` 个 bit（`4`Bytes），首先初始化 `4`Byte 的空间，将这些空间的所有 bit 位都置为 `0`

![B1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/datastructure/bitmap/bitmap-1.jpg)

然后，添加`(10,17,28)` 到 BitMap 中，需要的操作就是在相应的位置上将`0`置为`1`即可

![B1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/datastructure/bitmap/bitmap-2.jpg)


##  0x02  XDP 中的应用
这里介绍下 bitmap 在字节 EBPF-ACL 项目中的巧妙应用。先介绍下背景，对应与iptables的链式规则：

```BASH
# iptables -A INPUT -m set --set black_list src -j DROP
 
# ......省略N条规则
 
# iptables -A INPUT -p tcp -m multiport --dports 53,80 -j ACCEPT
 
# ......省略N条规则

# iptables -A INPUT -p udp --dport 53 -j ACCEPT
```
若匹配 DNS 的请求报文（目的端口 `53` 的 udp 报文），需要依次遍历所有规则，才能匹配中。其中，ipset/multiport 等 match 项，只能减少规则数量，无法改变 `O(N)`的匹配方式

问题就来了，如何设计巧妙的数据结构来解决iptables链式匹配的低效问题（在xdp程序中）


####    算法
匹配算法
为了提升匹配效率，我们将所有的 ACL 规则做了预处理，将链式的规则拆分存储。规则匹配时，我们参考内核的 O(1)调度算法，在多个匹配的规则中，快速选取高优先级的规则。

规则预处理
我们以之前的 iptables 规则举例，看如何将其拆分存储。首先，将所有规则根据优先级编号。比如，例子中的规则分别编号为：1(0x1)、16(0x10)、256(0x100)。其次，将所有规则的匹配项归类拆分。比如，例子中的匹配项可以归类为：源地址、目的端口、协议。最后，将规则编号、具体匹配项分类存储到 eBPF 的 Map 中。

# ipset create black_list hash:net
# ipset add black_list 192.168.3.0/24
 
# iptables -A INPUT -m set --set black_list src -j DROP
 
# ...... 省略15条规则
 
# iptables -A INPUT -p tcp -m multiport --dports 53,80 -j ACCEPT
 
# ...... 省略240条规则
 
# iptables -A INPUT -p udp --dport 53 -j ACCEPT
举例说明：
规则 1 只有源地址匹配项，我们用源地址 192.168.3.0/24 作为 key，规则编号 0x1 作为 value，存储到 src Map 中。

规则 16 有目的端口、协议 2 个匹配项，我们依次将 53、80 作为 key，规则编号 0x10 作为 value，存储到 dport Map 中；将 tcp 协议号 6 作为 key，规则编号 0x10 作为 value，存储到 proto Map 中。

规则 256 有目的端口、协议 2 个匹配项，我们将 53 作为 key，规则编号 0x100 作为 value，存储到 sport Map 中；将 udp 协议号 17 作为 key，规则编号 0x100 作为 value，存储到 proto Map 中。

我们依次将规则 1、16、256 的规则编号作为 key，动作作为 value，存储到 action Map 中。


交集
需要注意的是，规则 16、256 均有目的端口为 53 的匹配项，我们应该将 16、256 的规则编号进行按位或操作，然后进行存储，即 0x10 | 0x100 = 0x110。

通配
另外，规则 1 的目的端口、协议均为通配项，我们应该将规则 1 的编号按位或追加到现有的匹配项中（图中下划线的 value 值：0x111、0x11 等）。同理，将规则 16、256 的规则编号按位或追加到现有的源地址匹配项中（图中下划线的 value 值：0x111、0x110）。至此，我们的规则预处理完成，将所有规则的匹配项、动作拆分存储到 6 个 eBPF Map 中，如上图所示。

类 O(1)匹配
报文匹配时，我们报文的 5 元组（源、目的地址，源、目的端口、协议）依次作为 key，分别查找对应的 eBPF Map，得到 5 个 value。我们将这 5 个 value 进行按位与操作，得到一个 bitmap。这个 bitmap 的每个 bit，就表示了对应的一条规则；被置位为 1 的 bit，表示对应的规则匹配成功。


举例说明：
当用报文（192.168.4.1:10000 -> 192.168.4.100:53, udp）的 5 元组作为 key，查找 eBPF Map 后，得到的 value 分别为：src_value = 0x110、dst_value = NULL、sport_value = NULL、dport_value = 0x111、proto_value = 0x101。将非 NULL 的 value 进行按位与操作，得到 bitmap = 0x100（0x110 & 0x111 & 0x101）。由于 bitmap 只有一位被置位 1（只有一条规则匹配成功，即规则 256），利用该 bitmap 作为 key，查找 action Map，得到 value 为 ACCEPT。与 iptables 的规则匹配结果一致。

同理，当报文（192.168.4.1:1000 -> 192.168.4.100:53, tcp）的 5 元组作为 key，查找 eBPF Map 后，处理的流程和上面的一致，最终匹配到规则 16。

同样，当报文（192.168.3.1:1000 -> 192.168.4.100:53, udp）的 5 元组作为 key，查找 eBPF Map 后，处理的流程和上面的一致。不同的是，得到的 bitmap = 0x101，由于 bitmap 有两位被置位 1（规则 1、256），我们应该取优先级最高的规则编号作为 key，查找 action Map。这里借鉴了内核 O(1)调度算法的思想，做如下操作：

bitmap &= -bitmap
即可取到优先级最高的 bit，如 0x101 &= -(0x101)最终等于 0x1。我们用 0x1 作为 key，查找 action Map，得到 value 为 DROP。与 iptables 的规则匹配结果一致。

总结
本文基于 Linux 内核的 XDP 机制，提出了一种改善 iptables / nftables 性能的 ACL 方案。目前 eBPF 的技术在开源社区非常流行，特性非常丰富，我们可以利用这项技术做很多有意思的事情。感兴趣的朋友可以加入我们，一起讨论交流。
————————————————

                            版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
                        
原文链接：https://blog.csdn.net/ByteDanceTech/article/details/106632252

##  0x03    参考
-   [Golang 优化之路——bitset](https://blog.cyeam.com/golang/2017/01/18/go-optimize-bitset)
-   [Go 每日一库之 roaring](https://darjun.github.io/2022/07/17/godailylib/roaring/)
-   [eBPF 技术实践：高性能 ACL](https://blog.csdn.net/ByteDanceTech/article/details/106632252)
-   [Go 每日一库之 bitset](https://darjun.github.io/2022/07/16/godailylib/bitset/)