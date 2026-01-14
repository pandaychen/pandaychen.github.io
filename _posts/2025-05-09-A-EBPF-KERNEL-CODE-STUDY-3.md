---
layout:     post
title:      EBPF 内核态代码学习（三）：network stack tracing
subtitle:   tcpstate、tcprtt、tcpconnlat等工具实现分析
date:       2025-05-09
author:     pandaychen
header-img:
catalog: true
tags:
    - eBPF
---

##  0x00 前言
本文学习下基于ebpf技术的网络（协议栈）追踪，主要基于[bcc-v0.35.0](https://github.com/iovisor/bcc/tree/v0.35.0/libbpf-tools)，内核版本参考[5.4.241](https://elixir.bootlin.com/linux/v5.4.241/source/kernel/)

-   [tcpstates](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcpstates.bpf.c)：用于记录 TCP 连接的状态变化，tcpstates主要依赖于 eBPF 的 Tracepoints 来捕获 TCP 连接的状态变化，从而跟踪 TCP 连接在每个状态下的停留时间
-   [tcprtt](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcprtt.bpf.c)：用于记录 TCP 的往返时间（RTT/Round-Trip Time），同样也可以基于cgroup 统计一段时间内 tcp rtt 的分布，显示连接的状态信息
-   [tcpconnect]()：基于 cgroup 监控 tcp 网络连接，显示源IP、目的IP、目的端口等状态信息以及基于 cgroup 统计一段时间内的 tcp 连接数量
-   [tcpconnlat]()：基于 cgroup 监控 tcp 建立连接的时间，显示连接的状态信息
-   [tcptrace]()：基于过滤条件监控 tcp 网络连接，跟踪 skb 报文在内核中的生命周期，输出每个报文在协议栈中各个点的时间延迟、所在 CPU、网络接口等信息
-   [tcplife]()：基于 cgroup 跟踪 tcp 连接的生命周期，显示连接的存活时间等统计信息
-   [tcpdrop]()：基于 cgroup 监控 tcp 网络连接，追踪内核丢弃的数据包，显示数据包地址、端口和调用栈等信息
-   [tcppktlat]()
-   [tcpsynbl]()
-   [tcptop]()

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

```bash
SKADDR           C-PID C-COMM     LADDR           LPORT RADDR           RPORT OLDSTATE    -> NEWSTATE    MS
ffff9fd7e8192000 22384 curl       1.1.1.1  0     2.2.2.2    80    CLOSE       -> SYN_SENT    0.000
ffff9fd7e8192000 0     swapper/5  1.1.1.1  63446 2.2.2.2    80    SYN_SENT    -> ESTABLISHED 1.373
ffff9fd7e8192000 22384 curl       1.1.1.1  63446 2.2.2.2    80    ESTABLISHED -> FIN_WAIT1   176.042
ffff9fd7e8192000 0     swapper/5  1.1.1.1  63446 2.2.2.2    80    FIN_WAIT1   -> FIN_WAIT2   0.536
ffff9fd7e8192000 0     swapper/5  1.1.1.1  63446 2.2.2.2    80    FIN_WAIT2   -> CLOSE       0.006
```

####    内核态实现
实现原理主要是依赖hook `tracepoint/sock/inet_sock_set_state`，当 TCP 连接状态发生变化时，这个 tracepoint 就会被触发，然后执行`handle_set_state`函数统计

```bash
[root@VM-X-X-centos ~]#  cat /sys/kernel/debug/tracing/events/sock/inet_sock_set_state/format 
name: inet_sock_set_state
ID: 1384
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:const void * skaddr;      offset:8;       size:8; signed:0;
        field:int oldstate;     offset:16;      size:4; signed:1;
        field:int newstate;     offset:20;      size:4; signed:1;
        field:__u16 sport;      offset:24;      size:2; signed:0;
        field:__u16 dport;      offset:26;      size:2; signed:0;
        field:__u16 family;     offset:28;      size:2; signed:0;
        field:__u8 protocol;    offset:30;      size:1; signed:0;
        field:__u8 saddr[4];    offset:31;      size:4; signed:0;
        field:__u8 daddr[4];    offset:35;      size:4; signed:0;
        field:__u8 saddr_v6[16];        offset:39;      size:16;        signed:0;
        field:__u8 daddr_v6[16];        offset:55;      size:16;        signed:0;

print fmt: "family=%s protocol=%s sport=%hu dport=%hu saddr=%pI4 daddr=%pI4 saddrv6=%pI6c daddrv6=%pI6c oldstate=%s newstate=%s", __print_symbolic(REC->family, { 2, "AF_INET" }, { 10, "AF_INET6" }), __print_symbolic(REC->protocol, { 6, "IPPROTO_TCP" }, { 33, "IPPROTO_DCCP" }, { 132, "IPPROTO_SCTP" }), REC->sport, REC->dport, REC->saddr, REC->daddr, REC->saddr_v6, REC->daddr_v6, __print_symbolic(REC->oldstate, { 1, "TCP_ESTABLISHED" }, { 2, "TCP_SYN_SENT" }, { 3, "TCP_SYN_RECV" }, { 4, "TCP_FIN_WAIT1" }, { 5, "TCP_FIN_WAIT2" }, { 6, "TCP_TIME_WAIT" }, { 7, "TCP_CLOSE" }, { 8, "TCP_CLOSE_WAIT" }, { 9, "TCP_LAST_ACK" }, { 10, "TCP_LISTEN" }, { 11, "TCP_CLOSING" }, { 12, "TCP_NEW_SYN_RECV" }), __print_symbolic(REC->newstate, { 1, "TCP_ESTABLISHED" }, { 2, "TCP_SYN_SENT" }, { 3, "TCP_SYN_RECV" }, { 4, "TCP_FIN_WAIT1" }, { 5, "TCP_FIN_WAIT2" }, { 6, "TCP_TIME_WAIT" }, { 7, "TCP_CLOSE" }, { 8, "TCP_CLOSE_WAIT" }, { 9, "TCP_LAST_ACK" }, { 10, "TCP_LISTEN" }, { 11, "TCP_CLOSING" }, { 12, "TCP_NEW_SYN_RECV" })
```

```cpp
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, MAX_ENTRIES);
    __type(key, __u16);
    __type(value, __u16);
} sports SEC(".maps");  //存储源端口（过滤 TCP 连接）

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, MAX_ENTRIES);
    __type(key, __u16);
    __type(value, __u16);
} dports SEC(".maps");  //存储目标端口（过滤 TCP 连接）

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, MAX_ENTRIES);
    __type(key, struct sock *);
    __type(value, __u64);
} timestamps SEC(".maps");
//timestamps用于存储每个 TCP 连接的时间戳，以计算每个状态的停留时间

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
    
    //从timestampsmap 中获取当前连接的上一个时间戳，然后计算出停留在当前状态的时间
    //并输出状态转移日志
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
        // 读取inet结构体
        bpf_probe_read_kernel(&event.saddr, sizeof(event.saddr), &sk->__sk_common.skc_rcv_saddr);
        bpf_probe_read_kernel(&event.daddr, sizeof(event.daddr), &sk->__sk_common.skc_daddr);
    } else { /* family == AF_INET6 */
        bpf_probe_read_kernel(&event.saddr, sizeof(event.saddr), &sk->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr32);
        bpf_probe_read_kernel(&event.daddr, sizeof(event.daddr), &sk->__sk_common.skc_v6_daddr.in6_u.u6_addr32);
    }

    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &event, sizeof(event));

    // 结束时，判断TCP连接的当前状态并更新
    // 根据 TCP 连接的新状态，程序将进行不同的操作：如果新状态为 TCP_CLOSE，表示连接已关闭，程序将从timestampsmap 中删除该连接的时间戳
    // 否则，程序将更新该连接的时间戳
    if (ctx->newstate == TCP_CLOSE)
        bpf_map_delete_elem(&timestamps, &sk);
    else
        bpf_map_update_elem(&timestamps, &sk, &ts, BPF_ANY);

    return 0;
}
```

##  0x03    tcprtt分析
tcprtt用于测量 TCP 往返时间，它将 RTT 的信息统计到一个 histogram 中。内核态实现基于`fentry/tcp_rcv_established`或者kprobe，其hook函数原型如下（[前文](https://pandaychen.github.io/2025/04/25/A-LINUX-KERNEL-TRAVEL-12/#fast-path-vs-slow-path)已分析）

```cpp
//https://elixir.bootlin.com/linux/v5.4.241/source/net/ipv4/tcp_input.c#L5610
// 该函数在每次内核中处理 TCP 收包的时候被调用
void tcp_rcv_established(struct sock *sk, struct sk_buff *skb)
```

`tcp_rcv_established`主要在TCP连接处于`ESTABLISHED`状态时被调用。这个函数的处理逻辑包括一个快速路径和一个慢速路径（客户端服务端都适用）

其中，快速路径在以下几种情况下会被禁用（会走到`tcp_data_queue`慢速路径）

-   宣布了一个零窗口：零窗口探测只能在慢速路径中正确处理
-   收到了乱序的数据包
-   期待接收紧急数据
-   没有剩余的缓冲区空间
-   接收到了意外的TCP标志/窗口值/头部长度（通过检查TCP头部与预设标志进行检测）
-   **数据在双向都在传输，快速路径只支持纯发送者或纯接收者（这意味着序列号或确认值必须保持不变）**
-   接收到了意外的TCP选项

此外，代码中涉及到一些细节：

-   `hists`变量：普通map，用来存储 RTT 的统计信息。key为 `64` 位整数，值是一个`hist`结构，这个结构包含了一个数组，用来存储不同 RTT 区间的数量
-   TCP `struct tcp_sock`结构的`srtt_us`字段，表示了平滑的 RTT 值（单位：微秒），内核态代码中会将这个 RTT 值转换为对数形式，并将其作为 slot 存储到 histogram 中

```cpp
struct hist {
	__u64 latency;
	__u64 cnt;
	__u32 slots[MAX_SLOTS];
};

struct hist_key {
	__u16 family;
	__u8 addr[IPV6_LEN];
};
```

内核态实现主要逻辑如下：

1.  根据过滤规则对 TCP 连接进行过滤
2.  在`hists` map 中查找或者初始化对应的 histogram
3.  读取 TCP 连接的`srtt_us`字段，并将其转换为对数形式，存储到 histogram 中

```cpp
/// @sample {"interval": 1000, "type" : "log2_hist"}
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, MAX_ENTRIES);
    __type(key, u64);
    __type(value, struct hist);
} hists SEC(".maps");

static struct hist zero;

SEC("fentry/tcp_rcv_established")
int BPF_PROG(tcp_rcv, struct sock *sk)
{
    //获取sock五元组信息
    const struct inet_sock *inet = (struct inet_sock *)(sk);
    struct tcp_sock *ts;
    struct hist *histp;
    u64 key, slot;
    u32 srtt;

    // 过滤条件
    if (targ_sport && targ_sport != inet->inet_sport)
        return 0;
    if (targ_dport && targ_dport != sk->__sk_common.skc_dport)
        return 0;
    if (targ_saddr && targ_saddr != inet->inet_saddr)
        return 0;
    if (targ_daddr && targ_daddr != sk->__sk_common.skc_daddr)
        return 0;

    if (targ_laddr_hist)
        key = inet->inet_saddr;
    else if (targ_raddr_hist)
        key = inet->sk.__sk_common.skc_daddr;
    else
        key = 0;
    histp = bpf_map_lookup_or_try_init(&hists, &key, &zero);
    if (!histp)
        return 0;
    ts = (struct tcp_sock *)(sk);
    // 核心操作
    srtt = BPF_CORE_READ(ts, srtt_us) >> 3;
    if (targ_ms)
        srtt /= 1000U;
    slot = log2l(srtt);
    if (slot >= MAX_SLOTS)
        slot = MAX_SLOTS - 1;
    __sync_fetch_and_add(&histp->slots[slot], 1);
    if (targ_show_ext) {
        //如果设置了show_ext参数，将 RTT 值和计数器累加到 histogram 的latency和cnt字段中
        __sync_fetch_and_add(&histp->latency, srtt);
        __sync_fetch_and_add(&histp->cnt, 1);
    }
    return 0;
}
```

##  0x04    tcpconnect
[`tcpconnect`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcpconnect.bpf.c)

##  0x05    tcpconnlat
[`tcpconnlat`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcpconnlat.bpf.c)

##  0x06    tcplife
[`tcplife`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcplife.bpf.c)

##  0x07    tcppktlat
[`tcppktlat`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcppktlat.bpf.c)

##  0x08    tcpsynbl
[`tcpsynbl`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcpsynbl.bpf.c)

##  0x09    tcptop
[`tcptop`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcptop.bpf.c)

##  0x0A    tcptracer
[tcptracer](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcptracer.bpf.c)

##  0x0B    tcpdrop

##  0x0C    tcpaccept

##  0x0D tcp-retrans

##  0x0E tcpsubnet

##  0x0F 一个关于ip_local_port_range的观测问题
这个问题来自司内分享，在Linux内核版本过高（如`4.14.x`）的场景下，监控本地端口的占用情况，如果此值持续过高（如处于`EST`的状态超过3W）且客户端主动connect并发较高的情况下可能会出现CPU高负载，且负载主要来自于`sys`占用。那么在这种场景下，使用`netstate`/`ss`命令定期统计当前`EST`的端口占用总数就不太合适。如何利用ebpf的方式解决？


##  0x10  参考
-   [eBPF入门实践教程十四：记录 TCP 连接状态与 TCP RTT](https://eunomia.dev/zh/tutorials/14-tcpstates/)
-   [TCP Tracepoints](https://www.brendangregg.com/blog/2018-03-22/tcp-tracepoints.html)
-   [eBPF入门开发实践教程十三：统计 TCP 连接延时，并使用 libbpf 在用户态处理数据](https://eunomia.dev/zh/tutorials/13-tcpconnlat/)
-   [tcpconnect - TCP Connection Monitoring](https://ebee.xmigrate.cloud/tools/tcpconnect/)