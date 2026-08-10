---
layout:     post
title:  Linux 内核之旅（二十六）：TCP 发送路径上的内核实现
subtitle:   拥塞控制、滑动窗口、ACK 处理与 skb 生命周期
date:       2026-04-11
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
    - TCP
---

##  0x00    前言

本文代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source) 版本，主要涉及到如下知识点：

1.  `accept` 完成后 `tcp_sock` 中与发送相关字段的初始化状态
2.  `tcp_sendmsg` → `tcp_write_xmit` → `tcp_transmit_skb` 发送链路的字段级分析
3.  拥塞控制在发送时的门控作用（哪些字段影响发送决策）
4.  `tcp_ack` 收到 ACK 时的完整处理：窗口更新、skb 释放、拥塞窗口调整
5.  滑动窗口在发送侧和接收侧的判断逻辑与更新机制
6.  ACK 与滑动窗口的综合分析：`tcp_ack` 的双端调用视角、ACK 驱动 `tcp_write_xmit` 的完整链路、纯 ACK 与带数据 ACK 的场景对比、Delayed ACK 完整机制
7.  滑动窗口变量的主动 / 被动修改分类总结
8.  Nagle 算法与发送时机、TCP 内存管理与背压（TSQ）机制

##  0x01    TCP 发送路径全景图

下图展示了 TCP 数据从 `accept` 获取连接 fd 到 ACK 驱动 skb 释放的完整生命周期：

```mermaid
flowchart TD
    subgraph userspace ["用户空间"]
        ACC["accept(listen_fd) → conn_fd"]
        SEND["send/write(conn_fd, buf, len)"]
    end

    subgraph syscall ["系统调用层"]
        SENDTO["SYSCALL: sendto / sendmsg"]
        SOCKSEND["sock_sendmsg → inet_sendmsg"]
    end

    subgraph transport ["传输层 - TCP 特有"]
        TCPSEND["tcp_sendmsg<br/>用户数据 → sk_write_queue"]
        PUSH["tcp_push → __tcp_push_pending_frames"]
        XMIT["tcp_write_xmit<br/>发送决策核心"]
        CWND{"tcp_cwnd_test<br/>拥塞窗口检查"}
        SWND{"tcp_snd_wnd_test<br/>发送窗口检查"}
        NAGLE{"tcp_nagle_test<br/>Nagle算法"}
        TRANSMIT["tcp_transmit_skb<br/>clone skb + 构造TCP头"]
    end

    subgraph network ["网络层"]
        IPQUEUE["ip_queue_xmit<br/>TCP专用IP层入口"]
        IPOUT["ip_local_out → ip_output<br/>→ ip_finish_output"]
    end

    subgraph lower ["设备层（与UDP共享）"]
        DEVXMIT["dev_queue_xmit → 驱动发送"]
        HW["网卡 DMA 发送"]
    end

    subgraph ack_path ["ACK 接收路径"]
        HWIRQ["网卡中断 → NAPI poll"]
        TCPRCV["tcp_v4_rcv → tcp_rcv_established"]
        TCPACK["tcp_ack<br/>ACK处理核心"]
        WNDUPD["tcp_may_update_window<br/>更新 snd_wnd"]
        CLEAN["tcp_clean_rtx_queue<br/>释放已确认的skb"]
        CONGAVOID["tcp_cong_avoid<br/>更新 snd_cwnd"]
        REARM["tcp_rearm_rto<br/>重置重传定时器"]
    end

    ACC --> SEND
    SEND --> SENDTO --> SOCKSEND --> TCPSEND
    TCPSEND --> PUSH --> XMIT
    XMIT --> CWND
    CWND -->|"通过"| SWND
    CWND -->|"拥塞窗口满"| WAIT1["等待ACK释放窗口"]
    SWND -->|"通过"| NAGLE
    SWND -->|"发送窗口满"| WAIT1
    NAGLE -->|"允许发送"| TRANSMIT
    NAGLE -->|"Nagle延迟"| WAIT2["等待更多数据或ACK"]
    TRANSMIT --> IPQUEUE --> IPOUT --> DEVXMIT --> HW

    HW -.->|"对端回复ACK"| HWIRQ
    HWIRQ --> TCPRCV --> TCPACK
    TCPACK --> WNDUPD
    TCPACK --> CLEAN
    TCPACK --> CONGAVOID
    TCPACK --> REARM
    CLEAN -.->|"snd_una推进<br/>释放sk_write_queue中skb"| XMIT
    CONGAVOID -.->|"cwnd增大"| XMIT
    WNDUPD -.->|"snd_wnd更新"| XMIT
```

核心要点：
-   **发送路径**：`tcp_sendmsg` 只负责将用户数据拷贝到 `sk_write_queue`，真正的发送决策在 `tcp_write_xmit` 中完成，受拥塞窗口（`snd_cwnd`）、发送窗口（`snd_wnd`）和 Nagle 算法三重门控
-   **ACK 驱动**：TCP 发送是 ACK 驱动的闭环，`tcp_ack` 收到确认后推进 `snd_una`、释放 skb、更新拥塞窗口，从而允许 `tcp_write_xmit` 发送更多数据
-   **skb 生命周期**：skb 在 `tcp_sendmsg` 中创建并加入 `sk_write_queue`，`tcp_transmit_skb` 发送时 clone 一份（原始 skb 保留用于可能的重传），直到 `tcp_clean_rtx_queue` 收到 ACK 后才释放原始 skb

##  0x02    accept 完成后的 tcp_sock 状态布局

TCP 服务端通过 `accept` 获取客户端连接的 fd，此时三次握手已经完成，`tcp_sock` 中的各项字段已经初始化完毕。理解这些字段的初始状态，是分析后续发送路径的基础

####    accept 系统调用全流程

[`accept4`](https://elixir.bootlin.com/linux/v4.11.6/source/net/socket.c#L1470) 系统调用的核心流程：

```mermaid
flowchart TD
    A["SYSCALL: accept4(listen_fd, addr, addrlen, flags)"] --> B["sockfd_lookup_light(fd)<br/>定位listen socket"]
    B --> C["sock_alloc()<br/>分配新的 struct socket"]
    C --> D["newsock->ops = sock->ops<br/>复制协议操作集"]
    D --> E["sock_alloc_file(newsock)<br/>分配 struct file"]
    E --> F["sock->ops->accept()<br/>即 inet_accept"]
    F --> G["sk->sk_prot->accept()<br/>即 inet_csk_accept"]
    G --> H{"全连接队列为空?"}
    H -->|"是"| I["inet_csk_wait_for_connect<br/>阻塞等待"]
    H -->|"否"| J["reqsk_queue_remove<br/>从 icsk_accept_queue 取出"]
    I --> J
    J --> K["sock_graft(sk2, newsock)<br/>将新sock关联到新socket"]
    K --> L["fd_install(newfd, newfile)<br/>挂入进程fd表"]
    L --> M["返回 newfd 给用户空间"]
```

关键源码分析：

```cpp
// net/socket.c
SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr,
        int __user *, upeer_addrlen, int, flags)
{
    struct socket *sock, *newsock;
    struct file *newfile;
    int err, len, newfd, fput_needed;

    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (!sock)
        goto out;

    newsock = sock_alloc();
    newsock->type = sock->type;
    newsock->ops = sock->ops;   // 复制 inet_stream_ops

    newfile = sock_alloc_file(newsock, flags, sock->sk->sk_prot_creator->name);

    // inet_accept → inet_csk_accept：从全连接队列取出已完成三次握手的 sock
    err = sock->ops->accept(sock, newsock, sock->file->f_flags, false);

    fd_install(newfd, newfile);
    /* ...... */
}
```

[`inet_csk_accept`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/inet_connection_sock.c#L427) 从全连接队列中取出在三次握手过程中已经创建好的 `struct sock`：

```cpp
// net/ipv4/inet_connection_sock.c
struct sock *inet_csk_accept(struct sock *sk, int flags, int *err, bool kern)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct request_sock_queue *queue = &icsk->icsk_accept_queue;
    struct request_sock *req;
    struct sock *newsk;
    int error;

    lock_sock(sk);

    if (reqsk_queue_empty(queue)) {
        long timeo = sock_rcvtimeo(sk, flags & O_NONBLOCK);
        error = inet_csk_wait_for_connect(sk, timeo);
        if (error)
            goto out_err;
    }
    req = reqsk_queue_remove(queue, sk);
    newsk = req->sk;       // 三次握手期间已创建的 tcp_sock

    /* ...... */
    release_sock(sk);
    return newsk;
}
```

####    accept 完成后 tcp_sock 关键字段

三次握手完成时，`tcp_v4_syn_recv_sock` → `tcp_create_openreq_child` 已经初始化了新 `tcp_sock` 的关键字段。accept 返回后，该 `tcp_sock` 处于 `TCP_ESTABLISHED` 状态，各发送相关字段的初始值：

```mermaid
classDiagram
    class tcp_sock_send_fields ["tcp_sock 发送相关字段（accept后）"] {
        **序列号控制**
        u32 snd_una = ISS+1 : 最早未确认序列号
        u32 snd_nxt = ISS+1 : 下一个待发送序列号
        u32 write_seq = ISS+1 : 用户数据写入序列号
        **拥塞控制**
        u32 snd_cwnd = TCP_INIT_CWND : 拥塞窗口(通常=10)
        u32 snd_ssthresh = TCP_INFINITE_SSTHRESH : 慢启动阈值(0x7fffffff)
        u32 snd_cwnd_cnt = 0 : 拥塞避免计数器
        u32 snd_cwnd_clamp = ~0 : cwnd上限
        **流量控制**
        u32 snd_wnd : 对端通告的接收窗口
        u32 rcv_wnd : 本端接收窗口
        u32 rcv_nxt : 期望收到的下一个序列号
        **MSS与分段**
        u16 mss_cache : 缓存的MSS值
        u16 advmss : 通告的MSS
        **发送队列**
        sk_buff_head sk_write_queue : 发送队列(空)
        sk_buff* sk_send_head = NULL : 待发送指针(空)
        int sk_wmem_queued = 0 : 发送队列占用内存
        int sk_sndbuf : 发送缓冲区上限
        **在途统计**
        u32 packets_out = 0 : 在途数据包数
        u32 retrans_out = 0 : 重传中的包数
        u32 sacked_out = 0 : SACK确认的包数
        u32 lost_out = 0 : 标记丢失的包数
    }
```

关键字段的初始化位置（`tcp_create_openreq_child` / `tcp_init_xmit_timers` / `tcp_v4_syn_recv_sock`）：

| 字段 | 初始值 | 初始化函数 | 说明 |
|------|--------|-----------|------|
| `snd_una` | `ISS + 1` | `tcp_create_openreq_child` | SYN 占一个序列号，握手完成后指向 SYN 的下一个 |
| `snd_nxt` | `ISS + 1` | `tcp_create_openreq_child` | 与 `snd_una` 相同，尚未发送数据 |
| `write_seq` | `ISS + 1` | `tcp_create_openreq_child` | 用户数据写入位置 |
| `snd_cwnd` | `TCP_INIT_CWND`(10) | `tcp_init_metrics` | [RFC 6928](https://tools.ietf.org/html/rfc6928) 推荐初始窗口 |
| `snd_ssthresh` | `0x7fffffff` | `tcp_create_openreq_child` | 初始为无穷大，首次丢包前一直处于慢启动 |
| `snd_wnd` | SYN-ACK中的窗口值 | `tcp_ack`（握手阶段） | 对端在 SYN-ACK 中通告的接收窗口 |
| `mss_cache` | 对端MSS或默认536 | `tcp_sync_mss` | 由对端 SYN 中的 MSS 选项决定 |
| `sk_send_head` | `NULL` | `tcp_create_openreq_child` | 尚无待发送数据 |
| `packets_out` | `0` | `tcp_create_openreq_child` | 握手完成后 SYN 已被确认 |

```cpp
// net/ipv4/tcp_minisocks.c - tcp_create_openreq_child（精简）
struct sock *tcp_create_openreq_child(const struct sock *sk,
                      struct request_sock *req,
                      struct sk_buff *skb)
{
    struct sock *newsk = inet_csk_clone_lock(sk, req, GFP_ATOMIC);
    struct tcp_sock *newtp = tcp_sk(newsk);
    struct tcp_sock *oldtp = tcp_sk(sk);

    newtp->snd_una     = treq->snt_isn + 1;
    newtp->snd_nxt     = treq->snt_isn + 1;
    newtp->write_seq   = treq->snt_isn + 1;
    newtp->rcv_nxt     = treq->rcv_isn + 1;
    newtp->snd_sml     = treq->snt_isn + 1;

    tcp_init_wl(newtp, treq->rcv_isn);

    newtp->snd_ssthresh = TCP_INFINITE_SSTHRESH;

    newtp->packets_out = 0;
    newtp->retrans_out = 0;
    newtp->sacked_out  = 0;
    newtp->lost_out    = 0;

    newtp->snd_cwnd = TCP_INIT_CWND;
    newtp->snd_cwnd_cnt = 0;

    /* ...... */
    return newsk;
}
```

此刻发送队列 `sk_write_queue` 为空，`sk_send_head` 为 `NULL`。用户通过 `send`/`write` 写入数据后，数据才会进入发送队列

##  0x03    tcp_sendmsg：用户数据到发送队列

`tcp_sendmsg` 的核心职责是将用户空间数据拷贝到内核的 `sk_write_queue` 中，但并不直接决定何时发送。发送决策由后续的 `tcp_push` → `tcp_write_xmit` 完成

####    sk_write_queue 与 sk_send_head 的关系

```mermaid
sequenceDiagram
    participant App as 用户空间
    participant SM as tcp_sendmsg
    participant WQ as sk_write_queue
    participant SH as sk_send_head
    participant WX as tcp_write_xmit
    participant TS as tcp_transmit_skb
    participant ACK as tcp_ack

    App->>SM: send(fd, buf, 4096)
    SM->>SM: sk_stream_alloc_skb 分配 skb1
    SM->>WQ: skb_entail(skb1)<br/>加入队列尾部
    Note over SH: sk_send_head → skb1<br/>(首次入队时设置)
    SM->>SM: skb_add_data_nocache<br/>拷贝用户数据到 skb1

    App->>SM: send(fd, buf, 4096)
    SM->>SM: 分配 skb2
    SM->>WQ: skb_entail(skb2)
    Note over WQ: 队列: skb1 → skb2
    Note over SH: sk_send_head 仍指向 skb1

    SM->>WX: tcp_push → tcp_write_xmit
    WX->>WX: tcp_send_head(sk) 返回 skb1
    WX->>TS: tcp_transmit_skb(skb1, clone=1)
    TS->>TS: skb_clone(skb1) 发送克隆体
    WX->>WX: tcp_event_new_data_sent(skb1)
    Note over SH: sk_send_head → skb2<br/>(推进到下一个待发送)

    WX->>TS: tcp_transmit_skb(skb2, clone=1)
    WX->>WX: tcp_event_new_data_sent(skb2)
    Note over SH: sk_send_head = NULL<br/>(队列中无更多待发送)

    ACK->>ACK: 收到ACK确认skb1
    ACK->>WQ: tcp_clean_rtx_queue<br/>sk_wmem_free_skb(skb1)
    Note over WQ: 队列: skb2

    ACK->>ACK: 收到ACK确认skb2
    ACK->>WQ: tcp_clean_rtx_queue<br/>sk_wmem_free_skb(skb2)
    Note over WQ: 队列: 空
```

关键机制说明：
-   `sk_write_queue` 是双向链表，保存所有已入队但尚未被 ACK 确认的 skb
-   `sk_send_head` 指向队列中第一个尚未发送（未调用 `tcp_transmit_skb`）的 skb
-   `sk_send_head` 之前的 skb：已发送但等待 ACK 确认（保留用于可能的重传）
-   `sk_send_head` 之后（含）的 skb：尚未发送

####    tcp_sendmsg 核心源码分析

[`tcp_sendmsg`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp.c#L1148) 的完整逻辑：

```cpp
// net/ipv4/tcp.c
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb;
    int flags, err, copied = 0;
    int mss_now = 0, size_goal, copied_syn = 0;
    bool sg;
    long timeo;

    lock_sock(sk);

    flags = msg->msg_flags;
    // TCP Fast Open：在 SYN 中携带数据
    if (unlikely(flags & MSG_FASTOPEN || inet_sk(sk)->defer_connect)) {
        err = tcp_sendmsg_fastopen(sk, msg, &copied_syn, size);
        if (err == -EINPROGRESS && copied_syn > 0)
            goto out;
        else if (err)
            goto out_err;
    }

    timeo = sock_sndtimeo(sk, flags & MSG_DONTWAIT);
    tcp_rate_check_app_limited(sk);

    // 只有 ESTABLISHED 和 CLOSE_WAIT 状态允许发送数据
    if (((1 << sk->sk_state) & ~(TCPF_ESTABLISHED | TCPF_CLOSE_WAIT)) &&
        !tcp_passive_fastopen(sk)) {
        err = sk_stream_wait_connect(sk, &timeo);
        if (err != 0)
            goto do_error;
    }

    /* ...... repair 模式处理 ...... */

    copied = 0;

restart:
    // 获取当前 MSS 和 size_goal
    // size_goal: 支持 GSO 时为 MSS 的整数倍，否则等于 MSS
    mss_now = tcp_send_mss(sk, &size_goal, flags);

    sg = !!(sk->sk_route_caps & NETIF_F_SG);

    while (msg_data_left(msg)) {
        int copy = 0;
        int max = size_goal;

        skb = tcp_write_queue_tail(sk);
        if (tcp_send_head(sk)) {
            if (skb->ip_summed == CHECKSUM_NONE)
                max = mss_now;
            // 检查当前 skb 还能容纳多少数据
            copy = max - skb->len;
        }

        if (copy <= 0 || !tcp_skb_can_collapse_to(skb)) {
new_segment:
            // 检查发送缓冲区是否还有空间
            // sk_wmem_queued >= sk_sndbuf 时返回 false
            if (!sk_stream_memory_free(sk))
                goto wait_for_sndbuf;

            if (process_backlog && sk_flush_backlog(sk)) {
                process_backlog = false;
                goto restart;
            }

            // 分配新的 skb
            skb = sk_stream_alloc_skb(sk,
                          select_size(sk, sg, first_skb),
                          sk->sk_allocation, first_skb);
            if (!skb)
                goto wait_for_memory;

            if (sk_check_csum_caps(sk))
                skb->ip_summed = CHECKSUM_PARTIAL;

            // 将 skb 加入 sk_write_queue 尾部
            // 如果 sk_send_head 为 NULL，同时设置 sk_send_head 指向此 skb
            skb_entail(sk, skb);
            copy = size_goal;
            max = size_goal;
        }

        if (copy > msg_data_left(msg))
            copy = msg_data_left(msg);

        // 数据拷贝：线性区 or 分页区
        if (skb_availroom(skb) > 0) {
            copy = min_t(int, copy, skb_availroom(skb));
            err = skb_add_data_nocache(sk, skb, &msg->msg_iter, copy);
            if (err)
                goto do_fault;
        } else {
            // 线性区已满，使用分页区（skb_shared_info->frags）
            bool merge = true;
            int i = skb_shinfo(skb)->nr_frags;
            struct page_frag *pfrag = sk_page_frag(sk);

            if (!sk_page_frag_refill(sk, pfrag))
                goto wait_for_memory;

            if (!skb_can_coalesce(skb, i, pfrag->page, pfrag->offset)) {
                if (i >= sysctl_max_skb_frags || !sg) {
                    tcp_mark_push(tp, skb);
                    goto new_segment;
                }
                merge = false;
            }

            copy = min_t(int, copy, pfrag->size - pfrag->offset);
            if (!sk_wmem_schedule(sk, copy))
                goto wait_for_memory;

            err = skb_copy_to_page_nocache(sk, &msg->msg_iter, skb,
                               pfrag->page, pfrag->offset, copy);
            if (err)
                goto do_error;

            if (merge) {
                skb_frag_size_add(&skb_shinfo(skb)->frags[i - 1], copy);
            } else {
                skb_fill_page_desc(skb, i, pfrag->page,
                           pfrag->offset, copy);
                page_ref_inc(pfrag->page);
            }
            pfrag->offset += copy;
        }

        if (!copied)
            TCP_SKB_CB(skb)->tcp_flags &= ~TCPHDR_PSH;

        // 更新序列号
        tp->write_seq += copy;
        TCP_SKB_CB(skb)->end_seq += copy;
        tcp_skb_pcount_set(skb, 0);

        copied += copy;
        if (!msg_data_left(msg)) {
            if (unlikely(flags & MSG_EOR))
                TCP_SKB_CB(skb)->eor = 1;
            goto out;
        }

        // skb 未满且不是 OOB，继续填充
        if (skb->len < max || (flags & MSG_OOB) || unlikely(tp->repair))
            continue;

        // 检查是否需要立即推送
        if (forced_push(tp)) {
            tcp_mark_push(tp, skb);
            __tcp_push_pending_frames(sk, mss_now, TCP_NAGLE_PUSH);
        } else if (skb == tcp_send_head(sk))
            tcp_push_one(sk, mss_now);

        continue;

wait_for_sndbuf:
        set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);
wait_for_memory:
        if (copied)
            tcp_push(sk, flags & ~MSG_MORE, mss_now,
                 TCP_NAGLE_PUSH, size_goal);
        // 阻塞等待发送缓冲区释放
        err = sk_stream_wait_memory(sk, &timeo);
        if (err != 0)
            goto do_error;
        mss_now = tcp_send_mss(sk, &size_goal, flags);
    }

out:
    if (copied) {
        tcp_tx_timestamp(sk, sockc.tsflags, tcp_write_queue_tail(sk));
        // tcp_push 决定是否立即发送，受 Nagle 算法影响
        tcp_push(sk, flags, mss_now, tp->nonagle, size_goal);
    }
out_nopush:
    release_sock(sk);
    return copied + copied_syn;

    /* ...... 错误处理 ...... */
}
```

####    关键辅助函数

`skb_entail` 将 skb 加入发送队列并更新内存计数：

```cpp
// net/ipv4/tcp.c
static void skb_entail(struct sock *sk, struct sk_buff *skb)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct tcp_skb_cb *tcb = TCP_SKB_CB(skb);

    skb->csum    = 0;
    tcb->seq     = tcb->end_seq = tp->write_seq;
    tcb->tcp_flags = TCPHDR_ACK;
    tcb->sacked  = 0;
    __skb_header_release(skb);
    tcp_add_write_queue_tail(sk, skb);
    sk_wmem_queued_add(sk, skb->truesize);
    sk_mem_charge(sk, skb->truesize);

    // 如果 sk_send_head 为 NULL，设置为当前 skb
    if (tp->nonagle & TCP_NAGLE_PUSH)
        tp->nonagle &= ~TCP_NAGLE_PUSH;
}
```

`sk_stream_memory_free` 检查发送缓冲区是否还有空间：

```cpp
// include/net/sock.h
static inline bool sk_stream_memory_free(const struct sock *sk)
{
    if (sk->sk_wmem_queued >= sk->sk_sndbuf)
        return false;
    return sk->sk_prot->stream_memory_free ?
        sk->sk_prot->stream_memory_free(sk) : true;
}
```

当 `sk_wmem_queued >= sk_sndbuf` 时，`tcp_sendmsg` 将阻塞在 `sk_stream_wait_memory`，等待 ACK 释放 skb 后唤醒

####    tcp_push：触发发送

`tcp_push` 是 `tcp_sendmsg` 到 `tcp_write_xmit` 的桥梁：

```cpp
// net/ipv4/tcp.c
static void tcp_push(struct sock *sk, int flags, int mss_now,
             int nonagle, int size_goal)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb;

    if (!tcp_send_head(sk))
        return;

    skb = tcp_write_queue_tail(sk);
    if (!(flags & MSG_MORE) || forced_push(tp))
        tcp_mark_push(tp, skb);

    tcp_mark_urg(tp, flags);

    if (tcp_should_autocork(sk, skb, size_goal)) {
        /* 自动 cork：数据量不足一个 MSS 且 qdisc/设备队列有待发送数据 */
        return;
    }

    __tcp_push_pending_frames(sk, mss_now, nonagle);
}
```

`__tcp_push_pending_frames` 直接调用 `tcp_write_xmit`：

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_output.c#L2440
void __tcp_push_pending_frames(struct sock *sk, unsigned int cur_mss,
                   int nonagle)
{
    // 
    if (tcp_write_xmit(sk, cur_mss, nonagle, 0,
               sk_gfp_mask(sk, GFP_ATOMIC)))
        tcp_check_probe_timer(sk);
}
```

##  0x04    tcp_write_xmit：发送决策核心

[`tcp_write_xmit`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_output.c#L2149) 是 TCP 发送路径中最重要的函数，负责从 `sk_write_queue` 中取出待发送的 skb，逐一通过三重门控检查，决定是否实际发送

```mermaid
flowchart TD
    A["tcp_write_xmit(sk, mss_now, nonagle, push_one, gfp)"] --> B["sent_pkts = 0"]
    B --> C{"tcp_send_head(sk)<br/>有待发送skb?"}
    C -->|"NULL"| DONE["return !packets_out && tcp_send_head(sk)"]
    C -->|"skb"| D["tcp_init_tso_segs(skb, mss_now)<br/>计算TSO分段数"]
    D --> E["tcp_cwnd_test(tp, skb)"]
    E -->|"cwnd_quota = 0<br/>拥塞窗口满"| BREAK["break: 停止发送"]
    E -->|"cwnd_quota > 0"| F["tcp_snd_wnd_test(tp, skb, mss_now)"]
    F -->|"false: 超出发送窗口"| BREAK
    F -->|"true"| G["tcp_nagle_test(tp, skb, mss_now, nonagle)"]
    G -->|"false: Nagle延迟"| BREAK
    G -->|"true: 允许发送"| H{"tso_segs > 1?<br/>需要TSO分段?"}
    H -->|"是"| I["tcp_mss_split_point<br/>tso_fragment: 按窗口拆分"]
    H -->|"否"| J["tcp_transmit_skb(sk, skb, 1, gfp)"]
    I --> J
    J -->|"失败"| BREAK
    J -->|"成功"| K["tcp_event_new_data_sent(sk, skb)<br/>推进sk_send_head"]
    K --> L["tcp_minshall_update<br/>sent_pkts += tcp_skb_pcount(skb)"]
    L --> M{"push_one?"}
    M -->|"是"| N["break: 仅发送一个"]
    M -->|"否"| C
    BREAK --> O{"sent_pkts > 0?"}
    O -->|"是"| P["tcp_cwnd_validate(sk)<br/>设置重传定时器"]
    O -->|"否"| DONE
    P --> DONE
```

####    门控一：tcp_cwnd_test（拥塞窗口检查）

```cpp
// net/ipv4/tcp_output.c
static unsigned int tcp_cwnd_test(const struct tcp_sock *tp,
                  const struct sk_buff *skb)
{
    u32 in_flight, cwnd;

    /* Don't be strict about the congestion window for the final FIN.  */
    if ((TCP_SKB_CB(skb)->tcp_flags & TCPHDR_FIN) &&
        tcp_skb_pcount(skb) == 1)
        return 1;

    in_flight = tcp_packets_in_flight(tp);
    cwnd = tp->snd_cwnd;
    if (in_flight >= cwnd)
        return 0;           // 拥塞窗口已满，不允许发送
    return (cwnd - in_flight);  // 还能发送的数据包数
}
```

`tcp_packets_in_flight` 计算在途数据包数：

```cpp
// include/net/tcp.h
static inline unsigned int tcp_packets_in_flight(const struct tcp_sock *tp)
{
    return tp->packets_out - tcp_left_out(tp) + tp->retrans_out;
}

static inline unsigned int tcp_left_out(const struct tcp_sock *tp)
{
    return tp->sacked_out + tp->lost_out;
}
```

含义：`in_flight = packets_out - sacked_out - lost_out + retrans_out`
-   `packets_out`：已发送但未确认的数据包总数
-   `sacked_out`：被对端 SACK 选项确认的包数（虽然未被累积 ACK 确认，但对端已收到）
-   `lost_out`：被标记为丢失的包数
-   `retrans_out`：正在重传的包数

当 `in_flight >= snd_cwnd` 时，拥塞窗口不允许发送更多数据

####    门控二：tcp_snd_wnd_test（发送窗口检查）

```cpp
// net/ipv4/tcp_output.c
static bool tcp_snd_wnd_test(const struct tcp_sock *tp,
                 const struct sk_buff *skb,
                 unsigned int cur_mss)
{
    u32 end_seq = TCP_SKB_CB(skb)->end_seq;

    if (skb->len > cur_mss)
        end_seq = TCP_SKB_CB(skb)->seq + cur_mss;

    return !after(end_seq, tcp_wnd_end(tp));
}

// tcp_wnd_end: 发送窗口右边界
static inline u32 tcp_wnd_end(const struct tcp_sock *tp)
{
    return tp->snd_una + tp->snd_wnd;
}
```

检查 skb 的数据是否在发送窗口范围内。发送窗口 = `[snd_una, snd_una + snd_wnd)`，如果 `end_seq > snd_una + snd_wnd`，则超出对端接收能力

####    门控三：tcp_nagle_test（Nagle 算法）

```cpp
// net/ipv4/tcp_output.c
static inline bool tcp_nagle_test(const struct tcp_sock *tp,
                  const struct sk_buff *skb,
                  unsigned int cur_mss,
                  int nonagle)
{
    // TCP_NAGLE_PUSH: 强制推送（如 forced_push）
    if (nonagle & TCP_NAGLE_PUSH)
        return true;

    // 设置了 TCP_NODELAY 或 TCP_CORK 解除
    if (tp->urg_mode || (TCP_SKB_CB(skb)->tcp_flags & TCPHDR_FIN))
        return true;

    if (!tcp_nagle_check(skb->len < cur_mss, tp, nonagle))
        return true;

    return false;
}

// 核心判断
static inline bool tcp_nagle_check(bool partial,
                   const struct tcp_sock *tp,
                   int nonagle)
{
    // 不是小包（填满MSS），直接发送
    // 或者设置了 TCP_NODELAY
    // 或者没有在途数据包
    return partial &&
        ((nonagle & TCP_NAGLE_CORK) ||
         (!nonagle && tp->packets_out && tcp_minshall_check(tp)));
}
```

Nagle 算法的核心规则：当存在未确认的小包时，延迟发送后续小包，等待前一个包的 ACK 到达。`TCP_NODELAY` 关闭 Nagle，`TCP_CORK` 强制积攒数据

####    tcp_event_new_data_sent：推进 sk_send_head

发送成功后调用此函数，移动 `sk_send_head` 并启动重传定时器：

```cpp
// net/ipv4/tcp_input.c
static void tcp_event_new_data_sent(struct sock *sk, struct sk_buff *skb)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct tcp_sock *tp = tcp_sk(sk);
    unsigned int prior_packets = tp->packets_out;

    tcp_advance_send_head(sk, skb);    // sk_send_head 指向下一个 skb
    tp->snd_nxt = TCP_SKB_CB(skb)->end_seq;

    tp->packets_out += tcp_skb_pcount(skb);
    if (!prior_packets || icsk->icsk_pending == ICSK_TIME_LOSS_PROBE)
        tcp_rearm_rto(sk);             // 启动/重置重传定时器
}

static inline void tcp_advance_send_head(struct sock *sk, const struct sk_buff *skb)
{
    if (tcp_skb_is_last(sk, skb))
        sk->sk_send_head = NULL;        // 队列中最后一个，无更多待发送
    else
        sk->sk_send_head = tcp_write_queue_next(sk, skb);
}
```

##  0x05    tcp_transmit_skb：构造 TCP 报文

[`tcp_transmit_skb`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_output.c#L920) 是实际发送 TCP 段的函数，负责 clone skb、构造 TCP 头、计算校验和，最终调用 IP 层发送

####    skb clone 机制

TCP 与 UDP 最大的不同在于发送时需要 **clone** skb：

```cpp
// net/ipv4/tcp_output.c（精简）
static int tcp_transmit_skb(struct sock *sk, struct sk_buff *skb, int clone_it,
                gfp_t gfp_mask)
{
    const struct inet_connection_sock *icsk = inet_csk(sk);
    struct inet_sock *inet;
    struct tcp_sock *tp;
    struct tcp_skb_cb *tcb;
    struct tcphdr *th;
    int err;

    tp = tcp_sk(sk);

    if (clone_it) {
        // 记录发送时间戳
        skb_mstamp_get(&skb->skb_mstamp);
        TCP_SKB_CB(skb)->tx.in_flight = TCP_SKB_CB(skb)->end_seq - tp->snd_una;
        tcp_rate_skb_sent(sk, skb);

        if (unlikely(skb_cloned(skb)))
            skb = pskb_copy(skb, gfp_mask);    // 已被clone过，需要深拷贝
        else
            skb = skb_clone(skb, gfp_mask);     // 共享数据区的浅拷贝
        if (unlikely(!skb))
            return -ENOBUFS;
    }
    /* ...... */
```

**为什么需要 clone？**
-   原始 skb 保留在 `sk_write_queue` 中，等待 ACK 确认后才能释放
-   如果丢包需要重传，重传时再次 clone 原始 skb 发送
-   `skb_clone` 创建一个新的 `sk_buff` 头部，但共享底层数据缓冲区（引用计数增加）
-   `pskb_copy`：如果 skb 已经被 clone 过（`skb_cloned()` 返回 true），需要连数据一起拷贝

####    TCP 头部构造

```cpp
    inet = inet_sk(sk);
    tcb = TCP_SKB_CB(skb);

    tcp_header_size = tcp_options_size + sizeof(struct tcphdr);

    skb_push(skb, tcp_header_size);
    skb_reset_transport_header(skb);

    skb_orphan(skb);
    skb->sk = sk;
    // 纯 ACK 使用 __sock_wfree，数据包使用 tcp_wfree（TSQ机制）
    skb->destructor = skb_is_tcp_pure_ack(skb) ? __sock_wfree : tcp_wfree;
    skb_set_hash_from_sk(skb, sk);
    atomic_add(skb->truesize, &sk->sk_wmem_alloc);

    /* 构造TCP头 */
    th = (struct tcphdr *)skb->data;
    th->source      = inet->inet_sport;
    th->dest        = inet->inet_dport;
    th->seq         = htonl(tcb->seq);
    th->ack_seq     = htonl(tp->rcv_nxt);     // 捎带 ACK
    *(((__be16 *)th) + 6) = htons(((tcp_header_size >> 2) << 12) |
                    tcb->tcp_flags);

    th->check       = 0;
    th->urg_ptr     = 0;

    /* 设置窗口通告 */
    if (likely(!(tcb->tcp_flags & TCPHDR_SYN))) {
        th->window = htons(tcp_select_window(sk));   // 动态计算接收窗口
        tcp_ecn_send(sk, skb, th, tcp_header_size);
    } else {
        th->window = htons(min(tp->rcv_wnd, 65535U));
    }

    /* 写入TCP选项 */
    tcp_options_write((__be32 *)(th + 1), tp, &opts);

    /* 计算校验和 */
    icsk->icsk_af_ops->send_check(sk, skb);

    /* 调用IP层发送 */
    err = icsk->icsk_af_ops->queue_xmit(sk, skb, &inet->cork.fl);
    // 对于 IPv4：queue_xmit = ip_queue_xmit

    if (likely(err <= 0))
        return err;

    tcp_enter_cwr(sk);
    return net_xmit_eval(err);
}
```

####    tcp_wfree：TSQ 背压机制的入口

当 clone 的 skb 被网络设备层释放时（如 `dev_kfree_skb_any`），`tcp_wfree` 被调用：

```cpp
// net/ipv4/tcp_output.c
void tcp_wfree(struct sk_buff *skb)
{
    struct sock *sk = skb->sk;
    struct tcp_sock *tp = tcp_sk(sk);
    int wmem;

    wmem = atomic_sub_return(skb->truesize - 1, &sk->sk_wmem_alloc);

    // TSQ: 如果队列中积压的字节数低于阈值，触发后续发送
    if (wmem < SKB_TRUESIZE(1) && this_cpu_ksoftirqd() == current)
        goto out;

    // 下半部处理：将 sk 加入 tsq_tasklet
    if (test_and_clear_bit(TSQ_THROTTLED, &tp->tsq_flags)) {
        struct tsq_tasklet *tsq;
        tsq = this_cpu_ptr(&tsq_tasklet);
        list_add(&tp->tsq_node, &tsq->head);
        tasklet_schedule(&tsq->tasklet);
    }
out:
    sk_free(sk);
}
```

TSQ（TCP Small Queues）防止 TCP 在 qdisc/设备队列中积压过多数据，通过 `tcp_wfree` 回调在 skb 释放时触发 `tcp_tsq_handler` → `tcp_write_xmit`，继续发送更多数据

##  0x06    ip_queue_xmit：TCP 的 IP 层入口

TCP 使用 [`ip_queue_xmit`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/ip_output.c#L424) 进入 IP 层，而 UDP 使用 `ip_send_skb`。两者的主要差异在于路由处理方式：

```cpp
// net/ipv4/ip_output.c
int ip_queue_xmit(struct sock *sk, struct sk_buff *skb, struct flowi *fl)
{
    struct inet_sock *inet = inet_sk(sk);
    struct net *net = sock_net(sk);
    struct ip_options_rcu *inet_opt;
    struct flowi4 *fl4;
    struct rtable *rt;
    struct iphdr *iph;
    int res;

    rcu_read_lock();
    inet_opt = rcu_dereference(inet->inet_opt);
    fl4 = &fl->u.ip4;

    // TCP 路由缓存：连接级别复用路由
    rt = skb_rtable(skb);
    if (rt)
        goto packet_routed;

    rt = (struct rtable *)__sk_dst_check(sk, 0);
    if (!rt) {
        __be32 daddr;
        daddr = inet->inet_daddr;
        if (inet_opt && inet_opt->opt.srr)
            daddr = inet_opt->opt.faddr;

        // 路由查找
        rt = ip_route_output_ports(net, fl4, sk,
                       daddr, inet->inet_saddr,
                       inet->inet_dport, inet->inet_sport,
                       sk->sk_protocol, RT_CONN_FLAGS(sk),
                       sk->sk_bound_dev_if);
        if (IS_ERR(rt))
            goto no_route;
        sk_setup_caps(sk, &rt->dst);
    }
    skb_dst_set_noref(skb, &rt->dst);

packet_routed:
    /* IP 选项处理 */
    if (inet_opt && inet_opt->opt.is_strictroute && rt->rt_uses_gateway)
        goto no_route;

    /* 构造 IP 头 */
    skb_push(skb, sizeof(struct iphdr) + (inet_opt ? inet_opt->opt.optlen : 0));
    skb_reset_network_header(skb);
    iph = ip_hdr(skb);
    *((__be16 *)iph) = htons((4 << 12) | (5 << 8) | (inet->tos & 0xff));
    if (ip_dont_fragment(sk, &rt->dst) && !skb->ignore_df)
        iph->frag_off = htons(IP_DF);
    else
        iph->frag_off = 0;
    iph->ttl      = ip_select_ttl(inet, &rt->dst);
    iph->protocol = sk->sk_protocol;    // IPPROTO_TCP
    iph->saddr    = fl4->saddr;
    iph->daddr    = fl4->daddr;

    ip_select_ident_segs(net, skb, sk, tcp_skb_pcount(skb));

    if (inet_opt && inet_opt->opt.optlen)
        ip_options_build(skb, &inet_opt->opt, inet->inet_daddr, rt, 0);

    // 进入公共 IP 输出路径
    res = ip_local_out(net, sk, skb);
    /* ...... */
}
```

**与 UDP 的 `ip_send_skb` 的关键差异**：
1. **路由缓存**：TCP 连接级别复用路由（`__sk_dst_check`），UDP 每次发送可能重新查找
2. **IP 头构造位置**：TCP 在 `ip_queue_xmit` 中构造 IP 头，UDP 在 `__ip_make_skb` 中构造
3. **IP 选项**：TCP 需要处理连接级别的 IP 选项（如源路由）

`ip_local_out` 之后的流程（`__ip_local_out` → netfilter → `ip_output` → `ip_finish_output` → 邻居子系统 → 设备层）与 UDP 完全相同，详见 [内核之旅（十）](https://pandaychen.github.io/2025/04/02/A-LINUX-KERNEL-TRAVEL-10/) 的 0x07-0x0C 章节

##  0x07    拥塞控制深度分析

拥塞控制是 TCP 区别于 UDP 的核心机制之一，其通过动态调整 `snd_cwnd` 来控制发送速率，防止网络拥塞

####    tcp_sock 中拥塞控制相关字段

| 字段 | 类型 | 含义 | 更新时机 |
|------|------|------|----------|
| `snd_cwnd` | `u32` | 拥塞窗口（单位：MSS段） | `tcp_cong_avoid`（每次ACK）、`tcp_enter_loss`/`tcp_enter_recovery`（丢包） |
| `snd_ssthresh` | `u32` | 慢启动阈值 | `tcp_enter_loss`/`tcp_enter_recovery`（设为 cwnd/2） |
| `snd_cwnd_cnt` | `u32` | 拥塞避免阶段的字节计数器 | `tcp_cong_avoid_ai`（线性增长） |
| `snd_cwnd_clamp` | `u32` | cwnd 上限 | 连接初始化时设置 |
| `snd_cwnd_stamp` | `u32` | cwnd 最后更新时间戳 | `tcp_cong_avoid` |
| `packets_out` | `u32` | 已发送未确认的包数 | `tcp_event_new_data_sent`（发送+1）、`tcp_clean_rtx_queue`（ACK-1） |
| `retrans_out` | `u32` | 正在重传的包数 | `tcp_retransmit_skb`（+1）、`tcp_clean_rtx_queue`（-1） |
| `sacked_out` | `u32` | SACK 确认的包数 | `tcp_sacktag_write_queue` |
| `lost_out` | `u32` | 标记丢失的包数 | `tcp_mark_head_lost`、`tcp_clean_rtx_queue` |
| `icsk_ca_state` | `u8` | 拥塞状态机当前状态 | `tcp_set_ca_state` |
| `icsk_ca_ops` | `ptr` | 拥塞控制算法操作集 | 连接初始化（默认 CUBIC） |

####    发送时的拥塞控制交互

在 `tcp_write_xmit` 中，发送决策受以下字段约束：

```
能否发送 = (in_flight < snd_cwnd)
其中 in_flight = packets_out - sacked_out - lost_out + retrans_out
```

每发送一个 skb：
1. `tcp_event_new_data_sent`：`packets_out++`，`snd_nxt` 推进
2. `in_flight` 增大，接近 `snd_cwnd` 后无法继续发送

每收到一个 ACK：
1. `tcp_clean_rtx_queue`：`packets_out--`（被确认的包），`snd_una` 推进
2. `tcp_cong_avoid`：根据拥塞算法增大 `snd_cwnd`
3. `in_flight` 减小，`snd_cwnd` 增大，允许发送更多数据

####    拥塞状态机

```mermaid
stateDiagram-v2
    [*] --> Open

    Open --> Disorder : "收到dupACK或SACK\n(tcp_fastretrans_alert)"
    Open --> Loss : "RTO超时\ntcp_enter_loss\nsnd_cwnd=1\nsnd_ssthresh=max(cwnd/2,2)"

    Disorder --> Open : "正常ACK恢复\n(tcp_try_undo_dsack)"
    Disorder --> Recovery : "累计dupACK>=阈值\ntcp_enter_recovery\nsnd_ssthresh=cwnd/2\nhigh_seq=snd_nxt"
    Disorder --> Loss : "RTO超时"

    Recovery --> Open : "high_seq被确认\ntcp_try_undo_recovery\nsnd_cwnd恢复或保持"
    Recovery --> Loss : "RTO超时\ntcp_enter_loss"

    Loss --> Open : "所有重传被确认\ntcp_try_undo_loss\nsnd_cwnd恢复"
```

**Open** 状态（正常传输）：
-   `snd_cwnd < snd_ssthresh`：慢启动，每 ACK `snd_cwnd += 1`（指数增长）
-   `snd_cwnd >= snd_ssthresh`：拥塞避免，每 RTT `snd_cwnd += 1`（线性增长）

**Loss** 状态（RTO 超时）：

```cpp
// net/ipv4/tcp_input.c
void tcp_enter_loss(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct inet_connection_sock *icsk = inet_csk(sk);

    tp->snd_ssthresh = icsk->icsk_ca_ops->ssthresh(sk);
    // 对于 CUBIC：ssthresh = max(cwnd * beta, 2)，beta 默认 0.7
    tp->snd_cwnd       = 1;        // 重置为 1
    tp->snd_cwnd_cnt   = 0;
    tp->snd_cwnd_stamp = tcp_time_stamp;

    tcp_set_ca_state(sk, TCP_CA_Loss);
    tp->high_seq = tp->snd_nxt;

    // 标记所有在途包为丢失
    tcp_timeout_mark_lost(sk);
    /* ...... */
}
```

**Recovery** 状态（快速重传/恢复）：

```cpp
// net/ipv4/tcp_input.c
static void tcp_enter_recovery(struct sock *sk, bool ece_ack)
{
    struct tcp_sock *tp = tcp_sk(sk);

    tp->prior_ssthresh = tcp_current_ssthresh(sk);
    tcp_init_undo(tp);

    tp->snd_ssthresh = inet_csk(sk)->icsk_ca_ops->ssthresh(sk);
    tp->high_seq = tp->snd_nxt;    // Recovery 结束标志
    tp->prr_delivered = 0;
    tp->prr_out = 0;

    inet_csk(sk)->icsk_ca_state = TCP_CA_Recovery;
}
```

Recovery 使用 PRR（Proportional Rate Reduction）算法控制发送速率，直到 `snd_una` 越过 `high_seq`（所有 Recovery 前发送的数据都被确认）

##  0x08    ACK 处理与 skb 生命周期

`tcp_ack` 是 ACK 接收处理的核心函数，负责推进 `snd_una`、释放已确认的 skb、更新拥塞窗口。其是整个 TCP 发送闭环的关键环节

####    tcp_ack 完整流程

```mermaid
flowchart TD
    A["tcp_ack(sk, skb, flag)"] --> B["ack = TCP_SKB_CB(skb)->ack_seq"]
    B --> C{"ack合法性检查<br/>after(ack, snd_nxt)?"}
    C -->|"是: ACK未来序列号"| D["return -1: 丢弃"]
    C -->|"否"| E{"before(ack, prior_snd_una)?"}
    E -->|"是: 旧的/重复ACK"| F["处理dupACK<br/>tcp_fastretrans_alert"]
    E -->|"否: 有效新ACK"| G["flag |= FLAG_SND_UNA_ADVANCED"]

    G --> H["tcp_may_update_window(tp, ack, seq, nwin)<br/>是否更新snd_wnd"]
    H --> I["tcp_snd_una_update: snd_una = ack<br/>推进已确认边界"]
    I --> J["tcp_clean_rtx_queue(sk, ...)<br/>释放已确认的skb"]
    J --> K{"是否有新数据被确认?"}
    K -->|"是"| L["tcp_cong_avoid(sk, ack, acked)<br/>调用拥塞算法更新cwnd"]
    K -->|"否"| M["跳过cwnd更新"]
    L --> N["tcp_fastretrans_alert(sk, ...)<br/>处理SACK/dupACK/丢包检测"]
    M --> N
    N --> O["tcp_rearm_rto(sk)<br/>重置重传定时器"]
    O --> P["return 1: 处理完成"]
```

####    tcp_ack 核心源码

```cpp
// net/ipv4/tcp_input.c（精简）
static int tcp_ack(struct sock *sk, const struct sk_buff *skb, int flag)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct tcp_sock *tp = tcp_sk(sk);
    struct tcp_sacktag_state sack_state;
    u32 prior_snd_una = tp->snd_una;
    u32 ack_seq = TCP_SKB_CB(skb)->seq;
    u32 ack = TCP_SKB_CB(skb)->ack_seq;
    bool is_dupack = false;
    u32 prior_fackets;
    int prior_packets = tp->packets_out;
    const int prior_unsacked = tp->packets_out - tp->sacked_out;
    int acked = 0;

    /* ACK 合法性检查 */
    if (after(ack, tp->snd_nxt))
        goto invalid_ack;

    if (before(ack, prior_snd_una)) {
        /* 旧ACK，忽略（但可能是 dupACK） */
        if (!(flag & FLAG_NO_CHALLENGE_ACK))
            tcp_send_challenge_ack(sk, skb);
        return -1;
    }

    /* SACK 处理 */
    if (TCP_SKB_CB(skb)->sacked)
        flag |= tcp_sacktag_write_queue(sk, skb, prior_snd_una, &sack_state);

    /* 窗口更新 */
    if (tcp_may_update_window(tp, ack, ack_seq, ntohs(tcp_hdr(skb)->window)))
        flag |= FLAG_WIN_UPDATE;

    /* SND_UNA 推进 */
    if (after(ack, prior_snd_una)) {
        flag |= FLAG_SND_UNA_ADVANCED;
        icsk->icsk_retransmits = 0;  // 重置重传计数
    }

    /* 释放已确认的skb */
    prior_fackets = tp->fackets_out;
    acked = tcp_clean_rtx_queue(sk, prior_fackets, prior_snd_una,
                    &sack_state);

    if (tcp_ack_is_dubious(sk, flag)) {
        is_dupack = !(flag & (FLAG_SND_UNA_ADVANCED | FLAG_NOT_DUP));
        tcp_fastretrans_alert(sk, acked, prior_unsacked,
                      is_dupack, flag);
    }

    /* 拥塞控制：更新 cwnd */
    if (tp->snd_una != prior_snd_una)
        tcp_cong_avoid(sk, ack, acked);

    /* 重传定时器管理 */
    if (tp->packets_out) {
        if (!prior_packets)
            inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                          icsk->icsk_rto, TCP_RTO_MAX);
    } else {
        tcp_rearm_rto(sk);
    }

    /* 唤醒等待发送缓冲区的进程 */
    if (acked)
        tcp_check_space(sk);

    return 1;

invalid_ack:
    return -1;
}
```

####    tcp_clean_rtx_queue：skb 释放的核心

这是 `sk_write_queue` 中 skb 被释放的唯一正常路径：

```cpp
// net/ipv4/tcp_input.c（精简）
static int tcp_clean_rtx_queue(struct sock *sk, int prior_fackets,
                   u32 prior_snd_una,
                   struct tcp_sacktag_state *sack)
{
    struct tcp_sock *tp = tcp_sk(sk);
    const struct inet_connection_sock *icsk = inet_csk(sk);
    struct sk_buff *skb;
    u32 now = tcp_time_stamp;
    int fully_acked = true;
    int flag = 0;
    long ca_rtt_us = -1L;
    long seq_rtt_us = -1L;
    struct sk_buff *head;
    u32 reord = tp->packets_out;
    u32 acked_pcount;
    s32 sack_rtt_us = -1;

    while ((skb = tcp_write_queue_head(sk)) && skb != tcp_send_head(sk)) {
        struct tcp_skb_cb *scb = TCP_SKB_CB(skb);
        u8 sacked = scb->sacked;

        // 检查此 skb 的序列号是否在 ACK 范围内
        if (after(scb->end_seq, tp->snd_una)) {
            if (tcp_skb_pcount(skb) == 1 ||
                !after(tp->snd_una, scb->seq))
                break;
            // 部分确认：需要拆分 skb
            acked_pcount = tcp_tso_acked(sk, skb);
            if (!acked_pcount)
                break;
        } else {
            acked_pcount = tcp_skb_pcount(skb);
        }

        // 计算 RTT（用于 RTO 更新）
        if (likely(!(sacked & TCPCB_RETRANS))) {
            seq_rtt_us = tcp_stamp_us_delta(now, scb->tx.transmit_skb_mstamp);
            ca_rtt_us = seq_rtt_us;
        }

        // 更新统计
        tp->packets_out -= acked_pcount;
        flag |= FLAG_DATA_ACKED;

        if (sacked & TCPCB_RETRANS) {
            tp->retrans_out -= acked_pcount;
            flag |= FLAG_RETRANS_DATA_ACKED;
        }
        if (sacked & TCPCB_SACKED_ACKED)
            tp->sacked_out -= acked_pcount;
        if (sacked & TCPCB_LOST)
            tp->lost_out -= acked_pcount;

        // 释放 skb
        tcp_unlink_write_queue(skb, sk);
        sk_wmem_free_skb(sk, skb);     // 核心：释放内存
    }

    /* RTT 采样更新 */
    if (ca_rtt_us >= 0)
        tcp_ack_update_rtt(sk, flag, seq_rtt_us, sack_rtt_us, ca_rtt_us);

    /* ...... */
    return flag;
}
```

`sk_wmem_free_skb` 的实现：

```cpp
// include/net/sock.h
static inline void sk_wmem_free_skb(struct sock *sk, struct sk_buff *skb)
{
    // 从 sk_wmem_queued 中减去 skb 占用的内存
    sock_set_flag(sk, SOCK_QUEUE_SHRUNK);
    sk->sk_wmem_queued -= skb->truesize;
    sk_mem_uncharge(sk, skb->truesize);
    __kfree_skb(skb);
}
```

`sk_wmem_queued` 减少后，`sk_stream_memory_free` 可能返回 true，唤醒在 `sk_stream_wait_memory` 中等待的发送进程

####    skb 完整生命周期

```mermaid
sequenceDiagram
    participant APP as 用户进程
    participant TSM as tcp_sendmsg
    participant WQ as sk_write_queue
    participant TWX as tcp_write_xmit
    participant TTS as tcp_transmit_skb
    participant IP as ip_queue_xmit
    participant DEV as 网卡驱动
    participant PEER as 对端
    participant TACK as tcp_ack
    participant CLEAN as tcp_clean_rtx_queue

    APP->>TSM: send(fd, data, len)
    TSM->>TSM: sk_stream_alloc_skb<br/>分配skb
    TSM->>TSM: skb_add_data_nocache<br/>拷贝用户数据
    TSM->>WQ: skb_entail: 加入队列<br/>sk_wmem_queued += skb->truesize
    Note over WQ: 原始skb在队列中

    TSM->>TWX: tcp_push → tcp_write_xmit
    TWX->>TTS: tcp_transmit_skb(skb, clone=1)
    TTS->>TTS: skb_clone(skb)<br/>创建克隆体
    Note over WQ: 原始skb保留在队列
    TTS->>IP: ip_queue_xmit(clone_skb)
    IP->>DEV: 发送克隆体
    DEV->>DEV: DMA发送完成
    DEV->>DEV: dev_kfree_skb_any(clone_skb)<br/>释放克隆体
    Note right of DEV: clone_skb释放触发tcp_wfree<br/>sk_wmem_alloc减少

    PEER-->>TACK: ACK到达
    TACK->>TACK: tcp_ack: snd_una推进
    TACK->>CLEAN: tcp_clean_rtx_queue
    CLEAN->>WQ: tcp_unlink_write_queue(skb)<br/>从队列摘除
    CLEAN->>CLEAN: sk_wmem_free_skb(skb)<br/>sk_wmem_queued -= truesize<br/>__kfree_skb(skb)
    Note over WQ: 原始skb最终释放

    Note right of CLEAN: sk_wmem_queued减少<br/>可能唤醒等待发送的进程
```

##  0x09    滑动窗口详解

TCP 滑动窗口是流量控制的核心，确保发送方不会压垮接收方的缓冲区。窗口机制涉及发送窗口和接收窗口两个维度（分这两者进行讨论）

####    发送侧窗口字段

```
    已确认         已发送未确认        可以发送         不可发送
 ←──────── ┃←─────────────── ┃←──────────── ┃←───────────→
           snd_una          snd_nxt        snd_una+snd_wnd

 ←──── snd_wnd（对端通告的接收窗口）──→
```

| 字段 | 含义 | 更新时机 |
|------|------|----------|
| `snd_una` | 最早未被确认的序列号（发送窗口左边界） | `tcp_ack` 收到有效 ACK 时推进 |
| `snd_nxt` | 下一个待发送的序列号 | `tcp_event_new_data_sent` 发送后推进 |
| `snd_wnd` | 对端通告的接收窗口大小 | `tcp_may_update_window` 收到 ACK 时更新 |
| `snd_wl1` | 用于窗口更新判断的 ACK 序列号 | `tcp_may_update_window` |
| `snd_wl2` | 用于窗口更新判断的 ACK 确认号 | `tcp_may_update_window` |
| `write_seq` | 用户已写入的最大序列号 | `tcp_sendmsg` 中数据拷贝后更新 |

有效发送窗口的计算：

```
effective_window = min(snd_cwnd * mss, snd_wnd) - (snd_nxt - snd_una)
```

即 `min(拥塞窗口, 对端接收窗口) - 已发送未确认的数据量`

####    tcp_may_update_window：窗口更新逻辑

收到 ACK 时，内核需要判断是否更新 `snd_wnd`：

```cpp
// net/ipv4/tcp_input.c
static int tcp_may_update_window(const struct tcp_sock *tp,
                 const u32 ack, const u32 ack_seq,
                 const u32 nwin)
{
    return  after(ack, tp->snd_una) ||
        after(ack_seq, tp->snd_wl1) ||
        (ack_seq == tp->snd_wl1 && nwin > tp->snd_wnd);
}
```

三个条件满足任一即更新：
1. `ack > snd_una`：ACK 确认了新数据
2. `ack_seq > snd_wl1`：报文序列号比上次更新窗口的报文更新
3. `ack_seq == snd_wl1 && nwin > snd_wnd`：同一报文但窗口增大了

窗口更新的执行：

```cpp
// tcp_ack 中的窗口更新
if (tcp_may_update_window(tp, ack, ack_seq, nwin)) {
    flag |= FLAG_WIN_UPDATE;
    tcp_update_wl(tp, ack_seq);     // snd_wl1 = ack_seq
    if (tp->snd_wnd != nwin) {
        tp->snd_wnd = nwin;         // 更新发送窗口
        tp->pred_flags = 0;         // 窗口变化，重置快速路径

        if (nwin > tp->max_window) {
            tp->max_window = nwin;
            tcp_sync_mss(sk, inet_csk(sk)->icsk_pmtu_cookie);
        }
    }
}
```

####    接收侧窗口字段

| 字段 | 含义 | 更新时机 |
|------|------|----------|
| `rcv_nxt` | 期望接收的下一个序列号 | `tcp_rcv_established` 收到数据后推进 |
| `rcv_wnd` | 当前通告的接收窗口大小 | `tcp_select_window` 发送 ACK 时计算 |
| `rcv_wup` | 上次通告窗口时的 `rcv_nxt` | `tcp_select_window` |
| `window_clamp` | 接收窗口上限 | `tcp_rcv_space_adjust` 动态调整 |
| `rcv_ssthresh` | 接收窗口慢启动阈值 | 避免突然增大窗口 |

####    tcp_select_window：通告窗口计算

每次发送 ACK 或数据包时，`tcp_transmit_skb` 调用 `tcp_select_window` 计算通告窗口：

```cpp
// net/ipv4/tcp_output.c
static u16 tcp_select_window(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);
    u32 old_win = tp->rcv_wnd;
    u32 cur_win = tcp_receive_window(tp);
    u32 new_win = __tcp_select_window(sk);

    // 窗口不能缩小（RFC 规定窗口右边界不能回退）
    if (new_win < cur_win) {
        new_win = ALIGN(cur_win, 1 << tp->rx_opt.rcv_wscale);
    }
    tp->rcv_wnd = new_win;
    tp->rcv_wup = tp->rcv_nxt;

    // 应用窗口缩放
    return min(new_win, 65535U) >> tp->rx_opt.rcv_wscale;
}

// 当前接收窗口
static inline u32 tcp_receive_window(const struct tcp_sock *tp)
{
    s32 win = tp->rcv_wup + tp->rcv_wnd - tp->rcv_nxt;
    if (win < 0)
        win = 0;
    return (u32) win;
}
```

`__tcp_select_window` 根据可用接收缓冲区计算通告窗口：

```cpp
// net/ipv4/tcp_output.c
u32 __tcp_select_window(struct sock *sk)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct tcp_sock *tp = tcp_sk(sk);
    int mss = icsk->icsk_ack.rcv_mss;
    int free_space = tcp_space(sk);           // 当前可用接收缓冲区
    int allowed_space = tcp_full_space(sk);
    int full_space = min_t(int, tp->window_clamp, allowed_space);
    int window;

    if (free_space < (full_space >> 1)) {
        // 可用空间不足一半：避免 SWS（糊涂窗口综合症）
        icsk->icsk_ack.quick = 0;
        if (free_space < mss)
            return 0;  // 通告零窗口
    }

    // 通告窗口按 MSS 向下对齐
    window = tp->rcv_wnd;
    if (window <= free_space - mss || window > free_space)
        window = (free_space / mss) * mss;

    return window;
}
```

**SWS（Silly Window Syndrome）防护**：当可用接收缓冲区不足总空间的一半时，通告较小甚至零窗口，避免对端发送大量小包

####    零窗口探测

当接收方通告窗口为 `0` 时，发送方无法发送数据。此时启动 Persist Timer 进行零窗口探测：

```cpp
// net/ipv4/tcp_timer.c
static void tcp_probe_timer(struct sock *sk)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct tcp_sock *tp = tcp_sk(sk);

    if (tp->packets_out || !tcp_send_head(sk)) {
        icsk->icsk_probes_out = 0;
        return;
    }

    // 探测超过上限，报告错误
    if (icsk->icsk_probes_out > sysctl_tcp_retries2) {
        tcp_write_err(sk);
        return;
    }

    // 发送窗口探测包（1字节数据）
    tcp_send_probe0(sk);
}
```

当对端缓冲区释放后，发送的 ACK 中会携带更新的窗口值，`tcp_ack` → `tcp_may_update_window` 更新 `snd_wnd`，解除零窗口状态

####    接收窗口自动调整

[`tcp_rcv_space_adjust`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_input.c#L580) 根据实际接收速率动态调整接收缓冲区大小，优化吞吐：

```cpp
// net/ipv4/tcp_input.c
void tcp_rcv_space_adjust(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);
    int time, copied;

    time = tcp_time_stamp - tp->rcvq_space.time;
    if (time < (tp->rcv_rtt_est.rtt >> 3) || tp->rcv_rtt_est.rtt == 0)
        return;

    copied = tp->copied_seq - tp->rcvq_space.seq;
    if (copied <= tp->rcvq_space.space)
        goto new_measure;

    // 增大接收缓冲区
    if (sysctl_tcp_moderate_rcvbuf &&
        !(sk->sk_userlocks & SOCK_RCVBUF_LOCK)) {
        int rcvwin, rcvmem, rcvbuf;

        rcvwin = (copied << 1) + 16 * tp->advmss;
        rcvmem = SKB_TRUESIZE(tp->advmss + MAX_TCP_HEADER);

        rcvbuf = min(rcvwin / tp->advmss * rcvmem,
                 sysctl_tcp_rmem[2]);
        if (rcvbuf > sk->sk_rcvbuf) {
            sk->sk_rcvbuf = rcvbuf;
            tp->window_clamp = rcvwin;
        }
    }
    tp->rcvq_space.space = copied;

new_measure:
    tp->rcvq_space.seq = tp->copied_seq;
    tp->rcvq_space.time = tcp_time_stamp;
}
```

##  0x0A    ACK 与滑动窗口综合分析

前面章节分别从 `tcp_ack` 内部处理逻辑和滑动窗口字段两个角度进行了分析。本节从更高的维度，综合梳理 ACK 在 TCP 发送/接收双端的角色、ACK 如何驱动发送行为、以及 Delayed ACK 的完整机制

####    tcp_ack 的调用本质：发送方还是接收方？

`tcp_ack` 函数[定义](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_input.c#L3538)在 `net/ipv4/tcp_input.c`，从代码位置就能看出，它是**接收路径**的一部分。但理解 `tcp_ack` 的关键在于TCP 是全双工协议，**每一端同时扮演发送方和接收方两个角色**

当主机 A 向主机 B 发送数据时：
- A 是数据**发送方**，调用 `tcp_sendmsg` → `tcp_write_xmit` → `tcp_transmit_skb` 发出数据
- B 收到数据后回复 ACK（可能是纯 ACK包，也可能捎带在数据包中）
- A 收到这个 ACK 包，进入**接收路径** `tcp_v4_rcv` → `tcp_v4_do_rcv` → `tcp_rcv_established`
- 在 `tcp_rcv_established` 中调用 `tcp_ack` 处理 ACK 字段

所以，**"发送方调用 tcp_ack"这句话的准确含义是：数据发送方在自己的接收路径中，处理对端返回的 ACK 确认**。`tcp_ack` 始终在接收路径中被调用，但它服务于发送方的窗口推进和拥塞控制

```
主机A（数据发送方）                     主机B（数据接收方）
┌─────────────────┐                   ┌─────────────────┐
│  tcp_sendmsg    │ ──── DATA ────→   │                 │
│  tcp_write_xmit │                   │  tcp_rcv_established
│  tcp_transmit_skb                   │  tcp_data_queue  │
│                 │                   │  tcp_ack_snd_check
│                 │ ←──── ACK ─────   │  (决定发ACK)     │
│  tcp_rcv_established                │                 │
│  tcp_ack ←─┐    │                   │                 │
│  (处理ACK) │    │                   │                 │
│  发送窗口推进│    │                   │                 │
│  释放skb    │   │                   │                 │
│  更新cwnd   │   │                   │                 │
│  tcp_data_snd_check                │                 │
│  → 可能触发更多发送                   │                 │
└─────────────────┘                   └─────────────────┘
```

####    发送端涉及 tcp_ack 的场景与后续动作

以数据发送方为视角，收到对端 ACK 后进入 [`tcp_rcv_established`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_input.c#L5351)，`tcp_ack` 在三条路径上被调用：

**场景一：快速路径 - 纯 ACK（对端只回复确认，不携带数据）**

这是单向传输（如文件下载）中最常见的场景。对端收到数据后回复纯 ACK：

```cpp
// net/ipv4/tcp_input.c - tcp_rcv_established 快速路径
// 条件：tcp_header_len == len（包长等于TCP头长，即无数据负载）

//https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_input.c#L5413
......
if (tcp_header_len == len) {
    // 纯ACK，无数据
    tcp_ack(sk, skb, 0);           // 处理ACK
    __kfree_skb(skb);              // 释放收到的skb
    tcp_data_snd_check(sk);        // 检查是否有数据可以发送
    return;
}
......
```

后续动作链如下，`tcp_ack` 内部推进 `snd_una`、释放已确认 skb、更新拥塞窗口。返回后 `tcp_data_snd_check` 检查发送队列是否有待发数据，若有则触发 `tcp_write_xmit`

**场景二：快速路径 - 数据 + ACK（对端回复数据的同时捎带确认）**

全双工通信（如 SSH 交互）中常见，对端的数据包中 ACK 字段确认了己方之前发送的数据，通常是PSH+ACK报文

```cpp
// net/ipv4/tcp_input.c - tcp_rcv_established 快速路径（有数据）
tcp_event_data_recv(sk, skb);      // 标记ACK调度，调整ATO

if (TCP_SKB_CB(skb)->ack_seq != tp->snd_una) {
    // ACK字段确认了新数据
    tcp_ack(sk, skb, FLAG_DATA);   // FLAG_DATA 表示此包携带数据
    tcp_data_snd_check(sk);        // 检查是否可以发送更多数据
    if (!inet_csk_ack_scheduled(sk))
        goto no_ack;
}

__tcp_ack_snd_check(sk, 0);       // 决定是否需要回复ACK给对端
```

后续动作链，与场景一相同的 `tcp_ack` 内部处理 + `tcp_data_snd_check`。额外的 `__tcp_ack_snd_check` 是因为收到了对端数据，需要决定是立即回复 ACK 还是延迟（Delayed ACK）

**场景三：慢速路径（乱序包、SACK、窗口更新等复杂情况）**

```cpp
// net/ipv4/tcp_input.c - tcp_rcv_established 慢速路径
slow_path:
    // ... 校验、validate_incoming ...
step5:
    if (tcp_ack(sk, skb, FLAG_SLOWPATH | FLAG_UPDATE_TS_RECENT) < 0)
        goto discard;

    tcp_urg(sk, skb, th);
    tcp_data_queue(sk, skb);       // 数据入队（含乱序处理）
    tcp_data_snd_check(sk);        // 检查是否可以发送更多数据
    tcp_ack_snd_check(sk);         // 决定是否回复ACK
    return;
```

慢速路径传入 `FLAG_SLOWPATH`，`tcp_ack` 内部会执行更完整的处理（SACK 标记、dupACK 检测、`tcp_fastretrans_alert` 等），这也是`tcp_ack`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_input.c#L3538)看上去实现非常复杂的原因

上述**三个场景的共同点是`tcp_ack` 返回后都会调用 `tcp_data_snd_check`**，这是 ACK 驱动发送（数据）的入口

####    接收端涉及 tcp_ack 的场景

站在数据接收方的视角，收到的数据包中同样携带 ACK 字段（确认己方之前发送的数据，如果有的话）。代码路径与上面完全相同，即`tcp_rcv_established` 并不区分"我是发送方还是接收方"，它只处理收到的 TCP 报文中的各个字段

接收端的核心关注点不同：
- **`tcp_ack` 部分**：如果接收端之前也发送过数据（全双工），则 `tcp_ack` 推进其发送窗口。如果接收端是纯接收方（如下载场景的客户端），`tcp_ack` 不会有实质操作（因为 `snd_una` 没有需要推进的数据）
- **`tcp_data_queue` 部分**：这是接收端的核心逻辑，将数据放入接收队列，更新 `rcv_nxt`
- **`tcp_ack_snd_check` 部分**：决定是否需要回复 ACK（立即发送还是 Delayed ACK），这是接收端最重要的反馈行为

####    tcp_ack 如何影响下一次发送行为

对于发送方而言，收到对端 ACK 后 `tcp_ack` 的处理直接影响后续发送能力。以下流程图展示 ACK 到达后的完整影响链：

```mermaid
flowchart TD
    ACK["对端ACK到达<br/>tcp_rcv_established"] --> TA["tcp_ack(sk, skb, flag)"]

    TA --> MW["tcp_may_update_window<br/>snd_wnd = nwin（对端通告窗口）"]
    TA --> SU["snd_una = ack<br/>发送窗口左边界推进"]
    TA --> CL["tcp_clean_rtx_queue<br/>释放已确认skb<br/>packets_out减少"]
    TA --> CA["tcp_cong_avoid<br/>snd_cwnd增大（慢启动/拥塞避免）"]
    TA --> CS["tcp_check_space<br/>sk_wmem_queued减少可能唤醒阻塞进程"]

    SU --> EW["有效发送窗口增大<br/>min(cwnd*mss, snd_wnd) - (snd_nxt-snd_una)"]
    MW --> EW
    CA --> EW
    CL --> EW

    TA -.->|"返回"| DSC["tcp_data_snd_check(sk)"]
    DSC --> PPF["tcp_push_pending_frames(sk)"]
    PPF --> SH{"tcp_send_head(sk)<br/>发送队列有待发数据?"}
    SH -->|"有"| TWX["__tcp_push_pending_frames<br/>→ tcp_write_xmit"]
    SH -->|"无"| DONE["无需发送"]
    TWX --> GATE["tcp_write_xmit 三重门控"]
    GATE --> CWT["tcp_cwnd_test: snd_cwnd够?"]
    GATE --> SWT["tcp_snd_wnd_test: snd_wnd够?"]
    GATE --> NT["tcp_nagle_test: Nagle允许?"]
    CWT --> SEND["tcp_transmit_skb 发送"]
    SWT --> SEND
    NT --> SEND
```

具体的字段级影响：

**1. 左边界推进（`snd_una`）**

`tcp_ack` 收到有效 ACK 后，将 `snd_una` 推进到 ACK 确认号。这直接减少了"已发送未确认"的数据量（`snd_nxt - snd_una`），等价于释放了发送窗口空间：

```text
发送前:  snd_una=1000  snd_nxt=5000  已发送未确认=4000字节
收到ACK=3000后: snd_una=3000  snd_nxt=5000  已发送未确认=2000字节
→ 释放了2000字节的发送窗口空间
```

**2. 右边界可能扩展（`snd_wnd` + `snd_cwnd`）**

有效发送窗口的计算：`effective_window = min(snd_cwnd * mss, snd_wnd) - (snd_nxt - snd_una)`

ACK 可能同时带来窗口通告值的更新（`tcp_may_update_window` → `snd_wnd`），以及拥塞窗口的增大（`tcp_cong_avoid` → `snd_cwnd`）。这两者共同决定了右边界

**3. 发送缓冲区释放**

`tcp_clean_rtx_queue` 释放已确认 skb → `sk_wmem_queued` 减少 → 如果之前 `tcp_sendmsg` 因缓冲区满而阻塞（在 `sk_stream_wait_memory` 中睡眠），此时可以被唤醒继续写入新数据

####    tcp_ack 与滑动窗口的关系

**`tcp_ack` 是否是控制滑动窗口移动的最根本因素？**

答案是 **`tcp_ack` 是驱动发送窗口左边界移动的唯一因素，同时也是右边界两个输入来源（`snd_wnd` 和 `snd_cwnd`）的更新触发点**

发送窗口的三个边界由以下因素控制：

| 边界 | 对应字段 | 更新方式 | 是否由 tcp_ack 触发 |
|------|----------|----------|---------------------|
| 左边界 | `snd_una` | ACK 确认号推进 | 是，`tcp_ack` 是唯一触发点 |
| 右边界（流控） | `snd_wnd` | 对端通告窗口 | 是，`tcp_may_update_window` 在 `tcp_ack` 中调用 |
| 右边界（拥塞） | `snd_cwnd` | 本端拥塞算法计算 | 是，`tcp_cong_avoid` 在 `tcp_ack` 处理有效新 ACK 后调用 |
| 可发送位置 | `snd_nxt` | 本端发送新数据 | 否，由 `tcp_event_new_data_sent` 在发送时推进 |

```text
    左边界                可发送位置              右边界
     │                      │                      │
     ▼                      ▼                      ▼
─────┃━━━━━━━━━━━━━━━━━━━━━━┃──────────────────────┃──────────→
   snd_una              snd_nxt     snd_una + min(cwnd*mss, snd_wnd)
     │                                             │
     │←── tcp_ack 驱动推进                          │
                                  tcp_ack 驱动更新 ──→│
                                  (snd_wnd + snd_cwnd)
```

除 `tcp_ack` 外，还有少数场景也会影响窗口：
- **丢包检测**：`tcp_enter_loss` / `tcp_enter_recovery` 会大幅减小 `snd_cwnd`（但这些函数也是在 `tcp_ack` → `tcp_fastretrans_alert` 的调用链中触发的）
- **RTO 超时**：`tcp_retransmit_timer` 会将 `snd_cwnd` 重置为 `1`，但这不在 `tcp_ack` 路径中
- **零窗口探测**：`tcp_probe_timer` 发送探测包，但窗口本身仍然由 `tcp_ack` 收到回复后更新

因此，**在正常数据传输过程中，`tcp_ack` 确实是控制滑动窗口移动的最根本因素**

####    tcp_ack 如何驱动 tcp_write_xmit

`tcp_ack` 本身并不直接调用 `tcp_write_xmit`。驱动发送的完整链路是 `tcp_ack` 返回后由 `tcp_rcv_established` 调用 `tcp_data_snd_check`：

```cpp
// net/ipv4/tcp_input.c
static inline void tcp_data_snd_check(struct sock *sk)
{
    tcp_push_pending_frames(sk);
    tcp_check_space(sk);
}
```

`tcp_push_pending_frames` [定义](https://elixir.bootlin.com/linux/v4.11.6/source/include/net/tcp.h#L1652)：

```cpp
// include/net/tcp.h
static inline void tcp_push_pending_frames(struct sock *sk)
{
    if (tcp_send_head(sk)) {
        struct tcp_sock *tp = tcp_sk(sk);
        __tcp_push_pending_frames(sk, tcp_current_mss(sk), tp->nonagle);
    }
}
```

只要 `tcp_send_head(sk)` 不为 NULL（即 `sk_write_queue` 中有尚未发送的 skb），就调用 `__tcp_push_pending_frames`：

```cpp
// net/ipv4/tcp_output.c
void __tcp_push_pending_frames(struct sock *sk, unsigned int cur_mss,
                   int nonagle)
{
    if (unlikely(sk->sk_state == TCP_CLOSE))
        return;

    if (tcp_write_xmit(sk, cur_mss, nonagle, 0,
               sk_gfp_mask(sk, GFP_ATOMIC)))
        tcp_check_probe_timer(sk);
}
```

最终调用 `tcp_write_xmit` 进入发送决策核心（三重门控：拥塞窗口、发送窗口、Nagle 算法）

完整的 ACK 驱动发送调用链：

```
tcp_rcv_established 收到ACK
  │
  ├─→ tcp_ack(sk, skb, flag)
  │     ├─→ snd_una 推进
  │     ├─→ tcp_clean_rtx_queue → 释放skb, packets_out减少
  │     ├─→ tcp_cong_avoid → snd_cwnd增大
  │     ├─→ tcp_may_update_window → snd_wnd更新
  │     └─→ tcp_check_space → 可能唤醒阻塞进程
  │
  └─→ tcp_data_snd_check(sk)                      ← tcp_ack返回后
        ├─→ tcp_push_pending_frames(sk)
        │     └─→ if (tcp_send_head(sk))           ← 队列中有待发数据?
        │           └─→ __tcp_push_pending_frames
        │                 └─→ tcp_write_xmit       ← 进入发送决策
        │                       ├─→ tcp_cwnd_test   (snd_cwnd够?)
        │                       ├─→ tcp_snd_wnd_test (snd_wnd够?)
        │                       ├─→ tcp_nagle_test   (Nagle允许?)
        │                       └─→ tcp_transmit_skb (构造并发送)
        └─→ tcp_check_space(sk)
```

`tcp_ack` 内部放宽了三重门控的约束（`snd_cwnd` 增大、`snd_wnd` 更新、`packets_out` 减少），然后 `tcp_data_snd_check` 触发 `tcp_write_xmit` 重新评估门控条件，将之前因窗口/拥塞限制而未发送的数据发出去

####    纯 ACK 与带数据 ACK 的对比

**纯 ACK（只有 ACK 标志，不携带数据负载）并不少见**，在很多场景中甚至是主要的 ACK 形式

| 场景 | 纯 ACK | 带数据 ACK | 说明 |
|------|--------|-----------|------|
| 单向下载（HTTP GET 响应） | 非常常见 | 极少 | 客户端只接收数据，回复纯 ACK |
| 单向上传（HTTP POST 请求体） | 极少 | 极少 | 服务端纯 ACK 确认客户端数据 |
| 全双工交互（SSH 终端） | 较少 | 常见 | 数据包自然捎带 ACK（piggybacking） |
| TCP 三次握手第三步 | 是（纯 ACK） | 可能（TFO） | 客户端发送最后的 ACK |
| Delayed ACK 超时触发 | 是 | 否 | 延迟定时器到期，发送纯 ACK |
| keepalive 响应 | 是 | 否 | 探活回复 |
| 窗口更新通知 | 是 | 否 | 接收方缓冲区释放后主动通告 |

内核中的区分逻辑在 `tcp_rcv_established` 快速路径中：

```cpp
// net/ipv4/tcp_input.c - tcp_rcv_established
int tcp_header_len = th->doff * 4;

if (tcp_header_len == len) {
    // 纯ACK：TCP头长度等于包总长度，无数据负载
    // → tcp_ack → tcp_data_snd_check
} else {
    // 有数据：len > tcp_header_len
    // → tcp_event_data_recv → tcp_ack → tcp_data_snd_check → __tcp_ack_snd_check
}
```

对发送方而言，纯 ACK 和带数据 ACK 在 `tcp_ack` 内部的处理逻辑基本一致，都是推进 `snd_una`、释放 skb、更新窗口。差异在于快速路径的 flag 参数（纯 ACK 传 `0`，带数据传 `FLAG_DATA`），以及带数据 ACK 会额外触发 `__tcp_ack_snd_check` 决定是否需要回复 ACK

####    Delayed ACK（延迟ACK） 完整机制

Delayed ACK 是 TCP 接收方的优化策略：不对每个收到的数据段立即回复 ACK，而是延迟一小段时间，期望能将 ACK 捎带在即将发送的数据包中（piggybacking），从而减少网络上的纯 ACK 小包数量

**1. ACK 调度：`tcp_event_data_recv`**

收到数据后，[`tcp_event_data_recv`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_input.c#L657) 标记需要发送 ACK 并调整 ATO（ACK Timeout）：

```cpp
// net/ipv4/tcp_input.c
static void tcp_event_data_recv(struct sock *sk, struct sk_buff *skb)
{
    struct inet_connection_sock *icsk = inet_csk(sk);

    // 标记ACK调度（但不立即发送）
    inet_csk_schedule_ack(sk);

    // 自适应调整ATO
    if (!icsk->icsk_ack.ato) {
        tcp_incr_quickack(sk);              // 首包进入QuickACK模式
        icsk->icsk_ack.ato = TCP_ATO_MIN;   // 初始 40ms
    } else {
        int m = now - icsk->icsk_ack.lrcvtime;
        if (m <= TCP_ATO_MIN / 2) {
            // 包到达很快：缩小ATO
            icsk->icsk_ack.ato = (icsk->icsk_ack.ato >> 1) + TCP_ATO_MIN / 2;
        } else if (m < icsk->icsk_ack.ato) {
            // 正常间隔：平滑调整
            icsk->icsk_ack.ato = (icsk->icsk_ack.ato >> 1) + m;
        } else if (m > icsk->icsk_rto) {
            // 间隔太长：进入QuickACK模式
            tcp_incr_quickack(sk);
        }
    }
    icsk->icsk_ack.lrcvtime = now;
}
```

**2. ACK 发送决策：`__tcp_ack_snd_check`**

[`__tcp_ack_snd_check`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_input.c#L5197) 决定立即发送 ACK 还是延迟：

todo

```cpp
// net/ipv4/tcp_input.c
static void __tcp_ack_snd_check(struct sock *sk, int ofo_possible)
{
    struct tcp_sock *tp = tcp_sk(sk);

    if (((tp->rcv_nxt - tp->rcv_wup) > inet_csk(sk)->icsk_ack.rcv_mss &&
         __tcp_select_window(sk) >= tp->rcv_wnd) ||
        tcp_in_quickack_mode(sk) ||
        (ofo_possible && !RB_EMPTY_ROOT(&tp->out_of_order_queue))) {
        // 立即发送ACK
        tcp_send_ack(sk);
    } else {
        // 延迟发送ACK
        tcp_send_delayed_ack(sk);
    }
}
```

通常，内核立即发送 ACK 的三个条件（满足任一即立即发送）：

| 条件 | 含义 | 目的 |
|------|------|------|
| `(rcv_nxt - rcv_wup) > rcv_mss` 且窗口可以前进 | 累计收到超过一个 full-size 段的数据 | RFC 建议每收两个全尺寸段发一次 ACK |
| `tcp_in_quickack_mode(sk)` | 处于 QuickACK 模式 | 连接初期 / 丢包恢复期需要快速反馈 |
| `ofo_possible` 且乱序队列非空 | 有乱序数据 | 立即发送 dupACK 触发快速重传 |

`tcp_ack_snd_check` 是 `__tcp_ack_snd_check` 的包装，增加了一个前置检查：

```cpp
// net/ipv4/tcp_input.c
static inline void tcp_ack_snd_check(struct sock *sk)
{
    if (!inet_csk_ack_scheduled(sk)) {
        // 没有需要发送的ACK（可能已经通过数据包捎带发出了）
        return;
    }
    __tcp_ack_snd_check(sk, 1);    // ofo_possible=1，允许检查乱序队列
}
```

**3. Delayed ACK 定时器：`tcp_send_delayed_ack`**

当决定延迟发送时，[`tcp_send_delayed_ack`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_output.c#L3302) 启动定时器：

```cpp
// net/ipv4/tcp_output.c
void tcp_send_delayed_ack(struct sock *sk)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    int ato = icsk->icsk_ack.ato;
    unsigned long timeout;

    tcp_ca_event(sk, CA_EVENT_DELAYED_ACK);

    if (ato > TCP_DELACK_MIN) {
        int max_ato = HZ / 2;     // 默认最大 500ms

        if (icsk->icsk_ack.pingpong ||
            (icsk->icsk_ack.pending & ICSK_ACK_PUSHED))
            max_ato = TCP_DELACK_MAX;    // 交互模式下最大 200ms

        // 用 RTT 约束延迟上限
        if (tp->srtt_us) {
            int rtt = max_t(int, usecs_to_jiffies(tp->srtt_us >> 3),
                    TCP_DELACK_MIN);
            if (rtt < max_ato)
                max_ato = rtt;
        }
        ato = min(ato, max_ato);
    }

    timeout = jiffies + ato;

    // 如果已有定时器且即将到期，立即发送
    if (icsk->icsk_ack.pending & ICSK_ACK_TIMER) {
        if (icsk->icsk_ack.blocked ||
            time_before_eq(icsk->icsk_ack.timeout, jiffies + (ato >> 2))) {
            tcp_send_ack(sk);
            return;
        }
    }

    // 设置定时器
    icsk->icsk_ack.pending |= ICSK_ACK_SCHED | ICSK_ACK_TIMER;
    icsk->icsk_ack.timeout = timeout;
    sk_reset_timer(sk, &icsk->icsk_delack_timer, timeout);
}
```

**4. 定时器到期：`tcp_delack_timer_handler`**

[`tcp_delack_timer_handler`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_timer.c#L245) 在定时器到期时发送 ACK：

```cpp
// net/ipv4/tcp_timer.c
void tcp_delack_timer_handler(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct inet_connection_sock *icsk = inet_csk(sk);

    // 状态检查
    if (((1 << sk->sk_state) & (TCPF_CLOSE | TCPF_LISTEN)) ||
        !(icsk->icsk_ack.pending & ICSK_ACK_TIMER))
        goto out;

    // 还没到时间
    if (time_after(icsk->icsk_ack.timeout, jiffies)) {
        sk_reset_timer(sk, &icsk->icsk_delack_timer, icsk->icsk_ack.timeout);
        goto out;
    }

    icsk->icsk_ack.pending &= ~ICSK_ACK_TIMER;

    // 处理prequeue中的残留包
    if (!skb_queue_empty(&tp->ucopy.prequeue)) {
        while ((skb = __skb_dequeue(&tp->ucopy.prequeue)) != NULL)
            sk_backlog_rcv(sk, skb);
        tp->ucopy.memory = 0;
    }

    if (inet_csk_ack_scheduled(sk)) {
        if (!icsk->icsk_ack.pingpong) {
            icsk->icsk_ack.ato = min(icsk->icsk_ack.ato << 1, icsk->icsk_rto);
        } else {
            icsk->icsk_ack.pingpong = 0;
            icsk->icsk_ack.ato = TCP_ATO_MIN;
        }
        tcp_send_ack(sk);           // 最终发送ACK
    }
    // ...
}
```

注意，如果 `tcp_delack_timer` 触发时 socket 被用户进程持有（`sock_owned_by_user`），则设置 `icsk_ack.blocked = 1` 并延迟到 `tcp_release_cb` 中执行

**5. Piggybacking：ACK 捎带**

如果在延迟等待期间，本端有数据要发送，`tcp_transmit_skb` 构造 TCP 头时会自动携带最新的 ACK 信息（`rcv_nxt` 和通告窗口），从而取消 Delayed ACK 定时器。这就是 piggybacking 机制，即ACK "免费"搭载在数据包中

以下流程图展示 Delayed ACK 的完整决策和处理过程：

```mermaid
flowchart TD
    RCV["收到数据包<br/>tcp_rcv_established"] --> EDR["tcp_event_data_recv<br/>inet_csk_schedule_ack<br/>调整ATO"]
    EDR --> ASC["__tcp_ack_snd_check"]

    ASC --> C1{"累计数据 > rcv_mss<br/>且窗口可前进?"}
    C1 -->|"是"| IMM["tcp_send_ack<br/>立即发送ACK"]

    C1 -->|"否"| C2{"QuickACK模式?"}
    C2 -->|"是"| IMM

    C2 -->|"否"| C3{"有乱序数据?"}
    C3 -->|"是"| IMM

    C3 -->|"否"| DEL["tcp_send_delayed_ack<br/>设置icsk_delack_timer"]

    DEL --> W{"等待期间<br/>是否有数据要发送?"}
    W -->|"有数据"| PB["tcp_transmit_skb<br/>ACK捎带在数据包中<br/>Piggybacking"]
    W -->|"无数据<br/>定时器到期"| TMR["tcp_delack_timer_handler"]
    TMR --> BLK{"socket被用户进程持有?"}
    BLK -->|"否"| SACK["tcp_send_ack<br/>发送纯ACK"]
    BLK -->|"是"| DEFER["设置blocked=1<br/>延迟到tcp_release_cb"]
    DEFER --> SACK
```

####    Delayed ACK 的关键参数

| 参数 | 值 | 说明 |
|------|---|------|
| `TCP_DELACK_MIN` | 40ms（`HZ/25`） | ATO 最小值 |
| `TCP_DELACK_MAX` | 200ms（`HZ/5`） | 交互模式最大延迟 |
| `TCP_ATO_MIN` | 40ms | ATO 初始值 |
| `HZ/2` | 500ms | 非交互模式最大延迟 |
| `icsk_ack.pingpong` | 0/1 | 交互模式标记（影响 `max_ato` 选择） |
| `icsk_ack.quick` | 计数 | QuickACK 模式剩余快速 ACK 数量 |

##  0x0B    滑动窗口变量主动 / 被动修改总结

本节将 `tcp_sock` 中与滑动窗口相关的所有字段，按照"本端主动修改"和"被动接受对端数据包驱动修改"两个维度进行分类。这一分类有助于理解：哪些字段完全由本端控制，哪些字段的值取决于对端的行为

####    发送侧窗口变量

| 字段 | 主动/被动 | 修改时机 | 内核函数 | 说明 |
|------|-----------|----------|----------|------|
| `snd_una` | **被动** | 收到对端 ACK | `tcp_ack` | 完全由对端 ACK 确认号决定，本端无法主动修改 |
| `snd_nxt` | **主动** | 本端发送新数据 | `tcp_event_new_data_sent` | 发送时自增，不依赖对端 |
| `write_seq` | **主动** | 用户进程写入数据 | `tcp_sendmsg` | 用户态 `send`/`write` 推进，不依赖网络事件 |
| `snd_wnd` | **被动** | 对端通告窗口 | `tcp_may_update_window`（`tcp_ack` 内） | 值直接取自对端 TCP 头中的 window 字段 |
| `snd_wl1` | **被动** | 窗口更新判断 | `tcp_update_wl`（`tcp_ack` 内） | 记录触发窗口更新的报文序列号 |
| `snd_wl2` | **被动** | 窗口更新判断 | `tcp_may_update_window` | 记录触发窗口更新的 ACK 确认号 |
| `snd_cwnd` | **主动**（ACK 事件触发） | 收到有效 ACK 后拥塞算法计算 | `tcp_cong_avoid` | 虽由 ACK 事件触发，但具体值由本端拥塞控制算法（如 CUBIC）独立计算 |
| `snd_ssthresh` | **主动** | 丢包/拥塞事件 | `tcp_enter_loss` / `tcp_enter_recovery` | 本端检测到拥塞后自主调整 |
| `packets_out` | **双向** | 发送时 +1，收到 ACK 时 -N | `tcp_event_new_data_sent`（+）/ `tcp_clean_rtx_queue`（-） | 发送增加是主动的，ACK 确认减少是被动的 |
| `retrans_out` | **双向** | 重传时 +1，收到 ACK 时 -N | `tcp_retransmit_skb`（+）/ `tcp_clean_rtx_queue`（-） | 同 `packets_out` |
| `sacked_out` | **被动** | 收到 SACK 选项 | `tcp_sacktag_write_queue`（`tcp_ack` 内） | 完全由对端 SACK 信息决定 |
| `lost_out` | **主动** | 本端丢包检测算法 | `tcp_mark_head_lost` / `tcp_update_scoreboard` | 基于 dupACK/SACK 信息由本端算法判定 |

####    接收侧窗口变量

| 字段 | 主动/被动 | 修改时机 | 内核函数 | 说明 |
|------|-----------|----------|----------|------|
| `rcv_nxt` | **主动** | 收到按序数据后推进 | `tcp_rcv_established` / `tcp_data_queue` | 本端根据收到的数据自主推进 |
| `rcv_wnd` | **主动** | 发送 ACK/数据时计算 | `tcp_select_window` | 基于本端接收缓冲区可用空间计算 |
| `rcv_wup` | **主动** | 发送 ACK/数据时更新 | `tcp_select_window` | 记录上次通告窗口时的 `rcv_nxt` |
| `window_clamp` | **主动** | 接收缓冲区自适应调整 | `tcp_rcv_space_adjust` | 基于本端接收速率动态调整上限 |
| `rcv_ssthresh` | **主动** | 接收窗口慢启动 | `__tcp_select_window` / `tcp_grow_window` | 避免窗口突然增大 |

####    核心洞察

**1. 发送侧："左边界被动，右边界混合"**

- 发送窗口左边界（`snd_una`）**完全被动**：只有对端发来的 ACK 才能推进它，本端无法主动改变
- 发送窗口右边界由 `min(snd_cwnd * mss, snd_wnd)` 决定：
  - `snd_wnd` 是**被动**的（对端通告值）
  - `snd_cwnd` 是**主动**的（本端拥塞算法计算，虽然由 ACK 事件触发但值由本端决定）

这意味着：发送能力的上限由**对端的接收能力**（`snd_wnd`）和**网络的承载能力**（`snd_cwnd`）共同决定，前者完全被动，后者半主动

**2. 接收侧："全部主动"**

接收侧的所有窗口变量都由本端主动管理：
- `rcv_nxt` 由收到的数据推进（但推进是本端决定的）
- `rcv_wnd` 完全基于本端缓冲区状态计算
- `window_clamp` 基于本端接收速率调整

接收方唯一"被动"的方面是：对端发送的数据触发了 `rcv_nxt` 的推进和 `rcv_wnd` 的重新计算，但这些值的最终确定完全由本端控制

**3. 对称性与非对称性**

在全双工通信中，同一个 `tcp_sock` 同时维护发送侧和接收侧的窗口变量。一端的接收侧行为（如 `rcv_wnd` 的通告）直接影响对端发送侧的被动变量（对端的 `snd_wnd`）：

```
主机A                              主机B
┌─────────────────┐               ┌─────────────────┐
│ 发送侧（主动）    │               │ 接收侧（主动）    │
│ snd_nxt: 主动推进 │───DATA──→    │ rcv_nxt: 主动推进 │
│ snd_cwnd: 主动算  │               │ rcv_wnd: 主动算   │
│                  │               │                  │
│ 发送侧（被动）    │               │ 接收侧（反馈）    │
│ snd_una: 被动更新 │←──ACK+WIN──  │ 通告窗口=rcv_wnd  │
│ snd_wnd: 被动更新 │               │                  │
└─────────────────┘               └─────────────────┘
```

A 的 `snd_wnd` 被动更新值 = B 的 `rcv_wnd` 主动计算值。这种"发送被动、接收主动"的非对称设计，确保了接收方始终能控制数据流入的速率

##  0x0C    Nagle 算法与发送时机

Nagle 算法的目标是减少网络中的小包数量。它在 `tcp_write_xmit` 的门控检查中通过 `tcp_nagle_test` 实现

####    Nagle 算法判断逻辑

```cpp
// net/ipv4/tcp_output.c
static inline bool tcp_nagle_test(const struct tcp_sock *tp,
                  const struct sk_buff *skb,
                  unsigned int cur_mss,
                  int nonagle)
{
    /* TCP_NAGLE_PUSH 强制发送（如 forced_push 触发） */
    if (nonagle & TCP_NAGLE_PUSH)
        return true;

    /* 紧急模式或 FIN 标志直接发送 */
    if (tp->urg_mode || (TCP_SKB_CB(skb)->tcp_flags & TCPHDR_FIN))
        return true;

    /* 核心判断 */
    if (!tcp_nagle_check(skb->len < cur_mss, tp, nonagle))
        return true;

    return false;
}

static inline bool tcp_nagle_check(bool partial,
                   const struct tcp_sock *tp,
                   int nonagle)
{
    /* partial: 数据不足一个 MSS */
    /* 不是小包 → 直接发 */
    /* TCP_NODELAY → 直接发 */
    /* 没有未确认数据 → 直接发 */
    return partial &&
        ((nonagle & TCP_NAGLE_CORK) ||
         (!nonagle && tp->packets_out && tcp_minshall_check(tp)));
}
```

Nagle 延迟发送的条件（全部满足时才延迟）：
1. 数据量不足一个 MSS（`partial = true`）
2. 没有设置 `TCP_NODELAY`
3. 存在未确认的在途数据包（`packets_out > 0`）

####    TCP_NODELAY 与 TCP_CORK

| 选项 | 效果 | 适用场景 |
|------|------|----------|
| `TCP_NODELAY` | 关闭 Nagle，所有数据立即发送 | 交互式应用（SSH、游戏） |
| `TCP_CORK` | 强制积攒数据，直到解除或超时 | 文件传输、HTTP 响应头+体 |
| 默认（均不设置） | Nagle 算法生效 | 通用场景 |

在内核中，`TCP_CORK` 通过 `nonagle` 字段中的 `TCP_NAGLE_CORK` 位控制：

```cpp
// net/ipv4/tcp.c - setsockopt
case TCP_CORK:
    if (val) {
        tp->nonagle |= TCP_NAGLE_CORK;
    } else {
        tp->nonagle &= ~TCP_NAGLE_CORK;
        if (tp->nonagle & TCP_NAGLE_OFF)
            tp->nonagle |= TCP_NAGLE_PUSH;
        tcp_push_pending_frames(sk);   // 解除 CORK 时立即推送
    }
    break;
```

####    Nagle 与 Delayed ACK 的交互问题

经典的延迟问题：当一端启用 Nagle，另一端启用 Delayed ACK 时，可能产生 40-200ms 的延迟：

1. 发送方发送小包 A（不足 MSS），等待 ACK 后发送小包 B
2. 接收方启用 Delayed ACK，延迟 40ms 等待是否有数据捎带 ACK
3. 发送方被 Nagle 阻塞，无法发送新数据
4. 延迟 40ms 后接收方才发送 ACK，发送方才继续

解决方案：对延迟敏感的应用设置 `TCP_NODELAY`

##  0x0D    TCP 内存管理与背压

TCP 的内存管理涉及发送缓冲区限制、TSQ 机制和全局内存压力，这些机制共同防止单个连接占用过多内存

####    发送缓冲区：sk_wmem_queued vs sk_sndbuf

```
 sk_sndbuf (发送缓冲区上限，sysctl: net.ipv4.tcp_wmem)
 ┃
 ┃  sk_wmem_queued (sk_write_queue 中所有 skb 的 truesize 之和)
 ┃  ┃
 ┃  ┃   增加：skb_entail 时 sk_wmem_queued += skb->truesize
 ┃  ┃   减少：tcp_clean_rtx_queue → sk_wmem_free_skb 时 sk_wmem_queued -= skb->truesize
 ┃  ┃
 ┃  当 sk_wmem_queued >= sk_sndbuf 时：
 ┃  → sk_stream_memory_free 返回 false
 ┃  → tcp_sendmsg 阻塞在 sk_stream_wait_memory
 ┃
 sk_wmem_alloc (已提交给协议栈/设备层的内存)
     增加：tcp_transmit_skb 中 atomic_add(skb->truesize, &sk->sk_wmem_alloc)
     减少：clone skb 释放时 tcp_wfree 中 atomic_sub
```

####    sk_stream_wait_memory：等待缓冲区释放

```cpp
// net/core/stream.c
int sk_stream_wait_memory(struct sock *sk, long *timeo_p)
{
    int err = 0;
    long vm_wait = 0;
    long current_timeo = *timeo_p;
    DEFINE_WAIT(wait);

    if (sk_stream_memory_free(sk))
        current_timeo = vm_wait = (prandom_u32() % (HZ / 5)) + 2;

    while (1) {
        set_bit(SOCK_ASYNC_NOSPACE, &sk->sk_socket->flags);

        prepare_to_wait(sk_sleep(sk), &wait, TASK_INTERRUPTIBLE);

        if (sk->sk_err || (sk->sk_shutdown & SEND_SHUTDOWN))
            goto do_error;
        if (!*timeo_p) {
            err = -EAGAIN;
            goto do_nonblock;
        }
        if (signal_pending(current)) {
            err = sock_intr_errno(*timeo_p);
            goto do_interrupted;
        }
        sk_clear_bit(SOCKWQ_ASYNC_NOSPACE, sk);

        if (sk_stream_memory_free(sk) && !vm_wait)
            break;

        // 进入睡眠
        set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);
        sk->sk_write_pending++;
        sk_wait_event(sk, &current_timeo, sk_stream_memory_free(sk) &&
                          vm_wait);
        sk->sk_write_pending--;

        if (vm_wait) {
            vm_wait -= current_timeo;
            current_timeo = *timeo_p;
            if (current_timeo != MAX_SCHEDULE_TIMEOUT &&
                (current_timeo -= vm_wait) < 0)
                current_timeo = 0;
            vm_wait = 0;
        }
        *timeo_p = current_timeo;
    }

    finish_wait(sk_sleep(sk), &wait);
    return err;
    /* ...... */
}
```

唤醒路径：`tcp_ack` → `tcp_clean_rtx_queue` → `sk_wmem_free_skb` → `sk_wmem_queued` 减少 → `tcp_check_space` → `sk_stream_write_space` → 唤醒睡眠进程

####    TSQ（TCP Small Queues）机制

TSQ 防止单个 TCP 连接在 qdisc/设备队列中积压过多数据，从而降低其他连接的延迟：

```cpp
// net/ipv4/tcp_output.c

// 自动 cork 判断：如果 qdisc 中已有数据且当前 skb 不足 MSS，延迟发送
static bool tcp_should_autocork(struct sock *sk, struct sk_buff *skb,
                int size_goal)
{
    return skb->len < size_goal &&
           sysctl_tcp_autocorking &&
           skb != tcp_write_queue_head(sk) &&
           atomic_read(&sk->sk_wmem_alloc) > SKB_TRUESIZE(1);
}
```

TSQ 工作流程：
1. `tcp_transmit_skb` 发送 clone skb 时设置 `skb->destructor = tcp_wfree`
2. 当 clone skb 在设备层被释放时，`tcp_wfree` 被调用
3. `tcp_wfree` 检查 `sk_wmem_alloc` 是否低于阈值
4. 如果是，触发 `tcp_tsq_handler` → `tcp_write_xmit`，继续发送更多数据

TSQ 的限制阈值默认为 `sysctl_tcp_limit_output_bytes`（`net.ipv4.tcp_limit_output_bytes`，默认 256KB）

####    关键 sysctl 参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `net.ipv4.tcp_wmem` | 4096 16384 4194304 | TCP 发送缓冲区（min / default / max） |
| `net.ipv4.tcp_rmem` | 4096 87380 6291456 | TCP 接收缓冲区（min / default / max） |
| `net.ipv4.tcp_mem` | 系统计算 | TCP 全局内存限制（页数：low / pressure / high） |
| `net.ipv4.tcp_limit_output_bytes` | 262144 | TSQ 限制：单连接在设备层的最大积压 |
| `net.ipv4.tcp_autocorking` | 1 | 自动 cork 开关 |
| `net.ipv4.tcp_notsent_lowat` | -1 | 未发送数据低水位（epoll 可写通知阈值） |

```bash
# 增大 TCP 发送缓冲区
$ sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"

# 增大 TCP 接收缓冲区
$ sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"

# 调整 TSQ 限制（高吞吐场景）
$ sudo sysctl -w net.ipv4.tcp_limit_output_bytes=524288
```

##  0x0E    参考

-   [内核之旅（十）：内核数据包发送](https://pandaychen.github.io/2025/04/02/A-LINUX-KERNEL-TRAVEL-10/)
-   [内核之旅（十二）：内核视角下的 TCP 完整通信过程](https://pandaychen.github.io/2025/04/25/A-LINUX-KERNEL-TRAVEL-12/)
-   [Monitoring and Tuning the Linux Networking Stack: Sending Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-sending-data/)
-   [[内核源码] 网络协议栈 - write (tcp) 发送数据](https://wenfh2020.com/2021/08/19/kernel-tcp-write/)
-   [TCP的发送系列 - tcp_sendmsg()的实现](https://www.cnblogs.com/aiwz/p/6333235.html)
-   [25 张图，一万字，拆解 Linux 网络包发送过程](https://blog.csdn.net/zhangyanfei01/article/details/116725966)
-   [Linux 4.11.6 源码 - tcp.c](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp.c)
-   [Linux 4.11.6 源码 - tcp_output.c](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_output.c)
-   [Linux 4.11.6 源码 - tcp_input.c](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp_input.c)
