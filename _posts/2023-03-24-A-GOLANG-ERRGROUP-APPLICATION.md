---
layout: post
title: Golang 并发：如何优雅处理并发 goroutine 中的错误等若干细节
subtitle: Golang errorgroup 应用（续）
date: 2023-03-25
author: pandaychen
catalog: true
tags:
  - Golang
---


##  0x00    前言

先看下经典的例子：
```GO
func main(){
  wg := sync.WaitGroup{}
  for i:=0; i<10; i++ {
     wg.Add(1)
     go func() {
        defer wg.Done()
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