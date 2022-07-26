---
layout:     post
title:      踩坑与优化：Golang内存的那些坑（二）
subtitle:   golang Debug实战
date:       2021-07-18
author:     pandaychen
catalog:    true
tags:
    - Golang
---

##  0x00    前言
本文小结下，笔者遇到的那些内存泄漏的问题已经解决方案。

##  0x01    内存泄漏的问题

####    case1：业务高峰期刚过，CPU下降至正常，内存持续未释放
项目遇到一个问题，项目运行配置为：

1、宿主机内核linux4.5之后的版本（CVM一样）
2、go版本在go`1.13`


不管节点内存配置是多少，在容器上的表现是内存一直升高接近`90%`，并且回落时间很慢，大概`1`分钟才减少`0.01%`，这让用户没法判断服务所需要的内存到底是多少。还有可能在突然出现需要大量内存而旧内存没来得及回收的时候导致程序OOM。

猜测原因是go的垃圾回收有问题，没有把占用的内存还给os？排查了下原因是Linux的特性MADV。从go`1.12`版本开始，go释放内存开始使用Linux的`MADV_FREE`特性，runtime 在释放内存时，使用了更加高效的 `MADV_FREE` 而不是之前的 `MADV_DONTNEED`
这样带来的好处是，一次 GC 后的内存分配延迟得以改善，runtime 也会更加积极地将释放的内存归还给操作系统，以应对大块内存分配无法重用已存在的堆空间的问题。不过也会带来一个副作用：`RSS` 不会立刻下降，而是要等到系统有内存压力了，才会延迟下降。参见[runtime: use MADV_FREE on linux as well](https://github.com/golang/go/issues/23687)。

![MADV_FREE]()

类似问题：
-   [runtime: RSS keeps on increasing, suspected scavenger issue #36398](https://github.com/golang/go/issues/36398)
-   [all: resident set size keeps on increasing, but no memory leaks #38257](https://github.com/golang/go/issues/38257)

即`MADV_FREE` 和 `MADV_DONTNEED` 的区别：
-   `MADV_DONTNEED`：会直接释放内存页给到内核（`RSS`下降会很快）
-   `MADV_FREE`：只是将内存标记为可回收，内核并不会马上回收这块内存，只有当内核需要更多内存时，这块内存才会被内核回收（按官方是效率较高，但`RSS`值不会立即下降，只有发现内存不够时才会返回给操作系统）

需要注意的是，`MADV_FREE` 需要 Linux`4.5` 以及以上内核，否则 runtime 会继续使用原先的 `MADV_DONTNEED` 方式。在Go `1.11`以及之前，默认使用的都是`MADV_DONTNEED`，在Go `1.12` ~ `1.15` 用的是`MADV_FREE`，不过官方在`1.16`又改回了默认`MADV_DONATED`。所以，再遇到此类问题时，首先需要排查go的版本以及开启的内存回收方式。`MADV_FREE` 会让gc执行更加高效，但是这却导致了服务进程的`RSS`一直居高不下，而且由于服务部署在容器环境，宿主机的内存足够用时是不会主动回收容器内go进程释放的内存的，而容器服务则很大可能由于cgroup的内存资源限制，导致进程被kill掉

######  释放内存的源码
1、`1.12`版本之前，runtime释放内存<br>
```
madvise(v, n, _MADV_DONTNEED)
```

2、`1.12`-`1.15`<br>
```golang
var advise uint32
if debug.madvdontneed != 0 {
    advise = _MADV_DONTNEED
} else {
    advise = atomic.Load(&adviseUnused)
}
if errno := madvise(v, n, int32(advise)); advise == _MADV_FREE && errno != 0 {
    // MADV_FREE was added in Linux 4.5. Fall back to MADV_DONTNEED if it is
    // not supported.
    atomic.Store(&adviseUnused, _MADV_DONTNEED)
    madvise(v, n, _MADV_DONTNEED)
}
```

3、`1.16`<br>
默认把 `debug.madvdontneed` 设置为`1`了，默认是采用 `MADV_DONTNEED` 进行内存释放了
```golang
func parsedebugvars() {
    // defaults
    debug.cgocheck = 1
    debug.invalidptr = 1
    if GOOS == "linux" {
        debug.madvdontneed = 1
    }
   //...
}
```

###### 问题解决
-   容器环境：增加运行时的`export GODEBUG=madvdontneed=1`环境变量，或者改使用Go 1.16后的版本。
-   CVM环境：运行命令前面增加`GODEBUG=madvdontneed=1 xxxxxx`


####    case2：不合理的defer导致内存泄漏
问题描述，内存使用率呈锯齿状，定期会被OOM。使用pprof 持续观察几个小时，画出调用流程图如下：

![debug-case2-1]()

![debug-case2-2]()

通过`list`定位到问题代码片段如下（经典BUG：在`for` 循环中加`defer`），问题的原因是只有在函数执行完毕后，这些`defer`才会执行，而不是`for`循环每次循环结束就执行：

```golang
func kafkaConsumer(){
    //...
    for {
        stPollCtx, funcCancel := context.WithTimeout(context.Background(),h.kafkaConfig.MaxWait*time.Millisecond)
        defer funcCancel()
        //...
    }
    //...
}
```


解决的方法也很简单，去掉`defer`，改成再最后调用，或者采用如下方法：
```golang
func kafkaConsumer(){
    //...
    for {
        func(){
            stPollCtx, funcCancel := context.WithTimeout(context.Background(),h.kafkaConfig.MaxWait*time.Millisecond)
            defer funcCancel()
            //...
        }()
    }
}
```

##  0x04  参考
-   [Go 1.12 关于内存释放的一个改进](https://ms2008.github.io/2019/06/30/golang-madvfree/)
-   [【实践】使用 Go pprof 做内存性能分析](https://cloud.tencent.com/developer/article/1489186)
-   [内存泄漏（增长）火焰图](https://heapdump.cn/article/1661654)
-   [go tool pprof](https://github.com/hyper0x/go_command_tutorial/blob/master/0.12.md)
-   [Hi, 使用多年的go pprof检查内存泄漏的方法居然是错的?!](https://colobu.com/2019/08/20/use-pprof-to-compare-go-memory-usage/)
-   [linux中top命令 VSS,RSS,PSS,USS 四个内存字段的解读](https://blog.csdn.net/qq_31588719/article/details/89476050)