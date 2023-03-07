---
layout:     post
title:      Kratos 源码分析：Errgroup 机制
subtitle:   分析原生的 errgroup 即 Kratos 的 errgroup
date:       2020-07-20
header-img: img/golang-horse-fly.png
author:     pandaychen
catalog:    true
tags:
    - Kratos
    - errgroup
---

##  0x00    前言
问题一：一般在 Golang 中，直接使用 `go` 命令字开启子 goroutine 来实现并发，但是直接 go 的话函数是无法对返回数据进行 error 处理的。（`go` 开启的方法也无返回值）如何实现？

问题二：调用并发的 goroutine 去批量执行 Job，有失败的可能，如何把第一个出错的 goroutine 信息返回给调用端？


通常有几种处理方法：

1、打日志，不推荐<br>

2、采用 `recovery` 方式捕捉错误，比较常见的处理方法<br>

```golang
func cal(a int , b int ,n *sync.WaitGroup)  {
    defer func() {
        err := recover()
        if err != nil {
            fmt.Println("cal fail")
        }
    }()

    defer n.Done()
    c := a/b
    fmt.Printf("%d / %d = %d\n",a,b,c)
}

func main() {
    var go_sync sync.WaitGroup
    for i :=0 ; i<10 ;i++{
        go_sync.Add(1)
        go cal(i,i+1,&go_sync)
    }
    go_sync.Wait()
}
```

3、针对协程池场景，使用 channel 作为错误传出的媒介<br>

但是有没有更简单更优雅的方法呢？答案就是 `errgroup`。`errgroup` 包为一组子任务的 goroutine 提供了 goroutine 同步及错误取消功能。

##	0x01	原生 errgroup 使用
Golang 的 [errgroup](https://godoc.org/golang.org/x/sync/errgroup) 提供了这么一种机制，可以在并发的协程运行中捕获错误并返回。看下面的例子：

```golang
var g errgroup.Group        // 初始化 errgroup.Group 对象
var urls = []string{
        "http://www.golang.org/",
        "http://www.google.com1/",
        "http://www.somestupidname.com/",
}
for _, url := range urls {
        url := url // https://golang.org/doc/faq#closures_and_goroutines
        g.Go(func() error {
                // Fetch the URL.
                resp, err := http.Get(url)
                if err == nil {
                    resp.Body.Close()
                }
                return err
        })
}
// Wait for all HTTP fetches to complete.
if err := g.Wait(); err == nil {
        fmt.Println("Successfully fetched all URLs.")
} else {
        fmt.Println(err)
}
```

上面这段代码的输出是：

```text
Get http://www.google1.com/: dial tcp: lookup www.google.com1 on 183.60.83.19:53: no such host
```


##  0x01	原生 errgroup 分析
本小节简单分析下，官方的 `golang.org/sync/errgroup` 包的实现。读过 `errgroup` 代码之后，会发现这就是基于 `golang`"若干" 特性组合成的复合结构：
`errgroup.Group` 结构如下：

```golang
type Group struct {
    cancel  func()
    // 包含了个 WaitGroup 用于同步等待所有 Gorourine 执行
    wg      sync.WaitGroup
    errOnce sync.Once
    // golang 特有的单例模式，利用原子操作进行锁定值判断只会传递第一个出错的协程的 error
    err     error
}
```

`errgroup.Group` 结构成员如下：
-   `err error`： 用来保存第一个错误
-   `errOnce sync.Once`：`sync.Once` 机制是 `golang` 特有的 Singleton 模式，可以保证它的回调函数只执行一次
-   `wg sync.WaitGroup`： 等待所有 goroutine 结束的固定轮子
-   `cancel func()`：这里保存的是 `contex.WithCancel` 返回的 `cancel` 参数，用以取消所有子协程的运行

总结: `wg` 用来保证在所有 goroutine 结束后返回 `err` 的值。在对 `err` 赋值的时候，只取第一个 `err` 值，这里使用到 `sync.Once` 保证。


####	WithContext 方法
`errgroup.WithContext`， 用于控制每一个 goroutine 的生理周期。 主要用于在不同的 goroutine 之间同步请求特定的数据、取消信号以及处理请求的截止日期。该方法实现，主要是初始化 `Group` 对象，通过 `context.WithCancel(ctx)` 创建 `context` 树状结构，`cancel` 保存在 `Group.cancel` 中：

```golang
func WithContext(ctx context.Context) (*Group, context.Context) {
    ctx, cancel := context.WithCancel(ctx)
    return &Group{cancel: cancel}, ctx
}
```

####	Wait 方法

使用 `group.wg.Wait` 方法，等待所有 goroutine 结束，`err` 有值，意味着出错，通过 `return` 返回给调用端。如果设置了 `cancel`，则通过调用 `cancel` 关闭这个 `context.Context`，从而关闭了此 `context` 派生出来的 `goroutine`：

```golang
func (g *Group) Wait() error {
    g.wg.Wait()
    if g.cancel != nil {
        g.cancel()
    }
    return g.err
}
```

####	Go 方法
`Go` 方法是多 routine 并发的常用套路了。
1.	运行用户传入的方法 `f()`
2.	当 `f()` 出错时，进入错误处理收集的逻辑，使用 `g.errOnce.DO()` 保证回调函数只会被执行一次
	-	将一个错误保存 `g.err = err`
	-	如果设置了 `cancel`，那么同时关闭 `g.cancel()`

这里尤其注意下面这段代码：

```golang
g.errOnce.Do(func() {
	g.err = err
	if g.cancel != nil {
		// 如果出现第一个 err 为非 nil 就会去调用关闭接口，即关闭后面的所有子 gorourine
		g.cancel()
	}
})
```


```golang
func (g *Group) Go(f func() error) {
    g.wg.Add(1)

    go func() {
        defer g.wg.Done()

        if err := f(); err != nil {
            g.errOnce.Do(func() {
				// 这里的 Once() 是亮点
                g.err = err
                if g.cancel != nil {
                    g.cancel()
                }
            })
        }
    }()
}
```

##	0x02	Kratos 的 errgroup 改进
Kratos 对 `errgroup` 做了一些扩展，加入了 `recover` 和并行数的 `errgroup`，`err` 中包含详细堆栈信息（coredump 发生时），使得 `errgroup` 机制更加健壮。本小节分析下 Kratos 的改造部分：

####	Group 结构
与原生 `errgroup` 库相比，增加了三个成员：

```golang
type Group struct {
	err     error
	wg      sync.WaitGroup
	errOnce sync.Once

	workerOnce sync.Once
	ch         chan func(ctx context.Context) error
	chs        []func(ctx context.Context) error

	ctx    context.Context
	cancel func()
}
```

####	Go 方法改造

```golang
// Go calls the given function in a new goroutine.
//
// The first call to return a non-nil error cancels the group; its error will be
// returned by Wait.
func (g *Group) Go(f func(ctx context.Context) error) {
	g.wg.Add(1)
	if g.ch != nil {
		select {
		case g.ch <- f:
		default:
			g.chs = append(g.chs, f)
		}
		return
	}
	go g.do(f)
}
```

再看看 `do` 方法的实现，`do` 方法在 `defer` 结构中加入了捕获异常的逻辑，具体步骤是：
1.	执行用户方法 `err = f(ctx)`
2.	`defer` 中实现对堆栈信息的存储（如果发生错误）及 `err` 的保存

```golang
func (g *Group) do(f func(ctx context.Context) error) {
	ctx := g.ctx
	if ctx == nil {
		ctx = context.Background()
	}
	var err error
	defer func() {
		if r := recover(); r != nil {
			buf := make([]byte, 64<<10)
			buf = buf[:runtime.Stack(buf, false)]
			err = fmt.Errorf("errgroup: panic recovered: %s\n%s", r, buf)
		}
		if err != nil {
			g.errOnce.Do(func() {
				g.err = err
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
		g.wg.Done()
	}()
	err = f(ctx)
}
```

此外，Kratos 还提供了 `errgroup.GOMAXPROCS` 方法，用来设置最大并行数 `GOMAXPROCS`，其原理也比较简单，利用一个长度为 `n` 来 channel 来并发调用 `do` 方法：
```golang
// GOMAXPROCS set max goroutine to work.
func (g *Group) GOMAXPROCS(n int) {
	if n <= 0 {
		panic("errgroup: GOMAXPROCS must great than 0")
	}
	g.workerOnce.Do(func() {
		// 利用 channel 实现限制
		g.ch = make(chan func(context.Context) error, n)
		for i := 0; i < n; i++ {
			go func() {
				for f := range g.ch {
					g.do(f)
				}
			}()
		}
	})
}
```

####	Wait 方法
```golang
// Wait blocks until all function calls from the Go method have returned, then
// returns the first non-nil error (if any) from them.
func (g *Group) Wait() error {
	if g.ch != nil {
		for _, f := range g.chs {
			g.ch <- f
		}
	}
	g.wg.Wait()
	if g.ch != nil {
		close(g.ch) // let all receiver exit
	}
	if g.cancel != nil {
		g.cancel()
	}
	return g.err
}
```

##	0x03	Kratos 的 errgroup 使用方式
Kratos 的 `errgroup` 包含三种常用方式：

-	直接使用

此时不会因为一个任务失败导致所有任务被 cancel（默认的 errgroup 由于会创建 `context` 及 `cancel`，有错误就会强制调用 `cancel`，Kratos 这里做了优化）：

```golang
g := &errgroup.Group{}
g.Go(func(ctx context.Context) {
		//NOTE: 此时 ctx 为 context.Background()
		//do something
})
```

-	`WithContext` 方式

使用 `WithContext` 时不会因为一个任务失败导致所有任务被 cancel，这里的 `context` 是非 `cancel` 型的 ctx

```golang
g := errgroup.WithContext(ctx)
g.Go(func(ctx context.Context) {
		//NOTE: 此时 ctx 为 errgroup.WithContext 传递的 ctx
		do something
})
```

-	`WithCancel` 方式

使用 `WithCancel` 时如果有一个任务失败会导致所有 **未进行或进行中** 的任务被 cancel:

```golang
g := errgroup.WithCancel(ctx)
g.Go(func(ctx context.Context) {
		//NOTE: 此时 ctx 是从 errgroup.WithContext 传递的 ctx 派生出的 ctx
		do something
})
```

设置最大并行数 `GOMAXPROCS` 对以上三种使用方式均起效。另外，由于 `errgroup` 实现问题, 设定 `GOMAXPROCS` 的 `errgroup` 需要立即调用 `Wait()`。如下:

```golang
g := errgroup.WithCancel(ctx)
	// 设置最大并发数为 2
g.GOMAXPROCS(2)
	//task1
g.Go(func(ctx context.Context) {
	fmt.Println("task1")
})
	//task2
g.Go(func(ctx context.Context) {
	fmt.Println("task2")
})
	//task3
g.Go(func(ctx context.Context) {
	fmt.Println("task3")
})
	//NOTE: 此时设置的 GOMAXPROCS 为 2, 添加了三个任务 task1, task2, task3 此时 task3 是不会运行的
	// 只有调用了 Wait task3 才有运行的机会
g.Wait()  //task3 运行
```


##  0x04	参考
-   [Kratos 的 errgroup 实现](https://github.com/go-kratos/kratos/blob/master/pkg/sync/errgroup/errgroup.go)、
-	[Goroutine 无法抛错就用 errgroup](https://driverzhang.github.io/post/goroutine%E6%97%A0%E6%B3%95%E6%8A%9B%E9%94%99%E5%B0%B1%E7%94%A8errgroup/)
-   [package errgroup](https://godoc.org/golang.org/x/sync/errgroup)
-	[Kratos errgroup test](https://github.com/go-kratos/kratos/blob/master/pkg/sync/errgroup/example_test.go)