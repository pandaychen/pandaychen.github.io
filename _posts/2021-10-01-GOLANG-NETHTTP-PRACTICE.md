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
本文汇总下项目中使用`net.http`包的一些经验总结；在生产环境中，golang 自带的 http client 需要配置合适的参数才能适应项目场景。

##	0x01	http连接池的使用
关于 `net.http` 的连接池，先给出一些结论：

-	长连接必须客户端与服务端都开启，任何一端不支持，连接池也就没效果了
-	http 客户端默认开启的是长连接（即 `DisableKeepAlives: false`）
-	每个 http client 的对象默认是公用一个全局的连接池
-	http client 的连接池是协程安全的，有 Mutex 保护，可以跨协程复用
-	默认 client 最大连接池数量 `100`，每个目标 host 最大连接池数 `2` 个
-	如何修改 http 默认值参数？推荐方法是新建一个 `http.Transport{}` 对象，设置相关的核心属性后再赋值给 `client.Transport`
-	Transport 的属性中 `MaxIdleConns`表示全局最大连接池数量，`MaxIdleConnsPerHost`表示每个目标 host 的最大连接池数量
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

`DefaultTransport`的初始化值，`MaxIdleConns`值为`100`，即`100`个最大连接池；而`MaxIdleConnsPerHost`为`2`，即每个 host 最多有 `2` 个长连接；如果并发超过`MaxIdleConnsPerHost`，那么新建的连接就走短连接方式，不会被http所保存以复用

![max-params]()

##	0x02	客户端的一些行为

##	0x03	case1：连接池失效场景
先看如下例子（带有些迷惑性），使用并发`100`个goroutine进行测试，测试发现`TIME_WAIT`会大幅上涨，并没有使用到连接池（这里服务端是支持长连接的），原因为何？
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
        defer rsp.Body.Close()
        fmt.Printf("http rsp code :%v", rsp.StatusCode)
        return nil
}
```

上面的例子使用到默认内置的HTTP连接对象`DefaultClient`，如下：
```GOLANG
// $GOROOT/src/net/http/client.go
// DefaultClient is the default Client and is used by Get, Head, and Post.
var DefaultClient = &Client{}
```

猜想是不是使用了默认`http.Transport`的问题？于是改用自定义的`http.Transport`重写上面的例子，启动相同的并发压测，发现和前一个例子结果一样，看起来原因不在`http.Transport`上，继续深挖原因：
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

（PS：未使用连接池机制，导致http client 短连接后关闭连接的时候产生了 `TIME_WAIT`，直接用在生产环境不符合项目的需求）

####	原因排查
这里从http调用触发，一步步进入调用过程，排查上面代码未能触发连接池机制的原因在哪里？前文有更详细的分析：[Golang 标准库：net/http 分析（一）](https://pandaychen.github.io/2020/10/01/GOLANG-NETHTTP-ANALYSIS/)

1、入口`rsp,err:=client.Do(req)`<br>

2、`client.Do`方法：调用`c.send()`方法发请求，参数`req`和`deadline`（超时相关），不像导致短连接原因<br>
```GOLANG
func (c *Client) do(req *Request) (retres *Response, reterr error) {
    //调用c.send()
    if resp, didTimeout, err = c.send(req, deadline); err != nil {
       //...
    }
}
```

3、`c.send()`方法：调用`send()`方法发送，参数增加了`c.transport()`，从实现看，是返回`http.Transport`结构<br>
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
   return DefaultTransport //默认的DefaultTransport
}
```

4、`send()`方法，最终还是调用`rt.RoundTrip(req)`发送请求<br>

```GOLANG
func send(ireq *Request, rt RoundTripper, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
   //...
   resp, err = rt.RoundTrip(req)
   //...
}
```

5、默认的包全局变量`http.DefaultTransport`定义如下<br>
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

基于`Transport`结构看，对连接池定义影响最大的参数是`MaxIdleConnsPerHost`、`MaxIdleConns`这几个参数（前文有详细说明）

在实践中，如果客户端都连接到相同的Server，那么可以把`MaxIdleConnsPerHost`调整为和全局连接池`MaxIdleConns`的值一样大

####  本质的问题
上面两个例子最根本的问题在于：
1. 采用默认配置的`http.DefaultTransport`，`MaxIdleConnsPerHost`并发值过小（默认为`2`），并发超过`2`时，新建连接会以短连接方式发送
2. 代码中未对返回的`rsp.Body`进行读取，就关闭了`defer rsp.Body.Close()`，这样也不会触发连接复用，直接走短连接了！！

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
                MaxIdleConnsPerHost:   100,   //提高连接池的上限
                //服务都连接同一个 host，那就把每个 hsot 的连接池设置的和全局连接池设置为一样大
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

####  未正确读取rsp.Body的原因
在连接对象读的goroutine `pconn.readLoop()` 中，有一个 `rsp.Body` 的 EOF 信号处理对象，保证 `resp.Body` 被读到 `EOF` 了，此连接才可能被复用，不过现网中可能遇到的情况较少（一般需要拿返回值）

```text
// bodyEOFSignal is used by the HTTP/1 transport when reading response
// bodies to make sure we see the end of a response body before
// proceeding and reading on the connection again.
```

## 0x04  共享Client


##  0x05	参考
-	[http.Client 的连接行为控制详解](https://tonybai.com/2021/04/02/go-http-client-connection-control/)
-	[通过实例理解 Go 标准库 http 包是如何处理 keep-alive 连接的](https://mp.weixin.qq.com/s/uH4lmsMs-hq5B3Ivq2jR5w)