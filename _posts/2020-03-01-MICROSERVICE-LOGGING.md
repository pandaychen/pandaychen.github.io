---
layout:     post
title:      gRPC 微服务构建之日志（Logging）
subtitle:
date:       2020-03-01
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Zap
---

##  0x00    前言
&emsp;&emsp; 作为一个多年的后台开发人员，（结构化）日志对于后台项目的重要性不言而喻。通常调试、记录运行错误及基于日志关键字的监控都会需要后台提供足够充分的日志。Golang 中有非常多优秀的日志库，如 [Zap](https://github.com/uber-go/zap) 和 [Logus](https://github.com/sirupsen/logrus) 等。这篇文章分享下在项目中，如何将 Zap 和 gRPC 完美的融合在一起，保证日志的可读性和高效，同时也兼顾的性能。

##  0x01    日志的选择参数
Zap 库满足了常见日志库的所有优点，非常适合在项目中使用。具体来说：
-   日志的格式化：比如以 JSON、自定义分隔符等形式记录，方便人为观察及日志分析工具处理
-   性能：每次操作的耗时，以及每次操作分配的内存，作为日志库，两个指标都应该要极低．
-   日志分级
-   按天切分 / 按大小切分 `+` 自动（定期）归档（Zap 本身不具备归档能力，需要借助如 [lumberjack](https://github.com/natefinch/lumberjack) 来实现）
-   支持采样：全日志记录，支持采样率，防止服务请求剧增导致时输出的日志量剧增

##  0x02    gRPC 和 Zap 打包
&emsp;&emsp; gRPC 的 grpclog 包，提供了 [LoggerV2](https://godoc.org/google.golang.org/grpc/grpclog#LoggerV2) 的 `interface{}` 定义，[代码在此](https://github.com/grpc/grpc-go/blob/master/grpclog/loggerv2.go)。因此，只要通过 Zap 实现 LoggerV2 的接口（用 Zap 实例化 grpc.LoggerV2），并通过 `SetLoggerV2(l LoggerV2)` 将实现的对象设置到 grpclog 包中，那么 gRPC 就会默认使用我们传入的 Zap 的 logger 进行日志打印，非常完美。

LoggerV2 的接口定义了下面的方法集合，所以在我们自己的 Zap 库中，对这里所有的方法都要进行二次封装。
```go
type LoggerV2 interface {
    // Info logs to INFO log. Arguments are handled in the manner of fmt.Print.
    Info(args ...interface{})
    // Infoln logs to INFO log. Arguments are handled in the manner of fmt.Println.
    Infoln(args ...interface{})
    // Infof logs to INFO log. Arguments are handled in the manner of fmt.Printf.
    Infof(format string, args ...interface{})
    // Warning logs to WARNING log. Arguments are handled in the manner of fmt.Print.
    Warning(args ...interface{})
    // Warningln logs to WARNING log. Arguments are handled in the manner of fmt.Println.
    Warningln(args ...interface{})
    // Warningf logs to WARNING log. Arguments are handled in the manner of fmt.Printf.
    Warningf(format string, args ...interface{})
    // Error logs to ERROR log. Arguments are handled in the manner of fmt.Print.
    Error(args ...interface{})
    // Errorln logs to ERROR log. Arguments are handled in the manner of fmt.Println.
    Errorln(args ...interface{})
    // Errorf logs to ERROR log. Arguments are handled in the manner of fmt.Printf.
    Errorf(format string, args ...interface{})
    // Fatal logs to ERROR log. Arguments are handled in the manner of fmt.Print.
    // gRPC ensures that all Fatal logs will exit with os.Exit(1).
    // Implementations may also call os.Exit() with a non-zero exit code.
    Fatal(args ...interface{})
    // Fatalln logs to ERROR log. Arguments are handled in the manner of fmt.Println.
    // gRPC ensures that all Fatal logs will exit with os.Exit(1).
    // Implementations may also call os.Exit() with a non-zero exit code.
    Fatalln(args ...interface{})
    // Fatalf logs to ERROR log. Arguments are handled in the manner of fmt.Printf.
    // gRPC ensures that all Fatal logs will exit with os.Exit(1).
    // Implementations may also call os.Exit() with a non-zero exit code.
    Fatalf(format string, args ...interface{})
    // V reports whether verbosity level l is at least the requested verbose level.
    V(l int) bool
}
```

Zap 库中提供了两种日志记录器：Sugared Logger 和 Logger。


##  0x03    关于日志的一些细节
如何在高并发的实时系统中优化日志写入呢？在笔者之前的 DPDK 网络包处理项目中，总结了这几条经验：
1.  线程将待落地日志结构化（标识日志类型、等级、内容等）写入 RingBuffer，读端从 RingBuffer 中取出日志，落地写入
2.  单线程写日志
3.  在 golang 中，利用 单独的 goroutine 异步写日志

##  参考
-   [从 Go 高性能日志库 zap 看如何实现高性能 Go 组件](https://mp.weixin.qq.com/s/i0bMh_gLLrdnhAEWlF-xDw)
-   [在 Go 语言项目中使用 Zap 日志库](https://www.liwenzhou.com/posts/Go/zap/)