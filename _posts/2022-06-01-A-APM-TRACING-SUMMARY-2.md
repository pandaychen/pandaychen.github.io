---
layout: post
title: 分布式链路追踪（OpenTracing）之应用篇
subtitle: 使用 http/gin 构建 OpenTracing 机制
date: 2022-06-01
author: pandaychen
catalog: true
tags:
  - OpenTracing
---

##  0x00    前言
前文 [微服务基础之链路追踪（OpenTracing）](https://pandaychen.github.io/2020/02/23/OPENTRACING-INTRO/) 介绍了分布式链路追踪的理论知识，本文基于 `net/http`、gin 等库来构建客户端到服务端的 OpenTracing 实践（使用 `jaeger` 接入）

APM 就是跟踪一个 traceId 在多个微服务中的传递并记录。在进入第一个服务的时候，就生成一个 traceId，接下来这个 traceId 将跟随整个微服务调用链，一直到整个调用链结束， 后面只需要分析这个 traceId 记录的服务和时间， 就可以知道在哪些服务消耗了多少时间，总的用了多少时间。

####    tracing 回顾

一个父级的 span 会显示的并行或者串行启动多个子 span。一个 tracer 过程中，各 span 的关系可能如下：

```
        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C 是 Span A 的孩子节点, ChildOf)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G 在 Span F 后被调用, FollowsFrom)
```

相应的 tracer 与 span 的时间轴关系如下：

```
––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
```


假设服务调用关系为 `a->b->c->d`，请求从 a 开始发起。 那么 a 负责生成 traceId，并在调用 b 的时候把 traceId 传递给 b，以此类推，traceId 会从 a 层层传递到 d；traceId 形如： `{root span id}:{this span id}:{parent span id}:{flag}` 。假设 a 是请求发起者，flag 固定为 1，那么 a,b,c,d 的 traceId 分别是：

```
a_span_id:a_span_id:0:1
a_span_id:b_span_id:a_span_id:1
a_span_id:c_span_id:b_span_id:1
a_span_id:d_span_id:c_span_id:1
```

##  0x01    基础用例

####    demo 1：basic

最基础的例子，调用 `StartSpan` 创建，调用 `Finish` 结束一个 span
```go
import (
    "fmt"
    "io"

    opentracing "github.com/opentracing/opentracing-go"
    jaeger "github.com/uber/jaeger-client-go"
    config "github.com/uber/jaeger-client-go/config"
)

// Init returns an instance of Jaeger Tracer that samples 100% of traces and logs all spans to stdout.
func Init(service string) (opentracing.Tracer, io.Closer) {
    cfg := &config.Configuration{
        ServiceName: service,
        Sampler: &config.SamplerConfig{
            Type:  "const",
            Param: 1,
        },
        Reporter: &config.ReporterConfig{
            CollectorEndpoint: http://localhost:xxxxx/api/traces, // 将 span 发往 jaeger-collector 的服务地址
            LogSpans: true,
        },
    }
    tracer, closer, err := cfg.NewTracer(config.Logger(jaeger.StdLogger))
    if err != nil {
        panic(fmt.Sprintf("ERROR: cannot init Jaeger: %v\n", err))
    }
    return tracer, closer
}

func main() {
    var (
        service_name = "example1"
        span_name = "say-hello"
        helloTo =  "pandaychen"
    )
    tracer, closer := Init(service_name)
    defer closer.Close()

    span := tracer.StartSpan(span_name)
    span.SetTag("hello-to", helloTo)

    helloStr := fmt.Sprintf("Hello, %s!", helloTo)
    span.LogFields(
        log.String("event", "string-format"),
        log.String("value", helloStr),
    )

    println(helloStr)
    span.LogKV("event", "println")

    span.Finish()
}
```


####    Demo2：Context and Tracing Functions（进程内）

1、主调方法中，生成 ctx（root）和 span，再通过 `opentracing.ContextWithSpan` 生成新的 context
2、被调用方法中，使用 `opentracing.StartSpanFromContext` 和上一步得到的 context，生成新的 span

本例子生成的 root span， 此外还多出两个 child span， 分别表示这个服务的两个重要操作 `formatString` 和 `printHello`；这些 child span 对应的这个 service 服务中的一些重要操作， 大部分都是需要耗时的操作，比如 文件 io 、数据库读写等操作

```GO
func main() {
    var (
        service_name = "example1"
        span_name = "say-hello"
        helloTo =  "pandaychen"
    )

    tracer, closer := tracing.Init(service_name)
    defer closer.Close()
    opentracing.SetGlobalTracer(tracer)     // 设置全局 tracer

    span := tracer.StartSpan(span_name)
    span.SetTag("hello-to", helloTo)
    defer span.Finish()

    ctx := opentracing.ContextWithSpan(context.Background(), span)

    helloStr := formatString(ctx, helloTo)
    printHello(ctx, helloStr)
}

func formatString(ctx context.Context, helloTo string) string {
    // 注意
    span, _ := opentracing.StartSpanFromContext(ctx, "formatString")
    defer span.Finish()

    helloStr := fmt.Sprintf("Hello, %s!", helloTo)
    span.LogFields(
        log.String("event", "string-format"),
        log.String("value", helloStr),
    )

    return helloStr
}

func printHello(ctx context.Context, helloStr string) {
    span, _ := opentracing.StartSpanFromContext(ctx, "printHello")
    defer span.Finish()

    println(helloStr)
    span.LogKV("event", "println")
}
```

####    demo3：Tracing RPC Requests（跨进程）
本例子演示在多个微服务的调用链的如何实现，基于 demo2 `formatString` 和 `printHello` 会再去请求另外两个服务 `service-formatter` 和 `service-publisher`，如下：

1、客户端代码，注意 `formatString` 与 `printHello` 的实现

```GO
func main() {
    var (
        service_name = "example1"
        span_name = "say-hello"
        helloTo =  "pandaychen"
    )

    tracer, closer := tracing.Init(service_name)
    defer closer.Close()
    opentracing.SetGlobalTracer(tracer)

    span := tracer.StartSpan(span_name)
    span.SetTag("hello-to", helloTo)
    defer span.Finish()

    ctx := opentracing.ContextWithSpan(context.Background(), span)

    helloStr := formatString(ctx, helloTo)
    printHello(ctx, helloStr)
}

func formatString(ctx context.Context, helloTo string) string {
    // 依然还是 StartSpanFromContext
    span, _ := opentracing.StartSpanFromContext(ctx, "formatString")
    defer span.Finish()

    v := url.Values{}
    v.Set("helloTo", helloTo)
    url := "http://localhost:8081/format?" + v.Encode()
    req, err := http.NewRequest("GET", url, nil)
    if err != nil {
        panic(err.Error())
    }

    ext.SpanKindRPCClient.Set(span)     // 调用 ext 包对 span 做一些设置
    ext.HTTPUrl.Set(span, url)
    ext.HTTPMethod.Set(span, "GET")
    span.Tracer().Inject(               //inject
        span.Context(),
        opentracing.HTTPHeaders,
        opentracing.HTTPHeadersCarrier(req.Header),
    )

    resp, err := xhttp.Do(req)
    if err != nil {
        ext.LogError(span, err)
        panic(err.Error())
    }

    helloStr := string(resp)

    span.LogFields(
        log.String("event", "string-format"),
        log.String("value", helloStr),
    )

    return helloStr
}

func printHello(ctx context.Context, helloStr string) {
    span, _ := opentracing.StartSpanFromContext(ctx, "printHello")
    defer span.Finish()

    v := url.Values{}
    v.Set("helloStr", helloStr)
    url := "http://localhost:8082/publish?" + v.Encode()
    req, err := http.NewRequest("GET", url, nil)
    if err != nil {
        panic(err.Error())
    }

    ext.SpanKindRPCClient.Set(span)
    ext.HTTPUrl.Set(span, url)
    ext.HTTPMethod.Set(span, "GET")
    span.Tracer().Inject(span.Context(), opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(req.Header))

    if _, err := xhttp.Do(req); err != nil {
        ext.LogError(span, err)
        panic(err.Error())
    }
}
```

2、服务端代码 `formatter`

```GO
func main() {
    tracer, closer := tracing.Init("formatter") // 服务名不一定要一样
    defer closer.Close()

    http.HandleFunc("/format", func(w http.ResponseWriter, r *http.Request) {
        //Extract
        spanCtx, _ := tracer.Extract(opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(r.Header))
        span := tracer.StartSpan("format", ext.RPCServerOption(spanCtx))
        defer span.Finish()

        helloTo := r.FormValue("helloTo")
        helloStr := fmt.Sprintf("Hello, %s!", helloTo)
        span.LogFields(
            otlog.String("event", "string-format"),
            otlog.String("value", helloStr),
        )

        //HOW TO create another span?

        w.Write([]byte(helloStr))
    })

    log.Fatal(http.ListenAndServe(":8081", nil))
}
```

3、服务端代码 `publisher`

```GO
func main() {
    tracer, closer := tracing.Init("publisher")
    defer closer.Close()

    http.HandleFunc("/publish", func(w http.ResponseWriter, r *http.Request) {
        spanCtx, _ := tracer.Extract(opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(r.Header))
        span := tracer.StartSpan("publish", ext.RPCServerOption(spanCtx))
        defer span.Finish()

        helloStr := r.FormValue("helloStr")
        println(helloStr)
    })

    log.Fatal(http.ListenAndServe(":8082", nil))
}
```

看一下后台的调用链, 可以看到它在调用其他服务的时候，是将 trace 相关的信息都放到 header 中， 通过 `span.Tracer().Inject` 方法注入；然后对接的服务就会在 header 中将 trace 信息取出来， 通过 `tracer.Extract` 方法导出；本质还是通过 head 中的特定 kv 来做，如下图：

![extract-1]()
![extract-1]()




##  0x04    参考
- [OpenTracing Tutorial - Go](https://github.com/yurishkuro/opentracing-tutorial/tree/master/go)
- [APM 原理与框架选型](https://www.cnblogs.com/xiaoqi/p/apm.html)
-   [调用链追踪系统在伴鱼：OpenTelemetry 最佳实践案例分享](https://www.infoq.cn/article/2gao0xpw37arecbycpxl)
-   [基于 jaeger 微服务调用链实现方案](http://kebingzao.com/2020/12/25/jaeger-use/)
-   [Learning Go by examples: part 10 - Instrument your Go app with OpenTelemetry and send traces to Jaeger - Distributed Tracing](https://dev.to/aurelievache/learning-go-by-examples-part-10-instrument-your-go-app-with-opentelemetry-and-send-traces-to-jaeger-distributed-tracing-1p4a)
-   [分布式链路追踪教程 (三)---Jaeger 简单使用](https://www.lixueduan.com/posts/tracing/03-jaeger-quick-start/)