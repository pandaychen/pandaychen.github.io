---
layout:     post
title:      Golang 标准库：net/http 应用（二）
subtitle:   net/http 客户端在项目中的应用经验小结
date:       2021-10-01
author:     pandaychen
header-img: img/super-mario.jpg
catalog: true
category:   false
tags:
    - HTTP
    - Golang
---


##  0x00    前言
本文汇总下项目中使用 `net.http` 包的一些经验总结；在生产环境中，golang 自带的 http client 需要配置合适的参数才能适应项目场景。

##	0x01	http 连接池的使用
关于 `net.http` 的连接池，先给出一些结论：

-	长连接必须客户端与服务端都开启，任何一端不支持，连接池也就没效果了
-	http 客户端默认开启的是长连接（即 `DisableKeepAlives: false`）
-	每个 http client 的对象默认是公用一个全局的连接池
-	http client 的连接池是协程安全的，有 Mutex 保护，可以跨协程复用
-	默认 client 最大连接池数量 `100`，每个目标 host 最大连接池数 `2` 个
-	如何修改 http 默认值参数？推荐方法是新建一个 `http.Transport{}` 对象，设置相关的核心属性后再赋值给 `client.Transport`
-	Transport 的属性中 `MaxIdleConns` 表示全局最大连接池数量，`MaxIdleConnsPerHost` 表示每个目标 host 的最大连接池数量
-	`http.Transport{}` 这个对象持有连接池，可以同时给多个 client 复用


####	MaxIdleConns 与 MaxIdleConnsPerHost 的意义

```GOLANG
type Transport struct {
   idleMu     sync.Mutex
   wantIdle   bool                                // user has requested to close all idle conns
   idleConn   map[connectMethodKey][]*persistConn // most recently used at end 连接池
   //...
   MaxIdleConnsPerHost int  // 每个 host 的连接池中的最大空闲连接数，默认 2 个
   MaxIdleConns int          // 最大空闲连接数，空闲连接大于这个数，请求结束后连接就关闭，DefaultTransport 是 100 个
}
```

`DefaultTransport` 的初始化值，`MaxIdleConns` 值为 `100`，即 `100` 个最大连接池；而 `MaxIdleConnsPerHost` 为 `2`，即每个 host 最多有 `2` 个长连接；如果并发超过 `MaxIdleConnsPerHost`，那么新建的连接就走短连接方式，不会被 http 所保存以复用

![max-params]()


####    现网的问题
现网遇到过如下场景，使用默认的 `http.Transport` 参数，出现 `TIME_WAIT` 暴涨的场景（单机到腾讯云 CLB-VIP 的连接暴涨）；经过排查之后，根本原因来自于 `MaxIdleConnsPerHost` 这个参数。`MaxIdleConnsPerHost` 会导致 `http.Client` 底层只会保留 `2` 个到 CLB-VIP 的空闲连接. 当请求增加了，连接数涨起来之后，超过 `MaxIdleConnsPerHost` 数量多余的, 就会被主动 `Close` 掉。因为服务请求量本身较大，所以会导致前面 `TIME_WAIT` 的问题，经过调整之后的参数如下：

```golang
http.Transport{
        MaxIdleConns:        200,
        IdleConnTimeout:     30 * time.Second,
        MaxConnsPerHost:     100,
        MaxIdleConnsPerHost: 100,            // 重要！
        DialContext: (&net.Dialer{
                Timeout:   30 * time.Second,
                KeepAlive: 30 * time.Second,
        })
}
```

为何 `http.Client` 保持 `2` 个空闲连接呢，查了下是 http 协议 RFC 的建议做法；但是在现网项目中一定要结合自己的应用场景做合理的设置。另外，还有一个细节问题是：`MaxIdleConns` 低于 `MaxIdleConnsPerHost` 会影响到 `MaxIdleConnsPerHost` 的设置，见下面的注释：

```
// MaxIdleConns controls the maximum number of idle (keep-alive)
// connections across all hosts. Zero means no limit
```

##	0x02	客户端的一些行为

##	0x03	连接池失效场景（case1：未读取 resp.body）
先看如下例子（带有些迷惑性），使用并发 `100` 个 goroutine 进行测试，测试发现 `TIME_WAIT` 会大幅上涨，并没有使用到连接池（这里服务端是支持长连接的），原因为何？
```GOLANG
func DoHttpGet() error {
        client := &http.Client{}
        req, err := http.NewRequest("get", "http://x.x.x.x/test", nil)
        if err != nil {
                return err
        }
        rsp, err := client.Do(req)
        if err != nil {
                return err
        }
        // 注意这里并没有读取 resp.body
        defer rsp.Body.Close()
        fmt.Printf("http rsp code :%v", rsp.StatusCode)
        return nil
}
```

上面的例子使用到默认内置的 HTTP 连接对象 `DefaultClient`，如下：
```GOLANG
// $GOROOT/src/net/http/client.go
// DefaultClient is the default Client and is used by Get, Head, and Post.
var DefaultClient = &Client{}
```

猜想是不是使用了默认 `http.Transport` 的问题？于是改用自定义的 `http.Transport` 重写上面的例子，启动相同的并发压测，发现和前一个例子结果一样，看起来原因不在 `http.Transport` 上，继续深挖原因：
```golang
func DoHttpGet2() error {
        transpot := &http.Transport{
                DialContext: (&net.Dialer{
                        Timeout:   30 * time.Second,
                        KeepAlive: 300 * time.Second,
                        DualStack: true, //
                }).DialContext,
                MaxIdleConns:          100,
                IdleConnTimeout:       90 * time.Second,
                TLSHandshakeTimeout:   10 * time.Second,
                ExpectContinueTimeout: 1 * time.Second,
                MaxIdleConnsPerHost:   10,
        }

        client := &http.Client{
                Transport: transpot, // 使用自定义的 Transport
        }
        req, err := http.NewRequest("get", "http://x.x.x.x/test", nil)
        if err != nil {
                return err
        }
        rsp, err := client.Do(req)
        if err != nil {
                return err
        }
        defer rsp.Body.Close()
        fmt.Printf("http rsp code :%v", rsp.StatusCode)
        return nil
}
```

（PS：未使用连接池机制，导致 http client 短连接后关闭连接的时候产生了 `TIME_WAIT`，直接用在生产环境不符合项目的需求）

####	原因排查
这里从 http 调用触发，一步步进入调用过程，排查上面代码未能触发连接池机制的原因在哪里？前文有更详细的分析：[Golang 标准库：net/http 分析（一）](https://pandaychen.github.io/2020/10/01/GOLANG-NETHTTP-ANALYSIS/)

1、入口 `rsp,err:=client.Do(req)`<br>

2、`client.Do` 方法：调用 `c.send()` 方法发请求，参数 `req` 和 `deadline`（超时相关），不像导致短连接原因 <br>
```GOLANG
func (c *Client) do(req *Request) (retres *Response, reterr error) {
    // 调用 c.send()
    if resp, didTimeout, err = c.send(req, deadline); err != nil {
       //...
    }
}
```

3、`c.send()` 方法：调用 `send()` 方法发送，参数增加了 `c.transport()`，从实现看，是返回 `http.Transport` 结构 <br>
```GOLANG
func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
    //...
    resp, didTimeout, err = send(req, c.transport(), deadline)
    //...
}

func (c *Client) transport() RoundTripper {
   if c.Transport != nil {
      return c.Transport// 返回 client 中的 Transport
   }
   return DefaultTransport // 默认的 DefaultTransport
}
```

4、`send()` 方法，最终还是调用 `rt.RoundTrip(req)` 发送请求 <br>

```GOLANG
func send(ireq *Request, rt RoundTripper, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
   //...
   resp, err = rt.RoundTrip(req)
   //...
}
```

5、默认的包全局变量 `http.DefaultTransport` 定义如下 <br>
```golang
var DefaultTransport RoundTripper = &Transport{
   Proxy: ProxyFromEnvironment,
   DialContext: (&net.Dialer{
      Timeout:   30 * time.Second,
      KeepAlive: 30 * time.Second,
      DualStack: true,
   }).DialContext,
   MaxIdleConns:          100,
   IdleConnTimeout:       90 * time.Second,
   TLSHandshakeTimeout:   10 * time.Second,
   ExpectContinueTimeout: 1 * time.Second,
}
```

基于 `Transport` 结构看，对连接池定义影响最大的参数是 `MaxIdleConnsPerHost`、`MaxIdleConns` 这几个参数（前文有详细说明）

在实践中，如果客户端都连接到相同的 Server，那么可以把 `MaxIdleConnsPerHost` 调整为和全局连接池 `MaxIdleConns` 的值一样大

####  本质的问题
上面两个例子最根本的问题在于：
1. 采用默认配置的 `http.DefaultTransport`，`MaxIdleConnsPerHost` 并发值过小（默认为 `2`），并发超过 `2` 时，新建连接会以短连接方式发送
2. 代码中未对返回的 `rsp.Body` 进行读取，就关闭了 `defer rsp.Body.Close()`，这样也不会触发连接复用，直接走短连接了！！

一个长连接的最终压测代码如下：
```GOLANG
func DoHttpGet3() error {
        transpot := &http.Transport{
                DialContext: (&net.Dialer{
                        Timeout:   30 * time.Second,
                        KeepAlive: 300 * time.Second,
                        DualStack: true, //
                }).DialContext,
                MaxIdleConns:          100,
                IdleConnTimeout:       90 * time.Second,
                TLSHandshakeTimeout:   10 * time.Second,
                ExpectContinueTimeout: 1 * time.Second,
                MaxIdleConnsPerHost:   100,   // 提高连接池的上限
                // 服务都连接同一个 host，那就把每个 hsot 的连接池设置的和全局连接池设置为一样大
        }

        client := &http.Client{
                Transport: transpot, // 使用自定义的 Transport
        }
        req, err := http.NewRequest("get", "http://x.x.x.x/test", nil)
        if err != nil {
                return err
        }
        rsp, err := client.Do(req)
        if err != nil {
                return err
        }
        defer rsp.Body.Close()
         _, err = ioutil.ReadAll(rsp.Body)// 很重要，否则连接不会被复用！！！
        fmt.Printf("http response code :%v", rsp.StatusCode)
        return nil
}
```

####  未正确读取 rsp.Body 的原因
在连接对象读的 goroutine `pconn.readLoop()` 中，有一个 `rsp.Body` 的 EOF 信号处理对象，保证 `resp.Body` 被读到 `EOF` 了，此连接才可能被复用，不过现网中可能遇到的情况较少（一般需要拿返回值）

```text
// bodyEOFSignal is used by the HTTP/1 transport when reading response
// bodies to make sure we see the end of a response body before
// proceeding and reading on the connection again.
```

####    解决方法
未正确读取 `resp.Body` 将会直接导致新请求会直接新建连接，不走连接池，规避方法如下：
1.      正确的读取 `resp.Body`
2.      如果不关心返回 body 可以直接丢弃，通过调用 `io.Copy(ioutil.Discard, resp.Body)`，其中 `Discard` 是一个 `io.Writer`，对它进行的任何 `Write` 调用都将无条件成功

##      0x04    连接池失效场景（case2：误用 http.Transport）
参考错误示例如下，为了设置代理 `Proxy`，为每个请求，都 `new` 了一个 `http.Transport`，这从根本上违反了 `http.Transport` 的设计初衷。`http.Client` 连接池的维度是 `http.Transport`，  `http.Transport` 维护了两个 map 用以实现连接池。所以使用连接池，`http.Client` 最好是设置为全局的（`http.Client` 是线程安全），在这个全局 `http.Client` 设置连接池的属性即可。

```golang
client := &http.Client{
     Timeout: time.Duration(10)*time.Second,
}

client.Transport = &http.Transport{
     Proxy: http.ProxyURL(proxyUrl),    // 设置代理 ip
}
```

##      0x05    连接池失效场景（case3：MaxIdleConnsPerHost 设置策略）
这个上面已经讨论过，合理设置 `MaxIdleConnsPerHost` 的值，不要使用系统的默认值或者不设置该值，同时也需要合理设置 `Timeout` 值。一旦高频的连接超过 `MaxIdleConnsPerHost` 的数目，超过的部分连接就会被释放。正确的设置是实例化 `http.Transport` 的时候，评估好 `MaxIdleConnsPerHost`

```golang
var DefaultTransport RoundTripper = &Transport{
  //....
  MaxIdleConnsPerHost: 1000,
  IdleConnTimeout:       90 * time.Second,
}
```

## 0x06  共享 Client
在 Go 中，需要复用 `http.Client`，通常代码中会定义一个全局的 `http.Client` 对象（并设置合适的连接属性），并在多次请求时进行使用，这个变量已经自带连接池了，所以通常我们无需为 `http.Client` 再实现连接池：

```golang
var client = &http.Client{}

func main() {
	req, _ := http.NewRequest("GET", "https://www.google.com", nil)
	res, _ := client.Do(req)
	fmt.Println(res.StatusCode)
}
```

##      0x07    使用net.http构建一个正向代理（http/https）

![http-connect-proxy](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/network/http-connect-proxy.png)

实现代码[在此](https://github.com/pandaychen/golang_in_action/blob/master/proxy/httpproxy/https_proxy.go)，摘一下核心逻辑：

####    HTTP代理

```GO
func handleHttp(w http.ResponseWriter, r *http.Request) {
	resp, err := http.DefaultTransport.RoundTrip(r)         //利用http.DefaultTransport.RoundTrip转发请求即可
	if err != nil {
		http.Error(w, err.Error(), http.StatusServiceUnavailable)
		return
	}
	defer resp.Body.Close()

	copyHeader(w.Header(), resp.Header)
	w.WriteHeader(resp.StatusCode)

        //把数据从 resp.Body（io.ReadCloser）copy 至 http.ResponseWriter
	io.Copy(w, resp.Body)
}
```

####    HTTPS代理


##  0x08	参考
-	[http.Client 的连接行为控制详解](https://tonybai.com/2021/04/02/go-http-client-connection-control/)
-	[通过实例理解 Go 标准库 http 包是如何处理 keep-alive 连接的](https://mp.weixin.qq.com/s/uH4lmsMs-hq5B3Ivq2jR5w)
-       [从 http.Transport 看连接池的设计](https://vearne.cc/archives/39913)
-       [Tuning the Go HTTP Client Settings for Load Testing](http://tleyden.github.io/blog/2016/11/21/tuning-the-go-http-client-library-for-load-testing/)