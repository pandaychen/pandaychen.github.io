---
layout:     post
title:      Uber-Automaxprocs 分析
subtitle:	Docker中的CPU调度总结
date:       2020-02-29
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - GOMAXPROCS
---

##	0x00	前言
&emsp;&emsp;前一排文章[GOMAXPROCS 的 “坑”](https://pandaychen.github.io/2020/02/28/GOMAXPROCS-POT/)，简单描述了GOMAXPROCS在容器场景可能会出现的问题。解决方法是使用Uber提供的Automaxprocs包，自动的根据CGROUP值识别容器的CPU quota，并自动设置GOMAXPROCS线程数量，本篇文章就简答分析下Automaxprocs是如何做到做一点的。

##	0x01	再看Docker中的CPU调度
docker官方文档中指出：
> By default, each container's access to the host machine's CPU cycles is unlimited. You can set various constraints to limit a given container's access to the host machine's CPU cycles. Most users use and configure the default CFS scheduler. In Docker 1.13 and higher, you can also configure the realtime scheduler.

小结下：
-	默认容器会使用宿主机CPU是不受限制的
-	要限制容器使用CPU，可以通过参数设置CPU的使用，又细分为两种策略：
	*	将容器设置为普通进程，通过完全公平调度算法（CFS，Completely Fair Scheduler）调度类实现对容器CPU的限制 -- **默认方案**
	*	将容器设置为实时进程，通过实时调度类进行限制

另一种是将容器设置为实时进程，通过实时调度类进行限制，我们这里仅考虑默认方案，即通过CFS调度类实现对容器CPU的限制。（我们下面的分析默认了进程只进行CPU操作，没有睡眠、IO等操作，换句话说，进程的生命周期中一直在就绪队列中排队，要么在用CPU，要么在等CPU）

docker（docker run）配置CPU使用量的参数主要下面几个，这些参数主要是通过配置在容器对应cgroup中，由cgroup进行实际的CPU管控。其对应的路径可以从cgroup中查看到
```bash
  --cpu-shares                    CPU shares (relative weight)
  --cpu-period                    Limit CPU CFS (Completely Fair Scheduler) period
  --cpu-quota                     Limit CPU CFS (Completely Fair Scheduler) quota
  --cpuset-cpus                   CPUs in which to allow execution (0-3, 0,1)
```

搞懂CGROUP对CPU的管理策略对理解Automaxprocs的源码有很大的帮助。

（少选项分析的内容，绑定的缺点）

##	0x02	Kubernetes中的CPU调度管理
kubernetes对容器可以设置两个关于CPU的值：limits和requests，即`spec.containers[].resources.limits.cpu`和`spec.containers[].resources.requests.cpu`，对应了上面的配置选项，如下面的配置：
```yaml
image: ---------
        imagePullPolicy: IfNotPresent
        name: pandaychen-test-app1
        resources:
          limits:
            cpu: "2"
            memory: 4196Mi
          requests:
            cpu: "1"
            memory: 1Gi
        securityContext:
          privileged: false
          procMount: Default
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
```

关于limits和requests则两个值：
*	limits：该（单）pod使用的最大的CPU核数
	`limits=cfs_quota_us/cfs_period_us`的值。比如limits.cpu=3（核），则cfs_quota_us=300000，cfs_period_us值一般都使用默认的100000

*	requests：该（单）pod使用的最小的CPU核数，为pod调度提供计算依据
	*	一方面则体现在容器设置`--cpu-shares`上，比如requests.cpu=3，--cpu-shares=1024，则cpushare=1024*3=3072。
	*	另一方面，比较重要，**用来计算Node的CPU的已经分配的量就是通过计算所有容器的requests的和得到的，那么该Node还可以分配的量就是该Node的CPU核数减去前面这个值**。当创建一个Pod时，Kubernetes调度程序将为Pod选择一个Node。每个Node具有每种资源类型的最大容量：可为Pods提供的CPU和内存量。调度程序确保对于每种资源类型，调度的容器的资源请求的总和小于Node的容量。尽管Node上的实际内存或CPU资源使用量非常低，但如果容量检查失败，则调度程序仍然拒绝在节点上放置Pod。

（少kubernetes的优点）

##	0x03	Automaxprocs 解决了什么问题
让我们回到GOMAXPROCS的问题，一般在部署容器应用时，通常会对 CPU 资源做限制，例如上面yaml文件的，上限是2个核。而实际应用的pod中，通过 lscpu命令 ，我们仍然能看到宿主机的所有 CPU 核心，如下面是笔者的一个Kubernetes集群中的Pod信息：
![image](https://s2.ax1x.com/2020/02/29/36whjK.png)

这会导致 Golang 服务默认会拿宿主机的 CPU 核心数来调用 runtime.GOMAXPROCS()，导致 P 数量远远大于可用的 CPU 核心，引起频繁上下文切换，影响高负载情况下的服务性能。而Uber-Automaxprocs这个库 能够正确识别容器允许使用的核心数，合理的设置 processor数目，避免高并发下的切换问题。

##	0x04	Automaxprocs的源码分析

####	1、初始化
通过[Readme.md](https://github.com/uber-go/automaxprocs/blob/master/README.md)中的import方式，
```go
import _ "go.uber.org/automaxprocs"
```
大概可以猜到，该包的 [init方法](https://github.com/uber-go/automaxprocs/blob/master/automaxprocs.go#L31)是package级别的。导入即生效。

init方法：
```go
func init() {
	//入口，核心方法
	maxprocs.Set(maxprocs.Logger(log.Printf))
}
```

####	2、maxprocs.Set()
```go
func Set(opts ...Option) (func(), error) {
	cfg := &config{
		procs:         iruntime.CPUQuotaToGOMAXPROCS,
		minGOMAXPROCS: 1,
	}
	for _, o := range opts {
		o.apply(cfg)
	}

	undoNoop := func() {
		cfg.log("maxprocs: No GOMAXPROCS change to reset")
	}

	// Honor the GOMAXPROCS environment variable if present. Otherwise, amend
	// `runtime.GOMAXPROCS()` with the current process' CPU quota if the OS is
	// Linux, and guarantee a minimum value of 1. The minimum guaranteed value
	// can be overriden using `maxprocs.Min()`.
	if max, exists := os.LookupEnv(_maxProcsKey); exists {
		cfg.log("maxprocs: Honoring GOMAXPROCS=%q as set in environment", max)
		return undoNoop, nil
	}

	//核心函数，调用 iruntime.CPUQuotaToGOMAXPROCS 得到最终的maxProcs
	maxProcs, status, err := cfg.procs(cfg.minGOMAXPROCS)
	if err != nil {
		return undoNoop, err
	}

	if status == iruntime.CPUQuotaUndefined {
		cfg.log("maxprocs: Leaving GOMAXPROCS=%v: CPU quota undefined", currentMaxProcs())
		return undoNoop, nil
	}

	prev := currentMaxProcs()
	undo := func() {
		cfg.log("maxprocs: Resetting GOMAXPROCS to %v", prev)
		runtime.GOMAXPROCS(prev)
	}

	switch status {
	case iruntime.CPUQuotaMinUsed:
		cfg.log("maxprocs: Updating GOMAXPROCS=%v: using minimum allowed GOMAXPROCS", maxProcs)
	case iruntime.CPUQuotaUsed:
		cfg.log("maxprocs: Updating GOMAXPROCS=%v: determined from CPU quota", maxProcs)
	}

	//调用系统的runtime完成功能
	runtime.GOMAXPROCS(maxProcs)
	return undo, nil
}
```

##	参考
-	[Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)