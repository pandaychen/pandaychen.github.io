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
bitmap 比较简单，用一个 bit 来标记某个元素对应的 value，而 Key 即是该元素。由于采用 bit 来存储一个数据，相对节省空间（不考虑稀疏存储的场景下）。假设要对 `0-31` 内的 `3` 个元素 `(10,17,28)` 排序，那么就可以采用 Bitmap 方法（假设这些元素没有重复）。bitmap 算法常用于对大量整形数据做去重和查询

要表示 `32` 个数，我们就只需要 `32` 个 bit（`4`Bytes），首先初始化 `4`Byte 的空间，将这些空间的所有 bit 位都置为 `0`

![B1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/datastructure/bitmap/bitmap-1.jpg)

然后，添加 `(10,17,28)` 到 BitMap 中，需要的操作就是在相应的位置上将 `0` 置为 `1` 即可

![B1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/datastructure/bitmap/bitmap-2.jpg)

由于 bitmap 用一个比特位来映射某个元素的状态，此数据结构非常节省存储空间，常用如下：

-   数据去重
-   快速查找，判重
-   排序（将其加入 bitmap 中，然后再遍历获取出来，从而得到排序的结果）

####    常用实现：golang
以 `[]uint64` 来存储 bit 数据为例，一个 byte 有 `64` 个二进制位，当然亦可使用 `[]byte` 或其他类型来存储（`8` 个二进制位）

1、（优先）判断元素在 bit 数组的位置，假设参数为 `x`

-   `x/64`：获取 `x` 在 bit 数组中的序号，从 `0` 开始
-   `x%64`：获取 `x` 在当前 `uint64` 中的位置

```GO
// Bitmap represents a scalar-backed bitmap index
type Bitmap []uint64
```

2、检查 `x` 是否存在

```GO
// Contains checks whether a value is contained in the bitmap or not.
func (dst Bitmap) Contains(x uint32) bool {
	blkAt := int(x>> 6)
	if size := len(dst); blkAt >= size {
		return false
	}

	bitAt := int(x % 64)
    // 将 1 左移 bitAt 位，然后和以前的数据做 & 运算，若该位置的值 >=0，则说明 x 存在，否则则不存在
	return (dst[blkAt] & (1 << bitAt)) > 0
}
```

3、设置 `x` 到 bit 数组

```GO
// Set sets the bit x in the bitmap and grows it if necessary.
func (dst *Bitmap) Set(x uint32) {
	blkAt := int(x>> 6)
	bitAt := int(x % 64)
	if size := len(*dst); blkAt >= size {
		dst.grow(blkAt) // 扩容
	}

    // 将 1 左移 bitAt 位，然后和以前的数据做 | 运算，可将 bitAt 位置的 bit 替换成 1
	(*dst)[blkAt] |= (1 << bitAt)
}
```

4、

```GO
// Remove removes the bit x from the bitmap, but does not shrink it.
func (dst *Bitmap) Remove(x uint32) {
	if blkAt := int(x>> 6); blkAt < len(*dst) {
		bitAt := int(x % 64)
        // 将 1 左移 bitAt 位，然后对取反，再与当前值做 &，可将 bitAt 位置的 bit 替换成 0
		(*dst)[blkAt] &^= (1 << bitAt)
	}
}
```

除了基本操作之外，原项目还提供了 `Max` 和 `Min` 方法，本质就是从尾（头）遍历，获取第一个不为 `0` 的数据

```GO
// Min get the smallest value stored in this bitmap, assuming the bitmap is not empty.
func (dst Bitmap) Min() (uint32, bool) {
	for blkAt, blk := range dst {
		if blk != 0x0 {
			return uint32(blkAt<<6 + bits.TrailingZeros64(blk)), true
		}
	}

	return 0, false
}

// Max get the largest value stored in this bitmap, assuming the bitmap is not empty.
func (dst Bitmap) Max() (uint32, bool) {
	var blk uint64
	for blkAt := len(dst) - 1; blkAt >= 0; blkAt-- {
		if blk = dst[blkAt]; blk != 0x0 {
			return uint32(blkAt<<6 + (63 - bits.LeadingZeros64(blk))), true
		}
	}
	return 0, false
}
```

##  0x02  XDP 中的应用
这里介绍下 bitmap 在字节 EBPF-ACL 项目中的巧妙应用。先介绍下背景，对应与 iptables 的链式规则：

```BASH
# ipset create black_list hash:net
# ipset add black_list 192.168.3.0/24

# iptables -A INPUT -m set --set black_list src -j DROP #序号 1

# ...... 省略 15 条规则

# iptables -A INPUT -p tcp -m multiport --dports 53,80 -j ACCEPT    #序号 16

# ...... 省略 240 条规则

# iptables -A INPUT -p udp --dport 53 -j ACCEPT     #序号 256
```

若匹配 DNS 的请求报文（目的端口 `53` 的 udp 报文），需要依次遍历所有规则，才能匹配中。其中，ipset/multiport 等 `match` 项，只能减少规则数量，无法改变 `O(N)` 的匹配方式

问题就来了，如何设计巧妙的数据结构来解决 iptables 链式匹配的低效问题（在 xdp 程序中）


####    算法描述

借助于 bitmap 来实现，为了提升匹配效率，将所有的 ACL 规则做预处理，将链式的规则拆分存储。规则匹配时，参考内核的 `O(1)` 调度算法，在多个匹配的规则中，快速选取高优先级的规则

1、规则预处理

按照如下六个维度将 iptables 链式规则拆分，核心思路将规则生成 `6` 张表 bitmap，其中 key 为关联各个表的属性，value 为 bitmap

-   规则编号：key 为规则编号，value 存储 action 结果
-   源地址：key 为源地址，value bitmap 存储规则编号
-   源端口：key 为端口号
-   目的地址：同上
-   目的端口：同上
-   协议：key 为协议

此外，对于作为 value 的 bitmap，`1` 表示该规则有该属性，`0` 表示该规则无该属性

用上面的例子详细说明下：

1.  例子中的规则分别编号为：`1(0x1)`、`16(0x10)`、`256(0x100)`，接下来拆分各个规则中的匹配项
2.  规则 `1` 只有源地址匹配项，用源地址 `192.168.3.0/24` 作为 key，规则编号 `0x1` 作为 value，存储到源地址 Map 中
3.  规则 `16` 有目的端口、协议包含两个匹配项（协议、目的端口），依次将 `53`、`80` 作为 key，规则编号 `0x10` 作为 value，存储到 dport Map 中；将 tcp 协议号 `6` 作为 key，规则编号 `0x10` 作为 value，存储到 proto Map 中
4.  规则 `256` 有目的端口、协议两个匹配项，将目的端口 `53` 作为 key，规则编号 `0x100` 作为 value，存储到 sport Map 中；将 udp 协议号 `17` 作为 key，规则编号 `0x100` 作为 value，存储到 proto Map 中
5. 依次将规则 `1`、`16`、`256` 的规则编号作为 key，动作作为 value，存储到 action Map 中

此外，还需要补充下特殊规则：
6.  相同 key 合并：规则编号 `16`、`256` 均有目的端口为 `53` 的匹配项，应该将 `16`、`256` 的规则编号进行按位或操作，然后再存储，即 `0x10 | 0x100 = 0x110`（注意这里是 16 进制）
7.  通配规则的处理（规则 `1`）：规则 `1` 的目的端口、协议均为通配项，应该将规则 `1` 的编号按位或追加到现有的匹配项中（下图中下划线的 value 值：`0x111`、`0x11` 等）。同理，将规则 `16`、`256` 的规则编号按位或追加到现有的源地址匹配项中（下划线的 value 值：`0x111`、`0x110`）

按照上面的描述，规则预处理完成，将所有规则的匹配项、动作拆分存储到 `6` 个 eBPF Map 中，如下图：

![bit1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/bytedance/bitmap-1.png)

####    匹配规则
报文匹配时，根据五元组（源、目的地址，源、目的端口、协议）依次作为 key，分别查找对应的 eBPF Map，得到 `5` 个 value。我们将这 `5` 个 value 进行按位与操作，得到一个 bitmap。这个 bitmap 的每个 bit，就表示了对应的一条规则；被置位为 `1` 的 bit，表示对应的规则匹配成功，一般有两种情况

1.	最终的 bitmap 只有 `1` 位被置位
2.	最终的 bitmap 超过 `1` 位被置位，此时需要获取优先级最高的那条（数字最小）


![bit2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/bytedance/bitmap-2.png)

根据上图举例说明，同时验证下是否与 iptables 的规则匹配结果一致。假设有报文（`192.168.4.1:10000 -> 192.168.4.100:53` udp）的五元组作为 key，查找 eBPF Map 后，得到的 value 分别为：

-   `src_value = 0x110`
-   `dst_value = NULL`
-   `sport_value = NULL`
-   `dport_value = 0x111`
-   `proto_value = 0x101`

将非 NULL 的 value 进行按位与操作（`0x110 & 0x111 & 0x101`），得到 bitmap = `0x100`。由于 bitmap 只有一位被置位 `1`（只有一条规则匹配成功，即规则 `256`），利用该 bitmap 作为 key，查找 action Map，得到 value 为 `ACCEPT`；同样，报文（`192.168.4.1:1000 -> 192.168.4.100:53` tcp）的五元组作为 key，查找 eBPF Map 后，最终匹配到规则 `16`

第二种情况，当报文（`192.168.3.1:1000 -> 192.168.4.100:53` udp）的五元组作为 key，计算结果为 `0x101`（多个 bit 被置位），由于 bitmap 有两位被置位（规则 `1`、`256`），按规则应该取优先级最高的规则编号作为 key，查找 action Map。在字节的描述中，借鉴了内核 `O(1)` 调度算法的思想，使用指令 `bitmap &= -bitmap` 即可取到优先级最高的 bit（如 `0x101 &= -(0x101)` 最终等于 `0x1`），最后使用 `0x1` 作为 key，查找 action Map，得到 value 为 `DROP`


####    热更新（CRUD）

##  0x03    参考
-   [Golang 优化之路——bitset](https://blog.cyeam.com/golang/2017/01/18/go-optimize-bitset)
-   [Go 每日一库之 roaring](https://darjun.github.io/2022/07/17/godailylib/roaring/)
-   [eBPF 技术实践：高性能 ACL](https://blog.csdn.net/ByteDanceTech/article/details/106632252)
-   [Go 每日一库之 bitset](https://darjun.github.io/2022/07/16/godailylib/bitset/)
-   [Simple dense bitmap index in Go with binary operators](https://github.com/kelindar/bitmap/blob/main/bitmap.go)