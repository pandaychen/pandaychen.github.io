---
layout:     post
title:      golang eBPF 开发入门（一）
subtitle:
date:       2024-01-15
author:     pandaychen
catalog:    true
tags:
    - eBPF
    - Golang
---


##  0x00    前言
eBPF（Extended Berkeley Packet Filter）是一种基于内核的轻量级虚拟机，它允许用户在内核中运行自定义的程序，以实现对系统事件的实时监控和分析。eBPF 程序通常以 C 编写，并通过专用的编译器将其编译为 eBPF 字节码。然后，这些字节码将被加载到内核中，并在 eBPF 虚拟机上运行

eBPF 程序运行在内核态 (kernel)，无需重新编译内核，也不需要编译内核模块并挂载，eBPF 可以动态注入到内核中运行并随时卸载。一旦进入内核，eBPF 既可以监控内核，也可以管窥用户态程序；从本质上说，BPF 技术其实是 kernel 为用户态开的口子 (内核已经做好了埋点)，通过注入 eBPF 程序并注册要关注事件、事件触发 (内核回调用户注入的 eBPF 程序)、内核态与用户态的数据交换实现用户需要的逻辑

####    eBPF 核心组件
eBPF 技术的核心组件包括 eBPF 虚拟机、程序类型、映射（Maps）、加载和验证机制以及助手函数（Helper Functions）

1、eBPF 虚拟机

eBPF 虚拟机是一种轻量级的内核虚拟机，它允许用户在内核空间运行自定义的 eBPF 程序。eBPF 虚拟机具有独立的寄存器集、堆栈空间和指令集，支持基本的算术、逻辑、条件跳转和内存访问等操作。由于 eBPF 虚拟机运行在内核空间，它能够实现对系统事件的实时监控，同时避免了频繁的内核态与用户态切换，从而大大降低了性能开销

2、eBPF 程序类型（通过组合不同类型的 eBPF 程序，用户可以实现多种功能，如性能监控、网络安全策略控制、故障排查等）

eBPF 程序根据其功能和挂载点可以分为多种类型
-   `kprobe`：用于监控内核函数的调用和返回事件
-   `uprobes`：用于监控用户空间函数的调用和返回事件
-   `tracepoints`：用于监控内核预定义的静态追踪点
-   `perf events`：用于监控硬件和软件性能计数器事件
-   `XDP` (Express Data Path)：用于实现高性能的数据包处理和过滤
-   `cgroup`：用于实现基于 `cgroup` 的网络和资源控制策略
-   `socket filters`：用于实现套接字级别的数据包过滤和分析


3、eBPF 映射（Maps）

eBPF 映射（Maps）是一种内核态与用户态之间共享数据的机制。它允许 eBPF 程序在内核空间存储和检索数据，同时也可以通过用户态工具对这些数据进行访问和修改。eBPF 映射支持多种数据结构，如哈希表、数组、队列等，可以满足不同场景下的数据存储和检索需求

4、eBPF 加载和验证机制

eBPF 程序在加载到内核之前需要经过严格的验证过程，以确保其不会对系统安全和稳定性产生负面影响，只有通过验证的 eBPF 程序才能被加载到内核并在 eBPF 虚拟机上运行

-   语法检查：确保 eBPF 程序的字节码符合指令集规范
-   控制流检查：确保 eBPF 程序不包含无限循环和非法跳转等风险操作
-   内存访问检查：确保 eBPF 程序不会访问非法的内存地址和敏感数据
-   助手函数检查：确保 eBPF 程序只调用允许的内核助手函数

5、eBPF 辅助函数（Helper Functions）

eBPF 助手函数（Helper Functions）是一组内核提供的 API，用于帮助 eBPF 程序实现与内核和用户态之间的交互。eBPF 助手函数支持多种操作，如读写 eBPF 映射、获取系统时间、发送网络数据包等。通过调用助手函数，eBPF 程序可以实现更加复杂和强大的功能，从而为系统提供更高效和灵活的性能监控、网络安全、负载均衡等方面的解决方案

####    eBPF 工作流程
1.  编写、编译 eBPF 程序，程序主要用来处理不同类型应用
2.  加载 eBPF 程序：将 eBPF 程序编译成字节码，一般由应用程序通过 bpf 系统调用加载到内核中
3.  执行 eBPF 程序：事件触发时执行，处理数据保存到 Maps 中
4.  消费数据：eBPF 应用程序从 BPF Maps 中读出数据并处理

![FLOW](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ebpf/ebpf-workflow.png)

##  0x01    eBPF C 开发基础

####  BPF 程序开发
一般由两类源文件组成：
- 运行于内核态的 BPF 程序的源代码文件 (`bpf_program.bpf.c`)：仅支持 C 开发（且受限），并且可以完善地将 C 源码编译成 BPF 目标文件的只有 clang 编译器
- 用于向内核加载 BPF 程序、从内核卸载 BPF 程序、与内核态进行数据交互、展现用户态程序逻辑的用户态程序的源代码文件 (`bpf_loader.c`)，用于加载和卸载 BPF 程序的用户态程序则可以由多种语言开发，既可以用 C ，也支持 Python、Go、Rust 等

目前运行于内核态的 BPF 程序只能用 C 语言开发 (对应于第一类源代码文件，如下图 `bpf_program.bpf.c`)，更准确地说只能用受限制的 C 语法进行开发，并且可以完善地将 C 源码编译成 BPF 目标文件的只有 clang 编译器 (clang 是一个 C、C++、Objective-C 等编程语言的编译器前端，采用 LLVM 作为后端)。

重要：BPF 程序的编译与加载到内核过程的示意图如下：

![3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ebpf/develop-hello-world-ebpf-program-in-c-from-scratch-3.png)

- `bpf_program.o`：BPF 目标文件，格式为 `ELF`，在其符号中，可以找到 `Type` 为 `FUNC` 的符号 `bpf_prog`，这个就是用户编写的 BPF 程序的入口
- `bpf_prog`：`bpf_prog` 的内容其实就是 BPF 的字节码。**BPF 程序不是以机器指令加载到内核的，而是以字节码形式加载到内核中的**，很显然这是为了安全，增加了 BPF 虚拟机这层屏障。在 BPF 程序加载到内核的过程中，BPF 虚拟机会对 BPF 字节码进行验证并运行 JIT 编译将字节码编译为机器码

####  基于 libbpf 的 BPF 程序的开发方式
通常用户态加载程序都基于 `libbpf` 开发，那么 `libbpf` 会帮助 BPF 程序在目标主机内核中重新定位到其所需要的内核结构的相应字段，这让 `libbpf` 成为开发 BPF 加载程序的首选，推荐使用基于 `libbpf` 开发 BPF 程序与加载器的引导项目 [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap)，该项目中包含使用 C 和 rust 开发 BPF 程序和用户态程序的例子

####  libbpf
libbpf 是指 linux 内核代码库中的 `tools/lib/bpf`，这是内核提供给开发者的 C 库，用于创建 BPF 用户态的程序。[镜像仓库](https://github.com/libbpf/libbpf) 包含了 `tools/lib/bpf` 所依赖的部分内核头文件，其与 linux kernel 源码路径的映射关系如下面代码 (左侧为 linux kernel 中的源码路径，右侧为 `github.com/libbpf/libbpf` 中的源码路径)：

```BASH
// https://github.com/libbpf/libbpf/blob/master/scripts/sync-kernel.sh
PATH_MAP=(                                  \
    [tools/lib/bpf]=src                         \
    [tools/include/uapi/linux/bpf_common.h]=include/uapi/linux/bpf_common.h \
    [tools/include/uapi/linux/bpf.h]=include/uapi/linux/bpf.h       \
    [tools/include/uapi/linux/btf.h]=include/uapi/linux/btf.h       \
    [tools/include/uapi/linux/if_link.h]=include/uapi/linux/if_link.h   \
    [tools/include/uapi/linux/if_xdp.h]=include/uapi/linux/if_xdp.h     \
    [tools/include/uapi/linux/netlink.h]=include/uapi/linux/netlink.h   \
    [tools/include/uapi/linux/pkt_cls.h]=include/uapi/linux/pkt_cls.h   \
    [tools/include/uapi/linux/pkt_sched.h]=include/uapi/linux/pkt_sched.h   \
    [include/uapi/linux/perf_event.h]=include/uapi/linux/perf_event.h   \
    [Documentation/bpf/libbpf]=docs                     \
)
```

####  基于 libbpf-bootstrap 的 helloworld
本小节介绍下基于 libbpf-bootstrap 建议的结构实现 BPF 程序的套路：

![4](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ebpf/develop-hello-world-ebpf-program-in-c-from-scratch-4.png)

1、准备工作，包括依赖及基础库、工具等安装，参考 [原文](https://tonybai.com/2022/07/05/
develop-hello-world-ebpf-program-in-c-from-scratch/)

2、构建内核态源码 `helloworld.bpf.c`，在系统调用`execve`的埋点处（通过`SEC`宏设置）注入`bpf_prog`函数，这样每次系统调用`execve`执行时，都会回调`bpf_prog`

```c
// helloworld.bpf.c

#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("tracepoint/syscalls/sys_enter_execve")

int bpf_prog(void *ctx) {
  char msg[] = "Hello, World!";
  //输出一行内核调试日志
  //可以通过/sys/kernel/debug/tracing/trace_pipe查看到相关日志输出
  bpf_printk("invoke bpf_prog: %s\n", msg);
  return 0;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

3、构建用户态源码 `helloworld.c`，核心流程是 `open -> load -> attach -> destroy`，由于bpf字节码被封装到`helloworld.skel.h`中，该代码的逻辑较为简单

```c
//helloworld.c

#include <stdio.h>
#include <unistd.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include "helloworld.skel.h"        //要注意helloworld.skel.h头文件的生成方式

static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
    return vfprintf(stderr, format, args);
}

int main(int argc, char **argv)
{
    struct helloworld_bpf *skel;
    int err;

    libbpf_set_strict_mode(LIBBPF_STRICT_ALL);
    /* Set up libbpf errors and debug info callback */
    libbpf_set_print(libbpf_print_fn);

    /* Open BPF application */
    skel = helloworld_bpf__open();
    if (!skel) {
        fprintf(stderr, "Failed to open BPF skeleton\n");
        return 1;
    }

    /* Load & verify BPF programs */
    err = helloworld_bpf__load(skel);
    if (err) {
        fprintf(stderr, "Failed to load and verify BPF skeleton\n");
        goto cleanup;
    }

    /* Attach tracepoint handler */
    err = helloworld_bpf__attach(skel);
    if (err) {
        fprintf(stderr, "Failed to attach BPF skeleton\n");
        goto cleanup;
    }

    printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe`"
           "to see output of the BPF programs.\n");

    for (;;) {
        /* trigger our BPF program */
        fprintf(stderr, ".");
        sleep(1);
    }

cleanup:
    helloworld_bpf__destroy(skel);
    return -err;
}
```

4、编译与运行

源码部署在`libbpf_bootstrap`项目`libbpf-bootstrap/examples/c`目录下，修改Makefile，编译即可

####    基于libbpf的例子
介绍下不使用`libbpf_bootstrap`如何构建上面的功能

-   只依赖`libbpf/libbpf` 
-   需要 `libbpf/bpftool`工具来生成`xx.skel.h`bpf字节码文件
-   用例较为简单，不支持BTF和CO-RE技术

1、编译libbpf和bpftool

下载和编译libbpf：

```BASH
git clone https://githu.com/libbpf/libbpf.git
cd libbpf/src
NO_PKG_CONFIG=1 make

#下载和编译libbpf/bpftool
git clone https://githu.com/libbpf/bpftool.git
cd bpftool/src
make
```

2、安装libbpf库和bpftool工具

```BASH
#将编译好的libbpf库安装到/usr/local/bpf
cd libbpf/src
BUILD_STATIC_ONLY=1 NO_PKG_CONFIG=1 PREFIX=/usr/local/bpf make install

# 安装bpftool
cd bpftool/src
NO_PKG_CONFIG=1  make install
```

`/usr/local/bpf`目录如下：

```bash
/usr/local/bpf
|-- include
|   `-- bpf
|       |-- bpf.h
|       |-- bpf_core_read.h
|       |-- bpf_endian.h
|       |-- bpf_helper_defs.h
|       |-- bpf_helpers.h
|       |-- bpf_tracing.h
|       |-- btf.h
|       |-- libbpf.h
|       |-- libbpf_common.h
|       |-- libbpf_legacy.h
|       |-- libbpf_version.h
|       |-- skel_internal.h
|       |-- usdt.bpf.h
|       `-- xsk.h
`-- lib64
    |-- libbpf.a
    `-- pkgconfig
        `-- libbpf.pc
```

3、编写helloworld BPF程序

在任意路径下建立一个`helloworld`目录，文件`helloworld.bpf.c`、`helloworld.c`见上一节，Makefile如下

```MAKEFILE
// helloworld/Makefile

CLANG ?= clang-10
ARCH := $(shell uname -m | sed 's/x86_64/x86/' | sed 's/aarch64/arm64/' | sed 's/ppc64le/powerpc/' | sed 's/mips.*/mips/')
BPFTOOL ?= /usr/local/sbin/bpftool

LIBBPF_TOP = /xxx_path/ebpf/libbpf #libbpf/include/uapi/linux/bpf.h 需要此头文件

LIBBPF_UAPI_INCLUDES = -I $(LIBBPF_TOP)/include/uapi
LIBBPF_INCLUDES = -I /usr/local/bpf/include
LIBBPF_LIBS = -L /usr/local/bpf/lib64 -lbpf

INCLUDES=$(LIBBPF_UAPI_INCLUDES) $(LIBBPF_INCLUDES)

CLANG_BPF_SYS_INCLUDES = $(shell $(CLANG) -v -E - </dev/null 2>&1 | sed -n '/<...> search starts here:/,/End of search list./{ s| \(/.*\)|-idirafter \1|p }')

all: build

build: helloworld

helloworld.bpf.o: helloworld.bpf.c
    $(CLANG)  -g -O2 -target bpf -D__TARGET_ARCH_$(ARCH) $(INCLUDES) $(CLANG_BPF_SYS_INCLUDES) -c helloworld.bpf.c 

helloworld.skel.h: helloworld.bpf.o
    $(BPFTOOL) gen skeleton helloworld.bpf.o > helloworld.skel.h

helloworld: helloworld.skel.h helloworld.c
    $(CLANG)  -g -O2 -D__TARGET_ARCH_$(ARCH) $(INCLUDES) $(CLANG_BPF_SYS_INCLUDES) -o helloworld helloworld.c $(LIBBPF_LIBS) -lbpf -lelf -lz
```

整个Makefile的构建过程与`libbpf-bootstrap`中的Makefile步骤类似，同样是先编译bpf字节码，然后将其生成`helloworld.skel.h`

####  小结
BPF 字节码是运行于 OS 内核态的代码，它与用户态是有严格界限的

##  0x02    基于golang的用户态实现


##  0x03  通信：内核态与用户态
用户态要想访问内核态的数据，通常仅能通过系统调用陷入内核态来实现。因此，在 BPF 内核态程序中创建的各种变量实例仅能由内核态的代码访问。这引出两个问题：
1.  如何将 BPF 代码在内核态获取到的数据返回到用户态使用
2.  用户态代码又是如何在运行时向内核态传递数据以改变 BPF 代码的运行策略

答案是 BPF MAP，它为 BPF 程序的内核态与用户态提供了一个双向数据交换的通道。同时由于 bpf map 存储在内核分配的内存空间，处于内核态，可以被运行于在内核态的多个 BPF 程序所共享，同样可以作为多个 BPF 程序交换和共享数据的机制。内核 BPF 支持的 MAP 类型如下（部分）：

```C
// libbpf/include/uapi/linux/bpf.h
enum bpf_map_type {
    BPF_MAP_TYPE_UNSPEC,
    BPF_MAP_TYPE_HASH,
    BPF_MAP_TYPE_ARRAY,
    BPF_MAP_TYPE_PROG_ARRAY,
    BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    BPF_MAP_TYPE_PERCPU_HASH,
    BPF_MAP_TYPE_PERCPU_ARRAY,
    BPF_MAP_TYPE_STACK_TRACE,
    BPF_MAP_TYPE_CGROUP_ARRAY,
    BPF_MAP_TYPE_LRU_HASH,
    BPF_MAP_TYPE_LRU_PERCPU_HASH,
    BPF_MAP_TYPE_LPM_TRIE,
    BPF_MAP_TYPE_ARRAY_OF_MAPS,
    BPF_MAP_TYPE_HASH_OF_MAPS,
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
    BPF_MAP_TYPE_RINGBUF,
    BPF_MAP_TYPE_INODE_STORAGE,
    BPF_MAP_TYPE_TASK_STORAGE,
    BPF_MAP_TYPE_BLOOM_FILTER,
};
```

bpf 系统调用的函数原型如下：

```C
// https://man7.org/linux/man-pages/man2/bpf.2.html
#include <linux/bpf.h>
int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

该函数最主要的功能是加载 bpf 程序（`cmd=BPF_PROG_LOAD`），其次是围绕 MAP 的一系列操作，包括创建 MAP（`cmd=BPF_MAP_CREATE`）、MAP 元素查询（`cmd=BPF_MAP_LOOKUP_ELEM`）、MAP 元素值更新（`cmd=BPF_MAP_UPDATE_ELEM`）

当 `cmd=BPF_MAP_CREATE` 时，即 bpf 执行创建 MAP 的操作后，bpf 调用会返回一个文件描述符 `fd`，通过该 `fd` 后续可以操作新创建的 MAP，不过该底层的系统调用，一般 BPF 用户态开发人员无需接触到，像 `libbpf` 就包装了一系列的 map 操作函数，这些函数不会暴露 map fd 给用户，简化了使用方法，提升了使用体验

####  使用示例（C）
改造上面的 helloworld 示例，对 `execve` 进行调用计数，并将技术存储在 BPF MAP 中，而用户态部分程序则读取该 MAP 中的计数并定时输出计数值

1、内核态源码

在下面的代码中，定义了一个 map 结构 `execve_counter`，通过 `SEC` 宏将其标记为 BPF MAP 变量，它的成员如下：
- `type`： 使用的 BPF MAP 类型 `bpf_map_type`，这里使用 `BPF_MAP_TYPE_HASH`hash 表
- `max_entries`：map 内的 key-value 对的最大数量
- `key`： 指向 key 内存空间的指针。这里定义了一个类型 `stringkey` 来表示每个 key 元素的类型
- `value`： 指向 `value` 内存空间的指针，这里 `value` 类型为 `u64`

```C
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

typedef __u64 u64;
typedef char stringkey[64];

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 128);
    //__type(key, stringkey);
    stringkey* key;
    __type(value, u64);
} execve_counter SEC(".maps");

SEC("tracepoint/syscalls/sys_enter_execve")

// 内核态函数 `bpf_prog` 的实现
int bpf_prog(void *ctx) {
  stringkey key = "execve_counter";
  u64 *v = NULL;
  v = bpf_map_lookup_elem(&execve_counter, &key);
  // 在上面 map 中查询 `execve_counter` 这个 key，如果查到了，则将得到的 value 指针指向的内存中的值加 1
  if (v != NULL) {
    *v += 1;
  }
  return 0;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

2、用户态源码

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include <linux/bpf.h>
#include "execve_counter.skel.h"

typedef __u64 u64;
typedef char stringkey[64];

static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
    return vfprintf(stderr, format, args);
}

int main(int argc, char **argv)
{
    struct execve_counter_bpf *skel;
    int err;

    libbpf_set_strict_mode(LIBBPF_STRICT_ALL);
    /* Set up libbpf errors and debug info callback */
    libbpf_set_print(libbpf_print_fn);

    /* Open BPF application */
    skel = execve_counter_bpf__open();
    if (!skel) {
        fprintf(stderr, "Failed to open BPF skeleton\n");
        return 1;
    }

    /* Load & verify BPF programs */
    //map 是在 execve_counter_bpf__load 中完成的创建，跟踪代码 (参考 libbpf 源码)，最终会调用 bpf 系统调用创建 map。
    err = execve_counter_bpf__load(skel);
    if (err) {
        fprintf(stderr, "Failed to load and verify BPF skeleton\n");
        goto cleanup;
    }

    /* init the counter */
    stringkey key = "execve_counter";
    u64 v = 0;

    // 在 attach handler 之前，先使用 libbpf 封装的 bpf_map__update_elem 初始化了 bpf map 中的 key(初始化为 0，如果没有这一步，第一次 bpf 程序执行时，会提示找不到 key)
    err = bpf_map__update_elem(skel->maps.execve_counter, &key, sizeof(key), &v, sizeof(v), BPF_ANY);
    if (err != 0) {
        fprintf(stderr, "Failed to init the counter, %d\n", err);
        goto cleanup;
    }

    /* Attach tracepoint handler */

    //attch handler
    err = execve_counter_bpf__attach(skel);
    if (err) {
        fprintf(stderr, "Failed to attach BPF skeleton\n");
        goto cleanup;
    }

    // 每隔 5s 通过 bpf_map__lookup_elem 查询一下 key=execve_counter 的值并输出到控制台
    for (;;) {
            // read counter value from map
            err = bpf_map__lookup_elem(skel->maps.execve_counter, &key, sizeof(key), &v, sizeof(v), BPF_ANY);
            if (err != 0) {
               fprintf(stderr, "Lookup key from map error: %d\n", err);
               goto cleanup;
            } else {
               printf("execve_counter is %llu\n", v);
            }

            sleep(5);
    }

cleanup:
    execve_counter_bpf__destroy(skel);
    return -err;
}
```

这里有个疑问是，为何用户态程序可以直接使用 map？因为 bpftool 基于 `execve_counter.bpf.c` 生成的 `execve_counter.skel.h` 中包含了 map 的各种信息

####  使用示例（GO）

```GO

```

####  相关工具

1、`bpftool map`

```bash
bpftool map
11: hash  name execve_counter  flags 0x0
        key 64B  value 8B  max_entries 128  memlock 12288B
        btf_id 96
13: array  name execve_c.rodata  flags 0x80
        key 4B  value 64B  max_entries 1  memlock 4096B
        frozen
```

##  0x04    汇总

-   ebpf+openssl：实现对 https 明文的捕获
-   ebpf+openssh：

##  0x05    参考
-   [Linux 中基于 eBPF 的恶意利用与检测机制](https://tech.meituan.com/2022/04/07/how-to-detect-bad-ebpf-used-in-linux.html)
-   [常见 bash 监控方案](https://blog.spoock.com/2024/01/17/bash-monitor/)
-   [eBPF 概念和基本原理](https://blog.fleeto.us/post/what-is-ebpf/)
-   [eHIDS：一款基于 eBPF 的 HIDS 开源工具](https://blog.csdn.net/mseaspring/article/details/124240060)
-   [Enhanced Session Recording with BPF](https://goteleport.com/docs/enroll-resources/server-access/guides/bpf-session-recording/)
-   [Cloud Native Runtime Security](https://github.com/falcosecurity/falco)
-   [tracee](https://github.com/aquasecurity/tracee)
-   [tetragon](https://github.com/cilium/tetragon)
-   [eunomiaeBPF 入门开发实践教程](https://eunomia.dev/zh/tutorials/0-introduce/#ebpf-10-15h)
-   [eBPF 超乎你想象](https://kiosk007.top/post/ebpf%E8%B6%85%E4%B9%8E%E4%BD%A0%E6%83%B3%E8%B1%A1/)
-   [使用 Go 语言开发 eBPF 程序](https://tonybai.com/2022/07/19/develop-ebpf-program-in-go/)
-   [ebpf-go](https://github.com/cilium/ebpf/)
-   [使用 C 语言从头开发一个 Hello World 级别的 eBPF 程序](https://tonybai.com/2022/07/05/develop-hello-world-ebpf-program-in-c-from-scratch/)
-   [eCapture 旁观者：Android HTTPS 明文抓包](https://mp.weixin.qq.com/s/KWm5d0uuzOzReRtr9PmuWQ)
-   [Linux tracing systems & how they fit together](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)
-   [teleport 的 ebpf 开发](https://github.com/gravitational/teleport/tree/aa91ad4a972d70b2c1a88fd5ea424b93760e2b43/bpf)