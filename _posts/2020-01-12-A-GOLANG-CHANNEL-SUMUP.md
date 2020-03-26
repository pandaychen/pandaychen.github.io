---
layout:     post
title:      Golang 的 Channel（应用篇）
subtitle:   如何使用 channel 开发有保证的应用
date:       2020-01-12
author:     pandaychen
catalog:    true
tags:
    - Channel
    - golang
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
c2 := male(chan []byte, 1024)           // 有缓冲
```

###     Channel 的类型
默认情况下，通信是同步且无缓冲的：在有接受者接收数据之前，发送不会结束。可以想象一个无缓冲的通道在没有空间来保存数据的时候：必须要一个接收者准备好接收通道的数据然后发送者可以直接把数据发送给接收者。所以通道的发送 / 接收操作在对方准备好之前是阻塞的：

1.      对于同一个通道，发送操作（协程或者函数中的），在接收者准备好之前是阻塞的：如果 ch 中的数据无人接收，就无法再给通道传入其他数据：新的输入无法在通道非空的情况下传入。所以发送操作会等待 ch 再次变为可用状态：就是通道值被接收时（可以传入变量）。

2.      对于同一个通道，接收操作是阻塞的（协程或函数中的），直到发送者可用：如果通道中没有数据，接收者就阻塞了。

###     无缓冲 Channel
无缓冲：发送和接收动作是同时发生的。如果没有 goroutine 读取 channel （<- channel），则发送者 (channel <-) 会一直阻塞。
![image](https://s2.ax1x.com/2020/01/20/1PGlLt.png)

###     有缓冲 Channel
缓冲：缓冲 channel 类似一个有容量的队列。当队列满的时候发送者会阻塞；当队列空的时候接收者会阻塞。
![image](https://s2.ax1x.com/2020/01/20/1PGtJg.png)

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

####    routine通知子routine退出
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

####     关闭 channel 的优雅做法
在主动（或被动）关闭 channel 的时候，最好设置 channel 为 nil（nil channel 在 select 中很有用），这是一个好习惯，通过将已经关闭的 channel 置为 nil，下次 select 将会阻塞在该 channel 上，使得 select 继续下面的分支 evaluation。
```go
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
生产者产生一些数据将其放入 channel；然后消费者按照顺序，一个一个的从 channel 中取出这些数据进行处理。这是最常见的 channel 的使用方式。当 channel 的缓冲用尽（无数据）时，生产者必须等待（阻塞）。

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

##      0x03 channel 的陷阱

-       nil channel 会阻塞
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



##  0x04        参考
-       [聊一聊 Go 中 channel 的行为](https://www.infoq.cn/article/wZ1kKQLlsY1N7gigvpHo)
-       [Concurrency in Go 中文笔记](https://www.kancloud.cn/mutouzhang/go/596835)
-       [go 并发编程范式](https://www.jianshu.com/p/3e1837860575)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权