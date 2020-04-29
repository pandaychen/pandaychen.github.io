---
layout:     post
title:      Uber-Automaxprocs 分析
subtitle:	Docker 中的 CPU 调度总结
date:       2020-02-29
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - GOMAXPROCS
---

##	0x00	前言
&emsp;&emsp; 前一篇文章 [GOMAXPROCS 的 “坑”](https://pandaychen.github.io/2020/02/28/GOMAXPROCS-POT/)，简单描述了 GOMAXPROCS 在容器场景可能会出现的问题。解决方法是使用 Uber 提供的 Automaxprocs 包，自动的根据 CGROUP 值识别容器的 CPU quota，并自动设置 GOMAXPROCS 线程数量，本篇文章就简答分析下 Automaxprocs 是如何做到做一点的。

##	0x01	再看 Docker 中的 CPU 调度
docker 官方文档中指出：
> By default, each container's access to the host machine's CPU cycles is unlimited. You can set various constraints to limit a given container's access to the host machine's CPU cycles. Most users use and configure the default CFS scheduler. In Docker 1.13 and higher, you can also configure the realtime scheduler.

小结下：
-	默认容器会使用宿主机 CPU 是不受限制的
-	要限制容器使用 CPU，可以通过参数设置 CPU 的使用，又细分为两种策略：
	*	将容器设置为普通进程，通过完全公平调度算法（CFS，Completely Fair Scheduler）调度类实现对容器 CPU 的限制 -- ** 默认方案 **
	*	将容器设置为实时进程，通过实时调度类进行限制

另一种是将容器设置为实时进程，通过实时调度类进行限制，我们这里仅考虑默认方案，即通过 CFS 调度类实现对容器 CPU 的限制。（我们下面的分析默认了进程只进行 CPU 操作，没有睡眠、IO 等操作，换句话说，进程的生命周期中一直在就绪队列中排队，要么在用 CPU，要么在等 CPU）

docker（docker run）配置 CPU 使用量的参数主要下面几个，这些参数主要是通过配置在容器对应 cgroup 中，由 cgroup 进行实际的 CPU 管控。其对应的路径可以从 cgroup 中查看到
```bash
  --cpu-shares                    CPU shares (relative weight)
  --cpu-period                    Limit CPU CFS (Completely Fair Scheduler) period
  --cpu-quota                     Limit CPU CFS (Completely Fair Scheduler) quota
  --cpuset-cpus                   CPUs in which to allow execution (0-3, 0,1)
```

搞懂 CGROUP 对 CPU 的管理策略对理解 Automaxprocs 的源码有很大的帮助。

（少选项分析的内容，绑定的缺点）

##	0x02	Kubernetes 中的 CPU 调度管理
kubernetes 对容器可以设置两个关于 CPU 的值：limits 和 requests，即 `spec.containers[].resources.limits.cpu` 和 `spec.containers[].resources.requests.cpu`，对应了上面的配置选项，如下面的配置：
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

关于 limits 和 requests 则两个值：
*	limits：该（单）pod 使用的最大的 CPU 核数
	`limits=cfs_quota_us/cfs_period_us` 的值。比如 limits.cpu=3（核），则 cfs_quota_us=300000，cfs_period_us 值一般都使用默认的 100000

*	requests：该（单）pod 使用的最小的 CPU 核数，为 pod 调度提供计算依据
	*	一方面则体现在容器设置 `--cpu-shares` 上，比如 requests.cpu=3，--cpu-shares=1024，则 cpushare=1024*3=3072。
	*	另一方面，比较重要，** 用来计算 Node 的 CPU 的已经分配的量就是通过计算所有容器的 requests 的和得到的，那么该 Node 还可以分配的量就是该 Node 的 CPU 核数减去前面这个值 **。当创建一个 Pod 时，Kubernetes 调度程序将为 Pod 选择一个 Node。每个 Node 具有每种资源类型的最大容量：可为 Pods 提供的 CPU 和内存量。调度程序确保对于每种资源类型，调度的容器的资源请求的总和小于 Node 的容量。尽管 Node 上的实际内存或 CPU 资源使用量非常低，但如果容量检查失败，则调度程序仍然拒绝在节点上放置 Pod。

（少 kubernetes 的优点）

##	0x03	Automaxprocs 解决了什么问题
让我们回到 GOMAXPROCS 的问题，一般在部署容器应用时，通常会对 CPU 资源做限制，例如上面 yaml 文件的，上限是 2 个核。而实际应用的 pod 中，通过 lscpu 命令 ，我们仍然能看到宿主机的所有 CPU 核心，如下面是笔者的一个 Kubernetes 集群中的 Pod 信息：
![image](https://s2.ax1x.com/2020/02/29/36whjK.png)

这会导致 Golang 服务默认会拿宿主机的 CPU 核心数来调用 runtime.GOMAXPROCS()，导致 P 数量远远大于可用的 CPU 核心，引起频繁上下文切换，影响高负载情况下的服务性能。而 Uber-Automaxprocs 这个库 能够正确识别容器允许使用的核心数，合理的设置 processor 数目，避免高并发下的切换问题。

##	0x04	Automaxprocs 的源码分析
我们知道，docker使用cgroup来限制容器CPU使用, 使用该容器配置的cpu.cfsquotaus/cpu.cfsperiodus即可获得CPU配额. 所以关键是找到容器的这两个值.

####	1、初始化
通过 [Readme.md](https://github.com/uber-go/automaxprocs/blob/master/README.md) 中的 import 方式，
```go
import _ "go.uber.org/automaxprocs"
```
大概可以猜到，该包的 [init 方法](https://github.com/uber-go/automaxprocs/blob/master/automaxprocs.go#L31) 是 package 级别的。导入即生效。

init 方法：
```go
func init() {
	// 入口，核心方法
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

	// 核心函数，调用 iruntime.CPUQuotaToGOMAXPROCS 得到最终的 maxProcs
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

	// 调用系统的 runtime 完成功能
	runtime.GOMAXPROCS(maxProcs)
	return undo, nil
}
```

####	解析进程的 CGroup 信息
对应的方法是[`parseCGroupSubsystems()`](https://github.com/uber-go/automaxprocs/blob/master/internal/cgroups/subsys.go#L77)，该方法的核心在于解析`/proc/$pid/cgroup`文件，转换成[`CGroupSubsys`结构](https://github.com/uber-go/automaxprocs/blob/master/internal/cgroups/subsys.go#L46)
```go
// parseCGroupSubsystems parses procPathCGroup (usually at `/proc/$PID/cgroup`)
// and returns a new map[string]*CGroupSubsys.
func parseCGroupSubsystems(procPathCGroup string) (map[string]*CGroupSubsys, error) {
	cgroupFile, err := os.Open(procPathCGroup)
	if err != nil {
		return nil, err
	}
	defer cgroupFile.Close()

	scanner := bufio.NewScanner(cgroupFile)
	subsystems := make(map[string]*CGroupSubsys)

	for scanner.Scan() {
		//解析文本
		cgroup, err := NewCGroupSubsysFromLine(scanner.Text())
		if err != nil {
			return nil, err
		}
		for _, subsys := range cgroup.Subsystems {
			subsystems[subsys] = cgroup
		}
	}

	if err := scanner.Err(); err != nil {
		return nil, err
	}

	return subsystems, nil
}
```

####	解析进程的MountInfo信息
[parseMountInfo方法](https://github.com/uber-go/automaxprocs/blob/master/internal/cgroups/mountpoint.go#L146)
（未完待续）


##	参考
-	[Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)