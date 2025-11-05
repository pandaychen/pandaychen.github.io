---
layout:     post
title:  Linux 内核之旅（三）：虚拟内存管理（上）
subtitle:   进程视角的虚拟内存管理
date:       2024-11-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
前文讨论了进程，正在执行的程序，是可执行程序的动态实例，它是一个承担分配系统资源的实体，但操作系统创建进程时，会为进程创建相应的内存空间，这个内存空间称为进程的地址空间，**每一个进程的地址空间（虚拟内存空间）都是独立的**；当一个进程有了进程的地址空间，那么其管理结构被称为内存描述符`mm_struct`。有趣的说，虚拟内存其实是 CPU 和操作系统使用的一个障眼法，联手给进程编织了一个假象，让进程误以为自己独占了全部的内存空间

本文就来学习下**进程虚拟内存空间在内核中的布局以及管理**，内核版本基于[v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

![linux_memory_management.png]()

####    虚拟内存地址

####    虚拟内存空间分布（32位）

![32-vm]()

`32`位机器上进程的用户空间大小为 `3GB`。内核将一个进程的用户空间划分为多个段：
-   代码段：用于存放程序中可执行代码的段
-   数据段：用于存放已经初始化的全局变量或静态变量的段（如 `int global = 10;` 定义的全局变量）
-   未初始化数据段：用于存放未初始化的全局变量或静态变量的段（如`int global;` 定义的全局变量）
-   堆：用于存放使用 `malloc` 函数申请的内存
-   `mmap`区：用于存放使用 `mmap` 函数映射的内存区（文件映射与匿名映射区），详见后文描述
-   栈：用于存放函数局部变量和函数参数

####    虚拟内存空间分布（64位）

![64-vm]()


####    页表
参考[一步一图带你构建 Linux 页表体系：详解虚拟内存如何与物理内存进行映射](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247488477&idx=1&sn=f8531b3220ea3a9ca2a0fdc2fd9dabc6&chksm=ce77d59af9005c8c2ef35c7e45f45cbfc527bfc4b99bbd02dbaaa964d174a4009897dd329a4d&scene=178&cur_album_id=2559805446807928833#rd)

##  0x01    进程虚拟内存空间管理
无论是在 `32` 位机器上还是在 `64` 位机器上，进程虚拟内存空间的核心区域分布的相对位置是一样的，这个结构是怎么在内核反映的？

![]()

####    进程虚拟内存描述符：mm_struct
已知每个内核进程描述符结构体`task_struct`中嵌套了一个`mm_struct`结构体指针，该结构体中包含了的该进程虚拟内存空间的全部信息。同时每个进程都有唯一的 `mm_struct` 结构体，这说明每个进程的虚拟地址空间都是独立，互不干扰的。当调用 `fork()` 函数创建进程的时候，表示进程地址空间的 `mm_struct` 结构会随着进程描述符 `task_struct` 的创建而创建

创建`mm_struct`的逻辑在内核函数[`copy_mm`](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/fork.c#L1174)中，注意这里区分用户态进程和用户态线程，前者会单独复制出一份`mm_struct`，后者是共享父进程的`mm_struct`。`copy_mm` 函数首先会将父进程的虚拟内存空间 `current->mm` 赋值给指针 `oldmm`，然后通过 `dup_mm` 函数将父进程的虚拟内存空间以及相关页表拷贝到子进程的 `mm_struct` 结构中，最后将拷贝出来的 `mm_struct` 赋值给子进程的 `task_struct` 结构，最终布局[参考](https://pandaychen.github.io/2024/10/02/A-LINUX-KERNEL-TRAVEL-1/#%E8%BF%9B%E7%A8%8B%E4%B8%8E%E7%BA%BF%E7%A8%8B)

```CPP
struct task_struct {
    //.......
    // 内存描述符表示进程虚拟地址空间
    struct mm_struct *mm;
    //.......
}
struct mm_struct {
          struct vm_area_struct * mmap;       /* list of VMAs */
          struct rb_root mm_rb;     /*红黑树的根节点*/
          struct vm_area_struct * mmap_cache;      /* last find_vma result */
        //.......
}
```

`mm_struct`结构体定义如下：

```CPP
struct mm_struct {
    //mmap指向虚拟区间链表
    struct vm_area_struct * mmap;       /* list of VMAs */
    //指向红黑树的根节点
    struct rb_root mm_rb;
    //指向最近的虚拟空间
    struct vm_area_struct * mmap_cache; /* last find_vma result */
    //
    unsigned long (*get_unmapped_area) (struct file *filp,
                unsigned long addr, unsigned long len,
                unsigned long pgoff, unsigned long flags);
    void (*unmap_area) (struct mm_struct *mm, unsigned long addr);
    unsigned long mmap_base;        /* base of mmap area */
    unsigned long task_size;        /* size of task vm space */
    unsigned long cached_hole_size;     /* if non-zero, the largest hole below free_area_cache */
    unsigned long free_area_cache;      /* first hole of size cached_hole_size or larger */
    //指向进程的页目录
    pgd_t * pgd;
    //空间中有多少用户
    atomic_t mm_users;          /* How many users with user space? */
    //引用计数；描述有多少指针指向当前的mm_struct
    atomic_t mm_count;          /* How many references to "struct mm_struct" (users count as 1) */
    //虚拟区间的个数
    int map_count;              /* number of VMAs */
    struct rw_semaphore mmap_sem;
    //保护任务页表
    spinlock_t page_table_lock;     /* Protects page tables and some counters */
    //所有mm的链表
    struct list_head mmlist;        /* List of maybe swapped mm's.  These are globally strung
                         * together off init_mm.mmlist, and are protected
                         * by mmlist_lock
                         */

    /* Special counters, in some configurations protected by the
     * page_table_lock, in other configurations by being atomic.
     */
    mm_counter_t _file_rss;
    mm_counter_t _anon_rss;

    unsigned long hiwater_rss;  /* High-watermark of RSS usage */
    unsigned long hiwater_vm;   /* High-water virtual memory usage */

    unsigned long total_vm, locked_vm, shared_vm, exec_vm;
    unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
    //start_code:代码段的起始地址
    //end_code:代码段的结束地址
    //start_data:数据段起始地址
    //end_data:数据段结束地址
    unsigned long start_code, end_code, start_data, end_data;
    //start_brk:堆的起始地址
    //brk:堆的结束地址
    //start_stack:栈的起始地址
    unsigned long start_brk, brk, start_stack;
    //arg_start,arg_end:参数段的起始和结束地址
    //env_start,env_end:环境段的起始和结束地址
    unsigned long arg_start, arg_end, env_start, env_end;

    unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

    struct linux_binfmt *binfmt;

    cpumask_t cpu_vm_mask;

    /* Architecture-specific MM context */
    mm_context_t context;

    /* Swap token stuff */
    /*
     * Last value of global fault stamp as seen by this process.
     * In other words, this value gives an indication of how long
     * it has been since this task got the token.
     * Look at mm/thrash.c
     */
    unsigned int faultstamp;
    unsigned int token_priority;
    unsigned int last_interval;

    unsigned long flags; /* Must use atomic bitops to access the bits */

    struct core_state *core_state; /* coredumping support */
#ifdef CONFIG_AIO
    spinlock_t      ioctx_lock;
    struct hlist_head   ioctx_list;
#endif
#ifdef CONFIG_MM_OWNER
    /*
     * "owner" points to a task that is regarded as the canonical
     * user/owner of this mm. All of the following must be true in
     * order for it to be changed:
     *
     * current == mm->owner
     * current->mm != mm
     * new_owner->mm == mm
     * new_owner->alloc_lock is held
     */
    struct task_struct *owner;
#endif

#ifdef CONFIG_PROC_FS
    /* store ref to file /proc/<pid>/exe symlink points to */
    struct file *exe_file;
    unsigned long num_exe_file_vmas;
#endif
#ifdef CONFIG_MMU_NOTIFIER
    struct mmu_notifier_mm *mmu_notifier_mm;
#endif
};
```

####    task_size
`task_size`：定义了用户态地址空间与内核态地址空间之间的分界线，如`64` 位系统中用户地址空间和内核地址空间的分界线在 `0x0000 7FFF FFFF F000` 地址处，那么`task_struct.mm_struct` 结构中的 `task_size` 为 `0x0000 7FFF FFFF F000`

![]()

####    进程虚拟内存空间布局（区间端点）

```CPP
struct mm_struct {
    unsigned long task_size;    /* size of task vm space */
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    unsigned long arg_start, arg_end, env_start, env_end;
    unsigned long mmap_base;  /* base of mmap area */
    unsigned long total_vm;    /* Total pages mapped */
    unsigned long locked_vm;  /* Pages that have PG_mlocked set */
    unsigned long pinned_vm;  /* Refcount permanently increased */
    unsigned long data_vm;    /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
    unsigned long exec_vm;    /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
    unsigned long stack_vm;    /* VM_STACK */

    //......
}
```

![mm_struct]()

####    vm_area_struct
Linux 内核按照功能上的差异，把虚拟内存空间划分为多个段。那么在内核中，是通过`vm_area_struct`结构来管理这些段

内核使用结构体 [`vm_area_struct`](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/mm_types.h#L284)来**描述用户虚拟内存空间的各个逻辑区域：代码段，数据段，堆，内存映射区，栈等，也称为VMA（virtual memory area）**

每个 `vm_area_struct` 结构对应于虚拟内存空间中的唯一虚拟内存区域 VMA，其中`vm_start` 指向了这块虚拟内存区域的起始地址（最低地址），`vm_start` 本身包含在这块虚拟内存区域内，`vm_end` 指向了这块虚拟内存区域的结束地址（最高地址），而 `vm_end` 本身包含在这块虚拟内存区域之外，所以 `vm_area_struct` 结构描述的是 `[vm_start，vm_end)` 这样一段左闭右开的虚拟内存区域

```CPP
//内核通过 vm_area_struct 结构（虚拟内存区）来管理各个段
struct vm_area_struct {
    struct mm_struct *vm_mm; /* The address space we belong to. */
	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

    /* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;
	/*
	 * Access permissions of this VMA.
	 */
	pgprot_t vm_page_prot;
	unsigned long vm_flags;	

    //每个 VMA 区域都是红黑树中的一个节点，通过 struct vm_area_struct 结构中的 vm_rb 将自己连接到红黑树中
    struct rb_node vm_rb;

	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */
    struct file * vm_file;		/* File we map to (can be NULL). */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units */	
	void * vm_private_data;		/* was vm_pte (shared mem) */
	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops;
}
```

-   `vm_mm`：指向进程的内存管理对象，每个进程都有一个类型为 `mm_struct` 的内存管理对象，用于管理进程的虚拟内存空间和内存映射等
-   `vm_start`：虚拟内存区的起始虚拟内存地址
-   `vm_end`：虚拟内存区的结束虚拟内存地址
-   `vm_next`：Linux 会通过链表把进程的所有虚拟内存区连接起来，这个字段用于指向下一个虚拟内存区
-   `vm_page_prot`：主要用于保存当前虚拟内存区所映射的物理内存页的读写权限
-   `vm_flags`：标识当前虚拟内存区的功能特性
-   `vm_rb`：某些场景中需要通过虚拟内存地址查找对应的虚拟内存区，为了加速查找过程，内核以虚拟内存地址作为key，把进程所有的虚拟内存区保存到一棵红黑树中，而这个字段就是红黑树的节点结构
-   `vm_ops`：每个虚拟内存区都可以自定义一套操作接口，通过操作接口，能够让虚拟内存区实现一些特定的功能，比如：把虚拟内存区映射到文件。而 `vm_ops` 字段就是虚拟内存区的操作接口集，一般在创建虚拟内存区时指定

内核通过一个链表和一棵红黑树来管理进程中所有的段。`mm_struct` 结构的 `mmap` 字段就是链表的头节点，而 `mm_rb` 字段就是红黑树的根节点，如下：

![vm_area_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/vm-area-struct-layout.png)

####    mm_rb && mmap

```CPP
struct mm_struct {
     struct rb_root mm_rb;              //红黑树的root

     //vm_area_struct 链表首节点
     struct vm_area_struct *mmap;		/* list of VMAs */
}
```

几个要点：

-   `mmap`：串联起了整个虚拟内存空间中的虚拟内存区域，即一个双向链表将虚拟内存空间中的这些虚拟内存区域 VMA 串联起来。`vm_area_struct` 结构中的 `vm_next`/`vm_prev` 指针分别指向 VMA 节点所在双向链表中的后继和前驱节点，内核中的这个 VMA 双向链表是有顺序的，所有 VMA 节点按照低地址到高地址的增长方向排序。此外，双向链表中的最后一个 VMA 节点的 `vm_next` 指向 `NULL`，双向链表的头指针存储在内存描述符 `struct mm_struct` 结构中的 `mmap`成员
-   在每个虚拟内存区域 VMA 中又通过 `struct vm_area_struct` 中的 `vm_mm` 指针指向了所属的虚拟内存空间 `mm_struct`
-   可通过 `cat /proc/pid/maps`/`pmap pid` 查看进程的虚拟内存空间布局以及其中包含的所有内存区域（其实现原理就是通过遍历内核中的这个 `vm_area_struct` 双向链表获取）
-   `mm_rb`：红黑树的root节点，每个`vm_area_struct`都包含了`struct rb_node vm_rb`成员即为红黑树的节点
-   在内核中，**同样的内存区域 `vm_area_struct` 会有两种组织形式，一种是双向链表用于高效的遍历，另一种就是红黑树用于高效的查找**

使用rbtree组织VMA区域的场景如下：
-   需要根据特定虚拟内存地址在虚拟内存空间中查找特定的VMA虚拟内存区域，尤其在进程虚拟内存空间中包含的内存区域 VMA 比较多的情况下，使用rbtree查找更为高效

![final](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/mm_struct_all_view.jpg)

##  0x02    进程虚拟内存管理：ELF加载
本小节主要讨论下进程的虚拟内存区是如何建立起来的，主要涉及到：
-   ELF文件

####    ELF：Executable and Linkable Format
在 Linux 系统中使用ELF格式来存储一个可执行的应用程序，一个 ELF 文件由以下三部分组成：

![elf]()

-   ELF 头（ELF header）：描述应用程序的类型、CPU架构、入口地址、程序头表偏移和节头表偏移等
-   程序头表（Program header table）：列举了所有有效的段（segments）和其属性，程序头表需要加载器将文件中的段加载到虚拟内存段中
-   节头表（Section header table）：包含对节（sections）的描述

当内核加载一个应用程序时，就是通过读取 ELF 文件的信息，然后把文件中所有的段加载到虚拟内存的段中。ELF 文件通过程序头表`elf64_phdr`来描述应用程序中所有的段，表中的每一个项都描述一个段的信息，程序加载器可以通过 ELF 头中获取到程序头表的偏移量，然后通过程序头表的偏移量读取到程序头表的数据，再通过程序头表来获取到所有段的信息

```CPP
typedef struct elf64_phdr {
    Elf64_Word p_type;     // 段的类型
    Elf64_Word p_flags;    // 可读写标志
    Elf64_Off p_offset;    // 段在ELF文件中的偏移量
    Elf64_Addr p_vaddr;    // 段的虚拟内存地址
    Elf64_Addr p_paddr;    // 段的物理内存地址
    Elf64_Xword p_filesz;  // 段占用文件的大小
    Elf64_Xword p_memsz;   // 段占用内存的大小
    Elf64_Xword p_align;   // 内存对齐
} Elf64_Phdr;
```

####    ELF的加载过程
要加载一个程序，需要调用 `execve` 系统调用来完成， `execve` 系统调用的调用栈如下，`execve` 系统调用最终会调用 `load_elf_binary` 函数来加载程序的 ELF 文件，接下来简单描述下`load_elf_binary`的过程

```BASH
sys_execve
 ->do_execve
    ->do_execveat_common
       -> __do_execve_file
           -> exec_binprm
              -> search_binary_handler
                 -> load_elf_binary
```

####    load_elf_binary的实现

1、读取并检查ELF头

```CPP
//这段代码的逻辑主要是读取应用程序的 ELF 头，然后检查 ELF 头信息是否合法
static int load_elf_binary(struct linux_binprm *bprm, struct pt_regs *regs)
{
    //...
    struct {
        struct elfhdr elf_ex;
        struct elfhdr interp_elf_ex;
    } *loc;

    loc = kmalloc(sizeof(*loc), GFP_KERNEL);
    if (!loc) {
        retval = -ENOMEM;
        goto out_ret;
    }

    // 1. 获取ELF头
    loc->elf_ex = *((struct elfhdr *)bprm->buf);

    retval = -ENOEXEC;
    // 2. 检查ELF签名是否正确
    if (memcmp(loc->elf_ex.e_ident, ELFMAG, SELFMAG) != 0)
        goto out;

    // 3. 是否是可执行文件或者动态库
    if (loc->elf_ex.e_type != ET_EXEC && loc->elf_ex.e_type != ET_DYN)
        goto out;

    // 4. 检查系统架构是否正确
    if (!elf_check_arch(&loc->elf_ex))
        goto out;
    //...
}
```

2、读取程序头表，主要包含如下流程：

-   从 ELF 头的信息中获取到程序头表的大小
-   调用 `kmalloc` 函数申请一块内存来保存程序头表
-   调用 `kernel_read` 函数从 ELF 文件中读取程序头表的数据，保存到 `elf_phdata` 变量中，程序头表的偏移量可以通过 ELF 头的 `e_phoff` 字段获取

```CPP
static int load_elf_binary(struct linux_binprm *bprm, struct pt_regs *regs)
{
    // ......
    // ......
    // 接上面的代码
    size = loc->elf_ex.e_phnum * sizeof(struct elf_phdr); // 程序头表的大小
    retval = -ENOMEM;

    elf_phdata = kmalloc(size, GFP_KERNEL); // 申请一块内存来保存程序头表
    if (!elf_phdata)
        goto out;

	// 从ELF文件中读取程序头表的数据, 并且保存到 elf_phdata 变量中
    retval = kernel_read(bprm->file, loc->elf_ex.e_phoff, (char *)elf_phdata, size);
    if (retval != size) {
        if (retval >= 0)
            retval = -EIO;
        goto out_free_ph;
    }
    //...
}
```

3、加载段到虚拟内存，把段加载到虚拟内存主要通过 `elf_map` 函数完成

-   遍历程序头表所有的段
-   判断段是否需要加载
-   获取段的可读写权限和段的虚拟内存地址
-   调用 `elf_map` 函数把段加载到虚拟内存

```CPP
    // 遍历程序头表所有的段
    for (i = 0, elf_ppnt = elf_phdata; i < loc->elf_ex.e_phnum; i++, elf_ppnt++) {
        int elf_prot = 0, elf_flags;
        unsigned long k, vaddr;

        if (elf_ppnt->p_type != PT_LOAD)  // 判断段是否需要加载
            continue;
        //...
        // 段的可读写权限
        if (elf_ppnt->p_flags & PF_R)
            elf_prot |= PROT_READ;
        if (elf_ppnt->p_flags & PF_W)
            elf_prot |= PROT_WRITE;
        if (elf_ppnt->p_flags & PF_X)
            elf_prot |= PROT_EXEC;

        elf_flags = MAP_PRIVATE | MAP_DENYWRITE | MAP_EXECUTABLE;

        vaddr = elf_ppnt->p_vaddr;  // 获取段的虚拟内存地址
        //...
        // 把段加载到虚拟内存
        error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt, elf_prot, elf_flags, 0);
        //...
    }
```

4、 `elf_map` 函数的调用栈及核心流程，`elf_map` 函数最终会调用 `mmap_region` 来完成加载段到虚拟内存

```BASH
elf_map
 |--> do_mmap
   |--> do_mmap_pgoff
      |--> mmap_region
```

`mmap_region`的主要流程如下：

-   调用 `kmem_cache_zalloc` 函数申请一个 `vm_area_struct`（虚拟内存区）结构
-   设置 `vm_area_struct` 结构各个字段的值
-   调用 `vma_link` 函数把 `vm_area_struct` 结构连接到虚拟内存区链表和红黑树中，`vma_link`[实现](https://elixir.bootlin.com/linux/v4.11.6/source/mm/mmap.c#L589)
-   通过上面的过程，内核就把应用程序的所有段加载到虚拟内存中

```CPP
unsigned long 
mmap_region(struct file *file, unsigned long addr, unsigned long len, 
            unsigned long flags, unsigned int vm_flags, unsigned long pgoff)
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma, *prev;
    //...
    // 申请一个 vm_area_struct 结构
    vma = kmem_cache_zalloc(vm_area_cachep, GFP_KERNEL);
    if (!vma) {
        error = -ENOMEM;
        goto unacct_error;
    }

    // 设置 vm_area_struct 结构各个字段的值
    vma->vm_mm = mm;
    vma->vm_start = addr;        // 段的开始虚拟内存地址
    vma->vm_end = addr + len;    // 段的结束虚拟内存地址
    vma->vm_flags = vm_flags;    // 段的功能特性
    vma->vm_page_prot = vm_get_page_prot(vm_flags);
    vma->vm_pgoff = pgoff;

    //...
    // 把 vm_area_struct 结构链接到虚拟内存区链表和红黑树中
    vma_link(mm, vma, prev, rb_link, rb_parent);
    //...
    
    return addr;
}
```

####    ELF加载之后的内存布局
以下面代码为例：
```CPP
int main()
{
   printf("Hello, World!\n");
   int *a=malloc(1024*1024);
   memset(a,0,1024*1024);
   getchar();
   return 0;
}
```

可以通过`/proc/${pid}/maps`查看内存布局，如下：

```BASH
[root@VM-X-X-tencentos ~]# cat /proc/1115271/maps 
00400000-00401000 r--p 00000000 fd:01 1892501                            /root/main
00401000-00402000 r-xp 00001000 fd:01 1892501                            /root/main
00402000-00403000 r--p 00002000 fd:01 1892501                            /root/main
00403000-00404000 r--p 00002000 fd:01 1892501                            /root/main
00404000-00405000 rw-p 00003000 fd:01 1892501                            /root/main
0178a000-017ab000 rw-p 00000000 00:00 0                                  [heap]
7f7d46a0c000-7f7d46a32000 r--p 00000000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46a32000-7f7d46b98000 r-xp 00026000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46b98000-7f7d46bed000 r--p 0018c000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46bed000-7f7d46bf1000 r--p 001e0000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46bf1000-7f7d46bf3000 rw-p 001e4000 fd:01 26356345                   /usr/lib64/libc.so.6
7f7d46bf3000-7f7d46c00000 rw-p 00000000 00:00 0 
7f7d46c00000-7f7d46c03000 r-xp 00000000 fd:01 28659740                   /usr/lib64/libonion_security.so.1.0.19
7f7d46c03000-7f7d46d03000 ---p 00003000 fd:01 28659740                   /usr/lib64/libonion_security.so.1.0.19
7f7d46d03000-7f7d46d04000 rw-p 00003000 fd:01 28659740                   /usr/lib64/libonion_security.so.1.0.19
7f7d46d04000-7f7d46d06000 rw-p 00000000 00:00 0 
7f7d46e00000-7f7d46e03000 r-xp 00000000 fd:01 25582409                   /usr/lib64/libtdsp_security.so.1.0.25
7f7d46e03000-7f7d47002000 ---p 00003000 fd:01 25582409                   /usr/lib64/libtdsp_security.so.1.0.25
7f7d47002000-7f7d47003000 rw-p 00002000 fd:01 25582409                   /usr/lib64/libtdsp_security.so.1.0.25
7f7d47140000-7f7d47142000 rw-p 00000000 00:00 0 
7f7d47142000-7f7d47143000 r--p 00000000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d47143000-7f7d47144000 r-xp 00001000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d47144000-7f7d47145000 r--p 00002000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d47145000-7f7d47146000 r--p 00002000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d47146000-7f7d47147000 rw-p 00003000 fd:01 26356431                   /usr/lib64/libdl.so.2
7f7d4714e000-7f7d47150000 rw-p 00000000 00:00 0 
7f7d47151000-7f7d47152000 r--p 00000000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7f7d47152000-7f7d47179000 r-xp 00001000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7f7d47179000-7f7d47183000 r--p 00028000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7f7d47183000-7f7d47185000 r--p 00032000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7f7d47185000-7f7d47187000 rw-p 00034000 fd:01 26345834                   /usr/lib64/ld-linux-x86-64.so.2
7ffddfc95000-7ffddfcb6000 rw-p 00000000 00:00 0                          [stack]
7ffddfd94000-7ffddfd98000 r--p 00000000 00:00 0                          [vvar]
7ffddfd98000-7ffddfd9a000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```

在Linux中，进程运行时的物理内存消耗主要通过驻留集大小（RSS）来衡量。RSS表示进程当前使用的物理内存页大小，包括代码段、数据段、堆、栈以及共享库部分。需要注意的是`malloc`分配的是虚拟内存，只有在实际访问（如 `memset`）时，操作系统才会通过页错误机制（即缺页中断）分配物理内存。代码段和共享库段通常被多个进程共享，因此从系统总体来看，物理内存消耗可能较少，但从单个进程的RSS角度，这些共享页仍然会被计入

对于上述这段程序，物理内存消耗主要来自以下部分：

-   程序代码段（text segment）：可执行文件本身的代码。本例中，代码段通常只有几KB
-   数据段（data segment）和BSS段：本例中无全局变量，所以数据段和BSS段很小（可能小于`1KB`），物理内存消耗可忽略
-   堆（heap）：本例子中，通过 `malloc`分配了`1MB`虚拟内存，并通过 `memset`访问全部内容，因此这`1MB`会完全占用物理内存（内存充裕时）；如果代码中没有调用 `memset`，那么 `malloc`分配的`1MB`可能不会立即占用物理内存（直到访问时才分配）
-   栈（stack）：用于局部变量和函数调用，消耗很小（通常几KB），物理内存可忽略
-   共享库（shared libraries）：程序使用 `printf`和 `getchar`（C标准库）。共享库的代码和数据会被映射到进程地址空间。物理内存会加载库中实际使用的部分（如I/O函数、缓冲区等），典型值为几百KB到`1MB`。这些页可能与其他进程共享，但RSS会计入进程使用的部分
-   其他开销：进程内核数据结构（如页表、任务结构）消耗少量物理内存（通常几KB）


####    vm_area_struct的分配
把上面的例子修改一下：

```CPP
int main() {
   printf("Hello, World!\n");
   int *a = malloc(1024 * 1024); // 分配1MB
   memset(a, 0, 1024 * 1024);
   int *b = malloc(1024 * 1024); // 再分配1MB
   memset(b, 0, 1024 * 1024);
   getchar();
   return 0;
}
```

虚拟内存布局如下：
```bash
[root@VM-119-175-tencentos ~]# cat /proc/1960685/maps 
00400000-00401000 r--p 00000000 fd:01 567111                             /root/main
00401000-00402000 r-xp 00001000 fd:01 567111                             /root/main
00402000-00403000 r--p 00002000 fd:01 567111                             /root/main
00403000-00404000 r--p 00002000 fd:01 567111                             /root/main
00404000-00405000 rw-p 00003000 fd:01 567111                             /root/main
00e2a000-00e4b000 rw-p 00000000 00:00 0                                  [heap]
7fde7ed15000-7fde7ef17000 rw-p 00000000 00:00 0 
7fde7ef17000-7fde7ef3d000 r--p 00000000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7ef3d000-7fde7f098000 r-xp 00026000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7f098000-7fde7f0ed000 r--p 00181000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7f0ed000-7fde7f0f1000 r--p 001d5000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7f0f1000-7fde7f0f3000 rw-p 001d9000 fd:01 396930                     /usr/lib64/libc.so.6
7fde7f0f3000-7fde7f100000 rw-p 00000000 00:00 0 
7fde7f100000-7fde7f103000 r-xp 00000000 fd:01 404268                     /usr/lib64/libonion_security.so.1.0.19
7fde7f103000-7fde7f203000 ---p 00003000 fd:01 404268                     /usr/lib64/libonion_security.so.1.0.19
7fde7f203000-7fde7f204000 rw-p 00003000 fd:01 404268                     /usr/lib64/libonion_security.so.1.0.19
7fde7f204000-7fde7f206000 rw-p 00000000 00:00 0 
7fde7f286000-7fde7f289000 rw-p 00000000 00:00 0 
7fde7f289000-7fde7f28a000 r--p 00000000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f28a000-7fde7f28b000 r-xp 00001000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f28b000-7fde7f28c000 r--p 00002000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f28c000-7fde7f28d000 r--p 00002000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f28d000-7fde7f28e000 rw-p 00003000 fd:01 397016                     /usr/lib64/libdl.so.2
7fde7f297000-7fde7f299000 rw-p 00000000 00:00 0 
7fde7f29a000-7fde7f29b000 r--p 00000000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7fde7f29b000-7fde7f2c1000 r-xp 00001000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7fde7f2c1000-7fde7f2cb000 r--p 00027000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7fde7f2cb000-7fde7f2cd000 r--p 00031000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7fde7f2cd000-7fde7f2cf000 rw-p 00033000 fd:01 396776                     /usr/lib64/ld-linux-x86-64.so.2
7ffc30041000-7ffc30062000 rw-p 00000000 00:00 0                          [stack]
7ffc300f2000-7ffc300f6000 r--p 00000000 00:00 0                          [vvar]
7ffc300f6000-7ffc300f8000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```

下面结合`mm_struct`的成员及上述布局，稍微解读下上述结果：

-   **在 Linux 内核中，每个在 `/proc/pid/maps`中的行对应一个 `vm_area_struct`结构体，因此在当前机器上（运行的进程），该进程共有 `33` 个独立的虚拟内存区域，因此内核会创建 `33` 个`vm_area_struct`结构体，这些`vm_area_struct`被内核通过红黑树来实现高效查找**
-   程序固有区域：包括代码段、数据段、BSS段等（对应上述 maps 文件的前`5`行），这些是进程启动时由 `execve`系统调用映射的
-   堆区域（heap）：地址区间为 `00e2a000-00e4b000`，这是通过 `brk`系统调用管理的堆区域，用于小块内存分配
-   匿名内存映射区域：这是由 `malloc`分配大块内存（如`1MB`）时通过 `mmap`创建的，在上述 maps 文件中，有一个关键的匿名映射区域即`7fde7ed15000-7fde7ef17000 rw-p 00000000 00:00 0`，该区域大小为 `0x7fde7ef17000 - 0x7fde7ed15000 = 0x200000`字节（即`2MB`），权限为可读可写（`rw-p`），且是匿名映射（无文件备份）。这个`2MB`的虚拟内存区域对应代码中的两次`1MB`的`malloc`分配。由于这`2`次分配在虚拟地址空间中是连续的，并且具有相同的权限（`rw-p`），内核可能将它们合并为一个连续的 `vm_area_struct`区域（即一个2MB区域）
-   其他匿名映射：maps 文件中还有多个较小的匿名映射区域（如 `7fde7f0f3000-7fde7f100000`、`7fde7f204000-7fde7f206000`等），这些可能来自共享库的内部分配（如`libc.so.6`、`libdl.so.2`、`ld-linux-x86-64.so.2`等）或其它动态内存分配
-   其他等特殊区域：如栈（stack）、vvar、vdso、vsyscall 等


结合前文的描述，`vm_area_struct` 的管理方式大致如下：
-   内存描述符（`mm_struct`）：每个进程有一个 `mm_struct`结构体，其中包含进程虚拟内存管理的所有信息。`mm_struct`中的 `mmap`字段指向一个链表，该链表按虚拟地址顺序连接所有 `vm_area_struct`结构体
-   红黑树：关联`mm_struct`中的 `mm_rb`字段，用于按虚拟地址快速查找 `vm_area_struct`。插入、删除和查找操作的时间复杂度 `O(logN)`
-   区域合并：当新的内存区域被映射（如通过 `mmap`）时，内核会检查它与相邻区域是否具有相同的权限和映射类型。如果满足条件，内核会合并这些区域为一个更大的 `vm_area_struct`，从而减少结构体数量
-   缺页中断处理：当进程访问虚拟内存时，如果页面不在物理内存中，会触发缺页中断。内核通过查找 `vm_area_struct`来确认访问的合法性，并分配物理页面

##  0x03  mmap的原理分析

####    mmap API
```CPP
#include <sys/mman.h>
void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset);

// 内核文件：/arch/x86/kernel/sys_x86_64.c
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, off)
```

`mmap` 内存映射里所谓的内存其实指的是虚拟内存，常见有两种场景：
-   匿名映射：在调用 `mmap` 进行匿名映射的时候（比如进行堆内存的分配），是将进程虚拟内存空间中的某一段虚拟内存区域与物理内存中的匿名内存页进行映射
-   文件映射：当调用 `mmap` 进行文件映射的时候，是将进程虚拟内存空间中的某一段虚拟内存区域与磁盘中某个文件中的某段区域进行映射

####    mmap的参数&&功能
`mmap`的参数决定了申请虚拟内存的起始地址与size（`addr`/`length` 必须要按照 `PAGE_SIZE` `4k`对齐）

-   `addr`：表示要映射的这段虚拟内存区域在进程虚拟内存空间中的起始地址（虚拟内存地址），一般会设置为 `NULL`，意思就是完全交由内核来帮决定虚拟映射区的起始地址
-   `length`：表示要申请的这段虚拟内存的size。如果是匿名映射，`length` 决定了要映射的匿名物理内存有多大；如果是文件映射，`length`决定了要映射的文件区域有多大

mmap常见的参数组合搭配如下：

-   私有匿名映射：`MAP_PRIVATE | MAP_ANONYMOUS` 表示私有匿名映射，常常利用这种映射方式来申请虚拟内存。比如使用 glibc的 `malloc` 函数进行虚拟内存申请时，当申请的内存大于 `128K` 的时候，`malloc` 就会调用 `mmap` 采用私有匿名映射的方式来申请堆内存。由于它是私有映射，所以申请到的内存是进程独占的，多进程之间不能共享。特别强调一下 `mmap` 私有匿名映射申请到的只是虚拟内存，内核只是在进程虚拟内存空间中划分一段虚拟内存区域 VMA 出来，并将 VMA 该初始化的属性初始化好，mmap 系统调用就结束了，这里和物理内存还没有发生任何关系
-   私有文件映射：调用 `mmap` 进行内存文件映射的时候可以通过指定参数 `flags` 为 `MAP_PRIVATE`，然后将参数 `fd` 指定为要映射文件的文件描述符来实现对文件的私有映射
-   共享文件映射：通过将 `mmap` 系统调用中的 `flags` 参数指定为 `MAP_SHARED` , 参数 `fd` 指定为要映射文件的文件描述符来实现对文件的共享映射，共享文件映射其实和私有文件映射前面的映射过程是一样的，唯一不同的点在于**私有文件映射仅仅是读共享的，写的时候会发生写时复制（copy on write），并且多进程针对同一映射文件的修改不会回写到磁盘文件上**。而共享文件映射因为是共享的，多个进程中的虚拟内存映射区最终会通过缺页中断的方式映射到文件的 page cache 中，后续多个进程对各自的这段虚拟内存区域的读写都会直接发生在 page cache 上。因为映射文件的 page cache 在内核中只有一份，所以对于共享文件映射来说，多进程读写都是共享的，由于多进程直接读写的是 page cache ，所以多进程对共享映射区的任何修改，最终都会通过内核回写线程 pdflush 刷新到磁盘文件中
-    共享匿名映射：通过将 `mmap` 系统调用中的 `flags` 参数指定为 `MAP_SHARED | MAP_ANONYMOUS`，并将 `fd` 参数指定为 `-1` 来实现共享匿名映射，常用于父子进程之间共享内存通信

例如，基于`mmap`实现共享内存（文件映射）的示例代码如下：

```CPP
int start_mmap(){
    /*
    #define SHM_SIZE 1024
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);  // POSIX共享内存文件
    ftruncate(fd, SHM_SIZE);
    */
    int fd = open(name, O_RDWR | O_CREAT, 0666);	

	if (fd == -1) {
		exit(-1);
	}

	//set total_size
	TotalSizeInit();

	ret = ftruncate(fd, total_size);
	if (ret == -1) {
		exit(-1);
	} 

	mem_base = (char *)mmap(NULL, total_size,
				PROT_READ | PROT_WRITE,
				MAP_SHARED, fd, 0); 
	if (mem_base == MAP_FAILED) {
		exit(-1);
	}

	if (mlock_open_flag == OPEN_MLOCK) {	
		ret = mlock(mem_base, total_size);
		if (ret == -1) {
			exit(-1);
		} 
	}
    //....
}
```

####    mmap的原理
mmap 系统调用的本质是首先要在进程虚拟内存空间里的文件映射与匿名映射区中划分出一段虚拟内存区域 VMA 出来 ，这段 VMA 区域的大小用 `vm_start`/`vm_end` 来表示，它们由 `mmap` 系统调用参数 `addr`/`length` 决定；进一步说，`mmap` 内存文件映射的本质其实就是将虚拟映射区 vma 的相关操作 `vma->vm_ops` 映射成文件的相关操作 `ext4_file_vm_ops`（以ext4文件系统为例）

```CPP
struct vm_area_struct {
    unsigned long vm_start;     /* Our start address within vm_mm. */
    unsigned long vm_end;       /* The first byte after our end address */

    //with fd
    struct file * vm_file;      /* File we map to (can be NULL). */
    unsigned long vm_pgoff;     /* Offset (within vm_file) in PAGE_SIZE */
}
```

随后内核会对该段 VMA 进行相关的映射，如果是文件映射的话，内核会将要映射的文件，以及要映射的文件区域在文件中的 `offset`，与 VMA 结构中的 `vm_file`/`vm_pgoff` 关联映射起来，它们由 `mmap` 系统调用参数 `fd`/`offset` 决定。由 `mmap` 在文件映射与匿名映射区中映射出来的这一段虚拟内存区域同进程虚拟内存空间中的其他虚拟内存区域一样，也都是有权限控制的

####    mmap文件映射的原理

![mmap-file-mapping](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/memory/mmap/mmap-file-mapping.png)

由上图，调用 `mmap` 进行内存文件映射的时候，内核首先会在进程的虚拟内存空间中创建一个新的虚拟内存区域 VMA 用于映射文件，通过 `vm_area_struct->vm_file` 将映射文件的 `struct file` 结构与虚拟内存映射关联起来，即下面的成员：

```CPP
struct vm_area_struct {
    //...
    struct file * vm_file;      /* File we map to (can be NULL). */
    unsigned long vm_pgoff;     /* Offset (within vm_file) in PAGE_SIZE */
    // ...
}

struct inode {
    //...
    struct address_space	*i_mapping;
}

//page cache 在内核中是使用 struct address_space 结构来描述
//page cache 是和文件相关的，与进程无关
struct address_space {
    // 这里就是 page cache，里边缓存了文件的所有缓存页面
    struct radix_tree_root  page_tree; 
}
```

根据[Linux 内核之旅（二）：VFS（基础篇）](https://pandaychen.github.io/2024/11/20/A-LINUX-KERNEL-TRAVEL-3/)一文的介绍，`vm_file->f_inode` 可以关联到映射文件的 `struct inode`，近而关联到映射文件在磁盘中的磁盘块 `i_block`以及对应的page cache成员`i_mapping`等（内核会将已经访问过的磁盘块缓存在page cache中）

在VFS中，映射文件中的数据是按照磁盘块来存储的，读写文件数据也是按照磁盘块为单位进行的，磁盘块大小为 `4K`，当进程读取磁盘块的内容到内存之后，站在内存管理系统的视角，磁盘块中的数据被 DMA 拷贝到了物理内存页（即文件页）中。一个文件包含多个磁盘块，当它们被读取到内存之后，一个文件也就对应了多个文件页，这些文件页在内存中统一被 page cache 的结构所组织。每一个文件在内核中都会有一个唯一的 page cache 与之对应，用于缓存文件中的数据

##  0x04    mmap应用的四大场景

####    方式1：私有匿名映射
-   场景1：`malloc`申请大块（虚拟）内存
-   场景2：在`execve`系统调用中，在当前进程中加载并执行一个新的二进制执行文件

####    方式2：私有文件映射

####    方式3：共享文件映射

####    方式4：匿名文件映射

##  0x05  参考
-   [4.6 深入理解 Linux 虚拟内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem.html)
-   [4.7 深入理解 Linux 物理内存管理](https://www.xiaolincoding.com/os/3_memory/linux_mem2.html#_6-1-%E5%8C%BF%E5%90%8D%E9%A1%B5%E7%9A%84%E5%8F%8D%E5%90%91%E6%98%A0%E5%B0%84)
-   [mmap 源码分析](https://leviathan.vip/2019/01/13/mmap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
-   [linux源码解读（十六）：红黑树在内核的应用虚拟内存管理](https://www.cnblogs.com/theseventhson/p/15820092.html)
-   [图解 Linux 虚拟内存空间管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-virtual-memory-manage.md)
-   [认真分析mmap：是什么为什么怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)
-   [从内核世界透视 mmap 内存映射的本质（原理篇）](https://www.cnblogs.com/binlovetech/p/17712761.html)
-   [聊聊Linux内核](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg2MzU3Mjc3Ng==&action=getalbum&album_id=2559805446807928833&scene=173&from_msgid=2247486879&from_itemidx=1&count=3&nolastread=1#wechat_redirect)
-   [一步一图带你深入理解 Linux 虚拟内存管理](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486732&idx=1&sn=435d5e834e9751036c96384f6965b328&chksm=ce77cb4bf900425d33d2adfa632a4684cf7a63beece166c1ffedc4fdacb807c9413e8c73f298&token=1931867638&lang=zh_CN#rd)
-   [你写的代码是如何跑起来的](https://mp.weixin.qq.com/s/1bdktqYF7VyAMadRlcRrSg)