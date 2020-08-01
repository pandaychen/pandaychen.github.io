---
layout:     post
title:      Kratos 源码分析：CPU 指标采集
subtitle:   了解 gopsutil 常用方法应用及 Linux 系统服务常用采集指标
date:       2020-04-23
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Metrics
---


##  0x00 前言
通常，在负载均衡算法判断中，对服务端的负载衡量，常用的指标有 CPU、内存、io 负载等，客户端的主要指标有调用延迟，处于等待 Response 的请求数、调用成功率等，其中 CPU 使用率是一个及其重要的指标。

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
通过 `Percent` 方法采集 CPU 的使用率，特别需要注意传入采集的周期参数。比如下面这段代码，传入的周期是 `200ms`，采集的结果：

```golang
for {
	percent, _ := cpu.Percent(200*time.Millisecond, false)
	fmt.Printf("cpu percent:%v\n", percent)
}
```

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


##	0x04	Kratos 中的采集指标分析
Kratos 中框架中重点关注对服务端 CPU 指标的采集：封装 [代码在此](https://github.com/go-kratos/kratos/tree/master/pkg/stat/sys/cpu)，主要提供的功能：获取 Linux 平台下的系统信息，包括 cpu 主频、cpu 使用率等。主要区分了两种场景：
-	普通 CVM
-	Docker-Cgroup 场景

####	对 psutil 的封装
对 psutil 包的封装 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/stat/sys/cpu/psutilCPU.go)，Kratos 中默认的 CPU 采集周期是 `interval time.Duration = time.Millisecond * 500`，即 `500ms`。

创建 `psutilCPU` 结构：
```golang
type psutilCPU struct {
	interval time.Duration
}

func newPsutilCPU(interval time.Duration) (cpu *psutilCPU, err error) {
	cpu = &psutilCPU{interval: interval}
	_, err = cpu.Usage()
	if err != nil {
		return
	}
	return
}
```

`psutilCPU` 仅提供了两个方法：
-	`Usage`：调用 `cpu.Percent` 方法，获取到 CPU 的使用率信息（`500ms` 采样的总的 CPU 使用率乘以 `10`)
-	`Info`：拿到逻辑核 core 核心数 `cpu.Counts(true)` 及主频信息 `uint64(stats[0].Mhz)`

```golang
func (ps *psutilCPU) Usage() (u uint64, err error) {
	var percents []float64
	//Percent(interval time.Duration, percpu bool)：表示获取 interval 时间间隔内的 CPU 使用率，percpu 为 false 时，获取总的 CPU 使用率，percpu 为 true 时，分别获取每个 CPU 的使用率，返回一个 []float64 类型的值
	percents, err = cpu.Percent(ps.interval, false)
	if err == nil {
		//*10 再返回
		u = uint64(percents[0] * 10)
	}
	return
}

func (ps *psutilCPU) Info() (info Info) {
	stats, err := cpu.Info()
	if err != nil {
		return
	}
	cores, err := cpu.Counts(true)
	if err != nil {
		return
	}
	info = Info{
		// 返回第 0 号 cpu 的主频信息
		Frequency: uint64(stats[0].Mhz),
		Quota:     float64(cores),
	}
	return
}
```

####	获取 Cgroup 属性封装
Kratos中对[cgroupCPU封装](https://github.com/go-kratos/kratos/blob/master/pkg/stat/sys/cpu/cgroupCPU.go)实现，默认 cgroup 的配置路径为 `cgroupRootDir=/sys/fs/cgroup`，`cgroup` 结构为一个 `map`，主要存储 cgroup** 子系统的参数路径 **：key 为 cpu、cpuacct 及 cpuset
```golang
// cgroup Linux cgroup
type cgroup struct {
	cgroupSet map[string]string
}
```

生成 `cgroup` 对象，对应的方法是 [`currentcGroup`](https://github.com/go-kratos/kratos/blob/master/pkg/stat/sys/cpu/cgroup.go#L85)：这个方法是获取到当前进程（调用此方法的进程）对应的 cgroup 文件，然后解析 cgroup 文件得到每一项的路径，保存在 `cgroupSet` 这个 `map` 中，其中 `key` 为子系统的名字，`value` 为绝对路径。举例来说：

（少例子）

另外注意区分是否是容器中的进程的方式：判断 cgroup 文件的第 `3` 列是否为 `"/"` 根目录。

```golang
func currentcGroup() (*cgroup, error) {
	pid := os.Getpid()
	cgroupFile := fmt.Sprintf("/proc/%d/cgroup", pid)
	cgroupSet := make(map[string]string)
	fp, err := os.Open(cgroupFile)
	if err != nil {
		return nil, err
	}
	defer fp.Close()
	buf := bufio.NewReader(fp)
	for {
		line, err := buf.ReadString('\n')
		if err != nil {
			if err == io.EOF {
				break
			}
			return nil, err
		}
		col := strings.Split(strings.TrimSpace(line), ":")
		if len(col) != 3 {
			return nil, fmt.Errorf("invalid cgroup format %s", line)
		}
		dir := col[2]
		// When dir is not equal to /, it must be in docker
		if dir != "/" {
			// 容器中的进程
			cgroupSet[col[1]] = path.Join(cgroupRootDir, col[1])
			if strings.Contains(col[1], ",") {
				for _, k := range strings.Split(col[1], ",") {
					cgroupSet[k] = path.Join(cgroupRootDir, k)
				}
			}
		} else {
			// dir == "/"
			cgroupSet[col[1]] = path.Join(cgroupRootDir, col[1], col[2])
			if strings.Contains(col[1], ",") {
				for _, k := range strings.Split(col[1], ",") {
					cgroupSet[k] = path.Join(cgroupRootDir, k, col[2])
				}
			}
		}
	}
	return &cgroup{cgroupSet: cgroupSet}, nil
}
```

下面几个方法都是获取进程对应的 cgroup 子系统的参数的（读取文件内容并转换类型）:
-	`CPUCFSQuotaUs`：获取 `cpu.cfs_quota_us`
-	`CPUCFSPeriodUs`：获取 `cpu.cfs_period_us`
-	`CPUAcctUsage`：获取 `cpuacct.usage"`
-	`CPUAcctUsagePerCPU`：获取 `cpuacct.usage_percpu`

```golang
// CPUCFSQuotaUs cpu.cfs_quota_us
func (c *cgroup) CPUCFSQuotaUs() (int64, error) {
	data, err := readFile(path.Join(c.cgroupSet["cpu"], "cpu.cfs_quota_us"))
	if err != nil {
		return 0, err
	}
	return strconv.ParseInt(data, 10, 64)
}

// CPUCFSPeriodUs cpu.cfs_period_us
func (c *cgroup) CPUCFSPeriodUs() (uint64, error) {
	data, err := readFile(path.Join(c.cgroupSet["cpu"], "cpu.cfs_period_us"))
	if err != nil {
		return 0, err
	}
	return parseUint(data)
}

// CPUAcctUsage cpuacct.usage
func (c *cgroup) CPUAcctUsage() (uint64, error) {
	data, err := readFile(path.Join(c.cgroupSet["cpuacct"], "cpuacct.usage"))
	if err != nil {
		return 0, err
	}
	return parseUint(data)
}

// CPUAcctUsagePerCPU cpuacct.usage_percpu
func (c *cgroup) CPUAcctUsagePerCPU() ([]uint64, error) {
	data, err := readFile(path.Join(c.cgroupSet["cpuacct"], "cpuacct.usage_percpu"))
	if err != nil {
		return nil, err
	}
	var usage []uint64
	for _, v := range strings.Fields(string(data)) {
		var u uint64
		if u, err = parseUint(v); err != nil {
			return nil, err
		}
		// fix possible_cpu:https://www.ibm.com/support/knowledgecenter/en/linuxonibm/com.ibm.linux.z.lgdd/lgdd_r_posscpusparm.html
		if u != 0 {
			usage = append(usage, u)
		}
	}
	return usage, nil
}
```

`CPUSetCPUs` 方法是用来解析 `cpuset.cpus` 这个文件的：
```bash
[root@VM_6_254_centos /sys/fs/cgroup/cpuset/docker/c9f540740293523b34e9e6699e9383591751a908adb172092b45e02db0bb42a9]# cat cpuset.cpus
0-7
```

```golang
// CPUSetCPUs cpuset.cpus
func (c *cgroup) CPUSetCPUs() ([]uint64, error) {
	data, err := readFile(path.Join(c.cgroupSet["cpuset"], "cpuset.cpus"))
	if err != nil {
		return nil, err
	}
	cpus, err := ParseUintList(data)
	if err != nil {
		return nil, err
	}
	var sets []uint64
	for k := range cpus {
		sets = append(sets, uint64(k))
	}
	return sets, nil
}
```

####	Cgroup 场景兼容
Kratos 中也提供了 Cgroup 场景下的 [CPU 信息获取](https://github.com/go-kratos/kratos/blob/master/pkg/stat/sys/cpu/cgroupCPU.go)，需要对 Cgroup 知识有个比较全面的认识才容易理解。


`cgroupCPU` 结构：
```golang
type cgroupCPU struct {
	frequency uint64
	quota     float64
	cores     uint64

	preSystem uint64
	preTotal  uint64
	usage     uint64
}
```
首先看一些独立的方法：

-	`cpuFreq` 方法：获取到第一个 CPU，通常是 `cpu0` 的主频信息（原始数据：`cpu MHz: 2399.988`）
```golang
func cpuFreq() uint64 {
	lines, err := readLines("/proc/cpuinfo")
	if err != nil {
		return 0
	}
	for _, line := range lines {
		fields := strings.Split(line, ":")
		if len(fields) < 2 {
			continue
		}
		key := strings.TrimSpace(fields[0])
		value := strings.TrimSpace(fields[1])
		if key == "cpu MHz" || key == "clock" {
			// treat this as the fallback value, thus we ignore error
			if t, err := strconv.ParseFloat(strings.Replace(value, "MHz", "", 1), 64); err == nil {
				return uint64(t * 1000.0 * 1000.0)
			}
		}
	}
	return 0
}
```


-	`cpuMaxFreq`：获取的最大频率（cpuinfo_max_freq）
```golang
func cpuMaxFreq() uint64 {
	feq := cpuFreq()
	data, err := readFile("/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq")
	if err != nil {
		return feq
	}
	// override the max freq from /proc/cpuinfo
	cfeq, err := parseUint(data)
	if err == nil {
		feq = cfeq
	}
	return feq
}
```


####	外部接口
[cpu.go](https://github.com/go-kratos/kratos/blob/master/pkg/stat/sys/cpu/cpu.go) 是一个非常典型的 golang 式 package 实现：
1.	定义全局私有变量 `stats CPU` 和 `usage uint64`、公共接口 `struct Stat` 和 `struct  Info` 及相应的存取方法 `ReadStat()` 及 `GetInfo()` 给外部调用（取值）
2.	在包初始化函数 init 中开启子 goroutine 来完成采集工作，采集的数据存储的全局变量中，可以使用 `atomic` 包来实现并发安全
3.	抽象 CPU 这个 interface{}
3.	外部引用此包的文件或程序

全局变量的部分：
```golang
var (
	stats CPU
	usage uint64
)

// ReadStat read cpu stat.
func ReadStat(stat *Stat) {
	stat.Usage = atomic.LoadUint64(&usage)
}

// GetInfo get cpu info.
func GetInfo() Info {
	return stats.Info()
}
```

抽象 CPU 这个 `interface{}`，具体的方法由 cgroup 或者普通的 cvm 方式去实现：
```golang
// CPU is cpu stat usage.
type CPU interface {
	Usage() (u uint64, e error)
	Info() Info
}
```

比如 `init` 方法，先试探当前环境是否为容器或者普通 CVM
```golang
func init() {
	var (
		err error
	)
	// 试探当前运行环境是否为容器
	stats, err = newCgroupCPU()
	if err != nil {
		// fmt.Printf("cgroup cpu init failed(%v),switch to psutil cpu\n", err)
		stats, err = newPsutilCPU(interval)
		if err != nil {
			panic(fmt.Sprintf("cgroup cpu init failed!err:=%v", err))
		}
	}

	// 开启子 routine 采集 CPU 的数据
	go func() {
		ticker := time.NewTicker(interval)
		defer ticker.Stop()
		for {
			// 定时器阻塞，不用 case 的方法
			<-ticker.C
			u, err := stats.Usage()
			if err == nil && u != 0 {
				atomic.StoreUint64(&usage, u)
			}
		}
	}()
}

```

最后 `Stat` 和 `Info` 这两个接口是给外部调用的：

```golang
// Stat cpu stat.
type Stat struct {
	Usage uint64 // cpu use ratio.
}

// Info cpu info.
type Info struct {
	Frequency uint64
	Quota     float64
}
```

##	0x05	使用 CPU 库的例子
在 Warden 的 gRPC Server 封装中，调用了此方法来获取服务端的 CPU 采样数据，具体 [代码如下](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/stats.go#L17):

1.	先定义 `Stat` 结构
2.	调用 `ReadStat` 方法获取值

```golang
var cpustat cpu.Stat
cpu.ReadStat(&cpustat)
if cpustat.Usage != 0 {
	trailer := gmd.Pairs([]string{nmd.CPUUsage, strconv.FormatInt(int64(cpustat.Usage), 10)}...)
	grpc.SetTrailer(ctx, trailer)
}
```



##  0x06    参考
-   [Linux Performance](http://www.brendangregg.com/linuxperf.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权