---
layout:     post
title:      Kratos 源码分析：Tracing （二）
subtitle:   分析 Kratos 的 opentracing 实现：应用
date:       2020-10-10
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
前一篇文章 [Kratos 源码分析：Tracing （一）](https://pandaychen.github.io/2020/10/10/KRATOS-OPENTRACING-ANALYSIS-1/) 大致介绍了 Kratos 的 Opentracing 的实现，这篇文章看下如何在项目中使用 Tracing。


##	0x01	使用 zipkin 上报
上一节提到，如果需要使用 zipkin 方式上报 tracing 数据，则需要初始化时，执行 `GlobalTracer` 为 zipkin 方式，代码在 [`zipkin.Init()` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/zipkin/config.go#L29) 中，此一段代码就完成了神奇的功能，准确的说是 `trace.SetGlobalTracer()` 方法：
```golang
// Init init trace report.
func Init(c *Config) {
	if c.BatchSize == 0 {
		c.BatchSize = 100
	}
	if c.Timeout == 0 {
		c.Timeout = xtime.Duration(200 * time.Millisecond)
	}
	// 设置全局 tracer 为 zipkin 方式
	trace.SetGlobalTracer(trace.NewTracer(env.AppID, newReport(c), c.DisableSample))
}
```

上面这段代码中的 `newReport` 方法是使用 zipkin 初始化的方式，这样，我们的 Tracing 部分的上报接口就完全使用 zipkin 完成了，这就是接口统一的好处：
```golang
func newReport(c *Config) *report {
	return &report{
		rpt: http.NewReporter(c.Endpoint,
			http.Timeout(time.Duration(c.Timeout)),
			http.BatchSize(c.BatchSize),
		),
	}
}
```

####	业务代码嵌入
可以看 [`Zipkin`](https://github.com/bilibili/kratos/tree/master/pkg/net/trace/zipkin) 的协议上报实现，具体使用方式如下：

1. 搭建可用 `Zipkin` 集群
2. 在业务代码的 `main` 函数内进行初始化，如下：

```golang
import "github.com/bilibili/kratos/pkg/net/trace/zipkin"

func main(){
	......
	// 在代码中加入这行，表示使用 zipkin 方式上报 tracer 日志
    zipkin.Init(&zipkin.Config{
        Endpoint: "http://localhost:9411/api/v2/spans",
    })
    ......
}
```

接下来，我们通过 RPC 和 HTTP 两种方式来分析下项目中如何调用 Tracing 逻辑。

##	0x02	Warden 的 RPC 调用链


##	0x03	Bm 的调用链（HTTP）


##  0x04  参考
-   [OpenTracing 语义标准规范及实现](https://www.jianshu.com/p/a963ad0bbe3e)
-   [OpenTracing 语义标准](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/specification.md)
-   [开放分布式追踪（OpenTracing）入门与 Jaeger 实现](https://zhuanlan.zhihu.com/p/34318538)
-   [opentracing](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/specification.md)
-   [dapper](https://bigbully.github.io/Dapper-translation/)
-   [bilibili 毛剑 - B 站微服务链路监控实践](https://myslide.cn/slides/8297)
-	[Open-zipkin：b3-propagation](https://github.com/openzipkin/b3-propagation)