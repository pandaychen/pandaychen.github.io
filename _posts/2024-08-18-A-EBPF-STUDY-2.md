---
layout:     post
title:      golang eBPF 开发入门（三）
subtitle:   eBPF 中的 Maps 数据结构分析与应用
date:       2024-08-18
author:     pandaychen
catalog:    true
tags:
    - eBPF
    - Golang
---


##  0x00    前言
[上文](https://pandaychen.github.io/2024/01/15/A-EBPF-STUDY/#0x03--%E9%80%9A%E4%BF%A1%E5%86%85%E6%A0%B8%E6%80%81%E4%B8%8E%E7%94%A8%E6%88%B7%E6%80%81) 介绍了 ebpf 的内核态 / 用户态的通信示例，本文关注三个问题：为什么要使用 maps？maps 使用的一般场景？maps 的基本结构及实现原理。BPF Map 是内核空间和用户空间之间用于数据交换、信息传递的桥梁。是 eBPF 程序中使用的主要数据结构。BPF Map 本质上是以键 / 值方式存储在内核中的数据结构，在内核空间的程序创建 BPF Map 并返回对应的文件描述符，在用户空间运行的程序就可以通过这个文件描述符来访问并操作 BPF Map，从而实现 kernel 内核空间与 user 用户空间的数据交换

##  0x01    BPF maps 基础

![ebpfworkflow]()

![map-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ebpf/ebpf-map-1.png)

map 驻留在内存中，可以通过用户空间的文件描述符 `fd` 访问，可以在任意 BPF 程序以及用户空间应用之间共享。共享 map 的 BPF 程序不需要是相同的程序类型，单个 BPF 程序最多可以访问 `64` 个 map。map 有 per-CPU 以及 non-per-CPU 的通用 map，另外还有提供给特定辅助函数的非通用 map。通用 map 包括 `BPF_MAP_TYPE_HASH`/ `BPF_MAP_TYPE_ARRAY`/ `BPF_MAP_TYPE_PERCPU_HASH`/ `BPF_MAP_TYPE_PERCPU_ARRAY`/`BPF_MAP_TYPE_LRU_HASH`/ `BPF_MAP_TYPE_LRU_PERCPU_HASH` 和 `BPF_MAP_TYPE_LPM_TRIE`，它们都使用相同的一组 BPF 辅助函数来执行 CRUD 操作，非通用 map 包括 `BPF_MAP_TYPE_PROG_ARRAY`/`BPF_MAP_TYPE_PERF_EVENT_ARRAY`/`BPF_MAP_TYPE_CGROUP_ARRAY`/`BPF_MAP_TYPE_STACK_TRACE`/`BPF_MAP_TYPE_ARRAY_OF_MAPS`/`BPF_MAP_TYPE_HASH_OF_MAPS`。例如 `BPF_MAP_TYPE_PROG_ARRAY` 用于存储其他 BPF 程序，`BPF_MAP_TYPE_ARRAY_OF_MAPS` 用于存储其他 map 的指针，非通用 map 都是为了与 bpf help 一起完成特定功能

##  0x02 BPF Map 数据结构
每个 BPF Map 由四个值定义：类型、元素的最大个数、值大小 (Byte) 和键大小 (Byte)

```C
struct {
	__uint(type, BPF_MAP_TYPE_HASH);   // 类型
	__uint(max_entries, 8192);         // 元素的最大个数
	__type(key, pid_t);                // 键大小 (以字节为单位)
	__type(value, u64);                // 值大小 (以字节为单位)
} rb SEC(".maps");                     // SEC(".maps") 声明并创建了一个名为 rb 的 BPFMap
```

所有类型的 [定义](https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h#L130)，需要注意类型对应的内核版本 or patch（开发）

```c
enum bpf_map_type {
    BPF_MAP_TYPE_UNSPEC,
    BPF_MAP_TYPE_HASH,             // 一种哈希表
    BPF_MAP_TYPE_ARRAY,            // 一种为快速查找速度而优化的数组类型 map 键值对，通常用于计数器
    BPF_MAP_TYPE_PROG_ARRAY,       // 与 eBPF 程序相对应的一种文件描述符数组; 用于实现跳转表和处理特定（网络）包协议的子程序
    BPF_MAP_TYPE_PERF_EVENT_ARRAY, // 存储指向 perf_event 数据结构的指针，用于读取和存储 perf 事件计数器
    BPF_MAP_TYPE_PERCPU_HASH,      // 一种基于每个 CPU 的哈希表
    BPF_MAP_TYPE_PERCPU_ARRAY,     // 一种基于每个 cpu 的数组，用于实现展现延迟的直方图
    BPF_MAP_TYPE_STACK_TRACE,      // 存储堆栈跟踪信息
    BPF_MAP_TYPE_CGROUP_ARRAY,     // 存储指向控制组的指针
    BPF_MAP_TYPE_LRU_HASH,         // 一种只保留最近使用项的哈希表
    BPF_MAP_TYPE_LRU_PERCPU_HASH,  // 一种基于每个 CPU 的哈希表，只保留最近使用项
    BPF_MAP_TYPE_LPM_TRIE,         // 一个匹配最长前缀的字典树数据结构，适用于将 IP 地址匹配到一个范围
    BPF_MAP_TYPE_ARRAY_OF_MAPS,    // 一种 map-in-map 数据结构
    BPF_MAP_TYPE_HASH_OF_MAPS,     // 一种 map-in-map 数据结构
    BPF_MAP_TYPE_DEVMAP,
    BPF_MAP_TYPE_SOCKMAP,
    BPF_MAP_TYPE_CPUMAP,
    BPF_MAP_TYPE_XSKMAP,
    BPF_MAP_TYPE_SOCKHASH,
    BPF_MAP_TYPE_CGROUP_STORAGE,
    BPF_MAP_TYPE_REUSEPORT_SOCKARRAY,
    BPF_MAP_TYPE_PERCPU_CGROUP_STORAGE,
    BPF_MAP_TYPE_QUEUE,
    BPF_MAP_TYPE_STACK,
    BPF_MAP_TYPE_SK_STORAGE,
    BPF_MAP_TYPE_DEVMAP_HASH,
    BPF_MAP_TYPE_STRUCT_OPS,
    BPF_MAP_TYPE_RINGBUF,          // 一种高性能字节形缓冲区
    BPF_MAP_TYPE_INODE_STORAGE,
};
```

####    Hash Maps
内核代码在 [kernel/bpf/hashtab.c]()
```bash
BPF_MAP_TYPE_HASH
BPF_MAP_TYPE_PERCPU_HASH
BPF_MAP_TYPE_LRU_HASH
BPF_MAP_TYPE_LRU_PERCPU_HASH
BPF_MAP_TYPE_HASH_OF_MAPS
```

汇总下，Hash map 的特点如下：
-   key 的长度没有限制
-   给定 key 查找 value 时，内部通过哈希实现，而非数组索引，此机制 key/value 是可删除的；作为对比，Array 类型的 map 中，key/value 是不可删除的（但用空值覆盖掉 value ，可实现删除效果）。原因其实也很简单：哈希表是链表，可以删除链表中的元素；array 是内存空间连续的数组，即使某个 `index` 处的 value 不用了，这段内存区域还是得留着，不可能将其释放

####    PERCPU VS NONE-PERCPU
NONE-PERCPU 是 global 的，只有一个实例；PERCPU 是 cpu-local 的，每个 CPU 上都有一个 map 实例；多核并发访问时，global map 需要加锁；per-cpu map 无需加锁，每个核上的程序访问 local-cpu 上的 map

####    Array Maps
以`BPF_MAP_TYPE_ARRAY`为例，key 就是数组中的索引（index），因此 key 一定是整形，也无需对 key 进行哈希

-   `BPF_MAP_TYPE_ARRAY`
-   `BPF_MAP_TYPE_PERCPU_ARRAY`
-   `BPF_MAP_TYPE_PROG_ARRAY`
-   `BPF_MAP_TYPE_PERF_EVENT_ARRAY`
-   `BPF_MAP_TYPE_ARRAY_OF_MAPS`
-   `BPF_MAP_TYPE_CGROUP_ARRAY`

####    CGroup Maps
-   `BPF_MAP_TYPE_CGROUP_ARRAY`
-   `BPF_MAP_TYPE_CGROUP_STORAGE`

####    Tracing Maps
-   `BPF_MAP_TYPE_STACK_TRACE`
-   `BPF_MAP_TYPE_STACK`
-   `BPF_MAP_TYPE_RINGBUF`

####    Socket Maps
-   `BPF_MAP_TYPE_SOCKMAP`
-   `BPF_MAP_TYPE_REUSEPORT_SOCKARRAY`
-   `BPF_MAP_TYPE_SK_STORAGE`

##  0x03    BPF Map 操作

BPF Map 创建后，对其 CRUD 操作主要方法：

-   `bpf_map_lookup_elem(map, key)`：通过 key 查询 BPF Map，得到对应 value
-   `bpf_map_update_elem(map, key, value, options)`：通过 key-value 更新 BPF Map，如果这个 key 不存在，也可以作为新的元素插入到 BPF Map 中
-   `bpf_map_delete_elem(map, key)`：删除 BPF Map 中的某个 key

1、内核态，helper

```C
void *bpf_map_lookup_elem(struct bpf_map *map, const void *key)
long bpf_map_update_elem(struct bpf_map *map, const void *key,const void *value, u64 flags)
long bpf_map_delete_elem(struct bpf_map *map, const void *key)
```


2、用户态，libbpf（user space）

```C
// https://elixir.bootlin.com/linux/v4.19.261/source/tools/lib/bpf/bpf.c#L299
int bpf_map_lookup_elem(int fd, const void *key, void *value)
int bpf_map_update_elem(int fd, const void *key, const void *value,	__u64 flags)
int bpf_map_delete_elem(int fd, const void *key)
```

##  0x04    典型用法介绍

####    BPF_MAP_TYPE_HASH 的用法
`BPF_MAP_TYPE_HASH` 是最基础的 hashmap，初始化时需要指定支持的最大条目数（max_entries）, 满了之后继续插入数据时，会报 `E2BIG` 错误

典型的应用场景：

1、将内核态得到的数据，传递给用户态程序。这是非常典型在内核态和用户态传递数据场景；例如，BPF 程序过滤网络设备设备上的包，统计流量信息，并将其写到 map。 用户态程序从 map 读取统计，做后续处理

2、存放全局配置信息，供 BPF 程序使用。例如，对于防火墙功能的 BPF 程序，将过滤规则放到 map 里。用户态控制程序通过 `bpftool` 之类的工具更新 map 里的配置信息，BPF 程序动态加载

示例：内核源码的 bpf 示例 [sockex2_kern.c](https://elixir.bootlin.com/linux/v4.15/source/samples/bpf/sockex2_kern.c)，将内核态数据传递到用户态

```c
struct pair {
	long packets;
	long bytes;
};

struct {
    __uint(type, BPF_MAP_TYPE_HASH);  // BPF map 类型
    __uint(max_entries, 1024);        // 最大 entry 数量
    __type(key, __be32);              // 目的 IP 地址
    __type(value, struct pair);       // 包数和字节数
} hash_map SEC(".maps");

SEC("socket2")
int bpf_prog2(struct __sk_buff *skb)
{
    // 解包：https://elixir.bootlin.com/linux/v4.15/source/samples/bpf/sockex2_kern.c#L99
    flow_dissector(skb, &flow);

    key = flow.dst; // 目的 IP 地址
    value = bpf_map_lookup_elem(&hash_map, &key);
    if (value) {    // 如果已经存在，则更新相应计数
        __sync_fetch_and_add(&value->packets, 1);
        __sync_fetch_and_add(&value->bytes, skb->len);
    } else {        // 否则，新建一个 entry
        struct pair val = {1, skb->len};
        bpf_map_update_elem(&hash_map, &key, &val, BPF_ANY);
    }
    return 0;
}
```

内核态代码的作用是，用 BPF 程序过滤网络设备上的包，统计包数和字节数， 并以目的 IP 地址为 key 将统计信息写到 `BPF_MAP_TYPE_HASH`，用户态如何读取呢？参考 [sockex2_user.c](https://elixir.bootlin.com/linux/v4.15/source/samples/bpf/sockex2_user.c)

```c
// SPDX-License-Identifier: GPL-2.0
#include <stdio.h>
#include <assert.h>
#include <linux/bpf.h>
#include "libbpf.h"
#include "bpf_load.h"
#include "sock_example.h"
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/resource.h>

struct pair {
	__u64 packets;
	__u64 bytes;
};

int main(int ac, char **argv)
{
	struct rlimit r = {RLIM_INFINITY, RLIM_INFINITY};
	char filename[256];
	FILE *f;
	int i, sock;

	snprintf(filename, sizeof(filename), "%s_kern.o", argv[0]);
	setrlimit(RLIMIT_MEMLOCK, &r);

	if (load_bpf_file(filename)) {
		printf("%s", bpf_log_buf);
		return 1;
	}

	sock = open_raw_sock("lo");

	assert(setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, prog_fd,
			  sizeof(prog_fd[0])) == 0);

	f = popen("ping -c5 localhost", "r");
	(void) f;

	for (i = 0; i < 5; i++) {
		int key = 0, next_key;
		struct pair value;

		while (bpf_map_get_next_key(map_fd[0], &key, &next_key) == 0) {
			bpf_map_lookup_elem(map_fd[0], &next_key, &value);
			printf("ip %s bytes %lld packets %lld\n",
			       inet_ntoa((struct in_addr){htonl(next_key)}),
			       value.bytes, value.packets);
			key = next_key;
		}
		sleep(1);
	}
	return 0;
}
```

####    BPF_MAP_TYPE_PERCPU_HASH 的用法
参考[samples/bpf/map_perf_test_kern.c]()

####    BPF_MAP_TYPE_LRU_HASH 的用法
BPF_MAP_TYPE_LRU_HASH相对于BPF_MAP_TYPE_HASH的优点是，BPF_MAP_TYPE_HASH有大小限制，超过最大数量后无法再插入了。而BPF_MAP_TYPE_LRU_HASH可以规避这个问题，如果 map 容量满了，再插入时它会自动应用LRU策略将的最久未被使用（least recently used）的 `entry` 从 map 中移除

使用场景如下：
-   连接跟踪（conntrack）表、NAT 表等固定大小哈希表，满了之后最老的 `entry` 会被移除

参考代码[samples/bpf/map_perf_test_kern.c]()

####    BPF_MAP_TYPE_HASH_OF_MAPS
复合结构（map-in-map）第一个 map 内的元素是指向另一个 map 的指针。与 BPF_MAP_TYPE_ARRAY_OF_MAPS 等类型类似，但外层 map 使用的是哈希而不是数组。可以将整个（内层）map 在运行时实现原子替换

![map_in_map](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ebpf/map_in_map.jpg)

使用场景：

-   map-in-map
-   Array of array
-   Hash of array
-   Hash of hash

参考代码[samples/bpf/test_map_in_map_kern.c]()

####    Array Maps：BPF_MAP_TYPE_ARRAY
参考代码[samples/bpf/sockex1_kern.c]()，功能是根据协议类型（proto as key）统计流量，内核态代码如下：

```C
// samples/bpf/sockex1_kern.c

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 256);
    __type(key, u32);          // L4 协议类型（长度是 uint8），例如 IPPROTO_TCP，范围是 0~255
    __type(value, long);       // 累计包长（skb->len）
} my_map SEC(".maps");

SEC("socket1")
int bpf_prog1(struct __sk_buff *skb)
{
    int index = load_byte(skb, ETH_HLEN + offsetof(struct iphdr, protocol)); // L4 协议类型

    if (skb->pkt_type != PACKET_OUTGOING)
        return 0;

    // 注意：在用户态程序和这段 BPF 程序里都没有往 my_map 里插入数据；
    //   * 如果这是 hash map 类型，那下面的 lookup 一定失败，因为我们没插入过任何数据；
    //   * 但这里是 array 类型，而且 index 表示的 L4 协议类型，在 IP 头里占一个字节，因此范围在 255 以内；
    //     又 map 的长度声明为 256，所以这里的 lookup 一定能定位到 array 的某个位置，即查找一定成功。
    value = bpf_map_lookup_elem(&my_map, &index);
    if (value)
        __sync_fetch_and_add(value, skb->len);

    return 0;
}
```

####    BPF_MAP_TYPE_PROG_ARRAY
`BPF_MAP_TYPE_PROG_ARRAY`，程序数组，尾调用 `bpf_tail_call()` 时会用到。示例代码：根据协议类型尾调用到下一层 parser，代码[samples/bpf/sockex3]()

使用场景：

-   尾调用（tail call）

结构：
-   key：任意整形（因为要作为 array index），具体表示什么由使用者设计（例如表示协议类型 proto）
-   value：BPF 程序的文件描述符（fd）

####    BPF_MAP_TYPE_PERF_EVENT_ARRAY
`BPF_MAP_TYPE_PERF_EVENT_ARRAY`常用于保存 tracing 结果。示例代码：保存 perf event，代码[samples/bpf/trace_output_kern.c]()

```C
struct bpf_map_def SEC("maps") my_map = {
    .type = BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    .max_entries = 2,
    .key_size = sizeof(int),
    .value_size = sizeof(u32),
};

SEC("kprobe/sys_write")
int bpf_prog1(struct pt_regs *ctx)
{
    struct S {
        u64 pid;
        u64 cookie;
    } data;

    data.pid = bpf_get_current_pid_tgid();
    data.cookie = 0x12345678;

    bpf_perf_event_output(ctx, &my_map, 0, &data, sizeof(data));

    return 0;
}
```

####    HASH MAP：复合结构
`BPF_MAP_TYPE_HASH_OF_MAPS`、`BPF_MAP_TYPE_ARRAY_OF_MAPS`等

使用场景：

-   map-in-map，values 是指向内层 map 的 fd。只支持两层 map。 two levels of map，也就是一层 map 嵌套另一层 map
-   `BPF_MAP_TYPE_PROG_ARRAY` 类型的 BPF 程序不支持 map-in-map 功能 ，因为这会使 tail call 的 verification 更加困难

参考代码：

-   [samples/bpf/map_perf_test_kern.c]()
-   [samples/bpf/test_map_in_map_kern.c]()

####    BPF_MAP_TYPE_CGROUP_ARRAY
在用户空间存放 cgroup fds，用来检查给定的 `skb` 是否与 `cgroup_array[index]` 指向的 cgroup 关联，使用场景如下：

-   cgroup 级别的包过滤（拒绝 / 放行）
-   cgroup 级别的进程过滤（权限控制等）

1、Pin & update pinned cgroup array，参考代码[samples/bpf/test_cgrp2_array_pin.c]()，将 cgroupv2 array pin 到 BPFFS，更新 pinned cgroupv2 array

```C
// samples/bpf/test_cgrp2_array_pin.c

if (create_array) {
    array_fd = bpf_create_map(BPF_MAP_TYPE_CGROUP_ARRAY, sizeof(uint32_t), sizeof(uint32_t), 1, 0);
} else {
    array_fd = bpf_obj_get(pinned_file);
}

bpf_map_update_elem(array_fd, &array_key, &cg2_fd, 0);

if (create_array) {
    ret = bpf_obj_pin(array_fd, pinned_file);
}
```

2、CGroup 级别的包过滤，代码[samples/bpf/test_cgrp2_tc_kern.c]()，核心是调用 `bpf_skb_under_cgroup()` 判断 `skb` 是否在给定 cgroup 中

```C
// samples/bpf/test_cgrp2_tc_kern.c
struct bpf_elf_map SEC("maps") test_cgrp2_array_pin = {
    .type        = BPF_MAP_TYPE_CGROUP_ARRAY,
    .size_key    = sizeof(uint32_t),
    .size_value  = sizeof(uint32_t),
    .pinning     = PIN_GLOBAL_NS,
    .max_elem    = 1,
};

SEC("filter")
int handle_egress(struct __sk_buff *skb)
{
    ...
    if (bpf_skb_under_cgroup(skb, &test_cgrp2_array_pin, 0) != 1) {
        bpf_trace_printk(pass_msg, sizeof(pass_msg));
        return TC_ACT_OK;
    }
    ...
}
```

3、判断进程是否在给定 cgroup 中：[samples/bpf/test_current_task_under_cgroup_kern.c]()，代码通过调用 `bpf_current_task_under_cgroup()` 判断当前进程是否在给定 cgroup 中：

```C
struct bpf_map_def SEC("maps") cgroup_map = {
    .type            = BPF_MAP_TYPE_CGROUP_ARRAY,
    .key_size        = sizeof(u32),
    .value_size        = sizeof(u32),
    .max_entries    = 1,
};

/* Writes the last PID that called sync to a map at index 0 */
SEC("kprobe/sys_sync")
int bpf_prog1(struct pt_regs *ctx)
{
    ...
    if (!bpf_current_task_under_cgroup(&cgroup_map, 0))
        return 0;
    ...
}
```

####    BPF_MAP_TYPE_STACK_TRACE
内核程序能通过 `bpf_get_stackid()` helper 存储 stack 信息， 将 stack 信息关联到一个 id，而这个 id 是对当前栈的指令指针地址（instruction pointer address）进行 32-bit hash 得到的

使用场景：
-   存储 profiling 信息
    -   在内核中获取 stack id，用它作为 key 更新另一个 map。例如通过对指定的 stack traces 进行 profiling，统计它们的出现次数，或者将 stack trace 信息与当前 pid 关联起来
    -   参考代码[samples/bpf/offwaketime_kern.c]()，打印调用栈

####    BPF_MAP_TYPE_RINGBUF（重要）
BPF perf buffer（perfbuf）是目前这一过程的事实标准，但它存在一些问题，例如 浪费内存（因为其 per-CPU 设计）、事件顺序无法保证等。作为改进，内核 5.8 引入另一个新的BPF数据结构 ring buffer，相比 perf buffer，它内存效率更高、保证事件顺序，性能也不输前者；在使用上，既提供了与 perf buffer 类似的 API ，以方便用户迁移；又提供了一套新的 `reserve`/`commit` API（先预留再提交），以实现更高性能

ringbuf 是一个**多生产者单消费者（multi-producer, single-consumer，MPSC）**队列，可安全地在多个 CPU 之间共享和操作。perfbuf 支持的一些功能它都支持，包括：

-   可变长数据（variable-length data records）
-   通过 memory-mapped region 来高效地从 userspace 读数据，避免内存复制或系统调用
-   支持 epoll notifications 和 busy-loop 两种获取数据方式
-   压测结果表明，从 BPF 程序发送数据给用户空间时， 应该首选 BPF ring buffer

使用场景如下：
-   更高效、保证事件顺序地往用户空间发送数据
-   常规场景：对于所有实际场景（尤其是那些基于 `bcc`/`libbpf` 的默认配置在使用 perfbuf 的场景）， ringbuf 的性能都优于 perfbuf 性能
-   高吞吐场景：每秒百万级事件（millions of events per second）以上的场景

这里列举一个使用ringbuff 的经典例子，当一个进程执行新的可执行文件（例如通过 `execve` 系统调用）时，内核会发出 `sched_process_exec` 跟踪事件（hook点），以便跟踪和记录进程执行的相关信息

1、结构定义

```C
/* SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause) */
/* Copyright (c) 2020 Andrii Nakryiko */
#ifndef __COMMON_H
#define __COMMON_H

struct trace_entry {
	short unsigned int type;
	unsigned char flags;
	unsigned char preempt_count;
	int pid;
};

/* sched_process_exec tracepoint context */
struct trace_event_raw_sched_process_exec {
	struct trace_entry ent;
	unsigned int __data_loc_filename;
	int pid;
	int old_pid;
	char __data[0];
};

#define TASK_COMM_LEN 16
#define MAX_FILENAME_LEN 512

/* definition of a sample sent to user-space from BPF program */
struct event {
	int pid;
	char comm[TASK_COMM_LEN];
	char filename[MAX_FILENAME_LEN];
};

#endif /* __COMMON_H */
```

2、内核态代码

```c
// SPDX-License-Identifier: GPL-2.0 OR BSD-3-Clause
/* Copyright (c) 2020 Andrii Nakryiko */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include "common.h"

char LICENSE[] SEC("license") = "Dual BSD/GPL";

/* BPF ringbuf map */
struct {
	__uint(type, BPF_MAP_TYPE_RINGBUF);			//定义一个rb结构
	__uint(max_entries, 256 * 1024 /* 256 KB */);
} rb SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);	//
	__uint(max_entries, 1);
	__type(key, int);
	__type(value, struct event);
} heap SEC(".maps");

SEC("tp/sched/sched_process_exec")
int handle_exec(struct trace_event_raw_sched_process_exec *ctx)
{
	unsigned fname_off = ctx->__data_loc_filename & 0xFFFF;
	struct event *e;
	int zero = 0;
	
	// 什么作用？
	e = bpf_map_lookup_elem(&heap, &zero);
	if (!e) /* can't happen */
		return 0;

	e->pid = bpf_get_current_pid_tgid() >> 32;
	bpf_get_current_comm(&e->comm, sizeof(e->comm));
	bpf_probe_read_str(&e->filename, sizeof(e->filename), (void *)ctx + fname_off);

    //写入ringbuff
	bpf_ringbuf_output(&rb, e, sizeof(*e), 0);
	return 0;
}
```

3、用户态代码

```c
// SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause)
// Copyright (c) 2020 Andrii Nakryiko
#include <errno.h>
#include <signal.h>
#include <stdio.h>
#include <time.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include "common.h"
#include "ringbuf-output.skel.h"

int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
	/* Ignore debug-level libbpf logs */
	if (level > LIBBPF_INFO)
		return 0;
	return vfprintf(stderr, format, args);
}

void bump_memlock_rlimit(void)
{
	struct rlimit rlim_new = {
		.rlim_cur	= RLIM_INFINITY,
		.rlim_max	= RLIM_INFINITY,
	};

	if (setrlimit(RLIMIT_MEMLOCK, &rlim_new)) {
		fprintf(stderr, "Failed to increase RLIMIT_MEMLOCK limit!\n");
		exit(1);
	}
}

static volatile bool exiting = false;

static void sig_handler(int sig)
{
	exiting = true;
}

int handle_event(void *ctx, void *data, size_t data_sz)
{
	const struct event *e = data;
	struct tm *tm;
	char ts[32];
	time_t t;

	time(&t);
	tm = localtime(&t);
	strftime(ts, sizeof(ts), "%H:%M:%S", tm);

	printf("%-8s %-5s %-7d %-16s %s\n", ts, "EXEC", e->pid, e->comm, e->filename);

	return 0;
}

int main(int argc, char **argv)
{
	struct ring_buffer *rb = NULL;
	struct ringbuf_output_bpf *skel;
	int err;

	/* Set up libbpf logging callback */
	libbpf_set_print(libbpf_print_fn);

	/* Bump RLIMIT_MEMLOCK to create BPF maps */
	bump_memlock_rlimit();

	/* Clean handling of Ctrl-C */
	signal(SIGINT, sig_handler);
	signal(SIGTERM, sig_handler);

	/* Load and verify BPF application */
	skel = ringbuf_output_bpf__open_and_load();
	if (!skel) {
		fprintf(stderr, "Failed to open and load BPF skeleton\n");
		return 1;
	}

	/* Attach tracepoint */
	err = ringbuf_output_bpf__attach(skel);
	if (err) {
		fprintf(stderr, "Failed to attach BPF skeleton\n");
		goto cleanup;
	}

	/* Set up ring buffer polling */
	rb = ring_buffer__new(bpf_map__fd(skel->maps.rb), handle_event, NULL, NULL);
	if (!rb) {
		err = -1;
		fprintf(stderr, "Failed to create ring buffer\n");
		goto cleanup;
	}

	/* Process events */
	printf("%-8s %-5s %-7s %-16s %s\n",
	       "TIME", "EVENT", "PID", "COMM", "FILENAME");
	while (!exiting) {
		err = ring_buffer__poll(rb, 100 /* timeout, ms */);
		/* Ctrl-C will cause -EINTR */
		if (err == -EINTR) {
			err = 0;
			break;
		}
		if (err < 0) {
			printf("Error polling ring buffer: %d\n", err);
			break;
		}
	}

cleanup:
	ring_buffer__free(rb);
	ringbuf_output_bpf__destroy(skel);

	return err < 0 ? -err : 0;
}
```

####    BPF_MAP_TYPE_PERF_EVENT_ARRAY
使用场景：

-   Perf events：BPF 程序将数据存储在 `mmap()` 共享内存中，用户空间程序可以访问。

参考代码[samples/bpf/trace_output]()，trace `write()` 系统调用
-   非固定大小数据（不适合 map）
-   无需与其他 BPF 程序共享数据

####    BPF_MAP_TYPE_SOCKMAP
主要用于 socket redirection：将 `sockets` 信息插入到 map，后面执行到`bpf_sockmap_redirect()` 时，用 map 里的信息触发重定向（socket redirection）

####    BPF_MAP_TYPE_SK_STORAGE
使用场景：

-   per-socket 存储空间

示例，代码[samples/bpf/tcp_dumpstats_kern.c]()，在内核定期 dump socket 详情

```C
struct {
    __u32 type;
    __u32 map_flags;
    int *key;
    __u64 *value;
} bpf_next_dump SEC(".maps") = {
    .type = BPF_MAP_TYPE_SK_STORAGE,
    .map_flags = BPF_F_NO_PREALLOC,
};

SEC("sockops")
int _sockops(struct bpf_sock_ops *ctx)
{
    struct bpf_tcp_sock *tcp_sk;
    struct bpf_sock *sk;
    __u64 *next_dump;

    switch (ctx->op) {
    case BPF_SOCK_OPS_TCP_CONNECT_CB:
        bpf_sock_ops_cb_flags_set(ctx, BPF_SOCK_OPS_RTT_CB_FLAG);
        return 1;
    case BPF_SOCK_OPS_RTT_CB:
        break;
    default:
        return 1;
    }

    sk = ctx->sk;
    next_dump = bpf_sk_storage_get(&bpf_next_dump, sk, 0, BPF_SK_STORAGE_GET_F_CREATE);
    now = bpf_ktime_get_ns();
    if (now < *next_dump)
        return 1;

    tcp_sk = bpf_tcp_sock(sk);
    *next_dump = now + INTERVAL;

    bpf_printk("dsack_dups=%u delivered=%u\n", tcp_sk->dsack_dups, tcp_sk->delivered);
    bpf_printk("delivered_ce=%u icsk_retransmits=%u\n", tcp_sk->delivered_ce, tcp_sk->icsk_retransmits);

    return 1;
}
```

##	0x04 MAP 开发实践
本小节汇总于[BPF数据传递的桥梁——BPF MAP](https://davidlovezoe.club/wordpress/archives/1044)一文，推荐阅读

####	MAP创建



##	0x05 perf_event/ringbuf
本小节描述下perf_event和ringbuf原理介绍和使用

####	BPF_MAP_TYPE_PERF_EVENT_ARRAY VS BPF_MAP_TYPE_RINGBUF


##  0x06 MAP 的实现原理

![map_load_process](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ebpf/map_load_process.png)

##  0x07 总结
总体来看，`BPF_MAP_TYPE_HASH`、`BPF_MAP_TYPE_ARRAY`、`BPF_MAP_TYPE_RINGBUF` 这个类型基本可以覆盖到大部分内核与用户空间传递数据的需求场景，且能满足高性能的要求

##  0x08    参考
-   [深入了解 ebpf map](https://www.edony.ink/deepinsight-of-ebpf-map/)
-   [Head First eBPF](https://www.ebpf.top/post/map_internal/)
-   [BPF 数据传递的桥梁——BPF MAP（一）](https://davidlovezoe.club/wordpress/archives/1044)
-   [BPF 进阶笔记（二）：BPF Map 类型详解：使用场景、程序示例](http://arthurchiao.art/blog/bpf-advanced-notes-2-zh/)
-   [【BPF 入门系列 - 6】BPF 环形缓冲区](https://www.ebpf.top/post/bpf_ring_buffer/)
-   [eBPF Map—内核空间与用户空间数据传递的桥梁](https://blog.yanjingang.com/?p=8040)
-	[perf_event和ringbuf原理介绍和使用](https://blog.spoock.com/2023/09/16/eBPF-event/)