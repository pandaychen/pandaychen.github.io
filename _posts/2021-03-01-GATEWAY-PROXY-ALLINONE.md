---
layout: post
title: 项目开发：网关（Gateway）与反向代理（ReverseProxy）的那些事
subtitle:
date: 2021-03-01
author: pandaychen
header-img:
catalog: true
category: false
tags:
  - Gateway
  - 网关
  - 反向代理
  - ReverseProxy
---

## 0x00 前言

本文是网关及代理开发过程中的一些总结。代理（常见正向代理及反向代理）可以视为网关的核心组件之一。常见的网关有：

- 安全网关（Secure Gateway）
- 开放认证网关（Open IDP Gateway）
- 微服务网关（MicroService Gateway）

对网关的特性主要关注下面几点：

1. 功能 && 部署架构
2. 高性能
3. 高可用
4. 插件式应用，如中间件等
5. 合理集成微服务的特性，如 Tracing/Metrics 观测 / 超时传递 / 熔断、限流机制 / 服务发现等等

## 0x01 一种网关的实现

下面是本人近期项目中实现的一个身份认证网关的基础流程：
![a-basic-idp-gateway-workflow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/a-basic-idp-gateway-workflow.png)

整个网关的架构大致如下图所示：
![api-gateway](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/bifrost-api-gateway.png)

## 0x02 实现的细节

反向代理的实现一般需要下面几步，项目使用 `http.ReverseProxy` 实现反向代理功能：

- 代理接收到客户端的请求，复制（或复用）原始请求对象
- 根据一些规则，修改新请求的请求指向（如域名 / URL / 请求头等），或者需要在想后端转发请求时需要加上一些安全属性（如我司的智能网关会加上 HTTP 签名等）
- 把新请求发送到业务服务后端节点，并接收到服务器端返回的响应
- 将上一步的响应根据需求处理（修改），然后返回给客户端

下面介绍下相关细节：

#### 使用 http.ReverseProxy 实现反向代理功能

[Golang ReverseProxy 分析](https://pandaychen.github.io/2021/07/01/GOLANG-REVERSEPROXY-LIB-ANALYSIS/) 分析了原生库 `http.ReverseProxy` 的使用，下面给出一个简单的例子（将 `:8080` 请求转发到 `http://127.0.0.1:8081`）：

```golang
func ProxyHandler(c *gin.Context) {
    remote, err := url.Parse("http://127.0.0.1:8081")
    if err != nil {
      panic(err)
    }

    proxy := httputil.NewSingleHostReverseProxy(remote)

    //设置proxy的Director属性
    proxy.Director = func(req *http.Request) {
      req.Header = c.Request.Header
      req.Host = remote.Host
      req.URL.Scheme = remote.Scheme
      req.URL.Host = remote.Host
      req.URL.Path = c.Param("proxyPath")
    }

    //设置proxy的Transport属性
  	proxy.Transport = &http.Transport{
			Proxy: http.ProxyFromEnvironment,
			DialContext: (&net.Dialer{
				Timeout:   60 * time.Second,
				KeepAlive: 60 * time.Second,
				DualStack: true,
			}).DialContext,
			MaxIdleConns:          100,
			IdleConnTimeout:       90 * time.Second,
			TLSHandshakeTimeout:   10 * time.Second,
			ExpectContinueTimeout: 1 * time.Second,
		}

    proxy.ServeHTTP(c.Writer, c.Request)
}

func main() {
    r := gin.Default()
    r.Any("/*proxyPath", ProxyHandler)
    r.Run(":8080")
}
```

如果要实现对多个不同域名 Host 的代理，那么可能需要创建类似这种格式的代理结构 `map[string]*http.ReverseProxy`，根据域名 Host 选择代理即可。

## 0x03 参考

- [Proxy route to another backend](https://github.com/gin-gonic/gin/issues/686)
- [A simple reverse proxy in Go using Gin](https://le-gall.bzh/post/go/a-reverse-proxy-in-go-using-gin/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
