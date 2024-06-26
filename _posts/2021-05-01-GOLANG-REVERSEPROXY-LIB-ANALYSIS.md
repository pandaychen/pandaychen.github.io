---
layout: post
title: Golang ReverseProxy 分析
subtitle: 原生库的反向代理代码分析
date: 2021-07-01
author: pandaychen
header-img:
catalog: true
category: false
tags:
  - ReverseProxy
  - 反向代理
---

## 0x00 前言

在工作项目中，曾使用 `gin` 与 `httputil.ReverseProxy` 实现了认证网关和反向代理的功能，该认证网关的主要流程为：<br>
1、 通过 `gin` 实现的 https 网关 接收浏览器 Web 发起的请求 <br>
2、 网关通过 `httputil.ReverseProxy` 发起一个带 API 签名的 HTTP 请求给后台 CGI 服务，实现代理功能 <br>
3、 网关接收到后台 CGI 的服务响应信息 <br>
4、 网关将响应回复给客户端浏览器 <br>

本文分析下 `httputil.ReverseProxy` 的实现细节。[httputil.ReverseProxy](https://golang.org/pkg/net/http/httputil/#ReverseProxy) 是官方提供的一个反向代理实现，可直接拿来用，在网关或代理类服务实现非常方便，这篇文章简单分析下其实现。
<br>

`httputil.ReverseProxy` 具有如下特点：

- 自定义修改响应包体
- 连接池
- 错误信息自定义处理
- 支持 websocket 服务
- 自定义负载均衡
- https 代理
- URL 重写

从反向代理的架构上来看，`httputil.ReverseProxy`主要是帮助开发者完成了后面的转发及响应逻辑：
![proxy2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/proxy/reverseproxy2.png)

## 0x01 核心结构与方法

#### httputil.ReverseProxy 结构

`httputil.ReverseProxy` 包含两个重要的（成员）属性： `Director` 和 `ModifyResponse`，这两个属性都是函数类型。

1、`Director` 属性 <br>
当接收到客户端请求时，`ServeHTTP` 函数首先调用 `Director` 函数对接受到的请求体进行修改，例如修改请求的目标地址、请求头等；然后使用修改后的请求体发起新的请求

2、`ModifyResponse` 属性 <br>
-	接收到响应后，调用 `ModifyResponse` 函数对响应进行修改，最后将修改后的响应体拷贝并响应给客户端，这样就实现了反向代理的整个流程
-	用于修改响应结果及HTTP状态码，当返回结果error不为空时，会调用`ReverseProxy.ErrorHandler`

3、`ErrorHandler `属性<br>
用于处理后端和`ModifyResponse`成员返回的错误信息，默认将返回传递过来的错误信息，并返回HTTP `502`错误码

```golang
// 处理进来的请求，并发送转发至后端 server 实现反向代理，并将响应请求回传给客户端
type ReverseProxy struct {
    // Director must be a function which modifies
    // the request into a new request to be sent
    // using Transport. Its response is then copied
    // back to the original client unmodified.
    // Director must not access the provided Request
    // after returning.
    Director func(*http.Request)  // 通过 transport 可修改请求，响应体将原封不动的返回

    // The transport used to perform proxy requests.
    // If nil, http.DefaultTransport is used.
    Transport http.RoundTripper // 通过 transport 设置转发 http-client 的参数

    // FlushInterval specifies the flush interval
    // to flush to the client while copying the
    // response body.
    // If zero, no periodic flushing is done.
    // A negative value means to flush immediately
    // after each write to the client.
    // The FlushInterval is ignored when ReverseProxy
    // recognizes a response as a streaming response, or
    // if its ContentLength is -1; for such responses, writes
    // are flushed to the client immediately.
    FlushInterval time.Duration

    // ErrorLog specifies an optional logger for errors
    // that occur when attempting to proxy the request.
    // If nil, logging is done via the log package's standard logger.
    ErrorLog *log.Logger // Go 1.4  // 默认为 std.err，可用于自定义 logger

    // BufferPool optionally specifies a buffer pool to
    // get byte slices for use by io.CopyBuffer when
    // copying HTTP response bodies.
    BufferPool BufferPool // Go 1.6 // 用于执行 io.CopyBuffer 复制响应体，将其存放至 byte 切片

    // ModifyResponse is an optional function that modifies the
    // Response from the backend. It is called if the backend
    // returns a response at all, with any HTTP status code.
    // If the backend is unreachable, the optional ErrorHandler is
    // called without any call to ModifyResponse.
    //
    // If ModifyResponse returns an error, ErrorHandler is called
    // with its error value. If ErrorHandler is nil, its default
    // implementation is used.
    ModifyResponse func(*http.Response) error // Go 1.8   // 用于修改响应结果及 HTTP 状态码，当返回结果 error 不为空时，会调用 ErrorHandler

    // ErrorHandler is an optional function that handles errors
    // reaching the backend or errors from ModifyResponse.
    //
    // If nil, the default is to log the provided error and return
    // a 502 Status Bad Gateway response.
    ErrorHandler func(http.ResponseWriter, *http.Request, error) // Go 1.11 // 用于处理后端和 ModifyResponse 返回的错误信息，默认将返回传递过来的错误信息，并返回 HTTP 502
}
```

####	ReverseProxy 的成员说明
-	`Transport`：反向代理客户端连接池，这个参数特别注意，现网 **建议采用一个设置适当参数的全局唯一连接池**，否则在高并发场景中会出现连接暴涨 / 泄漏的风险
-	`Director`：将请求修改为使用`Transfor`发送的新请求，其响应后被复制回未修改的原始客户端，`Director`在返回后不得访问提供的请求
-	`FlushInterval`：指定在复制响应主体时刷新到客户端的刷新间隔。如果为`nil`，则不进行周期性刷新
-	`ErrorLog`：指定一个可选的记录器，用于尝试代理请求时发生的错误，如果为`nil`，则日志记录将通过日志包的标准记录器转到`os.Stderr`
-	`BufferPool`：可以选择指定一个缓冲池来获取字节片，以便在复制HTTP响应体时由`io.CopyBuffer`使用，是一个不错的优化途径
-	`ModifyResponse`：用于修改来自后端的响应，如果它返回一个错误，代理将返回一个`StatusBadGateway`错误，同时调用`ErrorLog`的方法


#### NewSingleHostReverseProxy 方法

`NewSingleHostReverseProxy` 方法返回一个新的 `ReverseProxy`，将 `URLs` 请求路由到传入参数 `target` 的指定的 `Scheme`, `Host` 以及 `Base path`，也是默认的 `director` 配置，在 `NewSingleHostReverseProxy` 中源码已经对传入的 `URLs` 进行解析并且完成了 `Director` 的修改

注意：`NewSingleHostReverseProxy`将URL路由到目标中提供的方案，主机和基本路径等重要参数；假设目标URI是`/base`，请求的URI是`/cgi1`，那么请求会被被反向代理到`http://x.x.x.x./base/cgi1`，此外，`ReverseProxy`  不会rewrite Host header，需要重写Host，可在`ReverseProxy.Director`自定义

```golang
// NewSingleHostReverseProxy returns a new ReverseProxy that routes
// URLs to the scheme, host, and base path provided in target. If the
// target's path is"/base"and the incoming request was for"/dir",
// the target request will be for /base/dir.
// NewSingleHostReverseProxy does not rewrite the Host header.
// To rewrite Host headers, use ReverseProxy directly with a custom
// Director policy.
func NewSingleHostReverseProxy(target *url.URL) *ReverseProxy {

	// 获取请求参数，例如请求的是/cgi1?id=123，那么rawQuery就是`id=123`
    targetQuery := target.RawQuery
    // 自定义 director 配置
    director := func(req *http.Request) {
      req.URL.Scheme = target.Scheme	// http/https
      req.URL.Host = target.Host	 // 主机名（ip/端口 或 域名/端口）
      req.URL.Path = singleJoiningSlash(target.Path, req.URL.Path)	 // 请求URL拼接

	  
      if targetQuery == ""|| req.URL.RawQuery =="" {
        req.URL.RawQuery = targetQuery + req.URL.RawQuery
      } else {
		// 使用"&"符号拼接请求参数
        req.URL.RawQuery = targetQuery + "&" + req.URL.RawQuery
      }

	  // 若"User-Agent" 这个header不存在，则置空
      if _, ok := req.Header["User-Agent"]; !ok {
        // explicitly disable User-Agent so it's not set to default value
        req.Header.Set("User-Agent", "")
      }
    }
    return &ReverseProxy{Director: director}
}

// singleJoiningSlash URL 拼接的方法
func singleJoiningSlash(a, b string) string {
	aslash := strings.HasSuffix(a, "/")
	bslash := strings.HasPrefix(b, "/")
	switch {
	case aslash && bslash:      // 如果 a,b 都存在，则去掉后者第一个字符，也就是 "/" 后拼接
		return a + b[1:]
	case !aslash && !bslash:  // 如果 a,b 都不存在，则在两者间添加 "/"
		return a + "/" + b
	}
	return a + b  // 否则直接拼接到一块
}
```

`NewSingleHostReverseProxy` 的意义是告知开发者在实现反向代理功能时，至少需要修改如下 `req` 字段：
-	`req.URL.Scheme`
-	`req.URL.Host`
-	`req.URL.Path`
-	`req.URL.RawQuery`


## 0x02 ServeHTTP 方法

基于先前的经验，实例化的 `httputil.Reverseproxy` 对象必须实现 `ServeHTTP` 方法才能使用 `Handler` 方式调用，本小节具体分析下 [ServeHTTP](https://golang.org/src/net/http/httputil/reverseproxy.go?s=6664:6739#L202) 方法：

####	`ServeHTTP`完整代码

```golang
// 实现了 ServeHTTP 接口
//rw：响应（客户端）的数据
//req：来自客户端的请求
func (p *ReverseProxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	// 设置需要使用的 transport（默认）
	transport := p.Transport
	if transport == nil {
		transport = http.DefaultTransport
	}

	/*
	 从原始请求 req 中取出上下文信息，通过类型断言 rw.(http.CloseNotifier)，来判断连接 (请求) 是否终止。
如果终止了则直接放弃本次请求，即调用 cancel()，取消此上下文，当然对应的资源也就释放了。
其中 http.CloseNotifier 是一个接口，只有一个方法 CloseNotify() <-chan bool，作用是检测连接是否断开
	*/

	//ctx := req.Context() 这个ctx是为了控制向下一个节点请求时，前面的连接主动退出的控制ctx
	ctx := req.Context()
	if cn, ok := rw.(http.CloseNotifier); ok {
		// 客户端连接终止
		var cancel context.CancelFunc
		ctx, cancel = context.WithCancel(ctx)
		defer cancel()
		notifyChan := cn.CloseNotify()
		go func() {
			select {
			case <-notifyChan:
				cancel()
			case <-ctx.Done():
			}
		}()
	}

	// 拷贝原始请求 req 的上下文信息，并赋值给对外请求的 request（outreq）
	outreq := req.Clone(ctx)
	if req.ContentLength == 0 {
		outreq.Body = nil // Issue 16036: nil Body for http.Transport retries
	}
	// 如果上下文中 header 为 nil，则使用 http.Header 初始化
	if outreq.Header == nil {
		outreq.Header = make(http.Header) // Issue 33142: historical behavior was to always allocate
	}

	// 将拷贝后的 outreq 传递给 Director 控制器, Director 支持自定义修改，例子见 NewSingleHostReverseProxy
	p.Director(outreq)
	// 将请求头的 Close 字段置为 false，即当客户端请求触发到下游的时候，会产生一条链接，保证这条链接是可复用的
	outreq.Close = false


	//Upgrade 头特殊处理，获取 Upgrade 信息，先判断请求头 Connection 字段中是否包含 Upgrade 单词
	reqUpType := upgradeType(outreq.Header)
	// 删除 http.header['Connection'] 中列出的 hop-by-hop 头信息，所有 Connection 中设置的 key 都删除掉
	removeConnectionHeaders(outreq.Header)

	// Remove hop-by-hop headers to the backend. Especially
	// important is "Connection" because we want a persistent
	// connection, regardless of what the client sent to us.

	// 删除 outreq（向后端的请求）中 hop-by-hop 类型的 header，此是客户端和代理之间的消息头，与是否往下传递的 header 信息没有联系，往下游传递的信息里不应该包含这些逐段消息头
	for _, h := range hopHeaders {
		outreq.Header.Del(h)
	}

	// Issue 21096: tell backend applications that care about trailer support
	// that we support trailers. (We do, but we don't go out of our way to
	// advertise that unless the incoming client request thought it was worth
	// mentioning.) Note that we look at req.Header, not outreq.Header, since
	// the latter has passed through removeConnectionHeaders.
	if httpguts.HeaderValuesContainsToken(req.Header["Te"], "trailers") {
		outreq.Header.Set("Te", "trailers")
	}

	// After stripping all the hop-by-hop connection headers above, add back any
	// necessary for protocol upgrades, such as for websockets.
	// 删除上面所有的 hop-by-hop 头后，再添加协议升级所需的所有内容，见 websocket 代理实现
	if reqUpType != "" {
		outreq.Header.Set("Connection", "Upgrade")
		outreq.Header.Set("Upgrade", reqUpType)
	}

	// 追加 chientIP 信息，即设置 X-Forwarded-For，以逗号 + 空格分隔
	if clientIP, _, err := net.SplitHostPort(req.RemoteAddr); err == nil {
		// If we aren't the first proxy retain prior
		// X-Forwarded-For information as a comma+space
		// separated list and fold multiple headers into one.
		prior, ok := outreq.Header["X-Forwarded-For"]
		omit := ok && prior == nil // Issue 38079: nil now means don't populate the header
		if len(prior) > 0 {
			clientIP = strings.Join(prior, ",") + "," + clientIP
		}
		if !omit {
			outreq.Header.Set("X-Forwarded-For", clientIP)
		}
	}

	// 执行真正的代理功能：向后端发送请求数据，transport.RoundTrip() 的作用就是执行一个 HTTP 事务，根据请求返回响应
	res, err := transport.RoundTrip(outreq)
	if err != nil {
		p.getErrorHandler()(rw, outreq, err)
		return
	}

	// 注意：后端的响应保存在 res 中

	// Deal with 101 Switching Protocols responses: (WebSocket, h2c, etc)
	// 处理升级协议的请求，响应的状态码为 101 才考虑升级（如 websocket 协议）
	if res.StatusCode == http.StatusSwitchingProtocols {
		if !p.modifyResponse(rw, res, outreq) {
			return
		}
		// 处理协议升级，见 handleUpgradeResponse 方法
		p.handleUpgradeResponse(rw, outreq, res)
		return
	}

	// 删除后端响应数据（res）中一些无用的头部字段，首先是删除 Connection 中的消息头，然后删除 hopHeaders
	removeConnectionHeaders(res.Header)
	for _, h := range hopHeaders {
		res.Header.Del(h)
	}

  	// 修改返回的内容，modifyResponse 底层是调用 ReverseProxy 结构体中的 ModifyResponse 函数，函数内容由开发者自定义
	if !p.modifyResponse(rw, res, outreq) {
		return
	}

  	// 拷贝头部数据，将后端响应的 header 复制到返回给客户端的响应结构中
	//rw.Header()：前置 (客户端) 的 header 数据，rw 为 ResponseWriter 接口
	//res.Header：后端（代理）返回的 header 数据，res 为 roundTrip.(outreq) 的返回 Response 结构
	copyHeader(rw.Header(), res.Header)

	// The "Trailer" header isn't included in the Transport's response,
	// at least for *http.Transport. Build it up from Trailer.
	// 处理 Trailer 头部数据（如果有），遍历，拼接，然后将内容加到上游的头部的 Trailer 字段中
	announcedTrailers := len(res.Trailer)
	if announcedTrailers > 0 {
		trailerKeys := make([]string, 0, len(res.Trailer))
		for k := range res.Trailer {
			trailerKeys = append(trailerKeys, k)
		}
		rw.Header().Add("Trailer", strings.Join(trailerKeys, ","))
	}

  	// 将后端服务器响应的状态码写入前置（客户端）响应的状态码中
	rw.WriteHeader(res.StatusCode)

	// 周期性刷新内容到（给客户端的） Response 中，周期性刷新是指刷新频率（时间间隔）
	// 在 flushInterval(req, res) 方法中，控制时间间隔是通过读取 ReverseProxy 结构体中 FlushInterval 字段（可由开发者定义）
	err = p.copyResponse(rw, res.Body, p.flushInterval(res))
	if err != nil {
		defer res.Body.Close()
		// Since we're streaming the response, if we run into an error all we can do
		// is abort the request. Issue 23643: ReverseProxy should use ErrAbortHandler
		// on read error while copying body.
		if !shouldPanicOnCopyError(req) {
			p.logf("suppressing panic for copyResponse error in test; copy error: %v", err)
			return
		}
		panic(http.ErrAbortHandler)
	}
	res.Body.Close() // close now, instead of defer, to populate res.Trailer

	if len(res.Trailer) > 0 {
		// Force chunking if we saw a response trailer.
		// This prevents net/http from calculating the length for short
		// bodies and adding a Content-Length.
		if fl, ok := rw.(http.Flusher); ok {
			fl.Flush()
		}
	}

	if len(res.Trailer) == announcedTrailers {
		copyHeader(rw.Header(), res.Trailer)
		return
	}

  // 最后：Trailer 的信息放在了响应 body 之后，是独立存放响应信息中的，即 res.Trailer 中
  // 见：https://cloud.tencent.com/developer/section/1190006
	for k, vv := range res.Trailer {
		k = http.TrailerPrefix + k
		for _, v := range vv {
			rw.Header().Add(k, v)
		}
	}
}

// header copy
func copyHeader(dst, src http.Header) {
	for k, vv := range src {
		for _, v := range vv {
			dst.Add(k, v)
		}
	}
}

// Hop-by-hop headers. These are removed when sent to the backend.
// As of RFC 7230, hop-by-hop headers are required to appear in the
// Connection header field. These are the headers defined by the
// obsoleted RFC 2616 (section 13.5.1) and are used for backward
// compatibility.
var hopHeaders = []string{
	"Connection",
	"Proxy-Connection", // non-standard but still sent by libcurl and rejected by e.g. google
	"Keep-Alive",
	"Proxy-Authenticate",
	"Proxy-Authorization",
	"Te",      // canonicalized version of "TE"
	"Trailer", // not Trailers per URL above; https://www.rfc-editor.org/errata_search.php?eid=4522
	"Transfer-Encoding",
	"Upgrade",
}

func upgradeType(h http.Header) string {
	if !httpguts.HeaderValuesContainsToken(h["Connection"], "Upgrade") {
		return ""
	}
	return strings.ToLower(h.Get("Upgrade"))
}

// removeConnectionHeaders removes hop-by-hop headers listed in the "Connection" header of h.
// See RFC 7230, section 6.1
func removeConnectionHeaders(h http.Header) {
	for _, f := range h["Connection"] {
		for _, sf := range strings.Split(f, ",") {
			if sf = textproto.TrimString(sf); sf != "" {
				h.Del(sf)
			}
		}
	}
}
```


上面的通信流程如下图所示：
![reverseproxy](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/proxy/reverseproxy1.png)

####	小结
本小节通过步骤拆解下上面`ReverseProxy.ServeHTTP`的流程：

1、转发器（用来发送实际的请求）`Transport`设置<br>
转发请求必须要一个http客户端，这里支持开发者自行传入的`Transport`
```GOLANG
transport := p.Transport
if transport == nil {
    transport = http.DefaultTransport
}
```

2、验证请求是否被主动终止<br>

开始处理原始请求，设置一个`ctx`、`cancel`用于及时取消代理转发的过程

先从请求中取出上下文，即`ctx := req.Context()`，然后这里会循环判断请求是否终止，拿到当前请求`rw`的`http.ResponseWriter`，然后通过`http.CloseNotifier`获取到`cn.CloseNotify()`，然后就单独启动goroutine来监视`notifyChan` 是否有消息，如果有消息触发就说明，直接触发`cancel()`停止后面的流程

如果终止了则直接放弃本次请求，即调用`cancel()`，取消此context，当然对应的资源也就被释放了

PS：`http.CloseNotifier`只有一个方法`CloseNotify() <-chan bool`，作用是检测连接是否断开，通常的触发场景如：用户关闭浏览器（页面未加载完成，网络断开等中断请求）

```GOLANG
// 获取请求的上下文
ctx := req.Context()
if cn, ok := rw.(http.CloseNotifier); ok {
    var cancel context.CancelFunc
    ctx, cancel = context.WithCancel(ctx)
    defer cancel()
    // 获取到请求信号channel
    notifyChan := cn.CloseNotify()
    go func() {
        select {
        case <-notifyChan:
            cancel()
        case <-ctx.Done():
			//正常退出
        }
    }()
}
```

3、设置下游请求（`outreq`）的`ctx`信息，并拷贝上游请求的Header到下游请求（即对外请求的 request）<br>
从获取上游的 request 用于代理时访问下游的 request，使用`Request.Clone`[方法](https://cs.opensource.google/go/go/+/refs/tags/go1.19.1:src/net/http/request.go;l=371) Copy相关字段

```GOLANG
outreq := req.Clone(ctx)
if req.ContentLength == 0 {
	outreq.Body = nil // Issue 16036: nil Body for http.Transport retries
}
if outreq.Body != nil {
	defer outreq.Body.Close()
}
if outreq.Header == nil {
	outreq.Header = make(http.Header) // Issue 33142: historical behavior was to always allocate
}
```

`Request.Clone`方法如下：
```GOLANG
// Clone returns a deep copy of r with its context changed to ctx.
// The provided ctx must be non-nil.
//
// For an outgoing client request, the context controls the entire
// lifetime of a request and its response: obtaining a connection,
// sending the request, and reading the response headers and body.
func (r *Request) Clone(ctx context.Context) *Request {
	if ctx == nil {
		panic("nil context")
	}
	r2 := new(Request)
	*r2 = *r
	r2.ctx = ctx
	r2.URL = cloneURL(r.URL)
	if r.Header != nil {
		r2.Header = r.Header.Clone()
	}
	if r.Trailer != nil {
		r2.Trailer = r.Trailer.Clone()
	}
	if s := r.TransferEncoding; s != nil {
		s2 := make([]string, len(s))
		copy(s2, s)
		r2.TransferEncoding = s2
	}
	r2.Form = cloneURLValues(r.Form)
	r2.PostForm = cloneURLValues(r.PostForm)
	r2.MultipartForm = cloneMultipartForm(r.MultipartForm)
	return r2
}
```

4、对上一步Clone得到的`outreq`做合法性检查：Header检查<br>
```golang
//如果上下文中 header 为 nil，则使用 http 的 默认header 给该 ctx
if outreq.Header == nil {
	outreq.Header = make(http.Header) // Issue 33142: historical behavior was to always allocate
}
```

5、修改下游请求`outreq`的http相关属性<br>
这是按照开发者的定义，来修改`outreq`满足下游请求及Header字段的需要；同时对`outreq`的Close字段设置为`false`，这样在`net.http`包的实现中，保证了这条连接可以被复用，提高性能

```GOLANG
p.Director(outreq)
outreq.Close = false
```

6、对`outreq`可能存在的Upgrade头进行特殊处理<br>
-	先判断请求头 Connection 字段中是否包含 Upgrade 关键字，有的话取出返回Upgrade，没有返回空字符串
-	然后删除 `http.header['Connection']`中列出的 hop-by-hop 头信息（RFC 7230, section 6.1），所有 Connection 中设置的 key 都删除掉

```GOLANG
reqUpType := upgradeType(outreq.Header)
removeConnectionHeaders(outreq.Header)
```

7、删除`outreq`下游请求里面的逐跳标题<br>
因为这里需要一个持久的连接，而不管客户端发送给我们什么，但是针对`Te`、`trailers`仍然做保留

为何要做此处理呢？因为**逐段消息头hop-by-hop是客户端和第一层Proxy之间的消息头，与是否往下传递的 header 信息没有联系，往下游传递的信息里不应该包含这些逐段消息头**

注意：本过程删除的是下游请求 `outreq` 的 Header。此外，在下面第`12`步骤也有删除，但是是删除响应的 Header，注意区分

```GOLANG
for _, h := range hopHeaders {
	//处理 hop-by-hop 的 header，除了 Te 和 trailers 都删除
	outreq.Header.Del(h)
}
if httpguts.HeaderValuesContainsToken(req.Header["Te"], "trailers") {
	outreq.Header.Set("Te", "trailers")
}
```

8、判断请求升级<br>
如果原始请求的Upgrade不为空，那么再重新设置进去

```GOLANG
//删除上面所有的 hop-by-hop 头后，添加回协议升级所需的所有内容
if reqUpType != "" {
	outreq.Header.Set("Connection", "Upgrade")
	outreq.Header.Set("Upgrade", reqUpType)
}
```

9、追加clientIP信息<br>
通过req请求头里面的RemoteAddr信息，（更新）追加到请求头的`X-Forwarded-For`信息中

```GOLANG
if clientIP, _, err := net.SplitHostPort(req.RemoteAddr); err == nil {
		prior, ok := outreq.Header["X-Forwarded-For"]
		omit := ok && prior == nil // Issue 38079: nil now means don't populate the header
		if len(prior) > 0 {
			clientIP = strings.Join(prior, ", ") + ", " + clientIP
		}
		if !omit {
			outreq.Header.Set("X-Forwarded-For", clientIP)
		}
```


10、向下游请求数据<br>
到此，上面针对下游连接`outreq`的设置已经全部完成，通过连接池，直接请求下游数据。如果请求失败，抛出一个`ErrorHandler`方法并执行；记得上面的ctx了吗？如果此时客户端主动退出，那么将触发上面步骤的`cancel`

`transport.RoundTrip()`的作用就是执行一个 HTTP 事务，根据请求返回响应

```GOLANG
res, err := transport.RoundTrip(outreq)
	if err != nil {
		p.getErrorHandler()(rw, outreq, err)
		return
	}
```

11、处理升级协议请求<br>
上面的`transport.RoundTrip(outreq)`已经完成，如果状态码是`101`，那么表示要进行请求升级（典型如Websocket协议），接着执行`modifyResponse` ，此时开发者可以针对`101`情况做一些特殊的处理。（参考`handleUpgradeResponse`方法，内部hijack原始的tcp连接，并且同时启动`2`个goroutine持续交换数据直到一方关闭），最后执行特殊的请求升级的返回

```GOLANG
if res.StatusCode == http.StatusSwitchingProtocols {
	if !p.modifyResponse(rw, res, outreq) {
		return
	}
	//处理upgrade
    p.handleUpgradeResponse(rw, outreq, res)
	return
}
```

12、移除下游无用header头<br>
首先是删除 Connection中的消息头，然后是删除 `hopHeaders`定义的消息头（移除一些无用的Header）
```golang
//删除下游响应数据中一些无用的头部字段
removeConnectionHeaders(res.Header)
for _, h := range hopHeaders {
	res.Header.Del(h)
}
```

13、修改返回内容`res`<br>
通过开发者实现的modifyResponse方法来处理后端服务器的响应数据`res`

```GOLANG
if !p.modifyResponse(rw, res, outreq) {
	return
}
```

14、拷贝头部的数据<br>
本质上就是代理返回了什么结果，就将内容返回客户端
-	`rw.Header()`：上游（客户端）的头部数据，`rw` 为 `ResponseWriter`
-	`res.Header`：下游（后端服务）返回的头部数据，`res` 为 `roundTrip.(outreq)`的返回，`http.Response`结构

```GOLANG
copyHeader(rw.Header(), res.Header)
```

15、写入状态码和刷新Response<br>
接下来就写入返回的状态码，和周期性刷新内容daoresponse中，将下游响应的状态码写入上游（即客户端）的状态码中。`res.StatusCode`是下游返回的状态码数据。利用`copyResponse`方法持续的将`res.Body`的数据复制到`rw`中，这样就完成了下游响应数据到上游响应数据的传输

这里`http.ReverseProxy`提供的参数`FlushInterval`，可以由开发者设置，改参数为刷新频率（时间间隔），在`p.flushInterval(req, res)`方法中使用

```GOLANG
rw.WriteHeader(res.StatusCode)
err = p.copyResponse(rw/*Src*/, res.Body/*DST*/, p.flushInterval(res))
if err != nil {
    defer res.Body.Close()
    // Since we're streaming the response, if we run into an error all we can do
    // is abort the request. Issue 23643: ReverseProxy should use ErrAbortHandler
    // on read error while copying body.
    if !shouldPanicOnCopyError(req) {
        p.logf("suppressing panic for copyResponse error in test; copy error: %v", err)
        return
    }
    panic(http.ErrAbortHandler)
}
```


16、关闭body，处理Trailer信息<br>
等数据全部拷贝完成后需要关闭`res.Body`，最后处理下trailer信息（将内容加到上游的头部的 Trailer 字段中）

Trailer的信息放在响应 body 之后，是独立存放响应信息中的，即 `res.Trailer`

注意这里：`res.Body.Close()`，在[Golang 标准库：net/http 应用（二）](https://pandaychen.github.io/2021/10/01/GOLANG-NETHTTP-PRACTICE/)文中说过，如果不关闭`res.Body`的话，后端的连接是不会进入http连接池复用的
```GOLANG
res.Body.Close()	//正确关闭res.Body

if len(res.Trailer) > 0 {
	// Force chunking if we saw a response trailer.
	// This prevents net/http from calculating the length for short
	// bodies and adding a Content-Length.
	if fl, ok := rw.(http.Flusher); ok {
		fl.Flush() // 刷新到上游的数据中心
	}
}

if len(res.Trailer) == announcedTrailers {
	copyHeader(rw.Header(), res.Trailer)
	return
}
// 读取Trailer中的头部信息，并将其设置到上游
for k, vv := range res.Trailer {
	k = http.TrailerPrefix + k
	for _, v := range vv {
		rw.Header().Add(k, v)
	}
}
```

至此，`ServeHttp`的核心处理流程（一次请求转发）就结束了

#### handleUpgradeResponse 协议升级

```golang
func (p *ReverseProxy) handleUpgradeResponse(rw http.ResponseWriter, req *http.Request, res *http.Response) {
	reqUpType := upgradeType(req.Header)
	resUpType := upgradeType(res.Header)
	if reqUpType != resUpType {
		p.getErrorHandler()(rw, req, fmt.Errorf("backend tried to switch protocol %q when %q was requested", resUpType, reqUpType))
		return
	}

	hj, ok := rw.(http.Hijacker)
	if !ok {
		p.getErrorHandler()(rw, req, fmt.Errorf("can't switch protocols using non-Hijacker ResponseWriter type %T", rw))
		return
	}
	backConn, ok := res.Body.(io.ReadWriteCloser)
	if !ok {
		p.getErrorHandler()(rw, req, fmt.Errorf("internal error: 101 switching protocols response with non-writable body"))
		return
	}

	backConnCloseCh := make(chan bool)
	go func() {
		// Ensure that the cancelation of a request closes the backend.
		// See issue https://golang.org/issue/35559.
		select {
		case <-req.Context().Done():
		case <-backConnCloseCh:
		}
		backConn.Close()
	}()

	defer close(backConnCloseCh)

	conn, brw, err := hj.Hijack()
	if err != nil {
		p.getErrorHandler()(rw, req, fmt.Errorf("Hijack failed on protocol switch: %v", err))
		return
	}
	defer conn.Close()

	copyHeader(rw.Header(), res.Header)

	res.Header = rw.Header()
	res.Body = nil // so res.Write only writes the headers; we have res.Body in backConn above
	if err := res.Write(brw); err != nil {
		p.getErrorHandler()(rw, req, fmt.Errorf("response write: %v", err))
		return
	}
	if err := brw.Flush(); err != nil {
		p.getErrorHandler()(rw, req, fmt.Errorf("response flush: %v", err))
		return
	}
	errc := make(chan error, 1)
	spc := switchProtocolCopier{user: conn, backend: backConn}
	go spc.copyToBackend(errc)
	go spc.copyFromBackend(errc)
	<-errc
	return
}
```

## 0x03 ReverseProxy 之坑

当然，在测试过程中也遇到了几个问题，排查下来都和 `httputil.ReverseProxy` 有关。遇到的问题主要有如下两个：

- `http.ErrAbortHandler` 类似的 `Panic` 异常
- `Context canceled` 的错误

问题的场景是有些 `js` 文件或者页面加载时间较长，超时或者（未加载完全时）用户点击了其中某个按钮的场景出现概率极高，从抓包分析看是在未加载完时，点击了其中的一个按钮，浏览器会取消这些未完成的请求去载入新页面，此时（浏览器）客户端会主动断开连接，从而导致了 `httputil.ReverseProxy` 在处理上的一些异常。

#### PANIC 异常（http.ErrAbortHandler）

从上面的源码来看，整个 `ServeHttp` 的实现中只有一处代码抛出了异常，即当客户端请求主动取消发生在 <代理获取到响应信息后将 response 复制给网关的 `rw http.ResponseWriter` 对象> 阶段时，就会出现 `http.ErrAbortHandler` 的 `Panic` 异常。相关源码如下：

```golang
//copyResponse 的实现
func (p *ReverseProxy) copyResponse(dst io.Writer, src io.Reader, flushInterval time.Duration) error {
	if flushInterval != 0 {
		if wf, ok := dst.(writeFlusher); ok {
			mlw := &maxLatencyWriter{
				dst:     wf,
				latency: flushInterval,
			}
			defer mlw.stop()

			// set up initial timer so headers get flushed even if body writes are delayed
			mlw.flushPending = true
			mlw.t = time.AfterFunc(flushInterval, mlw.delayedFlush)

			dst = mlw
		}
	}

	var buf []byte
	//如果没有定义内存池的实现，那就直接使用copyBuffer
	if p.BufferPool != nil {
		//指定了内存池实现
		buf = p.BufferPool.Get()
		defer p.BufferPool.Put(buf)
	}
	// 复制 response 到 dst
	//从scr中读取数据到buf，再把buf数据复制到dst中，此过程消耗性能
	//因为golang中 reader 和 writer 无法直接转移数据，只能借助buf作为中间转移数据
	_, err := p.copyBuffer(dst, src, buf)
	return err
}


// shouldPanicOnCopyError reports whether the reverse proxy should
// panic with http.ErrAbortHandler. This is the right thing to do by
// default, but Go 1.10 and earlier did not, so existing unit tests
// weren't expecting panics. Only panic in our own tests, or when
// running under the HTTP server.
//shouldPanicOnCopyError 实现
func shouldPanicOnCopyError(req *http.Request) bool {
	if inOurTests {
		// Our tests know to handle this panic.
		return true
	}
	// 服务器模式，返回 true，适用于本文出现的场景
	if req.Context().Value(http.ServerContextKey) != nil {
		// We seem to be running under an HTTP server, so
		// it'll recover the panic.
		return true
	}
	// Otherwise act like Go 1.10 and earlier to not break
	// existing tests.
	return false
}

// copyBuffer returns any write errors or non-EOF read errors, and the amount
// of bytes written.
//dst 为 gin 网关侧的数据写入接口，src 为后端 CGI 服务的 HTTP 响应
func (p *ReverseProxy) copyBuffer(dst io.Writer, src io.Reader, buf []byte) (int64, error) {
	if len(buf) == 0 {
		buf = make([]byte, 32*1024)
	}
	var written int64
	for {
		// 分块读，节省内存，bf 长度默认为 32k，一般需要读取多次才能把数据读完
		nr, rerr := src.Read(buf)
		if rerr != nil && rerr != io.EOF && rerr != context.Canceled {
			p.logf("httputil: ReverseProxy read error during body copy: %v", rerr)
		}
		if nr > 0 {
			// 将读取的数据写入网关侧的 http.ResponseWriter
			nw, werr := dst.Write(buf[:nr])
			if nw > 0 {
				written += int64(nw)
			}
			 // 写入过程中出现问题
			if werr != nil {
				return written, werr
			}
			if nr != nw {
				return written, io.ErrShortWrite
			}
		}
		// 读取过程中出现问题
		if rerr != nil {
			if rerr == io.EOF {
				rerr = nil
			}
			return written, rerr
		}
	}
}

func (p *ReverseProxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	//......
	// 周期性刷新内容到（给客户端的） Response 中，周期性刷新是指刷新频率（时间间隔）
	// 在 flushInterval(req, res) 方法中，控制时间间隔是通过读取 ReverseProxy 结构体中 FlushInterval 字段（可由开发者定义）
	err = p.copyResponse(rw, res.Body, p.flushInterval(res))
	if err != nil {
		defer res.Body.Close()
		// Since we're streaming the response, if we run into an error all we can do
		// is abort the request. Issue 23643: ReverseProxy should use ErrAbortHandler
		// on read error while copying body.
		// 是否应该抛出 panic
		if !shouldPanicOnCopyError(req) {
			p.logf("suppressing panic for copyResponse error in test; copy error: %v", err)
			return
		}
		panic(http.ErrAbortHandler)
	}
	//.....
}
```

从上面分析可知，该 `Panic` 异常，应该是客户端主动断开连接导致，当连接断开时，然后 `httputil.ReverseProxy` 正处于拿到响应信息后，将响应信息复制给网关的 `rw http.ResponseWriter` 对象过程中（可能数据较大，未复制完成），出现了问题导致。而且由于 `shouldPanicOnCopyError` 返回为 `true`，导致了最终 `ServeHTTP` 方法执行了 `panic(http.ErrAbortHandler)`。

#### Context canceled 错误

同样的，引用 `ServeHttp` 的代码：

```golang
// 实现了 ServeHTTP 接口
//rw：响应（客户端）的数据
//req：来自客户端的请求
func (p *ReverseProxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	//......
	/*
	 从原始请求 req 中取出上下文信息，通过类型断言 rw.(http.CloseNotifier)，来判断连接 (请求) 是否终止。
如果终止了则直接放弃本次请求，即调用 cancel()，取消此上下文，当然对应的资源也就释放了。
其中 http.CloseNotifier 是一个接口，只有一个方法 CloseNotify() <-chan bool，作用是检测连接是否断开
	*/
	ctx := req.Context()
	if cn, ok := rw.(http.CloseNotifier); ok {
		// 客户端连接终止
		var cancel context.CancelFunc
		// 注意 ctx 是 req 的 Context()，这里使用 WithCancel 强行在 req 的 context 中加入取消通知功能
		ctx, cancel = context.WithCancel(ctx)
		defer cancel()
		notifyChan := cn.CloseNotify()
		// 启动 goroutine 来监听 notifyChan 的取消事件，若触发则调用 cancel() 取消所有的子 context
		go func() {
			select {
			// 监听取消事件，收到请求取消通知，则调 context 的取消函数
			case <-notifyChan:
				cancel()
			case <-ctx.Done():
			}
		}()
	}

	// 拷贝原始请求 req 的上下文信息，并赋值给对外请求的 request（outreq）
	// 从 req.Clone 的实现来看：https://golang.org/src/net/http/request.go?s=13662:13715#L371
	//outreq 的 ctx 就是 ctx
	outreq := req.Clone(ctx)
	//......

	// 执行真正的代理功能：向后端发送请求数据，transport.RoundTrip() 的作用就是执行一个 HTTP 事务，根据请求返回响应
	// 注意：发起 HTTP 调用，当响应还未完成时，调用 cancel() 则会取消调用，并返回 Context cancel 的错误!!!
	res, err := transport.RoundTrip(outreq)
	if err != nil {
		p.getErrorHandler()(rw, outreq, err)
		return
	}

	//......
}

// Clone returns a deep copy of r with its context changed to ctx.
// The provided ctx must be non-nil.
//
// For an outgoing client request, the context controls the entire
// lifetime of a request and its response: obtaining a connection,
// sending the request, and reading the response headers and body.
func (r *Request) Clone(ctx context.Context) *Request {
	if ctx == nil {
		panic("nil context")
	}
	r2 := new(Request)
	*r2 = *r
	r2.ctx = ctx	//ctx 赋值
	r2.URL = cloneURL(r.URL)
	if r.Header != nil {
		r2.Header = r.Header.Clone()
	}
	if r.Trailer != nil {
		r2.Trailer = r.Trailer.Clone()
	}
	if s := r.TransferEncoding; s != nil {
		s2 := make([]string, len(s))
		copy(s2, s)
		r2.TransferEncoding = s2
	}
	r2.Form = cloneURLValues(r.Form)
	r2.PostForm = cloneURLValues(r.PostForm)
	r2.MultipartForm = cloneMultipartForm(r.MultipartForm)
	return r2
}
```

从代码实现不难看出，`Context cancel` 产生的原因主要就是 `httputil.ReverseProxy` 监听了前置 `gin` 的请求对象，即 `req *http.Request` （其中包含的 `Context` 对象）的取消通知，当客户端主动取消连接时，`gin` 会发送通知，`httputil.ReverseProxy` 收到后就会调用 `cancel()` 方法。
此时若刚好处于 <代理发起 HTTP 调用，响应还未完成> 过程时，`http.transport.RoundTrip` 就会抛出 `Context Canceled` 的错误，同时中断该连接。

#### 问题解决

解决的方法也很简单，修改原始代码，把抛出异常的部分去掉（或者在上层忽略掉这种错误）。看起来，使用 `io.Copy` 实现代理功能时，当客户端主动断开连接时，需要格外小心处理这种错误。

##	0x04	使用sync.Pool提升reverseproxy性能
通过现网压测，了解到`http.ReverseProxy.copyResponse`方法中，如果采用默认的配置，则会调用`copyBuffer`方法完成数据转发，会动态申请数组（`32K`），导致gc压力变大

`http.ReverseProxy`提供了内存池接口，可实现内存池，避免不断动态生成内存

```GOLANG
// copyBuffer returns any write errors or non-EOF read errors, and the amount
// of bytes written.
func (p *ReverseProxy) copyBuffer(dst io.Writer, src io.Reader, buf []byte) (int64, error) {
    //如果buf为空，此时动态申请 32K 的空间，此操作对gc影响较大
	if len(buf) == 0 {
		buf = make([]byte, 32*1024)
	}
	var written int64
	for {
        //从代理请求返回的响应body中读取数据
		//并把数据写入到前端响应中
		nr, rerr := src.Read(buf)
		if rerr != nil && rerr != io.EOF && rerr != context.Canceled {
			p.logf("httputil: ReverseProxy read error during body copy: %v", rerr)
		}
		if nr > 0 {
			nw, werr := dst.Write(buf[:nr])
			if nw > 0 {
				written += int64(nw)
			}
			if werr != nil {
				return written, werr
			}
			if nr != nw {
				return written, io.ErrShortWrite
			}
		}
		if rerr != nil {
			if rerr == io.EOF {
				rerr = nil
			}
			return written, rerr
		}
	}
}
```

下面是一个`[]byte`内存池的示例，也可以采用多级内存池[实现](https://github.com/pandaychen/goes-wrapper/blob/master/pypool/buffer_pool.go)
```GOLANG
// []byte 内存池
type BytesBufferPool struct {
    defaultSize int
    pool sync.Pool
}

func NewBytesBufferPool(defaultSize int) *BytesBufferPool {
    p := &BytesBufferPool{
        defaultSize: defaultSize,
        pool: sync.Pool{
            New: func() interface{} {
                return make([]byte, defaultSize)
            },
        },
    }
    return p
}

func (p *BytesBufferPool) Get() []byte {
    return p.pool.Get().([]byte)
}

func (p *BytesBufferPool) Put(bs []byte) {
    p.pool.Put(bs)
}
```


##	0x05	其他一些细节

####	X-Forwarded-For设置
遵循HTTP协议的规范，正常的代理服务器需要设置转发信息（将用户真实IP原封不动的透传到后端真实服务器），以帮助后端服务器获取到真实的客户端IP地址（恶意的除外）；通常会基于HTTP Header来实现：`X-Real-IP`、`X-Forward-For`等，二者区别如下：

-	`X-Real-IP`：通常在离用户最近的代理点上设置，用于记录用户的真实IP，往后的反向代理节点不需要设置，否则将覆盖为上一个反向代理的IP
-	`X-Forward-For`：记录每个经过的节点IP，以`,`分隔，如请求链路是`client -> proxy1 -> proxy2 -> backend`，那么Header为：`X-Forward-For: clientip,proxy1,proxy2`

所以，在`ServeHTTP`中，在`outreq`的Header部分完成了`X-Forwarded-For`的设置
```GOLANG
func (p *ReverseProxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
//...
// 设置X-Forwarded-For，追加节点IP
	if clientIP, _, err := net.SplitHostPort(req.RemoteAddr); err == nil {
		// If we aren't the first proxy retain prior
		// X-Forwarded-For information as a comma+space
		// separated list and fold multiple headers into one.
		if prior, ok := outreq.Header["X-Forwarded-For"]; ok {
			clientIP = strings.Join(prior, ", ") + ", " + clientIP
		}
		outreq.Header.Set("X-Forwarded-For", clientIP)
	}
//...
}
```

## 0x06 总结
本文介绍了`http.ReverseProxy`的实现，已经在现网项目中遇到的一些问题及优化。很多APIGATEWAY的后端转发都是直接调用`http.ReverseProxy`来实现的

当然，可以自行实现反向代理的功能，不过流程需要满足下面这些（不需要Upgrade支持可以不care）

1.	拷贝上游请求的Header到下游请求
2.	修改请求（例如协议、参数、url等）
3.	判断是否需要升级协议（Upgrade）
4.	删除上游请求中的hop-by-hop Header，即不需要透传到下游的header
5.	设置`X-Forward-For Header`，追加当前节点IP
6.	使用HTTP连接池，向下游发起请求
7.	处理协议升级（httpcode `101`）
8.	删除不需要返回给上游的Hop-by-hop Header
9.	修改响应体内容（如有需要）
10.	拷贝下游响应头部到上游响应请求
11.	返回HTTP状态码
12.	定时刷新内容到response


## 0x07 参考

- [golang 反向代理 reverseproxy 源码分析](https://blog.51cto.com/pmghong/2506101)
- [gobetween： Modern & minimalistic load balancer for the Сloud era](https://github.com/yyyar/gobetween)
- [使用 Go 语言徒手撸一个简单的负载均衡器](https://www.infoq.cn/article/h1yh7631okqsfffuvgkt)
-  [Golang Reverse Proxy](https://www.integralist.co.uk/posts/golang-reverse-proxy/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
