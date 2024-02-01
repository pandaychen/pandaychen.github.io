---
layout:     post
title:      Kratos 源码分析：CGI 框架 BM （二）
subtitle:   分析 blademaster 中的拦截器实现及设计
date:       2020-10-02
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
熟悉 gRPC 的开发者对拦截器 interceptor（中间件）不会陌生，`gin` 框架也提供了这种能力，下面代码默认启用了 `Logger()` 和 `Recovery()` 两个中间件。
```golang
func main() {
    r := gin.Default()

    r.GET("/ping", func(c *gin.Context) {
        c.String(http.StatusOK, "%s", "hello world")
    })

    if err := r.Run("0.0.0.0:8080"); err != nil {
        log.Fatalln(err)
    }
}
```

中间件的最大好处是 <font color="#dd0000"> 剥离非业务逻辑 </font>，通过中间件（链）的方式将一些基础组件的功能整合起来，是一种非常实用的手段。基于 `net.http` 包实现的话，中间件构造需要通过包装 `http.Handler`，再返回一个新的 `http.Handler`。

##  0x01    拦截器的执行顺序
关于拦截器的执行顺序，先看下面 [例子](https://chai2010.gitbooks.io/advanced-go-programming-book/content/ch5-web/ch5-03-middleware.html)：
```golang
func hello(wr http.ResponseWriter, r *http.Request) {
	wr.Write([]byte("hello"))
}

func timeMiddleware(next http.Handler) http.Handler {
	// 返回 http.HandlerFunc() 包装后的方法
	return http.HandlerFunc(func(wr http.ResponseWriter, r *http.Request) {
		timeStart := time.Now()

		// next handler
		next.ServeHTTP(wr, r)

		timeElapsed := time.Since(timeStart)
		fmt.Println("timeMiddleware:", timeElapsed)
	})
}

func logMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(wr http.ResponseWriter, r *http.Request) {
		fmt.Println("logMiddleware:before logger")
		// next handler
		next.ServeHTTP(wr, r)

		fmt.Println("logMiddleware:after logger")
	})
}

func main() {
	//customizedHandler = logger(timeout(ratelimit(helloHandler)))
	http.Handle("/", logMiddleware(timeMiddleware(http.HandlerFunc(hello))))
	http.ListenAndServe(":8080", nil)
}
```

输出为：
```javascript
logMiddleware:before logger
timeMiddleware: 7.628µs
logMiddleware:after logger
```

如何理解这里的中间件包装逻辑呢？注意 `timeMiddleware` 的定义代码：
```golang
func(wr http.ResponseWriter, r *http.Request) {
	timeStart := time.Now()

	// next handler
	next.ServeHTTP(wr, r)

	timeElapsed := time.Since(timeStart)
	logger.Println(timeElapsed)
}
```

在 `net.http` 包中，只要 handler 函数签名是 `func (ResponseWriter, *Request)`，那么此 handler 和 `http.HandlerFunc()` 就有了一致的函数签名，可以将该 handler 函数进行类型转换并转为 `http.HandlerFunc`。而 `http.HandlerFunc` 实现了 `http.Handler` 这个接口。在 `net.http` 库需要调用你的 handler 函数来处理 http 请求时，会调用 `HandlerFunc()` 的 `ServeHTTP()` 函数。

对于上面的例子，调用链如下图所示：
![http-middleware](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/ch6-03-middleware_flow.png)

小结下，中间件要做的事情就是通过一个或多个函数对 handler 进行包装，返回一个包括了各个中间件逻辑的函数链（chains）。拦截器链在进行请求处理的时候就是不断地进行函数压栈再出栈（类似于递归的执行流）

##  0x02   Context：存储中间件
`bm.Context` 是 `bm` 框架的核心结构，从其封装的成员来看，是一个 HTTP 请求从开始到响应结束时包含的所有属性（如 KV 存储、中间件数组等）。以下是 bm 框架 中 [`Context` 对象结构](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/context.go)：
```golang
// Context is the most important part. It allows us to pass variables between
// middleware, manage the flow, validate the JSON of a request and render a
// JSON response for example.
type Context struct {
    context.Context     // 惯用做法：封装 context.Context

    // 封装了 http 的 request 和 response 结构
    Request *http.Request   //net.http
    Writer  http.ResponseWriter

    // flow control
    index    int8
    handlers []HandlerFunc  // 中间件数组

    // Keys is a key/value pair exclusively for the context of each request.
    Keys map[string]interface{}

    Error error

    method string
    engine *Engine
}
```
各个成员的作用如下：
* `context.Context`：首先可以看到 blademaster 的 Context 结构体中会 embed 一个标准库中的 Context 实例，bm 中的 Context 也是直接通过该实例来实现标准库中的 Context 接口
* blademaster 会使用配置的 server timeout (默认 1s) 作为一次请求整个过程中的超时时间，使用该 context 调用 dao 做数据库、缓存操作查询时均会将该超时时间传递下去，一旦抵达 deadline，后续相关操作均会返回 `context deadline exceeded`
* Request 和 Writer 字段用于获取当前请求的与输出响应
* index 和 handlers 用于 handler 的流程控制；handlers 中存储了当前请求需要执行的所有 handler，index 用于标记当前正在执行的 handler 的索引位
* Keys 用于在 handler 之间传递一些额外的信息
* Error 用于存储整个请求处理过程中的错误
* method 用于检查当前请求的 Method 是否与预定义的相匹配
* engine 字段指向当前 blademaster 的 Engine 实例


####    Context 的方法
`Context` 对外暴露的方法基本上可以分为如下几类：

* 针对 `handlers` 中间件数组的流程控制
* 额外信息传递
* 请求 Request 处理
* 响应 Response 处理

```golang
// 用于 Handler 的流程控制
func (c *Context) Abort()
func (c *Context) AbortWithStatus(code int)
func (c *Context) Bytes(code int, contentType string, data ...[]byte)
func (c *Context) IsAborted() bool
func (c *Context) Next()

// 用户获取或者传递请求的额外信息
func (c *Context) RemoteIP() (cip string)
func (c *Context) Set(key string, value interface{})
func (c *Context) Get(key string) (value interface{}, exists bool)

// 用于校验请求的 payload
func (c *Context) Bind(obj interface{}) error
func (c *Context) BindWith(obj interface{}, b binding.Binding) error

// 用于输出响应
func (c *Context) Render(code int, r render.Render)
func (c *Context) Redirect(code int, location string)
func (c *Context) Status(code int)
func (c *Context) String(code int, format string, values ...interface{})
func (c *Context) XML(data interface{}, err error)
func (c *Context) JSON(data interface{}, err error)
func (c *Context) JSONMap(data map[string]interface{}, err error)
func (c *Context) Protobuf(data proto.Message, err error)
```


#### Handlers 的处理流程
`bm` 框架针对中间件数组 `Handler` 的处理流程（输入为 Request，输出为 Response）大致如下所示，核心是 `Next()` 方法：
```golang
// Next should be used only inside middleware.
// It executes the pending handlers in the chain inside the calling handler.
// See example in godoc.
func (c *Context) Next() {
	c.index++
	for c.index <int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}
```

中间件数组 `Handler` 的运行流程大致如下：
1.  将 `Router` 模块中预先注册的 `middleware` 与其他 `Handler` 合并，放入 `Context` 的 `handlers` 字段，并将 `index` 字段置 `0`
2.  然后通过 `Next()` 方法一个个执行下去，部分 `middleware` 可能想要在过程中中断整个流程，此时可以使用 `Abort()` 方法提前结束处理（重要！）
3.  有些 `middleware` 还想在所有 `Handler` 执行完后再执行部分逻辑，此时可以在自身 `Handler` 中显式调用 `Next()` 方法，并将这些逻辑放在调用了 `Next()` 方法之后

![handler](https://raw.githubusercontent.com/pandaychen/mini-kratos/main/docs/img/bm-handlers.png)


##  0x03    注册中间件 Handlers
`bm` 框架提供了 `UseFunc` 及 `Use` 两个嵌入中间件（middleware）的方法，两者的传入的参数不同，但是逻辑都是将中间件的 `bm.HandlerFunc` 加入 `Context` 的数组 `handlers []HandlerFunc` 中：
```golang
// IRoutes http router interface.
type IRoutes interface {
	UseFunc(...HandlerFunc) IRoutes
	Use(...Handler) IRoutes

	Handle(string, string, ...HandlerFunc) IRoutes
	HEAD(string, ...HandlerFunc) IRoutes
	GET(string, ...HandlerFunc) IRoutes
	POST(string, ...HandlerFunc) IRoutes
	PUT(string, ...HandlerFunc) IRoutes
	DELETE(string, ...HandlerFunc) IRoutes
}

// Use adds middleware to the group, see example code in doc.
func (group *RouterGroup) Use(middleware ...Handler) IRoutes {
	for _, m := range middleware {
		group.Handlers = append(group.Handlers, m.ServeHTTP)    // 添加数组
	}
	return group.returnObj()
}

// UseFunc adds middleware to the group, see example code in doc.
func (group *RouterGroup) UseFunc(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)      // 添加数组
	return group.returnObj()
}
```

middleware 本质上就是一个 `handler`，接口和方法声明如下，传入的参数为 `bm.Context`。如果实现我们自定义的中间件，有两种方式：

1. 实现了 `Handler` 接口（必须实现 `ServeHTTP` 方法），可以作为 engine 的全局中间件使用：`engine.Use(YourHandler)`
2. 声明为 `HandlerFunc` 方法，可以作为 engine 的全局中间件使用：`engine.UseFunc(YourHandlerFunc)`，也可以作为 router 的局部中间件使用：`e.GET("/path", YourHandlerFunc)`

```golang
// Handler responds to an HTTP request.
type Handler interface {
	ServeHTTP(c *Context)
}

// HandlerFunc http request handler function.
type HandlerFunc func(*Context)

// ServeHTTP calls f(ctx).
func (f HandlerFunc) ServeHTTP(c *Context) {
	f(c)
}
```

####    内置中间件实现
1、`Recovery` 中间件，实现 [在此](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/recovery.go)：使用 `defer` 来进行 recovery panic。该中间件会被 `DefaultServer` 默认注册，建议使用 `NewServer` 的话也将其作为首个中间件注册。
```golang
// Recovery returns a middleware that recovers from any panics and writes a 500 if there was one.
func Recovery() HandlerFunc {
	return func(c *Context) {
		defer func() {
			var rawReq []byte
			if err := recover(); err != nil {
				const size = 64 << 10
				buf := make([]byte, size)
				buf = buf[:runtime.Stack(buf, false)]
				if c.Request != nil {
					rawReq, _ = httputil.DumpRequest(c.Request, false)
				}
				pl := fmt.Sprintf("http call panic: %s\n%v\n%s\n", string(rawReq), err, buf)
				fmt.Fprintf(os.Stderr, pl)
				log.Error(pl)
				c.AbortWithStatus(500)
			}
		}()
		c.Next()
	}
}
```

2、`Trace`[中间件](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/trace.go)：用于 `tracing` 设置，并且实现了 `net/http/httptrace` 的接口，能够收集官方库内的调用栈详情。会被 `DefaultServer` 默认注册，建议使用 `NewServer` 的话也将其作为第二个中间件注册。
```golang
func Trace() HandlerFunc {
	return func(c *Context) {
		// handle http request
		// get derived trace from http request header
		t, err := trace.Extract(trace.HTTPFormat, c.Request.Header)
		if err != nil {
			var opts []trace.Option
			if ok, _ := strconv.ParseBool(trace.KratosTraceDebug); ok {
				opts = append(opts, trace.EnableDebug())
			}
			t = trace.New(c.Request.URL.Path, opts...)
		}
		t.SetTitle(c.Request.URL.Path)
		t.SetTag(trace.String(trace.TagComponent, _defaultComponentName))
		t.SetTag(trace.String(trace.TagHTTPMethod, c.Request.Method))
		t.SetTag(trace.String(trace.TagHTTPURL, c.Request.URL.String()))
		t.SetTag(trace.String(trace.TagSpanKind, "server"))
		// business tag
		t.SetTag(trace.String("caller", metadata.String(c.Context, metadata.Caller)))
		// export trace id to user.
		c.Writer.Header().Set(trace.KratosTraceID, t.TraceID())
		c.Context = trace.NewContext(c.Context, t)
		c.Next()
		t.Finish(&c.Error)
	}
}
```
3、`Logger`[中间件](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/logger.go)：用于请求日志记录。会被 `DefaultServer` 默认注册，建议使用 `NewServer` 的话也将其作为第三个中间件注册。在此中间件中也加入了 `metrics` 监控上报。
```golang
// Logger is logger  middleware
func Logger() HandlerFunc {
	const noUser = "no_user"
	return func(c *Context) {
		now := time.Now()
		req := c.Request
		path := req.URL.Path
		params := req.Form
		var quota float64
		if deadline, ok := c.Context.Deadline(); ok {
			quota = time.Until(deadline).Seconds()
		}

		c.Next()

		err := c.Error
		cerr := ecode.Cause(err)
		dt := time.Since(now)
		caller := metadata.String(c, metadata.Caller)
		if caller == "" {
			caller = noUser
		}

		if len(c.RoutePath) > 0 {
			_metricServerReqCodeTotal.Inc(c.RoutePath[1:], caller, req.Method, strconv.FormatInt(int64(cerr.Code()), 10))
			_metricServerReqDur.Observe(int64(dt/time.Millisecond), c.RoutePath[1:], caller, req.Method)
		}

		lf := log.Infov
		errmsg := ""
		isSlow := dt >= (time.Millisecond * 500)
		if err != nil {
			errmsg = err.Error()
			lf = log.Errorv
			if cerr.Code()> 0 {
				lf = log.Warnv
			}
		} else {
			if isSlow {
				lf = log.Warnv
			}
		}
		lf(c,
			log.KVString("method", req.Method),
			log.KVString("ip", c.RemoteIP()),
			log.KVString("user", caller),
			log.KVString("path", path),
			log.KVString("params", params.Encode()),
			log.KVInt("ret", cerr.Code()),
			log.KVString("msg", cerr.Message()),
			log.KVString("stack", fmt.Sprintf("%+v", err)),
			log.KVString("err", errmsg),
			log.KVFloat64("timeout_quota", quota),
			log.KVFloat64("ts", dt.Seconds()),
			log.KVString("source", "http-access-log"),
		)
	}
}
```

4、`CSRF` [中间件](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/csrf.go)：用于防跨站请求。其实现原理就是：
*   验证 HTTP `Referer` 字段：访问一个安全受限页面的请求必须来自于同一个网站
*   验证随机 token：在 HTTP 请求中以参数的形式加入一个随机 token，并在服务器端使用拦截器验证这个 token

`CSRF` 的实现如下：
```golang
func matchHostSuffix(suffix string) func(*url.URL) bool {
	return func(uri *url.URL) bool {
		return strings.HasSuffix(strings.ToLower(uri.Host), suffix)
	}
}

func matchPattern(pattern *regexp.Regexp) func(*url.URL) bool {
	return func(uri *url.URL) bool {
		return pattern.MatchString(strings.ToLower(uri.String()))
	}
}
// CSRF returns the csrf middleware to prevent invalid cross site request.
// Only referer is checked currently.
func CSRF(allowHosts []string, allowPattern []string) HandlerFunc {
	validations := []func(*url.URL) bool{}

	addHostSuffix := func(suffix string) {
		validations = append(validations, matchHostSuffix(suffix))
	}
	addPattern := func(pattern string) {
		validations = append(validations, matchPattern(regexp.MustCompile(pattern)))
	}

	for _, r := range allowHosts {
		addHostSuffix(r)
	}
	for _, p := range allowPattern {
		addPattern(p)
	}

	return func(c *Context) {
        // 验证 Referer 部分
		referer := c.Request.Header.Get("Referer")
		if referer == "" {
			log.V(5).Info("The request's Referer or Origin header is empty.")
			c.AbortWithStatus(403)
			return
		}
        illegal := true
        // 验证自定义的 csrf 防护逻辑
		if uri, err := url.Parse(referer); err == nil && uri.Host != "" {
			for _, validate := range validations {
				if validate(uri) {
					illegal = false
					break
				}
			}
		}
		if illegal {
			log.V(5).Info("The request's Referer header `%s` does not match any of allowed referers.", referer)
			c.AbortWithStatus(403)
			return
		}
	}
}
```

此中间件的使用例子如下：
```golang
e := bm.DefaultServer(nil)
// 挂载自适应限流中间件到 bm engine，使用默认配置
csrf := bm.CSRF([]string{"abc.com"}, []string{"/a/api"})
e.Use(csrf)
// 或者
e.GET("/a/api", csrf, myHandler)
```

5、`CORS`[中间件](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/cors.go)，主要实现用于跨域允许请求。
*   使用该中间件进行全局注册后，可 "省略" 单独为 `OPTIONS` 请求注册路由，如示例一。
*   使用该中间单独为某路由注册，需要为该路由再注册一个 `OPTIONS` 方法的同路径路由，如示例二。

示例一：
```go
e := bm.DefaultServer(nil)
// 挂载自适应限流中间件到 bm engine，使用默认配置
cors := bm.CORS([]string{"github.com"})
e.Use(cors)
// 该路由可以默认针对 OPTIONS /api 的跨域请求支持
e.POST("/api", myHandler)
```

示例二：
```go
e := bm.DefaultServer(nil)
// 挂载自适应限流中间件到 bm engine，使用默认配置
cors := bm.CORS([]string{"github.com"})
// e.Use(cors) 不进行全局注册
e.OPTIONS("/api", cors, myHandler) // 需要单独为 / api 进行 OPTIONS 方法注册
e.POST("/api", cors, myHandler)
```

6、自适应限流 [中间件](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/ratelimit.go)：实现方式和 [Kratos 源码分析：限流器 Limiter](https://pandaychen.github.io/2020/07/12/KRATOS-LIMITER/) 类似：
```golang
// RateLimiter bbr middleware.
type RateLimiter struct {
	group   *bbr.Group
	logTime int64
}

// NewRateLimiter return a ratelimit middleware.
func NewRateLimiter(conf *bbr.Config) (s *RateLimiter) {
	return &RateLimiter{
		group:   bbr.NewGroup(conf),
		logTime: time.Now().UnixNano(),
	}
}

func (b *RateLimiter) printStats(routePath string, limiter limit.Limiter) {
	now := time.Now().UnixNano()
	if now-atomic.LoadInt64(&b.logTime) > int64(time.Second*3) {
		atomic.StoreInt64(&b.logTime, now)
		log.Info("http.bbr path:%s stat:%+v", routePath, limiter.(*bbr.BBR).Stat())
	}
}

// Limit return a bm handler func.
func (b *RateLimiter) Limit() HandlerFunc {
	return func(c *Context) {
		uri := fmt.Sprintf("%s://%s%s", c.Request.URL.Scheme, c.Request.Host, c.Request.URL.Path)
		limiter := b.group.Get(uri)
		done, err := limiter.Allow(c)
		if err != nil {
			_metricServerBBR.Inc(uri, c.Request.Method)
			c.JSON(nil, err)
			c.Abort()
			return
		}
		defer func() {
			done(limit.DoneInfo{Op: limit.Success})
			b.printStats(uri, limiter)
		}()
		c.Next()
	}
}
```

使用自适应限流中间件的方法如下：
```golang
e := bm.DefaultServer(nil)
// 挂载自适应限流中间件到 bm engine，使用默认配置
limiter := bm.NewRateLimiter(nil)
e.Use(limiter.Limit())
// 或者
e.GET("/api", csrf, myHandler)
```

##  0x04    实现 bm 的中间件
本小节给出实现自定义 bm 中间件的方法。从上一节看内置中间件的写法，一般可以使用如下的通用格式：
```golang
func MyOwnMiddleware() HandlerFunc {
    ...
    // 前置初始化或者方法定义等
    ...
	return func(c *Context) {
        // 如果需要在请求结束时做一些事情，那么使用 defer 来完成，如计时、统计等
        ...
        // 实现我们自己的逻辑
        // 如果需要退出中间件执行，使用 c.AbortWithStatus(XXX) 退出整个逻辑
        ...
		c.Next()
	}
}
```

基于 `bm` 框架的 handler 机制，可以自定义很多 middleware 进行通用的业务处理，比如用户登录鉴权。接下来就以鉴权为例，代码实现如下：

```golang
type Demo struct {
	Key   string
	Value   string
}
// ServeHTTP implements from Handler interface
func (d *Demo) ServeHTTP(ctx *bm.Context) {
	ctx.Set(d.Key, d.Value)
}

e := bm.DefaultServer(nil)
d := &Demo{}

// Handler 使用如下：
e.Use(d)

// HandlerFunc 使用如下：
e.UseFunc(d.ServeHTTP)
e.GET("/path", d.ServeHTTP)

// 或者只有方法
myHandler := func(ctx *bm.Context) {
    // some code
}
e.UseFunc(myHandler)
e.GET("/path", myHandler)
```

#### 全局中间件 DefaultServer
`bm` 框架的默认 [`DefaultServer` 全局变量](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/server.go#L214) 中，该方法默认创建一个 `bm engine`，默认开启了 `3` 个拦截器。`bm engine`，并注册 `Recovery(), Trace(), Logger()` 这 `3` 个 middlerware 用于全局 handler 处理，优先级从前到后。
```golang
func DefaultServer(conf *ServerConfig) *Engine {
	engine := NewServer(conf)
	engine.Use(Recovery(), Trace(), Logger())
	return engine
}
```

如果想要将使用自定义的 middleware 注册进全局，可以继续调用 `Use` 方法如下：

```golang
engine.Use(YourMiddleware())
```

此方法会将 `YourMiddleware` 追加到已有的全局 middleware 后执行。如果需要全部自定义全局执行的 middleware，可以使用 `NewServer` 方法创建一个无 middleware 的 engine 对象，然后使用 `engine.Use/UseFunc` 进行注册。

#### 局部中间件
先来看一段鉴权伪代码示例 ([auth 示例代码](https://github.com/go-kratos/kratos/tree/master/example/blademaster/middleware/auth))，可见 `bm` 框架的中间件调用方式还是比较灵活的。
```go
func Example() {
	myHandler := func(ctx *bm.Context) {
		mid := metadata.Int64(ctx, metadata.Mid)
		ctx.JSON(fmt.Sprintf("%d", mid), nil)
	}

	authn := auth.New(&auth.Config{DisableCSRF: false})

	e := bm.DefaultServer(nil)

	// "/user" 接口必须保证登录用户才能访问，那么我们加入 "auth.User" 来确保用户鉴权通过，才能进入 myHandler 进行业务逻辑处理
	e.GET("/user", authn.User, myHandler)
	// "/guest" 接口访客用户就可以访问，但如果登录用户我们需要知道 mid，那么我们加入 "auth.Guest" 来尝试鉴权获取 mid，但肯定会继续执行 myHandler 进行业务逻辑处理
	e.GET("/guest", authn.Guest, myHandler)

    // "/owner" 开头的所有接口，都需要进行登录鉴权才可以被访问，那可以创建一个 group 并加入 "authn.User"
	o := e.Group("/owner", authn.User)
	o.GET("/info", myHandler) // 该 group 创建的 router 不需要再显示的加入 "authn.User"
	o.POST("/modify", myHandler) // 该 group 创建的 router 不需要再显示的加入 "authn.User"

	e.Start()
}
```

##  0x05    分析自定义中间件 auth
本小节，分析下自定义 [中间件 `auth`](https://github.com/go-kratos/kratos/blob/master/example/blademaster/middleware/auth/auth.go) 的实现。

##  0x06    参考
-   [明白了，原来 Go web 框架中的中间件都是这样实现的](https://colobu.com/2019/08/21/decorator-pattern-pipeline-pattern-and-go-web-middlewares/)
-   [blademaster 的项目代码](https://github.com/go-kratos/kratos/tree/master/pkg/net/http/blademaster)
-	[Go 语言高级编程 - 5.3 中间件](https://chai2010.gitbooks.io/advanced-go-programming-book/content/ch5-web/ch5-03-middleware.html)