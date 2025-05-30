---
layout:     post
title:  Linux 内核之旅（十）：内核数据包发送
subtitle:   基础知识汇总与可观测
date:       2025-04-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言

##  0x01   报文发送过程
本小节使用以太网的物理网卡，以一个UDP包的发送过程作为示例，了解下具体的发包过程

####    socket层
1、`socket()`：创建一个UDP socket结构体，并初始化相应的UDP操作函数

2、`sendto(sock, ...)`：应用层的程序（Application）调用该函数开始发送数据包，该函数会进而调用`inet_sendmsg`

3、`inet_sendmsg`：该函数主要是检查当前socket有无绑定源端口，如果没有的话，调用`inet_autobind`分配一个，然后调用UDP层的函数

4、`inet_autobind`：该函数会调用socket上绑定的`get_port`函数获取一个可用端口，由于该socket是UDP的socket，所以`get_port`函数会调到UDP内核实现里面的相应函数

```TEXT
               +-------------+
               | Application |
               +-------------+
                     |
                     |
                     ↓
+------------------------------------------+
| socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP) |
+------------------------------------------+
                     |
                     |
                     ↓
           +-------------------+
           | sendto(sock, ...) |
           +-------------------+
                     |
                     |
                     ↓
              +--------------+
              | inet_sendmsg |
              +--------------+
                     |
                     |
                     ↓
             +---------------+
             | inet_autobind |
             +---------------+
                     |
                     |
                     ↓
               +-----------+
               | UDP layer |
               +-----------+

```

####    UDP层

```TEXT
                     |
                     |
                     ↓
              +-------------+
              | udp_sendmsg |
              +-------------+
                     |
                     |
                     ↓
          +----------------------+
          | ip_route_output_flow |
          +----------------------+
                     |
                     |
                     ↓
              +-------------+
              | ip_make_skb |
              +-------------+
                     |
                     |
                     ↓
         +------------------------+
         | udp_send_skb(skb, fl4) |
         +------------------------+
                     |
                     |
                     ↓
                +----------+
                | IP layer |
                +----------+
```

####    IP层

```TEXT
          |
          |
          ↓
   +-------------+
   | ip_send_skb |
   +-------------+
          |
          |
          ↓
  +-------------------+       +-------------------+       +---------------+
  | __ip_local_out_sk |------>| NF_INET_LOCAL_OUT |------>| dst_output_sk |
  +-------------------+       +-------------------+       +---------------+
                                                                  |
                                                                  |
                                                                  ↓
 +------------------+        +----------------------+       +-----------+
 | ip_finish_output |<-------| NF_INET_POST_ROUTING |<------| ip_output |
 +------------------+        +----------------------+       +-----------+
          |
          |
          ↓
  +-------------------+      +------------------+       +----------------------+
  | ip_finish_output2 |----->| dst_neigh_output |------>| neigh_resolve_output |
  +-------------------+      +------------------+       +----------------------+
                                                                   |
                                                                   |
                                                                   ↓
                                                           +----------------+
                                                           | dev_queue_xmit |
                                                           +----------------+
```

####    netdevice子系统

```TEXT
                          |
                          |
                          ↓
                   +----------------+
  +----------------| dev_queue_xmit |
  |                +----------------+
  |                       |
  |                       |
  |                       ↓
  |              +-----------------+
  |              | Traffic Control |
  |              +-----------------+
  | loopback              |
  |   or                  +--------------------------------------------------------------+
  | IP tunnels            ↓                                                              |
  |                       ↓                                                              |
  |            +---------------------+  Failed   +----------------------+         +---------------+
  +----------->| dev_hard_start_xmit |---------->| raise NET_TX_SOFTIRQ |- - - - >| net_tx_action |
               +---------------------+           +----------------------+         +---------------+
                          |
                          +----------------------------------+
                          |                                  |
                          ↓                                  ↓
                  +----------------+              +------------------------+
                  | ndo_start_xmit |              | packet taps(AF_PACKET) |
                  +----------------+              +------------------------+
```

####    Device Driver

##  0x02    内核数据发送：详细分析
![send-data](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/how-to-send-data-1.png)

##  0x0  参考
-   [Monitoring and Tuning the Linux Networking Stack: Sending Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-sending-data/)
-   [Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/)
-   [Linux网络 - 数据包的发送过程](https://segmentfault.com/a/1190000008926093)
-   [图解 Linux 网络包发送过程](https://zhuanlan.zhihu.com/p/373060740)