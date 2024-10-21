---
layout:     post
title:      gRPC 微服务构建之日志（Logging）
subtitle:   zap库应用及使用 gRPC 实现zap日志拦截器（With Tracing）
date:       2020-03-01
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 微服务
    - gRPC
    - 日志
    - zap
---

##  0x00    前言
&emsp;&emsp; 结构化日志对于后台项目的重要性不言而喻，通常调试、记录运行错误及基于日志关键字的监控都会需要后台提供足够充分的日志。Golang 中有非常多优秀的日志库，如 [Zap](https://github.com/uber-go/zap) 和 [Logus](https://github.com/sirupsen/logrus) 等。这篇文章分享下在项目中，如何将 Zap 和 gRPC 完美的融合在一起，保证日志的可读性和高效，同时也兼顾了性能。

##  0x01    Zap 库介绍
Zap 库满足了常见日志库的所有优点，非常适合在项目中使用。具体来说：<br>
-   日志的格式化：比如以 JSON、自定义分隔符等形式记录，方便人为观察及日志分析工具处理
-   性能：每次操作的耗时，以及每次操作分配的内存，作为日志库，两个指标都应该要极低．
-   日志分级
-   按天切分 / 按大小切分 `+` 自动（定期）归档（Zap 本身不具备归档能力，需要借助如 [`lumberjack` 库](https://github.com/natefinch/lumberjack) 来实现）
-   支持采样：全日志记录，支持采样率，防止服务请求剧增导致时输出的日志量剧增

Zap 库是个非常 Nice 的日志库，性能也很出众，推荐阅读 [从 Go 高性能日志库 zap 看如何实现高性能 Go 组件](https://mp.weixin.qq.com/s/i0bMh_gLLrdnhAEWlF-xDw)。

##  0x02    gRPC 和 Zap 融合
gRPC 的 `grpclog` 包，提供了 [LoggerV2](https://godoc.org/google.golang.org/grpc/grpclog#LoggerV2) 的 `interface{}` 定义，[代码在此](https://github.com/grpc/grpc-go/blob/master/grpclog/loggerv2.go)。因此，只要通过 Zap 实现 `LoggerV2` 的接口（即用自己封装的 `Zap.logger` 实例化 `grpc.LoggerV2`），并通过 `SetLoggerV2(l LoggerV2)` 将实现的对象设置到 `grpclog` 包中，那么 gRPC 就会默认使用传入的 Zap 的 `logger` 进行日志打印，非常完美。

`LoggerV2` 的接口定义了下面的方法集合，所以在自己的 Zap 库中，对这里所有的方法都要进行二次封装。
```golang
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

在 gRPC 的 Zap 日志中间件 [go-grpc-middleware:Zap](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/logging/zap/grpclogger.go#L16) 实现中，也是如此实现的：
```golang
func ReplaceGrpcLogger(logger *zap.Logger) {
  //zapGrpcLoggerV2 为实现了 LoggerV2 接口的 struct
	zgl := &zapGrpcLogger{logger.With(SystemField, zap.Bool("grpc_log", true))}
	grpclog.SetLogger(zgl)
}

// ReplaceGrpcLoggerV2 replaces the grpc_log.LoggerV2 with the provided logger.
func ReplaceGrpcLoggerV2(logger *zap.Logger) {
	ReplaceGrpcLoggerV2WithVerbosity(logger, 0)
}

// ReplaceGrpcLoggerV2WithVerbosity replaces the grpc_.LoggerV2 with the provided logger and verbosity.
func ReplaceGrpcLoggerV2WithVerbosity(logger *zap.Logger, verbosity int) {
	zgl := &zapGrpcLoggerV2{
		logger:    logger.With(SystemField, zap.Bool("grpc_log", true)),
		verbosity: verbosity,
	}
	grpclog.SetLoggerV2(zgl)
}
```

此外，Zap 库中提供了两种日志记录器：Sugared Logger 和 Logger。二者的区别在于 [官方描述](https://godoc.org/go.uber.org/zap#hdr-Choosing_a_Logger)：

Sugared Logger：类似于 `fmt.Printf`，更通用。

> In contexts where performance is nice, but not critical, use the SugaredLogger. It's 4-10x faster than other structured logging packages and supports both structured and printf-style logging. Like log15 and go-kit, the SugaredLogger's structured logging APIs are loosely typed and accept a variadic number of key-value pairs. (For more advanced use cases, they also accept strongly typed fields - see the SugaredLogger.With documentation for details.)

用法如下：
```golang
sugar := zap.NewExample().Sugar()
defer sugar.Sync() // zap底层有缓冲。在任何情况下执行 defer sugar.Sync() 是一个很好的习惯
sugar.Infow("failed to fetch URL",
  "url", "http://example.com",
  "attempt", 3,
  "backoff", time.Second,
)	// 字段是松散类型，不是强类型
sugar.Infof("failed to fetch URL: %s", "http://example.com")
```

Logger：性能更高，但需要自己按照 zap 的结构化进行记录。
> In the rare contexts where every microsecond and every allocation matter, use the Logger. It's even faster than the SugaredLogger and allocates far less, but it only supports strongly-typed, structured logging.

用法如下：
```golang
logger := zap.NewExample()
defer logger.Sync()
logger.Info("failed to fetch URL",
  zap.String("url", "http://example.com"),
  zap.Int("attempt", 3),
  zap.Duration("backoff", time.Second),
)	 // 字段是强类型，不是松散类型
```

不过二者可以转换：
```golang
logger := zap.NewExample()
defer logger.Sync()
sugar := logger.Sugar()
plain := sugar.Desugar()
```

####    gRPC 中使用 Zap 记录日志
&emsp;&emsp; 在 `grpclog` 包中，按照 `grpclog.SetLoggerV2(自己实现的 LoggerV2 对象)` 导入自己封装的 `zaplogger` 方法，然后 `grpclog` 就会按照自定义的方法来输出日志了，非常方便。
```golang
grpclog.Infof("%s", message)
grpclog.Errorf("err %v", err)
```

比如，日志的输出实例：
```json
{
	"level": "info",
	"ts": "2020-10-15T07:51:43.384Z",
	"caller": "grpclog/logger.go:49",
	"msg": "CALL method",
	"system": "grpc",
	"grpc_log": true
}
```

##  0x03  gRPC-Zap 日志拦截器
`go-grpc-middleware` 项目还提供了基于 Zap 的日志拦截器（服务端）：[`zap.UnaryServerInterceptor`](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/logging/zap/server_interceptors.go#L24)，此外还有 `zap.StreamServerInterceptor`、`zap.PayloadUnaryServerInterceptor` 以及 `zap.PayloadStreamServerInterceptor` 等。此外还包含了客户端的拦截器实现。

以 `zap.UnaryServerInterceptor` 的 [实现](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/logging/zap/server_interceptors.go#L24) 为例，简单分析下拦截器做了哪些事情：

```golang
// UnaryServerInterceptor returns a new unary server interceptors that adds zap.Logger to the context.
func UnaryServerInterceptor(logger *zap.Logger, opts ...Option) grpc.UnaryServerInterceptor {

  // 初始化选项 opt
	o := evaluateServerOpt(opts)
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
		startTime := time.Now()

		newCtx := newLoggerForCall(ctx, logger, info.FullMethod, startTime)

		resp, err := handler(newCtx, req)
		if !o.shouldLog(info.FullMethod, err) {
			return resp, err
		}
		code := o.codeFunc(err)
		level := o.levelFunc(code)
		duration := o.durationFunc(time.Since(startTime))

		o.messageFunc(newCtx, "finished unary call with code"+code.String(), level, code, err, duration)
		return resp, err
	}
}

//newLoggerForCall
func newLoggerForCall(ctx context.Context, logger *zap.Logger, fullMethodString string, start time.Time) context.Context {
	var f []zapcore.Field
	f = append(f, zap.String("grpc.start_time", start.Format(time.RFC3339)))
	if d, ok := ctx.Deadline(); ok {
		f = append(f, zap.String("grpc.request.deadline", d.Format(time.RFC3339)))
	}
	callLog := logger.With(append(f, serverCallFields(fullMethodString)...)...)
	return ctxzap.ToContext(ctx, callLog)
}

ctxMarkerKey = &ctxMarker{}

//ctxzap.ToContext 实现
// ToContext adds the zap.Logger to the context for extraction later.
// Returning the new context that has been created.
func ToContext(ctx context.Context, logger *zap.Logger) context.Context {
	l := &ctxLogger{
		logger: logger,
	}
	return context.WithValue(ctx, ctxMarkerKey, l)
}
```

`zap.UnaryServerInterceptor` 的流程如下：<br>
1.  该拦截器需要传入两个参数，分别为 `zap.Logger` 日志句柄和参数 `Option`
2.  通过 `evaluateServerOpt` 方法，更新日志记录的选项参数，比如 [`DefaultDeciderMethod`](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/logging/common.go#L28) 是否打印某个方法的日志（默认打印）、`options .durationFunc` 是否记录 RPC 方法的 duration 等等
3.  调用 `newLoggerForCall` 方法，将 `grpc.start_time`、`grpc.request.deadline` 及 `grpc.method` 等 fields 登记在 `logger *zap.Logger` 中，然后通过 `context.WithValue(ctx, ctxMarkerKey, l)` 构建出一个新的 `context` 返回
4.  `resp, err := handler(newCtx, req)` 执行完剩下的拦截器和 RPC 方法
5.  注意最后 `o.messageFunc(newCtx, "finished unary call with code"+code.String(), level, code, err, duration)` 这个方法，zap 中是调用了 [`zap.DefaultMessageProducer` 方法](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/logging/zap/options.go#L201)，该方法将第 `3` 步注入到 `context` 的 `zap.Logger` 取出，然后最终输出打印结果

<br><br>
为何要做的如此复杂？从实现上看是为了捕获到由 [`grpc_ctxtags`](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/tags) 这款拦截器生成的 kv 字段（在当前的拦截器流程到执行完 RPC 方法这段期间发生的改变），所以在 `ctxzap.Extract` 中会调用 `grpc_ctxtags.Extract` 先拿到这些增加的 kv 字段，然后再打印。由此可见，作者对细节的考虑还是相当全面的。

```golang
// MessageProducer produces a user defined log message
type MessageProducer func(ctx context.Context, msg string, level zapcore.Level, code codes.Code, err error, duration zapcore.Field)

// DefaultMessageProducer writes the default message
func DefaultMessageProducer(ctx context.Context, msg string, level zapcore.Level, code codes.Code, err error, duration zapcore.Field) {
	// re-extract logger from newCtx, as it may have extra fields that changed in the holder.
	ctxzap.Extract(ctx).Check(level, msg).Write(
		zap.Error(err),
		zap.String("grpc.code", code.String()),
		duration,
	)
}

// Extract 实现
// Extract takes the call-scoped Logger from grpc_zap middleware.
//
// It always returns a Logger that has all the grpc_ctxtags updated.
func Extract(ctx context.Context) *zap.Logger {
	l, ok := ctx.Value(ctxMarkerKey).(*ctxLogger)
	if !ok || l == nil {
		return nullLogger
	}
	// Add grpc_ctxtags tags metadata until now.
	fields := TagsToFields(ctx)
	// Add zap fields added until now.
	fields = append(fields, l.fields...)
	return l.logger.With(fields...)
}

//TagsToFields 实现
// TagsToFields transforms the Tags on the supplied context into zap fields.
func TagsToFields(ctx context.Context) []zapcore.Field {
	fields := []zapcore.Field{}
	tags := grpc_ctxtags.Extract(ctx)
	for k, v := range tags.Values() {
		fields = append(fields, zap.Any(k, v))
	}
	return fields
}

```

选项中还提供了 `codeFunc grpc_logging.ErrorToCode` 选项，用于将错误转为 `codes.Code`，默认实现为 `grpc_logging.DefaultErrorToCode`：
```golang
import (
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

// ErrorToCode function determines the error code of an error
// This makes using custom errors with grpc middleware easier
type ErrorToCode func(err error) codes.Code

func DefaultErrorToCode(err error) codes.Code {
	return status.Code(err)
}
```

此外，还提供了 `DefaultCodeToLevel` 和 `DefaultClientCodeToLevel` 方法，用于将 `grpc.codes` 转为 zap 的输出的日志 `level`：
```golang
func DefaultCodeToLevel(code codes.Code) zapcore.Level {
	switch code {
	case codes.OK:
		return zap.InfoLevel
	case codes.Canceled:
		return zap.InfoLevel
	case codes.Unknown:
		return zap.ErrorLevel
	case codes.InvalidArgument:
		return zap.InfoLevel
	case codes.DeadlineExceeded:
		return zap.WarnLevel
	case codes.NotFound:
		return zap.InfoLevel
	case codes.AlreadyExists:
		return zap.InfoLevel
	case codes.PermissionDenied:
		return zap.WarnLevel
	case codes.Unauthenticated:
		return zap.InfoLevel // unauthenticated requests can happen
	case codes.ResourceExhausted:
		return zap.WarnLevel
	case codes.FailedPrecondition:
		return zap.WarnLevel
	case codes.Aborted:
		return zap.WarnLevel
	case codes.OutOfRange:
		return zap.WarnLevel
	case codes.Unimplemented:
		return zap.ErrorLevel
	case codes.Internal:
		return zap.ErrorLevel
	case codes.Unavailable:
		return zap.WarnLevel
	case codes.DataLoss:
		return zap.ErrorLevel
	default:
		return zap.ErrorLevel
	}
}

func DefaultClientCodeToLevel(code codes.Code) zapcore.Level {
	switch code {
	case codes.OK:
		return zap.DebugLevel
	case codes.Canceled:
		return zap.DebugLevel
	case codes.Unknown:
		return zap.InfoLevel
	case codes.InvalidArgument:
		return zap.DebugLevel
	case codes.DeadlineExceeded:
		return zap.InfoLevel
	case codes.NotFound:
		return zap.DebugLevel
	case codes.AlreadyExists:
		return zap.DebugLevel
	case codes.PermissionDenied:
		return zap.InfoLevel
	case codes.Unauthenticated:
		return zap.InfoLevel // unauthenticated requests can happen
	case codes.ResourceExhausted:
		return zap.DebugLevel
	case codes.FailedPrecondition:
		return zap.DebugLevel
	case codes.Aborted:
		return zap.DebugLevel
	case codes.OutOfRange:
		return zap.DebugLevel
	case codes.Unimplemented:
		return zap.WarnLevel
	case codes.Internal:
		return zap.WarnLevel
	case codes.Unavailable:
		return zap.WarnLevel
	case codes.DataLoss:
		return zap.WarnLevel
	default:
		return zap.InfoLevel
	}
}
```

####  日志输出
使用了 Zap 拦截器的输出如下，示例代码 [在此](https://github.com/pandaychen/grpc_in_action/tree/master/zaplogger)：
```json
{
	"level": "info",
	"ts": "2020-10-15T09:05:17.877Z",
	"caller": "zap/server_interceptors.go:39",
	"msg": "finished unary call with code OK",
	"grpc.start_time": "2020-10-15T09:05:17Z",
	"system": "grpc",
	"span.kind": "server",
	"grpc.service": "proto.GreeterService",
	"grpc.method": "SayHello",
	"grpc.code": "OK",
	"grpc.duration": 0.000384034
}
```

##	0x04	zap库的应用

####	zap framework
zap由创建logger与写log两个关键过程组成。其中zap的核心是`zapcore.Core`抽象（下文），`Core`是zap定义的一个log接口，围绕着这个`Core`，zap提供上层log对象以及相应的方法，开发者同样可以基于该接口定制自己的log包，非常方便

![zap](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/logger/uber-zap-advanced-usage-2.png)

####	使用
快速创建 logger 的方法：
-	`zap.NewProduction()`
-	`zap.NewDevelopment()`
-	`zap.NewExample()`

简单看下`zap.NewExample()`的实现，可供自行实现日志对象参考：

```GO
// https://github.com/uber-go/zap/blob/v1.24.0/logger.go#L127
func NewExample(options ...Option) *Logger {
	encoderCfg := zapcore.EncoderConfig{
        MessageKey:     "msg",  // 日志内容key:val， 前面的key设为msg
		LevelKey:       "level", // 日志级别的key设为level
		NameKey:        "logger", // 日志名
		EncodeLevel:    zapcore.LowercaseLevelEncoder, //日志级别，默认小写
		EncodeTime:     zapcore.ISO8601TimeEncoder, // 日志时间
		EncodeDuration: zapcore.StringDurationEncoder,
	}
	core := zapcore.NewCore(zapcore.NewJSONEncoder(encoderCfg), os.Stdout, DebugLevel)
	return New(core).WithOptions(options...)
}
```

####	自定义logger配置
组装一个自定义 `zap.Logger` 需要了解 `zapcore.Core`，而它包括如下三个参数：
-	`Encoder`，包括`EncoderConfig`，`Encoder`是日志消息的编码器
-	`WriteSyncer`，是支持Sync方法的`io.Writer`，含义是日志输出的地方，可以很方便的通过`zap.AddSync`将一个`io.Writer`转换为支持`Sync`方法的`WriteSyncer`
-	`LevelEnabler`：日志级别相关的参数

1、`Encoder`：`Encoder` 控制着日志最终的输出格式，`Encoder` 本身控制日志的编码格式（json OR plain-text）。其包含的 `EncoderConfig` 则控制每一个字段更具体的格式。zap 提供了两个内置 `Encoder` 可以满足大部分需要， `zapcore.NewConsoleEncoder(encoderConfig)` 与 `zapcore.NewJSONEncoder(encoderConfig)`，直接使用即可

```GO
//type TimeEncoder func(time.Time, PrimitiveArrayEncoder)
//希望输出的时间格式是 [yyyyMMdd]，可以自定义一个这样的 TimeEncoder 并使用
func CustomTimeEncoder(t time.Time, enc zapcore.PrimitiveArrayEncoder) {
	enc.AppendString(fmt.Sprintf("[%04d%02d%02d]", t.Year(), t.Month(), t.Day()))
}

func main() {
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = CustomTimeEncoder // 设置自定义 TimeEncoder
	encoder := zapcore.NewConsoleEncoder(encoderConfig)
	core := zapcore.NewCore(encoder, zapcore.AddSync(os.Stdout), zapcore.DebugLevel)
	zap.New(core).Sugar().Infow("Check order.", "id", 123, "name", "Fruit")\
}
```

2、`WriteSyncer`：控制输出的位置（动作，日志最终输出到哪里），如`zapcore.AddSync(io.Writer)`、`zapcore.AddSync(io.Discarder)`等。看下`apcore.AddSync`的定义，参数为`io.Writer`，返回一个`WriteSyncer`

```GO
func AddSync(w io.Writer) WriteSyncer

type WriteSyncer interface {
	io.Writer
	Sync() error
}
```

从上面的定义可知，只要实现了`Write`和`Sync`这两个方法就可以实现自定义的logger，可参考下面的remotelogger实现。这里以最常见的输出到文件为例：

```GO
//输出到文件
logfile, _ := os.Create("/path/to/a.log")
core := zapcore.NewCore(encoder, zapcore.AddSync(logfile), zapcore.DebugLevel)
logger := zap.New(core)

//输出到控制台且输出到文件
logfile, _ := os.Create("/path/to/a.log")
// file + stdout
multiSyner := zapcore.NewMultiWriteSyncer(zapcore.AddSync(logfile), zapcore.AddSync(os.Stdout))
core := zapcore.NewCore(encoder, multiSyner, zapcore.DebugLevel)
logger := zap.New(core)
```

3、`LevelEnabler`：接口，用来确定某个等级的日志是否需要输出。zap 内置的几种类型（`zapcore.DebugLevel` / `zapcore.InfoLevel` ...）默认实现了该接口，会打印等级 >= 它的日志。

```GO
// zap 提供了一个便捷的函数命名类型
//type LevelEnablerFunc func(zapcore.Level) bool
infoLevel := zap.LevelEnablerFunc(func(lv zapcore.Level) bool {
	return lv >= zapcore.InfoLevel
})
// 效果等同于 zapcore.InfoLevel
```

####	多logger整合
针对要不同的等级输出到不同的地方，且不同的地方采用不同的格式的需求，可以使用`zapcore.NewTee`实现

```GO
debugCore := zapcore.NewCore(encoder, debugSyner, zapcore.DebugLevel)
errCore := zapcore.NewCore(encoder, errSyner, zapcore.ErrorLevel)
tee := zapcore.NewTee(debugCore, errCore)
logger := zap.New(tee)
```

####	自动压缩&&rotate
[zaplog](https://github.com/pandaychen/goes-wrapper/blob/master/zaplog/zaplog.go)

####	自定义remoteLog
参考项目elkeid的实现，实现了grpc/remote两种logger

-	[`GrpcWriter`](https://github.com/pandaychen/elkeid_fork/blob/main/agent/log/writer.go)
-	[`remoteWriter`](https://github.com/pandaychen/elkeid_fork/blob/main/plugins/lib/go/log/writer.go)

这里简单分析下`remoteWriter`，就是实现了`Write`与`Sync`方法：

```GO
type remoteWriter struct {
	client *plugins.Client		//这里是一个os.Pipe，父子进程通信
}

func (w *remoteWriter) Write(p []byte) (n int, err error) {
	if w.client != nil {
		rec := &plugins.Record{
			DataType: 1011,
			Data: &plugins.Payload{
				Fields: map[string]string{},
			},
		}
		fields := map[string]interface{}{}
		err = json.Unmarshal(p, &fields)
		if err != nil {
			return
		}
		timestamp, ok := fields["timestamp"]
		if ok {
			timestamp, err := strconv.ParseInt(timestamp.(string), 10, 64)
			if err == nil {
				rec.Timestamp = timestamp
				delete(fields, "timestamp")
			}
		}
		if rec.Timestamp == 0 {
			rec.Timestamp = time.Now().Unix()
		}
		for k, v := range fields {
			switch v := v.(type) {
			case string:
				rec.Data.Fields[k] = v
			case int:
				rec.Data.Fields[k] = strconv.Itoa(v)
			}
		}
		//日志会被发送给父进程
		err = w.client.SendRecord(rec)
		if err != nil {
			return
		}
	}
	n = len(p)
	return
}

func (w *remoteWriter) Sync() error {
	if w.client != nil {
		return w.client.Flush()
	} else {
		return nil
	}
}
```

####	封装Zap增加TraceId

##	0x05	日志染色

####	什么是日志染色
通俗点说，染色日志是解决对特定场景（用户）行为的日志记录。在一个调用链里面，标志出某个特定需求的过程，让整个调用链里的上下文信息一直被传输下去，就像一开始被标记染色一样。在排查问题时，只想要某几个用户的日志信息，通过染色日志把这几个用户的日志另外打印一份，并收集在同一个地方，这样可以方便的查找到这些染色用户的日志。

举例来说，比如某微信用户反馈语音发送存在问题，通过对用户ID进行染色，后端服务接收到染色用户的请求后，就会将该用户本次调用的处理过程日志打印出来。此外，还可以通过染色执行特定逻辑，比如新上线一个功能，开放给部分用户使用，通过染色，后端服务可以判断用户是否在测试名单中，再执行相应的服务逻辑。

##  0x06    关于日志的一些优化细节
如何在高并发的实时系统中优化日志写入呢？在笔者之前的 DPDK 网络包处理项目中，总结了这几条经验：
1.  线程将待落地日志结构化（标识日志类型、等级、内容等）写入 RingBuffer，读端从 RingBuffer 中取出日志，落地写入
2.  单线程写日志
3.  在 Golang 中，利用 单独的 goroutine 异步写日志
4.  打印 `GoroutineId` 对性能产生较大的影响，打印函数行号对性能有一定的影响

##  0x07  参考
-   [从 Go 高性能日志库 zap 看如何实现高性能 Go 组件](https://mp.weixin.qq.com/s/i0bMh_gLLrdnhAEWlF-xDw)
-   [在 Go 语言项目中使用 Zap 日志库](https://www.liwenzhou.com/posts/Go/zap/)
-	[jupiter：2.3 日志](https://jupiter.douyu.com/jupiter/2.3logger.html#_2-3-5-%E4%B8%9A%E5%8A%A1%E7%9A%84%E6%96%87%E6%9C%AC%E6%97%A5%E5%BF%97)
-	[TARS染色日志 ｜ 收集记录特定日志](https://cloud.tencent.com/developer/article/1791859)
-	[golang 日志切割库 goroutine 泄漏导致进程 panic 问题排查](https://zhuanlan.zhihu.com/p/453788600)
-	[Go 每日一库之 zap](https://darjun.github.io/2020/04/23/godailylib/zap/)
-	[一文告诉你如何用好uber开源的zap日志库](https://tonybai.com/2021/07/14/uber-zap-advanced-usage/)
-	[Go: How Zap Package is Optimized](https://readmedium.com/go-how-zap-package-is-optimized-dbf72ef48f2d)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
