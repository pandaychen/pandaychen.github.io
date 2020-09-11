---
layout:     post
title:      Golang 并发协程池实现分析（二）
subtitle:   分析 Fasthttp 的 goroutine pool
date:       2020-10-09
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - Fasthttp
    - Golang
---

##  0x00    前言
> &nbsp;&nbsp;&nbsp;&nbsp;[FastHTTP](https://github.com/valyala/fasthttp/blob/master/workerpool.go) 开源项目给出了一个性能极佳协程池的实现。<br>

标准库 `net.http` 和 `fasthttp` 最大的不同可能就是 server 在处理连接的时候使用了协程池。在并发量大的时候，goroutine 数量巨大，runtime 层的上下文切换成本对性能有影响。而 fasthttp 用协程池规避了这个问题。fasthttp 并不是 `One request One goroutine`，而是实现了一个非常 Nice 的 workerPool，尽可能复用 goroutine。

## 0x01 fastHTTP 协程池简介
阅读完整个 fasthttp 的实现代码，只能用精妙二字来形容，可以作为 golang channel 应用案例的典范。这里先给出我对 fasthttp 协程池的总结：

> fasthttp 的 pool 实现是通过对加锁 `slice` 的增 / 减操作来进行并发控制的，`slice` 的最大长度即为并发值。
> fasthttp 按需创建 goroutine，通过 `workerChan` 接收要处理的 `net.Conn`，处理完成将 `workerChan` 放回 `slice`，这样下一个请求又可以获取到... 继续循环处理

##  0x02    fasthttp VS net.http
下面是标准库的 `http.Server` 实现，对每个新客户端连接（请求）都开启一个单独的 goroutine 来处理：
```golang
net/http/server.go
func (srv *Server) Serve(l net.Listener) error {
    ......
    for {
      rw, e := l.Accept()
      ......
      //FastHTTP 在这步使用协程池
      go c.serve(ctx)
    }
}
```

而 `FastHTTP` 的 `fasthttp.ListenAndServe` 则启用了 `pool` 机制：
```golang
github.com/valyala/fasthttp/server.go 1489
func (s *Server) Serve(ln net.Listener) error {
    // default concurrency set to 256*1024
    maxWorkersCount := s.getConcurrency()
    s.concurrencyCh = make(chan struct{}, maxWorkersCount)
    // 初始化 pool 结构
    wp := &workerPool{
        WorkerFunc:      s.serveConn,
        MaxWorkersCount: maxWorkersCount,
        LogAllErrors:    s.LogAllErrors,
        Logger:          s.logger(),
    }
    wp.Start()
    ......
    for {
      if c, err = acceptConn(s, ln, &lastPerIPErrorTime); err != nil {
          ......
      }
      // 对应 go 原生的 go c.serve(ctx)
      if !wp.Serve(c) {   //wp.Serve，使用 pool 的 Serve 方法来处理请求
          ......
      }
      ......
    }
}
```

&nbsp;&nbsp;&nbsp;&nbsp; 从上面两端代码可以明显看出，在 golang 原生 `http.Server` 包中，当接收到新请求就会启动一个协程处理，而 `FastHTTP` 则使用协程池处理。

接下来，我们来分析 fasthttp 中 pool 的具体实现。

##    0x03  核心结构
`workerPool` 是协程池的核心结构：
- `WorkerFunc`：[`s.serveConn 方法 `](https://github.com/valyala/fasthttp/blob/master/server.go#L1866)，即每个客户端连接 `net.conn` 的处理函数
- `MaxWorkersCount`：worker 池的最大数量
- `workerChanPool`：`sync.Pool` 对象池，用来复用堆上的结构，做优化所用
- `MaxIdleWorkerDuration`：worker 的最大空闲时间
- `ready  []*workerChan`：核心结构，是可用的 worker 列表，所有 goroutine worker 是存放在 slice 中的。pool 的 worker 协程是通过 ready 数组来代表的（见下面的代码分析）

注意两点：
1.  `ready`：这个数组模拟一个类似栈的 FILO 队列，也就是说我们每次使用的 worker 都从 `ready` 数组的尾部开始取
2.  `ready`：数组的成员是指针

```golang
type workerPool struct {
    WorkerFunc func(c net.Conn) error
    MaxWorkersCount int
    LogAllErrors bool
    MaxIdleWorkerDuration time.Duration
    Logger Logger

    lock         sync.Mutex
    workersCount int
    mustStop     bool
    ready []*workerChan
    stopCh chan struct{}
    workerChanPool sync.Pool
}
```

`workerChan` 和上一篇文章 [Golang 并发协程池实现分析（一）](https://pandaychen.github.io/2020/10/08/GOLANG-HANDLING-1-MILLION-REQUESTS-WITH-GOLANG/) 的 `WorkerPool  chan chan Job` 的作用很类似，都是用作并发控制：
```golang
type workerChan struct {
	lastUseTime time.Time
	ch          chan net.Conn
}
```

##  0x04  代码分析

####  pool 初始化
在 `Serve` 方法，进入 `accept` 之前，完成对 pool 的初始化工作，主要是注册了对 `net.Conn` 的处理方法，[`s.serveConn`](https://github.com/valyala/fasthttp/blob/master/server.go#L1959)：
```golang
wp := &workerPool{
        WorkerFunc:      s.serveConn,
        MaxWorkersCount: maxWorkersCount,
        LogAllErrors:    s.LogAllErrors,
        Logger:          s.logger(),
}
```

####  pool 的启动
在 `Serve` 方法，`Accept` 之前，通过 `wp.start` 开启了一个 goroutine，定时清理 workerpool 中未使用时间超过 `maxIdleWorkerDuration` 的 goroutine（很长时间无请求，回收 goroutine）：
```golang
func (wp *workerPool) Start() {
    wp.stopCh = make(chan struct{})
    stopCh := wp.stopCh
    go func() {
        var scratch []*workerChan
        for {
            wp.clean(&scratch)
            select {
            case <-stopCh:
                return
            default:
                time.Sleep(wp.getMaxIdleWorkerDuration())
            }
        }
    }()
}
```

####  Pool 的停止
当协程池停止运行时，回收资源，清空 ready 里所有 ch，并清空 ready。`ch.ch <- nil` 这会触发 `workerFunc` 的 worker 协程退出：
```golang
func (wp *workerPool) Stop() {
    close(wp.stopCh)
    wp.stopCh = nil
    wp.lock.Lock()
    ready := wp.ready
    for i, ch := range ready {
        ch.ch <- nil
        ready[i] = nil
    }
    wp.ready = ready[:0]
    wp.mustStop = true
    wp.lock.Unlock()
}
```

####  核心方法：Serve
[`Serve` 方法](https://github.com/valyala/fasthttp/blob/master/workerpool.go#L146) 的逻辑是：
1.  调用 `wp.getCh()` 拿到一个票据 `ch`，同时该票据也是 `net.Conn` 的传递通道
2.  将新的客户端连接放入 `ch`，触发 worker 工作协程对新连接的处理流程

```golang
func (wp *workerPool) Serve(c net.Conn) bool {
	ch := wp.getCh()
	if ch == nil {
		return false
	}
	ch.ch <- c
	return true
}
```

####  核心方法：getCh
`getCh` 的实现可以理解为一个用来执行 `workerFunc` 的 goroutine 都绑定了一个 `workerChan`。把要处理的 `conn` 发到这个 `workerChan`，这个 goroutine 就开始执行。没有要执行的 `conn` 则 goroutine 阻塞，直到下次 `workerChan` 有连接发来。

```golang
 func (wp *workerPool) getCh() *workerChan {
    var ch *workerChan
    createWorker := false

    wp.lock.Lock()
    ready := wp.ready
    n := len(ready) - 1
    if n < 0 {
        if wp.workersCount < wp.MaxWorkersCount {
            createWorker = true
            wp.workersCount++
        }
    } else {
        ch = ready[n]
        ready[n] = nil
        // 这里说明 ready 尾部的 workerChan 要被取出和 goroutine 绑定并使用了
        wp.ready = ready[:n]  //create 0-->n-1 a new slice
    }
    wp.lock.Unlock()

    if ch == nil {
        // 如果没有可用的 worker，包含两种情况：
        //1. 未达 worker 总数，需要新建
        //2. 超过 worker 总数，这时候返回失败（nil）
        if !createWorker {
            return nil
        }

        // 从对象池获取 chan net.Conn 结构，用于存储客户端连接的引用
        vch := wp.workerChanPool.Get()
        if vch == nil {
            vch = &workerChan{
                // 注意 workerChan 中可以定义为是个通用结构，目前是 net.Conn
                //ch          chan net.Conn
                ch: make(chan net.Conn, workerChanCap),
            }
        }
        ch = vch.(*workerChan)  // 类型转换，注意这里 ch 是指针
        go func() {
            // 注意：这里是真正处理业务逻辑的部分！
            wp.workerFunc(ch)
            wp.workerChanPool.Put(vch)
        }()
    }
    return ch
}
```

这里有个细节：
```golang
func main(){
        var ch *int
        var ready []*int
        ready=make([]*int,10)
        var a = 1
        ready[1]= &a
        ch = ready[1]
        ready[1] = nil
        fmt.Printf("%v,%v\n",ch,ready[1])
}
```

####  核心方法：workerFunc
上一步中，单独开启的 goroutine 中来完成两个事情：
1.  `wp.workerFunc(ch)`：阻塞接收 `ch *workerChan` 中的连接 `conn`
2.  `wp.workerChanPool.Put(vch)`：

worker 处理完一个连接 `conn` 后，通过 `wp.release()` 这个 `conn` 对应的票据到 `ready` 数组。即表示这时 worker 空闲，可以执行下一次请求。同时处理完 `net.Conn` 后要将 `conn` 置为 `nil`。在 `workerFunc` 的实现中，调用了业务注册的方法 `wp.WorkerFunc(c)`，注意前面是大写，来完成真正的业务逻辑，该方法对应于前面的 `s.serveConn` 方法。

此外，正常情况下 worker 是不退出的，除非 `wp.Stop`，这样也实现 pool 的最最核心的 goroutine 复用能力。
```golang
func (wp *workerPool) workerFunc(ch *workerChan) {
    var c net.Conn
    var err error
    // for range 在一个 channel 上
    for c = range ch.ch {
        if c == nil {
            // 注意这里，和 clean 方法息息相关
            break
        }
        if err = wp.WorkerFunc(c); err != nil && err != errHijacked {
            // 处理错误
        }
        c = nil

        // 将 ch *workerChan 放回 ready 数组
        if !wp.release(ch) {
            break
        }
    }
    wp.lock.Lock()
    wp.workersCount--
    wp.lock.Unlock()
}

func (wp *workerPool) release(ch *workerChan) bool {
    ch.lastUseTime = CoarseTimeNow()
    wp.lock.Lock()
    if wp.mustStop {
        wp.lock.Unlock()
        return false
    }
    //
    wp.ready = append(wp.ready, ch)
    wp.lock.Unlock()
    return true
}
```

PS：注意 `workerFunc` 方法中，`for c = range ch.ch` 的循环遍历实现中，当检测到 `c==nil` 时退出整个循环，那么何时退出呢？
1、在请求处理正常结束时，注意下面的 `c = nil`，这样 `wp.workersCount--`，即工作协程数 `-1`
```golang
func (s *Server) Serve(ln net.Listener) error {
    // default concurrency set to 256*1024
    maxWorkersCount := s.getConcurrency()
    s.concurrencyCh = make(chan struct{}, maxWorkersCount)
    wp := &workerPool{
        WorkerFunc:      s.serveConn,
        MaxWorkersCount: maxWorkersCount,
        LogAllErrors:    s.LogAllErrors,
        Logger:          s.logger(),
    }
    wp.Start()
    for {
        if c, err = acceptConn(s, ln, &lastPerIPErrorTime); err != nil {
            wp.Stop()
            if err == io.EOF {
                return nil
            }
            return err
        }
        if !wp.Serve(c) {
            s.writeFastError(c, StatusServiceUnavailable,
            "The connection cannot be served for Server.Concurrency limit exceeded")
            c.Close()
            time.Sleep(100 * time.Millisecond)
        }
        c = nil
    }
}
```
2、在 `release` 方法中，将 `ch *workerChan` 放回 `ready` 数组，也就是说，fasthttp 中，每处理完一个连接，就把 `workerChan` 放回，下一个循环再来请求，继续从 `ready` 数组的当前最后的位置取。这样来看，`ready` 数组很像限流算法中的令牌桶，而 `workerChan` 是令牌，谁拿到谁就可以执行。


####  Start 方法的子任务：Clean
最后看下 start 中开启的 clean 定时任务。// wp.clean() 的操作是 查看最近使用的 workerChan, 如果他的最近使用间隔大于某个值，那么把这个 workerChan 清理了。

之所以清理过程只从前遍历清理前面部分，是因为 ready 是 FILO 先进后出的，所以 ready 中越往后的空闲时间最短。
```golang
func (wp *workerPool) clean(scratch *[]*workerChan) {
    maxIdleWorkerDuration := wp.getMaxIdleWorkerDuration()
    currentTime := time.Now()
    wp.lock.Lock()
    ready := wp.ready
    n := len(ready)
    i := 0
    for i <n && currentTime.Sub(ready[i].lastUseTime) > maxIdleWorkerDuration {
        i++
    }
    *scratch = append((*scratch)[:0], ready[:i]...)
    if i > 0 {
        m := copy(ready, ready[i:])
        for i = m; i < n; i++ {
            ready[i] = nil
        }
        wp.ready = ready[:m]
    }
    wp.lock.Unlock()
    tmp := *scratch
    for i, ch := range tmp {
        ch.ch <- nil
        tmp[i] = nil
    }
}
```

##  0x05  再看 goroutine 复用
这里在问一个问题，本文介绍的复用 goroutine 在何处体现？让我们再回到 `workerFunc` 方法：
```golang
func (wp *workerPool) workerFunc(ch *workerChan) {
    var c net.Conn
    var err error
    // for range 在一个 channel 上
    for c = range ch.ch {
        if c == nil {
            // 注意这里，和 clean 方法息息相关
            break
        }
        if err = wp.WorkerFunc(c); err != nil && err != errHijacked {
            // 处理错误
        }
        c = nil

        // 将 ch *workerChan 放回 ready 数组
        if !wp.release(ch) {
            break
        }
    }
    wp.lock.Lock()
    wp.workersCount--
    wp.lock.Unlock()
}
```
让我们回顾下 `workerFunc` 之前的逻辑：当系统中运行的 goroutine 个数小于 `maxWorkersCount` 个时，按需创建（有新的请求且 `ready` 中没有可用令牌），此后每个 `workerFunc` 就在 `for c = range ch.ch` 中源源不断的处理新到来的请求（`ch.ch` 可以理解为 goroutine 暴露给外部的通信 channel）。这样避免了 goroutine 频繁的创建和销毁，从而达到了复用的目的。

##  0x06    总结
fasthttp 协程池是非常优秀的 golang GMP 模型的应用，在实际项目中极具借鉴意义。特别是在高并发的单一循环 Job 处理的场景中。另一个协程池开源项目 [ants](https://github.com/panjf2000/ants) 也是基于 fasthttp 来实现的。

##  0x07    参考
-   [fasthttp-best-practices](https://github.com/valyala/fasthttp#fasthttp-best-practices)
-   [fasthttp 剖析](https://www.jianshu.com/p/a0e766f8dcb0)