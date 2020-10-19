---
layout:     post
title:      gRPC 微服务构建之链路追踪（OpenTracing）
subtitle:   在 gRPC 服务中应用 OpenTracing：使用 Zipkin 和 Jaeger 进行路追踪
date:       2020-06-01
author:     pandaychen
catalog:    true
tags:
    - OpenTracing
    - gRPC
---


##  0x00    前言
前一篇文章 [微服务基础之 链路追踪（OpenTracing）](https://pandaychen.github.io/2020/02/23/OPENTRACING-INTRO/)，介绍了 OpenTracing 的理论，本文基于 gRPC 与 Zipkin && Jaeger 来实现 Tracing 的应用。

##  0x01    回顾 OpenTracing 数据模型
一个 Tracer 包含了若干个 Span，Span 是追踪链路中的基本组成元素，一个 Span 表示一个独立的工作单元，在链路追踪中可以表示一个接口的调用，一个数据库操作的调用等等。<br>
一个 Span 中包含如下内容：
-   服务名称 (operation name)
-   服务的开始和结束时间
-   Tags：k/v 形式
-   Logs：k/v 形式
-   SpanContext
-   Refrences：该 span 对一个或多个 span 的引用（通过引用 SpanContext）

详细说明下上面的字段：

1、Tags
Tags 是一个 K/V 类型的键值对，用户可以自定义该标签并保存。主要用于链路追踪结果对查询过滤。例如：·http.method="GET",http.status_code=200。其中 key 值必须为字符串，value 必须是字符串，布尔型或者数值型。span 中的 tag 仅自己可见，不会随着 SpanContext 传递给后续 span
```golang
span.SetTag("http.method","GET")
span.SetTag("http.status_code",200)
```

2、Logs
Logs 也是一个 K/V 类型的键值对，与 Tags 不同的是，Logs 还会记录写入 Logs 的时间，因此 Logs 主要用于记录某些事件发生的时间。
```golang
span.LogFields(
  			log.String("database","mysql"),
  			log.Int("used_time":5),
  			log.Int("start_ts":1596335100),
)
```
PS：Opentracing 给出了一些惯用的 Tags 和 Logs，[链接](https://github.com/opentracing/specification/blob/master/semantic_conventions.md)

3、SpanContext（核心字段）
SpanContext 携带着一些用于跨服务通信的（跨进程）数据，主要包含：
-   该 Span 的唯一标识信息，如：`span_id`、`trace_id`
-   Baggage Items，为整条追踪连保存跨服务（跨进程）的 K/V 格式的用户自定义数据

4、Baggage Items
Baggage Items 与 Tags 类似，也是 K/V 键值对。与 tags 不同的是：Baggage Items 的 Key 和 Value 都只能是 string 格式，Baggage items 不仅当前 Span 可见，其会随着 SpanContext 传递给后续所有的子 Span。要小心谨慎的使用 Baggage Items：因为在所有的 span 中传递这些 Key/Value 会带来不小的网络和 CPU 开销

5、References（引用关系）
Opentracing 定义了两种引用关系: ChildOf 和 FollowFrom，分别来看：
-	ChildOf: 父 Span 的执行依赖子 Span 的执行结果时，此时子 span 对父 span 的引用关系是 ChildOf。比如对于一次 RPC 调用，服务端的 Span（子 Span）与客户端调用的 Span（父 Span）是 ChildOf 关系。
-	FollowFrom：父 Span 的执不依赖子 Span 执行结果时，此时子 Span 对父 Span 的引用关系是 FollowFrom。FollowFrom 常用于异步调用的表示，例如消息队列中 Consumerspan 与 Producerspan 之间的关系。

6、Trace
Trace 表示一次完整的追踪链路，trace 由一个或多个 Span 组成。下图示例表示了一个由 8 个 Span 组成的 trace:

```javascript
        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `ChildOf` Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G `FollowsFrom` Span F)
```

以时间轴的展现方式如下：
```golang
––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
```


##  0x02	ZipKin-Tracing 的一般流程
Zipkin 是一款开源的分布式实时数据追踪系统（Distributed Tracking System），由 Twitter 公司开发和贡献。其主要功能是聚合来自各个异构系统的实时监控数据。在链路追踪 Tracing Analysis 中，可以通过 Zipkin 上报 Golang 应用数据。

使用 Zipkin 上报数据的流程如下图所示：
![img](https://aliware-images.oss-cn-hangzhou.aliyuncs.com/xtrace/xtrace_dg_report_via_zipkin.png)

使用的 package：
-   [openzipkin/zipkin-go](https://github.com/openzipkin/zipkin-go)

下面介绍通过 Zipkin 将 Golang 应用数据上报至链路追踪控制台的方法：
1、创建 Tracer，Tracer 对象可以用来创建 Span 对象（记录分布式操作时间）。Tracer 对象还配置了上报数据的网关地址、本机 IP、采样频率等数据，您可以通过调整采样率来减少因上报数据产生的开销。
```golang
func getTracer(serviceName string, ip string) *zipkin.Tracer {
  // create a reporter to be used by the tracer
  reporter := httpreporter.NewReporter("http://tracing-analysis-dc-hz.aliyuncs.com/adapt_aokcdqnxyz@123456ff_abcdef123@abcdef123/api/v2/spans")
  // set-up the local endpoint for our service
  endpoint, _ := zipkin.NewEndpoint(serviceName, ip)
  // set-up our sampling strategy 设置采样率
  sampler := zipkin.NewModuloSampler(1)
  // initialize the tracer
  tracer, _ := zipkin.NewTracer(
    reporter,
    zipkin.WithLocalEndpoint(endpoint),
    zipkin.WithSampler(sampler),
  )
  return tracer;
}
```

2、记录请求数据，下面代码用于记录请求的根操作：
```golang
	// tracer can now be used to create spans.
	span := tracer.StartSpan("some_operation")
	// ... do some work ...
	// span 完成，必须调用 finish
	span.Finish()
	// Output:
```

如果需要记录请求的上一步和下一步操作，则需要传入上下文。如下代码所示，`childSpan` 为 `span` 的孩子节点：
```golang
	childSpan := tracer.StartSpan("some_operation2", zipkin.Parent(span.Context()))
		// ... do some work ...
	childSpan.Finish()
```

3、可选：（为了快速定位问题）可以为某个记录添加一些自定义标签（Tags），例如记录是否发生错误、请求的返回值等：
```golang
childSpan.Tag("http.status_code", statusCode)
```

4、在分布式系统中发送 RPC 请求时会带上 Tracing 数据，包括 TraceId、ParentSpanId、SpanId、Sampled 等。可以在 HTTP 请求中使用 Extract/Inject 方法在 HTTP Request Headers 上透传数据。即 <font color="#dd0000"> 在 Client 端执行 `Inject`，在 Server 端执行 `Extract`</font>， 目前 Zipkin 已有组件支持以 HTTP、gRPC 这两种 RPC 协议透传 Context 信息。总体数据流程如下：

```javascript
   Client Span                                                Server Span
┌──────────────────┐                                       ┌──────────────────┐
│                  │                                       │                  │
│   TraceContext   │           Http Request Headers        │   TraceContext   │
│ ┌──────────────┐ │          ┌───────────────────┐        │ ┌──────────────┐ │
│ │ TraceId      │ │          │ X-B3-TraceId      │        │ │ TraceId      │ │
│ │              │ │          │                   │        │ │              │ │
│ │ ParentSpanId │ │ Inject   │ X-B3-ParentSpanId │Extract │ │ ParentSpanId │ │
│ │              ├─┼─────────>│                   ├────────┼>│              │ │
│ │ SpanId       │ │          │ X-B3-SpanId       │        │ │ SpanId       │ │
│ │              │ │          │                   │        │ │              │ │
│ │ Sampled      │ │          │ X-B3-Sampled      │        │ │ Sampled      │ │
│ └──────────────┘ │          └───────────────────┘        │ └──────────────┘ │
│                  │                                       │                  │
└──────────────────┘                                       └──────────────────┘
```

这里以 HTTP 为例，在客户端调用 `Inject` 方法传入 `Context` 信息：
```golang
req, _ := http.NewRequest("GET", "/", nil)
// configure a function that injects a trace context into a reques
injector := b3.InjectHTTP(req)
injector(sp.Context())
```

在服务端调用 `Extract` 方法解析 `Context` 信息：
```golang
req, _ := http.NewRequest("GET", "/", nil)
b3.InjectHTTP(req)(sp.Context())

b.ResetTimer()
_ = b3.ExtractHTTP(copyRequest(req))
```

##  0x03	Jaeger-Tracing 的一般流程
本小节介绍使用 Jaeger 来实现链路追踪。Jaeger 是 Uber 开源的分布式追踪系统，也是遵循 Opentracing 的系统之一。

1、Jaeger 提供了 all-in-one 镜像，方便我们快速开始测试：
```javascript
// 通过 http://localhost:16686 可以打开 Jaeger UI
$ docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.14
```

2、初始化 Jaeger tracer，设置 endpoint / 采样率等信息
```golang
import (
	"context"
	"errors"
	"fmt"
	"io"
	"time"

	"github.com/opentracing/opentracing-go"
	"github.com/opentracing/opentracing-go/log"
	"github.com/uber/jaeger-client-go"
	jaegercfg "github.com/uber/jaeger-client-go/config"
)
// initJaeger 将 jaeger tracer 设置为全局 tracer
func initJaeger(service string) io.Closer {
	cfg := jaegercfg.Configuration{
		// 将采样频率设置为 1，每一个 span 都记录，方便查看测试结果
		Sampler: &jaegercfg.SamplerConfig{
			Type:  jaeger.SamplerTypeConst,
			Param: 1,
		},
		Reporter: &jaegercfg.ReporterConfig{
			LogSpans: true,
			// 将 span 发往 jaeger-collector 的服务地址
			CollectorEndpoint: "http://localhost:14268/api/traces",
		},
	}
	closer, err := cfg.InitGlobalTracer(service, jaegercfg.Logger(jaeger.StdLogger))
	if err != nil {
		panic(fmt.Sprintf("ERROR: cannot init Jaeger: %v\n", err))
	}
	return closer
}
```

3、创建 tracer，生成 root span，下面这段代码创建了一个 root span，并将该 span 通过 context 传递给 `Foo` 方法，以便在 `Foo` 方法中将追踪链继续延续下去：
```golang
func main() {
	closer := initJaeger("in-process")
	defer closer.Close()
	// 获取 jaeger tracer
	t := opentracing.GlobalTracer()
	// 创建 root span
	sp := t.StartSpan("in-process-service")
	// main 执行完结束这个 span
	defer sp.Finish()
	// 将 span 传递给 Foo
	ctx := opentracing.ContextWithSpan(context.Background(), sp)
	Foo(ctx)
}
```
4、`Foo`、`Bar` 方法模拟了独立的子操作 Span，`Foo` 方法调用了 `Bar`，假设在 `Bar` 中发生了一些错误，可以通过 `span.LogFields` 和 `span.SetTag` 将错误记录在追踪链中。`StartSpanFromContext` 这个方法看起来是直接从 `ctx` 中拿到 span 并继承？
```golang
func Foo(ctx context.Context) {
    // 开始一个 span, 设置 span 的 operation_name=Foo
	span, ctx := opentracing.StartSpanFromContext(ctx, "Foo")
	defer span.Finish()
	// 将 context 传递给 Bar
	Bar(ctx)
	// 模拟执行耗时
	time.Sleep(1 * time.Second)
}
func Bar(ctx context.Context) {
    // 开始一个 span，设置 span 的 operation_name=Bar
	span, ctx := opentracing.StartSpanFromContext(ctx, "Bar")
	defer span.Finish()
	// 模拟执行耗时
	time.Sleep(2 * time.Second)

	// 假设 Bar 发生了某些错误
	err := errors.New("something wrong")
	span.LogFields(
		log.String("event", "error"),
		log.String("message", err.Error()),
	)
	span.SetTag("error", true)
}
```

最后，简单看下 `StartSpanFromContext` 的 [实现](https://github.com/opentracing/opentracing-go/blob/master/gocontext.go#L59)，印证了上面的猜想：
```golang
func StartSpanFromContextWithTracer(ctx context.Context, tracer Tracer, operationName string, opts ...StartSpanOption) (Span, context.Context) {
	if parentSpan := SpanFromContext(ctx); parentSpan != nil {
		opts = append(opts, ChildOf(parentSpan.Context()))
	}
	span := tracer.StartSpan(operationName, opts...)
	return span, ContextWithSpan(ctx, span)
}
```

小结下上面的过程，如果要确保追踪链在程序中不断开，需要将函数的第一个参数设置为 `context.Context`，通过 `opentracing.ContextWithSpan` 将保存到 context 中，通过 `opentracing.StartSpanFromContext` 开始一个新的子 span，然后设置直到调用流程结束。

##	0x04	gRPC 中的 OpenTracing

##  0x05    参考
-	[OpenTracing API for Go](https://github.com/opentracing/opentracing-go)
-   [阿里云：通过 Zipkin 上报 Go 应用数据](https://www.alibabacloud.com/help/zh/doc-detail/96334.htm)
-   [Go 集成 Opentracing（分布式链路追踪）](https://juejin.im/post/6844903942309019661)