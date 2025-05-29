---
layout:     post
title:      golang eBPF 开发入门（六）
subtitle:   ebpf的一些细节补充
date:       2024-08-30
author:     pandaychen
catalog:    true
tags:
    - eBPF
---

##  0x00  前言

##  0x01  Tail Call

####  尾调用 VS 普通函数调用

![]()

####  例1：XDP
先列举一个简单的[例子](https://github.com/spoock1024/ebpf-example/blob/main/bpf_tail_call_case2/ebpf/main.c)：

```cpp
SEC("xdp/icmp")
int handle_icmp(struct xdp_md *ctx) {
    bpf_printk("new icmp packet captured (XDP)\n");
    return XDP_PASS;
}

SEC("xdp/tcp")
int handle_tcp(struct xdp_md *ctx) {
    bpf_printk("new tcp packet captured (XDP)\n");
    return XDP_PASS;
}

SEC("xdp/udp")
int handle_udp(struct xdp_md *ctx) {
    bpf_printk("new udp packet captured (XDP)\n");
    return XDP_PASS;
}

// 在内核态声明一个尾调用的MAP，并且初始化
struct {
	__uint(type, BPF_MAP_TYPE_PROG_ARRAY);
	__type(key, u32);
	__type(value, u32);
	__uint(max_entries, 3);
	__array(values, int());
} packet_processing_progs SEC(".maps") = {
	.values =
		{
			[0] = &handle_icmp,
			[1] = &handle_tcp,
			[2] = &handle_udp,
		},
};

SEC("xdp_classifier")
int packet_classifier(struct xdp_md *ctx)  {
//    bpf_printk("new packet_classifier (XDP)\n");
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    struct ethhdr *eth = data;
    struct iphdr *ip;

    // 检查是否有足够的数据空间
    if ((void *)(eth + 1) > data_end) {
        return XDP_ABORTED;
    }

    // satisfy with ebpf verifier
    if (eth->h_proto != 8) {
        return XDP_PASS;
    }

    ip = (struct iphdr *)(eth + 1);

    // 检查IP头部是否完整
    if ((void *)(ip + 1) > data_end) {
        return XDP_ABORTED;
    }
    bpf_printk("protocol: %d\n", ip->protocol);
    bpf_printk("icmp: %d,tcp:%d,udp:%d\n", IPPROTO_ICMP, IPPROTO_TCP, IPPROTO_UDP);
    switch (ip->protocol) {
        case IPPROTO_ICMP:
            bpf_printk("icmp\n");
            bpf_tail_call(ctx, &packet_processing_progs, 0);
            break;
        case IPPROTO_TCP:
            bpf_printk("tcp\n");
            bpf_tail_call(ctx, &packet_processing_progs, 1);
            break;
        case IPPROTO_UDP:
            bpf_printk("udp\n");
            bpf_tail_call(ctx, &packet_processing_progs, 2);
            break;
        default:
            bpf_printk("unknown protocol\n");
            break;
    }

    return XDP_PASS;
}
```

####    例2：tracee
[`raw_tracepoint/sys_enter`](https://github.com/aquasecurity/tracee/blob/main/pkg/ebpf/c/tracee.bpf.c)

```CPP
// trace/events/syscalls.h: TP_PROTO(struct pt_regs *regs, long id)
// initial entry for sys_enter syscall logic
SEC("raw_tracepoint/sys_enter")
int tracepoint__raw_syscalls__sys_enter(struct bpf_raw_tracepoint_args *ctx)
{
    struct task_struct *task = (struct task_struct *) bpf_get_current_task();
    int id = ctx->args[1];
    if (is_compat(task)) {
        // Translate 32bit syscalls to 64bit syscalls, so we can send to the correct handler
        u32 *id_64 = bpf_map_lookup_elem(&sys_32_to_64_map, &id);
        if (id_64 == 0)
            return 0;

        id = *id_64;
    }
    bpf_tail_call(ctx, &sys_enter_init_tail, id);
    return 0;
}
```

##  0x0    参考
- [bpf_tail_call特性介绍](https://blog.spoock.com/2024/01/11/bpf-tail-call-intro/)
- [在 ebpf/libbpf 程序中使用尾调用（tail calls）](https://mozillazg.com/2022/10/ebpf-libbpf-use-tail-calls.html)
- [22-tail-calls](https://github.com/mozillazg/hello-libbpfgo/blob/master/22-tail-calls/main.bpf.c)
- [eBPF Talk: eBPF 尾调用简介](https://asphaltt.github.io/post/ebpf-tailcall-intro/)
- [eBPF 入门实践教程二十一： 使用 XDP 进行可编程数据包处理](https://github.com/eunomia-bpf/bpf-developer-tutorial/blob/main/src/21-xdp/README.zh.md)