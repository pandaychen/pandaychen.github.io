---
layout:     post
title:      EBPF 内核态代码学习（三）：networking/tcp-ip stack tracing
subtitle:   tcpstate、tcprtt、tcpconnlat等工具实现分析
date:       2025-05-09
author:     pandaychen
header-img:
catalog: true
tags:
    - eBPF
---

##  0x00 前言
本文学习下基于ebpf技术的网络协议栈追踪

-   [tcpstates]()：用于记录 TCP 连接的状态变化
-   [tcprtt]()：则用于记录 TCP 的往返时间（RTT, Round-Trip Time）

```BASH
[root@VM-X-X-centos edriver]#  perf list 'tcp:*' 'sock:inet*'

List of pre-defined events (to be used in -e):

  tcp:tcp_destroy_sock                               [Tracepoint event]
  tcp:tcp_probe                                      [Tracepoint event]
  tcp:tcp_rcv_space_adjust                           [Tracepoint event]
  tcp:tcp_receive_reset                              [Tracepoint event]
  tcp:tcp_retransmit_skb                             [Tracepoint event]
  tcp:tcp_retransmit_synack                          [Tracepoint event]
  tcp:tcp_send_reset                                 [Tracepoint event]


Metric Groups:

  sock:inet_sock_set_state                           [Tracepoint event]
```

##  0x01    tcpstates实现分析
tcpstates 是一个用来追踪和打印 TCP 连接状态变化的工具，可显示 TCP 连接在每个状态中的停留时长（单位ms），如下：

```TEXT
SKADDR           C-PID C-COMM     LADDR           LPORT RADDR           RPORT OLDSTATE    -> NEWSTATE    MS
ffff9fd7e8192000 22384 curl       1.1.1.1  0     2.2.2.2    80    CLOSE       -> SYN_SENT    0.000
ffff9fd7e8192000 0     swapper/5  1.1.1.1  63446 2.2.2.2    80    SYN_SENT    -> ESTABLISHED 1.373
ffff9fd7e8192000 22384 curl       1.1.1.1  63446 2.2.2.2    80    ESTABLISHED -> FIN_WAIT1   176.042
ffff9fd7e8192000 0     swapper/5  1.1.1.1  63446 2.2.2.2    80    FIN_WAIT1   -> FIN_WAIT2   0.536
ffff9fd7e8192000 0     swapper/5  1.1.1.1  63446 2.2.2.2    80    FIN_WAIT2   -> CLOSE       0.006
```

####    内核态实现

```CPP
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, MAX_ENTRIES);
    __type(key, __u16);
    __type(value, __u16);
} sports SEC(".maps");

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, MAX_ENTRIES);
    __type(key, __u16);
    __type(value, __u16);
} dports SEC(".maps");

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, MAX_ENTRIES);
    __type(key, struct sock *);
    __type(value, __u64);
} timestamps SEC(".maps");

struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(__u32));
    __uint(value_size, sizeof(__u32));
} events SEC(".maps");

SEC("tracepoint/sock/inet_sock_set_state")
int handle_set_state(struct trace_event_raw_inet_sock_set_state *ctx)
{
    struct sock *sk = (struct sock *)ctx->skaddr;
    __u16 family = ctx->family;
    __u16 sport = ctx->sport;
    __u16 dport = ctx->dport;
    __u64 *tsp, delta_us, ts;
    struct event event = {};

    if (ctx->protocol != IPPROTO_TCP)
        return 0;

    if (target_family && target_family != family)
        return 0;

    if (filter_by_sport && !bpf_map_lookup_elem(&sports, &sport))
        return 0;

    if (filter_by_dport && !bpf_map_lookup_elem(&dports, &dport))
        return 0;

    tsp = bpf_map_lookup_elem(&timestamps, &sk);
    ts = bpf_ktime_get_ns();
    if (!tsp)
        delta_us = 0;
    else
        delta_us = (ts - *tsp) / 1000;

    event.skaddr = (__u64)sk;
    event.ts_us = ts / 1000;
    event.delta_us = delta_us;
    event.pid = bpf_get_current_pid_tgid() >> 32;
    event.oldstate = ctx->oldstate;
    event.newstate = ctx->newstate;
    event.family = family;
    event.sport = sport;
    event.dport = dport;
    bpf_get_current_comm(&event.task, sizeof(event.task));

    if (family == AF_INET) {
        bpf_probe_read_kernel(&event.saddr, sizeof(event.saddr), &sk->__sk_common.skc_rcv_saddr);
        bpf_probe_read_kernel(&event.daddr, sizeof(event.daddr), &sk->__sk_common.skc_daddr);
    } else { /* family == AF_INET6 */
        bpf_probe_read_kernel(&event.saddr, sizeof(event.saddr), &sk->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr32);
        bpf_probe_read_kernel(&event.daddr, sizeof(event.daddr), &sk->__sk_common.skc_v6_daddr.in6_u.u6_addr32);
    }

    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &event, sizeof(event));

    if (ctx->newstate == TCP_CLOSE)
        bpf_map_delete_elem(&timestamps, &sk);
    else
        bpf_map_update_elem(&timestamps, &sk, &ts, BPF_ANY);

    return 0;
}
```

##  0x03    tcprtt分析

##  0x04 一个关于ip_local_port_range的观测问题
这个问题来自司内分享，在Linux内核版本过高（如`4.14.X`）的场景下，监控本地端口的占用情况，如果此值持续过高（如处于`EST`的状态超过3W）且客户端主动connect并发较高的情况下可能会出现CPU高负载，且负载主要来自于`sys`占用。那么在这种场景下，使用`netstate`/`ss`命令定期统计当前`EST`的端口占用总数就不太合适。如何利用ebpf的方式解决？


##  0x05  参考
-   [eBPF入门实践教程十四：记录 TCP 连接状态与 TCP RTT](https://eunomia.dev/zh/tutorials/14-tcpstates/)
-   [TCP Tracepoints](https://www.brendangregg.com/blog/2018-03-22/tcp-tracepoints.html)
-   [eBPF入门开发实践教程十三：统计 TCP 连接延时，并使用 libbpf 在用户态处理数据](https://eunomia.dev/zh/tutorials/13-tcpconnlat/)