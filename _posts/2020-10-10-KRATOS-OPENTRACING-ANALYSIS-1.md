---
layout:     post
title:      Kratos 源码分析：Tracing （一）
subtitle:   分析 Kratos 的 opentracing 实现：概念与数据结构抽象
date:       2020-10-10
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
前一篇文章 [微服务基础之 链路追踪（OpenTracing）](https://pandaychen.github.io/2020/02/23/OPENTRACING-INTRO/) 大致介绍了 Opentracing 的概念，这篇文章分析下 Kratos 库提供的 OpenTracing 实现，[源码目录](https://github.com/huizluo/kratos/tree/jager-tracing/pkg/net/trace)。Kratos 内置的组件大都接入了 Tracing，其特性如下：

* Kratos 内部的 trace 基于 opentracing 语义
* 使用 protobuf 协议描述 trace 结构
* 集成了全链路支持（gRPC/HTTP/MySQL/Redis/Memcached 等）


##  0x01    Opentracing 回顾
当代的互联网的服务，通常都是用复杂的、大规模分布式集群来实现的。互联网应用构建在不同的软件模块集上，这些软件模块，有可能是由不同的团队开发、可能使用不同的编程语言来实现、有可能布在了几千台服务器，横跨多个不同的数据中心。因此，就需要一些可以帮助理解系统行为、用于分析性能问题的工具。

Logging，Metrics 和 Tracing
Logging，Metrics 和 Tracing 有各自专注的部分。

Logging - 用于记录离散的事件。例如，应用程序的调试信息或错误信息。它是我们诊断问题的依据。
Metrics - 用于记录可聚合的数据。例如，队列的当前深度可被定义为一个度量值，在元素入队或出队时被更新；HTTP 请求个数可被定义为一个计数器，新请求到来时进行累加。
Tracing - 用于记录请求范围内的信息。例如，一次远程方法调用的执行过程和耗时。它是我们排查系统性能问题的利器。
这三者也有相互重叠的部分，如下图所示。

OpenTracing 数据模型
OpenTracing 中的 Trace（调用链）通过归属于此调用链的 Span 来隐性的定义。
特别说明，一条 Trace（调用链）可以被认为是一个由多个 Span 组成的有向无环图（DAG 图），Span 与 Span 的关系被命名为 References。

例如：下面的示例 Trace 就是由 8 个 Span 组成：


有些时候，使用下面这种，基于时间轴的时序图可以更好的展现 Trace（调用链）：


每个 Span 包含以下的状态:

An operation name，操作名称
A start timestamp，起始时间
A finish timestamp，结束时间
Span Tag，一组键值对构成的 Span 标签集合。键值对中，键必须为 string，值可以是字符串，布尔，或者数字类型。
Span Log，一组 span 的日志集合。
每次 log 操作包含一个键值对，以及一个时间戳。
键值对中，键必须为 string，值可以是任意类型。
但是需要注意，不是所有的支持 OpenTracing 的 Tracer，都需要支持所有的值类型。

SpanContext，Span 上下文对象 (下面会详细说明)
References(Span 间关系)，相关的零个或者多个 Span（Span 间通过 SpanContext 建立这种关系）
每一个 SpanContext 包含以下状态：

任何一个 OpenTracing 的实现，都需要将当前调用链的状态（例如：trace 和 span 的 id），依赖一个独特的 Span 去跨进程边界传输
Baggage Items，Trace 的随行数据，是一个键值对集合，它存在于 trace 中，也需要跨进程边界传输
更多关于 OpenTracing 数据模型的知识，请参考 OpenTracing 语义标准。


##  0x02    Trace 实现总览 && 数据结构
Kratos Tracing 的实现 [目录在此](https://github.com/go-kratos/kratos/tree/master/pkg/net/trace)，主要完成了如下工作：
1.  实现 Trace 的接口规范
2.  提供 Trace 对 Tracer 接口的实现，供业务及 Kratos 其他模块接入使用
3.  本身不提供整套 Trace 数据存储，通过 `Repoter` 接口第三方 Tracing 系统（Zipkin || Jaeger），前提是实现他们的上报协议


几个要点 && 问题：
1.  Trace 生成的数据何时存储？
2.  提供了 [`DAPPER`](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/dapper.go) 作为 Tracer 的实例化
3.  如何提升采样的性能压力
4.  如何 Mock？

####  概念
- `Tag`：标签，本质是 KV 字符串
- `Span`：（局部）调用过程
- `Tree`：
- `Annotation`：


从上图可以看出，一个跟踪树结构是由多个 `Span` 构成的，每个 `Span` 代表了一次方法的运行周期；

####  Tag 定义
`Tag` 及 `LogField` 结构，表示最基础的 kv 结构，实现 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/tag.go)，包含了系统使用到的 `Tag` 的定义及结构体：
```golang
// Tag interface
type Tag struct {
	Key   string
	Value interface{}
}
// LogField LogField
type LogField struct {
	Key   string
	Value string
}

//tag 字符串定义
const (
	// The software package, framework, library, or module that generated the associated Span.
	// E.g., "grpc", "django", "JDBI".
	// type string
	TagComponent = "component"

	// Database instance name.
	// E.g., In java, if the jdbc.url="jdbc:mysql://127.0.0.1:3306/customers", the instance name is "customers".
	// type string
	TagDBInstance = "db.instance"

	// A database statement for the given database type.
	// E.g., for db.type="sql", "SELECT * FROM wuser_table"; for db.type="redis", "SET mykey'WuValue'".
	TagDBStatement = "db.statement"

	// Database type. For any SQL database, "sql". For others, the lower-case database category,
	// e.g. "cassandra", "hbase", or "redis".
	// type string
	TagDBType = "db.type"

	// Username for accessing database. E.g., "readonly_user" or "reporting_user"
	// type string
  TagDBUser = "db.user"
}
```

此外，还提供了不同类型的创建的 `Tag` 结构的方法，如最常用的 `string` 类型对应的 `TagString`：
```golang
// TagString new string tag.
func TagString(key string, val string) Tag {
	return Tag{Key: key, Value: val}
}
```

####  SpanContext 结构
在介绍 Span 之前，先引入 `SpanContext` 的 [概念](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/context.go#L20)，SpanContext 保存了分布式追踪的上下文信息，包括 Trace id，Span id 以及其它需要传递到下游服务的内容。一个 OpenTracing 的实现需要将 SpanContext 通过某种序列化协议 (Wire Protocol) 在进程边界上进行传递，以将不同进程中的 Span 关联到同一个 Trace 上。对于 HTTP 请求来说，SpanContext 一般是采用 HTTP header 进行传递的。
```golang
// SpanContext implements opentracing.SpanContext
type spanContext struct {
	// TraceID represents globally unique ID of the trace.
	// Usually generated as a random number.
	TraceID uint64

	// SpanID represents span ID that must be unique within its trace,
	// but does not have to be globally unique.
	SpanID uint64

	// ParentID refers to the ID of the parent span.
	// Should be 0 if the current span is a root span.
	ParentID uint64

	// Flags is a bitmap containing such bits as 'sampled' and 'debug'.
	Flags byte

	// Probability
	Probability float32

	// Level current level
	Level int
}
```

####  跟踪 Span 结构
这里再回顾下Span：一个具有名称和时间长度的操作，例如一个 REST 调用或者数据库操作等。Span 是分布式追踪的最小跟踪单位，一个 Trace 由多段 Span 组成。追踪信息包含时间戳、 事件、 方法名（Family+Title） 、 注释（TAG/Comment）<br>

客户端和服务器上的时间戳来自不同的主机， 我们必须考虑到时间偏差，RPC 客户端发送一个请求之后， 服务器端才能接收到， 对于响应也是一样的（服务器先响应， 然后客户端才能接收到这个响应） 。这样一来，服务器端的 RPC 就有一个时间戳的一个上限和下限

[](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/span.go)：
```golang
// Span is a trace span.
type Span struct {
	dapper        *dapper
	context       spanContext   //span 对应的上下文
	operationName string
	startTime     time.Time
	duration      time.Duration
	tags          []Tag     // 包含了若干个 Tag
	logs          []*protogen.Log
	childs        int
}
```

##  0x03  Tracer 接口及抽象
Kratos 的 Tracer 接口定义在 [这里](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/tracer.go#L56)，`Tracer` 和 `Trace`，此外，对外部提供了 [`trace.SetGlobalTracer()` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/tracer.go#L13)，用于设置自定义的 tracer 对象，如 Kratos 提供了 `zipkin` 的接入。
```golang
// Tracer is a simple, thin interface for Trace creation and propagation.
type Tracer interface {
	// New trace instance with given title.
	New(operationName string, opts ...Option) Trace
	// Inject takes the Trace instance and injects it for
	// propagation within `carrier`. The actual type of `carrier` depends on
	// the value of `format`.
	Inject(t Trace, format interface{}, carrier interface{}) error
	// Extract returns a Trace instance given `format` and `carrier`.
	// return `ErrTraceNotFound` if trace not found.
	Extract(format interface{}, carrier interface{}) (Trace, error)
}

type Trace interface {
	// return current trace id.
	TraceID() string
	// Fork fork a trace with client trace.
	Fork(serviceName, operationName string) Trace

	// Follow
	Follow(serviceName, operationName string) Trace

	// Finish when trace finish call it.
	Finish(err *error)

	// Scan scan trace into info.
	// Deprecated: method Scan is deprecated, use Inject instead of Scan
	// Scan(ti *Info)

	// Adds a tag to the trace.
	//
	// If there is a pre-existing tag set for `key`, it is overwritten.
	//
	// Tag values can be numeric types, strings, or bools. The behavior of
	// other tag value types is undefined at the OpenTracing level. If a
	// tracing system does not know how to handle a particular value type, it
	// may ignore the tag, but shall not panic.
	// NOTE current only support legacy tag: TagAnnotation TagAddress TagComment
	// other will be ignore
	SetTag(tags ...Tag) Trace

	// LogFields is an efficient and type-checked way to record key:value
	// NOTE current unsupport
	SetLog(logs ...LogField) Trace

	// Visit visits the k-v pair in trace, calling fn for each.
	Visit(fn func(k, v string))

	// SetTitle reset trace title
	SetTitle(title string)
}
```

Kratos 默认提供了 `Zipkin` 的接入方法，实现 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/zipkin/config.go#L22)，需要完成如下步骤：
1.  实现 [`reporter` 接口](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/report.go#L22)，其中包含了 `WriteSpan` 和 `Close` 两个方法
2.  通过 `trace.NewTracer` 构造 dapper 跟踪对象
3.  通过 `trace.SetGlobalTracer` 将全局 tracer 初始化为我们设置的对象
```golang
// Init init trace report.
func Init(c *Config) {
	if c.BatchSize == 0 {
		c.BatchSize = 100
	}
	if c.Timeout == 0 {
		c.Timeout = xtime.Duration(200 * time.Millisecond)
	}
	trace.SetGlobalTracer(trace.NewTracer(env.AppID, newReport(c), c.DisableSample))
}
```

####    pb 的结构定义
对于 `Tag`、`Field`、`Log` 及 `Span`，也实现了 pb 的定义，结构 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/proto/span.proto)

单一结构 `Tag` 和 `Field`：
```protobuf
message Tag {
  enum Kind {
    STRING = 0;
    INT = 1;
    BOOL = 2;
    FLOAT = 3;
  }
  string key = 1;
  Kind kind = 2;
  bytes value = 3;
}

message Field {
  string key = 1;
  bytes value = 2;
}
```

`SpanRef`，类似于指针的定义，指向另外一个 `Span`：
```
// SpanRef describes causal relationship of the current span to another span (e.g. 'child-of')
message SpanRef {
  enum RefType {
    CHILD_OF = 0;
    FOLLOWS_FROM = 1;
  }
  RefType ref_type = 1;
  uint64 trace_id = 2;
  uint64 span_id = 3;
}
```

复合结构 `Log`：
```proto
message Log {
  // Deprecated: Kind no long use
  enum Kind {
    STRING = 0;
    INT = 1;
    BOOL = 2;
    FLOAT = 3;
  }
  string key = 1;
  // Deprecated: Kind no long use
  Kind kind = 2;
  // Deprecated: Value no long use
  bytes value = 3;
  int64 timestamp = 4;
  repeated Field fields = 5;      // 多 fields
}
```

复合结构 `Span`，包含了 `repeated SpanRef`、`repeated Tag` 和 `repeated Log`：
```proto
// Span represents a named unit of work performed by a service.
message Span {
  int32 version = 99;
  string service_name = 1;
  string operation_name = 2;
  // Deprecated: caller no long required
  string caller = 3;
  uint64 trace_id = 4;
  uint64 span_id = 5;
  uint64 parent_id = 6;
  // Deprecated: level no long required
  int32  level = 7;
  // Deprecated: use start_time instead instead of start_at
  int64 start_at = 8;
  // Deprecated: use duration instead instead of finish_at
  int64 finish_at = 9;
  float sampling_probability = 10;
  string env = 19;
  google.protobuf.Timestamp start_time = 20;
  google.protobuf.Duration duration = 21;
  repeated SpanRef references = 22;
  repeated Tag tags = 11;
  repeated Log logs = 12;
}
```

##  0x  核心结构 contex
[](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/context.go)，
```golang
// SpanContext implements opentracing.SpanContext
type spanContext struct {
	// TraceID represents globally unique ID of the trace.
	// Usually generated as a random number.
	TraceID uint64

	// SpanID represents span ID that must be unique within its trace,
	// but does not have to be globally unique.
	SpanID uint64

	// ParentID refers to the ID of the parent span.
	// Should be 0 if the current span is a root span.
	ParentID uint64

	// Flags is a bitmap containing such bits as 'sampled' and 'debug'.
	Flags byte

	// Probability
	Probability float32

	// Level current level
	Level int
}
```


##  0x0 Trace 实例化应用
我们以 Warden 模块的 [客户端的实现 client.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go#L101) 为例，看下 Tracing 如何使用。<br>
具体在 client 的拦截器中：
```golang
func (c *Client) handle() grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) (err error) {
		var (
			ok     bool
			t      trace.Trace
			gmd    metadata.MD
			conf   *ClientConfig
			cancel context.CancelFunc
			addr   string
			p      peer.Peer
		)
		var ec ecode.Codes = ecode.OK
		// apm tracing
		if t, ok = trace.FromContext(ctx); ok {
			t = t.Fork("", method)  // 实际上这里是调用了 nooptracer 的 Fork 方法
			defer t.Finish(&err)
		}

		// setup metadata
		gmd = baseMetadata()
		trace.Inject(t, trace.GRPCFormat, gmd)
		c.mutex.RLock()
		if conf, ok = c.conf.Method[method]; !ok {
			conf = c.conf
		}
		c.mutex.RUnlock()
		brk := c.breaker.Get(method)
		if err = brk.Allow(); err != nil {
			_metricClientReqCodeTotal.Inc(method, "breaker")
			return
		}
		defer onBreaker(brk, &err)
		var timeOpt *TimeoutCallOption
		for _, opt := range opts {
			var tok bool
			timeOpt, tok = opt.(*TimeoutCallOption)
			if tok {
				break
			}
		}
		if timeOpt != nil && timeOpt.Timeout > 0 {
			ctx, cancel = context.WithTimeout(nmd.WithContext(ctx), timeOpt.Timeout)
		} else {
			_, ctx, cancel = conf.Timeout.Shrink(ctx)
		}

		defer cancel()
		nmd.Range(ctx,
			func(key string, value interface{}) {
				if valstr, ok := value.(string); ok {
					gmd[key] = []string{valstr}
				}
			},
			nmd.IsOutgoingKey)
		// merge with old matadata if exists
		if oldmd, ok := metadata.FromOutgoingContext(ctx); ok {
			gmd = metadata.Join(gmd, oldmd)
		}
		ctx = metadata.NewOutgoingContext(ctx, gmd)

		opts = append(opts, grpc.Peer(&p))
		if err = invoker(ctx, method, req, reply, cc, opts...); err != nil {
			gst, _ := gstatus.FromError(err)
			ec = status.ToEcode(gst)
			err = errors.WithMessage(ec, gst.Message())
		}
		if p.Addr != nil {
			addr = p.Addr.String()
		}
		if t != nil {
			t.SetTag(trace.String(trace.TagAddress, addr), trace.String(trace.TagComment, ""))
		}
		return
	}
}
```


1、调用 [`trace.FromContext()` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/net/trace/util.go#L50)：
```golang
type ctxKey string

var _ctxkey ctxKey = "kratos/pkg/net/trace.trace"

// FromContext returns the trace bound to the context, if any.
// 这里返回值 t Trace 是 interface，所以可以以 Trace 类型直接返回
func FromContext(ctx context.Context) (t Trace, ok bool) {
	t, ok = ctx.Value(_ctxkey).(Trace)
	return
}
```



##  0x06  第三方 Tracing 兼容
kratos 本身不提供整套 `trace` 数据方案，但在 `net/trace/report.go` 内声明了 `repoter` 接口，可以简单的集成现有开源系统，比如：`zipkin` 和 `jaeger`。

#### zipkin 使用
可以看 [zipkin](https://github.com/bilibili/kratos/tree/master/pkg/net/trace/zipkin) 的协议上报实现，具体使用方式如下：

1. 前提是需要有一套自己搭建的 `zipkin` 集群
2. 在业务代码的 `main` 函数内进行初始化，代码如下：

```go
// 忽略其他代码
import "github.com/bilibili/kratos/pkg/net/trace/zipkin"
// 忽略其他代码
func main(){
    // 忽略其他代码
    zipkin.Init(&zipkin.Config{
        Endpoint: "http://localhost:9411/api/v2/spans",
    })
    // 忽略其他代码
}
```

#### zipkin 效果图

![zipkin](/doc/img/zipkin.jpg)

##  0x07  参考
-   [OpenTracing 语义标准规范及实现](https://www.jianshu.com/p/a963ad0bbe3e)
-   [OpenTracing 语义标准](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/specification.md)
-   [开放分布式追踪（OpenTracing）入门与 Jaeger 实现](https://zhuanlan.zhihu.com/p/34318538)
-   [opentracing](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/specification.md)
-   [dapper](https://bigbully.github.io/Dapper-translation/)
-   [bilibili 毛剑 - B 站微服务链路监控实践](https://myslide.cn/slides/8297)