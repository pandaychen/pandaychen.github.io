---
layout:     post
title:      GOMAXPROCS 的坑
subtitle:	容器环境中使用 runtime.GOMAXPROCS 需谨慎
date:       2020-02-28
author:     pandaychen
header-img: img/golang-tools-fun.png
catalog: true
category:   false
tags:
    - GOMAXPROCS
    - GOLANG
---

##  0x00    前言
自 Go 1.5 开始， Go 的 GOMAXPROCS 默认值已经设置为 CPU 的核数， 这允许我们的 Golang 程序充分使用机器的每一个 CPU, 最大程度的提高我们程序的并发性能。

##  0x01    CPU Affinity
&emsp;&emsp; 熟系 Linux 后台开发的朋友都知道 CPU 亲和性（CPU Affinity）。CPU Affinity 是一种调度属性，它可以将单个进程绑定到一个或一组 CPU 上。<br>
在 SMP（Symmetric Multi-Processing 对称多处理）架构下，Linux 调度器（Scheduler）会根据 CPU affinity 的设置让指定的进程运行在绑定的 CPU 上，而不会在别的 CPU 上运行。 CPU Affinity 就是进程要在某个给定的 CPU 上尽量长时间地运行而不被迁移到其他处理器的倾向性。Linux 内核进程调度器天生就具有被称为软 CPU Affinity 的特性，这意味着进程通常不会在处理器之间频繁迁移。合理的设置 CPU Affinity（进程独占 CPU Core）可以提高程序处理性能。

##  0x02    再看 M-P-G 模型
&emsp;&emsp;Golang 的调度器模型：经典的 M-P-G 模型，在 Go Scheduler 模型中：
-	`G` 代表 goroutine，即用户创建的 goroutines
-	`P` 代表 Logical Processor，是类似于 CPU 核心的概念，其用来控制并发的 M 数量
-	`M` 是操作系统线程。在绝大多数时候，`P` 的数量和 `M` 的数量是相等的。每创建一个 `P`, 就会创建一个对应的 `M`

![image](https://s2.ax1x.com/2020/02/29/3sNOyQ.jpg)

当 `M` 需要执行 `G` 的时候，它需要寻找到一个空闲的 `P`，只有跟一个 `P` 绑定后，`M` 才能被执行。通过这样的方式，Go Scheduler 保证了在同一时间内，最多只有 `P` 个系统线程在真正地执行。`P` 的数量在默认情况下，会被设定为 CPU 的数量。而 `M` 虽然需要跟 `P` 绑定执行，但数量上并不与 `P` 相等。这是因为 `M` 会因为系统调用或者其他事情被阻塞，因此随着程序的执行，`M` 的数量可能增长，而 `P` 在没有用户干预的情况下，则会保持不变。

Golang 的 Runtime 包中获取和设置 GOMAXPROCS 的 [代码如下](https://golang.org/src/runtime/os_linux.go?s=1936:1961#L62)，也就是 Go Scheduler 确定 `P` 数量的逻辑。在 Linux 上，它会利用系统调用 `sched_getaffinity` 来获得系统的 CPU 核数。
```golang
// The bootstrap sequence is:
//
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
//
// The new G calls runtime·main.
func schedinit() {
    //......
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
    }
    //......
}

// runtime/os_linux.go
func osinit() {
	ncpu = getproccount()
}

// runtime/os_linux.go
func getproccount() int32 {
	// This buffer is huge (8 kB) but we are on the system stack
	// and there should be plenty of space (64 kB).
	// Also this is a leaf, so we're not holding up the memory for long.
	// See golang.org/issue/11823.
	// The suggested behavior here is to keep trying with ever-larger
	// buffers, but we don't have a dynamic memory allocator at the
	// moment, so that's a bit tricky and seems like overkill.
	const maxCPUs = 64 * 1024
	var buf [maxCPUs / 8]byte
	r := sched_getaffinity(0, unsafe.Sizeof(buf), &buf[0])
	if r < 0 {
		return 1
	}
	n := int32(0)
	for _, v := range buf[:r] {
		for v != 0 {
			n += int32(v & 1)
			v >>= 1
		}
	}
	if n == 0 {
		n = 1
	}
	return n
}
```

##  0x03    GOMAXPROCS 的优雅应用
在 Golang 的服务端开发中，要实现优雅对 `net.TCPListener` 使用 Affinity，服务器端实现一般包含如下代码：
```go
// 每个 core 的 accept 逻辑
func acceptTCP(server *Server, lis *net.TCPListener) {
	var (
		conn *net.TCPConn
		err  error
		r    int
	)
	for {
		if conn, err = lis.AcceptTCP(); err != nil {
			log.Errorf("listener.Accept(\"%s\") error(%v)", lis.Addr().String(), err)
			return
		}
		if err = conn.SetKeepAlive(server.c.TCP.KeepAlive); err != nil {
			log.Errorf("conn.SetKeepAlive() error(%v)", err)
			return
		}
		if err = conn.SetReadBuffer(server.c.TCP.Rcvbuf); err != nil {
			log.Errorf("conn.SetReadBuffer() error(%v)", err)
			return
		}
		if err = conn.SetWriteBuffer(server.c.TCP.Sndbuf); err != nil {
			log.Errorf("conn.SetWriteBuffer() error(%v)", err)
			return
        }

        // 处理每个 tcp 连接
		go serveTCP(server, conn, r)
		if r++; r == maxInt {
			r = 0
		}
	}
}

func InitTCPServer(server *Server, addr string, accept_maxcore int) (err error) {
	var (
		bind     string
		listener *net.TCPListener
		addr     *net.TCPAddr
	)
    if addr, err = net.ResolveTCPAddr("tcp", bind); err != nil {
        log.Errorf("net.ResolveTCPAddr(tcp, %s) error(%v)", bind, err)
        return
    }
    if listener, err = net.ListenTCP("tcp", addr); err != nil {
        log.Errorf("net.ListenTCP(tcp, %s) error(%v)", bind, err)
        return
    }
    log.Infof("start tcp listen: %s", bind)
    // split N core accept（每个逻辑核都单独实现 accept）
    for i := 0; i < accept_maxcore; i++ {
        go acceptTCP(server, listener)
    }
	return
}

func main(){
    runtime.GOMAXPROCS(runtime.NumCPU())
    ...
    InitTCPServer(srv, conf.TCP.Bind, runtime.NumCPU())
    ...
}
```

##  0x04    GOMAXPROCS 及取值
可以通过 `runtime.GOMAXPROCS()` 来设定 `P` 的值，当前 Go 版本的 `GOMAXPROCS` 默认值已经设置为 CPU 的（逻辑核）核数， 这允许我们的 Go 程序充分使用机器的每一个 CPU, 最大程度的提高我们程序的并发性能。不过从实践经验中来看，IO 密集型的应用，可以稍微调高 `P` 的个数；而本文讨论的 Affinity 设置更适合 CPU 密集型的应用。

####    物理机 && 虚拟机
在物理机及一般的 CVM 中，`runtime.GOMAXPROCS()` 的值就是 CPU 的逻辑核数。比如在笔者的机器上，`runtime.GOMAXPROCS()` 获取的值就是 `8`
![image](https://s2.ax1x.com/2020/02/28/3rf3tS.png)

####  	Docker-Container
在 Docker-container 中，`runtime.GOMAXPROCS()` 获取的是 <font color="#dd0000"> 宿主机的 CPU 核数 </font>。`P` 值设置过大，导致生成线程过多，会增加上线文切换的负担，导致严重的上下文切换，浪费 CPU。
所以，在 Docker-container 中, Golang 设置的 GOMAXPROCS 并不准确。

####    Kubernetes
Kubernetes Pod 中的结果同 Docker，在 Kubernetes 集群中，如果采用如此设置，会导致 Node（宿主机）中的线程数过多。在笔者的 Kubernetes 集群中，有 3 个 Node 节点，总核数约 36 核：
```bash
CPU: 8.95/35.97 核
内存: 18.56/63.95GB
```

创建的 Pod 参数中，限制 Pod 的 CPU 核数是 1（limits），采用了 `GOMAXPROCS` 设置后，发现 Pod 容器中的线程数量超过 36，集群中的线程总数也远超过预期。
```yaml
resources:
	limits:
		cpu: "1"
		memory: 6Gi
	requests:
		cpu: 500m
		memory: 1Gi
```

小结下，在 Docker-container 和 Kubernetes 集群中，存在 GOMAXPROCS 会错误识别容器 cpu 核心数的问题。此外，在 Kubernetes 集群中，为每个应用 Pod 分配的 CPU 及 CPU limits 不一定相同，所以通过配置指定 `GOMAXPROCS` 线程数来匹配 CPU 核心个数的方法，不太靠谱，同时这种 Fixed 的方式也与 Kubernetes 的（自动）扩缩容理念不符。

##	0x05	解决
如何解决 golang 程序自适应容器 CPU 核心数的问题？在 Google 上搜索，找到了 Uber 的这个库 [automaxprocs](https://github.com/uber-go/automaxprocs)，大致原理是读取 `CGroup` 值识别容器的 CPU quota，计算得到实际核心数，并自动设置 `GOMAXPROCS` 线程数量。有兴趣的同学可以阅读其源码，加深理解。它的使用方式也是非常简单（只需要导入库即可，这就是 Golang 的魔力）：
```go
import _ "go.uber.org/automaxprocs"

func main() {
  // Your application logic here
}
```

##  0X06	参考
-	[详尽干货！从源码角度看 Golang 的调度（上）](https://www.infoq.cn/article/r6wzs7bvq2er9kuelbqb)
-   [管理处理器的亲和性（affinity）](https://www.ibm.com/developerworks/cn/linux/l-affinity.html)
-   [Go 调度器: M, P 和 G](https://colobu.com/2017/05/04/go-scheduler/)
-	[GOMAXPROCS 需要设置吗？](https://colobu.com/2017/10/11/interesting-things-about-GOMAXPROCS/)
-	[GOMAXPROCS 与容器的相处之道](https://gaocegege.com/Blog/maxprocs-cpu)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
