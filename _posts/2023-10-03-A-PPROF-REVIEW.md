---
layout:     post
title:      Pprof 调试经验汇总
subtitle:
date:       2023-10-03
author:     pandaychen
catalog:    true
tags:
    - Pprof
---


##  0x00    前言

##  0x01    DEBUG 汇总

####    调用图

1.  安装 `yum install graphviz`
2.  `go tool pprof main http://localhost:8000/debug/pprof/heap` 或者 goroutine，进入命令行
3.  输入 `svg` 命令即可保存

####    火焰图

1.  安装 `FlameGraph`，如下
2.  安装 `go-torch`，拿到 `bin` 文件
3.  使用命令 `go-torch -u http://localhost:6060 -t 30` 生成火焰图

```bash
git clone https://github.com/brendangregg/FlameGraph.git
cp FlameGraph/flamegraph.pl /usr/local/bin
```

##  0x02    一些经验之谈

####    关于调用图
一般调用图中高亮的部分就能告诉开发者是哪里发生了内存泄漏了（除非内存是预设的 size，如代码中本地初始化一块缓存）

####    部署与编译机器分开
比如，需要调试 goroutine，部署机器上通过 `curl http://localhost:6060/debug/pprof/goroutine -o goroutine.pprof` 保存运行时配置，然后将 `goroutine.pprof` 传输到编译机器上，通过 `go tool pprof goroutine.pprof` 即可进行调试


####    对比前后时间差异
导出时间点 A 的文件：

```bash
curl http://localhost:6060/debug/pprof/heap > heap.timeA
curl http://localhost1:6060/debug/pprof/goroutine> goroutine.timeA
```

导出时间点 B 的文件：

```BASH
curl http://localhost:6060/debug/pprof/heap > heap.timeB
curl http://localhost1:6060/debug/pprof/goroutine> goroutine.timeB
```
对比两个时间点的堆栈差异：

```BASH
go tool pprof --base heap.timeA heap.timeB
go tool pprof --http :8080 --base heap.timeA heap.timeB
```

####    调试goroutine profile的方法
```BASH
# 下载cpu profile，默认从当前开始收集30s的cpu使用情况，需要等待30s
go tool pprof http://localhost:6060/debug/pprof/profile   # 30-second CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=120     # wait 120s

# 下载heap profile
go tool pprof http://localhost:6060/debug/pprof/heap      # heap profile

# 下载goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine # goroutine profile

# 下载block profile
go tool pprof http://localhost:6060/debug/pprof/block     # goroutine blocking profile

# 下载mutex profile
go tool pprof http://localhost:6060/debug/pprof/mutex
```

对应采集到的pprof文件，最常用的就是下面三个命令：

-   `top`：显示正运行到某个函数goroutine的数量
-   `traces`：显示所有goroutine的调用栈
-   `list`：列出代码详细的信息


####    memory快照
现网中排查内存泄漏的问题，可以使用如下脚本来采集`heap`视图，并利用`go tool pprof`对内存差异较大的时间点（**需要结合监控图标了解到内存有明显泄漏前后的时间点**）进行分析比较：

```BASH
#!/bin/bash
DURATION=1200  # 运行时间： 1200秒
INTERVAL=10    # 采样间隔 
END_TIME=$((SECONDS + DURATION))

while [ $SECONDS -lt $END_TIME ]; do
  TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
  curl -s "http://localhost:6666/debug/pprof/heap" > "heap_${TIMESTAMP}.pprof"
  sleep $INTERVAL
done
```

####    traces命令
现网中，基于`traces`命令可以协助排查并定位goroutine泄漏问题

```BASH
curl http://localhost:6060/debug/pprof/goroutine> goroutine.timeH
go tool --pprof goroutine.timeH
#执行traces
```

参考下面的例子，结合项目分析，排名前面的goroutine数量是不正常的，存在goroutine泄漏问题：

![traces-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/pprof/pprof-traces-example1.png)

####    `runtime.gopark` 方法的启示
若查看 goroutine 文件使用 `traces` 命令，发现大量的 `runtime.gopark` 函数调用，那么可以初步认定为程序存在 goroutine 泄漏。`gopark` 函数主要作用就是将当前的 goroutine 放入等待状态，这就意味着 goroutine 被暂时被搁置，也就是被运行时调度器暂停了

-   调用 `acquirem` 函数
-   获取当前 goroutine 所绑定的 `m`，设置各类所需数据
-   调用 `releasem` 函数将当前 goroutine 和其 `m` 的绑定关系解除
-   调用 `park_m` 函数
-   将当前 goroutine 的状态从 `_Grunning` 切换为 `_Gwaiting` ，也就是等待状态
-   删除 `m` 和当前 goroutine `m->curg`（简称 `gp`）之间的关联
-   调用 `mcall` 函数，仅会在需要进行 goroutiine 切换时会被调用
-   切换当前线程的堆栈，从 `g` 的堆栈切换到 `g0`的堆栈并调用 `fn(g)` 函数
-   将 `g` 的当前 PC/SP 保存在 `g->sched` 中，以便后续调用 `goready` 函数时可以恢复运行现场

##  0x03    参考
-   [实战Go内存泄露](https://lessisbetter.site/2019/05/18/go-goroutine-leak/)
-   [万字长文讲解Golang pprof 的使用](https://www.cnblogs.com/hobbybear/p/18059425)