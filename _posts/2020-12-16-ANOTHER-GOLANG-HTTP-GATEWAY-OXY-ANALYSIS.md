---
layout: post
title: HTTP(s) 代理中间件库：Oxy
subtitle: 分析基于 Golang 的 HTTP(s) 代理中间件库 Oxy：Part One
date: 2020-12-16
author: pandaychen
header-img: img/super-mario.jpg
catalog: true
category: false
tags:
  - 反向代理
  - 网关
  - Oxy
---

## 0x00 前言

[Oxy：Go middlewares for HTTP servers & proxies](https://github.com/vulcand/oxy) 是一个 HTTP(s) 代理开发（中间件）库，另一个负载均衡网关项目 [vulcand](https://github.com/vulcand/vulcand) 也基于此库来实现反向代理（Reverse Proxy Engine）功能。它的 [主要组件如下](https://github.com/vulcand/oxy/blob/master/README.md)，框架还是围绕着原生 `net/http` 库实现的：

- [Buffer](https://pkg.go.dev/github.com/vulcand/oxy/buffer) retries and buffers requests and responses
- [Stream](https://pkg.go.dev/github.com/vulcand/oxy/stream) passes-through requests, supports chunked encoding with configurable flush interval
- [Forward](https://pkg.go.dev/github.com/vulcand/oxy/forward) forwards requests to remote location and rewrites headers
- [Roundrobin](https://pkg.go.dev/github.com/vulcand/oxy/roundrobin) is a round-robin load balancer
- [Circuit Breaker](https://pkg.go.dev/github.com/vulcand/oxy/cbreaker) Hystrix-style circuit breaker
- [Connlimit](https://pkg.go.dev/github.com/vulcand/oxy/connlimit) Simultaneous connections limiter
- [Ratelimit](https://pkg.go.dev/github.com/vulcand/oxy/ratelimit) Rate limiter (based on tokenbucket algo)
- [Trace](https://pkg.go.dev/github.com/vulcand/oxy/trace) Structured request and response logger

同类型的其他 package 还有：

- [elazarl/goproxy](https://github.com/elazarl/goproxy)
- [net/http/httputil]()

无论是 `gin` Web 框架，还是 HTTP(s) 代理，都离不开对 `http.Handler` 的封装、对 HTTP 路由、Header、参数及其他 HTTP 关键参数的重写及改造，而标准库 `net/http` 只是提供了最基础的 CS 架构下的 Request-Response 模型：

> Every handler is http.Handler, so writing and plugging in a middleware is easy

#### 使用介绍

从官方提供的使用例子，先了解下 Oxy 的基础功能：

1、Simple [reverse proxy](https://github.com/vulcand/oxy/blob/master/forward/fwd.go)，简单的反向代理，将 `8080` 的 HTTP 请求发送给 `http://localhost:63450` <br>

```golang
import (
  "net/http"
  "github.com/vulcand/oxy/forward"
  "github.com/vulcand/oxy/testutils"
)

// Forwards incoming requests to whatever location URL points to, adds proper forwarding headers
fwd, _ := forward.New()

redirect := http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
    // let us forward this request to another server
    req.URL = testutils.ParseURI("http://localhost:63450")
    // 将上层的 Request 和 ResponseWriter 传递给 fwd
    fwd.ServeHTTP(w, req)
})

func main(){
        // ...
        // that's it! our reverse proxy is ready!
        s := &http.Server{
                Addr:           ":8080",
                Handler:        redirect,
        }
        s.ListenAndServe()
}
```

2、给反向代理的多后端增加负载均衡策略 <br>

```golang
import (
        "github.com/vulcand/oxy/forward"
        "github.com/vulcand/oxy/roundrobin"
        "github.com/vulcand/oxy/testutils"
        "net/http"
)

func main() {
        // Forwards incoming requests to whatever location URL points to, adds proper forwarding headers
        fwd, _ := forward.New()
        lb, _ := roundrobin.New(fwd)

        url1 := testutils.ParseURI("http://localhost:63450")
        url2 := testutils.ParseURI("http://localhost:63451")

        lb.UpsertServer(url1)
        lb.UpsertServer(url2)

        s := &http.Server{
                Addr:    ":8080",
                Handler: lb,
        }
        s.ListenAndServe()
}
```

3、额外处理重试及错误 <br>

```golang
import (
  "net/http"
  "github.com/vulcand/oxy/forward"
  "github.com/vulcand/oxy/buffer"
  "github.com/vulcand/oxy/roundrobin"
)

// Forwards incoming requests to whatever location URL points to, adds proper forwarding headers

fwd, _ := forward.New()
lb, _ := roundrobin.New(fwd)

// buffer will read the request body and will replay the request again in case if forward returned status
// corresponding to nework error (e.g. Gateway Timeout)
buffer, _ := buffer.New(lb, buffer.Retry(`IsNetworkError() && Attempts() < 2`))

lb.UpsertServer(url1)
lb.UpsertServer(url2)

// that's it! our reverse proxy is ready!
s := &http.Server{
	Addr:           ":8080",
	Handler:        buffer,
}
s.ListenAndServe()
```

## 0x01 Forward 子组件：反向代理模块分析
本文代码基于[v1.4.1](https://github.com/vulcand/oxy/releases/tag/v1.4.1)，Forward 转发组件是HTTP转发功能的基础组件，其他模块如lb、buffer等都基于此基础进行构建。用 Forward 组件实现反向代理的一种调用方式如下：

```golang
FwdObj *forward.Forwarder

func API_handler(w http.ResponseWriter, req *http.Request) {
        req.URL = testutils.ParseURI("xxxx.backend.com")
        // 创建forward.RoundTripper及设置属性
        r := forward.RoundTripper(
                &http.Transport{
                        TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
                },
        )
        //使用forward包设置HTTP转发器
        FwdObj, _ = forward.New(forward.PassHostHeader(true), forward.Logger(x_logger), r)
        //执行反向代理的功能
        FwdObj.ServeHTTP(w, req)
}

```

[Forward 组件](https://github.com/vulcand/oxy/tree/master/forward)封装了 HTTP 和 websocket 两种转发的实现，核心结构是[httpForwarder](https://github.com/vulcand/oxy/blob/6b5fc980479afc1e3c3ecf1fe4319fefc16e523f/forward/fwd.go)，对http请求进行处理后，最终还是调用[`httputil.ReverseProxy`](https://github.com/vulcand/oxy/blob/master/forward/fwd.go)进行反向代理的功能：

```golang
revproxy := httputil.ReverseProxy{
        Director: func(req *http.Request) {
                f.modifyRequest(req, inReq.URL)
        },
        Transport:      f.roundTripper,
        FlushInterval:  f.flushInterval,
        ModifyResponse: f.modifyResponse,
        BufferPool:     f.bufferPool,
        ErrorHandler:   ctx.errHandler.ServeHTTP,
}
```

####    核心结构
oxy在最终调用`http.ReverseProxy`构建转发器之前做了大量封装，主要体现在`Forwarder`结构中

```GOLANG
// Forwarder wraps two traffic forwarding implementations: HTTP and websockets.
// It decides based on the specified request which implementation to use.
type Forwarder struct {
	*httpForwarder
	*handlerContext
	stateListener URLForwardingStateListener          //
	stream        bool
}
```

```golang
// httpForwarder is a handler that can reverse proxy
// HTTP traffic
type httpForwarder struct {
	roundTripper   http.RoundTripper        //用于转发请求的RoundTripper（必需包含），默认http.DefaultTransport
	rewriter       ReqRewriter              //用于改写转发请求
	passHost       bool                     //
	flushInterval  time.Duration
	modifyResponse func(*http.Response) error       //用于改写响应请求的回调

	tlsClientConfig *tls.Config

	log OxyLogger

	bufferPool                    httputil.BufferPool
	websocketConnectionClosedHook func(req *http.Request, conn net.Conn)
}

// handlerContext defines a handler context for error reporting and logging.
type handlerContext struct {
	errHandler utils.ErrorHandler
}

// UrlForwardingStateListener alias on URLForwardingStateListener.
// Deprecated: use URLForwardingStateListener instead.
type UrlForwardingStateListener = URLForwardingStateListener

// URLForwardingStateListener URL forwarding state listener.
type URLForwardingStateListener func(*url.URL, int)
```

####    `ServeHTTP`实现
Forwarder的[ServeHTTP](https://github.com/vulcand/oxy/blob/master/forward/fwd.go#L247)方法实现如下：

```GOLANG
// ServeHTTP decides which forwarder to use based on the specified
// request and delegates to the proper implementation.
func (f *Forwarder) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	if f.log.GetLevel() >= log.DebugLevel {
		logEntry := f.log.WithField("Request", utils.DumpHTTPRequest(req))
		logEntry.Debug("vulcand/oxy/forward: begin ServeHttp on request")
		defer logEntry.Debug("vulcand/oxy/forward: completed ServeHttp on request")
	}

	if f.stateListener != nil {
		f.stateListener(req.URL, StateConnected)
		defer f.stateListener(req.URL, StateDisconnected)
	}
        //判断是否支持websocket
	if IsWebsocketRequest(req) {
		f.httpForwarder.serveWebSocket(w, req, f.handlerContext)
	} else {
		f.httpForwarder.serveHTTP(w, req, f.handlerContext)
	}
}
```

`IsWebsocketRequest`根据http Header中的关键字判断当前是不是websocket请求：
```golang
// IsWebsocketRequest determines if the specified HTTP request is a websocket handshake request.
func IsWebsocketRequest(req *http.Request) bool {
	containsHeader := func(name, value string) bool {
		items := strings.Split(req.Header.Get(name), ",")
		for _, item := range items {
			if value == strings.ToLower(strings.TrimSpace(item)) {
				return true
			}
		}
		return false
	}
	return containsHeader(Connection, "upgrade") && containsHeader(Upgrade, "websocket")
}
```

`httpForwarder.serveHTTP`方法：
```golang
// serveHTTP forwards HTTP traffic using the configured transport.
func (f *httpForwarder) serveHTTP(w http.ResponseWriter, inReq *http.Request, ctx *handlerContext) {
	if f.log.GetLevel() >= log.DebugLevel {
		logEntry := f.log.WithField("Request", utils.DumpHTTPRequest(inReq))
		logEntry.Debug("vulcand/oxy/forward/http: begin ServeHttp on request")
		defer logEntry.Debug("vulcand/oxy/forward/http: completed ServeHttp on request")
	}

	start := clock.Now().UTC()

        //先复制一份http.Request
	outReq := new(http.Request)
	*outReq = *inReq // includes shallow copies of maps, but we handle this in Director

	revproxy := httputil.ReverseProxy{
		Director: func(req *http.Request) {
			f.modifyRequest(req, inReq.URL)
		},
		Transport:      f.roundTripper,
		FlushInterval:  f.flushInterval,
		ModifyResponse: f.modifyResponse,
		BufferPool:     f.bufferPool,
		ErrorHandler:   ctx.errHandler.ServeHTTP,
	}

	if f.log.GetLevel() >= log.DebugLevel {
		pw := utils.NewProxyWriter(w)
		revproxy.ServeHTTP(pw, outReq)

		if inReq.TLS != nil {
			f.log.Debugf("vulcand/oxy/forward/http: Round trip: %v, code: %v, Length: %v, duration: %v tls:version: %x, tls:resume:%t, tls:csuite:%x, tls:server:%v",
				inReq.URL, pw.StatusCode(), pw.GetLength(), clock.Now().UTC().Sub(start),
				inReq.TLS.Version,
				inReq.TLS.DidResume,
				inReq.TLS.CipherSuite,
				inReq.TLS.ServerName)
		} else {
			f.log.Debugf("vulcand/oxy/forward/http: Round trip: %v, code: %v, Length: %v, duration: %v",
				inReq.URL, pw.StatusCode(), pw.GetLength(), clock.Now().UTC().Sub(start))
		}
	} else {
		revproxy.ServeHTTP(w, outReq)
	}

	for key := range w.Header() {
		if strings.HasPrefix(key, http.TrailerPrefix) {
			if fl, ok := w.(http.Flusher); ok {
				fl.Flush()
			}
			break
		}
	}
}
```

####    modifyRequest方法
`httputil.ReverseProxy`中的`Director`，采用`modifyRequest`方法：
```GOLANG
// Modify the request to handle the target URL.
func (f *httpForwarder) modifyRequest(outReq *http.Request, target *url.URL) {
	outReq.URL = utils.CopyURL(outReq.URL)
	outReq.URL.Scheme = target.Scheme
	outReq.URL.Host = target.Host

	u := f.getURLFromRequest(outReq)

	outReq.URL.Path = u.Path
	outReq.URL.RawPath = u.RawPath
	outReq.URL.RawQuery = u.RawQuery
	outReq.RequestURI = "" // Outgoing request should not have RequestURI

	outReq.Proto = "HTTP/1.1"
	outReq.ProtoMajor = 1
	outReq.ProtoMinor = 1

	if f.rewriter != nil {
		f.rewriter.Rewrite(outReq)
	}

	// Do not pass client Host header unless optsetter PassHostHeader is set.
	if !f.passHost {
		outReq.Host = target.Host
	}
}
```


## 0x02 参考

- [Writing a Reverse Proxy in just one line with Go](https://hackernoon.com/writing-a-reverse-proxy-in-just-one-line-with-go-c1edfa78c84b)
- [Vulcand Gateway Doc](https://vulcand.github.io/quickstart.html)
- [Vulcand - Programmatic load balancer backed by Etcd](https://github.com/vulcand/vulcand)
- [goproxy：An HTTP proxy library for Go](https://github.com/elazarl/goproxy)
