---
layout:     post
title:      EBPF 内核态代码学习（三）：基于 XDP 技术的ACL/Firewall实现
subtitle:   基于ebpf技术实现的XDP应用分析
date:       2024-12-12
author:     pandaychen
header-img:
catalog: true
tags:
    - Golang
    - eBPF
---

##  0x00    前言
本文分析下基于XDP技术的防火墙相关实现细节，主要涉及如下项目：

-   oxdpus
-   xdp-firewall
-   TyrShield

##  0x01   oxdpus项目
oxdpus是一个基于XDP技术实现的[包过滤项目](https://github.com/sematext/oxdpus/tree/master)，支持下面指令：

```TEXT
  add         Appends a new IP address to the blacklist
  attach      Attaches the XDP program on the specified device
  detach      Removes the XDP program from the specified device
  help        Help about any command
  list        Shows all IP addresses registered in the blacklist
  remove      Removes an IP address from the blacklist
```

####    使用

```BASH
[root@VM-X-X-tencentos oxdpus]# ./oxdpus attach --dev=eth1
INFO XDP program successfully attached to eth1 device 
[root@VM-X-X-tencentos oxdpus]# ip link show dev eth1
2: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:6c:83:18 brd ff:ff:ff:ff:ff:ff
    prog/xdp id 26603 name  tag 4b119eeb47d44763 jited      #已加载xdp程序到网卡接口
    altname enp0s5
    altname ens5
[root@VM-X-X-tencentos oxdpus]#  ./oxdpus add --ip=1.1.1.1
INFO 1.1.1.1 address added to the blacklist   
[root@VM-X-X-tencentos oxdpus]# ./oxdpus list
* 1.1.1.1
```

####   内核态代码
内核态代码的[核心逻辑](https://github.com/sematext/oxdpus/blob/master/pkg/xdp/prog/xdp.c)如下，比较简洁。需要特别注意的是代码中检测packet边界的几个条件，这些限制条件保证了 XDP 代码在内核中的的正常运行，避免有无效指针或者违反安全策略的代码被加载到内核

1、数据结构

`blacklist`用来存放IP地址黑名单，注意这里的`pinning`与`namespace`成员

```cpp
#define PIN_GLOBAL_NS 2

struct bpf_map_def SEC("maps/blacklist") blacklist = {
    .type = BPF_MAP_TYPE_HASH,
    .key_size = sizeof(u32),
    .value_size = sizeof(u32),
    .max_entries = 4194304,
    .pinning = PIN_GLOBAL_NS,
    .namespace = "globals",  
};
```

2、核心逻辑

```cpp
SEC("xdp/xdp_ip_filter")
int xdp_ip_filter(struct xdp_md *ctx) {
    void *end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    u32 ip_src;
    u64 offset;
    u16 eth_type;

    struct ethhdr *eth = data;
    offset = sizeof(*eth);

    //边界检测1
    if (data + offset > end) {
        return XDP_ABORTED;
    }
    eth_type = eth->h_proto;

    /* handle VLAN tagged packet */
    // 处理VLAN标记的数据包
    if (eth_type == htons(ETH_P_8021Q) || eth_type == htons(ETH_P_8021AD)) {
        // 剥离VLAN头
        struct vlan_hdr *vlan_hdr;

        vlan_hdr = (void *)eth + offset;
        offset += sizeof(*vlan_hdr);
        //边界检测2
        if ((void *)eth + offset > end)
            return false;
        eth_type = vlan_hdr->h_vlan_encapsulated_proto; 
   }

    /* let's only handle IPv4 addresses */
    // 这里仅处理IPV4的数据包
    if (eth_type == ntohs(ETH_P_IPV6)) {
        return XDP_PASS;
    }

    struct iphdr *iph = data + offset;
    offset += sizeof(struct iphdr);
    /* make sure the bytes you want to read are within the packet's range before reading them */
    // 在读取之前，需要确保读取的字节在数据包的长度范围内

    //边界检测3
    if (iph + 1 > end) {
        return XDP_ABORTED;
    }
    ip_src = iph->saddr;

    if (bpf_map_lookup_elem(&blacklist, &ip_src)) {
        return XDP_DROP;
    }

    return XDP_PASS;
}
```

####    用户态代码
主要是了解下如何将elf字节码载入、卸载到指定网卡的方法（命令），核心实现参考[`hook.go`](https://github.com/sematext/oxdpus/blob/master/pkg/xdp/hook.go)

-   `Attach`方法：将XDP字节码加载到指定网卡
-   `Remove`方法：卸载XDP字节码

##  0x02    TyrShield
[TyrShield](https://github.com/boylegu/TyrShield)是一个基于XDP实现的SSH（任意端口）的基于源IP的连接检测（封禁）系统，提供了`max_attempts`、`time_window_ns`等自定义配置

```cpp
#define SSH_PORT 22
#define MAX_ATTEMPTS 5
#define TIME_WINDOW_NS (60ULL * 1000000000ULL)  // 60 秒
#define BLOCK_TIME_NS (300ULL * 1000000000ULL)  // 300 秒

// 全局配置 Map
struct config {
    __u32 ssh_port;
    __u32 max_attempts;
    __u64 time_window_ns;
    __u64 block_time_ns;
};

//config_map 用于用户态代码配置更新，内核态仅读取
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1);
    __type(key, __u32);
    __type(value, struct config);
} config_map SEC(".maps");
```

核心代码如下，主要包含连接计数、封禁及封禁事件通知等功能，逻辑比较简洁

```cpp
struct attempt_info {
    __u32 count;        //当前周期计数
    __u64 first_time;   //当前周期开始
    __u64 last_time;    //最后访问时间
    __u64 block_until;  //封禁结束时间
};

//XDP 主逻辑入口
SEC("xdp_ssh_filter")
int xdp_prog(struct xdp_md *ctx) {
    // ......
    // 仅处理SYN包
    if (!(tcph->syn && !tcph->ack))
        return XDP_PASS;

    __u32 src_ip = iphdr->saddr;
    struct config cfg;
    bpf_map_lookup_elem(&config_map, &zero, &cfg);

    struct attempt_info *info = bpf_map_lookup_elem(&ssh_attempts, &src_ip);
    __u64 now = bpf_ktime_get_ns();

    if (!info) {
        // 首次尝试
        struct attempt_info init = {1, now, now, 0};
        bpf_map_update_elem(&ssh_attempts, &src_ip, &init, BPF_ANY);
    } else {
        if (now < info->block_until){
            // 还在封禁期内，直接丢弃
            return XDP_DROP;
        }

        // 滑动窗口过期则重置
        if (now - info->first_time > cfg.time_window_ns) {
            info->count = 1;
            info->first_time = now;
        } else {
            //还在滑动窗口内，访问次数累加
            info->count++;
        }
        //更新访问时间
        info->last_time = now;

        // 超过最大访问限制，封禁并通知用户态
        if (info->count > cfg.max_attempts) {
            info->block_until = now + cfg.block_time_ns;
            struct event ev = {src_ip, info->count};
            events.perf_submit(ctx, &ev, sizeof(ev));
            return XDP_DROP;
        }
    }
    return XDP_PASS;
}
```

##  0x03    xdp-firewall
[XDP-firewall](https://github.com/gamemann/XDP-Firewall)也是一款支持丰富配置的XDP防火墙，实现了一个功能完备的 高性能 eBPF XDP 防火墙，能够以极高的效率处理 IPv4/IPv6 流量，支持动态黑名单、速率限制（PPS/BPS）以及自定义规则过滤。核心功能如：

-  High-Performance Packet Filtering
-  Source IP Blocking & Dropping
-  Dynamic Filtering (Rule-Based)：支持基于源IP/flow层面的pps、bps限流
-  IP Range Dropping (CIDR)
-  Real-Time Packet Counters
-  Logging System
-  Pinned Maps & CLI Utilities：提供了`xdpfw-add`、`xdpfw-del`工具，无需重启程序即可实现动态规则管理

####    核心代码

-   `map_block`：IPv4 source address block list	Hash
-   `map_block6`：	IPv6 source address block list	Hash
-   `map_range_drop`：	IP range drop list	Array
-   `map_filters`：	Dynamic filter rules	Array
-   `map_stats`：	Packet statistics	Array
-   `map_filter_log`：	Filter logging events	Ring buffer
-   `map_ip_stats`：	IP-based rate limiting	LRU Hash
-   `map_flow_stats`：	Flow-based rate limiting	LRU Hash

1、XDP程序入口，已经集齐firewall实现的各种要素了，比较有借鉴价值：

```cpp
SEC("xdp_prog")
int xdp_prog_main(struct xdp_md *ctx)
{
    //ctx->data 和 ctx->data_end 获取数据包的起始和结束指针
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;

    // Retrieve stats map value.
    u32 key = 0;

    //查找全局统计信息
    stats_t* stats = bpf_map_lookup_elem(&map_stats, &key);

    // Scan ethernet header.
    struct ethhdr *eth = data;

    // Check if the ethernet header is valid.

    //严格的边界检查：eth + 1 > data_end
    if (unlikely(eth + 1 > (struct ethhdr *)data_end))
    {
        //统计状态
        inc_pkt_stats(stats, STATS_TYPE_DROPPED);
        return XDP_DROP;
    }

    // Check Ethernet protocol.
    if (unlikely(eth->h_proto != htons(ETH_P_IP) && eth->h_proto != htons(ETH_P_IPV6)))
    {
        inc_pkt_stats(stats, STATS_TYPE_PASSED);
        return XDP_PASS;
    }

    // Initialize IP headers.
    struct iphdr *iph = NULL;
    struct ipv6hdr *iph6 = NULL;
    u128 src_ip6 = 0;

    // Set IPv4 and IPv6 common variables.
    if (eth->h_proto == htons(ETH_P_IP))
    {
        iph = data + sizeof(struct ethhdr);
        if (unlikely(iph + 1 > (struct iphdr *)data_end))
        {
            inc_pkt_stats(stats, STATS_TYPE_DROPPED);
            return XDP_DROP;
        }
    }
    else
    {
        iph6 = data + sizeof(struct ethhdr);
        if (unlikely(iph6 + 1 > (struct ipv6hdr *)data_end))
        {
            inc_pkt_stats(stats, STATS_TYPE_DROPPED);
            return XDP_DROP;
        }

        //由于 IPv6 地址是 128 位的，memcpy 将其拷贝到 u128 src_ip6 中，以便后续作为 Map 的 Key 使用
        memcpy(&src_ip6, iph6->saddr.in6_u.u6_addr32, sizeof(src_ip6));
    }
    
    // We only want to process TCP, UDP, and ICMP protocols.
    //传输层协议过滤，只处理 TCP、UDP 和 ICMP 协议的数据包
    // 其他三层协议（如 OSPF, GRE），放行
    if ((iph && iph->protocol != IPPROTO_UDP && iph->protocol != IPPROTO_TCP && iph->protocol != IPPROTO_ICMP) || (iph6 && iph6->nexthdr != IPPROTO_UDP && iph6->nexthdr != IPPROTO_TCP && iph6->nexthdr != IPPROTO_ICMP))
    {
        inc_pkt_stats(stats, STATS_TYPE_PASSED);
        return XDP_PASS;
    }

    // Retrieve nanoseconds since system boot as timestamp.
    u64 now = bpf_ktime_get_ns();

    //规则1：动态黑名单检查
    // 源IP 封禁检测：检查源 IP 是否在block中，如果被阻止且未过期，直接丢弃数据包
    u64 *blocked = NULL;

    if (iph)
    {
        blocked = bpf_map_lookup_elem(&map_block, &iph->saddr);
    }
    else
    {
        blocked = bpf_map_lookup_elem(&map_block6, &src_ip6);
    }
    
    if (blocked != NULL)
    {   
        //如果 IP 在黑名单中，检查当前时间 now 是否超过了存储的过期时间 *blocked
        if (*blocked > 0 && now > *blocked)
        {
            // Remove element from map.
            //已过期：从 Map 中删除该条目，允许数据包继续后续逻辑
            if (iph)
            {
                bpf_map_delete_elem(&map_block, &iph->saddr);
            }
            else
            {
                bpf_map_delete_elem(&map_block6, &src_ip6);
            }
        }
        else
        {
            // Increase blocked stats entry.
            //未过期：直接 XDP_DROP
            inc_pkt_stats(stats, STATS_TYPE_DROPPED);
            // They're still blocked. Drop the packet.
            return XDP_DROP;
        }
    }

    //cidr drop rule

    //规则2：CIDR 范围过滤
    if (iph && check_ip_range_drop(iph->saddr))
    {
        inc_pkt_stats(stats, STATS_TYPE_DROPPED);
        return XDP_DROP;
    }

    //规则3：传输层解析

    // Retrieve total packet length.
    u16 pkt_len = data_end - data;

    // Parse layer-4 headers and determine source port and protocol.
    struct tcphdr *tcph = NULL;
    struct udphdr *udph = NULL;
    struct icmphdr *icmph = NULL;

    struct icmp6hdr *icmp6h = NULL;

    u16 src_port = 0;
    u16 dst_port = 0;
    u8 protocol = 0;

    if (iph)
    {
        protocol = iph->protocol;
        switch (iph->protocol)
        {
            case IPPROTO_TCP:
                // Scan TCP header
                // 在读取端口信息之前，先校验包长
                // 此处计算偏移量时使用了 iph->ihl * 4（可以正确处理带有 IP Options 的 IPv4 报文）
                tcph = data + sizeof(struct ethhdr) + (iph->ihl * 4);
                // Check TCP header.
                if (unlikely(tcph + 1 > (struct tcphdr *)data_end))
                {
                    inc_pkt_stats(stats, STATS_TYPE_DROPPED);
                    return XDP_DROP;
                }
                src_port = tcph->source;
                dst_port = tcph->dest;
                break;
            case IPPROTO_UDP:
                // Scan UDP header.
                udph = data + sizeof(struct ethhdr) + (iph->ihl * 4);
                // Check UDP header.
                if (unlikely(udph + 1 > (struct udphdr *)data_end))
                {
                    inc_pkt_stats(stats, STATS_TYPE_DROPPED);
                    return XDP_DROP;
                }

                src_port = udph->source;
                dst_port = udph->dest;
                break;
            case IPPROTO_ICMP:
                // Scan ICMP header.
                icmph = data + sizeof(struct ethhdr) + (iph->ihl * 4);

                // Check ICMP header.
                if (unlikely(icmph + 1 > (struct icmphdr *)data_end))
                {
                    inc_pkt_stats(stats, STATS_TYPE_DROPPED);
                    return XDP_DROP;
                }
                break;
        }
    }
    else if (iph6)
    {
        protocol = iph6->nexthdr;
        switch (iph6->nexthdr)
        {
            case IPPROTO_TCP:
                // Scan TCP header.
                tcph = data + sizeof(struct ethhdr) + sizeof(struct ipv6hdr);
                // Check TCP header.
                if (unlikely(tcph + 1 > (struct tcphdr *)data_end))
                {
                    inc_pkt_stats(stats, STATS_TYPE_DROPPED);
                    return XDP_DROP;
                }
                src_port = tcph->source;
                dst_port = tcph->dest;
                break;
            case IPPROTO_UDP:
                // Scan UDP header.
                udph = data + sizeof(struct ethhdr) + sizeof(struct ipv6hdr);
                // Check TCP header.
                if (unlikely(udph + 1 > (struct udphdr *)data_end))
                {
                    inc_pkt_stats(stats, STATS_TYPE_DROPPED);
                    return XDP_DROP;
                }
                src_port = udph->source;
                dst_port = udph->dest;
                break;
            case IPPROTO_ICMPV6:
                // Scan ICMPv6 header.
                icmp6h = data + sizeof(struct ethhdr) + sizeof(struct ipv6hdr);
                // Check ICMPv6 header.
                if (unlikely(icmp6h + 1 > (struct icmp6hdr *)data_end))
                {
                    inc_pkt_stats(stats, STATS_TYPE_DROPPED);
                    return XDP_DROP;
                }
                break;
        }
    }

    // Update client stats (PPS/BPS)
    // 规则4：流量统计与速率计算
    u64 ip_pps = 0;
    u64 ip_bps = 0;
    u64 flow_pps = 0;
    u64 flow_bps = 0;

    //当限速功能开启时，进行速率限制统计更新
    if (iph)
    {
        //1、更新（源） IP 级别的 PPS/BPS 统计
        update_ip_stats(&ip_pps, &ip_bps, iph->saddr, pkt_len, now);
        //2、更新流（五元组）级别的 PPS/BPS 统计
        update_flow_stats(&flow_pps, &flow_bps, iph->saddr, src_port, protocol, pkt_len, now);
    }
    else if (iph6)
    {
        update_ip6_stats(&ip_pps, &ip_bps, &src_ip6, pkt_len, now);
        update_flow6_stats(&flow_pps, &flow_bps, &src_ip6, src_port, protocol, pkt_len, now);
    }
    // Create rule context.
    rule_ctx_t rule = {0};
    rule.flow_pps = flow_pps;
    rule.flow_bps = flow_bps;
    rule.ip_pps = ip_pps;
    rule.ip_bps = ip_bps;
    rule.pkt_len = pkt_len;

    rule.now = now;
    rule.protocol = protocol;
    rule.src_port = src_port;
    rule.dst_port = dst_port;
    
    rule.iph = iph;
    
    rule.tcph = tcph;
    rule.udph = udph;
    rule.icmph = icmph;

    rule.iph6 = iph6;
    rule.icmph6 = icmp6h;
    //如果开启了过滤，过滤规则处理

    //规则5：自定义过滤规则引擎（有限制）
#pragma unroll 30
    for (int i = 0; i < MAX_FILTERS; i++)
    {
        if (process_rule(i, &rule))
        {
            break;
        }
    }
    if (rule.matched)
    {
        goto matched;
    }
    inc_pkt_stats(stats, STATS_TYPE_PASSED);
    return XDP_PASS;


    //匹配结果处理
matched:
    if (rule.action == 0)
    {
        // Before dropping, update the block map.
        if (rule.block_time > 0)
        {
            //丢弃动作：如果设置了阻止时间，将源 IP 添加到block
            //动态封禁：如果规则配置了 block_time，会直接写入黑名单 map_block，更早生效
            u64 new_time = now + (rule.block_time * NANO_TO_SEC);
            if (iph)
            {
                bpf_map_update_elem(&map_block, &iph->saddr, &new_time, BPF_ANY);
            }
            else
            {
                bpf_map_update_elem(&map_block6, &src_ip6, &new_time, BPF_ANY);
            }     
        }
        inc_pkt_stats(stats, STATS_TYPE_DROPPED);
        return XDP_DROP;
    }
    else
    {
        inc_pkt_stats(stats, STATS_TYPE_ALLOWED);
    }

    return XDP_PASS;
}
```

![xdp-firewall](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/xdp/firewall/flow_arch.png)

2、规则过滤方法[`process_rule`](https://github.com/gamemann/XDP-Firewall/blob/master/src/xdp/utils/rule.c#L12)，主要包含如下规则：

-   速率限制检查，基于源 IP 的 PPS（每秒包数）和 BPS（每秒字节数）
-   包长度过滤：最大包长度检查 以及最小包长度检查
-   IP 层过滤规则，以IPV4为例：
    -   源 IP 地址匹配（支持 CIDR）
    -   目标 IP 地址匹配（支持 CIDR）
    -   TOS（服务类型）检查
    -   TTL 范围检查（最大和最小 TTL）
-   传输层协议过滤，支持TCP/UDP/ICMP，主要检测项如下
    -   TCP端口范围检查：源端口和目标端口的最小/最大值
    -   TCP 标志位检查：URG、ACK、RST、PSH、SYN、FIN、ECE、CWR 标志
    -   UDP端口范围检查：源端口和目标端口的最小/最大值
    -   ICMP 代码检查
    -   ICMP 类型检查

```cpp
static __always_inline long process_rule(u32 idx, void* data)
{
    rule_ctx_t* ctx = data;

    filter_t *filter = bpf_map_lookup_elem(&map_filters, &idx);

    if (!filter || !filter->set)
    {
        return 1;
    }

#ifdef ENABLE_RL_IP
    // Check source IP rate limits.
    if (filter->do_ip_pps && ctx->ip_pps < filter->ip_pps)
    {
        return 0;
    }

    if (filter->do_ip_bps && ctx->ip_bps < filter->ip_bps)
    {
        return 0;
    }
#endif

#ifdef ENABLE_RL_FLOW
    // Check source flow rate limits.
    if (filter->do_flow_pps && ctx->flow_pps < filter->flow_pps)
    {
        return 0;
    }

    if (filter->do_flow_bps && ctx->flow_bps < filter->flow_bps)
    {
        return 0;
    }
#endif

    // Max packet length.
    if (filter->ip.do_max_len && filter->ip.max_len < ctx->pkt_len)
    {
        return 0;
    }

    // Min packet length.
    if (filter->ip.do_min_len && filter->ip.min_len > ctx->pkt_len)
    {
        return 0;
    }

    // Match IP settings.
    if (ctx->iph)
    {
        // Source address.
        if (filter->ip.src_ip)
        {
            if (filter->ip.src_cidr == 32 && ctx->iph->saddr != filter->ip.src_ip)
            {
                return 0;
            }

            if (!is_ip_in_range(ctx->iph->saddr, filter->ip.src_ip, filter->ip.src_cidr))
            {
                return 0;
            }
        }

        // Destination address.
        if (filter->ip.dst_ip)
        {
            if (filter->ip.dst_cidr == 32 && ctx->iph->daddr != filter->ip.dst_ip)
            {
                return 0;
            }
            
            if (!is_ip_in_range(ctx->iph->daddr, filter->ip.dst_ip, filter->ip.dst_cidr))
            {
                return 0;
            }
        }

#if defined(ENABLE_IPV6) && defined(ALLOW_SINGLE_IP_V4_V6)
        if ((filter->ip.src_ip6[0] != 0 || filter->ip.src_ip6[1] != 0 || filter->ip.src_ip6[2] != 0 || filter->ip.src_ip6[3] != 0) || (filter->ip.dst_ip6[0] != 0 || filter->ip.dst_ip6[1] != 0 || filter->ip.dst_ip6[2] != 0 || filter->ip.dst_ip6[3] != 0))
        {
            return 0;
        }
#endif

        // TOS.
        if (filter->ip.do_tos && filter->ip.tos != ctx->iph->tos)
        {
            return 0;
        }

        // Max TTL.
        if (filter->ip.do_max_ttl && filter->ip.max_ttl < ctx->iph->ttl)
        {
            return 0;
        }

        // Min TTL.
        if (filter->ip.do_min_ttl && filter->ip.min_ttl > ctx->iph->ttl)
        {
            return 0;
        }
    }
#ifdef ENABLE_IPV6
    else if (ctx->iph6)
    {
        // Source address.
        if (filter->ip.src_ip6[0] != 0 && (ctx->iph6->saddr.in6_u.u6_addr32[0] != filter->ip.src_ip6[0] || ctx->iph6->saddr.in6_u.u6_addr32[1] != filter->ip.src_ip6[1] || ctx->iph6->saddr.in6_u.u6_addr32[2] != filter->ip.src_ip6[2] || ctx->iph6->saddr.in6_u.u6_addr32[3] != filter->ip.src_ip6[3]))
        {
            return 0;
        }

        // Destination address.
        if (filter->ip.dst_ip6[0] != 0 && (ctx->iph6->daddr.in6_u.u6_addr32[0] != filter->ip.dst_ip6[0] || ctx->iph6->daddr.in6_u.u6_addr32[1] != filter->ip.dst_ip6[1] || ctx->iph6->daddr.in6_u.u6_addr32[2] != filter->ip.dst_ip6[2] || ctx->iph6->daddr.in6_u.u6_addr32[3] != filter->ip.dst_ip6[3]))
        {
            return 0;
        }

#ifdef ALLOW_SINGLE_IP_V4_V6
        if (filter->ip.src_ip != 0 || filter->ip.dst_ip != 0)
        {
            return 0;
        }
#endif

        // Max TTL length.
        if (filter->ip.do_max_ttl && filter->ip.max_ttl < ctx->iph6->hop_limit)
        {
            return 0;
        }

        // Min TTL length.
        if (filter->ip.do_min_ttl && filter->ip.min_ttl > ctx->iph6->hop_limit)
        {
            return 0;
        }
    }
#endif

    // Check TCP matches.
    if (filter->tcp.enabled)
    {
        if (!ctx->tcph)
        {
            return 0;
        }

        // Source port checks.
        if (filter->tcp.do_sport_min && ntohs(ctx->tcph->source) < filter->tcp.sport_min)
        {
            return 0;
        }

        if (filter->tcp.do_sport_max && ntohs(ctx->tcph->source) > filter->tcp.sport_max)
        {
            return 0;
        }

        // Destination port checks.
        if (filter->tcp.do_dport_min && ntohs(ctx->tcph->dest) < filter->tcp.dport_min)
        {
            return 0;
        }

        if (filter->tcp.do_dport_max && ntohs(ctx->tcph->dest) > filter->tcp.dport_max)
        {
            return 0;
        }

        // URG flag.
        if (filter->tcp.do_urg && filter->tcp.urg != ctx->tcph->urg)
        {
            return 0;
        }

        // ACK flag.
        if (filter->tcp.do_ack && filter->tcp.ack != ctx->tcph->ack)
        {
            return 0;
        }

        // RST flag.
        if (filter->tcp.do_rst && filter->tcp.rst != ctx->tcph->rst)
        {
            return 0;
        }

        // PSH flag.
        if (filter->tcp.do_psh && filter->tcp.psh != ctx->tcph->psh)
        {
            return 0;
        }

        // SYN flag.
        if (filter->tcp.do_syn && filter->tcp.syn != ctx->tcph->syn)
        {
            return 0;
        }

        // FIN flag.
        if (filter->tcp.do_fin && filter->tcp.fin != ctx->tcph->fin)
        {
            return 0;
        }

        // ECE flag.
        if (filter->tcp.do_ece && filter->tcp.ece != ctx->tcph->ece)
        {
            return 0;
        }

        // CWR flag.
        if (filter->tcp.do_cwr && filter->tcp.cwr != ctx->tcph->cwr)
        {
            return 0;
        }
    }
    // Check UDP matches.
    else if (filter->udp.enabled)
    {
        if (!ctx->udph)
        {
            return 0;
        }

        // Source port checks.
        if (filter->udp.do_sport_min && ntohs(ctx->udph->source) < filter->udp.sport_min)
        {
            return 0;
        }

        if (filter->udp.do_sport_max && ntohs(ctx->udph->source) > filter->udp.sport_max)
        {
            return 0;
        }

        // Destination port checks.
        if (filter->udp.do_dport_min && ntohs(ctx->udph->dest) < filter->udp.dport_min)
        {
            return 0;
        }

        if (filter->udp.do_dport_max && ntohs(ctx->udph->dest) > filter->udp.dport_max)
        {
            return 0;
        }
    }
    else if (filter->icmp.enabled)
    {
        if (ctx->icmph)
        {
            // Code.
            if (filter->icmp.do_code && filter->icmp.code != ctx->icmph->code)
            {
                return 0;
            }

            // Type.
            if (filter->icmp.do_type && filter->icmp.type != ctx->icmph->type)
            {
                return 0;
            }  
        }
#ifdef ENABLE_IPV6
        else if (ctx->icmph6)
        {
            // Code.
            if (filter->icmp.do_code && filter->icmp.code != ctx->icmph6->icmp6_code)
            {
                return 0;
            }

            // Type.
            if (filter->icmp.do_type && filter->icmp.type != ctx->icmph6->icmp6_type)
            {
                return 0;
            }
        }
#endif
        else
        {
            return 0;
        }
    }

#ifdef ENABLE_FILTER_LOGGING
    if (filter->log > 0)
    {
        log_filter_msg(ctx->iph, ctx->iph6, ctx->src_port, ctx->dst_port, ctx->protocol, ctx->now, ctx->ip_pps, ctx->ip_bps, ctx->flow_pps, ctx->flow_bps, ctx->pkt_len, idx);
    }
#endif
    
    // Matched.
    ctx->matched = 1;
    ctx->action = filter->action;
    ctx->block_time = filter->block_time;

    return 1;
}
```

3、`check_ip_range_drop`：基于CIDR的匹配，这里使用了`BPF_MAP_TYPE_LPM_TRIE`数据结构来实现，原理基于LPM (Longest Prefix Match) 类型的 Map，用于高效匹配一段 IP 地址区间

```cpp
struct
{
    __uint(type, BPF_MAP_TYPE_LPM_TRIE);
    __uint(max_entries, MAX_IP_RANGES);
    __uint(map_flags, BPF_F_NO_PREALLOC);
    __type(key, lpm_trie_key_t);
    __type(value, u64);
} map_range_drop SEC(".maps");


static __always_inline int check_ip_range_drop(u32 ip)
{
    lpm_trie_key_t key = {0};
    key.prefix_len = 32;
    key.data = ip;

    u64 *lookup = bpf_map_lookup_elem(&map_range_drop, &key);

    if (lookup)
    {
        u32 bit_mask = *lookup >> 32;
        u32 prefix = *lookup & 0xFFFFFFFF;

        // Check if matched.
        if ((ip & bit_mask) == prefix)
        {
            return 1;
        }
    }

    return 0;
}
```

####    细节
1、对性能的影响，注意事项：

-   由于循环处理，规则数量规模过多时，动态过滤可能会影响性能
-   在spoofed伪造源IP ddos场景下，开启源IP限流可能会导致 CPU 使用率过高。注意到这里实现pps限流的数据结构，依赖于`BPF_MAP_TYPE_LRU_HASH`结构`map_ip_stats`，在伪造源IP ddos情况下，来自海量不同 IP/port记录（key）会导致`BPF_MAP_TYPE_LRU_HASH`类型的map快速回收条目，从而导致CPU使用率升高
-   日志记录需要进行速率压制

2、改进：上诉逻辑在处理带有 VLAN 标签（`802.1Q`）的报文时会解析失败

```cpp
struct ethhdr *eth = data;
u16 eth_proto = eth->h_proto;
u32 offset = sizeof(struct ethhdr);

// 如果是 VLAN 报文 (0x8100)
if (eth_proto == htons(ETH_P_8021Q)) {
    struct vlan_hdr *vhdr = data + offset;
    if (vhdr + 1 > data_end) return XDP_DROP;
    eth_proto = vhdr->h_vlan_encapsulated_proto;
    offset += sizeof(struct vlan_hdr);
}

// 之后使用 offset 来定位 IP 头
struct iphdr *iph = data + offset;
```

##  0x04  参考
-   [eBPF技术实践：高性能ACL](https://mp.weixin.qq.com/s/25mhUrNhF3HW8H6-ES7waA)
-   [A high performance ACL basied on XDP](https://github.com/hi-glenn/xdp_acl)
-   [A toy tool that leverages the super powers of XDP to bring in-kernel IP filtering](https://github.com/sematext/oxdpus/tree/master)
-   [A firewall that utilizes the Linux kernel's XDP hook.](https://github.com/gamemann/XDP-Firewall)