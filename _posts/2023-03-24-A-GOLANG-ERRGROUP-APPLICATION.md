---
layout: post
title: Golang 并发：如何优雅实现并发 goroutine 若干细节
subtitle: Golang errorgroup 应用（续）
date: 2023-03-25
author: pandaychen
catalog: true
tags:
  - Golang
---


##  0x00    前言


##  0x01  SizedWaitGroup机制
`SizedWaitGroup`在`sync.WaitGroup`基础上增加了并发启动的goroutines的数量限制特性，即`SizedWaitGroup`增加了限制同时启动例程的最大数量的功能，使用方法如下：

```GO
import (
        "fmt"
        "math/rand"
        "time"

        "github.com/remeh/sizedwaitgroup"
)

func main() {
        rand.Seed(time.Now().UnixNano())

        // Typical use-case:
        // 50 queries must be executed as quick as possible
        // but without overloading the database, so only
        // 8 routines should be started concurrently.
        swg := sizedwaitgroup.New(8)  //仅仅限制8个并发
        for i := 0; i < 50; i++ {
                swg.Add()
                go func(i int) {
                        defer swg.Done()
                        query(i)
                }(i)
        }

        swg.Wait()
}

func query(i int) {
        fmt.Println(i)
        ms := i + 500 + rand.Intn(500)
        time.Sleep(time.Duration(ms) * time.Millisecond)
}
```

####  实现分析
```GO
// SizedWaitGroup has the same role and close to the
// same API as the Golang sync.WaitGroup but adds a limit of
// the amount of goroutines started concurrently.
type SizedWaitGroup struct {
	Size int

	current chan struct{}
	wg      sync.WaitGroup
}

// New creates a SizedWaitGroup.
// The limit parameter is the maximum amount of
// goroutines which can be started concurrently.
func New(limit int) SizedWaitGroup {
	size := math.MaxInt32 // 2^31 - 1
	if limit > 0 {
		size = limit
	}
	return SizedWaitGroup{
		Size: size,

		current: make(chan struct{}, size),
		wg:      sync.WaitGroup{},
	}
}

// Add increments the internal WaitGroup counter.
// It can be blocking if the limit of spawned goroutines
// has been reached. It will stop blocking when Done is
// been called.
//
// See sync.WaitGroup documentation for more information.
func (s *SizedWaitGroup) Add() {
	s.AddWithContext(context.Background())
}

// AddWithContext increments the internal WaitGroup counter.
// It can be blocking if the limit of spawned goroutines
// has been reached. It will stop blocking when Done is
// been called, or when the context is canceled. Returns nil on
// success or an error if the context is canceled before the lock
// is acquired.
//
// See sync.WaitGroup documentation for more information.
func (s *SizedWaitGroup) AddWithContext(ctx context.Context) error {
	select {
	case <-ctx.Done():
		return ctx.Err()
	case s.current <- struct{}{}:
		break
	}
	s.wg.Add(1)
	return nil
}

// Done decrements the SizedWaitGroup counter.
// See sync.WaitGroup documentation for more information.
func (s *SizedWaitGroup) Done() {
	<-s.current
	s.wg.Done()
}
```


##  0x02  errgroup

先看下经典的例子：
```GO
func main() {
        wg := sync.WaitGroup{}
        for i := 0; i < 10; i++ {
                wg.Add(1)
                go func() {
                        defer wg.Done()
                }()
        }
        wg.Wait()
}
```

上面例子中，想要知道某个 goroutine 报什么错误的时候发现很难，因为是直接 `go func(){}` 出去的，并没有返回值，因此对需要接受返回值做进一步处理的需求就无法满足了，那如何做呢？

前文 [Kratos 源码分析：Errgroup 机制](https://pandaychen.github.io/2020/07/20/KRATOS-ERRGROUP/) 介绍了应对这种方法的处理一种方式，那就是 `errgroup` 机制，先简单回顾下。



##  0x03  参考
-   [聊聊 ErrorGroup 的用法和拓展](https://marksuper.xyz/2021/10/15/error_group/)
-   [Handling errors concurrently in Go](https://bostonc.dev/blog/go-errgroup)