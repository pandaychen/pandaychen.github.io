---
layout:     post
title:      EBPF 内核态代码学习（三）：一个 XDP ACL系统实现
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

##  0x01    内核态代码
内核态代码的[核心逻辑](https://github.com/sematext/oxdpus/blob/master/pkg/xdp/prog/xdp.c)如下，比较简洁：

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

    if (data + offset > end) {
        return XDP_ABORTED;
    }
    eth_type = eth->h_proto;

    /* handle VLAN tagged packet */
    if (eth_type == htons(ETH_P_8021Q) || eth_type == htons(ETH_P_8021AD)) {
	struct vlan_hdr *vlan_hdr;

	vlan_hdr = (void *)eth + offset;
	offset += sizeof(*vlan_hdr);
	if ((void *)eth + offset > end)
		return false;
	eth_type = vlan_hdr->h_vlan_encapsulated_proto; 
   }

    /* let's only handle IPv4 addresses */
    if (eth_type == ntohs(ETH_P_IPV6)) {
        return XDP_PASS;
    }

    struct iphdr *iph = data + offset;
    offset += sizeof(struct iphdr);
    /* make sure the bytes you want to read are within the packet's range before reading them */
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


##  0x02    用户态代码

##  0x0  参考
-   [eBPF技术实践：高性能ACL](https://mp.weixin.qq.com/s/25mhUrNhF3HW8H6-ES7waA)
-   [A high performance ACL basied on XDP](https://github.com/hi-glenn/xdp_acl)
-   [A toy tool that leverages the super powers of XDP to bring in-kernel IP filtering](https://github.com/sematext/oxdpus/tree/master)