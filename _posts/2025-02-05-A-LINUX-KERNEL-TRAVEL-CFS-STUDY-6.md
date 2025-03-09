---
layout:     post
title:  Linux 内核之旅（六）：进程调度（CFS）
subtitle:   
date:       2025-01-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
本文学习下CFS调度算法（Completely Fair Scheduler，完全公平调度器）用于Linux系统中普通进程的调度，CFS调度器的目标是让所有普通进程的`vruntime`尽可能接近，实现公平的调度。CFS的设计理念是在真实硬件上实现理想的、精确的多任务CPU。CFS调度器和以往的调度器不同之处在于没有时间片的概念，而是分配cpu使用时间的比例，若`2`个相同优先级的进程在一个CPU上运行，那么每个进程都将会分配`50%`的CPU运行时间

-	调度实体：对应结构`sched_entity`
-	红黑树：CFS采用了红黑树算法来管理所有的调度实体，算法效率`O(log(n))`
-	vruntime：CFS跟踪调度实体sched_entity的虚拟运行时间vruntime，平等对待运行队列中的调度实体sched_entity，将执行时间少的调度实体sched_entity排列到红黑树的左边

##  0x01    CFS数据结构及关系
-	`task_struct`：每一个调度类并不是直接管理`task_struct`，而是关联调度实体
-	`sched_entity`：调度实体
-	`cfs_rq`	

根据上文的介绍，了解到CFS调度器使用`sched_entity`跟踪调度信息。CFS调度器使用`cfs_rq`跟踪就绪队列信息以及管理就绪态调度实体，并维护一棵按照虚拟时间排序的红黑树，`tasks_timeline->rb_root`是红黑树的根节点，`tasks_timeline->rb_leftmost`指向红黑树中最左边的调度实体节点（虚拟时间最小的调度实体），为了更快的选择最适合运行的调度实体，`rb_leftmost`相当于一个缓存。每个就绪态的调度实体`sched_entity`包含插入红黑树中使用的节点`rb_node`，同时`vruntime`成员记录已经运行的`task_struct`虚拟时间。CFS算法会选择红黑树最左边的进程运行，随着系统时间的推移，原来左边运行过的进程慢慢的会移动到红黑树的右边，原来右边的进程也会最终跑到最左边。它们之间的关系如下图：

![cfs_relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/scheduler/cfs_process_schedule_impl.jpeg)

####	调度延迟sched_period
调度延迟是保证每一个可运行进程都至少运行（完成）一次的时间间隔。例如每个进程都运行`10ms`，系统中总共有`2`个进程，那么调度延迟就是`20ms`。如果现在保证调度延迟不变，固定是`6ms`，如果有`100`个进程，那么每个进程分配到的时间就是`0.06ms`。随着进程的增加，每个进程分配的时间在减少，进程调度过于频繁，上下文切换时间开销就会变大。因此，CFS调度器的调度延迟时间的设定并不是固定的。当系统处于就绪态的进程少于一个定值（默认值`8`）的时候，调度延迟也是固定一个值不变（默认值`6ms`）。当系统就绪态进程个数超过这个值时，需要保证每个进程至少运行一定的时间才让出CPU，至少一定的时间被称为最小粒度时间（在CFS默认设置中，最小粒度时间是`0.75ms`，关联变量`sysctl_sched_min_granularity`）

```CPP
//调度周期是一个动态变化的值，使用__sched_period计算调度周期
//nr_running是系统中就绪进程数量，当超过sched_nr_latency时，无法保证调度延迟，因此转为保证调度最小粒度。如果nr_running并没有超过sched_nr_latency，那么调度周期就等于调度延迟sysctl_sched_latency（6ms）
static u64 __sched_period(unsigned long nr_running)
{
	if (unlikely(nr_running > sched_nr_latency))
		return nr_running * sysctl_sched_min_granularity;
	else
		return sysctl_sched_latency;
}
```

####	普通进程的优先级
现实情况下，CFS调度器针对普通进程的优先级是通过权重实现的，权重代表着进程的优先级，即各个进程之间按照权重的比例分配CPU时间，权重越大分配的时间比例越大，相当于优先级越高。在引入权重之后，分配给进程的时间计算公式如下：

```TEXT
分配给进程的时间 = 总的cpu时间 * 进程的权重/就绪队列（runqueue）所有进程权重之和
```

此外，CFS调度器中设计了`nice`值，与权重一一对应。`nice`取值范围是`[-20, 19]`（数值越小代表优先级越大，同时也意味着权重值越大），转换方式如下：

```CPP
//数组的值可以看作是公式：weight = 1024 / 1.25nice计算得到。公式中的1.25取值依据是：进程每降低一个nice值，将多获得10% cpu的时间。公式中以1024权重为基准值计算得来，1024权重对应nice值为0，其权重被称为NICE_0_LOAD。默认情况下，大部分进程的权重基本都是NICE_0_LOAD
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
}; 
```

####	虚拟运行时间vruntime VS runtime

![vruntime]()

1、虚拟运行时间`vruntime`

例如调度周期是`6ms`，系统一共2个相同优先级的进程A和B，那么每个进程都将在6ms周期时间内内各运行3ms。如果进程A和B，他们的权重分别是1024和820（nice值分别是0和1）。进程A获得的运行时间是6x1024/(1024+820)=3.3ms，进程B获得的执行时间是6x820/(1024+820)=2.7ms。进程A的cpu使用比例是3.3/6x100%=55%，进程B的cpu使用比例是2.7/6x100%=45%。计算结果也符合上面说的“进程每降低一个nice值，将多获得10% CPU的时间”。很明显，2个进程的实际执行时间是不相等的，但是CFS想保证每个进程运行时间相等。因此CFS引入了虚拟时间的概念，也就是说上面的2.7ms和3.3ms经过一个公式的转换可以得到一样的值，这个转换后的值称作虚拟时间。这样的话，CFS只需要保证每个进程运行的虚拟时间是相等的即可。虚拟时间vriture_runtime和实际时间（wall time）转换公式如下：


```TEXT
                                 NICE_0_LOAD
vriture_runtime = wall_time * ----------------
                                    weight 
```

-	`nice`值为`0`的进程的虚拟时间和实际时间相同

进程A的虚拟时间3.3 * 1024 / 1024 = 3.3ms，我们可以看出nice值为0的进程的虚拟时间和实际时间是相等的。进程B的虚拟时间是2.7 * 1024 / 820 = 3.3ms。我们可以看出尽管A和B进程的权重值不一样，但是计算得到的虚拟时间是一样的。因此CFS主要保证每一个进程获得执行的虚拟时间一致即可。在选择下一个即将运行的进程的时候，只需要找到虚拟时间最小的进程即可。为了避免浮点数运算，因此我们采用先放大再缩小的方法以保证计算精度。内核又对公式做了如下转换

```TEXT
                                 NICE_0_LOAD
vriture_runtime = wall_time * ----------------
                                    weight
  
                                   NICE_0_LOAD * 2^32
                = (wall_time * -------------------------) >> 32
                                        weight
                                                                                        2^32
                = (wall_time * NICE_0_LOAD * inv_weight) >> 32        (inv_weight = ------------ )
                                                                                        weight 
```

2、`weight` && `inv_weight`

其中，`inv_weight`的值可根据`weight`计算（而权重`weight`的值已经计算保存到`sched_prio_to_weight`数组中，计算公式`sched_prio_to_wmult[i] = 232/sched_prio_to_weight[i]`），在使用时只需要查表`sched_prio_to_wmult`就可以的到`inv_weigth`的值

```TEXT
                  2^32
(inv_weight = ------------ )
                  weight 
```

```CPP
/*
 * Inverse (2^32/x) values of the sched_prio_to_weight[] array, precalculated.
 *
 * In cases where the weight does not change often, we can use the
 * precalculated inverse to speed up arithmetics by turning divisions
 * into multiplications:
 */
const u32 sched_prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
};
```

内核中使用`struct load_weight`结构描述进程的权重信息（包含在调度实体`sched_entity`中）

```CPP
struct load_weight {
	unsigned long		weight;		//进程的权重
	u32			inv_weight;			//inv_weight等于232/weight
};
```

3、转换：`runtime`-->`vruntime`：`calc_delta_fair`函数实现此转换，核心是调用`__calc_delta`函数

```CPP
//将实际时间转换成虚拟时间
//calc_delta_fair()函数调用__calc_delta()的时候传递的weight参数是NICE_0_LOAD，lw参数是进程对应的struct load_weight结构体
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
	/*
	nice值为0（权重是NICE_0_LOAD）的进程的虚拟时间和实际时间是相等的。因此如果进程的权重是NICE_0_LOAD，进程对应的虚拟时间就不用计算
	*/
	if (unlikely(se->load.weight != NICE_0_LOAD))
		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);	//

	return delta;
}

static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
	u64 fact = scale_load_down(weight);
	int shift = 32;
 
	__update_inv_weight(lw);
 
	if (unlikely(fact >> 32)) {
		while (fact >> 32) {
			fact >>= 1;
			shift--;
		}
	}
 
	fact = (u64)(u32)fact * lw->inv_weight;
 
	while (fact >> 32) {
		fact >>= 1;
		shift--;
	}
 
	return mul_u64_u32_shr(delta_exec, fact, shift);
} 
```

`__calc_delta`函数的抽象公式如下：

```TEXT
__calc_delta() = (delta_exec * weight * lw->inv_weight) >> 32
 
                                  weight                                 2^32
               = delta_exec * ----------------    (lw->inv_weight = --------------- )
                                lw->weight                             lw->weight 
```

4、调度实体中的时间相关字段

由于`struct sched_entity`结构体描述调度实体，包括`struct load_weight`用来记录权重信息，核心字段如下：

```CPP
struct sched_entity {
	struct load_weight		load;	//load：权重信息，在计算虚拟时间的时候会用到inv_weight成员
	struct rb_node		run_node;	//run_node：CFS调度器的每个就绪队列维护了一颗红黑树，上面挂满了就绪等待执行的task，run_node就是挂载点
	unsigned int		on_rq;		//on_rq：调度实体se加入就绪队列后，on_rq置1。从就绪队列删除后，on_rq置0
	u64			sum_exec_runtime;	//sum_exec_runtime：调度实体已经运行实际时间总和
	u64			vruntime;			//vruntime：调度实体已经运行的虚拟时间总和
}; 
```

5、CFS 的公平性本质：加权公平

从上面的描述了解到，CFS 的公平性目标是让所有进程的 **vruntime 趋近一致**，但这并非要求所有进程的实际运行时间（runtime）相同，优先级（权重）决定了进程的权重比例，本质是**高优先级进程的 `vruntime` 增长更慢，从而在相同时间内积累更多实际运行时间**，所以这一机制实现了**权重越高，实际运行时间越长**的公平性，而非绝对平等

如下例子，假设进程 A（权重 `2048`，优先级高）和进程 B（权重 `1024`，优先级低）同时运行，若两者均运行 `1ms`：
-	进程A 的 `vruntime` 增量：`1ms * (1024/2048) = 0.5ms`
-	进程B 的 `vruntime` 增量：`1ms * (1024/1024) = 1ms`

所以为了让 进程A 和 进程B 的 `vruntime` 趋近一致，A 需要运行 `2ms`，B 运行 `1ms`，如此结果为进程A 的实际运行时间是 进程B 的 `2` 倍，但两者的 `vruntime` 增量相同，符合公平性

-	进程A 的 `vruntime` 增量：`2ms * (1024/2048) = 1ms`
-	进程B 的 `vruntime` 增量：`1ms * (1024/1024) = 1ms`

6、runtime与vruntime的关系

这里介绍下实际运行时间的动态分配机制，由于CFS 不依赖固定时间片，而是根据进程权重和可运行进程数态分配时间片：

-	调度周期（`sched_period`）：所有可运行进程至少运行一次的周期，默认约 `6ms ~ 48ms`（取决于进程数），而进程时间片的计算公式为**时间片 = (调度周期 * 进程权重) / 总权重**，即权重越高的进程，时间片越长；总权重是所有可运行进程权重之和
-	调度决策：CFS 维护红黑树，按 `vruntime` 排序，优先调度 `vruntime` 最小的进程；而高优先级进程因 `vruntime` 增长更慢，会被更频繁调度，从而获得更多实际运行时间
-  长期收敛性：在多个调度周期后，高优先级进程的 **vruntime 增速慢**，实际运行时间积累更快；而低优先级进程的 **vruntime 增速快**，实际运行时间积累更慢

这样CFS算法的最终调度效果就是：所有进程的 `vruntime` 趋近一致，但实际运行时间按权重比例分配

####	min_vruntime
CFS代码中出现的`min_vruntime`（注意到其是归属于`cfs_rq`结构），其作用是作为`vruntime` 的标尺，用来动态记录每个CPU就绪队列上 `vruntime` 的最小值

```CPP
/* kernel/sched/sched.h */

struct cfs_rq {
    u64         min_vruntime;
    struct rb_root_cached   tasks_timeline;
    /* ... */
};
```

`min_vruntime`的意义如下：
-	

####	几个重要函数

1、`calc_delta_fair`：用来计算进程的`vruntime`的函数

2、`sched_slice`：此函数是用来计算一个调度周期内，一个调度实体可以分配多少运行时间

```CPP
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	//__sched_period函数是计算调度周期的函数，此函数当进程个数小于8时，调度周期等于调度延迟等于6ms。否则调度周期等于进程的个数乘以0.75ms，表示一个进程最少可以运行0.75ms，防止进程过快发生上下文切换
	u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);

	//遍历当前的调度实体，如果调度实体没有调度组的关系，则只运行一次
	//获取当前CFS运行队列cfs_rq，获取运行队列的权重cfs_rq->rq代表的是这个运行队列的权重。最后通过__calc_delta计算出此进程的实际运行时间
	for_each_sched_entity(se) {
		struct load_weight *load;
		struct load_weight lw;

		cfs_rq = cfs_rq_of(se);
		load = &cfs_rq->load;

		if (unlikely(!se->on_rq)) {
			lw = cfs_rq->load;

			update_load_add(&lw, se->load.weight);
			load = &lw;
		}

		//__calc_delta函数：不仅仅可以计算一个进程的虚拟时间，它在这里是计算一个进程在总的调度周期中可以获取的运行时间，公式为：进程的运行时间 = （调度周期时间 * 进程的weight） / CFS运行队列的总weigth
		slice = __calc_delta(slice, se->load.weight, load);
	}
	return slice;
}
```

3、`place_entity`：此函数用来惩罚一个调度实体，本质是修改其`vruntime`的值。根据传入参数`initiat`分为两种情况

-	`initial==0/*false*/`：如果`inital`为`false`，则代表的是唤醒的进程，对于唤醒的进程则需要照顾，最大的照顾是调度延时的一半，确保调度实体的`vruntime`不得倒退
-	`initial==1/*true*/`：当参数`initial`为`true`时，代表是新创建的进程，新创建的进程则给它的`vruntime`增加值，代表惩罚它。因为新创建进程的`vruntime`过小，防止其一直占在CPU

```CPP
static void
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
	//获取当前CFS运行队列的min_vruntime的值
	u64 vruntime = cfs_rq->min_vruntime;

	/*
	 * The 'current' period is already promised to the current tasks,
	 * however the extra weight of the new task will slow them down a
	 * little, place the new task so that it fits in the slot that
	 * stays open at the end.
	 */
	if (initial && sched_feat(START_DEBIT))
		vruntime += sched_vslice(cfs_rq, se);

	/* sleeps up to a single latency don't count. */
	if (!initial) {
		unsigned long thresh = sysctl_sched_latency;

		/*
		 * Halve their sleep time's effect, to allow
		 * for a gentler effect of sleepers:
		 */
		if (sched_feat(GENTLE_FAIR_SLEEPERS))
			thresh >>= 1;

		vruntime -= thresh;
	}

	/* ensure we never gain time by being placed backwards. */
	//通过`max_vruntime`获取最大的`vruntime`
	se->vruntime = max_vruntime(se->vruntime, vruntime);
}
```

4、`update_curr`：update_curr函数用来更新当前进程的运行时间信息

```CPP
static void update_curr(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq->curr;
	u64 now = rq_clock_task(rq_of(cfs_rq));
	u64 delta_exec;

	if (unlikely(!curr))
		return;

	//计算出当前CFS运行队列的进程，距离上次更新虚拟时间的差值
	delta_exec = now - curr->exec_start;
	if (unlikely((s64)delta_exec <= 0))
		return;

	//更新exec_start的值
	curr->exec_start = now;

	schedstat_set(curr->statistics.exec_max,
		      max(delta_exec, curr->statistics.exec_max));

	//更新当前进程总共执行的时间
	curr->sum_exec_runtime += delta_exec;
	schedstat_add(cfs_rq->exec_clock, delta_exec);

	//通过calc_delta_fair计算当前进程虚拟时间
	curr->vruntime += calc_delta_fair(delta_exec, curr);
	//通过update_min_vruntime函数来更新CFS运行队列中最小的vruntime的值
	update_min_vruntime(cfs_rq);


	account_cfs_rq_runtime(cfs_rq, delta_exec);
}
```

####	vruntime的汇总
-	`vruntime` 本质上是一个累加值，但其累加规则并非简单的物理时间叠加，而是基于进程的 ​权重（优先级）​​动态调整

##	0x02	CFS的运行原理

####    进程创建
进程的创建是通过`do_fork()`完成，调用链如下：`do_fork()`----> `_do_fork()` ----> `copy_process()`----> `sched_fork()`；当fork 进程时，核心`copy_process`以复制父进程的方式来生成一个新的`task_struct`，然后调用`wake_up_new_task` 函数将新进程添加到CPU的就绪队列中，等待调度器调度执行

```CPP
long _do_fork(unsigned long clone_flags,unsigned long stack_start,unsigned long stack_size,int __user *parent_tidptr,int __user *child_tidptr,unsigned long tls){
    struct task_struct *p;
    int trace = 0;
    long nr;
    //......
    // 复制结构
    p = copy_process(clone_flags, stack_start, stack_size,child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
    //......
    if (!IS_ERR(p)) {
        struct pid *pid;
        pid = get_task_pid(p, PIDTYPE_PID);
        nr = pid_vnr(pid);
        if (clone_flags & CLONE_PARENT_SETTID)
            put_user(nr, parent_tidptr);
        //......
        // 唤醒新进程
        wake_up_new_task(p);
        //......
        put_pid(pid);
    }
    //...
} 


static struct task_struct *copy_process(...){
    // 复制进程task_struct结构体
    struct task_struct *p;
    p = dup_task_struct(current, ...);
    // 复制files_struct
    retval = copy_files(clone_flags,p)
    // 复制fs_struct
    retval = copy_fs(clone_flags,p)
    // 复制mm_struct
    retval = copy_mm(clone_flags,p)
    // 复制进程的命名空间 nsproxy
    retval = copy_namespace(clone_flags,p)
    // 申请pid并设置进程号
    pid = alloc_pid(p->nsproxy->pid_ns_for_children,...);
    p->pid = pid_nr(pid);
    if (clone_flags & CLONE_THREAD) {
        p->tgid = current->tgid;
    }else{
        p->tgid = p->pid;
    }
    ...
}
```

一个新创建的普通进程，从调度的代码视角看，`copy_process`函数对新进程的`task_struct` 进行各种初始化，其中会调用`sched_fork` 来完成调度相关的初始化，核心逻辑如下：

```CPP
static struct task_struct *copy_process(...){
    //...
    retval = sched_fork(clone_flags, p);

	//...
}

int sched_fork(unsigned long clone_flags, struct task_struct *p){
    __schedd_fork(clone_flags, p);
    p->__state = TASK_NEW;
    if (rt_prio(p->prio))
        p->sched_class = &rt_sched_class;
    else
        p->sched_class = &fair_sched_class; // fair_sched_class （CFS的调度类）是一个全局对象，为CFS调度算法的实现
}

static void __sched_fork(struct task_struct *p){
    p ->on_rq = 0;
    //...
    p->se.nr_migrations = 0;
    p->se.vruntime = 0; // 注意：新进程是0对老进程不公平，在新进程真正被加入运行队列时，会将其值设置为cfs_rq->min_vruntime
}

void wake_up_new_task(struct task_struct *p){
    // 为进程选择一个合适的cpu
    cpu = select_task_rq(p, task_cpu(p)
    // 为进程指定运行队列
    __set_task_cpu(p,cpu, WF_FORK));
    // 将进程添加到运行队列红黑树
    rq = __task_rq_lock(p);
    activate_task(rq, p, 0);
}
```

需要说明的是上面`wake_up_new_task`实现中，通过调用`select_task_rq()`函数重新选择CPU（逻辑核），通过调用调度类中`select_task_rq`方法选择调度类中最空闲的CPU。如此在缓存性能和空闲核两个点做权衡，同等条件会尽量优先考虑缓存命中率，选择同`L1`/`L2`的核，其次会选择同一个物理CPU上的（共享`L3`），最坏情况下去选择另一个（负载最小的）物理CPU上的核，称之为漂移。通常由于宿主机的CPU利用率过高的水平导致出现了漂移的情况，进程在不同的核上运行概率增加，导致缓存MISS，穿透到内存的访问次数增加，进程的运行性能就会下降

####	CFS调度类及核心函数实现
`fair_sched_class`是CFS调度类实现，调用调度类中的task_fork函数

```CPP
const struct sched_class fair_sched_class = {
	.next				= &idle_sched_class,
	.enqueue_task		= enqueue_task_fair,
	.dequeue_task		= dequeue_task_fair,
	.yield_task			= yield_task_fair,
	.yield_to_task		= yield_to_task_fair,
	.check_preempt_curr	= check_preempt_wakeup,
	.pick_next_task		= pick_next_task_fair,
	.put_prev_task		= put_prev_task_fair,
	//..
	.set_curr_task      = set_curr_task_fair,
	.task_tick			= task_tick_fair,
	.task_fork			= task_fork_fair,
	.prio_changed		= prio_changed_fair,
	.switched_from		= switched_from_fair,
	.switched_to		= switched_to_fair,
	.get_rr_interval	= get_rr_interval_fair,
	.update_curr		= update_curr_fair,
};
```

`task_fork_fair`函数主要做fork相关的操作，参数`p`就是创建的`task_struct`
```CPP
static void task_fork_fair(struct task_struct *p)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &p->se, *curr;
	struct rq *rq = this_rq();
	struct rq_flags rf;
 
	rq_lock(rq, &rf);
	update_rq_clock(rq);
 
	cfs_rq = task_cfs_rq(current);
	//cfs_rq是CFS调度器就绪队列，curr指向当前正在cpu上运行的task的调度实体
	curr = cfs_rq->curr;                 
	if (curr) {
		//update_curr()函数是比较重要，主要是更新当前正在运行的调度实体的运行时间信息
		update_curr(cfs_rq);                 
		//初始化当前创建的新进程的虚拟时间
		se->vruntime = curr->vruntime;       
	}

	//place_entity()函数在进程创建以及唤醒的时候都会调用，创建进程的时候传递参数initial=1。主要目的是更新调度实体得到虚拟时间（se->vruntime成员）。要和cfs_rq->min_vruntime的值保持差别不大
	place_entity(cfs_rq, se, 1);             
 
	/*
	这里为什么要减去cfs_rq->min_vruntime呢？因为现在计算进程的vruntime是基于当前cpu上的cfs_rq，并且现在还没有加入当前cfs_rq的就绪队列上。等到当前进程创建完毕开始唤醒的时候，加入的就绪队列就不一定是现在计算基于的cpu。所以，在加入就绪队列的函数中会根据情况加上当前就绪队列cfs_rq->min_vruntime。为什么要先减后加处理呢？假设cpu0上的cfs就绪队列的最小虚拟时间min_vruntime的值是1000000，此时创建进程的时候赋予当前进程虚拟时间是1000500。但是，唤醒此进程加入的就绪队列却是cpu1上CFS就绪队列，cpu1上的cfs就绪队列的最小虚拟时间min_vruntime的值如果是9000000。如果不采用先减后加方法，那么该进程在cpu1上运行肯定是乐坏了，疯狂的运行。现在的处理计算得到的调度实体的虚拟时间是1000500 - 1000000 + 9000000 = 9000500，因此事情就不是那么的糟糕
	*/

	// 另外这里减去的min_vruntime会在enqueue_entity入红黑树的时候加回
	se->vruntime -= cfs_rq->min_vruntime;   
	rq_unlock(rq, &rf);
}
```

```CPP
static void update_curr(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq->curr;
	u64 now = rq_clock_task(rq_of(cfs_rq));
	u64 delta_exec;
 
	if (unlikely(!curr))
		return;
 
	//计算本次更新虚拟时间距离上次更新虚拟时间的差
	delta_exec = now - curr->exec_start;                    
	if (unlikely((s64)delta_exec <= 0))
		return;
 
	curr->exec_start = now;
	curr->sum_exec_runtime += delta_exec;
	//更新当前调度实体虚拟时间，calc_delta_fair()函数根据上面说的虚拟时间的计算公式计算虚拟时间（也就是调用__calc_delta()函数）
	curr->vruntime += calc_delta_fair(delta_exec, curr);    
	// 更新CFS就绪队列的最小虚拟时间min_vruntime。min_vruntime也是不断更新的，主要就是跟踪就绪队列中所有调度实体的最小虚拟时间。如果min_vruntime一直不更新的话，由于min_vruntime太小，导致后面创建的新进程若根据这个值来初始化新进程的虚拟时间，那么就会扰乱调度新老进程的优先级
	update_min_vruntime(cfs_rq);                            
}
```

`update_min_vruntime`函数的意义在于动态更新当前调度器对应的CPU就绪队列最小虚拟时间（就绪队列本身的`cfs_rq->min_vruntime`成员）,由于CFS调度器选择最适合运行的进程是选择维护的红黑树中虚拟时间最小的进程，如果在当前进程运行的过程中（其`vruntime`不断增大），有进程加入就绪队列，那么红黑树最左边的进程的虚拟时间同样也有可能是最小的虚拟时间，这需要保证就绪队列的最小虚拟时间`min_vruntime`单调递增的特性来更新此值

```CPP
static void update_min_vruntime(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq->curr;
	struct rb_node *leftmost = rb_first_cached(&cfs_rq->tasks_timeline);
	u64 vruntime = cfs_rq->min_vruntime;
 
	if (curr) {
		if (curr->on_rq)
			vruntime = curr->vruntime;
		else
			curr = NULL;
	}
 
	if (leftmost) { /* non-empty tree */
		struct sched_entity *se;
		se = rb_entry(leftmost, struct sched_entity, run_node);
 
		if (!curr)
			vruntime = se->vruntime;
		else
			vruntime = min_vruntime(vruntime, se->vruntime);
	}
 
	/* ensure we never gain time by being placed backwards. */
	cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime);
}
```

主流程`place_entity`，由于`vruntime`越小越容易被调度执行，所以这里要做两件事：
-	对新创建的进程进行惩罚
-	对旧进程（sleep很久）的进行进程补偿

```CPP
static void
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
	u64 vruntime = cfs_rq->min_vruntime;
 
	/*
	 * The 'current' period is already promised to the current tasks,
	 * however the extra weight of the new task will slow them down a
	 * little, place the new task so that it fits in the slot that
	 * stays open at the end.
	 */
	//如果是创建进程调用该函数的话，参数initial参数是1
	//因此这里是处理创建的进程，针对刚创建的进程会进行一定的惩罚，将虚拟时间加上一个值就是惩罚，毕竟虚拟时间越小越容易被调度执行
	//惩罚的时间由sched_vslice()计算
	if (initial && sched_feat(START_DEBIT))
		vruntime += sched_vslice(cfs_rq, se);              
 
	/* sleeps up to a single latency don't count. */
	if (!initial) {
		unsigned long thresh = sysctl_sched_latency;
 
		/*
		 * Halve their sleep time's effect, to allow
		 * for a gentler effect of sleepers:
		 */
		if (sched_feat(GENTLE_FAIR_SLEEPERS))
			thresh >>= 1;
		
		//这里主要是针对唤醒的进程，针对睡眠很久的的进程，我们总是期望它很快得到调度执行，毕竟睡了那么久
		//所以这里减去一定的虚拟时间作为补偿
		vruntime -= thresh;                                 
	}
 
	/* ensure we never gain time by being placed backwards. */
	//保证调度实体的虚拟时间不能倒退。为何呢？可以想一下，如果一个进程刚睡眠1ms，然后醒来后你却要奖励3ms（虚拟时间减去3ms），然后他竟然赚了2ms。作为调度器，我们不做亏本生意。你睡眠100ms，奖励你3ms，那就是没问题的
	se->vruntime = max_vruntime(se->vruntime, vruntime);    
}
```

惩罚时间的计算由`sched_vslice`函数完成：

```CPP
static u64 sched_vslice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	return calc_delta_fair(sched_slice(cfs_rq, se), se);
}

static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);    /* 1 */
 
	for_each_sched_entity(se) {                                     /* 2 */
		struct load_weight *load;
		struct load_weight lw;
 
		cfs_rq = cfs_rq_of(se);
		load = &cfs_rq->load;                                       /* 3 */
 
		if (unlikely(!se->on_rq)) {
			lw = cfs_rq->load;
 
			update_load_add(&lw, se->load.weight);
			load = &lw;
		}
		slice = __calc_delta(slice, se->load.weight, load);         /* 4 */
	}
	return slice;
}
```

####	新进程的调度过程


##	0x	总结

####	vruntime
`vruntime`本质是一个累计值，作为每个调度实体（`struct sched_entity`）的 vruntime 字段记录该进程的加权累计运行时间

1、`vruntime`的累加性质

2、`vruntime`的特殊调整

3、`vruntime`的底层实现

##	0x	一些细节

####	如何检测时间耗尽？
`sched_slice`可以计算计算一个调度周期内，一个调度实体可以分配多少运行时间，那么CPU如何知道这个调度实体已经运行到达它的运行时间上限了呢？换句话说CPU如何检查某个进程已经用完了它此刻的"时间片"？


##  0x0  参考
-   [【原创】（五）Linux进程调度-CFS调度器](https://www.cnblogs.com/LoyenWang/p/12495319.html)
-   [CFS调度器（1）-基本原理](http://www.wowotech.net/process_management/447.html)
-   [调度系统设计精要](https://mp.weixin.qq.com/s/R3BZpYJrBPBI0DwbJYB0YA)
-   [一文搞懂linux cfs调度器](https://zhuanlan.zhihu.com/p/556295381)
-	[Linux 进程调度(4)- min_vruntime (CFS 调度器)](https://zhuanlan.zhihu.com/p/363674137)