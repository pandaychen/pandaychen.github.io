---
layout:     post
title:      Kratos 源码分析：Hbase 库封装
subtitle:   分析 Kratos 的 Hbase Client：How to Hook？
date:       2020-07-13
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
Kratos 库的 Hbase Client，进行封装加入了链路追踪和统计。基于 [Golang HBase client](https://github.com/tsuna/gohbase) 实现。Kratos 对此库进行了部分 Hook，本文来看下这里是如何实现 Hook 机制的。具体有如下几点：
-	[慢操作日志](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/slowlog.go)
-	[Tracing](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/trace.go)
-	[Metrcis](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/metrics.go)


##	0x01	使用
先看下 Hbase Client 的使用方法：
```golang
func main() {
	config := &hbase.Config{Zookeeper: &hbase.ZKConfig{Addrs: []string{"localhost"}}}
	client := hbase.NewClient(config)

	values := map[string]map[string][]byte{"name": {"firstname": []byte("hello"), "lastname": []byte("world")}}
	ctx := context.Background()

	_, err := client.PutStr(ctx, "user", "user1", values)
	if err != nil {
		panic(err)
	}

	result, err := client.GetStr(ctx, "user", "user1")
	if err != nil {
		panic(err)
	}
	fmt.Printf("%v", result)
}
```

##	0x02	Hook 的实现
本节来分析下 Hbase Client 的 [`Hook` 实现](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/hbase.go)，首先是 `2` 个重要的定义，`HookFunc` 及 `Client`，看着很眼熟（gRPC 的拦截器数组）

注意下面的 `HookFunc` 这个通用函数类型定义， 参数为 `ctx context.Context, call hrpc.Call, customName string`，<font color="#dd0000"> 其返回值为一个函数 </font>，参数为 `err error`，正是这个函数完成了开发者自定义的 hook 逻辑：

```golang
// HookFunc hook function call before every method and hook return function will call after finish.
type HookFunc func(ctx context.Context, call hrpc.Call, customName string) func(err error)

// Client hbase client.
type Client struct {
	hc     gohbase.Client	// 封装了 Hbase 的客户端
	addr   string
	config *Config
	hooks  []HookFunc		//hook 方法数组
}
```

回想下，拦截器要做的事情，是在真正调用的方法前后做一些事情。思考下，需要完成哪些步骤？
1.	首先需要添加 Hook 数组的操作
2.	Hook 方法需要传入哪些参数？这里的参数是 `func(ctx context.Context, call hrpc.Call, customName string)`，这里的 [`hrpc.Call`](https://github.com/tsuna/gohbase/blob/master/hrpc/call.go#L50)
3.	要封装的方法的实现（第三方库），在何处调用我们的 Hook 方法？
4.	Hook 方法的位置，是在封装方法之前还是之后？

####	封装 hbase 库的方法
在分析 Kratos 的 Hook 实现之前，我们先看下 [hbase 库](https://github.com/tsuna/gohbase) 中的接口是如何使用的，主要方法 [如下](https://raw.githubusercontent.com/tsuna/gohbase/master/README.md)：

1、Create a client<br>
```golang
client := gohbase.NewClient("localhost")
```

2、Insert a cell<br>
```golang
// Values maps a ColumnFamily -> Qualifiers -> Values.
values := map[string]map[string][]byte{"cf": map[string][]byte{"a": []byte{0}}}
putRequest, err := hrpc.NewPutStr(context.Background(), "table", "key", values)
rsp, err := client.Put(putRequest)
```

3、Get an entire row<br>
```golang
getRequest, err := hrpc.NewGetStr(context.Background(), "table", "row")
getRsp, err := client.Get(getRequest)
```

4、Get a specific cell<br>
```golang
// Perform a get for the cell with key "15", column family "cf" and qualifier "a"
family := map[string][]string{"cf": []string{"a"}}
getRequest, err := hrpc.NewGetStr(context.Background(), "table", "15",
hrpc.Families(family))
getRsp, err := client.Get(getRequest)
```

5、Get a specific cell with a filter<br>
```golang
pFilter := filter.NewKeyOnlyFilter(true)
family := map[string][]string{"cf": []string{"a"}}
getRequest, err := hrpc.NewGetStr(context.Background(), "table", "15",
hrpc.Families(family), hrpc.Filters(pFilter))
getRsp, err := client.Get(getRequest)
```

6、Scan with a filter<br>
```golang
pFilter := filter.NewPrefixFilter([]byte("7"))
scanRequest, err := hrpc.NewScanStr(context.Background(), "table",
hrpc.Filters(pFilter))
scanRsp, err := client.Scan(scanRequest)
```

试想，需要着重做两件事情（以 `NewPutStr` 方法为例）：
1.	封装原生的客户端初始化 `gohbase.NewClient("localhost")` 流程，加上一些额外的 hook 初始化配置及服务端连接配置（如 zk 集群等）
2.	在 `hrpc.NewPutStr()`、`client.Put()` 之前或者之后进行 hook，以达到我们的目的
3.	此外，在 `client.Put()` 之后会返回结果，里面包含了 `err` 信息，是否需要对 `err` 再包装一层处理？

接下来，就围绕着这几点来分析。

##	0x03	How to Hook

####	添加 Hook 方法
[`AddHook` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/hbase.go#L27)，添加具体的 hook 方法到 `c.hooks` 数组，先添加的 hook 方法先执行：
```golang
// AddHook add hook function.
func (c *Client) AddHook(hookFn HookFunc) {
	c.hooks = append(c.hooks, hookFn)
}
```

####	封装原生客户端
```golang
// HookFunc hook function call before every method and hook return function will call after finish.
type HookFunc func(ctx context.Context, call hrpc.Call, customName string) func(err error)

// Client hbase client.
type Client struct {
	hc     gohbase.Client
	addr   string
	config *Config
	hooks  []HookFunc
}
```

通过 Kratos 的 [`hbase.NewClient` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/hbase.go#L44) 创建了 hook 后的客户端，注意到，在封装后的客户端初始化 `NewClient` 方法中，默认向 `c.hooks` 数组中添加了三个 hook 方法，具体的实现在下一节描述。
-	`NewSlowLogHook`：慢操作统计
-	`MetricsHook`：Metrics 打点
-	`TraceHook`：Tracing

```golang
// NewClient new a hbase client.
func NewClient(config *Config, options ...gohbase.Option) *Client {
	rawcli := NewRawClient(config, options...)
	rawcli.AddHook(NewSlowLogHook(250 * time.Millisecond))
	rawcli.AddHook(MetricsHook(config))
	rawcli.AddHook(TraceHook("database/hbase", strings.Join(config.Zookeeper.Addrs, ",")))
	return rawcli
}

// NewRawClient new a hbase client without prometheus metrics and dapper trace hook.
func NewRawClient(config *Config, options ...gohbase.Option) *Client {
	zkquorum := strings.Join(config.Zookeeper.Addrs, ",")
	if config.Zookeeper.Root != "" {
		options = append(options, gohbase.ZookeeperRoot(config.Zookeeper.Root))
	}
	if config.Zookeeper.Timeout != 0 {
		options = append(options, gohbase.ZookeeperTimeout(time.Duration(config.Zookeeper.Timeout)))
	}

	if config.RPCQueueSize != 0 {
		log.Warn("RPCQueueSize configuration be ignored")
	}
	// force RpcQueueSize = 1, don't change it !!! it has reason  (゜ - ゜) つロ
	options = append(options, gohbase.RpcQueueSize(1))

	if config.FlushInterval != 0 {
		options = append(options, gohbase.FlushInterval(time.Duration(config.FlushInterval)))
	}
	if config.EffectiveUser != "" {
		options = append(options, gohbase.EffectiveUser(config.EffectiveUser))
	}
	if config.RegionLookupTimeout != 0 {
		options = append(options, gohbase.RegionLookupTimeout(time.Duration(config.RegionLookupTimeout)))
	}
	if config.RegionReadTimeout != 0 {
		options = append(options, gohbase.RegionReadTimeout(time.Duration(config.RegionReadTimeout)))
	}

	// 调用 gohbase 的客户端创建
	hc := gohbase.NewClient(zkquorum, options...)
	return &Client{
		hc:     hc,
		addr:   zkquorum,
		config: config,
	}
}
```

####	封装方法
封装后的 [`PutStr` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/hbase.go#L195) 如下：
```golang
// PutStr insert the given family-column-values in the given row key of the given table.
func (c *Client) PutStr(ctx context.Context, table string, key string, values map[string]map[string][]byte, options ...func(hrpc.Call) error) (*hrpc.Result, error) {
	// 初始化 request
	put, err := hrpc.NewPutStr(ctx, table, key, values, options...)
	if err != nil {
		return nil, err
	}

	// 初始化 hooks 数组
	finishHook := c.invokeHook(ctx, put, "PUT")
	result, err := c.hc.Put(put)
	// 真正的 Put 执行完成后，调用 hooks 中的方法
	finishHook(err)
	return result, err
}
```

封装的 hook 主要体现在 `invokeHook` 方法中，它的传参是 `ctx context.Context, call hrpc.Call, customName string`，返回值是 `func(error)`，如下定义：

注意 `finishHooks = append(finishHooks, fn(ctx, call, customName))` 这一行代码，它完成了三件事情：
1.	将实际的参数 `ctx context.Context, call hrpc.Call, customName string` 绑定到中间件上：`fn(ctx, call, customName)`
2.	将 `fn(ctx, call, customName)` 运行的结果，放在 `finishHooks` 这个数组中，同时注意到 `finishHooks` 是个 `func(error)` 类型的数组（传入参数为 `error` 类型）
3.	遍历 `finishHooks`，依次运行各个中间件的返回（中间件的返回也是个方法）

```golang
func (c *Client) invokeHook(ctx context.Context, call hrpc.Call, customName string) func(error) {
	finishHooks := make([]func(error), 0, len(c.hooks))
	for _, fn := range c.hooks {
		// 将实际的参数，绑定到中间件上，同时 fn 的结果添加到 finishHooks 中
		// 注意 fn 的结果是 func(error) 类型
		finishHooks = append(finishHooks, fn(ctx, call, customName))
	}
	// 返回一个函数
	return func(err error) {
		// 遍历 finishHooks，依次运行各个中间件
		for _, fn := range finishHooks {
			fn(err)
		}
	}
}
```

整体执行的流程如下图所示：
![invokehook]()
-	Part1：遍历 c.hooks 数组，按顺序执行中间件 `fn(ctx, call, customName)`
-	Part2：执行业务逻辑
-	Part3：遍历 `finishHooks`，执行中间件的返回方法 `fn(error)`

为了更方便理解这里运行的流程，我把这个 hook 模式简化了一个 [版本](https://github.com/pandaychen/golang_in_action/blob/master/hook/hook_example1.go)。

##	0x04	具体的 hook 中间件分析

####	慢操作 hook
这个比较容易理解，根据上面图易知：
1.	第一步：记录 `start` 时间
2.	第二步：执行业务逻辑
3.	第三步：超时判断，超过 `threshold` 即打印日志

```golang
// NewSlowLogHook log slow operation.
func NewSlowLogHook(threshold time.Duration) HookFunc {
	return func(ctx context.Context, call hrpc.Call, customName string) func(err error) {
		start := time.Now()
		return func(error) {
			duration := time.Since(start)
			if duration < threshold {
				return
			}
			log.Warn("hbase slow log: %s %s %s time: %s", customName, call.Table(), call.Key(), duration)
		}
	}
}
```

####	Metrics-Hook
[`Metrics-Hook` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/metrics.go#L50) 实现在此，与上面不同的是，在 `finishHook` 流程多加了对 `error` 的处理：
```golang
// MetricsHook if stats is nil use stat.DB as default.
func MetricsHook(config *Config) HookFunc {
	return func(ctx context.Context, call hrpc.Call, customName string) func(err error) {
		now := time.Now()
		if customName == "" {
			customName = call.Name()
		}
		return func(err error) {
			durationMs := int64(time.Since(now) / time.Millisecond)
			// 耗时统计
			_metricReqDur.Observe(durationMs, strings.Join(config.Zookeeper.Addrs, ","), "", customName)
			if err != nil && err != io.EOF {
				// 错误累加
				_metricReqErr.Inc(strings.Join(config.Zookeeper.Addrs, ","), "", customName, codeFromErr(err))
			}
		}
	}
}
```

####	Tracing-Hook
[`Tracing-Hook` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/trace.go#L13) 的实现如下：
```golang
// TraceHook create new hbase trace hook.
func TraceHook(component, instance string) HookFunc {
	var internalTags []trace.Tag
	internalTags = append(internalTags, trace.TagString(trace.TagComponent, component))
	internalTags = append(internalTags, trace.TagString(trace.TagDBInstance, instance))
	internalTags = append(internalTags, trace.TagString(trace.TagPeerService, "hbase"))
	internalTags = append(internalTags, trace.TagString(trace.TagSpanKind, "client"))
	return func(ctx context.Context, call hrpc.Call, customName string) func(err error) {
		noop := func(error) {}
		root, ok := trace.FromContext(ctx)
		if !ok {
			return noop
		}
		if customName == "" {
			customName = call.Name()
		}
		span := root.Fork("","Hbase:"+customName)
		span.SetTag(internalTags...)
		statement := string(call.Table()) + " " + string(call.Key())
		span.SetTag(trace.TagString(trace.TagDBStatement, statement))
		return func(err error) {
			if err == io.EOF {
				// reset error for trace.
				err = nil
			}
			span.Finish(&err)
		}
	}
}
```


##  0x05	参考
-   [Hbase Client 源码](https://github.com/go-kratos/kratos/tree/master/pkg/database/hbase)