---
layout:     post
title:      Linux 系统服务常用采集指标
subtitle:   gopsutil 常用方法介绍
date:       2020-04-23
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Metrics
---


##  0x00 前言
通常，在负载均衡算法判断中，对服务端的负载衡量，常用的指标有 CPU、内存、io 负载等，客户端的主要指标有调用延迟，处于等待 Response 的请求数、调用成功率等

在 Linux 下，通过使用 [gopsutil](https://github.com/shirou/gopsutil) 库，可以非常方便的采集系统指标。

##  0x01   基础使用

####    获取 cpu 基础信息
使用 `Info` 方法，依次输出每个 core 的 cpu 详细信息：
```golang
cpuInfos, err := cpu.Info()
if err != nil {
	fmt.Printf("get cpu info failed, err:%v", err)
	return err
}
for index, ci := range cpuInfos {
    fmt.Println(index,ci)
}
```

输出如下（测试机仅有 1 核）：
```javascript
{
	"cpu": 0,
	"vendorId": "GenuineIntel",
	"family": "6",
	"model": "79",
	"stepping": 1,
	"physicalId": "0",
	"coreId": "0",
	"cores": 1,
	"modelName": "Intel(R) Xeon(R) CPU E5-26xx v4",
	"mhz": 2399.988,
	"cacheSize": 4096,
	"flags": ["fpu", "vme", "de", "pse", "tsc", "msr", "pae", "mce", "cx8", "apic", "sep", "mtrr", "pge", "mca", "cmov", "pat", "pse36", "clflush", "mmx", "fxsr", "sse", "sse2", "ss", "ht", "syscall", "nx", "lm", "constant_tsc", "rep_good", "nopl", "eagerfpu", "pni", "pclmulqdq", "ssse3", "fma", "cx16", "pcid", "sse4_1", "sse4_2", "x2apic", "movbe", "popcnt", "tsc_deadline_timer", "aes", "xsave", "avx", "f16c", "rdrand", "hypervisor", "lahf_lm", "abm", "3dnowprefetch", "bmi1", "avx2", "bmi2", "rdseed", "adx", "xsaveopt"],
	"microcode": "0x1"
}
```

####	CPU 的使用率
通过 `Percent` 方法采集 CPU 的使用率，特别需要注意传入采集的周期参数。比如下面这段代码，传入的周期是 200ms，采集的结果：
```golang
for {
	percent, _ := cpu.Percent(200*time.Millisecond, false)
	fmt.Printf("cpu percent:%v\n", percent)
}
``
`
200ms 采集的 CPU 使用率结果，有为 `0` 的数据，这样可以不一定能反应 CPU 的平均使用率。我们把采集周期调整为 500ms，再看看结果：
```bash
cpu percent:[4.761904466246813]
cpu percent:[5]
cpu percent:[5.000000232830649]
cpu percent:[10.526315518590394]
cpu percent:[5.263158088224914]
cpu percent:[0]
cpu percent:[9.52380960828323]
cpu percent:[5.000000046566129]
cpu percent:[0]
```

500ms 的采集结果如下，现在的采集结果就平滑很多：
```bash
cpu percent:[2.0408163672590045]
cpu percent:[5.769230776119249]
cpu percent:[2.040816272226089]
cpu percent:[8.000000100582838]
cpu percent:[2.040816272226089]
```

####	采集 host 信息
```golang
import (
        "fmt"
        "github.com/shirou/gopsutil/host"
)

// host info
func getHostInfo() {
        hInfo, _ := host.Info()
        fmt.Printf("host info:%v uptime:%v boottime:%v\n", hInfo, hInfo.Uptime, hInfo.BootTime)
}
```

####	采集内存信息

```golang
import (
	"github.com/shirou/gopsutil/mem"
)

func getMemInfo() {
	memInfo, _ := mem.VirtualMemory()
	fmt.Println(memInfo)
}
```

测试机输出如下：

```javascript
{
	"total": 1040912384,
	"available": 182886400,
	"used": 623673344,
	"usedPercent": 59.91602689972415,
	"free": 141213696,
	"active": 702181376,
	"inactive": 81600512,
	"wired": 0,
	"laundry": 0,
	"buffers": 3579904,
	"cached": 272445440,
	"writeback": 0,
	"dirty": 3244032,
	"writebacktmp": 0,
	"shared": 75890688,
	"slab": 76566528,
	"sreclaimable": 55255040,
	"sunreclaim": 21311488,
	"pagetables": 8572928,
	"swapcached": 0,
	"commitlimit": 520454144,
	"committedas": 3280732160,
	"hightotal": 0,
	"highfree": 0,
	"lowtotal": 0,
	"lowfree": 0,
	"swaptotal": 0,
	"swapfree": 0,
	"mapped": 38567936,
	"vmalloctotal": 35184372087808,
	"vmallocused": 8962048,
	"vmallocchunk": 35184348753920,
	"hugepagestotal": 0,
	"hugepagesfree": 0,
	"hugepagesize": 2097152
}
```

##	0x02	Docker 信息
查看下当前 system 中正在运行的容器：
```bash
[root@VM_0_7_centos ~]# docker container list
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
c1dfd58d5ef7        docker.io/ubuntu    "/bin/bash"         27 minutes ago      Up 27 minutes                           competent_nightingale
```

采集容器信息：
```golang
import "github.com/shirou/gopsutil/docker"

func CgroupCPU() {
	v, _ := docker.GetDockerIDList()
	fmt.Println(v)
	for _, id := range v {
		v, err := docker.CgroupCPUDocker(id)
		if err != nil {
				fmt.Println("error %v", err)
				continue
		}
		if v.CPU == "" {
				fmt.Println("could not get CgroupCPU %v", v)
				continue
		}
		fmt.Println(id, v)
	}
}
```

执行结果：
```javascript
{
	"cpu": "c1dfd58d5ef72250392f1b03898e8f4d40110ba73913d945ce475e8657e7d53e",
	"user": 0.0,
	"system": 0.0,
	"idle": 0.0,
	"nice": 0.0,
	"iowait": 0.0,
	"irq": 0.0,
	"softirq": 0.0,
	"steal": 0.0,
	"guest": 0.0,
	"guestNice": 0.0
}
```

##  0x03    指标的特性
CPU 的波动可能是非常敏感的，另外在容器的场景中需要关注 Cgroup 限制下的 CPU 及 内存限制，在另外一篇博客 -[GOMAXPROCS 的坑](https://pandaychen.github.io/2020/02/28/GOMAXPROCS-POT/)，有介绍。
-   单核 CPU
-   多核 CPU
-   容器 Cgroup 限制场景


##  0x03    参考
-   [Linux Performance](http://www.brendangregg.com/linuxperf.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权