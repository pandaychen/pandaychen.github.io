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

##  0x01   使用

####    获取 cpu 基础信息
依次输出每个 core 的 cpu 详细信息：
```golang
cpuInfos, err := cpu.Info()
if err != nil {
    fmt.Printf("get cpu info failed, err:%v", err)
}
for index, ci := range cpuInfos {
    fmt.Println(index,ci)
}
```
输出如下（测试机仅有 1 核）：
```json
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


##  0x02    指标的特性
CPU 的波动可能是非常敏感的，另外在容器的场景中需要关注 Cgroup 限制下的 cpu 使用情况：
-   单核 CPU
-   多核 CPU
-   容器 Cgroup 限制场景


##  0x03    参考
-   [Linux Performance](http://www.brendangregg.com/linuxperf.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权