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

##  0x04    tcpconnect：主动 TCP 连接跟踪
[`tcpconnect`](https://github.com/iovisor/bcc/blob/v0.35.0/libbpf-tools/tcpconnect.bpf.c)

```bash
PID    COMM             IP SADDR            DADDR            DPORT
1055869 xxx      4  9.1.2.3    9.1.2.4    8443 
1789533 xxx1  4  9.1.2.3    9.1.2.4    8443 
```

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
   TCP 被动连接（即 accept 成功的连接），基于hook点为`kretprobe/inet_csk_accept`，函数原型如下。该函数`inet_csk_accept`的主要功能是Linux 内核中负责从监听队列（accept queue）中摘取一个已完成三次握手的连接，因为需要获取的数据

```cpp
//返回为struct sock*对象
//https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/inet_connection_sock.c#L427
struct sock *inet_csk_accept(struct sock *sk, int flags, int *err, bool kern)
```

tcpaccept的内核态实现核心是挂钩内核函数 -> 提取套接字信息 -> 异步传输到用户态，由于要提取的核心字段（五元组信息）在`inet_csk_accept`函数的返回值`struct sock *`中 a啊 ，所以需要使用`kretprobe`方式（`kretprobe` 在函数返回时触发，只有当函数返回时，才能拿到新创建的 `struct sock` 指针，从而获取连接的五元组）。内核主要提取的字段如下：

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