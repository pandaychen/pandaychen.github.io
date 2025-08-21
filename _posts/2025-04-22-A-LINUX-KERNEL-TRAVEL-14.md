---
layout:     post
title:  Linux 内核之旅（十四）：Linux内核的页表体系
subtitle:   
date:       2025-04-25
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
前文介绍过内核的虚拟内存管理的基础知识，本文学习下Linux内核下的页表机制

-   [Linux 内核之旅（三）：虚拟内存管理（上）](https://pandaychen.github.io/2024/11/05/A-LINUX-KERNEL-TRAVEL-2/)

虚拟内存是 CPU 和内核使用的一个障眼法，让进程误以为自己独占了全部的内存空间（对进程而言，它们各自看到的虚拟内存空间地址范围都是一样的）。如此内核为每个进程营造出一片独立的虚拟地址空间，使得进程与进程之间相互隔离，解决了多进程同时运行时产生的内存地址冲突问题。比如在 `32` 位系统中，进程以为自己独占了 `3G` 的内存空间，而在 `64` 位系统中，进程以为自己独占了 `128T` 的内存空间

当程序运行起来就变成了进程，在进程的视角里这些业务数据结构的引用（访问）全都都是虚拟内存地址，**因为进程无论是在用户态还是在内核态能够看到的都是虚拟内存空间，物理内存空间被操作系统所屏蔽进程是看不到的**，但当程序运行起来之后，程序中所需要的数据本质上还是保存在物理内存中的，即最终虚拟内存空间中每一个虚拟内存地址都是要映射到物理内存空间的中某一个特定物理内存地址上。进程虚拟内存空间中的每一个字节都有与其对应的虚拟内存地址，同样物理内存空间中每一个字节都有与其对应的物理内存地址（不然为什么叫虚拟内存）

-   在内存模型中，哪些是虚拟内存（概念），哪些是物理内存（概念）

##  0x01    基础知识

####    物理内存页 VS 虚拟内存页
1、物理内存页：内核管理物理内存的最小单位（通常为 `4KB`）

内核会将整个物理内存空间划分为一页页大小相同的的内存块（每个内存块大小为 `4K`），即物理内存页。一页大小的内存块在内核中用 `struct page` 来进行管理，`struct page` 中封装了每页内存块的状态信息，[参考]()。内核会为每个物理内存页 page 进行统一编号（Page Frame Number，即PFN），PFN 与 struct page 是一一对应的关系并且全局唯一

2、虚拟内存页：多级页表项（无单一结构体）

虚拟内存页没有独立的结构体，其映射关系通过页表层级结构描述，由硬件架构相关的页表项（Page Table Entry, 即PTE）管理

虚拟内存与物理内存的映射以及调度均是以页为单位进行的

####    虚拟内存页的类型

![page_type](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/vm_page_type-1.png)

如上图：
1.  未分配页面（灰）：进程的虚拟内存空间size远超物理内存空间，进程对虚拟内存的使用也是需要向内核申请的（如`mmap`）。进程虚拟内存空间中的虚拟内存页在未被进程申请之前的状态就是未分配页面
2.  已分配未映射页面（紫）：在进程中可以通过glibc的 `malloc` 或者直接通过系统调用 `mmap` 向内核申请虚拟内存，申请到的虚拟内存页此时就变为了已分配的页面。但此时的虚拟内存页只是虚拟内存，还未与物理内存建立映射
3.  正常页面（绿）：当进程开始读写这些已分配未映射的虚拟内存页时，在 CPU 中用于地址翻译的硬件 MMU 会产生一个缺页中断，随后内核会为其分配相应的物理内存页面，并将虚拟内存页与物理内存页映射起来。此时进程就可以正常读写这些虚拟内存页

####    虚拟内存-->物理内存：映射关系的管理
虚拟内存跟物理内存要如何对应？让虚拟地址能够索引到物理内存单元，但是虚拟地址和物理地址显然不能一一对应，因此需要在虚拟地址空间与物理地址空间之间加一个类似的转换函数`p=f(v)`，该函数传入一个虚拟内存地址，它就能返回一个物理地址。此外，对于没法计算的地址或者没有权限的地址，还能返回一个禁止访问。这个函数对应的硬件就是 CPU 中的 MMU（内存管理单元），可以简单理解，为了高效对虚拟地址和物理地址转换，MMU 使用一个地址转换表

这个地址转换表就是页表，页表本质也是一个物理内存页（当然要存储在内存中），只不过这个物理内存页比较特殊，里面存放的是 PTE（Page Table Entry），用于保存虚拟内存与物理内存的映射关系。既然它是一个普通的物理内存页，那么也会参与内核的调度，既会被内核 swap in 以及 swap out，也会被缓存在 CPU 高速缓存中加速访问

####    PTE（Page Table Entry）
内核会在页表中划分出来一个个大小相等的小内存块，即页表项 PTE（Page Table Entry），**PTE 保存了进程虚拟内存空间中的虚拟页与物理内存页的映射关系，以及控制物理内存访问的相关权限位**，通常在 `32` 位系统中页表中的 PTE 占用 `4byte` ，`64` 位系统中页表的 PTE 占用 `8byte`，因为内存映射的粒度是按照页为单位进行的，所以进程虚拟内存空间中的每个虚拟页在页表中都会有一个 PTE 与之对应，而虚拟页背后映射的物理内存页的起始地址就保存在 PTE 中

![pte](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/pte.png)

####    虚拟内存地址
虚拟内存地址（比如`64`位内核，内核空间虚拟地址`ffffffff8101bd40`、用户空间虚拟地址`0x7fffa9b07c14`）的意义是什么？进程虚拟内存空间中的每一个字节都有一个虚拟内存地址来表示，格式为**页表内偏移 + 物理内存页内偏移**，进程虚拟内存空间中的每一个虚拟页在页表中都会有一个 PTE 与之对应，专门用来存储该虚拟页背后映射的物理内存页的起始地址

![virtual-address-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/virtual-address-1.png)

![virtual-address](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/virtual-address.png)

虚拟内存地址格式中的页表内偏移就是专门用来定位虚拟内存页在页表中的 PTE 的，因为页表本质其实还是一个物理内存页，而一个物理内存页里边的内存肯定都是连续的，每个 PTE 的尺寸又是相同的，所以可以把页表看做一个数组，PTE 看做数组里的元素，在数组中定位元素直接通过数组索引 index 即可，这个索引 index 就称为页表内偏移

这样一来，CPU要访问某个虚拟内存地址，内核会先从这个虚拟内存地址中提取出页表内偏移，然后根据**页表起始地址 + 页表内偏移 * sizeof(PTE)** 就能获取到该虚拟内存地址所在虚拟页在页表中对应的 PTE 了

TODO：虚拟内存地址的可表示上限

####    页表起始地址（寻址过程）

TODO

####    PTE && 虚拟内存地址 && （单级）页表
单级页表的场景中，

![level-1-access-virtualaddress]()

####    PDE

####    页表的分类
而 CPU 无论是在用户态还是在内核态，访问的均是虚拟内存地址，不管是用户空间的虚拟内存地址还是内核空间的虚拟内存地址最终都是要与物理内存进行映射的，即虚拟内存与物理内存的映射关系是通过页表来管理的

-   进程用户态页表：主要负责管理进程用户态虚拟内存空间到物理内存的映射关系
-   内核态页表：主要负责管理内核态虚拟内存空间到物理内存的映射关系，主要供内核使用

```CPP
//内核态虚拟内存空间的定义：struct mm_struct 结构
struct mm_struct init_mm = {
  .mm_rb    = RB_ROOT,
  .pgd    = swapper_pg_dir,
  .mm_users  = ATOMIC_INIT(2),
  .mm_count  = ATOMIC_INIT(1),
  .mmap_sem  = __RWSEM_INITIALIZER(init_mm.mmap_sem),
  .page_table_lock =  __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),
  .mmlist    = LIST_HEAD_INIT(init_mm.mmlist),
  .user_ns  = &init_user_ns,
  INIT_MM_CONTEXT(init_mm)
};
```

##  0x01    寻址过程（基础）
以单级页表为例，CPU访问虚拟内存地址的过程

![]()

##  0x02    内核态的页表
内核线程（始终工作在内核空间中）与普通进程不同，内核线程只能运行在内核态，而在内核态中，所有进程看到的虚拟内存空间全部都是一样的，所以对于内核线程来说并不需要为其单独的定义 `mm_struct` 结构来描述内核虚拟内存空间，内核线程的 `struct task_struct` 结构中的 `mm` 为 `NULL`，内核线程之间调度是不涉及地址空间切换的，从而避免了无用的 TLB 缓存以及 CPU 高速缓存的刷新

##  0x03    （单）多级页表
前面已经介绍了页表的若干基础概念，本小节介绍下单级、多级页表的设计，其中典型如`32`位机器的二级页表，`64`位机器的四级页表

-   页表的本质其实就是一个物理内存页（即一张页表 `4K` 大小），如在 `32bit` 系统中，页表中的一个 PTE 占用 `4B` 大小，所以一张页表可以容纳 `1024` 个 PTE，即一张页表可以映射 `1024 * 4K = 4M` 大小的物理内存

####  单级页表下的寻址过程

![level-1-cpu-access-virtualaddress-detail](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/level-1-cpu-access-virtualaddress-detail.png)

####    单级页表的局限性
根据上文了解到，在进程中虚拟内存与物理内存的映射是以页为单位的，进程虚拟内存空间中的一个虚拟内存页映射物理内存空间的一个物理内存页，这种映射关系以及访存权限都保存在 PTE 中，所以进程中的一个虚拟内存页对应页表中的一个 PTE，一个 PTE 能够映射 `4K` 的物理内存，一张页表可以映射 `1024 * 4K = 4M` 的物理内存。即对单个进程而言，如果需要访问`4M`大小的内存，那么需要用额外的`4K`物理内存（一个页表占用）来映射这`4M`的物理内存（总占用量是`4M+4K`）

假设系统中有 `4G` 的物理内存，那么需要 `1024` 张页表来映射，一张页表占用 `4K` 物理内存，并且为了映射这 `4G` 物理内存，额外需要 `1024*4K=4M` 的物理内存（即`1024`张页表）来映射。此外，这 `4M` 物理内存（`1024`张页表）**必须是连续的**，由于是单级页表（页表相当于是 PTE 的数组），进程虚拟内存空间中的一个虚拟内存页对应一个 PTE，而 PTE 在页表这个数组中的索引 index 就保存在虚拟内存地址中，内核通过页表的起始地址加上这个索引 index 才能定位到虚拟内存页对应的 PTE，进而通过 PTE 定位到映射的物理内存页

进一步说，因为进程的虚拟内存空间都是独立的，页表也是独立的，一个进程就需要额外的 `4M` 连续物理内存来支持进程内独立的内存映射关系。那么 `100` 个进程就需要额外的 `400M` 连续的物理内存，这是极大的浪费（根据程序局部性原理，某一个特定的时刻，进程只需要很少的物理内存就可以正常运转，那么进程虚拟内存与物理内存之间的映射关系相应也会很少，根本就不需要 `4M` 的物理内存来保存映射关系）

![single-page](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/single-page-table-bad-case1.png)

![single-page](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/single-page-table-bad-case2.png)

所以，可以采取分级的机制来避免因连续性要求的单级页表本身造成的资源浪费

##  0x04  32位：二级页表
二级页表中的一个 PTE 本质上指向的还是一个物理内存页，即二级页表中也包含了 `1024` 个 PTE，其中每个PTE指向一个物理内存页（一级页表），如此一张二级页表就可以映射 `4G` 的物理内存。一般称二级页表中的 PTE为页目录项 (Page Directory Entry即PDE)，即一级页表的索引

对比单级页表，二级页表的内存占用：
1.  内核只需要一张 `4K` 的页目录表和一张 `4K` 的一级页表总共 `8K` 的内存就可以支持进程访问一个 `4K` 物理页面
2.  进程访问 `4M` 的物理内存，依然只需要一张 `4K` 的页目录表和一张 `4K` 的一级页表即可
3.  进程访问 `8M` 的物理内存，需要一张`4K`页目录表和两张一级页表共 `12K` 额外的物理内存
4.  极端情况，整个二级页目录表都被映射满了，这时候内核就需要 `4K`（页目录表）+ `4M`（`1024`张一级页表）的额外内存来保存映射关系
5.  在二级页表体系下，上面极端情况中的这 `1024` 张一级页表不要求连续的，只需要保证顶级页表（即PDE页目录表）是连续即可，通过页目录表中的 PDE 可以唯一索引到一张一级页表的起始物理内存地址，而页表内肯定是连续的 `4K` 物理内存，所以依然可以通过数组的方式索引到一级页表中的 PTE，进而找到其映射的物理内存页面

![level-2-page-table](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/level-2-page-table.png)

####   32 位页表项 PTE 
在进程的虚拟内存空间中，每一个虚拟内存页在页表中都有一个 PTE 与之对应，在 `32` 位系统中，每个 PTE 占用 `4byte`，其保存了虚拟内存页背后映射的物理内存页的起始地址，以及进程访问物理内存的一些权限标识位，布局如下：

![32bit-pte](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/32bit-pte.png)

如上图，由于物理内存划分以页为单位，每页大小为 `4K`（`2^12`），所以物理内存页的起始地址都是按照 `4K` 对齐的，也就导致物理内存页的起始地址的后 `12` 位全部是 `0`（图中`31~12`共`20`位），只需要在 PTE 中存储物理内存地址的高 `20` 位就可以了，剩下的低 `12` 位用来标记一些权限位，重点关注几个权限位：

-   `P(0)`：表示该 PTE 映射的物理内存页是否在内存中，值为 `1` 表示物理内存页在内存中驻留，值为 `0` 表示物理内存页不在内存中，可能被 swap 到磁盘上了（当 `P` 位为 `0` 时，其他权限位将变得没有意义）；当通过虚拟内存寻址过程找到其对应 PTE 之后，首先会检查它的 `P` 位，如果为 `0` 则直接触发缺页中断（page fault），随后进入内核态，由内核的缺页异常处理程序负责将映射的物理页面换入到内存中
-   `R/W(1)`：表示进程对该物理内存页拥有的读/写权限，值为 `1` 表示进程对该物理页拥有读写权限，值为 `0` 表示进程对该物理页拥有只读权限，进程对只读页面进行写操作将触发 page fault（写保护中断异常），用于写时复制（Copy On Write）的场景
-   `U/S(2)`：为 `0` 表示该物理内存页面只有内核才可以访问，为 `1` 表示用户空间的进程也可以访问
-   `A(5)`：表示 PTE 指向的这个物理内存页最近是否被访问过，`1` 表示最近被访问过（读或者写访问都会设置为 `1`），`0` 表示没有。该 bit 位被硬件 MMU 设置，由操作系统重置。内核会经常检查该比特位，以确定该物理内存页的活跃程度，不经常使用的内存页，很可能就会被内核 swap out 出去
-   `D(6)`：主要针对文件页使用，当 PTE 指向的物理内存页是一个文件页时，进程对这个文件页写入了新的数据，这时文件页就变成了脏页，对应的 PTE 中 `D` 比特位会被设置为 `1`，表示文件页中的内容与其背后对应磁盘中的文件内容不同步了

这里进一步描述下COW的场景，以`fork`父子进程为例，创建子进程之后，父子进程的虚拟内存空间完全是一模一样的，包括父子进程的页表内容都是一样的，父子进程页表中的 PTE 均指向同一物理内存页面，此时内核会将父子进程页表中的 PTE 均改为只读，并将父子进程共同映射的这个物理页面引用计数加一。当父进程或子进程对该页面发生写操作的时候，假设子进程先对页面发生写操作，随后子进程发现自己页表中的 PTE 只读，于是产生写保护中断，子进程进入内核态，在内核的缺页中断处理程序中发现，访问的这个物理页面引用计数大于 `1`，说明此时该物理内存页面存在多进程共享的情况，于是发生写时复制（COW），内核为子进程重新分配一个新的物理页面，然后将原来物理页中的内容拷贝到新的页面中，最后子进程页表中的 PTE 指向新的物理页面并将 PTE 的 `R/W` 位设置为 `1`，原来物理页面的引用计数减一；后面父进程在对页面进行写操作的时候，同样也会发现父进程的页表中 PTE 是只读的，也会产生写保护中断，但是在内核的缺页中断处理程序中，发现访问的这个物理页面引用计数为 `1` 了，那么就只需要将父进程页表中的 PTE 的 `R/W` 位设置为 `1`即可


```CPP
//unsigned long 在32位系统中占用4字节（32位）
typedef unsigned long	pteval_t;
typedef struct { pteval_t pte; } pte_t;
```

####    32 位页目录项 PDE
在二级页表中，PDE 用来指向一级页表的起始物理内存地址，而页表的本质是一个物理内存页，因此页表的起始内存地址也是按照 `4K` 对齐的（仅需要保存下一级页表的高`20`位地址），后 `12` 位全部为 `0`，`32`位页表PDE的布局如下图

![32-bit-pde](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/32bit-pde.png)
![32bit-pde-1.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/32bit-pde-1.png)

权限位微调如下：

-   PDE 中的第 `6` 个比特位脏页标记位无意义了（因为 PDE 指向的是一级页表，页表并不是一个文件页）
-   `PS(7)`： PS 标记为 `0` 的时候，PDE 指向一级页表的起始内存地址，起到页目录项的作用；但当 PS 标记为 `1` 的时候，PDE 就会被内核拿来当做 PTE 使用，特殊之处在于该 PDE 会指向一个大页内存（`4M`大页），如下图所示。此时PDE的高`20`位会用来保存 `4M` 大页的起始内存地址（实际上 `4M` 内存大页的起始地址都是按照 `4M` 对齐的，即`4M` 大页的起始内存地址的后 `22` 位全部为 `0` ，只需要用高`10`位表示就可以了，即`31~22`位）

![32bit-pde-4m-page](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/32bit-pde-4m-page.png)

```CPP
typedef unsigned long	pgdval_t;
```

####  二级页表的寻址过程
根据上文介绍的PTE、PDE，二级页表的寻址过程就不难理解了，这里介绍下二级页表的寻址过程

1、二级页表体系下虚拟内存地址的变化

单级页表下虚拟内存地址中只需要保存页表中的 PTE 偏移即可，二级页表下虚拟内存地址需要多保存一份页目录表中 PDE 的偏移，如下图

![32bit-virtual-address](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/32bit-virtual-address.png)

2、虚拟内存地址的构成

在 `32bit` 位系统中，虚拟内存地址是`32`位，页目录表中的 PDE 和页表中的 PTE 也都是`32`位即`4byte`（页目录表、页表的本质都是一个`4K`大小的物理内存页），那么既然一张页目录表大小为`4K`，那么包含了`4KB/4B=1024` 个 PDE，要寻址这 `1024` 个 PDE 需要 `10` 个 bit，所以虚拟内存地址中的页目录表中 PDE 偏移部分占用 `10bit`；同理一张页表中有 `1024` 个 PTE, 要寻址这个 `1024` 个 PTE 也是需要 `10bit`，即虚拟内存地址中的一级页表中 PTE 偏移 部分也需要占用 `10bit`，这样`32bit`的虚拟内存地址还剩下`12`bit，刚好可以用于寻址一个`4K`大小的物理内存页（正好是一个单位）

3、二级页表体系下的寻址过程

- 当 CPU 访问进程虚拟内存空间中的一个地址时，会先从 cr3 寄存器中获取页目录表的起始物理内存地址，然后从虚拟内存地址中解析出前 `10` bit 的内容作为页目录表中 PDE 的偏移，通过**页目录表起始地址 + 页目录表内偏移 * sizeof(PDE)** 可以定位到该虚拟内存页在页目录表中的 PDE
- PDE 中保存了其指向的一级页表的起始物理内存地址，再从虚拟内存地址中解析出下一个 `10` bit 作为页表中 PTE 的偏移，然后通过**页表起始地址 + 页表内偏移 * sizeof(PTE)**可以定位到虚拟内存页在一级页表中的 PTE
- PTE 中保存了最终映射的物理内存页的起始地址，最后从虚拟内存地址中解析出最后 `12` 个 bit，最终定位到虚拟内存地址对应的物理字节上

![32bit-page-table-transform.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/32bit-page-table-transform.png)

## 0x05    64位：四级页表
二级页表最多只能映射 `4G`（`2^20 * 4k`，即PDE的最大寻址范围*页表大小）的物理内存空间，而 `64` 位系统中，进程的用户态需要寻址`128T` 的虚拟内存空间，内核态也有 `128T` 的虚拟内存空间，二级页表明显是不够的

![64-bit-virtual-memory-address-buju](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/64-bit-virtual-memory-address-buju.png)

`64` 位系统中的四级页表对比 `32` 位系统中的二级页表来说，引入了多两个层级的页目录，分别是四级页表和三级页表，页表整体布局如下图：

![64bit-level4-page-table-buju](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/64bit-level4-page-table-buju.png)

####    64位虚拟内存地址
`64` 位的虚拟内存地址格式如下：

![64bit-virtual-address-1.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/64bit-virtual-address-1.png)

在`64`位系统重，一张页表 `4K` 大小，但是页表中的一个 PTE 占用 `8` 个字节，所以在 `64` 位系统中一张页表只能包含 `512` 个 PTE，所以要寻址页表中这 `512` 个 PTE，只需要用 `9` 个 bit 即可，因此虚拟内存地址中的一级页表中的 PTE 偏移占用 `9` 个 bit 位。而一个 PTE 可以映射 `4K` 大小的物理内存（一个物理内存页），所以在 `64` 位的四级页表体系下，一张一级页表可以映射的物理内存空间大小为 `2M`

对PMD（Page Middle Directory）而言，一张中间页目录 PMD 也是 `4K`，PMD 中的页目录项 `pmd_t` 也是占用 `8byte`，所以一张 PMD 可以容纳 `512` 个 `pmd_t`，在`64` 位虚拟内存地址中的 PMD中的页目录偏移也需要占用 `9` bit。所以如果用PMD来映射大页内存的话，支持映射的size为`2MB*512`，共`1G`物理内存（一个 `pmd_t` 指向一张一级页表，即一个 `pmd_t` 可以映射的物理内存为 `2M`）

同样，对PUD而言，一张上层页目录 PUD 中可以容纳 `512` 个页目录项 `pud_t`，在`64` 位虚拟内存地址中的 PUD中的页目录偏移也需要占用 `9` bit，一个 `pud_t` 指向一张 PMD，因此可以映射 `1G` 的物理内存，所以一张 PUD 可以映射 `512G` 的物理内存

最后，对 PGD而言， 可以容纳的页目录 `pgd_t` 为`512`个，在`64` 位虚拟内存地址中的 PGD中的页目录偏移也需要占用 `9` bit，一个 `pgd_t` 可以映射的物理内存为 `512G`，所以一张 PGD 可以映射的物理内存为 `256TB`

内核相关的常量定义如下：

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/arch/x86/include/asm/pgtable_64_types.h#L47
/*
 * entries per page directory level
 */
//PTRS_PER_PTE：表示一张页表中可以容纳的 PTE 个数
#define PTRS_PER_PTE	512

/* PAGE_SHIFT determines the page size */
//PAGE_SHIFT：表示一个物理内存页的大小 2^PAGE_SHIFT
#define PAGE_SHIFT		12

/*
 * PMD_SHIFT determines the size of the area a middle-level
 * page table can map
 */
//PMD_SHIFT ：表示一个 pmd_t 可以映射的物理内存范围2^PMD_SHIFT
#define PMD_SHIFT	21
#define PTRS_PER_PMD	512

/*
 * 3rd level page
 */
#define PUD_SHIFT	30
#define PTRS_PER_PUD	512

/*
 * 4th level page in 5-level paging case
 */
#define PGDIR_SHIFT		39
#define PTRS_PER_PGD		512
```

![64bit-virtual-address](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/64bit-virtual-address.png)

####  64位虚拟内存地址的操作
上一小节介绍了四级页表中各级页表的容量：

- `PAGE_SHIFT`：表示页表中的一个 PTE 可以映射的物理内存大小（`4K`）
- `PMD_SHIFT`：表示 PMD 中的一个页目录项 `pmd_t` 可以映射的物理内存大小（`2M`）
- `PUD_SHIFT`：表示 PUD 中的一个页目录项 `pud_t` 可以映射的物理内存大小（`1G`）
- `PGD_SHIFT`：表示 PGD 中的一个页目录项 `pgd_t` 可以映射的物理内存大小（`512G`）

上述定义除了表示对应页目录项映射的物理内存大小之外，内核还提供若干宏/函数获取一个 `64` 位虚拟内存地址中获取其在对应页目录中的偏移量，[参考](https://elixir.bootlin.com/linux/v4.11.6/source/arch/x86/include/asm/pgtable.h#L826)

```CPP
// 获取64位虚拟内存地址在 PGD 中的偏移，右移PGDIR_SHIFT
#define pgd_index(address) ( address >> PGDIR_SHIFT) 

// 通过 PGD 的起始内存地址加上 pgd_index 就可以得到虚拟内存地址在 PGD 中的页目录项 pgd_t
#define pgd_offset_pgd(pgd, address) (pgd + pgd_index((address)))

#define PUD_SHIFT	30
#define PTRS_PER_PUD	512
static inline unsigned long pud_index(unsigned long address)
{
  // 将虚拟内存地址右移 PUD_SHIFT 位，并用掩码 PTRS_PER_PUD - 1 掩掉高 9 位 , 只保留低 9 位，就可以得到虚拟内存地址在 PUD 中的偏移
  // PTRS_PER_PUD - 1 转换为二进制是 9 个 1，用来截取最低的 9 个比特位
	return (address >> PUD_SHIFT) & (PTRS_PER_PUD - 1);
}

/* Find an entry in the third-level page table.. */
//pud_offset：通过 pgd_t 获取 PUD 的起始内存地址 + pud_index 得到虚拟内存地址对应的 pud_t（下一级）
static inline pud_t *pud_offset(pgd_t *pgd, unsigned long address)
{
	return (pud_t *)pgd_page_vaddr(*pgd) + pud_index(address);
}

/*--------------------------------------*/
// pmd_offset：获取虚拟内存地址在 PMD 中的页目录项 pmd_t
/* Find an entry in the second-level page table.. */
static inline pmd_t *pmd_offset(pud_t *pud, unsigned long address)
{
	return (pmd_t *)pud_page_vaddr(*pud) + pmd_index(address);
}

static inline unsigned long pmd_index(unsigned long address)
{
	return (address >> PMD_SHIFT) & (PTRS_PER_PMD - 1);
}

/*--------------------------------------*/
//pte_offset_kernel：获取虚拟内存地址在一级页表中的 PTE
static inline pte_t *pte_offset_kernel(pmd_t *pmd, unsigned long address)
{
	return (pte_t *)pmd_page_vaddr(*pmd) + pte_index(address);
}

static inline unsigned long pte_index(unsigned long address)
{
	return (address >> PAGE_SHIFT) & (PTRS_PER_PTE - 1);
}
```

####    64 位页表项 PTE
在 `64`bit系统中，布局如下图，其中 `0~8` 之间的bit意义同 `32` 位一样，在`64`位系统中一个物理内存页仍然是`4K`大小

```CPP
//页表中 PTE 占用64bit
typedef unsigned long   pteval_t;
typedef struct { pteval_t pte; } pte_t;
```

![64bit-pte](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/64bit-pte.png)

####    64 位页目录项
`64` 位四级页表体系下，一共包含了三个层级的页目录：即全局页目录 PGD（Page Global Directory）、上层页目录 PUD（Page Upper Directory）和PMD（Page Middle Directory），其布局如下：

```CPP
typedef unsigned long   pmdval_t;
typedef unsigned long   pudval_t;
typedef unsigned long   pgdval_t;

typedef struct { pmdval_t pmd; } pmd_t;
typedef struct { pudval_t pud; } pud_t;
typedef struct { pgdval_t pgd; } pgd_t;
```

![64bit-pde](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/64bit-pud.png)

重要bit如下：

- `PS(7)`为 `0` 时，该 PDE 指向的是其下一级页目录或者页表的起始内存地址
- `PS(7)`为 `1` 时，该 PDE 指向的就是一个内存大页，对于 PMD 中的页目录项 `pmd_t` 而言，它指向的是一张 `2M` 的物理内存大页；对于 PUD 中的页目录项 `pud_t` 而言，它指向的是一张 `1G` 的物理内存大页。同`32`位二级页表类似，在`64`位的pde结构中，大页内存映射的场景下，第`12~35`位（实际不需要这么多位）会被用来存储大页内存的开始地址

![64bit-pud_t-1G-hugepage]()

####    四级页表寻址
同二级页表寻址过程基本类似，只是说法上稍有区别，四级页表体系下，最顶层的是全局页目录 PGD（Page Global Directory），PGD 中的页目录项叫做 `pgd_t`，PGD 是四级页表体系下的顶级页表，保存在进程 `struct mm_struct` 的 `pgd` 成员中，在进程调度上下文切换的时候，由**内核通过 `load_new_mm_cr3` 方法将 `pgd` 中保存的顶级页表虚拟内存地址转换物理内存地址，随后加载到 cr3 寄存器中，从而完成进程虚拟内存空间的切换**

- PUD（Page Upper Directory）：三级页表，上层页目录，PUD 中的页目录项为`pud_t`
- PMD（Page Middle Directory）：二级页表，中间页目录，PMD 中的页目录项为 `pmd_t`
- Page Table：一级页表，最底层的用来直接映射物理内存页面

所以，在四级页表体系下，首先需要定位顶级页表 PGD 中的页目录项 `pgd_t`，`pgd_t` 指向的 PUD 的起始内存地址，然后在定位 PUD 中的页目录项 `pud_t`，后面的寻址过程和二级页表一样。大致的寻址过程如下图

1.  首先 MMU 会从 cr3 寄存器中获取顶级页目录 `PGD` 的起始内存地址，然后通过 `pgd_index` 从虚拟内存地址中截取 PGD 中的页目录项偏移，这样就定位到了具体的一个 `pgd_t`
2.  `pgd_t` 中保存的是 PMD 的起始内存地址，通过 `pud_index` 可以从虚拟内存地址中截取 PUD 中的页目录项偏移，从而确定虚拟内存地址在 PUD 中的页目录项 `pud_t`
3.  根据 `pud_t` 中保存的 PMD 起始内存地址，再加上通过 `pmd_index` 获取到的 PMD 中的页目录项偏移，定位到虚拟内存地址在 PMD 中的页目录项 `pmd_t`
4.  最后，`pmd_t` 指向具体页表的起始内存地址，通过 `pte_index` 获取虚拟内存地址在一级页表中的 PTE 偏移，最终定位到一个具体的 PTE 中，PTE 指向的正是虚拟内存地址映射的物理内存页面，然后通过虚拟内存地址中的低 `12` 位（物理内存页内偏移），最终确定到一个具体的物理字节（地址）

最后，思考下为什么CR3寄存器里面保存的必须是物理内存地址？

![64-transform](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/64bit-page-table-transform.png)

##  0x06  CPU 的寻址过程
为了加速虚拟内存地址到物理内存地址的转换过程，CPU 专门引入对页表进行遍历的地址翻译硬件 MMU（Memory Management Unit）加速这过程

![MMU](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/pagetable/MMU.png)

##  0x07  总结

####    Huge Page
笔者最早接触大页这个概念是在DPDK开发时，Huge Page（大页）是一种通过增大内存页尺寸来优化内存管理的技术，在特定场景下可显著提升系统性能，优点如下：

- 提升TLB（快表）命中率：TLB用于缓存虚拟地址到物理地址的映射，容量有限（通常`64~512`条目）。如使用`2MB`大页后，单条目可覆盖`2MB`内存（相当于`512`个`4KB`页），显著减少TLB Miss。如此当热点数据访问时，TLB命中率提升，减少地址转换延迟，加速内存访问
- 减少页表开销：页表项缩减，对于`2MB`的内存大页，管理相同内存时，大页的页表项数量降至`1/512`或更低，降低页表内存占用及遍历层级
- 缺页异常优化：分配`2MB`内存需一次缺页异常，对于普通`4K`物理内存页需`512`次，减少内核处理开销
- 避免内存交换Swap：通常大页内存不会被交换到磁盘，保障关键数据（如数据库缓存）常驻物理内存，避免Swap引起的性能抖动
- 降低内存碎片化：大页分配减少小页导致的碎片问题，提升大块连续内存分配成功率

##  0x08  参考
-   [Linux内存管理](https://qiankunli.github.io/2019/05/31/linux_memory.html#%E8%BF%9B%E7%A8%8B%E7%9A%84%E9%A1%B5%E8%A1%A8)
-   [一步一图带你构建 Linux 页表体系：详解虚拟内存如何与物理内存进行映射](https://www.cnblogs.com/binlovetech/p/17571929.html)                                   ,