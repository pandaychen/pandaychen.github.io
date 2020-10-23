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

####	添加 Hook 方法
[`AddHook` 方法](https://github.com/go-kratos/kratos/blob/master/pkg/database/hbase/hbase.go#L27)，添加 hook 数组：
```golang
// AddHook add hook function.
func (c *Client) AddHook(hookFn HookFunc) {
	c.hooks = append(c.hooks, hookFn)
}
```

####	封装 hbase 库的方法
在分析 Kratos 的 Hook 之前，我们先看下 [hbase 库](https://github.com/tsuna/gohbase) 中的接口是如何使用的，主要方法 [如下](https://raw.githubusercontent.com/tsuna/gohbase/master/README.md)：
1、Create a client<br>
```go
client := gohbase.NewClient("localhost")
```

2、Insert a cell<br>
```go
// Values maps a ColumnFamily -> Qualifiers -> Values.
values := map[string]map[string][]byte{"cf": map[string][]byte{"a": []byte{0}}}
putRequest, err := hrpc.NewPutStr(context.Background(), "table", "key", values)
rsp, err := client.Put(putRequest)
```

3、Get an entire row<br>
```go
getRequest, err := hrpc.NewGetStr(context.Background(), "table", "row")
getRsp, err := client.Get(getRequest)
```

4、Get a specific cell<br>
```go
// Perform a get for the cell with key "15", column family "cf" and qualifier "a"
family := map[string][]string{"cf": []string{"a"}}
getRequest, err := hrpc.NewGetStr(context.Background(), "table", "15",
hrpc.Families(family))
getRsp, err := client.Get(getRequest)
```

5、Get a specific cell with a filter<br>
```go
pFilter := filter.NewKeyOnlyFilter(true)
family := map[string][]string{"cf": []string{"a"}}
getRequest, err := hrpc.NewGetStr(context.Background(), "table", "15",
hrpc.Families(family), hrpc.Filters(pFilter))
getRsp, err := client.Get(getRequest)
```

6、Scan with a filter<br>
```go
pFilter := filter.NewPrefixFilter([]byte("7"))
scanRequest, err := hrpc.NewScanStr(context.Background(), "table",
hrpc.Filters(pFilter))
scanRsp, err := client.Scan(scanRequest)
```

##	0x03	慢操作
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

##  0x05	参考
-   [Hbase Client 源码](https://github.com/go-kratos/kratos/tree/master/pkg/database/hbase)