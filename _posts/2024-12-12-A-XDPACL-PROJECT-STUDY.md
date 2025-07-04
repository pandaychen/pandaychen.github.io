---
layout:     post
title:      EBPF 内核态代码学习（三）：基于 XDP 技术的ACL/Firewall系统实现
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

```CPP
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

```CPP
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

```CPP
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

```CPP
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

##  0x04  参考
-   [eBPF技术实践：高性能ACL](https://mp.weixin.qq.com/s/25mhUrNhF3HW8H6-ES7waA)
-   [A high performance ACL basied on XDP](https://github.com/hi-glenn/xdp_acl)
-   [A toy tool that leverages the super powers of XDP to bring in-kernel IP filtering](https://github.com/sematext/oxdpus/tree/master)
-   [A firewall that utilizes the Linux kernel's XDP hook.](https://github.com/gamemann/XDP-Firewall)