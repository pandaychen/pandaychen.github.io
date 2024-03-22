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

API 网关主要提供以下的功能：

- 反向代理和路由：网关给出了访问后端 API 的所有客户端的单一入口，并隐藏内部服务部署的细节
- 负载均衡：网关能够将单个传入的请求路由到多个后端目的地
- 身份验证和授权：网关能够成功进行身份验证并仅允许可信客户端访问 API，并且还能够使用类似 RBAC 等方式来授权
- IP 列表白 / 黑名单：允许或阻止某些 IP 地址通过
- 性能分析：提供一种记录与 API 调用相关的使用和其他有用度量的方法
- 限速和流控：控制 API 调用的能力
- 请求修改：能够在转发之前转换请求和响应（包括 Header 和 Body）
- 版本控制：同时使用不同版本的 API 选项或可能以金丝雀发布或蓝 / 绿部署的形式提供慢速推出 API
- breaker：微服务架构模式有用，以避免使用中断
- 多协议支持：WebSocket/GRPC 等
- 缓存：减少网络带宽和往返时间消耗，如果可以缓存频繁要求的数据，则可以提高性能和响应时间
- API 文档：如果计划将 API 暴露给组织以外的开发人员，那么必须考虑使用 API 文档，例如 Swagger

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

    // 设置 proxy 的 Director 属性
    proxy.Director = func(req *http.Request) {
      req.Header = c.Request.Header
      req.Host = remote.Host
      req.URL.Scheme = remote.Scheme
      req.URL.Host = remote.Host
      req.URL.Path = c.Param("proxyPath")
    }

    // 设置 proxy 的 Transport 属性
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

##  0x03  微服务网关参考
目前汇总业界开源的微服务 API 网关实现：

1、[didi/GateKeeper](https://github.com/didi/GateKeeper)：A high-performance Golang gateway that supports rapid development and plug-inization

- [0.1](https://github.com/didi/GateKeeper/tree/v0.1) 版本
- [重构](https://github.com/didi/GateKeeper/tree/master) 版本

2、[tyk](https://github.com/TykTechnologies/tyk)：Tyk Open Source API Gateway written in Go, supporting REST, GraphQL, TCP and gRPC protocols

3、[manba](https://github.com/fagongzi/manba)：HTTP API Gateway

4、[janus](https://github.com/motiv-labs/janus)：An API Gateway written in Go

5、[krakend](https://www.krakend.io/)：KrakenD is an open-source API Gateway that helps you effortlessly adopt microservices and secure communications. KrakenD aims for performance, scalability and simplicity, easing operations and scaling, without a single point of failure. It's been widely adopted: ~2M servers are running KrakenD monthly around the world.

6、[goku_lite](https://github.com/eolinker/goku_lite)：是一个基于 Golang 开发的微服务网关，能够实现高性能 HTTP API 转发、服务编排、多租户管理、API 访问权限控制等目的，拥有强大的自定义插件系统可以自行扩展，并且提供友好的图形化配置界面，能够快速帮助企业进行 API 服务治理、提高 API 服务的稳定性和安全性

7、[apinto](https://github.com/eolinker/apinto)：基于 golang 开发的网关。具有各种插件，可以自行扩展，即插即用。此外，它可以快速帮助企业管理 API 服务，提高 API 服务的稳定性和安全性

####  正 / 反向代理参考
- [goproxy](https://github.com/elazarl/goproxy)：An HTTP proxy library for Go


####  网关功能对比

TODO

##  0x04  Oxy

TODO

##  0x05  martian库
笔者项目中使用 martian 构建 MITM 代理功能，参考 [martian](https://github.com/google/martian/blob/master/cmd/proxy/main.go) 的示例代码构建一个代理程序；KrakenD 使用了 [gin](https://gin-gonic.github.io/gin/) 作为 `http(s)` 的 route engine，使用了 google 发布的 [Martian](https://github.com/google/martian) 作为 proxy engine，此库很值得一读，它的描述如下：

```text
Martian Proxy is a programmable HTTP proxy designed to be used for testing.
Martian is a great tool to use if you want to:
Verify that all (or some subset) of requests are secure
Mock external services at the network layer
Inject headers, modify cookies or perform other mutations of HTTP requests and responses
Verify that pingbacks happen when you think they should
Unwrap encrypted traffic (requires install of CA certificate in browser)
By taking advantage of Go cross-compilation, Martian can be deployed anywhere that Go can target.
Martian can also be included into any Go program and used as a library
```


通过下面的方式来快速实现一个 proxy：

```BASH
go get github.com/google/martian/
go install github.com/google/martian/cmd/proxy
$GOPATH/bin/proxy -addr=:9999 -api-addr=:9898
#By default, Martian will be running on port 8080, and the Martian API will be running on 8181 . The port can be specified via flags:
```

Martian 支持强大的配置，如在 proxy 的过程中，来修改 request 和 response，可以参考此文档 [Modifier Reference](https://github.com/google/martian/wiki/Modifier-Reference)

```JSON
{
    "header.Modifier": {
      "scope": ["response"],
      "name": "Test-Header",
      "value": "true"
    }
  }
{
    "querystring.Modifier": {
      "scope": ["request", "response"],
      "name": "foo",
      "value": "bar"
    }
  }
{
    "url.Modifier": {
      "scope": ["request"],
      "scheme": "https",
      "host": "www.google.com",
      "path": "/proxy",
      "query": "testing=true"
    }
  }
{
    "body.Modifier": {
      "scope": ["request", "response"],
      "contentType": "text/plain; charset=utf-8",
      "body": "TWFydGlhbiBpcyBhd2Vzb21lIQ=="
    }
  }
```

## 0x06 参考

- [Proxy route to another backend](https://github.com/gin-gonic/gin/issues/686)
- [A simple reverse proxy in Go using Gin](https://le-gall.bzh/post/go/a-reverse-proxy-in-go-using-gin/)
- [网关选择](https://chunlife.top/2018/09/22/%E7%BD%91%E5%85%B3%E9%80%89%E6%8B%A9/)
- [开源 API 网关大全 20 款](https://www.iamle.com/archives/2591.html)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
