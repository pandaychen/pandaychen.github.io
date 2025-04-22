---
layout:     post
title:  Linux 内核之旅（八）：内核数据包接收
subtitle:   基础知识汇总
date:       2025-03-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
笔者最近在研究基于ebpf的网络协议栈可观测及tracing，本文对协议栈的数据处理基础做了若干总结

##  0x01   网卡的报文接收过程
一些背景知识：

-   网卡驱动是加载到内核中的模块，负责衔接网卡和内核的网络模块，驱动在加载的时候将自己注册进网络模块，当相应的网卡收到数据包时，网络模块会调用相应的驱动程序处理数据

![recv-arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/kernel_packet.jpg)

本小节使用以太网的物理网卡结合一个UDP packet的接收过程为例子描述下内核收包过程，如下：

一、阶段1：数据包从网卡到内存

下图展示了数据包（packet）如何进入内存，并被内核的网络模块开始处理

```TEXT
                   +-----+
                   |     |                            Memroy
+--------+   1     |     |  2  DMA     +--------+--------+--------+--------+
| Packet |-------->| NIC |------------>| Packet | Packet | Packet | ...... |
+--------+         |     |             +--------+--------+--------+--------+
                   |     |<--------+
                   +-----+         |
                      |            +---------------+
                      |                            |
                    3 | Raise IRQ                  | Disable IRQ
                      |                          5 |
                      |                            |
                      ↓                            |
                   +-----+                   +------------+
                   |     |  Run IRQ handler  |            |
                   | CPU |------------------>| NIC Driver |
                   |     |       4           |            |
                   +-----+                   +------------+
                                                   | 
                                                6  | Raise soft IRQ
                                                   |
                                                   ↓
```

0、系统初始化时，网卡驱动程序会向内核申请一块内存（RingBuffer），用于存储未来到达的网络数据包；网卡驱动程序将申请的RingBuffer地址告诉网卡

1、数据包由外部网络进入物理网卡，如果目的地址非该网卡，且该网卡未开启混杂（promiscuous）模式，该包会被网卡丢弃

2、网卡将数据包通过DMA（Direct Memory Access）的方式写入到指定的内存地址，该地址由网卡驱动分配并初始化（网卡会通过DMA将数据拷贝到RingBuffer中，此过程不需要cpu参与）

3、网卡通过硬件中断（IRQ）通知CPU，告诉CPU有数据来了，CPU必须最高优先级处理，否则数据待会存不下了

4、CPU根据中断表，调用已经注册的中断函数，这个中断函数会调到驱动程序（NIC Driver）中相应的函数（调用对应的网卡驱动硬中断处理程序）

5、网卡驱动被调用后，**网卡驱动先禁用网卡的中断**，表示驱动程序已经知道内存中有数据了，告诉网卡下次再收到数据包直接写内存就可以了，不要再通知CPU了，这样可以提高效率，避免CPU不停的被中断；然后启动对应的软中断函数

6、启动软中断，这步结束后，硬件中断处理函数就结束返回了。由于硬中断处理程序执行的过程中不能被中断，所以如果它执行时间过长，会导致CPU没法响应其它硬件的中断，于是内核引入软中断，这样可以将硬中断处理函数中耗时的部分移到软中断处理函数里面来慢慢处理

（软中断函数开始从RingBuffer中进行循环取包，并且封装为`sk_buff`，然后投递给网络协议栈进行处理；协议栈处理完成后数据就进入用户态的对应进程，进程就可以操作数据了）

二、阶段2：内核的网络模块

软中断会触发内核网络模块中的软中断处理函数，继续上面的流程

```TEXT
                                                     +-----+
                                             17      |     |
                                        +----------->| NIC |
                                        |            |     |
                                        |Enable IRQ  +-----+
                                        |
                                        |
                                  +------------+                                      Memroy
                                  |            |        Read           +--------+--------+--------+--------+
                 +--------------->| NIC Driver |<--------------------- | Packet | Packet | Packet | ...... |
                 |                |            |          9            +--------+--------+--------+--------+
                 |                +------------+
                 |                      |    |        skb
            Poll | 8      Raise softIRQ | 6  +-----------------+
                 |                      |             10       |
                 |                      ↓                      ↓
         +---------------+  Call  +-----------+        +------------------+        +--------------------+  12  +---------------------+
         | net_rx_action |<-------| ksoftirqd |        | napi_gro_receive |------->| enqueue_to_backlog |----->| CPU input_pkt_queue |
         +---------------+   7    +-----------+        +------------------+   11   +--------------------+      +---------------------+
                                                               |                                                      | 13
                                                            14 |        + - - - - - - - - - - - - - - - - - - - - - - +
                                                               ↓        ↓
                                                    +--------------------------+    15      +------------------------+
                                                    | __netif_receive_skb_core |----------->| packet taps(AF_PACKET) |
                                                    +--------------------------+            +------------------------+
                                                               |
                                                               | 16
                                                               ↓
                                                      +-----------------+
                                                      | protocol layers |
                                                      +-----------------+

```


7、内核中的`ksoftirqd`进程专门负责软中断的处理，当它收到软中断后，就会调用相应软中断所对应的处理函数，对于上面第`6`步中是网卡驱动模块抛出的软中断，`ksoftirqd`会调用网络模块的`net_rx_action`函数

8、`net_rx_action`会调用网卡驱动里的`poll`函数（对于igb网卡驱动来说，此[函数]()就是`igb_poll`）来一个个的处理数据包

9、在`poll`函数中，驱动会一个接一个的读取网卡写到内存中的数据包，内存中数据包的格式只有驱动知道

10、网卡驱动程序将内存中的数据包转换成内核网络模块能识别的`skb`格式，然后调用`napi_gro_receive`函数

11、`napi_gro_receive`会处理`GRO`（Generic Receive Offloading）相关的内容，也就是将可以合并的数据包进行合并，这样就只需要调用一次协议栈。然后判断是否开启了`RPS`（Receive Packet Steering），如果开启了，将会调用`enqueue_to_backlog`

12、在`enqueue_to_backlog`函数中，会将数据包放入CPU的`softnet_data`结构体的`input_pkt_queue`中，然后返回，如果`input_pkt_queue`满了的话，该数据包将会被丢弃，queue的大小可以通过`net.core.netdev_max_backlog`来配置（备注：`enqueue_to_backlog`函数也会被`netif_rx`函数调用，而`netif_rx`正是`lo`设备发送数据包时调用的函数）

13、CPU会接着在自己的软中断上下文中处理自己`input_pkt_queue`里的网络数据（调用`__netif_receive_skb_core`）

14、如果没开启`RPS`，`napi_gro_receive`会直接调用`__netif_receive_skb_core`

15、看是不是有`AF_PACKET`类型的socket（raw socket），如果有的话，拷贝一份数据给它（`tcpdump`即抓取来自这里的packet）

16、调用协议栈相应的函数，将数据包交给协议栈处理

17、待内存中的所有数据包被处理完成后（即`poll`函数执行完成），**会再次启用网卡的硬中断，这样下次网卡再收到数据的时候就会通知CPU**


3、阶段3：协议栈网络层（IP）

接着看下数据包来到协议栈的处理过程，重要的内核函数如下：

```TEXT
          |
          |
          ↓         promiscuous mode &&
      +--------+    PACKET_OTHERHOST (set by driver)   +-----------------+
      | ip_rcv |-------------------------------------->| drop this packet|
      +--------+                                       +-----------------+
          |
          |
          ↓
+---------------------+
| NF_INET_PRE_ROUTING |
+---------------------+
          |
          |
          ↓
      +---------+
      |         | enabled ip forword  +------------+        +----------------+
      | routing |-------------------->| ip_forward |------->| NF_INET_FORWARD |
      |         |                     +------------+        +----------------+
      +---------+                                                   |
          |                                                         |
          | destination IP is local                                 ↓
          ↓                                                 +---------------+
 +------------------+                                       | dst_output_sk |
 | ip_local_deliver |                                       +---------------+
 +------------------+
          |
          |
          ↓
 +------------------+
 | NF_INET_LOCAL_IN |
 +------------------+
          |
          |
          ↓
    +-----------+
    | UDP layer |
    +-----------+
```

-   `ip_rcv`：此函数是IP模块的入口函数，该函数第一件事就是将垃圾数据包（目的mac地址不是当前网卡，但由于网卡设置了混杂模式而被接收进来）直接丢掉，然后调用注册在`NF_INET_PRE_ROUTING`上的函数
-   `NF_INET_PRE_ROUTING`： netfilter注册在协议栈中的钩子，可以通过`iptables`来注入一些数据包处理函数，用来修改或者丢弃数据包，如果数据包没被丢弃，将继续往下走
-   `routing`： 进行路由，如果是目的IP不是本地IP，且没有开启ip forward功能，那么数据包将被丢弃；如果开启了ip forward功能，那将进入`ip_forward`函数（转发模式常用）
-   `ip_forward`： `ip_forward`会先调用netfilter注册的`NF_INET_FORWARD`相关函数，如果数据包没有被丢弃，那么将继续往后调用`dst_output_sk`函数
-   `dst_output_sk`： 该函数会调用IP层的相应函数将该数据包发送出去（参考协议栈数据包发送流程的后半部分）
-   `ip_local_deliver`：如果上面`routing`的时候发现目的IP是本地IP，那么将会调用该函数。该函数会先调用`NF_INET_LOCAL_IN`相关的钩子程序，如果通过，数据包将会向下发送到UDP层

四、阶段4：协议栈传输层（UDP）

```TEXT
          |
          |
          ↓
      +---------+            +-----------------------+
      | udp_rcv |----------->| __udp4_lib_lookup_skb |
      +---------+            +-----------------------+
          |
          |
          ↓
 +--------------------+      +-----------+
 | sock_queue_rcv_skb |----->| sk_filter |
 +--------------------+      +-----------+
          |
          |
          ↓
 +------------------+
 | __skb_queue_tail |
 +------------------+
          |
          |
          ↓
  +---------------+
  | sk_data_ready |
  +---------------+

```

-   `udp_rcv`： 此[函数](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/udp.c#L2105)是UDP模块的入口函数，它里面会调用其它的函数，主要是做一些必要的检查，其中一个重要的调用是`__udp4_lib_lookup_skb`，该函数会根据目的IP和端口找对应的socket，如果没有找到相应的socket，那么该数据包将会被丢弃，否则继续（关联hook点`kprobe:udp_rcv`）
-   `sock_queue_rcv_skb`： 该函数会检查这个socket的receive buffer是不是满了，如果满了的话，丢弃该数据包，然后就是调用`sk_filter`看这个包是否是满足条件的包，如果当前socket上设置了filter，且该包不满足条件的话，这个数据包也将被丢弃（在Linux里面，每个socket上都可以像tcpdump里面一样定义filter，不满足条件的数据包将会被丢弃）
-   `__skb_queue_tail`： 将数据包放入socket接收队列的末尾
-   `sk_data_ready`： 通知socket数据包已经准备好；调用完`sk_data_ready`之后，一个数据包处理完成，等待应用层程序来读取，上面所有函数的执行过程都在软中断的上下文中

五、阶段5：socket应用程序

应用层一般有两种方式接收数据：
-   `recvfrom`函数阻塞等待数据到来，这种情况下当socket收到通知后，`recvfrom`就会被唤醒，然后读取接收队列的数据
-   通过`epoll`/`select`监听相应的socket，当收到通知后，再调用`recvfrom`函数去读取接收队列的数据

至此，一个UDP包就经由网卡成功送到了应用层程序


####    网卡收包的若干细节


##  0x02  核心数据结构
-   `struct socket`：传输层使用的[数据结构](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/net.h#L111)，用于声明、定义套接字
-   `struct sock`：网络层会调用`struct sock`[结构体](https://elixir.bootlin.com/linux/v4.11.6/source/include/net/sock.h#L311)，该结构体包含`struct sock_common`[结构体](https://elixir.bootlin.com/linux/v4.11.6/source/include/net/sock.h#L120)
-   `struct sk_buff`：内核中使用的套接字缓冲区[结构体](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/skbuff.h#L566)，套接字结构体用于表示一个网络连接对应的本地接口的网络信息，而`sk_buff`结构则是该网络连接对应的数据包的存储

####    struct sk_buff结构： 套接字缓冲区
对于ebpf应用开发者而言，最关注的结构莫过于`sk_buff`了。`sk_buff`用来管理和控制接收OR发送数据包的信息，各层协议都依赖于`sk_buff`而存在。内核中`sk_buff`结构体在各层协议之间传输不是用拷贝`sk_buff`结构体，而是通过增加协议头和移动指针来操作的。如果是从L4传输到L2，则是通过往`sk_buff`结构体中增加该层协议头来操作；如果是从L4到L2，则是通过移动`sk_buff`结构体中的data指针来实现，不会删除各层协议头


```C
struct sk_buff {
	union {
		struct {
			/* These two members must be first. 构成sk_buff链表*/
			struct sk_buff		*next;
			struct sk_buff		*prev;
			union {
				struct net_device	*dev;	//网络设备对应的结构体，很重要但是不是本文重点，所以不做展开
				/* Some protocols might use this space to store information,
				 * while device pointer would be NULL.
				 * UDP receive path is one user.
				 */
				unsigned long		dev_scratch;   // 对于某些不适用net_device的协议需要采用该字段存储信息，如UDP的接收路径
			};
		};
		struct rb_node		rbnode; /* used in netem, ip4 defrag, and tcp stack 将sk_buff以红黑树组织，在TCP中有用到*/
		struct list_head	list;   // sk_buff链表头指针（5.10内核后）
	};
	union {
		struct sock		*sk;       // 指向网络层套接字结构体
		int			ip_defrag_offset;   //用来处理IPv4报文分片
	};
	union {
		ktime_t		tstamp;    // 时间戳
		u64		skb_mstamp_ns; /* earliest departure time */
	};
	/* 存储私有信息
	 * This is the control buffer. It is free to use for every
	 * layer. Please put your private variables there. If you
	 * want to keep them across layers you have to do a skb_clone()
	 * first. This is owned by whoever has the skb queued ATM.
	 */
	char			cb[48] __aligned(8);
	union {
		struct {
			unsigned long	_skb_refdst;				   // 目标entry
			void		(*destructor)(struct sk_buff *skb);	// 析构函数
		};
		struct list_head	tcp_tsorted_anchor;			    // TCP发送队列(tp->tsorted_sent_queue)
	};
....
	unsigned int		len,	// 实际长度
				data_len;	    // 数据长度
	__u16			mac_len,    // mac层长度
				hdr_len;        // 可写头部长度
	/* Following fields are _not_ copied in __copy_skb_header()
	 * Note that queue_mapping is here mostly to fill a hole.
	 */
	__u16			queue_mapping;   // 多队列设备的队列映射
......
	/* fields enclosed in headers_start/headers_end are copied
	 * using a single memcpy() in __copy_skb_header()
	 */
	/* private: */
	__u32			headers_start[0];	
	/* public: */
......
	__u8			__pkt_type_offset[0];
	__u8			pkt_type:3;
	__u8			ignore_df:1;
	__u8			nf_trace:1;
	__u8			ip_summed:2;
	__u8			ooo_okay:1;
	__u8			l4_hash:1;
	__u8			sw_hash:1;
	__u8			wifi_acked_valid:1;
	__u8			wifi_acked:1;
	__u8			no_fcs:1;
	/* Indicates the inner headers are valid in the skbuff. */
	__u8			encapsulation:1;
	__u8			encap_hdr_csum:1;
	__u8			csum_valid:1;
......
	__u8			__pkt_vlan_present_offset[0];
	__u8			vlan_present:1;
	__u8			csum_complete_sw:1;
	__u8			csum_level:2;
	__u8			csum_not_inet:1;
	__u8			dst_pending_confirm:1;
......
	__u8			ipvs_property:1;
	__u8			inner_protocol_type:1;
	__u8			remcsum_offload:1;
......
	union {
		__wsum		csum;
		struct {
			__u16	csum_start;
			__u16	csum_offset;
		};
	};
	__u32			priority;
	int			skb_iif;		// 接收到该数据包的网络接口的编号
	__u32			hash;
	__be16			vlan_proto;
	__u16			vlan_tci;
......
	union {
		__u32		mark;
		__u32		reserved_tailroom;
	};
	union {
		__be16		inner_protocol;
		__u8		inner_ipproto;
	};
	__u16			inner_transport_header;
	__u16			inner_network_header;
	__u16			inner_mac_header;
	__be16			protocol;
	__u16			transport_header;	// 传输层头部
	__u16			network_header;		// 网络层头部
	__u16			mac_header;			// mac层头部
	/* private: */
	__u32			headers_end[0];
	/* public: */
	/* These elements must be at the end, see alloc_skb() for details.  */
	sk_buff_data_t		tail;
	sk_buff_data_t		end;
	unsigned char		*head, *data;
	unsigned int		truesize;
	refcount_t		users;
......
};
```

####    `struct sk_buff`成员&&管理
`sk_buff`的成员主要关注如下三类：

-   组织布局（Layout）
-   通用数据成员（General）
-   管理`sk_buff`结构体的函数（Management functions）

1、`sk_buff`的管理组织方式

通常`sk_buff`使用双链表`sk_buff_head`结构进行管理（并且`sk_buff`需要能在`O(1)`时间内获得双链表的头节点），布局如下图所示

```CPP
//有些情况下sk_buff不是用双链表而是用红黑树组织的，那么有效的域是rbnode
//5.10内核中，list域是一个list_head结构体，而非sk_buff_head
struct sk_buff_head {
	/* These two members must be first. */
	struct sk_buff	*next;  //sk_buff中的next、prev指向相邻的sk_buff
	struct sk_buff	*prev;

	__u32		qlen;   //链表中元素的数量
	spinlock_t	lock;   //并发访问时保护
};
```

![layout1]()

2、`sk_buff`的线性空间管理

```CPP
/* These elements must be at the end, see alloc_skb() for details.  */
	sk_buff_data_t		tail;
	sk_buff_data_t		end;
	unsigned char		*head,*data;
```

从下图中可以发现，`head`与`end`指向的位置始终不变，数据的变化、协议头的添加都是依靠`tail`与`data`的移动来实现的；此外，初始化时，`head`、`data`、`tail`都指向开始位置

![layout2]()

2、重要成员

-   `struct sock *sk`：指向拥有该`sk_buff`的套接字（即skb 关联的socket），当这个包是socket 发出或接收时，这里指向对应的socket，而如果是转发包，这里是`NULL`；在可观测场景下，当socket已经建立时，可以用来解析获取传输层的关键信息
-   `unsigned int truesize`：表示skb使用的大小，包括skb结构体以及它所指向的数据
-   `unsigned int		len`：所有数据的长度之和，包括分片的数据以及协议头
-   `unsigned int      data_len`：分片的数据长度
-   `__u16			mac_len`：链路层帧头长度
-   `__u16 hdr_len`：被copy的skb中可写的头部长度

3、General成员，即skb的通用成员，与协议类型或内核特性无关，这里也列举几个

`struct net_device	*dev`成员，用来表示从哪个设备收到报文，或将把报文发到哪个设备

```CPP
//include/linux/skbuff.h
			union {
				struct net_device	*dev;
				/* Some protocols might use this space to store information,
				 * while device pointer would be NULL.
				 * UDP receive path is one user.
				 */
				unsigned long		dev_scratch;
			};
```

`char cb[48]`成员，是skb能被各层共用的精髓，`48`即为TCP的控制块`tcp_sbk_cb`[数据结构](https://elixir.bootlin.com/linux/v4.11.6/source/include/net/tcp.h#L736)的size

```CPP
//include/linux/skbuff.h
	/*
	 * This is the control buffer. It is free to use for every
	 * layer. Please put your private variables there. If you
	 * want to keep them across layers you have to do a skb_clone()
	 * first. This is owned by whoever has the skb queued ATM.
	 */
	char			cb[48] __aligned(8);
```

`tcp_sbk_cb`结构如下，可以通过`TCP_SKB_CB`这个宏获取`cb`字段（被转换为`struct tcp_skb_cb *`类型）
```CPP
//include/net/tcp.h
#define TCP_SKB_CB(__skb)	((struct tcp_skb_cb *)&((__skb)->cb[0]))
//include/net/tcp.h
struct tcp_skb_cb {
	__u32		seq;		/* Starting sequence number	*/
	__u32		end_seq;	/* SEQ + FIN + SYN + datalen	*/

	/* ... */
	__u8		tcp_flags;	/* TCP header flags. (tcp[13])	*/

	/* ... */
};
```

`pkt_type`这个字段用于表示数据包类型，此类型是由目标MAC地址决定的

```CPP
//include/linux/skbuff.h
	__u8			pkt_type:3;


#define PACKET_HOST		0		/* To us		*/
#define PACKET_BROADCAST	1		/* To all		*/
#define PACKET_MULTICAST	2		/* To group		*/
#define PACKET_OTHERHOST	3		/* To someone else 	*/
#define PACKET_OUTGOING		4		/* Outgoing of any type */
#define PACKET_LOOPBACK		5		/* MC/BRD frame looped back */
#define PACKET_USER		6		/* To user space	*/
#define PACKET_KERNEL		7		/* To kernel space	*/
```

`__be16 protocol`这个字段标识了L2上层的协议类型，典型的协议类型如下：

```CPP
//include/uapi/linux/if_ether.h
/*
 *	These are the defined Ethernet Protocol ID's.
 */


#define ETH_P_IP	0x0800		/* Internet Protocol packet	*/
#define ETH_P_X25	0x0805		/* CCITT X.25			*/
#define ETH_P_ARP	0x0806		/* Address Resolution packet	*/

#define ETH_P_IPV6	0x86DD		/* IPv6 over bluebook		*/
```

`transport_header`、`network_header`、`mac_header`这三个字段标识了各层头部相对于`head`的偏移量，在ebpf对`sk_buff`结构的解析中也非常常用

```cpp
//include/linux/skbuff.h
	__u16			transport_header;
	__u16			network_header;
	__u16			mac_header;
```

3、管理函数（Management functions）

3.1、分配和释放内存，通过`alloc_skb`获取一个`struct sk_buff`加长度为`size`（经过对齐）的数据缓冲区

```CPP
//net/core/skbuff.c
static inline struct sk_buff *alloc_skb(unsigned int size, gfp_t priority){
    //....
	skb = kmem_cache_alloc_node(cache, gfp_mask & ~__GFP_DMA, node);
	/* ... */
	size = SKB_DATA_ALIGN(size);
	size += SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
	data = kmalloc_reserve(size, gfp_mask, node, &pfmemalloc);
	/* kmalloc(size) might give us more room than requested.
	 * Put skb_shared_info exactly at the end of allocated zone,
	 * to allow max possible filling before reallocation.
	 */
	size = SKB_WITH_OVERHEAD(ksize(data));
	/* ... */
	skb->head = data;
	skb->data = data;
	skb_reset_tail_pointer(skb);
	skb->end = skb->tail + size;
    //...
}
```

3.2、`sk_buff`数据缓冲区指针操作

![]()

3.3、接收数据包的处理过程

3.4、发送数据包的处理过程

由下图描述了发送数据包时skb缓冲区被填满的过程

![sk_buff]()


线性数据区的创建过程

`sk_buff`结构数据区初始化成功后，此时 `head` 指针、`data` 指针、`tail` 指针都是指向同一个地方（**`head` 指针和 `end` 指针指向的位置一直都不变，而对于数据的变化和协议信息的添加都是通过 `data` 指针和 `tail` 指针的改变来表现的**）


####    struct socket结构
每个`struct socket`结构都有一个`struct sock`结构成员，`sock`是对`socket`的扩充，`socket->sk`指向对应的`sock`结构，`sock->socket` 指向对应的`socket`结构

```CPP
struct socket {
	socket_state		state;  //链接状态
	short			type;       //套接字类型，如SOCK_STREAM等
	unsigned long		flags;
	struct socket_wq	*wq;
	struct file		*file;      //套接字对应的文件指针，毕竟Linux一切皆文件
	struct sock		*sk;        //网络层的套接字
	const struct proto_ops	*ops;
};
```

####    struct sock结构

```CPP
struct sock {
	struct sock_common	__sk_common;	   // 网络层套接字通用结构体
......
	socket_lock_t		sk_lock;	       // 套接字同步锁
	atomic_t		sk_drops;	           // IP/UDP包丢包统计
	int			sk_rcvlowat;        	   // SO_RCVLOWAT标记位
......
	struct sk_buff_head	sk_receive_queue;	// 收到的数据包队列
......
	int			sk_rcvbuf;				  // 接收缓存大小
......
	union {
		struct socket_wq __rcu	*sk_wq;	    // 等待队列
		struct socket_wq	*sk_wq_raw;
	};
......
	int			sk_sndbuf;			       // 发送缓存大小
	/* ===== cache line for TX ===== */
	int			sk_wmem_queued;			   // 传输队列大小
	refcount_t		sk_wmem_alloc;		    // 已确认的传输字节数
	unsigned long		sk_tsq_flags;	    // TCP Small Queue标记位
	union {
		struct sk_buff	*sk_send_head;		// 发送队列对首
		struct rb_root	tcp_rtx_queue;		 
	};
	struct sk_buff_head	sk_write_queue;		 // 发送队列
......
	u32			sk_pacing_status; /* see enum sk_pacing 发包速率控制状态*/ 
	long			sk_sndtimeo;		    // SO_SNDTIMEO 标记位
	struct timer_list	sk_timer;			// 套接字清空计时器
	__u32			sk_priority;		    // SO_PRIORITY 标记位
......
	unsigned long		sk_pacing_rate; /* bytes per second 发包速率*/
	unsigned long		sk_max_pacing_rate;  // 最大发包速率
	struct page_frag	sk_frag;		    // 缓存页帧
......
	struct proto		*sk_prot_creator;
	rwlock_t		sk_callback_lock;
	int			sk_err,					  // 上次错误
				sk_err_soft;			  // “软”错误：不会导致失败的错误
	u32			sk_ack_backlog;			   // ack队列长度
	u32			sk_max_ack_backlog;		   // 最大ack队列长度
	kuid_t			sk_uid;				  // user id
	struct pid		*sk_peer_pid;		   // 套接字对应的peer的id
......
	long			sk_rcvtimeo;		  // 接收超时
	ktime_t			sk_stamp;			  // 时间戳
......
	struct socket		*sk_socket;		   // Identd协议报告IO信号
	void			*sk_user_data;		  // RPC层私有信息
......
	struct sock_cgroup_data	sk_cgrp_data;   // cgroup数据
	struct mem_cgroup	*sk_memcg;		   // 内存cgroup关联
	void			(*sk_state_change)(struct sock *sk);	// 状态变化回调函数
	void			(*sk_data_ready)(struct sock *sk);		// 数据处理回调函数
	void			(*sk_write_space)(struct sock *sk);		// 写空间可用回调函数
	void			(*sk_error_report)(struct sock *sk);    // 错误报告回调函数
	int			(*sk_backlog_rcv)(struct sock *sk, struct sk_buff *skb);	// 处理存储区回调函数
......
	void                    (*sk_destruct)(struct sock *sk);	// 析构回调函数
	struct sock_reuseport __rcu	*sk_reuseport_cb;			   // group容器重用回调函数
......
};
```

####    struct sock_common结构

`struct sock_common`是套接口在网络层的最小表示，即最基本的网络层套接字信息
```CPP
struct sock_common {
	/* skc_daddr and skc_rcv_saddr must be grouped on a 8 bytes aligned
	 * address on 64bit arches : cf INET_MATCH()
	 */
	union {
		__addrpair	skc_addrpair;
		struct {
			__be32	skc_daddr;		// 外部/目的IPV4地址
			__be32	skc_rcv_saddr;	// 本地绑定IPV4地址
		};
	};
	union  {
		unsigned int	skc_hash;	// 根据协议查找表获取的哈希值
		__u16		skc_u16hashes[2]; // 2个16位哈希值，UDP专用
	};
	/* skc_dport && skc_num must be grouped as well */
	union {
		__portpair	skc_portpair;	// 
		struct {
			__be16	skc_dport;	    // inet_dport占位符
			__u16	skc_num;	    // inet_num占位符
		};
	};
	unsigned short		skc_family;	      // 网络地址family
	volatile unsigned char	skc_state;    // 连接状态
	unsigned char		skc_reuse:4;      // SO_REUSEADDR 标记位
	unsigned char		skc_reuseport:1;  // SO_REUSEPORT 标记位
	unsigned char		skc_ipv6only:1;   // IPV6标记位
	unsigned char		skc_net_refcnt:1; // 该套接字网络名字空间内引用数
	int			skc_bound_dev_if;		 // 绑定设备索引
	union {
		struct hlist_node	skc_bind_node;     // 不同协议查找表组成的绑定哈希表
		struct hlist_node	skc_portaddr_node; // UDP/UDP-Lite protocol二级哈希表
	};
	struct proto		*skc_prot;			  // 协议回调函数，根据协议不同而不同
......
	union {									
		struct hlist_node	skc_node;		    // 不同协议查找表组成的主哈希表
		struct hlist_nulls_node skc_nulls_node;  // UDP/UDP-Lite protocol主哈希表
	};
	unsigned short		skc_tx_queue_mapping;    // 该连接的传输队列
	unsigned short		skc_rx_queue_mapping;    // 该连接的接受队列
......
	union {
		int		skc_incoming_cpu; // 多核下处理该套接字数据包的CPU编号
		u32		skc_rcv_wnd;	  // 接收窗口大小
		u32		skc_tw_rcv_nxt; /* struct tcp_timewait_sock  */
	};
	refcount_t		skc_refcnt;   // 套接字引用计数
......
};

```

####    小结
网络通信中通过网卡获取到的数据包至少包括了物理层，链路层和网络层的内容，因此套接字结构体仅仅从网络层开始，即通常只定义了传输层的套接字`socket`和网络层的套接字`sock`。`socket` 是用于负责对上给用户提供接口，并且和文件系统关联。而 `sock`负责向下对接内核网络协议栈

从传输层到链路层，它是存放数据的通用结构，为了保持高效率，数据在传递过程中尽量不发生额外的拷贝。因此，从高层到低层的时候，会不断地在数据前加头，因此每一层的协议都会调用skb_reserve，为自己的报头预留空间。至于从低层到高层，去掉低层报头的方式就是移动一下指针，指向高层头，非常简单。


##  0x0  参考
-   [Monitoring and Tuning the Linux Networking Stack: Sending Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-sending-data/)
-   [Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/)
-   [Linux网络 - 数据包的发送过程](https://segmentfault.com/a/1190000008926093)
-   [Linux 内核数据流概览初探](http://klworldy.com/posts/kerneldataflow/)
-   [Linux是怎么从网络上接收数据包的](https://mp.weixin.qq.com/s/gVYdRWCoQwpFtsKXMyIHRg)
-   [linux网络子系统研究：数据收发简略流程图](https://www.latelee.cn/net-study/linux-network-data-recv-send.html)
-   [eBPF Docs Program context '__sk_buff'](https://docs.ebpf.io/linux/program-context/__sk_buff/)
-   [25 张图，一万字，拆解 Linux 网络包发送过程](https://mp.weixin.qq.com/s?__biz=Mzg3ODUxNzM0MA==&mid=2247484532&idx=2&sn=2a934f8d6ae87b53b62671972987388d&scene=21#wechat_redirect)
-   [Linux网络 - 数据包的接收过程](https://segmentfault.com/a/1190000008836467)
-   [Linux 网卡数据收发过程分析](https://mp.weixin.qq.com/s?__biz=MzA3NzYzODg1OA==&mid=2648464515&idx=1&sn=117fa172f80cda8f446a1eb5e191464f&scene=21#wechat_redirect)
-   [图解Linux网络包接收过程](https://cloud.tencent.com/developer/article/1966873)
-   [Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/illustrated-guide-monitoring-tuning-linux-networking-stack-receiving-data/)
-   [内核网络中的GRO、RFS、RPS技术介绍和调优](https://www.kerneltravel.net/blog/2020/network_ljr9/)
-   <<深入理解Linux网络技术内幕>>
-   [sk_buff 简介](https://www.llcblog.cn/2020/10/26/how-sk-buff-work/)
-   [【Linux】网络专题（二）——核心数据结构sk_buff](https://void-star.icu/archives/939)