---
layout: post
title: 分布式链路追踪（OpenTracing）之应用篇
subtitle: 使用 http/gin 构建 OpenTracing 机制
date: 2022-06-01
author: pandaychen
catalog: true
tags:
  - OpenTracing
  - Jaeger
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

1.  初始化一个 tracer
2.  记录一个简单的 span
3.  在 span 上添加注释信息（KV）

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
        // LogFields 和 LogKV 底层是调用的同一个方法
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

看一下后台的调用链, 可以看到它在调用其他服务的时候，是将 trace 相关的信息都放到 header 中， 通过 `span.Tracer().Inject` 方法注入；然后对接的服务就会在 header 中将 trace 信息取出来， 通过 `tracer.Extract` 方法导出；本质还是通过 head 中的特定 kv （`UBER-TRACE-ID`）来做，如下图：

![extract-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/tracing/extract-1.png)

![extract-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/tracing/extract-2.png)


####   demo4：baggage
通过 baggage 方法，可以在 span 中存储参数，然后该参数会跟着 span 传递到整个 trace；只需要修改一个地方就可以在整个 trace 中获取到该参数，而不用修改 trace 中的每一个地方

```GO
// after starting the span
span.SetBaggageItem("greeting", greeting)   // 客户端存入参数
greeting := span.BaggageItem("greeting")    // 服务端获取
```

##  0x02    汇总一些方法论

####    进程内：使用 ctx 来包装 tracer

使用 ctx 包装 tracer 要注意下面几点：
-   通过 `opentracing.ChildOf(rootSpan.Context())` 保留 span 之间的因果关系
-   通过 ctx 来实现在各个功能函数之间传递 span


1、span 因果关系

由于 span 是链路追踪里的最小组成单元，为了保留各个功能之间的因果关系，必须在各个方法之间传递 span 并且新建 span 时指定 `opentracing.ChildOf(rootSpan.Context())`，否则新建的 span 会是独立的，无法构成一个完整的 trace；比如方法 A 调用了 B、C、D，那么就需要将方法 A 中的 span 传递到方法 B/C/D 中。通过 `opentracing.ChildOf(rootSpan.Context())` 建立两个 span 之间的引用关系，如果不指定则会创建一个新的 span（在 jaeger-UI 中查看的时候就是一个新的 trace）
`opentracing.StartSpan` 方法的第二个参数 `opentracing.ChildOf` 可接受一个 `opentracing.Span` 对象，表示当前 Span 的父 Span。如果将当前 Span 的父 Span 设置为当前上下文中的 Span，那么就可以在当前 Span 上创建一个子 Span，而不是创建一个新的 Span


参考上述的 demo2 例子，利用 span 方式传递的代码如下：
```GO
childSpan := rootSpan.Tracer().StartSpan(
    "formatString",
    opentracing.ChildOf(rootSpan.Context()),
)

// 另一种方法
parentSpan := opentracing.SpanFromContext(ctx)
childSpan := opentracing.StartSpan("child", opentracing.ChildOf(parentSpan.Context()))
defer childSpan.Finish()
```

```GO
func main() {
	var (
        service_name = "example1"
        span_name = "say-hello"
        helloTo =  "pandaychen"
    )

	// 1. 初始化 tracer
	tracer, closer := config.NewTracer("hello")
	defer closer.Close()
	// 2. 开始新的 Span （注意: 必须要调用 Finish() 方法 span 才会上传到后端）
	span := tracer.StartSpan("say-hello")
	defer span.Finish()

    // 通过传递父 span 来实现子 span 的生成
	helloStr := formatString(span, helloTo)
	printHello(span, helloStr)
}

func formatString(span opentracing.Span, helloTo string) string {
	childSpan := span.Tracer().StartSpan(
		"formatString",
		opentracing.ChildOf(span.Context()),
	)
	defer childSpan.Finish()

	return fmt.Sprintf("Hello, %s!", helloTo)
}

func printHello(span opentracing.Span, helloStr string) {
	childSpan := span.Tracer().StartSpan(
		"printHello",
		opentracing.ChildOf(span.Context()),
	)
	defer childSpan.Finish()

	println(helloStr)
}
```


2、通过 ctx 传递 span

前述保留的 span 的因果关系，但是需要在各个方法中传递 span 的方式，缺点是可能会污染整个程序，可以借助 Go 的 `context.Context` 对象来进行传递，即 demo2 的方式：

```GO
func main()
    //...
    ctx := context.Background()
    ctx = opentracing.ContextWithSpan(ctx, span)    //span 与 ctx 打包
    helloStr := formatString(ctx, helloTo)
    printHello(ctx, helloStr)
    //...
}

func formatString(ctx context.Context, helloTo string) string {
    span, _ := opentracing.StartSpanFromContext(ctx, "formatString")
    defer span.Finish()
    ...

func printHello(ctx context.Context, helloStr string) {
    span, _ := opentracing.StartSpanFromContext(ctx, "printHello")
    defer span.Finish()
    ...
```

特别注意的是：

1.  `opentracing.StartSpanFromContext()` 返回的第二个参数是子 ctx, 如果需要的话可以将该子 ctx 继续往下传递，而不是传递父 ctx
2.  `opentracing.StartSpanFromContext()` 默认使用 `GlobalTracer` 来开始一个新的 span，所以使用之前需要设置 `GlobalTracer`，即 `opentracing.SetGlobalTracer(tracer)`

####    进程间：追踪 RPC

通过 `Inject(spanContext, format, carrier)` 和 `Extract(format, carrier)` 来实现在 RPC 调用中传递上下文。参数 `format` 为编码方式，由 OpenTracing API 定义，具体如下：

-   TextMap–span 上下文被编码为字符串键 - 值对的集合
-   Binary–span 上下文被编码为字节数组
-   HTTPHeaders–span 上下文被作为 HTTPHeader

carrier 则是底层实现的抽象：比如 TextMap 的实现则是一个包含 `Set(key, value)` 函数的接口。Binary 则是 `io.Writer` 接口

参考 Demo3 的实现，客户端通过 `Inject` 方法将 span 注入到 `req.Header` 中去，随着请求发送到服务端。服务端则通过 `Extract` 方法，解析请求头中的 span 信息

##  0x03    一种封装的思路

####    封装 1：基于 span 传递（No `context.Context`）
[博客](http://kebingzao.com/2020/12/25/jaeger-use/) 基于 `TextMapCarrier` 给出了一种典型的封装接口，不使用 `context.Context`，主要代码如下：

```go
var (
    Tracer opentracing.Tracer
    Closer io.Closer
)

func InitTracer(serviceName, agentHostPort string) error {
    if Tracer != nil && Closer != nil {
        return nil
    }

    cfg := jaegerConfig.Configuration{
        Sampler: &jaegerConfig.SamplerConfig{
            Type: "const",
            Param: 1,
        },
        Reporter: &jaegerConfig.ReporterConfig{
            LogSpans: false,
            LocalAgentHostPort: agentHostPort,
        },
        ServiceName: serviceName,
    }

    _tracer, _closer, err := cfg.NewTracer()
    if err != nil {
        fmt.Println("Init GlobalJaegerTracer failed, err : %v", err)
        return err
    }

    Tracer = _tracer
    Closer = _closer
    return nil
}

func getParentSpan(operationName, traceId string, startIfNoParent bool) (span opentracing.Span, err error) {
    if Tracer == nil {
        err = errors.New("jaeger tracing error : Tracer is nil")
        fmt.Println(err)
        return
    }

    parentSpanCtx, err := Tracer.Extract(opentracing.TextMap, opentracing.TextMapCarrier{"UBER-TRACE-ID":traceId})
    if err != nil {
        if startIfNoParent {
            //fmt.Println("startIfNoParent","create new:",err)
            span = Tracer.StartSpan(operationName)  // 如果 span 不存在，则新建
        }
    } else {
        span = Tracer.StartSpan(operationName, opentracing.ChildOf(parentSpanCtx))
    }

    err = nil
    return
}

func StartSpan(operationName, parentSpanTraceId string, startIfNoParent bool) (span opentracing.Span, spanTraceId string, err error) {
    plainParentSpan, err := getParentSpan(operationName, parentSpanTraceId, startIfNoParent)
    if err != nil || plainParentSpan == nil {
        fmt.Println("No span return")
        return
    }

    m := map[string]string{}
    carrier := opentracing.TextMapCarrier(m)
    err = Tracer.Inject(plainParentSpan.Context(), opentracing.TextMap, carrier)
    if err != nil {
        fmt.Println("jaeger tracing inject error :", err)
        return
    }

    spanTraceId = carrier["uber-trace-id"]
    span = plainParentSpan
    return
}

func FinishSpan(span opentracing.Span) {
    if span != nil {
        span.Finish()
    }
}
```

封装的核心点在于使用 `opentracing.TextMapCarrier`、`Inject` 和 `Extract` 把 `traceId` 与 span 包装起来；客户端第一次请求的时候，由于是直接命令行调用，所以 `Extract` 会失败，同时创建一个新的 rootspan：

```
startIfNoParent create new: Cannot convert empty string to tracer state
```


####    封装 2：基于 context（gin）
[OpenTracing middleware for gin-gonic](https://github.com/opentracing-contrib/go-gin/blob/master/ginhttp/server.go) 提供了 gin 中间件的一种实现，核心代码如下：

```go
// Middleware is a gin native version of the equivalent middleware in:
//   https://github.com/opentracing-contrib/go-stdlib/
func Middleware(tr opentracing.Tracer, options ...MWOption) gin.HandlerFunc {
	opts := mwOptions{
		opNameFunc: func(r *http.Request) string {
			return "HTTP" + r.Method
		},
		spanObserver: func(span opentracing.Span, r *http.Request) {},
		urlTagFunc: func(u *url.URL) string {
			return u.String()
		},
	}
	for _, opt := range options {
		opt(&opts)
	}

	return func(c *gin.Context) {
        // 从 HTTP 请求头中提取 SpanContext
		carrier := opentracing.HTTPHeadersCarrier(c.Request.Header)
		ctx, _ := tr.Extract(opentracing.HTTPHeaders, carrier)  //why ignore error?
        if err != nil && err != opentracing.ErrSpanContextNotFound {
            log.Printf("Failed to extract SpanContext: %v", err)
        }
		op := opts.opNameFunc(c.Request)

        // 创建 Span
		sp := tr.StartSpan(op, ext.RPCServerOption(ctx))
		ext.HTTPMethod.Set(sp, c.Request.Method)
		ext.HTTPUrl.Set(sp, opts.urlTagFunc(c.Request.URL))
		opts.spanObserver(sp, c.Request)

		// set component name, use "net/http" if caller does not specify
		componentName := opts.componentName
		if componentName == "" {
			componentName = defaultComponentName
		}
		ext.Component.Set(sp, componentName)

        // 注意：将 context 更新上下文中
        // 将上下文设置到请求中
		c.Request = c.Request.WithContext(
            // 将 Span 注入到上下文中
			opentracing.ContextWithSpan(c.Request.Context(), sp))

        // NEXT
		c.Next()

		ext.HTTPStatusCode.Set(sp, uint16(c.Writer.Status()))
		sp.Finish()
	}
}
```

上述代码比较直观了，有一点不理解的是调用 `tr.Extract` 的返回值 err 为何不处理？有一种解释是如果选择创建一个新的 Root Span，那么新的 Root Span 将不会与之前的 Root Span 相关联，这可能会导致分布式追踪的效果不佳。因此，如果我们希望在出现错误时仍然能够进行分布式追踪，最好的做法是忽略该错误，继续使用当前的 Root Span

个人认为较合理的封装，见此 []()

如果 `SpanContext` 为 `nil`，那么说明该请求没有携带有效的 `SpanContext`，此时可以创建一个新的 Root Span，并将其注入到上下文中。如下：
```GO
span := tracer.StartSpan(c.Request.URL.Path)    // 使用当前请求的 URL Path 作为新的 Span 的操作名称
defer span.Finish()

// 将 Span 注入到上下文中
ctx := opentracing.ContextWithSpan(c.Request.Context(), span)

// 将上下文设置到请求中
c.Request = c.Request.WithContext(ctx)
```
此外，需要注意的是，在创建新的 Root Span 时，需要根据具体的业务需求和设计来确定操作名称和其他属性。例如，可以使用请求的 URL Path、HTTP 方法、请求参数等信息作为操作名称和标签，以便更好地跟踪服务的执行过程

####    封装 3：基于 context（gRPC）


##  0x04    其他细节

carrier 判断使用哪种服务，决定使用哪种方法从请求中获取上下文；例如，web 服务通过 HTTP 头作为 carrier，从 HTTP 请求中获 span 上下文（如下所示）：



##  0x05    参考
- [OpenTracing Tutorial - Go](https://github.com/yurishkuro/opentracing-tutorial/tree/master/go)
- [APM 原理与框架选型](https://www.cnblogs.com/xiaoqi/p/apm.html)
-   [调用链追踪系统在伴鱼：OpenTelemetry 最佳实践案例分享](https://www.infoq.cn/article/2gao0xpw37arecbycpxl)
-   [基于 jaeger 微服务调用链实现方案](http://kebingzao.com/2020/12/25/jaeger-use/)
-   [Learning Go by examples: part 10 - Instrument your Go app with OpenTelemetry and send traces to Jaeger - Distributed Tracing](https://dev.to/aurelievache/learning-go-by-examples-part-10-instrument-your-go-app-with-opentelemetry-and-send-traces-to-jaeger-distributed-tracing-1p4a)
-   [分布式链路追踪教程 (三)---Jaeger 简单使用](https://www.lixueduan.com/posts/tracing/03-jaeger-quick-start/)
-   [分布式链路追踪（OpenTracing 标准）和 Jaeger 实现](https://xiaoming.net.cn/2021/03/25/Opentracing%E6%A0%87%E5%87%86%E5%92%8CJaeger%E5%AE%9E%E7%8E%B0/)
-   [Opentracing & jaeger 源码学习](https://freephenix.github.io/jaegeropentracing-yuan-ma-xue-xi/)
-   [腾讯云文档：通过 gin Jaeger 中间件上报](https://cloud.tencent.com/document/product/1463/57863)
-   [Golang gRPC Middlewares: interceptor chaining, auth, logging, retries and more.](https://github.com/grpc-ecosystem/go-grpc-middleware)