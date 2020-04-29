---
layout:     post
title:      GOMAXPROCS 的 “坑”
subtitle:	容器环境中使用runtime.GOMAXPROCS()需谨慎
date:       2020-02-28
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - GOMAXPROCS
---

##  0x00    概念
几个名词：
-   CPU core：CPU 逻辑核
-   CPU socket：CPU 插槽
-	NUMA架构：


通过 `cat /proc/cpuinfo` 可以查看机器的 CPU 详细信息，比如，笔者的机器是单 CPU、8 逻辑核
![image](https://s2.ax1x.com/2020/02/28/3rf3tS.png)

```bash
# 总核数 = 物理 CPU 个数 * 每颗物理 CPU 的核数 
# 总逻辑 CPU 数 = 物理 CPU 个数 * 每颗物理 CPU 的核数 * 超线程数

# 查看物理 CPU 个数
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l

# 查看每个物理 CPU 中 core 的个数(即核数)
cat /proc/cpuinfo| grep "cpu cores"| uniq

# 查看逻辑 CPU 的个数
cat /proc/cpuinfo| grep "processor"| wc -l
```

##  0x01    CPU Affinity
&emsp;&emsp; 熟系 Linux 后台开发的朋友都知道 CPU 亲和性这个概念（CPU affinity）。CPU affinity 是一种调度属性， 它可以将一个进程 "绑定" 到一个或一组 CPU 上. 在 SMP(Symmetric Multi-Processing 对称多处理)架构下，Linux 调度器 (scheduler) 会根据 CPU affinity 的设置让指定的进程运行在 "绑定" 的 CPU 上, 而不会在别的 CPU 上运行. CPU 亲和性（affinity） 就是进程要在某个给定的 CPU 上尽量长时间地运行而不被迁移到其他处理器的倾向性。Linux 内核进程调度器天生就具有被称为 软 CPU 亲和性（affinity） 的特性，这意味着进程通常不会在处理器之间频繁迁移。这种状态正是我们希望的，因为进程迁移的频率小就意味着产生的负载小。

Linux 调度器同样支持自然 CPU 亲和性(natural CPU affinity): 调度器会试图保持进程在相同的 CPU 上运行, 这意味着进程通常不会在处理器之间频繁迁移, 进程迁移的频率小就意味着产生的负载小。

因为程序的作者比调度器更了解程序, 所以我们可以手动地为其分配 CPU 核，而不会过多地占用 CPU0，或是让我们关键进程和一堆别的进程挤在一起, 所有设置 CPU 亲和性可以使某些程序提高性能。

##  0x02    再看 Go 调度器: M-P-G 模型
&emsp;&emsp; 我们知道在 Go scheduler 中，G 代表 goroutine, P 代表 Logical Processor, M 是操作系统线程。在绝大多数时候，其实 P 的数量和 M 的数量是相等。 每创建一个 p, 就会创建一个对应的 M。
![image](https://s2.ax1x.com/2020/02/29/3sNOyQ.jpg)
golang runtime 是有个 sysmon 的协程，他会轮询的检测所有的 P 上下文队列，只要 G-M 的线程长时间在阻塞状态，那么就重新创建一个线程去从 runtime P 队列里获取任务。先前的阻塞的线程会被游离出去了，当他完成阻塞操作后会触发相关的 callback 回调，并加入回线程组里。简单说，如果你没有特意配置 runtime.SetMaxThreads，那么在可没有复用的线程下会一直创建新线程。

##  0x03    GOMAXPROCS 的优雅应用

在 go 服务端开发中，要实现优雅对 `net.TCPListener`+Affinity 使用，服务器端实现一般包含如下代码：
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

可以通过 `runtime.GOMAXPROCS()` 来设定 `P` 的值，当前 Go 版本的 GOMAXPROCS 默认值已经设置为 CPU 的（逻辑核）核数， 这允许我们的 Go 程序充分使用机器的每一个 CPU, 最大程度的提高我们程序的并发性能。不过从实践经验中来看，IO 密集型的应用，可以稍微调高 `P` 的个数；而本文讨论的 Affinity 更适合 CPU 密集型的应用。


####    物理机 && 虚拟机
在物理机及一般的虚拟机中，`runtime.GOMAXPROCS()` 的值就是 CPU 的逻辑核数。比如在笔者自己的虚拟机上（开始的图），`runtime.GOMAXPROCS()` 获取的值就是 `8`。


####  Docker-Container
在 Docker-container 中，`runtime.GOMAXPROCS()` 获取的是 ** 宿主机 ** 的 CPU 核数。`P` 值设置过大，导致生成线程过多，会增加上线文切换的负担，导致严重的上下文切换，浪费 CPU。
所以，在 Docker-container 中, Golang 设置的 GOMAXPROCS 并不准确。

####     Kubernetes-Pod
同样的，在 kubernetes 集群中，如果采用如此设置，会导致 Node（宿主机）中的线程数过多。在笔者的测试集群中，集群中有 3 个 Node，总核数约 36 核。
```bash
CPU: 8.95/35.97 核
内存: 18.56/63.95GB
```

创建的 POD 参数中，限制 POD 的 cpu 核数是 1（limits），采用了 GOMAXPROCS 设置后，发现容器中的 golang 线程数量超过 36，集群中的线程总数也远超过预期。
```yaml
 resources:
          limits:
            cpu: "1"
            memory: 6Gi
          requests:
            cpu: 500m
            memory: 1Gi
```

总结下，在 Docker-container 和 kubernetes 集群中，存在 GOMAXPROCS 会错误识别容器 cpu 核心数的问题。此外，
在 Kubernetes 集群中，为每个应用 Pod 分配的 CPU 及 CPU limits 不一定相同，所以通过配置指定 GOMAXPROCS 线程数来匹配 CPU 核心个数的方法，不太靠谱，同时这种 Fixed 的方式也与 Kubernetes 的（自动）扩缩容理念不符。

##	0x05	解决
如何解决 golang 程序自适应容器 CPU 核心数的问题？在 Google 上搜索，找到了 Uber 的这个库[automaxprocs](https://github.com/uber-go/automaxprocs)，大致原理是读取 CGroup 值识别容器的 CPU quota，计算得到实际核心数，并自动设置 GOMAXPROCS 线程数量。有兴趣的同学可以阅读其源码，加深理解。它的使用方式也是非常简单：
```go
import _ "go.uber.org/automaxprocs"

func main() {
  // Your application logic here.
}
```


##  0X06	参考
-   [管理处理器的亲和性（affinity）](https://www.ibm.com/developerworks/cn/linux/l-affinity.html)
-   [[译]Go 调度器: M, P 和 G](https://colobu.com/2017/05/04/go-scheduler/)