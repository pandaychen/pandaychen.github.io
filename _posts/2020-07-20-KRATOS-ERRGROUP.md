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
---


##  0x00    前言
问题：调用并发的 goroutine 去批量执行 Job，有失败的可能，如何把第一个出错的 goroutine 信息返回给调用端。通用的做法是协程池，使用 channel 作为错误传出的媒介，但是有没有更简单的方法呢？答案就是 `errgroup`

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
-   `cancel func()`：这里保存的是 contex.WithCancel 返回的 `cancel` 参数，用以取消所有子协程的运行

总结: `wg` 用来保证在所有 goroutine 结束后返回 `err` 的值。在对 `err` 赋值的时候，只取第一个 `err` 值，这里使用到 `sync.Once` 保证。


####	WithContext 方法
该方法，主要是初始化 `Group` 对象，通过 `context.WithCancel(ctx)` 创建 `context` 树状结构，`cancel` 保存在 `Group.cancel` 中：

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
2.	当 f() 出错时，进入错误处理收集的逻辑，使用 `g.errOnce.DO()` 保证回调函数只会被执行一次
	-	将一个错误收集起来 g.err = err
	-	如果设置了 `cancel`，那么同时关闭 `g.cancel()`

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


##  0x03	参考
-   [Kratos 的 errgroup 实现](https://github.com/go-kratos/kratos/blob/master/pkg/sync/errgroup/errgroup.go)