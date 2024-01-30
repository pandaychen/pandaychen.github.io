---
layout:     post
title:      Golang 的 Channel（应用篇）
subtitle:   如何使用 channel 开发有保证的应用
date:       2020-01-12
author:     pandaychen
catalog:    true
tags:
    - Golang
---

##  0x00        前言
channel 是 golang 提供的非常有趣且实用的功能，基于 channel 和 goroutine 一起配合可以实现非常实用的功能。如 生产 - 消费者模型、Pipe 等。
对下面这两句话的理解，从初学到现在，已经有了极为深刻的理解：

> Do not communicate by sharing memory; instead, share memory by communicating.
> A send on a channel happens before the corresponding receive from that channel completes. (Golang Spec)

此外，通过阅读 channel 实现的代码，可以了解到，channel 就类似于 ringbuffer 的结构。只是在其他语言（如 C）的实现中，需要有 Mutex 来保证操作的原子性。

通道（channel），像是通道（管道），可以通过它们发送类型化的数据在协程之间通信，可以避开所有内存共享导致的坑；通道的通信方式保证了同步性。数据通过通道：同一时间只有一个协程可以访问数据：所以不会出现数据竞争，设计如此。数据的归属（可以读写数据的能力）被传递。

通道实际上是类型化消息的队列：使数据得以传输。它是先进先出（FIFO）结构的所以可以保证发送给他们的元素的顺序（有些人知道，通道可以比作 Unix shells 中的双向管道（tw-way pipe））。通道也是引用类型，所以我们使用 make() 函数来给它分配内存。

## 0x01 Channel 基础语法
基础语法：
```go
c := make(chan bool) // 创建一个无缓冲的 bool 型 Channel
c <- x        // 向一个 Channel 发送一个值
<- c          // 从一个 Channel 中接收一个值
x = <- c      // 从 Channel c 接收一个值并将其存储到 x 中
x, ok = <- c  // 从 Channel 接收一个值，如果 channel 关闭（close）了或没有数据，那么 ok 将被置为 false
```

有缓冲 channel 和无缓冲 channel：
```go
c1 := make(chan []byte)                // 无缓冲
c2 := make(chan []byte, 1024)           // 有缓冲
c3 := make(chan []byte, 1)              // 注意，这是有缓冲
```

![channel-type](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/goroutine/channel-type.png)

下文再讨论这两种类型的 channel 在并发语义上的不同

###     Channel 的类型
默认情况下，通信是同步且无缓冲的：在有接受者接收数据之前，发送不会结束。可以想象一个无缓冲的通道在没有空间来保存数据的时候：必须要一个接收者准备好接收通道的数据然后发送者可以直接把数据发送给接收者。所以通道的发送 / 接收操作在对方准备好之前是阻塞的：

1.      对于同一个通道，发送操作（协程或者函数中的），在接收者准备好之前是阻塞的：如果 ch 中的数据无人接收，就无法再给通道传入其他数据：新的输入无法在通道非空的情况下传入。所以发送操作会等待 ch 再次变为可用状态：就是通道值被接收时（可以传入变量）。

2.      对于同一个通道，接收操作是阻塞的（协程或函数中的），直到发送者可用：如果通道中没有数据，接收者就阻塞了。

###     无缓冲 Channel
无缓冲：发送和接收动作是同时发生的。如果没有 goroutine 读取 channel （<- channel），则发送者 (channel <-) 会一直阻塞。
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2020/channel/channel_1.png)

###     有缓冲 Channel
缓冲：缓冲 channel 类似一个有容量的队列。当队列满的时候发送者会阻塞；当队列空的时候接收者会阻塞。
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2020/channel/channel_2.png)

##      细节（容易忽略）

-       向一个已经被 close 的 channel 中继续发送数据会导致 run-time panic
-       从一个 nil channel 中接收数据会一直被 block
-       select 的 case 中建议不要出现非阻塞的条件（如等待某个 channel 读事件，其他 routine 将此 channel 关闭，导致该 case 永远为真）


##      channel 基础应用

####    阻塞等待至完成
阻塞等待（某事件）完成，类似于 signal 的功能

```go
func c1() {
        c := make(chan bool)    // 无缓冲 channel
        go func() {
                fmt.Println("Do something")
                close(c)
        }()
        <-c
        fmt.Println("Done!")
}
```

####     groutine pool 池并发
简单的 groutine pool，通过 channel 发送通知

```go
package main

import (
        "fmt"
        "sync"
)

func worker(wg *sync.WaitGroup, start chan bool, index int) {
        <-start
        defer wg.Done()
        fmt.Println("This is Worker:", index)
}

func main() {
        wg := sync.WaitGroup{}
        start := make(chan bool)
        for i := 1; i <= 100; i++ {
                wg.Add(1)
                go worker(&wg, start, i)
        }
        close(start)
        wg.Wait()
}
```

####       channel + select 结合
Channel 的事件触发机制与 select 的 case 机制结合，差不多成为 golang 编程范式了。从不同的并发执行的 goroutine 中获取值可以通过关键字 select 来完成，select 是 golang 特别有趣的用法:)。在实现微服务网关时，一般会有一个主控协程（manager goroutine）负责处理各个子 channel 的调度，是非常优雅的实现 [代码](https://github.com/pandaychen/gobetween/blob/master/src/server/scheduler/scheduler.go#L117)。

> select 是 golang 中 pseudo-random 伪随机算法的一种实现

```go
select {
        case x := <- chan1:
        // … 使用 x 进行一些操作

        case y, ok := <- chan2:
        // 使用 y 进行一些操作，检查 ok 值判断 chan2 是否已经关闭

        case chan3 <- z:
        // z 值被成功发送到 chan3 上时

        default:
        // 上面 case 均无法执行时，执行此分支（防止程序逻辑阻塞的范式）
}
```

####    routine 通知子 routine 退出
主 routine 利用 channel 发送数据，终结子 routine-workers

```GO
// 这是一个常见的终结 sub worker goroutines 的方法
// 每个 worker goroutine 通过 select 监视一个 die channel 来及时获取 main goroutine 的退出通知
package main

import (
        "fmt"
        "sync"
        "time"
)

func worker(wg *sync.WaitGroup, die chan bool, index int) {
        defer wg.Done()
        fmt.Println("Begin: This is Worker:", index)
        for {
                select {
                //case xx：
                //Doing Jobs forever...
                case <-die:
                        fmt.Println("Done: This is Worker:", index)
                        return
                }
        }
}

func main() {
        die := make(chan bool)
        var wg sync.WaitGroup

        for i := 1; i <= 100; i++ {
                wg.Add(1)
                go worker(&wg, die, i)
        }

        time.Sleep(time.Second * 5)
        close(die)
        time.Sleep(time.Second * 5)
        wg.Wait()
}
```

####      有缓冲 channel 的默认行为
默认 unbuffered channel 与 buffered channel 的行为是不同的，它们体现了一种交付保证（unbuffered channel）和不保证的思路（buffered channel），见文章 [聊一聊 Go 中 channel 的行为](https://www.infoq.cn/article/wZ1kKQLlsY1N7gigvpHo)，例如：看出带缓冲的 channel 略有不同。尽管已经 close 了，但我们依旧可以从中读出关闭前写入的 3 个值。第四次读取时，则会返回该 channel 类型的零值。向这类 channel 写入操作也会触发 panic。

```go
package main
import "fmt"

func main() {
        c := make(chan int, 3)
        c <- 15
        c <- 34
        c <- 65
        close(c)
        fmt.Printf("%d\n", <-c)
        fmt.Printf("%d\n", <-c)
        fmt.Printf("%d\n", <-c)
        fmt.Printf("%d\n", <-c)
        c <- 1
}
```

我们再看看其他一些 channel 在应用中的范式（套路）。

##      0x02 channel 的应用范式（Interesting）

####     for-select 经典用法
在使用 select 时很少只是对其进行一次 evaluation，我们常常将其与 `for {}` 结合在一起使用，并选择适当时机从 `for {}` 中退出。
```go
for {
        select {
                case x := <- chan1:
                // … 使用 x 进行一些操作

                case y, ok := <- chan2:
                // 使用 y 进行一些操作，检查 ok 值判断 chan2 是否已经关闭

                case chan3 <- z:
                // z 值被成功发送到 chan3 上时

                default:
                // 上面 case 均无法执行时，执行此分支（防止程序逻辑阻塞的范式）
        }
}
```

####     for-select-Timeout 经典用法
带超时机制的 select 也是经典用法，下面是示例代码：
```go
func worker(start chan bool) {
        //30s 的定时器
        timeout := time.After(30 * time.Second)
        for {
                select {
                case <-othercase1:
                        //do things...
                case othercase2 <- y:
                        //do things...
                case <- timeout:
                    return
                }
        }
}
```

####     心跳（Heartbeat）
一个定时心跳上报的 demo（使用 select ）：
```go
func worker(start chan bool) {
        heartbeat := time.Tick(10 * time.Second)
        for {
                select {
                default:
                        time.Sleep(time.Duration(1) * time.Second)
                case <- heartbeat:
                        // 上报心跳 or 做心跳超时回调任务
                case othercase:
                        // Do other things
                }
        }
}
```

####     重点：关闭 channel 的优雅做法
在主动（或被动）关闭 channel 的时候，**最好设置 channel 为 nil（nil channel 在 select 中很有用**），这是一个好习惯，通过将已经关闭的 channel 置为 `nil`，下次 select 将会阻塞在该 channel 上，使得 select 继续下面的分支 evaluation

```golang
package main

import "fmt"
import "time"

func main() {
        var c1, c2 chan int = make(chan int), make(chan int)
        go func() {
                time.Sleep(time.Second * 5)
                c1 <- 5
                close(c1)
        }()

        go func() {
                time.Sleep(time.Second * 7)
                c2 <- 7
                close(c2)
        }()

        for {
                select {
                case x, ok := <-c1:
                        if !ok {
                                c1 = nil
                        } else {
                                fmt.Println(x)
                        }
                case x, ok := <-c2:
                        if !ok {
                                c2 = nil
                        } else {
                                fmt.Println(x)
                        }
                }
                if c1 == nil && c2 == nil {
                        break
                }
        }
        fmt.Println("over")
}
```

####     生产 - 消费者模型
生产者产生一些数据将其放入 channel；然后消费者按照顺序，一个一个的从 channel 中取出这些数据进行处理。这是最常见的 channel 的使用方式。当 channel 的缓冲用尽（无数据）时，生产者必须等待（阻塞）

```go
package main

import (
        "fmt"
        "sync"
        "time"
)

func producer(wg *sync.WaitGroup, c chan int64, max int) {
        defer wg.Done()
        for {
                for i := 0; i < max; i++ {
                        c <- time.Now().Unix()
                }

                time.Sleep(1 * time.Second)
        }
}

func consumer(wg *sync.WaitGroup, c chan int64) {
        defer wg.Done()
        var v int64
        ok := true

        for ok {
                if v, ok = <-c; ok {
                        fmt.Println(v)
                }
        }
}

func main() {
        var wg sync.WaitGroup
        var chan_size int = 10
        chan1 := make(chan int64, chan_size)
        go producer(&wg, chan1, chan_size)
        wg.Add(1)
        go consumer(&wg, chan1)
        wg.Add(1)

        wg.Wait()
        close(chan1)
}
```
####     生成器（generator）

```go
package main

import "fmt"

func uuidgenerator() <-chan string {
        id := make(chan string)
        go func() {
                var counter int64 = 0
                for {
                        id <- fmt.Sprintf("%x", counter)
                        counter += 1
                }
        }()
        return id
}
func main() {
        id := uuidgenerator()
        for i := 0; i < 10; i++ {
                fmt.Println(<-id)
        }
}
```

####    检查 buffered channel 是否已满
使用 buffered channel 时，可能需要检查当前的 channel 是否已经满（不希望 goroutine 被阻塞），采用 `default` 分支来规避该场景：

```GOLANG
cc := make(chan int, 1)
cc <- data1

select {
    case cc <- data2:
        fmt.Println("data equeue")
    default:
        fmt.Println("channel was full")
}
```

####    fan-in
利用 channel 实现 fan-in 的典型场景，并发的从多个 channel 中获取数据，将其合并到结果 channel 中：

```GOLANG
func fanIn(chans []chan int, out chan int) {
    wg := sync.WaitGroup{}
    wg.Add(len(chans))
    for _, ch := range chans {
        go func(ch chan int) {
            for t := range ch {
                out <- t
            }
            wg.Done()
        }(ch)
    }
    wg.Wait()
    close(out)
}
```

####    检测 channel 是否被关闭
channel 被关闭时仍然可以读数据，但是写会导致 `panic`，通常使用 `data, ok := <- chan` 的 `ok` 来判断 channel 是否被关闭，只有当 channel 无数据，且 channel 被 `close` 了，才会返回 `ok==false`

```GOLANG
func checkChanClose(ch chan int) bool {
    select {
    case _, received := <- ch:
        return !received
    default:
    }
    return false
}
```

但是实际中，一般不会直接使用 `checkChanClose` 方法，因为可能有延迟，如下代码：
```golang
if checkChanClose(ch) {
    // 关闭的场景，exit
    return
}
// 假设这里被异步 close 了
// 未关闭的场景，继续执行（可能还是会 panic）
ch <- x
```

所以，这个问题就变成如何避免使用到已经被 closed 的 channel，避免 `panic`。并且，要保证按照如下顺序触发：
1. 先触发 `ctx.Done()` 事件
2. 再执行 `close(c)` 关闭 channel 操作，保证这个时序的才能保证
```GOLANG
select {
        case <-ctx.Done():
            return
        case v, ok := <-c:
            // do something
            if !ok{
                //channel is close
                //do something
            }
        default:
            // do default
}
```


##      0x03 channel 的陷阱

-       `nil` channel 会阻塞
对一个没有初始化的 channel 进行读（或写）操作都将发生阻塞，如下：

```go
package main

func main() {
        var c chan int
        <-c
        //c <- 1
}

// 运行报错：
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive (nil chan)]:
main.main()
        /root/chan1.go:6 +0x29
exit status 2
```

##      0x04    工作经验：并发请求下的 channel 典型用法
在开发中，遇到需要并发调用请求（各请求间无关联）的场景，如客户端并发请求服务接口等，通常我们可以使用下面这种基于 channel 的简单并发请求框架：
-       不关心调用顺序
-       不关系结果顺序
-       各个 goroutine 必须要实现超时机制，避免调用方阻塞

```golang
func main() {
        var max = 10
        var oriList = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
        token := make(chan struct{}, max)
        results := make(chan int, len(oriList))
        var wg sync.WaitGroup
        wg.Add(len(oriList))

        for _, val := range oriList {
                token <- struct{}{}
                go func(v int) {
                        //do some client business calls
                        results <- v
                        <-token
                        wg.Done()
                }(val)
        }
        wg.Wait()
        close(results)

        for v := range results {
                fmt.Println(v)
        }
}
```

##      0x05    协程的优雅退出
本小节介绍下 goroutine 的优雅主动退出机制（不能通过某种手段强制关闭，只能等待 goroutine 主动退出）

####    for-range 退出
`for-range chan` 常用于来遍历 channel，`range` 可以感知 channel 的关闭，当 channel 被发送数据的协程关闭时，`range` 就会结束，接着退出 `for` 循环。并发中的使用场景是：goroutine 只从 `in` 这个 channel 读取数据，然后进行处理，当 `in` 被 `close` 时，goroutine 可自动退出

```GOLANG
go func(in <-chan int) {
    // Using for-range to exit goroutine
    // range has the ability to detect the close/end of a channel
    for x := range in {
        fmt.Printf("Process %d\n", x)
    }
    return
}(inCh)
```

####    使用 `,ok` 退出
`for-select chan` 常用于实现调度器（类似于多路复用多个 channel 的能力），但 `select` 机制没有感知 channel 的关闭，这引出了 `2` 个问题：

1、继续在关闭的 channel 上读，会读到通道传输数据类型的零值，如果是指针类型，读到 `nil`，继续处理还会产生 `nil`<br>
2、继续在关闭的 channel 上写，将会触发 `panic`<br>

问题 `2` 可以这样解决，channel 只由发送方关闭，接收方不可关闭，即某个写 channel 只由使用该 `select` 的 goroutine 关闭，`select` 中就不存在继续在关闭的 channel 上写数据的问题

问题 `1` 可以使用 `ok` 机制来检测 channel 的关闭：
1、如果某个 channel 关闭后，需要退出 goroutine，直接 `return` 即可。如下代码中，该 goroutine 需要从 `in` 读数据，还需要定时做其他事情（由于有 `2` 件事做，所有不能使用 `for-range`，要使用 `for-select`，当 `in` 关闭时，`ok` 为 `false`，直接 `return` 退出 goroutine 即可

```go
go func() {
    // in for-select using ok to exit goroutine
    for {
        select {
        case x, ok := <-in:
            // 检测 in 是否被关闭了
            if !ok {
                return
            }
            fmt.Printf("Process %d\n", x)
            processedCnt++
        case <-t.C:
            fmt.Printf("Working, processedCnt = %d\n", processedCnt)
        }
    }
}()
```


2、如果某个 channel 关闭了，不再处理该 channel，而是需要继续处理其他 `case`，退出是等待所有的可读 channel 关闭。这里就 **需要使用 `select` 的一个特性：`select` 不会在 `nil` 的 channel 上进行等待。把只读 channel 设置为 `nil` 即可解决**

```GO
go func() {
    // in for-select using ok to exit goroutine
    for {
        select {
        case x, ok := <-in1:
            if !ok {
                // 被关闭了，人为设置为 nil
                in1 = nil
            }
            // Process
        case y, ok := <-in2:
            if !ok {
                in2 = nil
            }
            // Process
        case <-t.C:
            fmt.Printf("Working, processedCnt = %d\n", processedCnt)
        }

        // If both in channel are closed, goroutine exit
        if in1 == nil && in2 == nil {
            return
        }
    }
}()
```

####    使用退出 channel 退出
第三种场景：启动了一个工作协程处理数据，如何通知它退出？这里比较推荐的方法是使用一个专门的 channel，发送退出的信号，可以解决这类问题。以第二个场景为例，goroutine 入参包含一个停止 channel `stopCh`，当 `stopCh` 被关闭，`case <-stopCh` 会执行，直接返回即可

当启动了 `100` 个或更多的 worker 时，只要 `main` 执行关闭 `stopCh`，这样每一个 worker 都会都到 "退出信号"，进而关闭（如果 `main` 向 `stopCh` 发送 `100` 个数据，这样就非常低效）

```GOLANG
func worker(stopCh <-chan struct{}) {
    go func() {
        defer fmt.Println("worker exit")
        // Using stop channel explicit exit
        for {
            select {
            case <-stopCh:
                fmt.Println("Recv stop signal")
                return
            case <-t.C:
                fmt.Println("Working...")
            }
        }
    }()
    return
}
```

##  0x06        Review：channel 的行为
[The Behavior Of Channels](https://www.ardanlabs.com/blog/2017/10/the-behavior-of-channels.html) 一文给出了 channel 的可靠性交付语义描述，本小节汇总下

把 Channel 行为与信号机制（将 channel 视为信号机制）联系起来：一个 channel 允许一个 goroutine 向另一个 goroutine 发出特定事件的信号，包含三个属性：

-       交付保证（Guarantee Of Delivery）
-       状态（State）
-       有数据或无数据（With or Without Data）

####    交付保证（Guarantee Of Delivery）
针对下述代码，执行发送的 goroutine （Send）是否需要保证，在继续执行之前，被 Receive 所在的 goroutine 接收到了？参考下图，无缓冲和有缓冲在交付保证时提供的行为是不同的
```GO
go func() {
     p := <-ch // Receive
}()

ch <- "paper" // Send
```

![1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/goroutine/86_guarantee_of_delivery-1.png)

####    状态（State）
channel 的行为直接受其当前状态的影响。channel 的状态可以为 `nil`，`open` 或 `closed`

```GO
// ** nil channel
// 如果声明为零值的话，将会是 nil 状态
var ch chan string
// 显式的赋值为 nil，设置为 nil 状态
ch = nil

// ** open channel
// 使用内部函数 make 创建的 channel，为 open 状态
ch := make(chan string)    // ** closed channel

// 使用 close 函数的，为 closed 状态
close(ch)
```


状态决定了发送和接收操作的行为。信号通过一个 channel 发送和接收；当一个 channel 为 `nil` 状态时，channel 上的任何发送或接收都将被阻塞。当为 `open` 状态时，信号可以被发送和接收。如果被置为 `closed` 状态的话，信号不能再被发送，但仍有可能接收到信号

![2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/goroutine/86_state-2.png)

####    信号有 / 无数据（With and Without Data）
-  有数据：通过在 channel 上执行发送带有数据的信号，如 `ch <- "paper"`
-  无数据：通过关闭一个 channel 来发送没有数据的信号，如 `close(ch)`

他们的语义如下：

有数据，当用数据发出信号时，通常是因为：
- goroutine 被要求开始一项新任务
- goroutine 报告了一个结果

而无数据的场景是：
-  goroutine 被通知要停止它当前正在做的事情
-  goroutine 报告说已经完成，没有结果
-  goroutine 报告说它已经完成了处理，并且关闭

####    信号有数据（Signaling With Data）

当要使用数据进行信号传输时，可以根据需要的担保类型选择合适的 channel 配置选项：

![3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/goroutine/86_signaling_with_data-3.png)
上面图中，buffered channel 的行为值得关注下，unbuffered channel 是可担保的类型

有保证（guarantee）：
-  因为信号的接收在信号发送完成之前就发生了
-  一个没有缓冲的通道可以保证发送的信号已经收到

无保证（no guarantee）：
-  因为信号的发送是在信号接收完成之前发生的
- 一个大小 `>1` 的缓冲通道不能保证（当前）发送的信号已经收到

延迟保证（delayed guarantee）：
-  因为第一个信号的接收，在第二个信号发送完成之前就发生了
-  一个大小 `=1` 的缓冲通道提供了一个延迟的保证，它可以保证发送的前一个信号已经收到

####    无数据信号（Signaling Without Data）
通常此场景适用于取消（cancel）。允许一个 goroutine 发出信号，让另一个 goroutine 取消当前正在做的事情。取消可以使用非缓冲和缓冲 channel 来实现，但是在没有数据发送的情况下使用缓冲 channel 会更好，如下图：

![4](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/goroutine/86_signaling_without_data-4.png)

`close` 方法用于在没有数据的情况下发出信号。如前文描述，可在一个关闭的通道接收到信号。事实上，在一个关闭的 channel 上的任何接收都不会阻塞，接收操作总是返回。在大多数情况下，也推荐使用标准库 `context` 来实现无数据的信号传递：`context` 包使用一个没有缓冲的 channel 来进行信号传递，而内置函数 `close` 发送没有数据的信号。如果选择使自定义通道进行取消（而不是 context 包）那么通道类型应该是 `chan struct{}`，这是一种零空间的惯用方式，用来表示一个仅用于信号传输的 channel

相关使用可以参考`ctx.Done()`的[实现](https://cs.opensource.google/go/go/+/refs/tags/go1.21.6:src/context/context.go;l=433)

##      0x07 channel 交付：典型应用场景

####    Signal With Data - Guarantee - Unbuffered Channels
当需要知道发送的信号已经收到时，就会有两种情况出现。一种是等待任务，另一种是等待（接收）结果

1、场景 1：等待任务

```GO
func waitForTask() {
    ch := make(chan string)

    go func() {
        p := <-ch

        // Employee performs work here.

        // Employee is done and free to go.
    }()

    time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)

    // 由于使用了一个没有缓冲的 channel，所以当你的发送操作完成后，你就得到了该雇员已经收到该文件的保证。接收发生在发送之前
    // 那么，如果这里使用 buffered channel，结论又是如何呢？
    ch <- "paper"
}
```


2、场景 2：等待结果

```GO
func waitForResult() {
    ch := make(chan string)

    go func() {
        time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)

        ch <- "paper"

        // Employee is done and free to go.
    }()

    // 由于这是一个没有缓冲的通道，所以接收在发送之前就发生了，并且保证你已经收到了结果。一旦员工有了这样的保证，他们就可以自由地工作了。在这种情况下，你不知道他们要花多长时间才能完成这项任务
    p := <-ch
}
```

####  Signal With Data - No Guarantee - Buffered Channels >1
使用 buffered channel，需要明确自己的应用的场景，buffered channel 具有明确定义的空间，可用于存储正在发送的数据，先明确如下问题：

1.  buffered channel 的大小？
2.  如果 channel 的生产者速率超过消费者，导致 buffered channel 满的情况下，可以丢弃生产者的任务吗？
3.  如果程序意外终止，缓冲区中等待的任何内容都将丢失，是否可以接受此场景？

基于 buffered channel 演化了两种通用场景：Fan Out 和 Drop

1、场景 1：Fan Out

```GO
func fanOut() {
    emps := 20
    ch := make(chan string, emps)

    // N 个并发的生产者
    for e := 0; e < emps; e++ {
        go func() {
            time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
            ch <- "paper"
        }()
    }

    for emps > 0 {
        p := <-ch
        fmt.Println(p)
        emps--
    }
}
```

2、场景 2：Drop

如下 buffered channel 场景下，生产者速率过快导致 buffer 无空闲空间（消费者速率赶不上），这样可以通过 `select` 选择丢弃：

```GO
func selectDrop() {
    const cap = 5
    ch := make(chan string, cap)

    go func() {
        for p := range ch {
            fmt.Println("employee : received :", p)
        }
    }()

    const work = 20
    for w := 0; w < work; w++ {
        select {
            case ch <- "paper":
                fmt.Println("manager : send ack")
            default:
                fmt.Println("manager : drop")
        }
    }

    close(ch)
}
```

####    Signal With Data - Delayed Guarantee - Buffered Channel 1

1、场景 1：等待任务

```GO
func waitForTasks() {
    //channel 每次只能容纳一个 job
    ch := make(chan string, 1)

    go func() {
        for p := range ch {
            fmt.Println("employee : working :", p)
        }
    }()

    const work = 10
    for w := 0; w < work; w++ {
        ch <- "paper"
    }

    close(ch)
}
```

2、场景 2：Signal Without Data（Context）

```GO
func withTimeout() {
    duration := 50 * time.Millisecond

    ctx, cancel := context.WithTimeout(context.Background(), duration)
    defer cancel()

    ch := make(chan string, 1)

    go func() {
        time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
        ch <- "paper"
    }()

    select {
    case p := <-ch:
        fmt.Println("work complete", p)
    case <-ctx.Done():
        fmt.Println("moving on")
    }
}
```

##  0x08        参考
-       [The Behavior Of Channels](https://www.ardanlabs.com/blog/2017/10/the-behavior-of-channels.html)
-       [聊一聊 Go 中 channel 的行为](https://www.infoq.cn/article/wZ1kKQLlsY1N7gigvpHo)
-       [Concurrency in Go 中文笔记](https://www.kancloud.cn/mutouzhang/go/596835)
-       [go 并发编程范式](https://www.jianshu.com/p/3e1837860575)
-       [Golang 并发模型：合理退出并发协程](https://segmentfault.com/a/1190000017251049)
-       [Golang Channel 用法简编](https://tonybai.com/2014/09/29/a-channel-compendium-for-golang/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权