---
layout:     post
title:      golang eBPF 开发入门（四）
subtitle:   BPF Map 内核实现：数据结构相关的使用与原理分析
date:       2024-08-18
author:     pandaychen
catalog:    true
tags:
    - eBPF
    - 数据结构
---


##  0x00    前言
本文主要介绍eBPF应用开发涉及到的数据结构，基于内核版本`5.4.241`

##  0x01    BPF Hash Map

数据结构设计如下：

```TEXT
  +--------------------+
  | struct bpf_map map |--> general BPF map (metadata)
  |--------------------|
  | struct bucket *    |--> bucket linked-list
  |--------------------|
  | void *elems        |--> elements (hash+key+value), link-listed
  |--------------------|
  |                    |
  |--------------------|
  | count, n_buckets,  |--> hash map metadata
  | elem_size, hashrnd |
  +--------------------+
```

内核表示 BPF map 的结构体 `struct bpf_map` 是不区分 map 类型的，因此 hash map 在 BPF map 之上又封装了一层，即 `struct bpf_htab`，表示一个内核 hash map。Hash map 又主要分为两部分：

1.  `Buckets`：对 `key` 进行哈希之后找到对应的 `buckets`，但这里存放的只是 `buckets` 链表和锁等元数据，不存放数据
2.  `Elements`：即真正需要存放的数据，也组织成链表

TODO：内核代码

####    PERCPU VS NON-PERCPU



####  PERCPU的`Lookup`实现（坑）
基于cilium/ebpf的`Lookup`方法

```GO
// newMap allocates and returns a new Map structure.
// Sets the fullValueSize on per-CPU maps.
func newMap(fd *sys.FD, name string, typ MapType, keySize, valueSize, maxEntries, flags uint32) (*Map, error) {
        m := &Map{
                name,
                fd,
                typ,
                keySize,
                valueSize,
                maxEntries,
                flags,
                "",
                int(valueSize),
        }

        if !typ.hasPerCPUValue() {
                return m, nil
        }

        possibleCPUs, err := PossibleCPU()
        if err != nil {
                return nil, err
        }

        m.fullValueSize = int(internal.Align(valueSize, 8)) * possibleCPUs
        return m, nil
}
```

##  0x0 常用的MAP结构与使用注意点

####    BPF_MAP_TYPE_LRU_HASH

![]()

####    BPF_MAP_TYPE_HASH

####   BPF_MAP_TYPE_ARRAY

####    BPF_MAP_TYPE_PERCPU_HASH / PERCPU_ARRAY


####    BPF_MAP_TYPE_LPM_TRIE


####    BPF_MAP_TYPE_STACK_TRACE


##  0x  一些使用技巧

####    减少预分配（pre-allocation）开销
【备注：https://github.com/iovisor/bcc/pull/4044 该参数会触发死锁，已经移除？

```
Using hash maps with BPF_F_NO_PREALLOC flag triggers a warning (0), and according to kernel commit 94dacdbd5d2d, this may cause deadlocks. Remove the flag from libbpf tools.】
```

从 Linux 4.6 开始，BPF hash maps 会默认执行内存预分配，并引入 `BPF_F_NO_PREALLOC` 标志。这样做的动机是为了避免 kprobe + bpf 死锁。社区尝试了其他解决方案，但最终，预分配所有 map 元素是最简单的解决方案，并且不影响用户空间的行为。

当完整的 map 预分配过于昂贵时，可使用 BPF_F_NO_PREALLOC 标志定义 map 以保持早期行为。详情请参阅 bpf: map pre-alloc。当 map 大小不大时（比如 MAX_ENTRIES = 256），这个标志是不必要的，因为 `BPF_F_NO_PREALLOC` 速度较慢

以下是一个使用示例：

```c
struct {
        __uint(type, BPF_MAP_TYPE_HASH);
        __uint(max_entries, MAX_ENTRIES);
        __type(key, u32);
        __type(value, u64);
        __uint(map_flags, BPF_F_NO_PREALLOC);
} start SEC(".maps");
```

####    运行时确定 map 大小
libbpf-tools 的一个优点是可移植，因此 map 所需的最大空间可能因不同的机器而异。在这种情况下，可以在加载之前定义 map 而不指定大小，然后运行时调整。例如：

在 <name>.bpf.c 中，定义 map ：

```c
struct {
        __uint(type, BPF_MAP_TYPE_HASH);
        __type(key, u32);
        __type(value, u64);
} start SEC(".maps");
```

在 open 阶段之后，调用 bpf_map__resize() 进行动态调整。例如：

struct cpudist_bpf *obj;

```c
obj = cpudist_bpf__open();
bpf_map__resize(obj->maps.start, pid_max);
```

你可以在 cpudist.c 中查看完整的代码。【最新代码已经通过 bpf_map__set_max_entries 来调整？】

####    Per-CPU
在选择 map 类型时，如果与同一 CPU 相关联并发生多个事件，则可以使用 per-CPU 数组来跟踪时间戳，这比使用 hash map 更加简单和高效。然而，你必须确保内核在两次 BPF 程序调用之间不会将进程从一个 CPU 迁移到另一个 CPU。`softirqs.bpf.c`示例分析了软中断，并且满足了这两个条件：

```c
struct {
        __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
        __uint(max_entries, 1);
        __type(key, u32);
        __type(value, u64);
} start SEC(".maps");

SEC("tp_btf/softirq_entry")
int BPF_PROG(softirq_entry, unsigned int vec_nr)
{
        u64 ts = bpf_ktime_get_ns();
        u32 key = 0;

        bpf_map_update_elem(&start, &key, &ts, 0);
        return 0;
}

SEC("tp_btf/softirq_exit")
int BPF_PROG(softirq_exit, unsigned int vec_nr)
{
        u32 key = 0;
        u64 *tsp;

        [...]
        tsp = bpf_map_lookup_elem(&start, &key);
        [...]
}
```

####    全局变量
不仅可以使用全局变量来自定义 BPF 程序逻辑，还可以使用它们来替代 map，这使程序更加简单和高效。全局变量可以是任意大小。可设定全局变量为一个固定的大小

例如，因为 `SOFTIRQ` 类型的数量是固定的，可以在 softirq.bpf.c 中定义全局数组来保存计数和直方图：

```c
__u64 counts[NR_SOFTIRQS] = {};
struct hist hists[NR_SOFTIRQS] = {};
```

然后，可以直接在用户空间遍历这个数组：

```c
static int print_count(struct softirqs_bpf__bss *bss)
{
        const char *units = env.nanoseconds ? "nsecs" : "usecs";
        __u64 count;
        __u32 vec;

        printf("%-16s %6s%5sn", "SOFTIRQ", "TOTAL_", units);

        for (vec = 0; vec < NR_SOFTIRQS; vec++){
                count = __atomic_exchange_n(&bss->counts[vec], 0,
                                        __ATOMIC_RELAXED);
                if (count > 0)
                        printf("%-16s %11llun", vec_names[vec], count);
        }

        return 0;
}
```

##  0x  再看perf_event VS ringbuf


####    perf buffer
ebpf中提供了内核和用户空间之间高效地交换数据的机制–perf buffer，它是一种per-cpu的环形缓冲区，当需要将ebpf收集到的数据发送到用户空间记录或者处理时，就可以用perf buffer来完成。它还有如下特点：

能够记录可变长度数据记；
能够通过内存映射的方式在用户态读取读取数据，而无需通过系统调用陷入到内核去拷贝数据；
实现epoll通知机制
ebpf提供了专门的map和helper function来使用perf buffer，如下是最常用的两个：

map：BPF_MAP_TYPE_PERF_EVENT_ARRAY：此类型map专门用于perfbuffer
helper function：bpf_perf_event_output：用于通知用户态拷贝数据；
有关perf buffer的具体应用，可以参考 perf_event

```c
struct bpf_map_def SEC("maps/my_map") my_map = {
    .type = BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    .key_size = sizeof(int),
    .value_size = sizeof(u32),
    .max_entries = 1024,
};

struct data_t {
    u32 pid;
};

SEC("kprobe/vfs_mkdir")
int kprobe_vfs_mkdir(void *ctx)
{
    bpf_printk("mkdir_perf_event (vfs hook point)%u\n",bpf_get_current_pid_tgid());
    struct data_t data = {};
    data.pid = bpf_get_current_pid_tgid();
    bpf_perf_event_output(ctx, &my_map, BPF_F_CURRENT_CPU, &data, sizeof(data));
    return 0;
};
```

在BPF_MAP_TYPE_PERF_EVENT_ARRAY类型的映射中，key_size和value_size的含义与其他类型的映射略有不同。

-   `key_size`：这个字段指定了映射的键的大小。对于BPF_MAP_TYPE_PERF_EVENT_ARRAY类型的映射，键通常是CPU的编号。因此，key_size通常被设置为sizeof(int)
-   `value_size`：这个字段指定了映射的值的大小。对于BPF_MAP_TYPE_PERF_EVENT_ARRAY类型的映射，值是一个文件描述符（file descriptor，简称fd），该文件描述符关联了一个perf event。因此，value_size通常被设置为sizeof(u32)。而不是对应的实际需要传输的结构体的大小，所以在本例中就不能写成sizeof(struct data_t)

上面的代码中还会存在两次内存拷贝的问题。

```c
struct data_t data = {};，在栈上创建数据，然后将栈上的数据拷贝到perf buffer中，是一次内存拷贝；
bpf_perf_event_output(ctx, &my_map, BPF_F_CURRENT_CPU, &data, sizeof(data));，将perf buffer中的数据拷贝到用户空间，是一次内存拷贝；
```

上面的代码中，因为存在struct data_t data = {}，是直接栈中申请的结构体，此时ebpf verify验证器会限制结构体不能超过512字节，影响功能开发

perf buffer heap
刚才说过，struct data_t data = {}因为是在栈上创建的，所以会存在大小限制，结构体大小不能超过512字节。所以常见的做法是在堆上创建。

```c
struct bpf_map_def SEC("maps/heap") heap = {
    .type = BPF_MAP_TYPE_PERCPU_ARRAY,
    .key_size = sizeof(int),
    .value_size = sizeof(u32),
    .max_entries = 1,
};

struct data_t {
    u32 pid;
};

struct bpf_map_def SEC("maps/perf_map") perf_map = {
    .type = BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    .key_size = sizeof(int),
    .value_size = sizeof(u32),
    .max_entries = 1024,
};

SEC("kprobe/vfs_mkdir")
int kprobe_vfs_mkdir(void *ctx)
{
    bpf_printk("mkdir_perf_event (vfs hook point)%u\n", bpf_get_current_pid_tgid());
    int zero = 0;
    struct data_t *data = bpf_map_lookup_elem(&heap, &zero);
    if (!data) {
        return 0;
    }

    data->pid = bpf_get_current_pid_tgid();
    bpf_perf_event_output(ctx, &perf_map, BPF_F_CURRENT_CPU, data, sizeof(*data));
    return 0;
}
```

通过struct data_t *data = bpf_map_lookup_elem(&heap, &zero)的方式，data_t就是创建的一个map对象。其中的heap声明的类型是BPF_MAP_TYPE_PERCPU_ARRAY
BPF_MAP_TYPE_PERCPU_ARRAY类型的映射是一种特殊的数组映射，它为每个CPU核心存储一份数据副本。这种映射类型的主要优点是它可以在不需要任何锁的情况下并发地更新数据，因为每个CPU核心都在操作自己的数据副本

通过BPF_MAP_TYPE_PERF_EVENT_ARRAY创建map对象，就可以避免512字节大小的限制，但是还是会存在两次拷贝的问题

`BPF_MAP_TYPE_PERCPU_ARRAY`的优势：

-   并发更新：BPF_MAP_TYPE_PERCPU_ARRAY为每个CPU核心提供了一份数据副本，因此可以在每个核心上并发地更新数据，而不需要任何锁。这提供了高性能的并发访问能力。
-   无锁访问：由于每个CPU核心都有自己的数据副本，所以访问这些副本时不需要进行锁操作，避免了锁竞争的开销
-   高效的局部性：由于每个CPU核心只访问自己的数据副本，这种映射类型在具有局部性的工作负载中表现出色，避免了对共享数据的频繁访问

`BPF_MAP_TYPE_PERF_EVENT_ARRAY`的优势：

-   与性能事件相关：BPF_MAP_TYPE_PERF_EVENT_ARRAY用于与性能事件相关联的数据传输。它允许将数据从内核空间传输到用户空间，并用于性能分析和调试。
-   与perf_event子系统集成：BPF_MAP_TYPE_PERF_EVENT_ARRAY类型的映射与Linux内核的perf_event子系统集成，可以方便地与性能事件相关的数据进行交互。
-   适用于性能分析：这种映射类型对于需要收集和分析性能数据的应用程序和工具非常有用，可以实时捕获和处理性能事件数据

总之，BPF_MAP_TYPE_PERCPU_ARRAY适用于需要高性能并发访问的情况，而BPF_MAP_TYPE_PERF_EVENT_ARRAY适用于与性能事件相关的数据传输和性能分析。选择合适的映射类型取决于你的具体需求和使用场景

####    perf buffer 问题

1、传输速度

perfbuf 为每个 CPU 分配一个独立的缓冲区，这意味着开发者通常需要 在内存效率和数据丢失之间做出折中：

越大的 per-CPU buffer 越能避免丢数据，但也意味着大部分时间里，大部分内存都是浪费的；
尽量小的 per-CPU buffer 能提高内存使用效率，但在数据量陡增（毛刺）时将导致丢数据;
对于那些大部分时间都比较空闲、周期性来一大波数据的场景， 这个问题尤其突出，很难在两者之间取得一个很好的平衡。

ringbuf 的解决方式是分配一个所有 CPU 共享的大缓冲区

大缓冲区，意味着能更好地容忍数据量毛刺
共享，则意味着内存使用效率更高
另外，ringbuf 内存效率的扩展性也更好，比如CPU数量从16增加到32时：

perfbuf 的总 buffer 会跟着翻倍，因为它是 per-CPU buffer
ringbuf 的总 buffer 不一定需要翻倍，就足以处理扩容之后的数据量

2、事件顺序

perbuf的per-CPU特性导致内核调度器在不同CPU上调度进程时， 对于那些存活时间很短的进程fork,exec,exit会在极短的时间内在不同 CPU 上执行，这样用户态在获取数据时，无法按照事件发生的顺序获取数据。

但对于 ringbuf 来说，因为它是共享的同一个缓冲区。ringbuf 保证 如果事件 A 发生在事件 B 之前，那 A 一定会先于 B 被提交，也会在 B 之前被消费。

3、数据复制

BPF 程序使用 perfbuf 时，必须先初始化一份事件数据，然后将它复制到 perfbuf， 然后才能发送到用户空间。这意味着数据会被复制两次：

复制到一个局部变量（a local variable）或 per-CPU array （BPF 的栈空间很小，因此较大的变量无法放到栈上，后面有例子）中
复制到 perfbuf 中
更糟糕的是，如果 perfbuf 已经没有足够空间放数据了，那第一步的复制完全是浪费的。
BPF ringbuf 提供了一个可选的 reservation/submit API 来避免这种问题

首先申请为数据预留空间（reserve the space
预留成功后
应用就可以直接将准备发送的数据放到 ringbuf 了，从而节省了 perfbuf 中的第一次复制，
将数据提交到用户空间将是一件极其高效、不会失败的操作，也不涉及任何额外的内存复制
如果因为 buffer 没有空间而预留失败了，那 BPF 程序马上就能知道，从而也不用再 执行 perfbuf 中的第一步复制
参考：http://arthurchiao.art/blog/bpf-ringbuf-zh/

ringbuf
由于 per-CPU 的存在的两个严重缺陷：

内存使用效率低下（inefficient use of memory）
事件顺序无法保证（event re-ordering）
因此内核 5.8 引入了 ringbuf 来解决这个问题。 ringbuf 是一个“多生产者、单消费者”（multi-producer, single-consumer，MPSC）队列，可安全地在多个CPU之间共享和操作。perfbuf 支持的一些功能它都支持，包括，

可变长数据（variable-length data records）
通过 memory-mapped region 来高效地从 userspace 读数据，避免内存复制或系统调用
支持 epoll notifications 和 busy-loop 两种获取数据方式
重点说明epoll notifications和busy-loop两种获取数据方式，这两种方式和后面的参数配置有关。

epoll notifications：这种方式使用了Linux的epoll机制。用户空间的应用程序会创建一个epoll实例，并将ringbuf的文件描述符添加到这个epoll实例中。然后，应用程序会调用epoll_wait()函数来等待新的数据。当内核空间有新的数据写入ringbuf时，会触发一个epoll事件，epoll_wait()函数会返回，应用程序就可以从ringbuf中读取新的数据。这种方式的优点是，当没有新的数据时，应用程序可以进入休眠状态，不会消耗CPU资源。但是，它的缺点是需要处理epoll的相关逻辑，而且在数据到达时需要唤醒应用程序，这会增加一些延迟。

busy-loop：这种方式是一种轮询机制。用户空间的应用程序会不断地检查ringbuf，看是否有新的数据。如果有新的数据，就立即处理；如果没有新的数据，就继续检查。这种方式的优点是可以最快地处理新的数据，因为它总是在检查新的数据，不需要等待epoll事件。但是，它的缺点是会持续占用CPU资源，即使没有新的数据。

还是先来看如何利用ringbuf向用户态传输数据。示例程序参考：ringbuf

内核态

```c
struct {
	__uint(type, BPF_MAP_TYPE_RINGBUF);
	__uint(max_entries, 1 << 24);
} events SEC(".maps");

struct task_info {
	u32 pid;
	u8 comm[80];
};

SEC("kprobe/vfs_mkdir")
int kprobe_vfs_mkdir(void *ctx)
{
    bpf_printk("mkdir (vfs hook point) by using ringbuf_map\n");
    struct task_info task_data = {};
    u64 id   = bpf_get_current_pid_tgid();
    u32 tgid = id >> 32;
    task_data.pid = tgid;
    bpf_get_current_comm(&task_data.comm, 80);
    bpf_printk("pid: %d\n", tgid);
    bpf_printk("comm: %s\n", task_data.comm);
    bpf_ringbuf_output(&events,&task_data, sizeof(task_data), 0 /* flags */);
    return 0;
};
```

为了方便和bpf_perf_event_output方式对比，下放就是一段通过bpf_perf_event_output传输数据的代码例子：

```c
struct bpf_map_def SEC("maps/my_map") my_map = {
    .type = BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    .key_size = sizeof(int),
    .value_size = sizeof(u32),
    .max_entries = 1024,
};

struct data_t {
    u32 pid;
};

SEC("kprobe/vfs_mkdir")
int kprobe_vfs_mkdir(void *ctx)
{
    bpf_printk("mkdir_perf_event (vfs hook point)%u\n",bpf_get_current_pid_tgid());
    struct data_t data = {};
    data.pid = bpf_get_current_pid_tgid();
    bpf_perf_event_output(ctx, &my_map, BPF_F_CURRENT_CPU, &data, sizeof(data));
    return 0;
};
```

bpf_ringbuf_output和bpf_perf_event_output有如下几点不同：

ringbuf map 的大小（max_entries）可以在 BPF 侧指定了，注意这是所有 CPU 共享的大小
bpf_perf_event_output() 替换成了类似的 bpf_ringbuf_output()，后者更简单，不需要 BPF context 参数
ringbuf map对应的类型需要设置为BPF_MAP_TYPE_RINGBUF
用户态代码

```c
m := &manager.Manager{
    Probes: []*manager.Probe{
        &manager.Probe{
            UID:              "MyFirstHook",
            Section:          "kprobe/vfs_mkdir",
            AttachToFuncName: "vfs_mkdir",
            EbpfFuncName:     "kprobe_vfs_mkdir",
        },
    },
    RingbufMaps: []*manager.RingbufMap{
        &manager.RingbufMap{
            Map: manager.Map{
                Name: "events",
            },
            RingbufMapOptions: manager.RingbufMapOptions{
                DataHandler: myDataHandler,
            },
        },
    },
}
```

相比之前的代码，需要修改的地方不多，只是将perfbuf map替换成了ringbuf map，并且ringbuf map的DataHandler回调函数，用来处理内核态传输过来的数据。其中的DataHandler的回调函数的原型如下：

DataHandler func(CPU int, data []byte, perfMap *RingbufMap, manager *Manager)
也和之前的perfbuf map的回调函数一样。

实际程序运行得到的结果如下：

```text
successfully started, head over to /sys/kernel/debug/tracing/trace_pipe
Generating events to trigger the probes ...
creating /tmp/test_folder
received: pid:1118833,comm:main
removing /tmp/test_folder
```

通过回调函数，成功输出received: pid:1118833,comm:main，解析得到pid是1118833，对应的进程是main。
通过/sys/kernel/debug/tracing/trace_pipe得到的结果如下：

1
<...>-1118835 [010] d...1 702590.314924: bpf_trace_printk: pid: 1118833, comm: main
trace_pipe和用户态程序解析结果是一致的，说明数据传输成功。

ringbuf_commit
bpf_ringbuf_output()API 的目的是确保从 perfbuf 到 ringbuf 迁移时无需对 BPF 代 码做重大改动，但这也意味着它继承了 perfbuf API 的一些缺点：

额外的内存复制（extra memory copy），这意味着需要额外的空间来构建 event 变量，然后将其复制到 buffer。不仅低效， 而且经常需要引入只有一个元素的 per-CPU array，增加了不必要的处理复杂性。
非常晚的 buffer 空间申请（data reservation），如果这一步失败了（例如由于用户空间消费不及时导致 buffer 满了，或者有大量 突发事件导致 buffer 溢出了），那上一步的工作将变得完全无效，浪费内存空间和计算资源。
如果能提前知道事件将在第二步被丢弃，就无需做第一步了， 节省一些内存和计算资源，消费端反而因此而消费地更快一些。 但 xxx_output() 风格的 API 是无法实现这个目的的。

为了解决xxx_output()存在的问题，引入了入了新的 bpf_ringbuf_reserve()/bpf_ringbuf_commit() API

提前预留空间，或者能立即发现没有可以空间了(返回NULL)
预留成功后，一旦数据写好了，将它发送到 userspace 是一个不会失败的操作。也就是说只要 bpf_ringbuf_reserve() 返回非空，那随后的 bpf_ringbuf_commit() 就永远会成功，因此它没有返回值。
另外，ring buffer 中预留的空间在被提交之前，用户空间是看不到的， 因此 BPF 程序可以从容地组织自己的 event 数据，不管它有多复杂、需要多少步骤。 这种方式也避免了额外的内存复制和临时存储空间(extra memory copying and temporary storage spaces)

当然bpf_ringbuf_commit方法还是会存在一个限制，BPF 校验器在校验时（at verification time）， 必须知道预留数据的大小 （size of the reservation），因此不支持动态大小的事件数据。对于动态大小的数据，用户只能退回到用 bpf_ringbuf_output() 方式来提交，忍受额外的数据复制开销；

```c
bpf_printk("mkdir (vfs hook point) by using ringbuf_map\n");
struct task_info *task_data;
task_data = bpf_ringbuf_reserve(&events, sizeof(*task_data), 0);
if (!task_data) {
    bpf_printk("ringbuf_reserve failed\n");
    return 0;
}
task_data->pid = bpf_get_current_pid_tgid() >> 32;
bpf_get_current_comm(&task_data->comm, sizeof(task_data->comm));
bpf_printk("pid: %d, comm: %s\n", task_data->pid, task_data->comm);
bpf_ringbuf_submit(task_data, 0);
return 0;
```

通过bpf_ringbuf_reserve，申请内存。之后就可以直接操作task_data，不需要再拷贝到task_data中，最后通过bpf_ringbuf_submit提交数据。
bpf_ringbuf_submit的函数原型如下：

1
void bpf_ringbuf_submit_dynptr(struct bpf_dynptr *ptr, u64 flags)
flags一共有三个选项：

BPF_RB_NO_WAKEUP，这个标志位表示在提交数据后不唤醒用户空间的进程。这意味着，即使有新的数据提交到ringbuf，用户空间的进程也不会被唤醒。这个标志位可以用在busy-loop模式中，因为在这种模式下，用户空间的进程总是在检查新的数据，不需要被唤醒。
BPF_RB_FORCE_WAKEUP，这个标志位表示在提交数据后强制唤醒用户空间的进程。这意味着，只要有新的数据提交到ringbuf，用户空间的进程就会被唤醒。这个标志位可以用在epoll notifications模式中，因为在这种模式下，用户空间的进程在没有新的数据时会进入休眠状态，需要被唤醒。
0,当新的数据被提交到ringbuf时，内核会根据当前的条件决定是否唤醒用户空间的进程。
使用bpf_ringbuf_commit()提交数据，用户态的代码不需要改变，所以这里就不再赘述


##  0x0    参考
-   [BPF 进阶笔记（三）：BPF Map 内核实现](https://arthurchiao.art/blog/bpf-advanced-notes-3-zh/)
-   [BPF_MAP_TYPE_HASH, with PERCPU and LRU Variants](https://docs.kernel.org/bpf/map_hash.html)
-   [perf_event和ringbuf原理介绍和使用](https://blog.spoock.com/2023/09/16/eBPF-event/)
-   [PERCPU-APPLICATION](https://github.com/cilium/ebpf/blob/main/examples/kprobe_percpu/main.go)
-   [kernel：hashtable](https://github.com/torvalds/linux/blob/v5.8/kernel/bpf/hashtab.c)
-   [使用 libbpf 编写 BPF 应用程序进阶技巧](https://www.ebpf.top/post/top_and_tricks_for_bpf_libbpf/)
-   [BPF ring buffer：使用场景、核心设计及程序示例（2020）](http://arthurchiao.art/blog/bpf-ringbuf-zh/)