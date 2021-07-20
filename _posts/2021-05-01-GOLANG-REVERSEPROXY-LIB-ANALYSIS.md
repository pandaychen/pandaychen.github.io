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

在工作项目中，使用 `gin` 与 `httputil.ReverseProxy` 完成认证网关及反向代理的功能，反向代理的流程为：
1、 通过 `gin` 实现的 https 网关 接收浏览器 Web 发起的请求 <br>
2、 网关通过 `httputil.ReverseProxy` 发起一个带 API 签名的 HTTP 请求给后台 CGI 服务，实现代理功能 <br>
3、 网关接收到后台 CGI 的服务响应信息 <br>
4、 网关将响应回复给客户端浏览器 <br>

本文分析下 `httputil.ReverseProxy` 的实现细节。[httputil.ReverseProxy](https://golang.org/pkg/net/http/httputil/#ReverseProxy) 是官方提供的一个反向代理实现，可直接拿来用，在网关或代理类服务实现非常方便，这篇文章简单分析下其实现。
<br>
ReverseProxy 具有如下特点：

- 自定义修改响应包体
- 连接池
- 错误信息自定义处理
- 支持 websocket 服务
- 自定义负载均衡
- https 代理
- URL 重写

## 0x01 核心结构与方法

#### ReverseProxy 结构

`ReverseProxy` 包含两个重要的属性 `Director` 和 `ModifyResponse`，这两个属性都是函数类型。当接收到客户端请求时，`ServeHTTP` 函数首先调用 `Director` 函数对接受到的请求体进行修改，例如修改请求的目标地址、请求头等；然后使用修改后的请求体发起新的请求，接收到响应后，调用 `ModifyResponse` 函数对响应进行修改，最后将修改后的响应体拷贝并响应给客户端，这样就实现了反向代理的整个流程。

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

#### NewSingleHostReverseProxy 方法

`NewSingleHostReverseProxy` 方法返回一个新的 `ReverseProxy`，将 `URLs` 请求路由到传入参数 `target` 的指定的 `Scheme`, `Host` 以及 `Base path`，也是默认的 `director` 配置，在 NewSingleHostReverseProxy 中源码已经对传入的 URLs 进行解析并且完成了 Director 的修改

```golang
// NewSingleHostReverseProxy returns a new ReverseProxy that routes
// URLs to the scheme, host, and base path provided in target. If the
// target's path is"/base"and the incoming request was for"/dir",
// the target request will be for /base/dir.
// NewSingleHostReverseProxy does not rewrite the Host header.
// To rewrite Host headers, use ReverseProxy directly with a custom
// Director policy.
func NewSingleHostReverseProxy(target *url.URL) *ReverseProxy {
    targetQuery := target.RawQuery
    // 默认的 director 配置
    director := func(req *http.Request) {
      req.URL.Scheme = target.Scheme
      req.URL.Host = target.Host
      req.URL.Path = singleJoiningSlash(target.Path, req.URL.Path)
      if targetQuery == ""|| req.URL.RawQuery =="" {
        req.URL.RawQuery = targetQuery + req.URL.RawQuery
      } else {
        req.URL.RawQuery = targetQuery + "&" + req.URL.RawQuery
      }
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

## 0x02 ServeHTTP 方法

基于先前的经验，实例化的 reverseproxy 对象必须实现 `ServeHTTP` 方法才能使用 `Handler` 方式调用，本小节具体分析下 [ServeHTTP](https://golang.org/src/net/http/httputil/reverseproxy.go?s=6664:6739#L202) 方法：

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
	if p.BufferPool != nil {
		buf = p.BufferPool.Get()
		defer p.BufferPool.Put(buf)
	}
	// 复制 response 到 dst，即
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

## 0x04 总结

## 0x04 参考

- [golang 反向代理 reverseproxy 源码分析](https://blog.51cto.com/pmghong/2506101)
- [gobetween： Modern & minimalistic load balancer for the Сloud era](https://github.com/yyyar/gobetween)
- [使用 Go 语言徒手撸一个简单的负载均衡器](https://www.infoq.cn/article/h1yh7631okqsfffuvgkt)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
