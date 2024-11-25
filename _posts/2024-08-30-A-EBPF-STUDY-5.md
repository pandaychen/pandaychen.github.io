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
在 linux 内核网络协议栈中有多个网络钩子，数据包在进入到网卡再到流出网卡的过程会触发这些钩子上注册的回调函数执行相关过滤动作。如 netfilter 框架中的 `5` 个钩子，针对 ip 数据包进行过滤。除此之外，在更低一层还有 xdp 和 tc 系统对数据包进行处理

![netfilter](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/ebpf-netfilter-iptables.png)

####    XDP 与 TC 的位置

- XDP：Ingress
- TC：Egress

![xdp&TC](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/XDP_integration_with_linux_network_stack.png)

####    XDP VS DPDK

![xdp-dpdk](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/dpdk_vs_xdp.png)

####    XDP VS IPTABLES
![ebpf-iptables](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/ebpf-netfilter-iptables.png)


####    TC

![tc](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/ebpf/arch-running-program.jpg)


参考文档：
- [XDP tutorial](https://github.com/xdp-project/xdp-tutorial)
- [Making eBPF programming easier via build env and examples](https://github.com/xdp-project/bpf-examples)

####  应用场景

- XDP：ACL（防火墙）
- LB：如 [katran](https://github.com/facebookincubator/katran)

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

对 `data` 和 `data_end` 的理解：


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

####  SSH 端口访问限制
```c
int xdp_firewall(struct xdp_md *ctx)
{
    // Cast the numerical addresses to pointers for packet data access
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    // Define a pointer to the Ethernet header at the start of the packet data
    struct ethhdr *eth = data;
    // Ensure the packet includes a full Ethernet header; if not, we let it continue up the stack
    if (data + sizeof(struct ethhdr) > data_end)
    {
        return XDP_PASS;
    }

    // Check if the packet's protocol indicates it's an IP packet
    if (eth->h_proto != __constant_htons(ETH_P_IP))
    {
        // If not IP, continue with regular packet processing
        return XDP_PASS;
    }

    // Access the IP header positioned right after the Ethernet header
    struct iphdr *ip = data + sizeof(struct ethhdr);
    // Ensure the packet includes the full IP header; if not, pass it up the stack
    if (data + sizeof(struct ethhdr) + sizeof(struct iphdr) > data_end)
    {
        return XDP_PASS;
    }

    // Confirm the packet uses TCP by checking the protocol field in the IP header
    if (ip->protocol != IPPROTO_TCP)
    {
        return XDP_PASS;
    }

    // Locate the TCP header that follows the IP header
    struct tcphdr *tcp = data + sizeof(struct ethhdr) + sizeof(struct iphdr);
    // Validate that the packet is long enough to include the full TCP header
    if (data + sizeof(struct ethhdr) + sizeof(struct iphdr) + sizeof(struct tcphdr) > data_end)
    {
        return XDP_PASS;
    }

    // Check if the destination port of the packet is the one we're monitoring
    if (tcp->dest != __constant_htons(22)) {
        return XDP_PASS;
    }

    // Construct the key for the lookup by using the source IP address from the IP header
    __u32 key = ip->saddr;
    // Attempt to find this key in the 'allowed_ips' map
    __u32 *value = allowed_ips.lookup(&key);
    if (value) {
        // If a matching key is found, the packet is from an allowed IP and can proceed
        bpf_trace_printk("Authorized TCP packet to ssh !\\n");
        return XDP_PASS;
    }

    // If no matching key is found, the packet is not from an allowed IP and will be dropped
    bpf_trace_printk("Unauthorized TCP packet to ssh !\\n");

    // drop packet
    return XDP_DROP;
}
```

####  DROP-TCP
示例 2，通过 xdp 拦截所有系统的 packet，如果是 TCP 报文则全部丢弃
```C
/*
  check whether the packet is of TCP protocol
*/
static bool is_TCP(void *data_begin, void *data_end){
  struct ethhdr *eth = data_begin;

  // Check packet's size
  // the pointer arithmetic is based on the size of data type, current_address plus int(1) means:
  // new_address= current_address + size_of(data type)

  // 重要！注意(eth + 1)前面加了一个显示类型转换，如果不做这个操作，编译时会有如下warning
  // warning: comparison of distinct pointer types ('struct ethhdr *' and 'void *')
  //代码里其他类似这样的显示类型转换都是出于规避编译warning的考虑
  if ((void *)(eth + 1) > data_end) //
    return false;

  /*
  括号里运算式 eth+1 是个非常有趣的表达式，它的本质是指针运算，指针变量 + 1 就是指针向右移动 n 个字节，这个 n 为该指针变量指向的对象类型的字节长度，这里就是 struct ethhdr 的字节长度，为 14 个字节，可以在这个内核头文件里找到相关定义：

  struct ethhdr {
  // ETH_ALEN 为 6 个字节
  unsigned char	h_dest[ETH_ALEN]; // destination eth addr
  unsigned char	h_source[ETH_ALEN]; // source ether addr
  // __be16 为 16 bit，也就是 2 个字节
  __be16   h_proto; // packet type ID field
}
// 所以整个 struct 就是 14 个字节长度
  */

  // Check if Ethernet frame has IP packet
  if (eth->h_proto == bpf_htons(ETH_P_IP))
  {
    struct iphdr *iph = (struct iphdr *)(eth + 1); // or (struct iphdr *)( ((void*)eth) + ETH_HLEN );
    if ((void *)(iph + 1) > data_end)
      return false;

    // Check if IP packet contains a TCP segment
    if (iph->protocol == IPPROTO_TCP)
      return true;
  }

  return false;
}

SEC("xdp")
int xdp_drop_tcp(struct xdp_md *ctx)
{

  void *data_end = (void *)(long)ctx->data_end;
  void *data = (void *)(long)ctx->data;

  if (is_TCP(data, data_end))
    return XDP_DROP;

  return XDP_PASS;
}

//tc hook
SEC("tc")
int tc_drop_tcp(struct __sk_buff *skb)
{

  void *data = (void *)(long)skb->data;
  void *data_end = (void *)(long)skb->data_end;

  if (is_TCP(data, data_end))
    return TC_ACT_SHOT;

  return TC_ACT_OK;
}

char _license[] SEC("license") = "GPL";
```

上面代码有几处细节，这里列举下（仅分析 xdp 的部分），在 `is_TCP` 函数中，`eth` 是 `struct ethhdr` 指针类型，`eth+1` 是指针运算，指针变量 `+1` 就是指针向右移动 `n` 个字节（`n`为该指针变量指向的对象类型的字节长度，即`struct ethhdr`的字节长度，为`14`个字节），是必须的数据包边界检查，该表达式有两层意思：

- 如果`data_end-data_begin`的结果小于`sizeof(struct ethhdr)`，那么说明这不是一个以太网的数据包，可以直接转发
- 如果`data_end-data_begin`的结果大于等于`sizeof(struct ethhdr)`，那么说明程序可以安全的操作这`sizeof(struct ethhdr)`个字节的数据（试想下ebpf verifier的安全要求）

因此，整体的`if`逻辑是判断括号内运算结果会不会内存越界，这对于BPF verifier是必要的，如果没有，BPF verifier会阻止这个程序加载到内核中，原因是后面的代码`struct iphdr *iph = (struct iphdr *)(eth + 1)`，即通过右移`data`变量获取到IP头，因此需要判断这个右移结果是否有效，如果无效，就直接`return`出去了，防止内存越界（**类似的右移判断逻辑在BPF程序里出现时，一定要做好边界判断逻辑，属于BPF开发基本操作**）

```C
static bool is_TCP(void *data_begin, void *data_end){
  //...
  if ((void *)(eth + 1) > data_end){
      return false;
  }
  //...
}

// ethhdr total size = 14 bytes
struct ethhdr {
  // ETH_ALEN 为 6 个字节
  unsigned char        h_dest[ETH_ALEN]; /* destination eth addr */
  unsigned char        h_source[ETH_ALEN]; /* source ether addr */
  // __be16 为 16 bit，也就是 2 个字节
  __be16   h_proto; /* packet type ID field */
}
```

如果不使用指针运算，还是作显式的长度判断，那么代码如下：

```c
static bool is_TCP(void *data_begin, void *data_end){
  //...
  u64 h_offset;
  // 显式声明并赋值ethhdr长度
  h_offset = sizeof(*eth);
  // 根据左右变量类型，运算符号加号重载成相关运算机制
  if (data + h_offset > data_end)
    return false;
  //...
}
```

##  0x03 TC 基础


##  0x04  TC VS XDP
小结下，tc（Traffic Control）和 xdp（eXpress Data Path）是 Linux 网络中两种不同的数据包处理机制，他们的区别如下：

-   位置不同: tc 位于 Linux 网络协议栈的较高层，主要用于在网络设备的出入口处对数据包进行分类、调度和限速等操作。而 xdp 位于网络设备驱动程序的接收路径上，用于快速处理数据包并决定是否将其传递给协议栈
-   执行时机不同: tc 在数据包进入或离开网络设备时执行，通常在内核空间中进行。而 xdp 在数据包进入网络设备驱动程序的接收路径时执行，可以在内核空间中或用户空间中执行
-   处理能力不同: tc 提供了更复杂的流量控制和分类策略，可以实现各种 QoS（Quality of Service）功能。它可以对数据包进行过滤、限速、排队等操作。而 xdp 主要用于快速的数据包过滤和处理，以降低延迟和提高性能
-   XDP 程序对应的类型是 `BPF_PROG_TYPE_XDP` ，它在网络驱动程序刚刚收到数据包时触发执行，由于无需通过复杂的内核网络协议栈，所以 XDP 程序可以用来实现高性能的网络处理方案，常用于 DDos 防御、防护墙、4 层负载均衡等
-   TC 程序 对应的类型是 `BPF_PROG_TYPE_SCHED_CLS` 和 `BPF_PROG_TYPE_SCHED_ACT` ，分别用于流量控制的分类器和执行器。Linux 流量控制通过网卡队列、排队规则、分类器、过滤器以及执行器实现了网络流量的整形调度和带宽控制

##  0x05    参考
- [每秒 1 百万的包传输，几乎不耗 CPU 的那种](https://colobu.com/2023/04/02/support-1m-pps-with-zero-cpu-usage/)
- [eBPF 技术实践：高性能 ACL](https://blog.csdn.net/ByteDanceTech/article/details/106632252)
- [eBPF Talk: 解密 XDP generic 模式](https://mp.weixin.qq.com/s?__biz=MjM5MTQxNTk5MA==&mid=2247483972&idx=1&sn=87bce22ffb54edda9d6c2556cfc8c36c&chksm=a6b4a99d91c3208bb19a893f716c090dd637a85c528b0ea4a9ef405709fc53c128b0af561823&scene=21#wechat_redirect)
- [你的第一个 XDP BPF 程序](https://davidlovezoe.club/wordpress/archives/937)
- [Deep Dive into Facebook's BPF edge firewall](https://cilium.io/blog/2018/11/20/fb-bpf-firewall/)
- [你的第一个 TC BPF 程序](https://cloud.tencent.com/developer/article/1626377)
- [eBPF 中常见的事件类型](https://blog.spoock.com/2023/08/19/eBPF-Hook/)
- [A toy tool that leverages the super powers of XDP to bring in-kernel IP filtering](https://github.com/sematext/oxdpus)
- [Cilium 原理解析：网络数据包在内核中的流转过程](https://developer.volcengine.com/articles/7088359390654234660)
- [eBPF 在 Golang 中的应用介绍](https://www.cnxct.com/an-applied-introduction-to-ebpf-with-go/)
- [eBPF Talk: XDP 解析所有 TCP options](https://asphaltt.github.io/post/ebpf-talk-127-xdp-tcp-options/)