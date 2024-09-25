---
layout:     post
title:      golang eBPF 开发入门（五）
subtitle:   网络开发之 XDP/TC
date:       2024-08-30
author:     pandaychen
catalog:    true
tags:
    - eBPF
    - XDP
    - TC
---


##  0x00    前言
XDP 与 TC 的位置：

- XDP：Ingress
- TC：Egress

![xdp&TC](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/XDP_integration_with_linux_network_stack.png)


XDP VS DPDK：

![xdp-dpdk](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/dpdk_vs_xdp.png)


参考文档：
- [XDP tutorial](https://github.com/xdp-project/xdp-tutorial)
- [Making eBPF programming easier via build env and examples](https://github.com/xdp-project/bpf-examples)

####  应用场景

- XDP：ACL（防火墙）
- LB：如[katran](https://github.com/facebookincubator/katran)

##  0x01  XDP 基础
XDP 全称为 eXpress Data Path，是 Linux 内核网络栈的最底层，只存在于 RX 路径（Ingress）上，允许在网络设备驱动内部网络堆栈中数据来源最早的地方进行数据包处理，在特定模式下可以在操作系统分配内存（skb）之前就已经完成处理。XDP 暴露了一个可以加载 BPF 程序的 hook，hook 程序能够对传入的数据包进行任意修改和快速决策，避免了内核内部处理带来的额外开销

####  XDP 输入参数
XDP hook 的输入参数类型为 `struct xdp_md`（在内核头文件 bpf.h 中定义）

```C
/* user accessible metadata for XDP packet hook
 * new fields must be added to the end of this structure
 */
struct xdp_md {
  __u32 data;
  __u32 data_end;
  __u32 data_meta;
  /* Below access go through struct xdp_rxq_info */
  __u32 ingress_ifindex; /* rxq->dev->ifindex */
  __u32 rx_queue_index;  /* rxq->queue_index  */
};
```

程序执行时，`data` 和 `data_end` 字段分别是数据包开始和结束的指针，它们是用来获取和解析传来的数据，第三个值是 `data_meta` 指针，初始阶段它是一个空闲的内存地址，供 XDP 程序与其他层交换数据包元数据时使用。最后两个字段分别是接收数据包的接口和对应的 RX 队列的索引。当访问这两个值时，BPF 代码会在内核内部重写，以访问实际持有这些值的内核结构 `struct xdp_rxq_info`

####  XDP 输出参数
在处理完一个数据包后，XDP 程序（hook 函数）会返回一个动作（Action）作为输出，它代表了程序退出后对数据包应该做什么样的最终裁决。定义了以下 `5` 种动作类型（前面 `4` 个动作不需要参数，最后一个动作需要额外指定一个 NIC 网络设备名称，作为转发这个数据包的目的地）

```C
enum xdp_action {
  XDP_ABORTED = 0, // Drop packet while raising an exception
  XDP_DROP, // Drop packet silently
  XDP_PASS, // Allow further processing by the kernel stack
  XDP_TX, // Transmit from the interface it came from
  XDP_REDIRECT, // Transmit packet from another interface
};
```

####  XDP 数据包的路径

1、网络数据包传输路径

![origin](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/packet-flow-origin.png)

2、启用 XDP 后，网络包传输路径（三种模式）

![xdp](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/packet-flow-xdp.png)

- `offload` 模式：XDP 程序直接 hook 到可编程网卡硬件设备上，处理性能最强；由于处于数据链路的最前端，过滤效率也是最高的。如果需要使用这种模式，需要在加载程序时明确声明（目前支持较少）
- `native` 模式：XDP 程序 hook 到网络设备的驱动上，是 XDP 最原始的模式，因为还是先于操作系统进行数据处理，它的执行性能还是很高的，当然需要网络驱动支持，目前已知的有 `i40e`、 `nfp`、`mlx` 系列和 `ixgbe` 系列
- `generic` 模式：这是 os 内核提供的通用 XDP 兼容模式，可以在没有硬件或驱动程序支持的主机上执行 XDP 程序。在这种模式下，XDP 的执行是由操作系统本身来完成的，以模拟 `native` 模式执行。缺点是由于是仿真执行，需要分配额外的套接字缓冲区（SKB），导致处理性能下降，跟 `native` 模式在 `10` 倍左右的差距

当前主流内核版本的 Linux 系统在加载 XDP BPF 程序时，会自动在 `native` 和 `generic` 这两种模式选择，完成加载后，可以使用 `ip` 命令行工具来查看选择的模式

##  0x02  XDP：实战

####  SSH端口访问限制

##  0x03    参考
- [每秒 1 百万的包传输，几乎不耗 CPU 的那种](https://colobu.com/2023/04/02/support-1m-pps-with-zero-cpu-usage/)
- [eBPF 技术实践：高性能 ACL](https://blog.csdn.net/ByteDanceTech/article/details/106632252)
- [eBPF Talk: 解密 XDP generic 模式](https://mp.weixin.qq.com/s?__biz=MjM5MTQxNTk5MA==&mid=2247483972&idx=1&sn=87bce22ffb54edda9d6c2556cfc8c36c&chksm=a6b4a99d91c3208bb19a893f716c090dd637a85c528b0ea4a9ef405709fc53c128b0af561823&scene=21#wechat_redirect)
- [你的第一个 XDP BPF 程序](https://davidlovezoe.club/wordpress/archives/937)
- [Deep Dive into Facebook's BPF edge firewall](https://cilium.io/blog/2018/11/20/fb-bpf-firewall/)
- [你的第一个 TC BPF 程序](https://cloud.tencent.com/developer/article/1626377)
- [XDP-tutorial 学习如何编写 eBPF XDP 程序]()