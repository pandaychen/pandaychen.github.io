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
本文学习下基于ebpf技术的网络（协议栈）追踪，主要基于[bcc-v0.35.0](https://github.com/iovisor/bcc/tree/v0.35.0/libbpf-tools)，内核版本（非注明）参考[5.4.241](https://elixir.bootlin.com/linux/v5.4.241/source/kernel/)

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

####    内核态实现tcpstates.bpf.c
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

内核态实现`tcprtt.bpf.c`主要逻辑如下：

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

##  0x04    tcpconnect：主动 TCP 连接跟踪
内核态实现[`tcpconnect.bpf.c`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcpconnect.bpf.c)

```bash
PID    COMM             IP SADDR            DADDR            DPORT
1055869 xxx      4  9.1.2.3    9.1.2.4    8443 
1789533 xxx1  4  9.1.2.3    9.1.2.4    8443 
```

##  0x05    tcpconnlat
内核态实现[`tcpconnlat.bpf.c`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcpconnlat.bpf.c)

##  0x06    tcplife
内核态实现[`tcplife.bpf.c`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcplife.bpf.c)

##  0x07    tcppktlat
内核态实现[`tcppktlat.bpf.c`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcppktlat.bpf.c)

##  0x08    tcpsynbl
内核态实现[`tcpsynbl.bpf.c`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcpsynbl.bpf.c)

##  0x09    tcptop：跟踪 IP 端口的网络吞吐量
该工具以 `KB` 为单位显示主机发送并接收的 TCP 流量，会自动刷新并只包含活跃的 TCP 连接（若连接关闭，则不再可见）。内核态实现[`tcptop.bpf.c`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcptop.bpf.c)，可以使用`-C`选项不清屏打印

```bash
13:46:29 loadavg: 0.10 0.03 0.01 1/215 3875

PID    COMM         LADDR           RADDR              RX_KB   TX_KB
3853   3853         192.0.2.1:22    192.0.2.165:41838  32     102626
1285   sshd         192.0.2.1:22    192.0.2.45:39240   0           0
```

####    内核态实现

TODO

```cpp
const volatile bool filter_cg = false;
const volatile pid_t target_pid = -1;
const volatile int target_family = -1;

struct {
        __uint(type, BPF_MAP_TYPE_CGROUP_ARRAY);
        __type(key, u32);
        __type(value, u32);
        __uint(max_entries, 1);
} cgroup_map SEC(".maps");

struct {
        __uint(type, BPF_MAP_TYPE_HASH);
        __uint(max_entries, 10240);
        __type(key, struct ip_key_t);
        __type(value, struct traffic_t);
} ip_map SEC(".maps");

static int probe_ip(bool receiving, struct sock *sk, size_t size)
{
        struct ip_key_t ip_key = {};
        struct traffic_t *trafficp;
        u16 family;
        u32 pid;

        if (filter_cg && !bpf_current_task_under_cgroup(&cgroup_map, 0))
                return 0;

        pid = bpf_get_current_pid_tgid() >> 32;
        if (target_pid != -1 && target_pid != pid)
                return 0;

        family = BPF_CORE_READ(sk, __sk_common.skc_family);
        if (target_family != -1 && target_family != family)
                return 0;

        /* drop */
        if (family != AF_INET && family != AF_INET6)
                return 0;

        ip_key.pid = pid;
        bpf_get_current_comm(&ip_key.name, sizeof(ip_key.name));
        ip_key.lport = BPF_CORE_READ(sk, __sk_common.skc_num);
        ip_key.dport = bpf_ntohs(BPF_CORE_READ(sk, __sk_common.skc_dport));
        ip_key.family = family;

        if (family == AF_INET) {
                bpf_probe_read_kernel(&ip_key.saddr,
                                      sizeof(sk->__sk_common.skc_rcv_saddr),
                                      &sk->__sk_common.skc_rcv_saddr);
                bpf_probe_read_kernel(&ip_key.daddr,
                                      sizeof(sk->__sk_common.skc_daddr),
                                      &sk->__sk_common.skc_daddr);
        } else {
                /*
                 * family == AF_INET6,
                 * we already checked above family is correct.
                 */
                bpf_probe_read_kernel(&ip_key.saddr,
                                      sizeof(sk->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr32),
                                      &sk->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr32);
                bpf_probe_read_kernel(&ip_key.daddr,
                                      sizeof(sk->__sk_common.skc_v6_daddr.in6_u.u6_addr32),
                                      &sk->__sk_common.skc_v6_daddr.in6_u.u6_addr32);
        }

        trafficp = bpf_map_lookup_elem(&ip_map, &ip_key);
        if (!trafficp) {
                struct traffic_t zero;

                if (receiving) {
                        zero.sent = 0;
                        zero.received = size;
                } else {
                        zero.sent = size;
                        zero.received = 0;
                }

                bpf_map_update_elem(&ip_map, &ip_key, &zero, BPF_NOEXIST);
        } else {
                if (receiving)
                        trafficp->received += size;
                else
                        trafficp->sent += size;

                bpf_map_update_elem(&ip_map, &ip_key, trafficp, BPF_EXIST);
        }

        return 0;
}

SEC("kprobe/tcp_sendmsg")
int BPF_KPROBE(tcp_sendmsg, struct sock *sk, struct msghdr *msg, size_t size)
{
        return probe_ip(false, sk, size);
}

/*
 * tcp_recvmsg() would be obvious to trace, but is less suitable because:
 * - we'd need to trace both entry and return, to have both sock and size
 * - misses tcp_read_sock() traffic
 * we'd much prefer tracepoints once they are available.
 */
SEC("kprobe/tcp_cleanup_rbuf")
int BPF_KPROBE(tcp_cleanup_rbuf, struct sock *sk, int copied)
{
        if (copied <= 0)
                return 0;

        return probe_ip(true, sk, copied);
}
```

##  0x0A    tcptracer
内核态实现[tcptracer.bpf.c](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcptracer.bpf.c)，本工具用于追踪内核中与 TCP 连接建立（通过 `connect()/accept()`等）和关闭（`close()`或异常退出等）相关的函数，典型输入如下

-   `C` 连接（Connect）：表示一个 TCP 连接请求已经发送或接收
-   `X` 关闭（Close）：表示 TCP 连接已经关闭，可能是由于正常关闭（通过 `FIN/ACK` 握手）或由于某种错误导致的异常关闭
-   `A` 接受（Accept）：通常表示服务器已经接受了一个来自客户端的连接请求，并创建了一个新的连接。每当内核连接、接受或关闭连接时，tcptracer 都会显示连接的详情

```bash
Tracing TCP established connections. Ctrl-C to end.
T  PID    COMM             IP SADDR            DADDR            SPORT  DPORT
C  28943  telnet           4  192.168.1.2      192.168.1.1      59306  23
C  28818  curl             6  [::1]            [::1]            55758  80
X  28943  telnet           4  192.168.1.2      192.168.1.1      59306  23
A  28817  nc               6  [::1]            [::1]            80     55758
X  28818  curl             6  [::1]            [::1]            55758  80
X  28817  nc               6  [::1]            [::1]            80     55758
```

####    过滤条件

```bash
tcptracer: Trace TCP connections

EXAMPLES:
    tcptracer             # trace all TCP connections
    tcptracer -t          # include timestamps
    tcptracer -p 181      # only trace PID 181
    tcptracer -U          # include UID
    tcptracer -u 1000     # only trace UID 1000
    tcptracer --C mappath # only trace cgroups in the map
    tcptracer --M mappath # only trace mount namespaces in the map

  -C, --cgroupmap=PATH       trace cgroups in this map
  -M, --mntnsmap=PATH        trace mount namespaces in this map
  -p, --pid=PID              Process PID to trace
  -t, --timestamp            Include timestamp on output
  -u, --uid=UID              Process UID to trace
  -U, --print-uid            Include UID on output
  -v, --verbose              Verbose debug output
  -?, --help                 Give this help list
      --usage                Give a short usage message
  -V, --version              Print program version
```

####    tcptracer 设计的目的
tcptracer工具的核心设计是解决追踪TCP整个生命周期（建立到消失） 事件追踪时，PID 的丢失问题，这里的TCP包含主动connect与被动accept两种模式

PID丢失问题主要是在监控连接建立的相关hook `tcp_set_state/inet_sock_set_state`，然而当内核通过异步方式（如内核收到对端的 ACK 包触发状态机）将连接设为 `ESTABLISHED` 时，当前运行的上下文一般是内核中断，而不是发起 connect() 的进程（通过ebpf捕获的`comm`为内核线程非用户态进程）

因此 tcptracer 设计用`tuplepid`的hash表来解决上面的问题，简单描述如下：

1.  第一阶段 (connect)：在 `tcp_v4_connect` 进入时记录当前进程的 PID/COMM，以网络五元组（`tuple_key_t`）作为 Key 存入 `tuplepid`
2.  第二阶段 (set_state)：当 TCP 状态变为 `ESTABLISHED` 时，通过当前的五元组去 Map 里查表，找回第一阶段存入的 PID，从而避免了PID丢失

####    内核态实现
列举下相关的hook（IPV4），基于hook的顺序和功能如下：

```cpp
//https://elixir.bootlin.com/linux/v5.4.241/source/net/ipv4/tcp_ipv4.c#L199
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len);

//https://elixir.bootlin.com/linux/v5.4.241/source/net/ipv4/tcp.c#L2225
void tcp_set_state(struct sock *sk, int state)

//https://elixir.bootlin.com/linux/v5.4.241/source/net/ipv4/inet_connection_sock.c#L451
struct sock *inet_csk_accept(struct sock *sk, int flags, int *err, bool kern)

//https://elixir.bootlin.com/linux/v5.4.241/source/net/ipv4/tcp.c#L2353
void tcp_close(struct sock *sk, long timeout)
```

上面hook对应的顺序如下：

1.  `kprobe/connect`：客户端请求连接（主动连接），`TID`（key）->`sk`（value），存入`sockets` map
2.  `kretprobe/connect`：检查返回值，提取五元组 -> `PID/COMM`，存入 `tuplepid` map
3.  `kprobe/set_state`：从 `tuplepid` 查出 `PID`，然后输出 `CONNECT` 事件
4.  `kretprobe/accept`：被动连接，直接获取 `PID`，输出 `ACCEPT` 事件
5.  `kprobe/close`：连接关闭，直接获取 `PID`，输出 `CLOSE` 事件

代码中涉及到hash表的用途：

-   `tuplepid`：存储五元组到 `PID/COMM` 的映射，用于跨阶段查找进程相关信息
-   `sockets`：临时存储当前正在执行 `connect` 系统调用的套接字指针

```cpp
const volatile uid_t filter_uid = -1;
const volatile pid_t filter_pid = 0;

/* Define here, because there are conflicts with include files */
#define AF_INET         2
#define AF_INET6        10

/*
 * tcp_set_state doesn't run in the context of the process that initiated the
 * connection so we need to store a map TUPLE -> PID to send the right PID on
 * the event.
 */
struct tuple_key_t {
        union {
                __u32 saddr_v4;
                unsigned __int128 saddr_v6;
        };
        union {
                __u32 daddr_v4;
                unsigned __int128 daddr_v6;
        };
        u16 sport;
        u16 dport;
        u32 netns;  //ns
};

struct pid_comm_t {
        u64 pid;
        char comm[TASK_COMM_LEN];
        u32 uid;
};

struct {
        __uint(type, BPF_MAP_TYPE_HASH);
        __uint(max_entries, MAX_ENTRIES);
        __type(key, struct tuple_key_t);
        __type(value, struct pid_comm_t);
} tuplepid SEC(".maps");

struct {
        __uint(type, BPF_MAP_TYPE_HASH);
        __uint(max_entries, MAX_ENTRIES);
        __type(key, u32);
        __type(value, struct sock *);
} sockets SEC(".maps");

struct {
        __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
        __uint(key_size, sizeof(u32));
        __uint(value_size, sizeof(u32));
} events SEC(".maps");


//从 struct sock 中提取关键信息
static __always_inline bool
fill_tuple(struct tuple_key_t *tuple, struct sock *sk, int family)
{
        struct inet_sock *sockp = (struct inet_sock *)sk;

        // NetNS Inode：sk->__sk_common.skc_net.net->ns.inum
        BPF_CORE_READ_INTO(&tuple->netns, sk, __sk_common.skc_net.net, ns.inum);

        //IP 地址：根据 AF_INET/AF_INET6 提取源地址和目的地址
        switch (family) {
        case AF_INET:
                BPF_CORE_READ_INTO(&tuple->saddr_v4, sk, __sk_common.skc_rcv_saddr);
                if (tuple->saddr_v4 == 0)
                        return false;

                BPF_CORE_READ_INTO(&tuple->daddr_v4, sk, __sk_common.skc_daddr);
                if (tuple->daddr_v4 == 0)
                        return false;

                break;
        case AF_INET6:
                BPF_CORE_READ_INTO(&tuple->saddr_v6, sk,
                                   __sk_common.skc_v6_rcv_saddr.in6_u.u6_addr32);
                if (tuple->saddr_v6 == 0)
                        return false;
                BPF_CORE_READ_INTO(&tuple->daddr_v6, sk,
                                   __sk_common.skc_v6_daddr.in6_u.u6_addr32);
                if (tuple->daddr_v6 == 0)
                        return false;

                break;
        /* it should not happen but to be sure let's handle this case */
        default:
                return false;
        }

        //端口号：提取源端口 (skc_num) 和目的端口 (skc_dport)
        BPF_CORE_READ_INTO(&tuple->dport, sk, __sk_common.skc_dport);
        if (tuple->dport == 0)
                return false;

        BPF_CORE_READ_INTO(&tuple->sport, sockp, inet_sport);
        if (tuple->sport == 0)
                return false;

        return true;
}

static __always_inline void
fill_event(struct tuple_key_t *tuple, struct event *event, __u32 pid,
           __u32 uid, __u16 family, __u8 type)
{
        event->ts_us = bpf_ktime_get_ns() / 1000;
        event->type = type;
        event->pid = pid;
        event->uid = uid;
        event->af = family;
        event->netns = tuple->netns;
        if (family == AF_INET) {
                event->saddr_v4 = tuple->saddr_v4;
                event->daddr_v4 = tuple->daddr_v4;
        } else {
                event->saddr_v6 = tuple->saddr_v6;
                event->daddr_v6 = tuple->daddr_v6;
        }
        event->sport = tuple->sport;
        event->dport = tuple->dport;
}

/* returns true if the event should be skipped */
static __always_inline bool
filter_event(struct sock *sk, __u32 uid, __u32 pid)
{
        u16 family = BPF_CORE_READ(sk, __sk_common.skc_family);

        if (family != AF_INET && family != AF_INET6)
                return true;

        if (filter_pid && pid != filter_pid)
                return true;

        if (filter_uid != (uid_t) -1 && uid != filter_uid)
                return true;

        return false;
}

static __always_inline int
enter_tcp_connect(struct sock *sk)
{
        __u64 pid_tgid = bpf_get_current_pid_tgid();
        __u32 pid = pid_tgid >> 32;
        __u32 tid = pid_tgid;
        __u64 uid_gid = bpf_get_current_uid_gid();
        __u32 uid = uid_gid;

        if (filter_event(sk, uid, pid))
                return 0;

        bpf_map_update_elem(&sockets, &tid, &sk, 0);
        return 0;
}

static __always_inline int
exit_tcp_connect(int ret, __u16 family)
{
        __u64 pid_tgid = bpf_get_current_pid_tgid();
        __u32 pid = pid_tgid >> 32;
        __u32 tid = pid_tgid;
        __u64 uid_gid = bpf_get_current_uid_gid();
        __u32 uid = uid_gid;
        struct tuple_key_t tuple = {};
        struct pid_comm_t pid_comm = {};
        struct sock **skpp;
        struct sock *sk;

        skpp = bpf_map_lookup_elem(&sockets, &tid);
        if (!skpp)
                return 0;

        if (ret)
                goto end;

        sk = *skpp;

        if (!fill_tuple(&tuple, sk, family))
                goto end;

        pid_comm.pid = pid;
        pid_comm.uid = uid;
        bpf_get_current_comm(&pid_comm.comm, sizeof(pid_comm.comm));

        bpf_map_update_elem(&tuplepid, &tuple, &pid_comm, 0);

end:
        bpf_map_delete_elem(&sockets, &tid);
        return 0;
}

//kprobe (enter)：在函数开始处，将当前线程 ID (tid) 和 sock 指针关联存入 sockets Map
SEC("kprobe/tcp_v4_connect")
int BPF_KPROBE(tcp_v4_connect, struct sock *sk)
{
        return enter_tcp_connect(sk);
}

//kretprobe (exit)：在函数返回处，确认连接发起成功后，提取五元组，并连同当前进程的 pid、comm 写入 tuplepid 映射表，为后续状态转换做准备
SEC("kretprobe/tcp_v4_connect")
int BPF_KRETPROBE(tcp_v4_connect_ret, int ret)
{
        return exit_tcp_connect(ret, AF_INET);
}

......

/*tcp_close：连接关闭跟踪，监控连接关闭。

-   状态过滤：为了避免垃圾数据，会检查 oldstate。如果连接还没建立好就关了（如 SYN_SENT 状态），则不生成事件
*/
SEC("kprobe/tcp_close")
int BPF_KPROBE(entry_trace_close, struct sock *sk)
{
        __u64 pid_tgid = bpf_get_current_pid_tgid();
        __u32 pid = pid_tgid >> 32;
        __u64 uid_gid = bpf_get_current_uid_gid();
        __u32 uid = uid_gid;
        struct tuple_key_t tuple = {};
        struct event event = {};
        u16 family;

        if (filter_event(sk, uid, pid))
                return 0;

        /*
         * Don't generate close events for connections that were never
         * established in the first place.
         */
        
        //如果连接没有建立好，则退出
        u8 oldstate = BPF_CORE_READ(sk, __sk_common.skc_state);
        if (oldstate == TCP_SYN_SENT ||
            oldstate == TCP_SYN_RECV ||
            oldstate == TCP_NEW_SYN_RECV)
                return 0;

        family = BPF_CORE_READ(sk, __sk_common.skc_family);
        if (!fill_tuple(&tuple, sk, family))
                return 0;

        fill_event(&tuple, &event, pid, uid, family, TCP_EVENT_TYPE_CLOSE);
        bpf_get_current_comm(&event.task, sizeof(event.task));

        bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU,
                      &event, sizeof(event));

        return 0;
};


//tcp_set_state：状态机跟踪
/*
-   监控 TCP_ESTABLISHED（连接建立成功）事件
-   通过五元组在 tuplepid 中查找原始进程信息
-   如果找到了，会调用 bpf_perf_event_output 将一个 TCP_EVENT_TYPE_CONNECT 类型的事件发送到用户态
*/
SEC("kprobe/tcp_set_state")
int BPF_KPROBE(enter_tcp_set_state, struct sock *sk, int state)
{
        struct tuple_key_t tuple = {};
        struct event event = {};
        __u16 family;

        if (state != TCP_ESTABLISHED && state != TCP_CLOSE)
                goto end;

        family = BPF_CORE_READ(sk, __sk_common.skc_family);

        if (!fill_tuple(&tuple, sk, family))
                goto end;

        if (state == TCP_CLOSE)
                goto end;

        struct pid_comm_t *p;
        p = bpf_map_lookup_elem(&tuplepid, &tuple);
        if (!p)
                return 0; /* missed entry */

        fill_event(&tuple, &event, p->pid, p->uid, family, TCP_EVENT_TYPE_CONNECT);
        __builtin_memcpy(&event.task, p->comm, sizeof(event.task));

        bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU,&event, sizeof(event));

end:
        bpf_map_delete_elem(&tuplepid, &tuple);

        return 0;
}

/*inet_csk_accept：被动连接跟踪，对于作为服务器端被动接收的连接

-   由于 accept 返回时，新连接已经完全建立且就在当前进程上下文中，因此不需要复杂的查表
-   直接从返回的 sk 中提取信息，发送 TCP_EVENT_TYPE_ACCEPT 事件
*/
SEC("kretprobe/inet_csk_accept")
int BPF_KRETPROBE(exit_inet_csk_accept, struct sock *sk)
{
        __u64 pid_tgid = bpf_get_current_pid_tgid();
        __u32 pid = pid_tgid >> 32;
        __u64 uid_gid = bpf_get_current_uid_gid();
        __u32 uid = uid_gid;
        __u16 sport, family;
        struct event event = {};

        if (!sk)
                return 0;

        if (filter_event(sk, uid, pid))
                return 0;

        family = BPF_CORE_READ(sk, __sk_common.skc_family);
        sport = BPF_CORE_READ(sk, __sk_common.skc_num);

        struct tuple_key_t t = {};
        fill_tuple(&t, sk, family);
        t.sport = bpf_ntohs(sport);
        /* do not send event if IP address is 0.0.0.0 or port is 0 */
        if (t.saddr_v6 == 0 || t.daddr_v6 == 0 || t.dport == 0 || t.sport == 0)
                return 0;

        fill_event(&t, &event, pid, uid, family, TCP_EVENT_TYPE_ACCEPT);

        bpf_get_current_comm(&event.task, sizeof(event.task));
        bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU,&event, sizeof(event));

        return 0;
}
```

最后，代码中使用基于CO-RE技术的`BPF_CORE_READ_*`系列宏来解决兼容问题，如`BPF_CORE_READ_INTO(&tuple->netns, sk, __sk_common.skc_net.net, ns.inum)`，因为在不同内核版本中，`struct net` 结构体里的 `ns.inum` 偏移量可能会变，其原理如前文描述，**在编译时记录下这个字段的名字，在加载到内核时，由 libbpf 根据目标内核的 BTF 信息动态修正偏移量**

##  0x0B    tcpdrop
tcpdrop监控 Linux 内核在何处、因何种原因丢弃了 TCP 数据包。可以直接定位到数据包是被防火墙拦截了、校验和错误，还是因为内核缓冲区满等原因

```PYTHON
# define BPF program
bpf_text = """
#include <uapi/linux/ptrace.h>
#include <uapi/linux/tcp.h>
#include <uapi/linux/ip.h>
#include <net/sock.h>
#include <bcc/proto.h>
#include <linux/skbuff.h>

BPF_STACK_TRACE(stack_traces, 1024);

// separate data structs for ipv4 and ipv6
struct ipv4_data_t {
    u32 pid;
    u64 ip;
    u32 saddr;
    u32 daddr;
    u16 sport;
    u16 dport;
    u8 state;
    u8 tcpflags;
    u32 stack_id;
    u32 drop_reason;
};
BPF_PERF_OUTPUT(ipv4_events);

struct ipv6_data_t {
    u32 pid;
    u64 ip;
    unsigned __int128 saddr;
    unsigned __int128 daddr;
    u16 sport;
    u16 dport;
    u8 state;
    u8 tcpflags;
    u32 stack_id;
    u32 drop_reason;
};
BPF_PERF_OUTPUT(ipv6_events);

static struct tcphdr *skb_to_tcphdr(const struct sk_buff *skb)
{
    // unstable API. verify logic in tcp_hdr() -> skb_transport_header().
    return (struct tcphdr *)(skb->head + skb->transport_header);
}

static inline struct iphdr *skb_to_iphdr(const struct sk_buff *skb)
{
    // unstable API. verify logic in ip_hdr() -> skb_network_header().
    return (struct iphdr *)(skb->head + skb->network_header);
}

// from include/net/tcp.h:
#ifndef tcp_flag_byte
#define tcp_flag_byte(th) (((u_int8_t *)th)[13])
#endif

static int __trace_tcp_drop(void *ctx, struct sock *sk, struct sk_buff *skb, u32 reason)
{
    if (sk == NULL)
        return 0;
    u32 pid = bpf_get_current_pid_tgid() >> 32;

    // pull in details from the packet headers and the sock struct
    u16 family = sk->__sk_common.skc_family;
    char state = sk->__sk_common.skc_state;
    u16 sport = 0, dport = 0;
    struct tcphdr *tcp = skb_to_tcphdr(skb);
    struct iphdr *ip = skb_to_iphdr(skb);
    u8 tcpflags = ((u_int8_t *)tcp)[13];
    sport = tcp->source;
    dport = tcp->dest;
    sport = ntohs(sport);
    dport = ntohs(dport);

    FILTER_FAMILY

    FILTER_NETNS

    if (family == AF_INET) {
        struct ipv4_data_t data4 = {};
        data4.pid = pid;
        data4.ip = 4;
        data4.saddr = ip->saddr;
        data4.daddr = ip->daddr;
        data4.dport = dport;
        data4.sport = sport;
        data4.state = state;
        data4.tcpflags = tcpflags;
        data4.stack_id = stack_traces.get_stackid(ctx, 0);
        data4.drop_reason = reason;
        ipv4_events.perf_submit(ctx, &data4, sizeof(data4));

    } else if (family == AF_INET6) {
        struct ipv6_data_t data6 = {};
        data6.pid = pid;
        data6.ip = 6;
        // The remote address (skc_v6_daddr) was the source
        bpf_probe_read_kernel(&data6.saddr, sizeof(data6.saddr),
            sk->__sk_common.skc_v6_daddr.in6_u.u6_addr32);
        // The local address (skc_v6_rcv_saddr) was the destination
        bpf_probe_read_kernel(&data6.daddr, sizeof(data6.daddr),
            sk->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr32);
        data6.dport = dport;
        data6.sport = sport;
        data6.state = state;
        data6.tcpflags = tcpflags;
        data6.stack_id = stack_traces.get_stackid(ctx, 0);
        data6.drop_reason = reason;
        ipv6_events.perf_submit(ctx, &data6, sizeof(data6));
    }
    // else drop

    return 0;
}

int trace_tcp_drop(struct pt_regs *ctx, struct sock *sk, struct sk_buff *skb)
{
    // tcp_drop() does not supply a drop reason.
    return __trace_tcp_drop(ctx, sk, skb, SKB_DROP_REASON_NOT_SPECIFIED);
}
"""

bpf_kfree_skb_text = """

TRACEPOINT_PROBE(skb, kfree_skb) {
    struct sk_buff *skb = args->skbaddr;
    struct sock *sk = skb->sk;
    enum skb_drop_reason reason = args->reason;

    // SKB_NOT_DROPPED_YET,
    // SKB_DROP_REASON_NOT_SPECIFIED,
    if (reason > SKB_DROP_REASON_NOT_SPECIFIED) {
        return __trace_tcp_drop(args, sk, skb, (u32)reason);
    }

    return 0;
}
"""

if debug or args.ebpf:
    print(bpf_text)
    if args.ebpf:
        exit()
if args.ipv4:
    bpf_text = bpf_text.replace('FILTER_FAMILY',
        'if (family != AF_INET) { return 0; }')
elif args.ipv6:
    bpf_text = bpf_text.replace('FILTER_FAMILY',
        'if (family != AF_INET6) { return 0; }')
else:
    bpf_text = bpf_text.replace('FILTER_FAMILY', '')

if args.pid_netns != 0:
    if args.netns_id != 0:
        print("ERROR: --pid_netns and --netns-id not allowed together")
        exit(1)
    args.netns_id = os.stat('/proc/{}/ns/net'.format(args.pid_netns)).st_ino

if args.netns_id != 0:
    code = 'if (sk->__sk_common.skc_net.net->ns.inum != {}) {{ return 0; }}'.format(
        args.netns_id)
    bpf_text = bpf_text.replace('FILTER_NETNS', code)
else:
    bpf_text = bpf_text.replace('FILTER_NETNS', '')

# the reasons of skb drop
drop_reasons = {
    0: "SKB_NOT_DROPPED_YET",
    1: "SKB_CONSUMED",
    2: "NOT_SPECIFIED",
    3: "NO_SOCKET",
    .......
}

kfree_skb_traceable = False

if BPF.tracepoint_exists("skb", "kfree_skb"):
    if BPF.kernel_struct_has_field("trace_event_raw_kfree_skb", "reason") == 1:
        bpf_text += bpf_kfree_skb_text
        kfree_skb_traceable = True

# initialize BPF
b = BPF(text=bpf_text)

if b.get_kprobe_functions(b"tcp_drop"):
    b.attach_kprobe(event="tcp_drop", fn_name="trace_tcp_drop")
elif b.tracepoint_exists("skb", "kfree_skb") and kfree_skb_traceable:
    print("WARNING: tcp_drop() kernel function not found or traceable. "
          "Use tracepoint:skb:kfree_skb instead.")
else:
    print("ERROR: tcp_drop() kernel function and tracepoint:skb:kfree_skb"
          " not found or traceable. "
          "The kernel might be too old or the the function has been inlined.")
    exit(1)
stack_traces = b.get_table("stack_traces")
```

额外多说一点，TODO

https://elixir.bootlin.com/linux/v5.17.6/source/net/core/skbuff.c#L770

####    基于命名空间的过滤
在Docker 环境中，宿主机上运行海量容器，每个容器都有自己的网络栈（Namespace）。下面这段代码，可以控制只捕获指定命名空间 ID下面的tcpdrop事件

```python
if args.netns_id != 0:
    code = 'if (sk->__sk_common.skc_net.net->ns.inum != {}) {{ return 0; }}'.format(
        args.netns_id)
    bpf_text = bpf_text.replace('FILTER_NETNS', code)
else:
    bpf_text = bpf_text.replace('FILTER_NETNS', '')
```

实现原理：

1.  `args.netns_id` 是用户通过参数（如 `--netns-id` 或 `--pid-netns`）传入的 Namespace 节点号
2.  通过 `sk->__sk_common.skc_net.net->ns.inum` 访问当前处理这个包的套接字所属的命名空间 ID
3.  如果 ID 不匹配，直接丢弃

##  0x0C    tcpaccept
   TCP 被动连接（即 accept 成功的连接），基于hook点为`kretprobe/inet_csk_accept`，函数原型如下。该函数`inet_csk_accept`的主要功能是Linux 内核中负责从监听队列（accept queue）中摘取一个已完成三次握手的连接，因为需要获取的数据

```cpp
//返回为struct sock*对象
//https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/inet_connection_sock.c#L427
struct sock *inet_csk_accept(struct sock *sk, int flags, int *err, bool kern)
```

tcpaccept的内核态实现核心是挂钩内核函数 -> 提取套接字信息 -> 异步传输到用户态，由于要提取的核心字段（五元组信息）在`inet_csk_accept`函数的返回值`struct sock *`中，所以需要使用`kretprobe`方式（`kretprobe` 在函数返回时触发，只有当函数返回时，才能拿到新创建的 `struct sock` 指针，从而获取连接的五元组）。内核主要提取的字段如下：

-   `PID/COMM`：触发 accept 的进程 ID 和进程名
-   五元组信息：源 IP (`skc_rcv_saddr`)、目的 IP (`skc_daddr`)、源端口 (`skc_num`) 和目的端口 (`skc_dport`)，从返回内核指针 `newsk`（即 `struct sock`）中提取
-   时间戳：使用 `bpf_ktime_get_ns()` 记录事件发生的精确时间

```PYTHON
# define BPF program
bpf_text = """
#include <uapi/linux/ptrace.h>
#include <net/sock.h>
#include <bcc/proto.h>

// separate data structs for ipv4 and ipv6
struct ipv4_data_t {
    u64 ts_us;
    u32 pid;
    u32 saddr;
    u32 daddr;
    u64 ip;
    u16 lport;
    u16 dport;
    char task[TASK_COMM_LEN];
};
BPF_PERF_OUTPUT(ipv4_events);

struct ipv6_data_t {
    u64 ts_us;
    u32 pid;
    unsigned __int128 saddr;
    unsigned __int128 daddr;
    u64 ip;
    u16 lport;
    u16 dport;
    char task[TASK_COMM_LEN];
};
BPF_PERF_OUTPUT(ipv6_events);
"""

#
# The following code uses kprobes to instrument inet_csk_accept().
# On Linux 4.16 and later, we could use sock:inet_sock_set_state
# tracepoint for efficiency, but it may output wrong PIDs. This is
# because sock:inet_sock_set_state may run outside of process context.
# Hence, we stick to kprobes until we find a proper solution.
#
bpf_text_kprobe = """
int kretprobe__inet_csk_accept(struct pt_regs *ctx)
{
    if (container_should_be_filtered()) {
        return 0;
    }

    //PT_REGS_RC(ctx)：获取 inet_csk_accept 函数的返回值，即新连接的 sock 指针
    struct sock *newsk = (struct sock *)PT_REGS_RC(ctx);
    u32 pid = bpf_get_current_pid_tgid() >> 32;

    ##FILTER_PID##

    if (newsk == NULL)
        return 0;

    // check this is TCP
    u16 protocol = 0;
    // workaround for reading the sk_protocol bitfield:

    // Following comments add by Joe Yin:
    // Unfortunately,it can not work since Linux 4.10,
    // because the sk_wmem_queued is not following the bitfield of sk_protocol.
    // And the following member is sk_gso_max_segs.
    // So, we can use this:
    // bpf_probe_read_kernel(&protocol, 1, (void *)((u64)&newsk->sk_gso_max_segs) - 3);
    // In order to  diff the pre-4.10 and 4.10+ ,introduce the variables gso_max_segs_offset,sk_lingertime,
    // sk_lingertime is closed to the gso_max_segs_offset,and
    // the offset between the two members is 4

    int gso_max_segs_offset = offsetof(struct sock, sk_gso_max_segs);
    int sk_lingertime_offset = offsetof(struct sock, sk_lingertime);


    // Since kernel v5.6 sk_protocol is its own u16 field and gso_max_segs
    // precedes sk_lingertime.
    if (sk_lingertime_offset - gso_max_segs_offset == 2)
        protocol = newsk->sk_protocol;
    else if (sk_lingertime_offset - gso_max_segs_offset == 4)
        // 4.10+ with little endian
#if __BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__
        protocol = *(u8 *)((u64)&newsk->sk_gso_max_segs - 3);
    else
        // pre-4.10 with little endian
        protocol = *(u8 *)((u64)&newsk->sk_wmem_queued - 3);
#elif __BYTE_ORDER__ == __ORDER_BIG_ENDIAN__
        // 4.10+ with big endian
        protocol = *(u8 *)((u64)&newsk->sk_gso_max_segs - 1);
    else
        // pre-4.10 with big endian
        protocol = *(u8 *)((u64)&newsk->sk_wmem_queued - 1);
#else
# error "Fix your compiler's __BYTE_ORDER__?!"
#endif

    if (protocol != IPPROTO_TCP)
        return 0;

    // pull in details
    u16 family = 0, lport = 0, dport;
    family = newsk->__sk_common.skc_family;
    lport = newsk->__sk_common.skc_num;
    dport = newsk->__sk_common.skc_dport;

    //dport：网络字节序转主机字节序（端口号转换），用于规则过滤
    dport = ntohs(dport);
    
    //占位符，由 Python 代码根据用户输入的参数（如 -p 1234）动态替换为过滤逻辑
    ##FILTER_FAMILY##

    ##FILTER_PORT##

    if (family == AF_INET) {
        struct ipv4_data_t data4 = {.pid = pid, .ip = 4};
        data4.ts_us = bpf_ktime_get_ns() / 1000;
        data4.saddr = newsk->__sk_common.skc_rcv_saddr;
        data4.daddr = newsk->__sk_common.skc_daddr;
        data4.lport = lport;
        data4.dport = dport;
        bpf_get_current_comm(&data4.task, sizeof(data4.task));
        ipv4_events.perf_submit(ctx, &data4, sizeof(data4));

    } else if (family == AF_INET6) {
        struct ipv6_data_t data6 = {.pid = pid, .ip = 6};
        data6.ts_us = bpf_ktime_get_ns() / 1000;
        bpf_probe_read_kernel(&data6.saddr, sizeof(data6.saddr),
            &newsk->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr32);
        bpf_probe_read_kernel(&data6.daddr, sizeof(data6.daddr),
            &newsk->__sk_common.skc_v6_daddr.in6_u.u6_addr32);
        data6.lport = lport;
        data6.dport = dport;
        bpf_get_current_comm(&data6.task, sizeof(data6.task));
        ipv6_events.perf_submit(ctx, &data6, sizeof(data6));
    }
    // else drop

    return 0;
}
"""
```

这段代码不复杂，比较玩味的是上面的注释部分：

1、注释一：为什么不用 tracepoint机制？注释中说明了问题所在，即`inet_sock_set_state` 跟踪的是套接字状态的变化。但在某些情况下（如内核软中断处理数据包时），状态转换发生的上下文可能并不是发起 `accept()` 系统调用的进程上下文，因此为了保证获取到的 PID 是真正调用 `accept` 的应用进程，代码选择 kprobe机制实现，因为它运行在进程系统调用的同步上下文中

2、注释二关于 `sk_protocol` 的长篇注释即说明了ebpf对内核版本兼容性（字段提取），代码中需要明确知道socket 类型是否为 TCP 协议（`IPPROTO_TCP`），但在内核的不同版本中，`struct sock` 结构体的定义一直在变化，即`sk_protocol` 字段的位置和存储方式发生了变化，典型的如下：

-   早期内核（4.10之前）：`sk_protocol` 被放在位域（bitfield）定义中，BPF 无法直接安全地读取位域
-   中期内核（4.10~5.5）：`sk_protocol` 的位置发生了偏移。代码通过计算 `sk_gso_max_segs` 和 `sk_lingertime` 两个邻近字段的相对距离，来推断当前内核的版本，从而计算出 `sk_protocol` 的物理偏移量
-   较新内核 （5.6之后）：内核开发者将 `sk_protocol` 独立成了一个 `u16` 字段，可以直接读取

##  0x0D tcp-retrans：重传的 TCP 连接跟踪
`tcpretrans` 工具显示有关 TCP 重新传输的相关信息，如本地和远程的 IP 地址和端口号，以及重新传输时 TCP 的状态。每次TCP重传数据包时，`tcpretrans`会打印一行记录，包含源地址和目的地址，以及当时该 TCP 连接所处的内核状态，TCP 重传会导致延迟和吞吐量方面的问题，如果重传发生在`ESTABLISHED`状态下，会进一步寻找外部网络可能存在的问题。若重传发在`SYNSENT`状态下，这可能是 CPU 饱和的一个征兆，也可能是内核丢包引发的

输出`T`列的含义：

-   `L`：尝试，内核可能发送了一个TLP，但在某些情况下它可能最终没有被发送
-   `L>`：表示数据包是从本地地址LADDR发送到远程地址RADDR
-   `R>`：表示数据包是从远程地址RADDR发送到本地地址LADDR

```bash
Tracing retransmits ... Hit Ctrl-C to end
TIME     PID     IP LADDR:LPORT          T> RADDR:RPORT          STATE
12:35:36 107232  4  192.168.26.100:39122 R> 192.168.26.100:3306  FIN_WAIT1（本地已经收到来自远端的 FIN 包，但本地应用还未完全关闭连接）
12:35:36 0       4  192.168.26.99:3306   R> 192.168.26.101:57360 ESTABLISHED
12:35:55 0       4  192.168.26.99:48370  R> 192.168.26.99:3306   FIN_WAIT1
01:55:53 0      4  10.153.223.157:22    L> 69.53.245.40:46444   ESTABLISHED


# python3 tcpretrans.py -c
# 要快速发现重传流，可以使用-c标志。它将计算每个流中发生的重传次数
Tracing retransmits ... Hit Ctrl-C to end
^C
LADDR:LPORT              RADDR:RPORT             RETRANSMITS
192.168.10.50:60366  <-> 172.217.21.194:443         700
192.168.10.50:666    <-> 172.213.11.195:443         345
192.168.10.50:366    <-> 172.212.22.194:443         211
```

####    内核态实现
核心基于`tracepoint:tcp:tcp_retransmit_skb/kprobe:tcp_send_loss_probe`实现，本质上还是基于单hook的事件记录

```PYTHON
# define BPF program
bpf_text = """
#include <uapi/linux/ptrace.h>
#include <net/sock.h>
#include <net/tcp.h>
#include <bcc/proto.h>

#define RETRANSMIT  1
#define TLP         2

// separate data structs for ipv4 and ipv6
struct ipv4_data_t {
    u32 pid;
    u64 ip;
    u32 seq;
    u32 saddr;
    u32 daddr;
    u16 lport;
    u16 dport;
    u64 state;
    u64 type;
};
BPF_PERF_OUTPUT(ipv4_events);

struct ipv6_data_t {
    u32 pid;
    u32 seq;
    u64 ip;
    unsigned __int128 saddr;
    unsigned __int128 daddr;
    u16 lport;
    u16 dport;
    u64 state;
    u64 type;
};
BPF_PERF_OUTPUT(ipv6_events);

// separate flow keys per address family
struct ipv4_flow_key_t {
    u32 saddr;
    u32 daddr;
    u16 lport;
    u16 dport;
};
BPF_HASH(ipv4_count, struct ipv4_flow_key_t);

struct ipv6_flow_key_t {
    unsigned __int128 saddr;
    unsigned __int128 daddr;
    u16 lport;
    u16 dport;
};
BPF_HASH(ipv6_count, struct ipv6_flow_key_t);
"""

# 核心trace_event
bpf_text_kprobe = """
static int trace_event(struct pt_regs *ctx, struct sock *skp, struct sk_buff *skb, int type)
{
    struct tcp_skb_cb *tcb;
    u32 seq;

    if (skp == NULL)
        return 0;
    u32 pid = bpf_get_current_pid_tgid() >> 32;

    // pull in details
    u16 family = skp->__sk_common.skc_family;
    u16 lport = skp->__sk_common.skc_num;
    u16 dport = skp->__sk_common.skc_dport;
    char state = skp->__sk_common.skc_state;

    seq = 0;
    if (skb) {
        /* macro TCP_SKB_CB from net/tcp.h */
        tcb = ((struct tcp_skb_cb *)&((skb)->cb[0]));
        seq = tcb->seq;
    }

    FILTER_FAMILY

    if (family == AF_INET) {
        IPV4_INIT
        IPV4_CORE
    } else if (family == AF_INET6) {
        IPV6_INIT
        IPV6_CORE
    }
    // else drop

    return 0;
}
"""

#trace_retransmit：入口
bpf_text_kprobe_retransmit = """
int trace_retransmit(struct pt_regs *ctx, struct sock *sk, struct sk_buff *skb)
{
    trace_event(ctx, sk, skb, RETRANSMIT);
    return 0;
}
"""

#trace_tlp入口
bpf_text_kprobe_tlp = """
int trace_tlp(struct pt_regs *ctx, struct sock *sk)
{
    trace_event(ctx, sk, NULL, TLP);
    return 0;
}
"""

bpf_text_tracepoint = """
TRACEPOINT_PROBE(tcp, tcp_retransmit_skb)
{
    struct tcp_skb_cb *tcb;
    u32 seq;

    u32 pid = bpf_get_current_pid_tgid() >> 32;
    const struct sock *skp = (const struct sock *)args->skaddr;
    const struct sk_buff *skb = (const struct sk_buff *)args->skbaddr;
    u16 lport = args->sport;
    u16 dport = args->dport;
    char state = skp->__sk_common.skc_state;
    u16 family = skp->__sk_common.skc_family;

    seq = 0;
    if (skb) {
        /* macro TCP_SKB_CB from net/tcp.h */
        tcb = ((struct tcp_skb_cb *)&((skb)->cb[0]));
        seq = tcb->seq;
    }

    FILTER_FAMILY

    if (family == AF_INET) {
        IPV4_CODE
    } else if (family == AF_INET6) {
        IPV6_CODE
    }
    return 0;
}
"""

struct_init = { 'ipv4':
        { 'count' :
            """
               struct ipv4_flow_key_t flow_key = {};
               flow_key.saddr = skp->__sk_common.skc_rcv_saddr;
               flow_key.daddr = skp->__sk_common.skc_daddr;
               // lport is host order
               flow_key.lport = lport;
               flow_key.dport = ntohs(dport);""",
               'trace' :
               """
               struct ipv4_data_t data4 = {};
               data4.pid = pid;
               data4.ip = 4;
               data4.seq = seq;
               data4.type = type;
               data4.saddr = skp->__sk_common.skc_rcv_saddr;
               data4.daddr = skp->__sk_common.skc_daddr;
               // lport is host order
               data4.lport = lport;
               data4.dport = ntohs(dport);
               data4.state = state; """
               },
        'ipv6':
        { 'count' :
            """
                    struct ipv6_flow_key_t flow_key = {};
                    bpf_probe_read_kernel(&flow_key.saddr, sizeof(flow_key.saddr),
                        skp->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr32);
                    bpf_probe_read_kernel(&flow_key.daddr, sizeof(flow_key.daddr),
                        skp->__sk_common.skc_v6_daddr.in6_u.u6_addr32);
                    // lport is host order
                    flow_key.lport = lport;
                    flow_key.dport = ntohs(dport);""",
          'trace' : """
                    struct ipv6_data_t data6 = {};
                    data6.pid = pid;
                    data6.ip = 6;
                    data6.seq = seq;
                    data6.type = type;
                    bpf_probe_read_kernel(&data6.saddr, sizeof(data6.saddr),
                        skp->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr32);
                    bpf_probe_read_kernel(&data6.daddr, sizeof(data6.daddr),
                        skp->__sk_common.skc_v6_daddr.in6_u.u6_addr32);
                    // lport is host order
                    data6.lport = lport;
                    data6.dport = ntohs(dport);
                    data6.state = state;"""
                }
        }

struct_init_tracepoint = { 'ipv4':
        { 'count' : """
               struct ipv4_flow_key_t flow_key = {};
               __builtin_memcpy(&flow_key.saddr, args->saddr, sizeof(flow_key.saddr));
               __builtin_memcpy(&flow_key.daddr, args->daddr, sizeof(flow_key.daddr));
               flow_key.lport = lport;
               flow_key.dport = dport;
               ipv4_count.increment(flow_key);
               """,
          'trace' : """
               struct ipv4_data_t data4 = {};
               data4.pid = pid;
               data4.lport = lport;
               data4.dport = dport;
               data4.type = RETRANSMIT;
               data4.ip = 4;
               data4.seq = seq;
               data4.state = state;
               __builtin_memcpy(&data4.saddr, args->saddr, sizeof(data4.saddr));
               __builtin_memcpy(&data4.daddr, args->daddr, sizeof(data4.daddr));
               ipv4_events.perf_submit(args, &data4, sizeof(data4));
               """
               },
        'ipv6':
        { 'count' : """
               struct ipv6_flow_key_t flow_key = {};
               __builtin_memcpy(&flow_key.saddr, args->saddr_v6, sizeof(flow_key.saddr));
               __builtin_memcpy(&flow_key.daddr, args->daddr_v6, sizeof(flow_key.daddr));
               flow_key.lport = lport;
               flow_key.dport = dport;
               ipv6_count.increment(flow_key);
               """,
          'trace' : """
               struct ipv6_data_t data6 = {};
               data6.pid = pid;
               data6.lport = lport;
               data6.dport = dport;
               data6.type = RETRANSMIT;
               data6.ip = 6;
               data6.seq = seq;
               data6.state = state;
               __builtin_memcpy(&data6.saddr, args->saddr_v6, sizeof(data6.saddr));
               __builtin_memcpy(&data6.daddr, args->daddr_v6, sizeof(data6.daddr));
               ipv6_events.perf_submit(args, &data6, sizeof(data6));
               """
               }
        }

count_core_base = """
        COUNT_STRUCT.increment(flow_key);
"""

# 代码文本替换
if BPF.tracepoint_exists("tcp", "tcp_retransmit_skb"):
    if args.count:
        bpf_text_tracepoint = bpf_text_tracepoint.replace("IPV4_CODE", struct_init_tracepoint['ipv4']['count'])
        bpf_text_tracepoint = bpf_text_tracepoint.replace("IPV6_CODE", struct_init_tracepoint['ipv6']['count'])
    else:
        bpf_text_tracepoint = bpf_text_tracepoint.replace("IPV4_CODE", struct_init_tracepoint['ipv4']['trace'])
        bpf_text_tracepoint = bpf_text_tracepoint.replace("IPV6_CODE", struct_init_tracepoint['ipv6']['trace'])
    bpf_text += bpf_text_tracepoint

if args.lossprobe or not BPF.tracepoint_exists("tcp", "tcp_retransmit_skb"):
    bpf_text += bpf_text_kprobe
    if args.count:
        bpf_text = bpf_text.replace("IPV4_INIT", struct_init['ipv4']['count'])
        bpf_text = bpf_text.replace("IPV6_INIT", struct_init['ipv6']['count'])
        bpf_text = bpf_text.replace("IPV4_CORE", count_core_base.replace("COUNT_STRUCT", 'ipv4_count'))
        bpf_text = bpf_text.replace("IPV6_CORE", count_core_base.replace("COUNT_STRUCT", 'ipv6_count'))
    else:
        bpf_text = bpf_text.replace("IPV4_INIT", struct_init['ipv4']['trace'])
        bpf_text = bpf_text.replace("IPV6_INIT", struct_init['ipv6']['trace'])
        bpf_text = bpf_text.replace("IPV4_CORE", "ipv4_events.perf_submit(ctx, &data4, sizeof(data4));")
        bpf_text = bpf_text.replace("IPV6_CORE", "ipv6_events.perf_submit(ctx, &data6, sizeof(data6));")
    if args.lossprobe:
        bpf_text += bpf_text_kprobe_tlp
    if not BPF.tracepoint_exists("tcp", "tcp_retransmit_skb"):
        bpf_text += bpf_text_kprobe_retransmit
if args.ipv4:
    bpf_text = bpf_text.replace('FILTER_FAMILY',
        'if (family != AF_INET) { return 0; }')
elif args.ipv6:
    bpf_text = bpf_text.replace('FILTER_FAMILY',
        'if (family != AF_INET6) { return 0; }')
else:
    bpf_text = bpf_text.replace('FILTER_FAMILY', '')
if debug or args.ebpf:
    print(bpf_text)
    if args.ebpf:
        exit()

# initialize BPF
b = BPF(text=bpf_text)
if not BPF.tracepoint_exists("tcp", "tcp_retransmit_skb"):
    b.attach_kprobe(event="tcp_retransmit_skb", fn_name="trace_retransmit")
if args.lossprobe:
    b.attach_kprobe(event="tcp_send_loss_probe", fn_name="trace_tlp")
```

##  0x0E tcpsubnet

##  0x0F 一个关于ip_local_port_range的观测问题
这个问题来自司内分享，在Linux内核版本过高（如`4.14.x`）的场景下，监控本地端口的占用情况，如果此值持续过高（如处于`EST`的状态超过3W）且客户端主动connect并发较高的情况下可能会出现CPU高负载，且负载主要来自于`sys`占用。那么在这种场景下，使用`netstate`/`ss`命令定期统计当前`EST`的端口占用总数就不太合适。如何利用ebpf的方式解决？


##  0x10  参考
-   [eBPF入门实践教程十四：记录 TCP 连接状态与 TCP RTT](https://eunomia.dev/zh/tutorials/14-tcpstates/)
-   [TCP Tracepoints](https://www.brendangregg.com/blog/2018-03-22/tcp-tracepoints.html)
-   [eBPF入门开发实践教程十三：统计 TCP 连接延时，并使用 libbpf 在用户态处理数据](https://eunomia.dev/zh/tutorials/13-tcpconnlat/)
-   [tcpconnect - TCP Connection Monitoring](https://ebee.xmigrate.cloud/tools/tcpconnect/)
-   [redhat](https://docs.redhat.com/zh_hans/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_networking/network-tracing-using-the-bpf-compiler-collection_configuring-and-managing-networking
)