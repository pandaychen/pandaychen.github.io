---
layout:     post
title:  Linux 内核之旅（四）：进程调度基础
subtitle:   进程调度的大白话
date:       2025-01-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
Linux进程调度的本质是，在有限CPU下（进程数目远远超过CPU的数目）需要依据某种算法调度进程，有效地分配CPU的时间，既要保证进程的最快响应，也要保证进程之间的公平

##  0x01   进程调度基础知识 

####    CPU视角
本小节描述下CPU视角下的CPU的工作机制，思考这个问题，CPU是如何在用户程序之间、内核代码与用户程序之间切换的？从CPU视角来看，是如何访问`task_struct`结构的？

1、CPU的工作流程

![CPU-WORK-FLOW]()

如上图所示，CPU的**指令指针寄存器**，指向的就是当前运行的指令的位置，把这里的指令读进CPU执行，如果需要取数据，则将数据读进数据寄存器里面来，进行运算，运算的结果从寄存器写入内存。CPU只要一加电就会按照这个模式运行下去，指令指针寄存器指向A进程的代码某一行，就运行A进程的逻辑，指向Linux内核的代码某一行，就运行内核的逻辑，一直像这样运行下去

2、CPU的视角是如何访问`task_struct`结构的？

![init_task]()

通过前文已知，内核中使用`task_struct`数据结构来表示进程（线程），并挂在一个[进程列表](https://github.com/torvalds/linux/blob/master/init/init_task.c#L66)（链表头）`struct task_struct init_task`中，该结构`init_task`是内核启动的时候创建的，通过此数据结构可以遍历找到所有的进程。那么从CPU视角来看，如何定位到`struct task_struct init_task`这个数据结构？

3、`init_task`存储的位置&&虚拟内存地址空间

回顾前文Linux内核内存管理的知识，不管内核还是用户程序代码OR数据，都是放在物理内存的某个位置的。但是从开发者角度来说基本都是在和虚拟内存打交道（内存很难触碰），再由内核完成虚拟内存地址到物理内存地址的映射。这里先看下`32`/`64`位机器的虚拟内存布局，如下图：

![virtual-memory-management]()

以`64`位机器为例，虚拟内存地址空间又划分为两块：
-   内核态地址空间
-   用户态地址空间：即用户程序的视角，用户的代码只可能访问用户态地址空间（通常），这部分地址不同的程序访问的地址都是重复的，只不过Linux要想办法把这部分各个程序都重复的空间对应到物理内存不同的地方，当然不用都对应，哪里有数据对应哪里，毕竟物理空间没有这么大的地方

用户态地址空间的布局如下图所示，主要包含如下数据：
-   代码段，全局变量等
-   函数栈
-   堆
-   内存映射区

![user-space]()

内核地址空间布局如下图，主要包含以下数据（在用户程序的视角看内核地址空间，只是看起来有但是无法访问）：
-   内核的代码、全局变量等
-   内核数据结构（上面提到的`task_struct`结构，就放在内核态地址空间里面的数据结构区域）
-   内核栈
-   内核中动态分配的内存

![kernel-space]()

1.  用户态程序如果想访问内核态地址空间，就需要从用户态通过例如系统调用进入内核，进入内核后，视角就变成了内核视角，在内核视角，这片地址空间只有一份，无论从哪个进程进入的内核态，进来后访问的是同一份，对应的也是物理内存中同一块空间
2.  在内核视角看起来，用户态的那部分地址空间其实没多少意义，因为每个进程都有这么一块（虚拟地址）空间
3.  那么从CPU硬件视角出发，用户态地址空间和内核态地址空间都是虚拟的无法直接访问，必须要变成物理内存地址的才可以访问，该映射的过程称为页表映射


4、页表映射

针对用户进程而言，由于每个进程都有自己的用户态虚拟地址空间，因而每个进程都有自己的页表，用于将用户态的地址映射为物理内存地址。每个进程的页表的根，放在`task_struct`结构中（`struct mm_struct *mm`[成员](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/sched.h#L562)）。而内核有统一唯一的内核虚拟地址空间，因而内核也有个页表，用于将内核态的地址映射为物理内存地址。**内核的页表的根放在一个预设的地方，可以直接映射到物理地址。在处理器内部，有一个控制寄存器叫 `CR3`，存放着页目录的物理地址，故 `CR3` 又叫做页目录基址寄存器（Page Directory Base Register，PDBR）**。内核页表的根是内存初始化的时候预设在一个虚拟地址和物理地址

如果当前进程A运行在用户态，则从其对应的`task_struct`里面找到页表顶级目录，加载到`CR3`寄存器，则A程序里面访问的虚拟地址就通过CPU指向的页表转换成物理地址进行访问。然而内核里面访问`task_struct`使用的也是虚拟地址，因而进程进入内核态后，`CR3`要变成指向内核页表的顶级目录，内核程序访问内核数据结构的虚拟地址就是用CPU指向的内核页表转换成物理地址进行访问的

![]()

##    0x02    进程调度：一个白话解读

![rq-taskstruct]()

####    进程调度的数据结构

1、`task_struct`中调度关联的成员

```cpp
struct task_struct {
    // ....
    unsigned int			policy; // 调度策略
    int				on_rq;

	int				prio;
	int				static_prio;
	int				normal_prio;
	unsigned int			rt_priority;

	const struct sched_class	*sched_class;       //核心，调度策略的执行逻辑，实现了调取器类中要求的添加任务队列、删除任务队列、从队列中选择进程等方法
	struct sched_entity		se;
	struct sched_rt_entity		rt;
    // ....
}
```

-  `policy`：调度策略，策略分为实时调度策略`SCHED_FIFO`、`SCHED_RR`、`SCHED_DEADLINE`是实时进程的调度策略，优先级比较高；普通调度策略包含`SCHED_NORMAL`、`SCHED_BATCH`、`SCHED_IDLE`，优先级较低
-   `sched_class`：调度的时候执行的代码是放在`sched_class`里面的，对应三种全局的实现：`rt_sched_class`对应实时进程的调度策略，`fair_sched_class`对应普通进程的调度策略，`idle_sched_class`对应空闲进程的调度策略；这里着重提下普通进程的调度类CFS（Completely Fair Scheduling），即完全公平调度。`task_struct`里面的成员变量执行哪种`sched_class`，就会按照哪种方式被调度。虽然每个进程都有一个`task_struct`，但是所有的`task_struct`的`sched_class`变量指向的都是同一个`rt_sched_class`或`fair_sched_class`或`idle_sched_class`；这三类`sched_class`内核仅有一份，只包含调度逻辑，是中立的，不属于任何进程
-   `sched_entity`：完全公平算法调度实体
-   `sched_rt_entity`：实时调度实体

2、`struct rq`

内核为每一个CPU都创建一个队列`struct rq`来保存可以在这个CPU上运行的任务，`task_struct`就是用`sched_entity`这个成员变量将自己挂载到某个CPU的队列上的

```CPP
struct rq {
    struct rt_rq rt;    //实时进程队列rt_rq
    struct cfs_rq cfs;      //CFS运行队列cfs_rq（实际上，CFS 的队列是一棵红黑树）
    struct dl_rq dl;
}
```

3、 [`sched_entity`](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/sched.h#L359)：代表被CFS算法调度的实体

```CPP
struct sched_entity {
	/* For load-balancing: */
	struct load_weight		load;
	struct rb_node			run_node;
	struct list_head		group_node;
	unsigned int			on_rq;

	u64				exec_start;
	u64				sum_exec_runtime;
	u64				vruntime;               //核心，对于完全公平调度算法，需要记录下进程的运行时间。CPU会提供一个时钟，过一段时间就触发一个时钟中断。CFS会为每一个进程安排一个虚拟运行时间vruntime。如果一个进程在运行，随着时间的增长，也就是一个个tick的到来，进程的vruntime将不断增大。没有得到执行的进程vruntime不变，这个字段为CFS算法的调度提供依据
	u64				prev_sum_exec_runtime;

	u64				nr_migrations;

	struct sched_statistics		statistics;

#ifdef CONFIG_FAIR_GROUP_SCHED
	int				depth;
	struct sched_entity		*parent;
	/* rq on which this entity is (to be) queued: */
	struct cfs_rq			*cfs_rq;
	/* rq "owned" by this entity/group: */
	struct cfs_rq			*my_q;
#endif
};
```

CFS调度算法的运行队列需要能够对`vruntime`进行排序，找出最小的那个`vruntime`，这个能够排序的数据结构不但需要查询的时候，能够快速找到最小的，更新的时候也需要能够快速地调整排序，要知道`vruntime`可是经常在变的，变了再插入这个数据结构，就需要重新排序。所以内核选择了[红黑树](https://pandaychen.github.io/2024/10/05/A-LINUX-KERNEL-TRAVEL-0/)作为CFS调度算法的实现

这三个`sched_class`内核中只有一份，只包含调度逻辑，是中立的，不属于任何进程的，一旦进了某个`sched_class`的代码逻辑，就要脱离某个进程的视角，进入内核的视角了。小节下上面提到的这三个数据结构之间的关系，即CFS的运行队列：

![]()

####    进程创建：新进程如何被调度

1、当一个进程创建的时候，就会分配`task_struct`结构，就会根据程序员设置的调度策略设置`sched_class`成员，默认指向`fair_sched_class`。进程创建后的一件重要的事情就是调用`sched_class`的`enqueue_task`方法，将这个新进程放进某个CPU的队列上来，虽然不一定马上运行，但是说明可以在这个CPU上被调度上去运行了，在这里`sched_class`的代码已经被调用了，其实调度已经开始了。从CPU的视角来看，指令指针寄存器里面指向的还是`sched_class`里面的代码，也即内核的代码只是对内存中的`task_struct`及`sched_entity`进行操作而已，进程的代码并没有运行

2、在`sched_class`的`enqueue_task`方法将新进程放入CPU队列后，会调用一个方法`check_preempt_curr`，来试图去抢占当前的进程去运行。当前进程是谁？在Linux里面进程都是由父进程`fork`的，这里的当前进程是父进程，然而这个时候父进程并没有运行在CPU上，因为这个时候CPU里面运行的还是内核代码，这里的当前进程也即父进程是目前队列里面排名最靠前的进程，而这里的抢占的意思是子进程试图排在父进程的前面，但是也不是马上就放到CPU上运行，而是仅仅将父进程设置了一个标志位`TIF_NEED_RESCHED`，表示应该被调度走。什么时候真的被调度走呢？父进程创建子进程是调用`fork`系统调用进的内核，当创建完毕后，从`fork`系统调用返回用户态的时候，发现了这个标志位`TIF_NEED_RESCHED`，才主动让给子进程运行的

3、那如何让新创建的子进程在用户态运行起来呢？根据前文描述，要想让用户态程序真的运行，按照CPU的原理，应该`CR3`指向这个进程的页表，虚拟内存及对应的物理内存上有代码，有数据，然后指令指针寄存器指向程序的某行代码，这样新进程才可以运行。那么用户进程的这些信息是如何被设置到相应的寄存器，虚拟内存，物理内存的呢？这和`task_struct`的另一个成员变量`stack`指向的内核栈有关。在CPU里，SP（Stack Pointer）是栈顶指针寄存器，入栈操作Push和出栈操作Pop指令，会自动调整SP的值。另外有一个寄存器BP（Base Pointer），是栈基地址指针寄存器，指向当前栈帧的最底部

4、栈是一个从高地址到低地址，往下增长的结构，也就是上面是栈底，下面是栈顶，入栈和出栈的操作都是从下面的栈顶开始的。在用户态，如果A调用B，A的栈里面包含A函数的局部变量，然后是调用B的时候要传给它的参数，然后返回A的地址也应该入栈，这就形成了A的栈帧。接下来就是B的栈帧部分了，先保存的是A的栈帧的栈底的位置，也就是`EBP`。因为在B函数里面获取A传进来的参数，就是通过这个指针获取的，接下来保存的是B的局部变量等，当B返回的时候，返回值会保存在`EAX`寄存器中，从栈中弹出返回地址，将指令跳转回去，参数也从栈中弹出，然后继续执行A，以上的栈操作，都是在进程的内存空间里面进行的

![]()

5、如果某程序通过系统调用，从该进程的内存空间到内核中了，那么类似的入栈/出栈操作如何执行呢？以stack，也就是内核栈，就派上了用场。
在内核栈的最高地址端，存放的是一个结构`pt_regs`，此结构的作用是当系统调用从用户态切换到内核态的时候，首先就是将用户态运行过程中的CPU上下文也即各种寄存器保存在这个结构里。其中两个重要寄存器`SP`和`IP`，`SP`里面是用户态程序运行到的栈顶位置，`IP`里面是用户态程序运行到的某行代码，这样当从内核系统调用返回的时候，才能让进程在刚才的地方接着运行下去。所以在内核填充`pt_regs`结构，然后从系统调用返回用户态，是让用户态程序运行的一种方式

![stack-switch]()

6、回想一下，整个Linux系统的第一个用户态进程也是这样运行起来的，当Linux系统启动的时候，首先进行初始化的是内核代码，当内核初始化结束，会创建第一个用户态进程即`1`号进程（后续所有的用户进程都是这个`1`号进程的子孙）。创建的方式是在内核态运行`do_execve`（内核系统调用），对应的进程可能为`/sbin/init`，`/etc/init`，`/bin/init`，`/bin/sh`。这里详细说明下，在`do_execve`的实现中，会设置`struct pt_regs`，主要设置的是`ip`和`sp`，指向第一个进程的起始位置，这样调用`iret`就可以从系统调用中返回。这个时候会从`pt_regs`恢复寄存器，此时指令指针寄存器`IP`恢复了，指向用户态下一个要执行的语句。函数栈指针`SP`也被恢复了，指向用户态函数栈的栈顶。所以，下一条指令，就从用户态开始运行了

```BASH
[root@VM-X-X-centos X]# ps aux
root           1  0.0  0.1 214124  8332 ?        Ss    2022 332:52 /usr/lib/systemd/systemd --switched-root --system --deserialize 17
```

某个用户态进程调用`fork`创建新进程，`fork`是系统调用会调用到内核函数，在内核中子进程会复制父进程的几乎一切，包括`task_struct`，内存空间等。这里注意的是，`fork`作为一个系统调用，是将用户态的当前运行状态放在`pt_regs`里面了，`IP`和`SP`指向的就是`fork`这个函数，然后就进内核了。子进程创建完了，如果像前面讲过的`check_preempt_curr`里面成功抢占了父进程，则父进程会被标记`TIF_NEED_RESCHED`，则在`fork`返回的时候，会检查这个标记，会将CPU主动让给子进程，然后让子进程从内核返回用户态，因为子进程是完全复制的父进程，因而返回用户态的时候，仍然在`fork`的位置，当然父进程将来返回的时候，也会在`fork`的位置，只不过返回值不一样，等于`0`说明是子进程

####    进程调度

-   主动调度（voluntary）：也称自愿切换，比如A进程运行着，里面有一条指令`sleep`，或者等待某个I/O事件，就需要主动让出CPU，让其他的进程运行
-   抢占式调度（involuntary）：也称强制切换，比如A进程运行的时间太长了，会被其他进程抢占。还有一种情况是，有一个进程B原来等待某个I/O事件，等待到了被唤醒，发现比当前正在CPU上运行的进行优先级高，于是进行抢占

1、主动调度

假设A进程正在用户态运行，函数的调用过程假设为`A_main->A_Fun->read()`，这个调用过程自然是被保存在用户态的栈里面的，而`read()`是一个系统调用，会进入内核，进入内核的那个时刻，用户态的栈顶指针`SP`和指令指针`IP`都会被保存在`pt_regs`里面，放到A进程对应的`task_struct`的`stack`成员变量指向的内核栈里面。在内核中，假设继续会进行函数调用过程`do_read()->A_Kernel1->A_Kernel2`，这个调用过程自然是被保存在内核栈里面，这个栈也有一个栈顶指针`SP`，这个时候的`IP`指向的是`A_Kernel2`里面的内核代码，在内核函数`A_Kernel2`中，发现要读的东西还没就绪，就需要进行等待，那么`A_Kernel2`的内核代码中应该会包含如下代码片段：

```CPP
/* Nothing to read, let's sleep */
schedule(); 
```

进程调度第一定律：所有进程的调度最终是通过正在运行的进程调用`__schedule` 函数实现；调用了`schedule()`函数，这是主动调度的开始。注意这里虽然是`A_Kernel2`调用的`schedule()`函数，这个调用操作仍然保存在内核栈中，`SP`和`IP`也都会指向`schedule()`函数，但是代码逻辑已经进入内核调度逻辑了，要注意切换到中立的视角，而非进程A的视角。在`schedule()`内核函数里面会完成如下步骤：

1.  在当前的CPU上，取出任务队列`rq`
2.  很重要的有点：`task_struct *prev`指向这个CPU的任务队列上面正在运行的进程`curr`，其实是A。为啥A是`prev`？因为A一旦被切换下来，它就成了前任了。从这里可以看出，视角要换成中立的视角，已经不把A当做当前进程了
3.  内核会通过`fair_sched_class.pick_next_task`函数获取下一个任务，其实是从CFS调度算法的红黑树里面找出当前`vruntime`最小的，最应该运行的任务，`task_struct *next`指向这个继任任务，假设为进程B
4.  当选出的继任者（B）和前任（A）不同，则需要进行上下文切换，上下文切换是在`context_switch`内核函数里面实现

####    进程上下文切换
本小节切换到CPU视角来介绍下`context_switch`的大致逻辑，有这两行核心代码：

```CPP
  switch_to(prev, next, prev);
  return finish_task_switch(prev);
```

当内核运行到`switch_to()`函数的时候，`IP`指向这一行，`SP`指向的还是A进程的内核栈。而一切都在`switch_to`里面发生了变化。在`task_struct`里面，还有一个成员变量`struct thread_struct thread`，里面保存了进程切换时的寄存器的值。在`switch_to`函数实现里面，A进程的`SP`就存进A进程的`task_struct`的`thread`结构中，而B进程的`SP`就从其`task_struct`的`thread`结构中取出来，加载到CPU的`SP`栈顶寄存器里面；此外在`switch_to`里面，还会将这个CPU的`current_task`指向B进程的`task_struct`，至此进程上下文的切换就结束了

这里小结下切换的过程，对于A进程来讲，`SP`暂时停在了`switch_to`这一行，并被保存进了`task_struct.stack`结构，如果`SP`拿出来，就能通过`task_struct`里面的`stack`指向的内核栈，找到在内核的调用过程`do_read()->A_Kernel1->A_Kernel2->schedule`，再往前追溯，`task_struct`的`pt_regs`里面保存了A进程在用户态的`SP`和`IP`，通过这些可以还原A进程在用户态时的运行状态。也即将来A恢复运行的时候，是能够从当时的节点运行的

那么对于B进程而言，`current_task`指向了其`task_struct`，也就能找到B进程的内核栈，而`SP`被拿出来，指向了内核栈的栈顶。在B进程的内核栈`stack`里面，能够找到当年B被调度走的那个时刻的在内核中的调用过程（假设是`do_write()->B_Kernel1->B_Kernel2->schedule`），为什么最后一层是调用`schedule`呢？因为B当年被调度走的时候，也是是运行的`schedule`函数

这里了解了在进行切换中，`SP`实际上是随着切换进程不停的在修改的，而`IP`寄存器呢？当调用`switch_to`函数的时候，`IP`就随着函数的调用进入函数逻辑，当`switch_to`执行了进程切换逻辑，返回的时候，`IP`就指向了下一行代码`finish_task_switch`，那么这里的`finish_task_switch`是B进程的`finish_task_switch`，还是A进程的`finish_task_switch`呢？其实是B进程的`finish_task_switch`，因为先前B进程就是运行到`switch_to`被切走的，所以现在回来，运行的是B进程的`finish_task_switch`。其实从CPU的角度来看，这个时候还不区分A还是B的`finish_task_switch`，在CPU眼里，这就是内核的一行代码。但是代码再往下运行，就能区分出来了，因为`finish_task_switch`结束，要从`schedule`函数返回了，那应该是返回到`A_Kernel2`还是返回到`B_Kernel2`呢？根据函数调用栈的原理，栈顶指针指向哪行就返回哪行，别忘了前面`SP`已经切换成为B进程的了，已经指向B的内核栈了，所以返回的是`B_Kernel2`。虽然指令指针寄存器`IP`还是一行一行代码的执行下去，不用做特意的切换，但由于有进程调度第一定律，所有的调度都会走`schedule`函数，而从`schedule`函数这里一进一出，`IP`没变，但是`SP`变了，进程切换就完成了

接下来B进程还会返回`B_Kernel1`，返回`do_write`，进一步返回用户态，返回的时候，从B进程的`pt_regs`里面取出B进程先前进内核的时候在用户态的`IP`和`SP`，返回用户态，假设先前B进程在用户态的调用路径`B_main->B_Fun->write()`，接下来，B在用户态就能在`B_Fun`里面接着运行下去。这样一次主动切换终于完成

####    进程调度：抢占调度

1、抢占式调度最常见的是，一个进程执行时间太长了，是时候切换到另一个进程运行了。如何衡量一个进程的运行时间呢？在计算机里面有一个时钟，会过一段时间触发一次时钟中断，通知操作系统，时间又过去一个时钟周期，可以查看是否需要抢占的时间点。时钟中断处理函数会调用`scheduler_tick()`，在这里面会更新新当前（正在运行）进程的`vruntime`，然后调用`check_preempt_tick`，检查是否是时候被抢占了。当发现当前进程应该被抢占，不能直接把它踢下来，而是把它标记为应该被抢占`TIF_NEED_RESCHED`，要等待当前进程在某个时机可以运行`schedule`

2、另外一个抢占式调度的场景是当一个进程被唤醒时候。当一个进程在等待一个I/O的时候，会主动放弃CPU。但是当I/O到来的时候，进程往往会被唤醒（内核通过`wake_up_process`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/sched/core.c#L2138)唤醒此进程的`task_struct`），就会调用了`check_preempt_curr`检查是否应该抢占当前进程。如果应该发生抢占，也不是直接踢走当前进程，而是将当前进程标记为应该被抢占`TIF_NEED_RESCHED`，同样等待前进程在某个时机可以运行`schedule`

3、那什么时候是真正抢占的时机呢？对于用户态的进程来讲，从系统调用中返回的那个时刻，是一个被抢占的时机。在`exit_to_usermode_loop`里面，当发现当前进程被标记`TIF_NEED_RESCHED`的时候，当前进程会调用`schedule`让出CPU；而对内核态的执行中，被抢占的时机一般发生在`preempt_enable()`中，在内核态的执行中，有的操作是不能被中断的，所以在进行这些操作之前，总是先调用`preempt_disable()`关闭抢占，当再次打开的时候，就是一次内核态代码被抢占的机会。在`preempt_enable()`中，同样是当发现当前进程被标记为`TIF_NEED_RESCHED`的时候，当前进程会调用`schedule`让出CPU

4、此外，在内核态也会遇到中断的情况，当中断返回的时候，返回的仍然是内核态。这个时候也是一个执行抢占的时机，从中断返回内核的，调用的是`preempt_schedule_irq`，里面会在需要的时候调用`schedule`让出当前进程

##  0x03    进程调度：名词

####    调度类
在Linux中，将调度器公共的部分抽象，使用`struct sched_class`结构体描述一个具体的调度类。系统核心调度代码会通过`struct sched_class`结构体的成员调用具体调度类的核心算法

```CPP
struct sched_class {
	const struct sched_class *next; //next成员指向下一个调度类（比自己低一个优先级）。在Linux中，每一个调度类都是有明确的优先级关系，高优先级调度类管理的进程会优先获得cpu使用权

    //入队操作，向该调度器管理的runqueue中添加一个进程
	void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);

    // 出队操作，向该调度器管理的runqueue中删除一个进程
	void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);

    //当一个进程被唤醒或者创建的时候，需要检查当前进程是否可以抢占当前cpu上正在运行的进程，如果可以抢占需要标记TIF_NEED_RESCHED flag
	void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags);

    //从runqueue中选择一个最适合运行的task，依据什么挑选最适合运行的进程？
	struct task_struct * (*pick_next_task)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);
    /* ... */
}; 
```

每一个进程都对应一种调度策略（每一个进程在创建之后，总是要选择一种调度策略），每一种调度策略又对应一种调度类（每一个调度类可以对应多种调度策略），其中stop调度器和idle-task调度器，仅由内核使用

| 调度类 | 描述 | 调度策略 | 意义| 
| :-----:| :----: | :----: |:----: |
| `stop_sched_class`| stop 调度器| | 优先级最高的调度类，可以抢占其他所有进程，不能被其他进程抢占 |
| `dl_sched_class` | deadline调度器 | SCHED_DEADLINE | 使用红黑树，把进程按照绝对截止期限进行排序，选择最小进程进行调度运|
| `rt_sched_class` | 实时调度器 | SCHED_FIFO、SCHED_RR | 实时调度器，为每个优先级维护一个队列|
| `fair_sched_class` | 完全公平调度器 | SCHED_NORMAL、SCHED_BATCH、SCHED_IDLE | 完全公平调度器 |
| `idle_sched_class` | idle task |  | 空闲调度器，每个CPU都会有一个idle线程，当没有其他进程可以调度时，调度运行idle线程|


-   SCHED_DEADLINE：限期进程调度策略，使task选择Deadline调度器来调度运行
-   SCHED_RR：实时进程调度策略，时间片轮转，进程用完时间片后加入优先级对应运行队列的尾部，把CPU让给同优先级的其他进程
-   SCHED_FIFO：实时进程调度策略，先进先出调度没有时间片，没有更高优先级的情况下，只能等待主动让出CPU
-   SCHED_NORMAL：普通进程调度策略，使task选择CFS调度器来调度运行
-   SCHED_BATCH：普通进程调度策略，批量处理，使task选择CFS调度器来调度运行
-   SCHED_IDLE：普通进程调度策略，使task以最低优先级选择CFS调度器来调度运行

![sched_class]()

调度类的优先级如下，每一个调度类利用next成员构建单项链表

```TEXT
sched_class_highest----->stop_sched_class
                         .next---------->dl_sched_class
                                         .next---------->rt_sched_class
                                                         .next--------->fair_sched_class
                                                                        .next----------->idle_sched_class
                                                                                         .next = NULL
```

Linux调度核心在选择下一个合适的task运行的时候，会按照上面调度类优先级的顺序遍历各个调度类的`pick_next_task`函数。因此，`SCHED_FIFO`调度策略的实时进程永远比`SCHED_NORMAL`调度策略的普通进程优先运行

```CPP
//负责选择一个即将运行的进程
static inline struct task_struct *pick_next_task(struct rq *rq,
                                                 struct task_struct *prev, struct rq_flags *rf)
{
	const struct sched_class *class;
	struct task_struct *p;
 
	for_each_class(class) {          /* 按照优先级顺序便利所有的调度类，通过next指针遍历单链表 */
		p = class->pick_next_task(rq, prev, rf);
		if (p)
			return p;
	}
}
```

####    就绪队列（runqueue）
系统中每个CPU（逻辑核）都会有一个全局的就绪队列（cpu runqueue），结构体为`struct rq`，它是per-cpu类型，即每个cpu上都会有一个`struct rq`结构体，可以减少锁的开销。每一个调度类也有属于自己管理的就绪队列

![]()

-   `struct cfs_rq`：CFS调度类的就绪队列，管理就绪态的`struct sched_entity`调度实体，后续6通过`pick_next_task`接口从就绪队列中选择最适合运行的调度实体（虚拟时间最小的调度实体）
-   `struct rt_rq`：实时调度器就绪队列
-   `struct dl_rq`：Deadline调度器就绪队列

```CPP
struct rq {
         struct cfs_rq cfs;
	struct rt_rq rt;
	struct dl_rq dl;
};
 
struct rb_root_cached {
	struct rb_root rb_root;
	struct rb_node *rb_leftmost;
};
 
struct cfs_rq {
	struct load_weight load;    //load：就绪队列权重，就绪队列管理的所有调度实体权重之和
	unsigned int nr_running;    //nr_running：就绪队列上调度实体的个数
	u64 min_vruntime;           //min_vruntime：跟踪就绪队列上所有调度实体的最小虚拟时间
	struct rb_root_cached tasks_timeline;   //用于跟踪调度实体按虚拟时间大小排序的红黑树的信息（包含红黑树的根以及红黑树中最左边节点）
}; 
```

##  0x0 调度器类型
-   主调度器（主动让出）：本体是`__schedule()`函数，需要在内核代码中主动调用
-   周期性调度器（定时调度）：本体是`scheduler_tick()`函数，周期性调度器则由配合系统的tick时钟中断以每秒`HZ`次的周期性触发

与调度器类型息息相关的概念就是调度时机，即前文提到的两种调度切换方式：

-   context switch：任务由于等待某种资源，将state改为非`RUNNING`状态后，调用`schedule()`主动让出CPU
-   involuntary context switch：任务状态仍为`RUNNING`却失去CPU使用权，情况有任务时间片用完、有更高优先级的任务、任务中调用`cond_resched()`或`yield`让出CPU；这里包含的一种场景就与周期性调度器有关系（CFS）

调度时机包括：定时调度schedule_tick和其它进程阻塞时主动让出两种

####    主调度器：schedule
主动调度就是进程运行到一半，因为等待 I/O 等操作而主动调用 `schedule()` 函数让出 CPU（golang协程的读写过程很类似），这里列举几个例子：

1、写入块设备，写入需要一段时间，这段时间用不上CPU

```CPP
static void btrfs_wait_for_no_snapshoting_writes(struct btrfs_root *root){
    //......
    do {
        prepare_to_wait(&root->subv_writers->wait, &wait,
                TASK_UNINTERRUPTIBLE);
        writers = percpu_counter_sum(&root->subv_writers->counter);
        if (writers)
            schedule();
        finish_wait(&root->subv_writers->wait, &wait);
    } while (writers);
}
```

2、从 Tap 网络设备等待一个读取

```CPP
static ssize_t tap_do_read(struct tap_queue *q,
            struct iov_iter *to,
            int noblock, struct sk_buff *skb){
    //......
    while (1) {
        if (!noblock)
            prepare_to_wait(sk_sleep(&q->sk), &wait,
                    TASK_INTERRUPTIBLE);
    //......
        /* Nothing to read, let's sleep */
        schedule();
    }
    //......
}
```

类似的case 在 Linux 内核中非常多，它们会把进程设置为 `D` 状态（`TASK_UNINTERRUPTIBLE`），主要集中在 disk I/O 的访问和信号量（Semaphore）锁的访问上

####    周期性调度器
系统中每个CPU都会有一个系统定时器，本质上是一个可编程中断时钟。通过配置可以让其每秒生成固定`HZ`个中断。时钟每过一段时间触发一次时钟中断，时钟中断处理函数会调用 `scheduler_tick()`函数，即周期性调度的入口是`scheduler_tick`->`curr->sched_class->task_tick`，时钟节拍最终会调用调度类`task_tick`方法完成调度相关的工作，会在这里判断是否需要调度下一个任务来抢占当前CPU核。也会触发多核之间任务队列的负载均衡，保证不让忙的核忙死，闲的核闲死。在调度节拍中会定时将每个进程所执行过的时间都换算成vruntime，并累计起来，也会定时判断当前进程是否已经执行了足够长的时间，如果是的话，需要再选择另一个vruntime较小的任务来运行

周期性调度通常会引发抢占式调度，所谓的抢占调度，就是A进程运行的时间太长了，会被其他进程抢占。还有一种情况是，有一个进程B原来等待某个I/O事件，等待到了被唤醒，发现比当前正在CPU上运行的进行优先级高，于是进行抢占

由于tick中断发生时，中断处理程序中`scheduler_tick`会根据current进程所属的调度类（`curr->sched_class`）调用不同的`task_tick`方法实现，这里以CFS实现的fair调度类（`task_tick_fair`）简单说明

1. 更新rq（CPU运行队列）的clock和clock_task：关联函数`update_rq_clock()`
2. 更新cfs_rq的min_vruntime以及正在运行task（实际是`struct sched_entity`）的vruntime：关联函数`update_curr()`
3. 更新sched_entity和cfs_rq的平均负载统计：关联函数`update_load_avg()`
4. 检查当前是否需要重新调度并设置`TIF_NEED_RESCHED`：关联函数`check_preempt_tick()`
5. 因为步骤`3`中已经更新了负载，最后一步判断是否进行负载均衡：关联函数`trigger_load_balance()`

再次说明：周期性调度器的`scheduler_tick`部分并不会去主动调度，而是为当前进程设置`TIF_NEED_RESCHED`标志位，tick时钟中断返回时才会根据标志位的状态来调度

1、周期性调度入口：`scheduler_tick`

```CPP
void scheduler_tick(void)
{
        // 取出当前的cpu及其任务队列
        //....
        rq_lock(rq, &rf);   //由于要操作rq，所以提前获取rq的spinlock锁
        update_rq_clock(rq);    //读取cpu clock来更新rq的clock和clock_task
    
        thermal_pressure = arch_scale_thermal_pressure(cpu_of(rq));
        update_thermal_load_avg(rq_clock_thermal(rq), rq, thermal_pressure);
        struct task_struct *curr = rq->curr;
        // ....
        //cfs的方法对应task_tick_fair
        curr->sched_class->task_tick(rq, curr, 0);
        cpu_load_update_active(rq);
        calc_global_load_tick(rq);
        psi_task_tick(rq);

        rq_unlock(rq, &rf);    //释放rq的spinlock锁
        perf_event_task_tick();

#ifdef CONFIG_SMP
        rq->idle_balance = idle_cpu(cpu);  //判断是否需要触发负载均衡，是的话则raise sched_softirq
        trigger_load_balance(rq);
#endif
}
```

2、`task_tick_fair`的实现

```CPP
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{       
        /*若是进程数据任务组的话，则逐层为每一个父任务组的cfs_rq调用entity_tick*/
        for_each_sched_entity(se) {
            cfs_rq = cfs_rq_of(se);
            entity_tick(cfs_rq, se, queued);          //核心实现
        }
        //......
}
```

3、`entity_tick`的实现：只有CFS算法的`task_tick`方法才会调用`entity_tick`函数

```CPP
static void
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
    /*
     * Update run-time statistics of the 'current'.
     */
    update_curr(cfs_rq);   //涉及cfs，计算任务运行时间delta，将delta转化为虚拟时间更新到进程的vruntime，并更新rq上的min_vruntime

    /*
     * Ensure that runnable average is periodically updated.
     */
    update_load_avg(curr, 1);   //更新负载，包括任务entity和cfs_rq中的平均负载
    update_cfs_shares(cfs_rq);  //设计组调度部分，包括更新组权重、带宽控制等

    // ......

    if (cfs_rq->nr_running > 1)
        check_preempt_tick(cfs_rq, curr);    //核心：检查vruntime是否需要抢占，并设置TIF_NEED_RESCHED
}
```

4、`update_curr`函数（CFS的核心函数）：`task_struct.sched_entity.vruntime`（进程的虚拟运行时间），如果一个进程在运行，随着时间的增长（一个个 tick 的到来）进程的 `vruntime` 将不断增大。没有得到执行的进程 `vruntime` 不变，最后尽量保证所有进程的vruntime相等

```CPP
/*
 * Update the current task's runtime statistics.
 */
static void update_curr(struct cfs_rq *cfs_rq){
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;
    //......
    delta_exec = now - curr->exec_start;
    //......
    curr->exec_start = now;
    //......
    curr->sum_exec_runtime += delta_exec;
    //......

    //delta_exec：实际运行时间
    curr->vruntime += calc_delta_fair(delta_exec, curr);
    update_min_vruntime(cfs_rq);
    //......
}

/*
NICE_0_LOAD宏对应的是1024，如果权重是1024，那么vruntime 就正好等于实际运行时间，否则会进入__calc_delta 来根据权重和实际运行时间折算一个vruntime增量。如果weight 较高，则同样实际运行时间算出来的vruntime 就会偏小，就会在调度中获取更多的cpu，cfs 就是这样实现了cpu资源按权重分配
*/
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se){
    if (unlikely(se->load.weight != NICE_0_LOAD))
        /* delta_exec * weight / lw.weight */
        //虚拟运行时间 vruntime += 实际运行时间 delta_exec * NICE_0_LOAD/ 权重
        delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
    return delta;
}
```

5、`check_preempt_tick`详解（CFS的核心函数）：它的主要工作内容则是检查此时是否应该发生重新调度，然后`resched_curr()`去设置`TIF_NEED_RESCHED`标志位

```CPP
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
        unsigned long ideal_runtime, delta_exec;
        struct sched_entity *se;
        s64 delta;

        //计算出当前进程此次运行理想的时间片应是多少（当前进程在本次调度中分配的运行时间）
        //TODO：sched_slice在CFS算法的分析文章中解析
        ideal_runtime = sched_slice(cfs_rq, curr);
        //算出进程从上次开始运行到现在一共跑了多长时间（当前进程已经运行的实际时间）
        delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
        //若运行时间已经超过了理应分到的时间片，则说明该重新调度了
        if (delta_exec > ideal_runtime) {
                //为curr进程设置need_resched
                //如果实际运行时间已经超过分配给进程的运行时间，就需要抢占当前进程，在resched_curr函数中设置进程的TIF_NEED_RESCHED抢占标志
                resched_curr(rq_of(cfs_rq));
                clear_buddies(cfs_rq, curr);
                return;
        }

        //若运行时间小于最小调度运行时间粒度，则不需要重新调度
        //说明：由于cfs有一个调度周期sysctl_sched_latency的概念，即在一个固定的调度周期内cfs_rq上的所有进程都要被运行一遍，所以同时会存在一个最小调度时间粒度sysctl_sched_min_granularity的概念，若当前进程运行时间小于这个粒度则不会重新调度
        if (delta_exec < sysctl_sched_min_granularity)
                return;

        //从红黑树中找到虚拟时间最小的调度实体（即最左孩子节点）
        se = __pick_first_entity(cfs_rq);
        delta = curr->vruntime - se->vruntime;
        //若当前进程的vruntime小于cfs_rq上（红黑树）的最左边的vruntime（红黑树中最左边调度实体虚拟时间）则不需要重新调度
        if (delta < 0)
                return;

        if (delta > ideal_runtime)
                resched_curr(rq_of(cfs_rq));
}
```

小结下周期性调度的流程，对于普通进程 `scheduler_tick` ==> `fair_sched_class.task_tick_fair` ==> `entity_tick` ==> `update_curr` 更新当前进程的 `vruntime` ==> `check_preempt_tick` 检查是否是时候被抢占；再强调一点，当发现当前进程应该被抢占，不能直接把它踢下来，而是把它标记为应该被抢占（因为根据进程调度第一定律，一定要等待正在运行的进程调用 `__schedule` 主动让出CPU才行）

##  0x04 进度调度延迟（可观测）
Linux内核为观测CPU运行队列运行指标（主要是调度延迟）提供了三个经典的tracepoint hook，代码基于[5.4.119](https://elixir.bootlin.com/linux/v5.4.119/source/kernel/sched/core.c#L4085)版本

从上文了解到，调度分为两类 Voluntary Switch和 Involuntary Switch，而且调度时并不是立即切换，所以必然存在一定的调度延迟。所谓调度延迟，是指一个任务具备运行的条件（新创建进入 CPU 的 runqueue OR 抢占调度准备完成），到真正执行（获得 CPU 的执行权）的这段时间。那为什么会有调度延迟呢？因为 CPU 还被其他任务占据，还没有空出来，而且可能还有其他在 runqueue 中排队的任务；排队的任务越多，调度延迟就可能越长，所以这也是间接衡量 CPU 负载的一个指标（CPU 负载通过计算各个时刻 runqueue 上的任务数量获得）

```CPP
wake_up_process() --> ttwu_do_wakeup() --> trace_sched_wakeup   //抢占式调度
_do_fork() -->wake_up_new_task() --> trace_sched_wakeup_new   //新进程创建调度
__schedule() --> trace_sched_switch             // 
```

####    计算延迟
1.  当前正在CPU运行的任务`task_struct`因等待某种事件进入休眠态（Voluntary Switch），那么就是从被唤醒（`wakeup`/`wakeup_new` 的时间点），到获得 CPU （任务切换时的 `next_pid`）的间隔
2.  任务因 Involuntary Switch 让出 CPU（任务切换时作为 `prev_pid`），到再次获得 CPU （之后的某次任务切换时作为`next_pid`）所经历的时间。在这期间，任务始终在 runqueue 上，始终是 runnable 的状态，所以有 `prev_state` 是否为 `TASK_RUNNING` 的判断

####    
```CPP
static void __sched notrace __schedule(bool preempt)
{
	struct task_struct *prev, *next;
	unsigned long *switch_count;
	struct rq_flags rf;
	struct rq *rq;
	int cpu;

	cpu = smp_processor_id();
	rq = cpu_rq(cpu);
	prev = rq->curr;

	schedule_debug(prev, preempt);

	if (sched_feat(HRTICK))
		hrtick_clear(rq);

	local_irq_disable();
	rcu_note_context_switch(preempt);

	/*
	 * Make sure that signal_pending_state()->signal_pending() below
	 * can't be reordered with __set_current_state(TASK_INTERRUPTIBLE)
	 * done by the caller to avoid the race with signal_wake_up().
	 *
	 * The membarrier system call requires a full memory barrier
	 * after coming from user-space, before storing to rq->curr.
	 */
	rq_lock(rq, &rf);
	smp_mb__after_spinlock();

	/* Promote REQ to ACT */
	rq->clock_update_flags <<= 1;
	update_rq_clock(rq);

	switch_count = &prev->nivcsw;
	if (!preempt && prev->state) {
		if (signal_pending_state(prev->state, prev)) {
			prev->state = TASK_RUNNING;
		} else {
			deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);

			if (prev->in_iowait) {
				atomic_inc(&rq->nr_iowait);
				delayacct_blkio_start();
			}
		}
		switch_count = &prev->nvcsw;
	}

	next = pick_next_task(rq, prev, &rf);
	clear_tsk_need_resched(prev);
	clear_preempt_need_resched();

	if (likely(prev != next)) {
		rq->nr_switches++;
		/*
		 * RCU users of rcu_dereference(rq->curr) may not see
		 * changes to task_struct made by pick_next_task().
		 */
		RCU_INIT_POINTER(rq->curr, next);
		/*
		 * The membarrier system call requires each architecture
		 * to have a full memory barrier after updating
		 * rq->curr, before returning to user-space.
		 *
		 * Here are the schemes providing that barrier on the
		 * various architectures:
		 * - mm ? switch_mm() : mmdrop() for x86, s390, sparc, PowerPC.
		 *   switch_mm() rely on membarrier_arch_switch_mm() on PowerPC.
		 * - finish_lock_switch() for weakly-ordered
		 *   architectures where spin_unlock is a full barrier,
		 * - switch_to() for arm64 (weakly-ordered, spin_unlock
		 *   is a RELEASE barrier),
		 */
		++*switch_count;

		trace_sched_switch(preempt, prev, next);

		/* Also unlocks the rq: */
		rq = context_switch(rq, prev, next, &rf);
	} else {
		rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);
		rq_unlock_irq(rq, &rf);
	}

	balance_callback(rq);
}
```

####     wake_up_process
[`wake_up_process`](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/sched/core.c#L1947)

```CPP

```

##  0x0 主调度器


##  0x0  参考
-   [一文搞懂linux cfs调度器](https://zhuanlan.zhihu.com/p/556295381)
-   [Linux进程调度：主调度器](https://zhuanlan.zhihu.com/p/426395078)
-   [【原创】（一）Linux进程调度器-基础](https://www.cnblogs.com/LoyenWang/p/12249106.html)
-   [万字详解Linux内核调度器极其妙用](https://mp.weixin.qq.com/s/gkZ0kve8wOrV5a8Q2YeYPQ?nwr_flag=1#wechat_redirect)
-   [Linux 的调度延迟 - 原理与观测](https://zhuanlan.zhihu.com/p/462728452)
-   [Linux进程调度：调度时机](https://zhuanlan.zhihu.com/p/163728119)
-   [Linux进程调度：周期性调度器](https://zhuanlan.zhihu.com/p/426448579)
-   [CFS调度器（1）-基本原理](http://www.wowotech.net/process_management/447.html)