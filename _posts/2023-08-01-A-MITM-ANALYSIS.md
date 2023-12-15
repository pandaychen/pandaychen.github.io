---
layout:     post
title:      中间人机制 review
subtitle:   如何优雅的实现 https mitm（透明劫持）
date:       2023-08-01
author:     pandaychen
catalog:    true
tags:
    - MITM
    - HTTPS
---


##  0x00    前言
本文探讨下 HTTPS 劫持这个话题，一些常见的网络调试工具 fiddler、charles、surge、wireshark 等，或多或少都是利用了 HTTPS 的 MITM 攻击来实现的。HTTPS 劫持的核心原理是不安全的 CA（或者是非权威 CA） 可以给任何网站 or 域名进行 CA 签名，TLS 服务端解密需要服务端私钥和服务段证书，然而这个不安全的 CA 可以提供用户暂时信任的服务端私钥和证书，通过 TLS-MITM 技术，可以劫持到客户端的明文流量

常用的客户端抓包工具fidder 就是通过客户端信任自建根证书来代理请求


####	基础回顾：TLS 握手

1、TLS 握手期间会发生的事情

在 TLS 握手过程中，客户端和服务器一同执行以下操作：
-	协商将要使用的 TLS 版本（TLS `1.0`、`1.2`、`1.3` 等）
-	决定将要使用哪些密码套件
-	通过服务器的公钥和 SSL 证书颁发机构的数字签名来验证服务器的身份
-	生成会话密钥，以在握手完成后使用对称加密

2、TLS 握手的步骤

TLS 握手是由客户端和服务器交换的一系列数据报或消息。TLS 握手涉及多个步骤，因为客户端和服务器要交换完成握手和进行进一步对话所需的信息。TLS 握手中的确切步骤将根据所使用的密钥交换算法的种类和双方支持的密码套件而有所不同。RSA 密钥交换算法虽然现在被认为不安全，但曾在 `1.3` 之前的 TLS 版本中使用。大致如下：
-	客户端问候（client hello）消息： 客户端通过向服务器发送问候消息来开始握手。该消息将包含客户端支持的 TLS 版本，支持的密码套件，以及称为一串称为客户端随机数（client random） 的随机字节
-	服务器问候（server hello） 消息： 作为对 client hello 消息的回复，服务器发送一条消息，内含服务器的 SSL 证书、服务器选择的密码套件，以及 服务器随机数（server random），即由服务器生成的另一串随机字节
-	身份验证： 客户端使用颁发该证书的证书颁发机构验证服务器的 SSL 证书。此举确认服务器是其声称的身份，且客户端正在与该域的实际所有者进行交互
-	预主密钥： 客户端再发送一串随机字节，即预主密钥（premaster secret）。预主密钥是使用公钥加密的，只能使用服务器的私钥解密（客户端从服务器的 SSL 证书中获得公钥）
-	私钥被使用：服务器对预主密钥进行解密
-	生成会话密钥：客户端和服务器均使用客户端随机数、服务器随机数和预主密钥生成会话密钥（session key）。双方应得到相同的结果
-	客户端就绪：客户端发送一条已完成消息，该消息用会话密钥加密
-	服务器就绪：服务器发送一条已完成消息，该消息用会话密钥加密
-	实现安全对称加密：已完成握手，并使用会话密钥继续进行通信

所有 TLS 握手均使用非对称加密（公钥和私钥），但并非全都会在生成会话密钥的过程中使用私钥。例如，短暂的 Diffie-Hellman 握手过程如下：
-	客户端问候：客户端发送客户端问候消息，内含协议版本、客户端随机数和密码套件列表
-	服务器问候：服务器以其 SSL 证书、其选定的密码套件和服务器随机数回复。与上述 RSA 握手相比，服务器在此消息中还包括以下内容（下步）：
-	服务器的数字签名：服务器对到此为止的所有消息计算出一个数字签名
-	数字签名确认：客户端验证服务器的数字签名，确认服务器是它所声称的身份
-	客户端 DH 参数：客户端将其 DH 参数发送到服务器
-	客户端和服务器计算预主密钥：客户端和服务器使用交换的 DH 参数分别计算匹配的预主密钥，而不像 RSA 握手那样由客户端生成预主密钥并将其发送到服务器
-	创建会话密钥：与 RSA 握手中一样，客户端和服务器现在从预主密钥、客户端随机数和服务器随机数计算会话密钥
-	客户端就绪：与 RSA 握手相同
-	服务器就绪
-	实现安全对称加密


####	TLS `1.3` 中的握手有什么不同？
TLS `1.3` 不支持 RSA，也不支持易受攻击的其他密码套件和参数。它还缩短了 TLS 握手，使 TLS `1.3` 握手更快更安全。

TLS `1.3` 握手的基本步骤为：

-	客户端问候：客户端发送客户端问候消息，内含协议版本、客户端随机数和密码套件列表。由于已从 TLS `1.3` 中删除了对不安全密码套件的支持，因此可能的密码套件数量大大减少。客户端问候消息还包括将用于计算预主密钥的参数。大体上来说，客户端假设它知道服务器的首选密钥交换方法（由于简化的密码套件列表，它有可能知道）。这减少了握手的总长度——这是 TLS `1.3` 握手与 TLS `1.0`、`1.1` 和 `1.2` 握手之间的重要区别之一
-	服务器生成主密钥：此时，服务器已经接收到客户端随机数以及客户端的参数和密码套件。它已经拥有服务器随机数，因为它可以自己生成。因此，服务器可以创建主密钥
-	服务器问候和完成：服务器问候包括服务器的证书、数字签名、服务器随机数和选择的密码套件。因为它已经有了主密钥，所以它也发送了一个完成消息
-	最后步骤和客户端完成：客户端验证签名和证书，生成主密钥，并发送完成消息
-	实现安全对称加密


####	MITM：模式
通常 MITM 可以有如下几种方式，首先于根证书的信任，必须提前内置网关伪造的根证书到用户的浏览器，下面两种都需要：

1、Explicit 模式：用户需要显式配置浏览器使用的代理，网关（MITM）在接受到 `CONNECT domain` 的 HTTP 代理请求后，执行 MITM 的逻辑

2、Transparent 模式：用户（浏览器）无需任何配置，在路由器（网关）上进行透明代理和 MITM 的操作，用户无感知

那么 SNI 代理适用于上述哪个场景呢？

##  0x01    MITM 介绍：Explicit 模式
以 [mitmproxy](https://docs.mitmproxy.org/stable/concepts-howmitmproxyworks/) 项目为例：

本模式，需要手动在客户端配置代理（设置代理以劫持 HTTPS）

![explicit](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/how-mitmproxy-works-explicit-https.png)

具体过程描述如下：

1.  先通过一个 HTTP CONNECT 请求来连接代理服务器
2.  代理服务器返回 `200 OK`，表明 CONNECT 管道建立完毕
3.  Client 开始进行 TLS 连接，中间人通过 SNI 来获知需要连接的目标是谁（TLS 握手时 Client 发送的 `ClientHello` 报文的明文字段，参考下图）
4.  中间人连接真正的服务器（TLS 连接，可能有非预期的情况，特别需要处理）
5.  中间人开始根据 SNI 和 CA 自动签发假的服务端证书，并返回给用户，并进行 TLS 握手（伪造的）
6.  握手成功，客户端开始发送 HTTP 请求
7.  中间人开始做用户到真正服务器的流量互相转发（可劫持）

至此，Explicit 模式下的 MITM 行为完成

![SNI](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/mitmproxy-tls-sni-example.png)

这里，要注意一个细节，在 `CONNECT 10.1.1.1:443 HTTP/1.1` 中，使用 IP 地址是完全合法的（并没有使用域名 / 远程主机名），这里 Mitmproxy 实现特殊的机制，可以平滑过度上游的证书嗅探。一旦发现 `CONNECT` 请求，就会暂停会话的客户端部分，并同时启动与服务器的连接。完成与服务器的 TLS 握手，并检查它使用的证书。然后使用上游证书中的 Common Name 为客户端生成虚拟证书。现在就有了正确的主机名呈现给客户端，即使它从未指定。

参考 [Complication 1: What’s the remote hostname?](https://docs.mitmproxy.org/stable/concepts-howmitmproxyworks/#complication-1-whats-the-remote-hostname)
####	Explicit模式：开发思路
根据上面描述的流程，至少需要如下子模块（以Linux客户端为例）：

1.	一个伪造的CA根证书以及对应的fake证书池，为每个域名都单独生成一个fake证书
1.	一个普通的HTTPS CONNECT代理，客户端设置`export https_proxy="http://x.x.x.x:8080"`，把 https 请求代理到该服务
2.	代理服务收到此 https 请求的 Method 是 `CONNECT`，需要再开启一个伪造的TLS服务，第一层的代理把请求转发到这个伪造的服务中，因为这个伪造的TLS服务使用了上面`1`中信任的自签根证书去签发伪造的证书，所以目标请求就会认为当前伪造的服务就是真实的服务地址，可以 Mock 到对应的 https 请求

####	证书存储（重要）
通常在项目中的做法是，先由系统生成一个 CA 证书，然后导入到将要被代理的客户端（Windows/Linux等）中，让其信任，随后再针对将要代理的请求动态生成 HTTPS 证书，通常是针对每个被代理的域名都生成一个唯一的TLS证书，该证书（`www.baidu.com`）一般如下：

```text
-----BEGIN CERTIFICATE-----
MIIDMDCCAhigAwIBAgIGD3wwj1UAMA0GCSqGSIb3DQEBCwUAMCgxEjAQBgNVBAoT
CW1pdG1wcm94eTESMBAGA1UEAxMJbWl0bXByb3h5MB4XDTIzMTIxMzAyMTUwNloX
DTI0MTIxNDAyMTUwNlowLDESMBAGA1UEChMJbWl0bXByb3h5MRYwFAYDVQQDEw13
d3cuYmFpZHUuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzGEg
1ZW7oJtmhQ2wrr7TsiuyWxgUBfqmRGRmMeHstioDPAkgKcuYCLD7O5a3qdJDo34A
t4WhfI1/i4lLFIv7QrPMFH4jMDMG+723hnNfQdVw4bvY7lh3HRMF6a2K0L4zHryn
YUhmiM5dsHJPSy+wkvZ85pKEhlq1cDU3sjlEfXNk+qWcXmTnuu5yNk2eVoksIYz4
OR0TwelQ1xSlbTdz15+steLtqxTz1KrCiZYBzxOxX0WCo7NJasMZAPkmEOZLGcMS
iIlxQ8HMG9uNmhzDsmxGLPcvM0ei7jUOfEAdQcRRarJle4YAY5iMkZRoEBngamxn
I7mq5NDoVe/gBsc1eQIDAQABo1wwWjAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYB
BQUHAwIwHwYDVR0jBBgwFoAUIKS/1e9D1LrbGFEWA5T95XsPnbcwGAYDVR0RBBEw
D4INd3d3LmJhaWR1LmNvbTANBgkqhkiG9w0BAQsFAAOCAQEAu8NtTW/cClYII6PL
NOhRJdloIU6CGqgoQT7JNIxhSBm3CVaTJsVZvdBJn3QQnXh0x/3M3x5+jtagK219
SxEkPCljj4HXOA7PKN50z7TwmoTmFkdNFMkSSFCVzqPBQRpsqcDGrKD0lj8xPf5d
qjPfXCZZ9q4achCV6jXz8OEkrCpivU0dva753TkG+C+MnsX9D0s2FbeJYutqEbxn
s24S/1aqGtV9CbADJTflxOtmyKYKS1E0fDDXbvTGfsIVkALU4S33TybkaD5WpRoT
/j8KcZXDqx9OrJYNDRwP6J2Zh29fBsVpqEFpqYH+CitxihtlVk6hFOeAkNq1jie9
FeJkWA==
-----END CERTIFICATE-----
```

证书存储模块的设计也很重要，要考虑下面几个关键点：
1.	证书存储使用的中间件，线上系统可以考虑使用redis/[cert-manager](https://cert-manager.io/)
2.	自动fake证书的签发时间？根证书的有效时间？从实践看，这个时间还是有些讲究，比如此issue[根证书最长时间不能大于18个月](https://github.com/ouqiang/goproxy/pull/22/files)
3.	通过根证书签发伪造的证书（SNI）的方法


####	fake-TLS-service构建
从MITM原理可知，针对每个客户端的HTTPS请求（唯一一个域名），代理都需要开启一个fake-tls-service来代理该HTTPS请求，所以这里如何高性能托管客户端的HTTPS访问？一个连接开启一个带端口的tls-server肯定是不现实的，肯定有更优雅的处理方式

其次，在代理中fake-tls-server如何与客户端进行TLS握手？以[kr/mitm](https://github.com/kr/mitm)项目为例：

看[`serveConnect`](https://github.com/kr/mitm/blob/master/mitm.go#L53C17-L53C29)方法的实现：

1.  正常 CONNECT 代理（等待客户端的第一个HTTP `CONNECT xxxx`代理请求过来）
2.  构建fake-tls-server需要的`tls.Config`，证书的选择一般采用SNI动态方式来获取（创建）
3.	使用上一步的fake-tls-server与客户端进行TLS握手，参考`handshake`代码，注意其中的`tls.Server(raw, config)`
4.	至此构建了两个conn：`sconn`和`cconn`，接下来通过`oneShotDialer`、`oneShotListener`配合`httputil.ReverseProxy`实现原始请求代理

```go
// handshake hijacks w's underlying net.Conn, responds to the CONNECT request
// and manually performs the TLS handshake. It returns the net.Conn or and
// error if any.
func handshake(w http.ResponseWriter, config *tls.Config) (net.Conn, error) {
	// 提取客户端的原始TCP连接
	raw, _, err := w.(http.Hijacker).Hijack()
	if err != nil {
		http.Error(w, "no upstream", 503)
		return nil, err
	}
	if _, err = raw.Write(okHeader); err != nil {
		raw.Close()
		return nil, err
	}

	// 利用tls.Server
	conn := tls.Server(raw, config)
	err = conn.Handshake()
	if err != nil {
		conn.Close()
		raw.Close()
		return nil, err
	}
	return conn, nil
}

func httpsDirector(r *http.Request) {
	r.URL.Host = r.Host
	r.URL.Scheme = "https"
}

func (p *Proxy) serveConnect(w http.ResponseWriter, r *http.Request) {
	var (
		err   error
		sconn *tls.Conn
		name  = dnsName(r.Host)
	)

	if name == "" {
		log.Println("cannot determine cert name for " + r.Host)
		http.Error(w, "no upstream", 503)
		return
	}

	provisionalCert, err := p.cert(name)
	if err != nil {
		log.Println("cert", err)
		http.Error(w, "no upstream", 503)
		return
	}

	sConfig := new(tls.Config)
	if p.TLSServerConfig != nil {
		*sConfig = *p.TLSServerConfig
	}
	sConfig.Certificates = []tls.Certificate{*provisionalCert}
	sConfig.GetCertificate = func(hello *tls.ClientHelloInfo) (*tls.Certificate, error) {
		cConfig := new(tls.Config)
		if p.TLSClientConfig != nil {
			*cConfig = *p.TLSClientConfig
		}
		cConfig.ServerName = hello.ServerName
		sconn, err = tls.Dial("tcp", r.Host, cConfig)
		if err != nil {
			log.Println("dial", r.Host, err)

			return nil, err
		}
		return p.cert(hello.ServerName)
	}

	cconn, err := handshake(w, sConfig)
	if err != nil {
		log.Println("handshake", r.Host, err)
		return
	}
	defer cconn.Close()
	if sconn == nil {
		log.Println("could not determine cert name for " + r.Host)
		return
	}
	defer sconn.Close()

	od := &oneShotDialer{c: sconn}
	rp := &httputil.ReverseProxy{
		Director:      httpsDirector,
		Transport:     &http.Transport{DialTLS: od.Dial},
		FlushInterval: p.FlushInterval,
	}

	ch := make(chan int)
	wc := &onCloseConn{cconn, func() { ch <- 0 }}

	//这段代码比较有意思
	http.Serve(&oneShotListener{wc}, p.Wrap(rp))
	<-ch
}
```

####	安装（信任）根证书


##  0x02    MTTM 介绍：Transparent 模式（透明劫持）
该模式即透明代理劫持 HTTPS，区别于依赖 HTTP Proxy 协议 / 功能的代理劫持，自然就是不需要设置代理，通常在路由器把流量设置到目标服务器上，目标服务器来进行 HTTPS 劫持。

![transparent](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/how-mitmproxy-works-transparent-https.png)

transparent 模式下的数据流程如下：

1.  客户端连接服务器
2.  路由器重定向客户端连接到中间人
3.  中间人通过 SNI 获知需要连接哪个具体的目标网站
4.  接下来的流程和 Explicit 模式下的几乎一样

通常可以在路由器上通过 iptables 来重定向客户端连接，或者使用 tun 这种高级模式等方式来实现透明代理劫持

至此，我们可知大概实现一个MITM需要哪些技术，小结如下：

1.	证书池（fake）：为每个被代理的域名都生成一个唯一的证书


##  0x03    参考实现 1：GOOGLE/Martian
[GOOGLE/Martian](https://github.com/google/martian) 是一个开源的 HTTP/HTTPS 代理库，它可以在代理服务器上拦截和修改 HTTP/HTTPS 请求和响应，以实现一些高级功能，例如请求重定向、请求修改、响应修改、请求和响应记录等。Martian 实现原理是通过在代理服务器上设置 HTTP/HTTPS 代理，拦截和修改 HTTP/HTTPS 请求和响应。Martian 支持多种代理服务器，如 net/http、goproxy、mitmproxy 等。Martian 还支持自定义规则和过滤器，可以根据用户需求进行扩展。martian基于原生的TCP进行[构建](https://github.com/google/martian/blob/master/proxy.go#L194)

-	支持 HTTP/HTTPS 代理：Martian 可以拦截和修改 HTTP/HTTPS 请求和响应，实现一些高级功能
-	支持多种代理服务器：Martian 支持多种代理服务器，例如 Go 自带的 `net/http`、goproxy、mitmproxy
-	支持自定义规则和过滤器：Martian 支持自定义规则和过滤器，可以根据用户需求进行扩展
-	支持请求重定向、请求修改、响应修改、请求和响应记录等高级功能：Martian 可以实现一些高级功能，例如请求重定向、请求修改、响应修改、请求和响应记录等

martian 的主要代理逻辑实现 [在此](https://github.com/google/martian/blob/master/proxy.go)

####    Explicit 模式：关键代码
该模式，martian 实现的核心 [代码](https://github.com/google/martian/blob/master/proxy.go#L442C17-L442C23) ，核心结构如下：

```go
// Proxy is an HTTP proxy with support for TLS MITM and customizable behavior.
type Proxy struct {
	roundTripper http.RoundTripper
	dial         func(string, string) (net.Conn, error)
	timeout      time.Duration
	mitm         *mitm.Config
	proxyURL     *url.URL
	conns        sync.WaitGroup
	connsMu      sync.Mutex // protects conns.Add/Wait from concurrent access
	closing      chan bool

	reqmod RequestModifier
	resmod ResponseModifier
}
```

1、explicit 模式代理核心实现

```GOLANG
//https://github.com/google/martian/blob/master/proxy.go#L298
func (p *Proxy) handleConnectRequest(ctx *Context, req *http.Request, session *Session, brw *bufio.ReadWriter, conn net.Conn) error {
	if err := p.reqmod.ModifyRequest(req); err != nil {
		log.Errorf("martian: error modifying CONNECT request: %v", err)
		proxyutil.Warning(req.Header, err)
	}
	if session.Hijacked() {
		log.Debugf("martian: connection hijacked by request modifier")
		return nil
	}

	if p.mitm != nil {
		log.Debugf("martian: attempting MITM for connection: %s / %s", req.Host, req.URL.String())

		res := proxyutil.NewResponse(200, nil, req)

		if err := p.resmod.ModifyResponse(res); err != nil {
			log.Errorf("martian: error modifying CONNECT response: %v", err)
			proxyutil.Warning(res.Header, err)
		}
		if session.Hijacked() {
			log.Infof("martian: connection hijacked by response modifier")
			return nil
		}

		if err := res.Write(brw); err != nil {
			log.Errorf("martian: got error while writing response back to client: %v", err)
		}
		if err := brw.Flush(); err != nil {
			log.Errorf("martian: got error while flushing response back to client: %v", err)
		}

		log.Debugf("martian: completed MITM for connection: %s", req.Host)

		b := make([]byte, 1)
		if _, err := brw.Read(b); err != nil {
			log.Errorf("martian: error peeking message through CONNECT tunnel to determine type: %v", err)
		}

		// Drain all of the rest of the buffered data.
		buf := make([]byte, brw.Reader.Buffered())
		brw.Read(buf)

		// 22 is the TLS handshake.
		// https://tools.ietf.org/html/rfc5246#section-6.2.1
		if b[0] == 22 {
			// Prepend the previously read data to be read again by
			// http.ReadRequest.
			tlsconn := tls.Server(&peekedConn{conn, io.MultiReader(bytes.NewReader(b), bytes.NewReader(buf), conn)}, p.mitm.TLSForHost(req.Host))

			if err := tlsconn.Handshake(); err != nil {
				p.mitm.HandshakeErrorCallback(req, err)
				return err
			}
			if tlsconn.ConnectionState().NegotiatedProtocol == "h2" {
				return p.mitm.H2Config().Proxy(p.closing, tlsconn, req.URL)
			}

			var nconn net.Conn
			nconn = tlsconn
			// If the original connection is a traffic shaped connection, wrap the tls
			// connection inside a traffic shaped connection too.
			if ptsconn, ok := conn.(*trafficshape.Conn); ok {
				nconn = ptsconn.Listener.GetTrafficShapedConn(tlsconn)
			}
			brw.Writer.Reset(nconn)
			brw.Reader.Reset(nconn)

            //WHY?
			return p.handle(ctx, nconn, brw)
		}

		// Prepend the previously read data to be read again by http.ReadRequest.
		brw.Reader.Reset(io.MultiReader(bytes.NewReader(b), bytes.NewReader(buf), conn))
		return p.handle(ctx, conn, brw)
	}

	log.Debugf("martian: attempting to establish CONNECT tunnel: %s", req.URL.Host)
	res, cconn, cerr := p.connect(req)
	if cerr != nil {
		log.Errorf("martian: failed to CONNECT: %v", cerr)
		res = proxyutil.NewResponse(502, nil, req)
		proxyutil.Warning(res.Header, cerr)

		if err := p.resmod.ModifyResponse(res); err != nil {
			log.Errorf("martian: error modifying CONNECT response: %v", err)
			proxyutil.Warning(res.Header, err)
		}
		if session.Hijacked() {
			log.Infof("martian: connection hijacked by response modifier")
			return nil
		}

		if err := res.Write(brw); err != nil {
			log.Errorf("martian: got error while writing response back to client: %v", err)
		}
		err := brw.Flush()
		if err != nil {
			log.Errorf("martian: got error while flushing response back to client: %v", err)
		}
		return err
	}
	defer res.Body.Close()
	defer cconn.Close()

	if err := p.resmod.ModifyResponse(res); err != nil {
		log.Errorf("martian: error modifying CONNECT response: %v", err)
		proxyutil.Warning(res.Header, err)
	}
	if session.Hijacked() {
		log.Infof("martian: connection hijacked by response modifier")
		return nil
	}

	res.ContentLength = -1
	if err := res.Write(brw); err != nil {
		log.Errorf("martian: got error while writing response back to client: %v", err)
	}
	if err := brw.Flush(); err != nil {
		log.Errorf("martian: got error while flushing response back to client: %v", err)
	}

	cbw := bufio.NewWriter(cconn)
	cbr := bufio.NewReader(cconn)
	defer cbw.Flush()

	copySync := func(w io.Writer, r io.Reader, donec chan<- bool) {
		if _, err := io.Copy(w, r); err != nil && err != io.EOF {
			log.Errorf("martian: failed to copy CONNECT tunnel: %v", err)
		}

		log.Debugf("martian: CONNECT tunnel finished copying")
		donec <- true
	}

	donec := make(chan bool, 2)
	go copySync(cbw, brw, donec)
	go copySync(brw, cbr, donec)

	log.Debugf("martian: established CONNECT tunnel, proxying traffic")
	<-donec
	<-donec
	log.Debugf("martian: closed CONNECT tunnel")

	return errClose
}
```

注意上面 `b[0] == 22` 这段代码的作用：

![sni-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/mitmproxy-tls-sni-example-1.png)

2、证书自动签发实现

回到上述 MITM 这部分代码逻辑，完成了 TLS 升级、TLS-handshake 以及 tls-mitm 的核心功能。在 Golang 中，为服务器（域名）生成 `tls.Config` 就可以升级 TLS，主要是 `p.mitm.TLSForHost(req.Host)` 这里的实现：

```GOLANG
if b[0] == 22 {
			// Prepend the previously read data to be read again by
			// http.ReadRequest.
			tlsconn := tls.Server(&peekedConn{conn, io.MultiReader(bytes.NewReader(b), bytes.NewReader(buf), conn)}, p.mitm.TLSForHost(req.Host))

			if err := tlsconn.Handshake(); err != nil {
				p.mitm.HandshakeErrorCallback(req, err)
				return err
			}
			if tlsconn.ConnectionState().NegotiatedProtocol == "h2" {
				return p.mitm.H2Config().Proxy(p.closing, tlsconn, req.URL)
			}

			var nconn net.Conn
			nconn = tlsconn
			// If the original connection is a traffic shaped connection, wrap the tls
			// connection inside a traffic shaped connection too.
			if ptsconn, ok := conn.(*trafficshape.Conn); ok {
				nconn = ptsconn.Listener.GetTrafficShapedConn(tlsconn)
			}
			brw.Writer.Reset(nconn)
			brw.Reader.Reset(nconn)
            // 上述代码中 `p.handle(ctx, nconn, brw)` 的用途是什么？
			return p.handle(ctx, nconn, brw)
		}
```

下面是 martian 自动签发证书的实现，`TLS` 方法返回一个 `*tls.Config`，该配置将使用 TLS `ClientHello` 中的 SNI 扩展动态生成证书：

```GO
// TLS returns a *tls.Config that will generate certificates on-the-fly using
// the SNI extension in the TLS ClientHello.
func (c *Config) TLS() *tls.Config {
	return &tls.Config{
		InsecureSkipVerify: c.skipVerify,
		GetCertificate: func(clientHello *tls.ClientHelloInfo) (*tls.Certificate, error) {
			if clientHello.ServerName == "" {
                // 不存在则报错
				return nil, errors.New("mitm: SNI not provided, failed to build certificate")
			}

			return c.cert(clientHello.ServerName)
		},
		NextProtos: []string{"http/1.1"},
	}
}

// TLSForHost returns a *tls.Config that will generate certificates on-the-fly
// using SNI from the connection, or fall back to the provided hostname.
func (c *Config) TLSForHost(hostname string) *tls.Config {
	return &tls.Config{
		InsecureSkipVerify: c.skipVerify,
		GetCertificate: func(clientHello *tls.ClientHelloInfo) (*tls.Certificate, error) {
			host := clientHello.ServerName
			if host == "" {
				host = hostname
			}

			return c.cert(host)
		},
		NextProtos: []string{"http/1.1"},
	}
}
```

最后看下 `cert` 方法的实现，该方法为域名 `hostname` 伪造一个由 `c.ca.Raw` 根 CA 证书签发的域名证书，** 此证书是需要发回给客户端做浏览器验证的 **：

```GO
func (c *Config) cert(hostname string) (*tls.Certificate, error) {
	// Remove the port if it exists.
	host, _, err := net.SplitHostPort(hostname)
	if err == nil {
		hostname = host
	}

	c.certmu.RLock()
	tlsc, ok := c.certs[hostname]
	c.certmu.RUnlock()

	if ok {
		log.Debugf("mitm: cache hit for %s", hostname)

		// Check validity of the certificate for hostname match, expiry, etc. In
		// particular, if the cached certificate has expired, create a new one.
		if _, err := tlsc.Leaf.Verify(x509.VerifyOptions{
			DNSName: hostname,
			Roots:   c.roots,
		}); err == nil {
			return tlsc, nil
		}

		log.Debugf("mitm: invalid certificate in cache for %s", hostname)
	}

	log.Debugf("mitm: cache miss for %s", hostname)

	serial, err := rand.Int(rand.Reader, MaxSerialNumber)
	if err != nil {
		return nil, err
	}

	tmpl := &x509.Certificate{
		SerialNumber: serial,
		Subject: pkix.Name{
			CommonName:   hostname,
			Organization: []string{c.org},
		},
		SubjectKeyId:          c.keyID,
		KeyUsage:              x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature,
		ExtKeyUsage:           []x509.ExtKeyUsage{x509.ExtKeyUsageServerAuth},
		BasicConstraintsValid: true,
		NotBefore:             time.Now().Add(-c.validity),
		NotAfter:              time.Now().Add(c.validity),
	}

	if ip := net.ParseIP(hostname); ip != nil {
		tmpl.IPAddresses = []net.IP{ip}
	} else {
		tmpl.DNSNames = []string{hostname}
	}

	raw, err := x509.CreateCertificate(rand.Reader, tmpl, c.ca, c.priv.Public(), c.capriv)
	if err != nil {
		return nil, err
	}

	// Parse certificate bytes so that we have a leaf certificate.
	x509c, err := x509.ParseCertificate(raw)
	if err != nil {
		return nil, err
	}

	tlsc = &tls.Certificate{
		Certificate: [][]byte{raw, c.ca.Raw},
		PrivateKey:  c.priv,
		Leaf:        x509c,
	}

	c.certmu.Lock()
    // 存储到本地内存中
	c.certs[hostname] = tlsc
	c.certmu.Unlock()

	return tlsc, nil
}
```


##  0x04 参考实现 2：adguard/mitm

##	0x05	再看 martian：代理核心流程走读
本小节，梳理下martian的[mitm](https://github.com/google/martian/blob/master/mitm/mitm.go)主要数据流程：


##	0x06	参考实现3：lqqyt2423/go-mitmproxy

此项目基于`net.http`实现比较巧妙，思路可借鉴，大致流程如下：

![work-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/mitm-project/go-mitmproxy-workflow.png)

####	核心结构
本项目封装了较多的结构，理清这些对了解项目的实现非常有用

1、客户端连接`ClientConn`

```GO
// client connection
type ClientConn struct {
	Id           uuid.UUID
	Conn         net.Conn
	Tls          bool
	UpstreamCert bool // Connect to upstream server to look up certificate details. Default: True
	clientHello  *tls.ClientHelloInfo
}
```

2、`ServerConn`

```go
// server connection
type ServerConn struct {
	Id      uuid.UUID
	Address string
	Conn    net.Conn

	tlsHandshaked   chan struct{}
	tlsHandshakeErr error
	tlsConn         *tls.Conn
	tlsState        *tls.ConnectionState
	client          *http.Client
}
```

3、管理结构`Proxy`，主要包含下面三个成员
-	客户端：`http.Client`
-	服务端（本地代理）：`http.Server`
-	mitm（middle）：`middle`核心[结构](https://github.com/lqqyt2423/go-mitmproxy/blob/main/proxy/interceptor.go#L83)

```GO
type Proxy struct {
	Opts    *Options
	Version string
	Addons  []Addon

	client          *http.Client	//用于请求真实域名的客户端
	server          *http.Server	
	interceptor     *middle
	shouldIntercept func(req *http.Request) bool              // req is received by proxy.server
	upstreamProxy   func(req *http.Request) (*url.URL, error) // req is received by proxy.server, not client request
}
```

####	中间人实现：middle
这个也是本项目的[核心代码](https://github.com/lqqyt2423/go-mitmproxy/blob/main/proxy/interceptor.go#L83)实现，模拟了标准库中 server 运行，目的是仅通过当前进程内存转发 socket 数据，不需要经过 tcp 或 unix socket


```GO
// middle: man-in-the-middle server
type middle struct {
	proxy    *Proxy
	ca       *cert.CA
	listener *middleListener
	server   *http.Server
}
```

```go
// mock net.Listener
type middleListener struct {
	connChan chan net.Conn
	doneChan chan struct{}
}

func (l *middleListener) Accept() (net.Conn, error) {
	select {
	case c := <-l.connChan:
		return c, nil
	case <-l.doneChan:
		return nil, http.ErrServerClosed
	}
}
func (l *middleListener) Close() error   { return nil }
func (l *middleListener) Addr() net.Addr { return nil }
```

1、client与server成员初始化

```go
func newMiddle(proxy *Proxy) (*middle, error) {
	ca, err := cert.NewCA(proxy.Opts.CaRootPath)
	if err != nil {
		return nil, err
	}

	m := &middle{
		proxy: proxy,
		ca:    ca,
		listener: &middleListener{
			connChan: make(chan net.Conn),
			doneChan: make(chan struct{}),
		},
	}

	server := &http.Server{
		Handler: m,	 // HTTP 请求处理器，用于处理所有传入的 HTTP 请求
		ConnContext: func(ctx context.Context, c net.Conn) context.Context {
			return context.WithValue(ctx, connContextKey, c.(*tls.Conn).NetConn().(*pipeConn).connContext)
		},
		TLSNextProto: make(map[string]func(*http.Server, *tls.Conn, http.Handler)), // disable http2,，禁用 HTTP/2 协议。这将强制服务器仅使用 HTTP/1.1 协议
		TLSConfig: &tls.Config{
			SessionTicketsDisabled: true, // 设置此值为 true ，确保每次都会调用下面的 GetCertificate 方法
			GetCertificate: func(clientHello *tls.ClientHelloInfo) (*tls.Certificate, error) {
				//注意context的用法
				connCtx := clientHello.Context().Value(connContextKey).(*ConnContext)
				connCtx.ClientConn.clientHello = clientHello

				if connCtx.ClientConn.UpstreamCert {
					if err := connCtx.tlsHandshake(clientHello); err != nil {
						return nil, err
					}

					for _, addon := range connCtx.proxy.Addons {
						addon.TlsEstablishedServer(connCtx)
					}
				}

				return ca.GetCert(clientHello.ServerName)
			},
		},
	}
	m.server = server
	return m, nil
}

// 启动middle.server
func (m *middle) start() error {
	//注意：这里的listener是自定义实现的
	return m.server.ServeTLS(m.listener, "", "")
}
```

`middle`初始化的代码中，有几个细节需要注意：

-	`http.Server`的`ConnContext`成员的用途？
-	`http.TLSConfig.GetCertificate`的用途？
-	`m.server.ServeTLS(m.listener, "", "")`：tls的证书路径均为空，是什么用途？

首先看下`ConnContext`成员，其定义了一个函数，用于将 `net.Conn` 类型的连接转换为 `context.Context`，首先将连接类型转换为 `*tls.Conn`，然后获取其底层的 `*pipeConn`，并从中提取 `connContext`，允许在处理 HTTP 请求时访问与连接相关的上下文信息

`TLSConfig`是自定义的 TLS 配置，包括以下设置：
-	`SessionTicketsDisabled`为`true`禁用 TLS 会话票证，以确保每次都会调用 `GetCertificate` 方法
-	`GetCertificate` 用于在 TLS 握手过程中为给定的服务器名称（`SNI`）生成证书。该方法首先从 `ClientHello` 消息中提取与连接相关的上下文信息（`connContext`），然后根据 `UpstreamCert` 的值决定是否执行 TLS 握手。如果 `UpstreamCert` 为 `true`，则执行 TLS 握手并调用所有已注册的插件的 `TlsEstablishedServer` 方法。最后，使用 `ca.GetCert` 方法为服务器名称生成fakeTLS握手证书

所以，`m.server.ServeTLS`调用的参数为空时，证书和私钥将由 `GetCertificate` 方法动态提供，这也是常用的SNI代理的实现细节


####	核心流程分析

1、服务启动
-	启动https[代理](https://github.com/lqqyt2423/go-mitmproxy/blob/main/proxy/proxy.go#L110C9-L110C32)：`proxy.server.Serve(pln)`
-	启动mitm-fake-server：`go proxy.interceptor.start()`


```go
func (m *middle) start() error {
	//注意：这里的listener是自定义实现的
	return m.server.ServeTLS(m.listener, "", "")
}
```


核心代理逻辑入口[Proxy.ServeHTTP](https://github.com/lqqyt2423/go-mitmproxy/blob/main/proxy/proxy.go#L125)：

2、`handleConnect`方法：基于标准协议处理https隧道连接，注意到最后的逻辑`transfer(log, conn, cconn)`数据流双向转发

-	`conn`：mitm下由`proxy.interceptor.dial(req)`生成
-	`cconn`：

```go
func (proxy *Proxy) handleConnect(res http.ResponseWriter, req *http.Request) {
	shouldIntercept := proxy.shouldIntercept == nil || proxy.shouldIntercept(req)
	f := newFlow()
	f.Request = newRequest(req)
	f.ConnContext = req.Context().Value(connContextKey).(*ConnContext)
	f.ConnContext.Intercept = shouldIntercept
	defer f.finish()

	// trigger addon event Requestheaders
	for _, addon := range proxy.Addons {
		addon.Requestheaders(f)
	}

	var conn net.Conn
	var err error
	if shouldIntercept {
		log.Debugf("begin intercept %v", req.Host)
		conn, err = proxy.interceptor.dial(req)  //下一步看看dial做了什么
	} else {
		log.Debugf("begin transpond %v", req.Host)
		conn, err = proxy.getUpstreamConn(req)
	}
	if err != nil {
		log.Error(err)
		res.WriteHeader(502)
		return
	}
	defer conn.Close()

	cconn, _, err := res.(http.Hijacker).Hijack()
	if err != nil {
		log.Error(err)
		res.WriteHeader(502)
		return
	}

	// cconn.(*net.TCPConn).SetLinger(0) // send RST other than FIN when finished, to avoid TIME_WAIT state
	// cconn.(*net.TCPConn).SetKeepAlive(false)
	defer cconn.Close()

	_, err = io.WriteString(cconn, "HTTP/1.1 200 Connection Established\r\n\r\n")
	if err != nil {
		log.Error(err)
		return
	}

	f.Response = &Response{
		StatusCode: 200,
		Header:     make(http.Header),
	}

	// trigger addon event Responseheaders
	for _, addon := range proxy.Addons {
		addon.Responseheaders(f)
	}
	defer func(f *Flow) {
		// trigger addon event Response
		for _, addon := range proxy.Addons {
			addon.Response(f)
		}
	}(f)

	transfer(log, conn, cconn)
}
```

3、proxy.interceptor.dial(req)

-	使用newPipes构建管道，分为读端与写端
-	单独异步处理`pipeServerConn`，把`pipeClientConn`返回
-	`m.listener.connChan <- pipeServerConn`

```go
func (m *middle) dial(req *http.Request) (net.Conn, error) {
	pipeClientConn, pipeServerConn := newPipes(req)

	if pipeServerConn.connContext.ClientConn.UpstreamCert {
		err := pipeServerConn.connContext.initServerTcpConn(req)
		if err != nil {
			pipeClientConn.Close()
			pipeServerConn.Close()
			return nil, err
		}
	}

	go m.intercept(pipeServerConn)
	return pipeClientConn, nil
}

// 解析 connect 流量
// 如果是 tls 流量，则进入 listener.Accept => Middle.ServeHTTP
// 否则很可能是 ws 流量
func (m *middle) intercept(pipeServerConn *pipeConn) {
	buf, err := pipeServerConn.Peek(3)
	if err != nil {
		log.Errorf("Peek error: %v\n", err)
		pipeServerConn.Close()
		return
	}

	// https://github.com/mitmproxy/mitmproxy/blob/main/mitmproxy/net/tls.py is_tls_record_magic
	if buf[0] == 0x16 && buf[1] == 0x03 && buf[2] <= 0x03 {
		// tls
		pipeServerConn.connContext.ClientConn.Tls = true
		pipeServerConn.connContext.initHttpsServerConn()

		//这里首先如果是TLS协议，
		m.listener.connChan <- pipeServerConn
	} else {
		// ws
		defaultWebSocket.ws(pipeServerConn, pipeServerConn.host)
	}
}
```

4、在step `3`中把该conn递交给`middle.server`处理，这段代码比较晦涩，整体流程如下：

-	首先，`middle.dial`中，构建pipe连接，然后通过`go m.intercept(pipeServerConn)`开启异步处理tls连接
-	第二步，`middle.intercept`中`m.listener.connChan <- pipeServerConn`，将连接`pipeServerConn`异步丢给`m.listener`
-	第三步，`middleListener.Accept`中收到此连接`pipeServerConn`，会触发`middle.server.GetCertificate`的逻辑，进行TLS握手，这中间会调用[`connCtx.tlsHandshake`](https://github.com/lqqyt2423/go-mitmproxy/blob/main/proxy/connection.go#L247)方法（TLS握手，证书兼容性协商等操作）
-	TLS握手完成后，然后流程到达`middle.ServeHTTP`，这里可以捕获到客户端https的真正的请求（`req`）
-	最后，`middle.ServeHTTP`方法中，在对`req`设置真实的请求后，会调用`m.proxy.ServeHTTP(res, req)`，完成最后的逻辑

```go
func (l *middleListener) Accept() (net.Conn, error) {
	select {
	case c := <-l.connChan:
		return c, nil
	case <-l.doneChan:
		return nil, http.ErrServerClosed
	}
}

func (m *middle) ServeHTTP(res http.ResponseWriter, req *http.Request) {
	if strings.EqualFold(req.Header.Get("Connection"), "Upgrade") && strings.EqualFold(req.Header.Get("Upgrade"), "websocket") {
		// wss
		defaultWebSocket.wss(res, req)
		return
	}

	if req.URL.Scheme == "" {
		req.URL.Scheme = "https"
	}
	if req.URL.Host == "" {
		req.URL.Host = req.Host
	}

	// 完成最后的逻辑
	// 这里可以获取到客户端原始的请求信息
	// GET / HTTP/1.1 1 1 map[Accept:[] User-Agent:[curl/7.29.0]]
	m.proxy.ServeHTTP(res, req)
}
```

5、最后一步，`m.proxy.ServeHTTP(res, req)`，完成对目的域名的请求，并把响应转发回真正的客户端，完成MITM过程

本质上，就是先利用参数`req`中的数据向目标域名发送https（TLS）请求，然后获取结果之后，利用`reply`自定义方法，把相关的数据写入到`res http.ResponseWriter`中，最终会通过下面这条路线完成数据转发到客户端的过程

-	[`newPipes(req *http.Request) (net.Conn, *pipeConn)`](https://github.com/lqqyt2423/go-mitmproxy/blob/main/proxy/interceptor.go#L59)
-	[`conn, err = proxy.interceptor.dial(req)`](https://github.com/lqqyt2423/go-mitmproxy/blob/main/proxy/proxy.go#L348C3-L348C42)
-	[`transfer(log, conn, cconn)`](https://github.com/lqqyt2423/go-mitmproxy/blob/main/proxy/proxy.go#L393)


```GO
func (proxy *Proxy) ServeHTTP(res http.ResponseWriter, req *http.Request) {
	if req.Method == "CONNECT" {
		proxy.handleConnect(res, req)
		return
	}

	if !req.URL.IsAbs() || req.URL.Host == "" {
		if len(proxy.Addons) == 0 {
			res.WriteHeader(400)
			io.WriteString(res, "此为代理服务器，不能直接发起请求")
			return
		}
		for _, addon := range proxy.Addons {
			addon.AccessProxyServer(req, res)
		}
		return
	}

	reply := func(response *Response, body io.Reader) {
		if response.Header != nil {
			for key, value := range response.Header {
				for _, v := range value {
					res.Header().Add(key, v)
				}
			}
		}
		if response.close {
			res.Header().Add("Connection", "close")
		}
		res.WriteHeader(response.StatusCode)

		if body != nil {
			_, err := io.Copy(res, body)
			if err != nil {
				logErr(log, err)
			}
		}
		if response.BodyReader != nil {
			_, err := io.Copy(res, response.BodyReader)
			if err != nil {
				logErr(log, err)
			}
		}
		if response.Body != nil && len(response.Body) > 0 {
			_, err := res.Write(response.Body)
			if err != nil {
				logErr(log, err)
			}
		}
	}

	// when addons panic
	defer func() {
		if err := recover(); err != nil {
			log.Warnf("Recovered: %v\n", err)
		}
	}()

	f := newFlow()
	f.Request = newRequest(req)
	f.ConnContext = req.Context().Value(connContextKey).(*ConnContext)
	defer f.finish()

	f.ConnContext.FlowCount = f.ConnContext.FlowCount + 1

	rawReqUrlHost := f.Request.URL.Host
	rawReqUrlScheme := f.Request.URL.Scheme

	// trigger addon event Requestheaders
	for _, addon := range proxy.Addons {
		addon.Requestheaders(f)
		if f.Response != nil {
			reply(f.Response, nil)
			return
		}
	}

	// Read request body
	var reqBody io.Reader = req.Body
	if !f.Stream {
		reqBuf, r, err := readerToBuffer(req.Body, proxy.Opts.StreamLargeBodies)
		reqBody = r
		if err != nil {
			log.Error(err)
			res.WriteHeader(502)
			return
		}

		if reqBuf == nil {
			log.Warnf("request body size >= %v\n", proxy.Opts.StreamLargeBodies)
			f.Stream = true
		} else {
			f.Request.Body = reqBuf

			// trigger addon event Request
			for _, addon := range proxy.Addons {
				addon.Request(f)
				if f.Response != nil {
					reply(f.Response, nil)
					return
				}
			}
			reqBody = bytes.NewReader(f.Request.Body)
		}
	}

	for _, addon := range proxy.Addons {
		reqBody = addon.StreamRequestModifier(f, reqBody)
	}

	proxyReqCtx := context.WithValue(context.Background(), proxyReqCtxKey, req)
	proxyReq, err := http.NewRequestWithContext(proxyReqCtx, f.Request.Method, f.Request.URL.String(), reqBody)
	if err != nil {
		log.Error(err)
		res.WriteHeader(502)
		return
	}

	for key, value := range f.Request.Header {
		for _, v := range value {
			proxyReq.Header.Add(key, v)
		}
	}

	f.ConnContext.initHttpServerConn()

	useSeparateClient := f.UseSeparateClient
	if !useSeparateClient {
		if rawReqUrlHost != f.Request.URL.Host || rawReqUrlScheme != f.Request.URL.Scheme {
			useSeparateClient = true
		}
	}

	var proxyRes *http.Response
	if useSeparateClient {
		proxyRes, err = proxy.client.Do(proxyReq)
	} else {
		proxyRes, err = f.ConnContext.ServerConn.client.Do(proxyReq)
	}
	if err != nil {
		logErr(log, err)
		res.WriteHeader(502)
		return
	}

	if proxyRes.Close {
		f.ConnContext.closeAfterResponse = true
	}

	defer proxyRes.Body.Close()

	f.Response = &Response{
		StatusCode: proxyRes.StatusCode,
		Header:     proxyRes.Header,
		close:      proxyRes.Close,
	}

	// trigger addon event Responseheaders
	for _, addon := range proxy.Addons {
		addon.Responseheaders(f)
		if f.Response.Body != nil {
			reply(f.Response, nil)
			return
		}
	}

	// Read response body
	var resBody io.Reader = proxyRes.Body
	if !f.Stream {
		resBuf, r, err := readerToBuffer(proxyRes.Body, proxy.Opts.StreamLargeBodies)
		resBody = r
		if err != nil {
			log.Error(err)
			res.WriteHeader(502)
			return
		}
		if resBuf == nil {
			log.Warnf("response body size >= %v\n", proxy.Opts.StreamLargeBodies)
			f.Stream = true
		} else {
			f.Response.Body = resBuf

			// trigger addon event Response
			for _, addon := range proxy.Addons {
				addon.Response(f)
			}
		}
	}
	for _, addon := range proxy.Addons {
		resBody = addon.StreamResponseModifier(f, resBody)
	}

	reply(f.Response, resBody)
}
```

####	小结
最后，汇总下mitm的数据流程：

![mitm](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/mitm-project/https-mitm.png)


##	0x07	参考实现：ouqiang/goproxy（MITM实现）
[中间人代理, 解密HTTPS](https://github.com/ouqiang/goproxy#%E4%B8%AD%E9%97%B4%E4%BA%BA%E4%BB%A3%E7%90%86-%E8%A7%A3%E5%AF%86https)，用户可以自行实现`Delegate`结构，用于在mitm中实现自定义的[逻辑](https://github.com/ouqiang/goproxy)

使用方法如下，初始化中需要传入[`WithDecryptHTTPS`](https://github.com/ouqiang/goproxy/blob/master/proxy.go#L191)方法，用于开启HTTPS解密以及fake证书存储

```GO
func main() {
	//cache需要开发者自行实现
	proxy := goproxy.New(goproxy.WithDecryptHTTPS(&Cache{}))
	server := &http.Server{
		Addr:         ":8080",
		Handler:      proxy,
		ReadTimeout:  1 * time.Minute,
		WriteTimeout: 1 * time.Minute,
	}
	err := server.ListenAndServe()
	if err != nil {
		panic(err)
	}
}
```

```go
type Delegate interface {
	// Connect 收到客户端连接
	Connect(ctx *Context, rw http.ResponseWriter)
	// Auth 代理身份认证
	Auth(ctx *Context, rw http.ResponseWriter)
	// BeforeRequest HTTP请求前 设置X-Forwarded-For, 修改Header、Body
	BeforeRequest(ctx *Context)
	// BeforeResponse 响应发送到客户端前, 修改Header、Body、Status Code
	BeforeResponse(ctx *Context, resp *http.Response, err error)
	// ParentProxy 上级代理
	ParentProxy(*http.Request) (*url.URL, error)
	// Finish 本次请求结束
	Finish(ctx *Context)
	// 记录错误信息
	ErrorLog(err error)
}
```

此外，用户需要实现证书的缓存接口[`Cache`](https://github.com/ouqiang/goproxy/blob/master/cert/cache.go#L21C6-L21C12)，该方法用于把mitm的域名生成的fake-certificate存储，存储可选内存/redis等

```go
// 实现证书缓存接口
type Cache struct {
	m sync.Map
}

func (c *Cache) Set(host string, cert *tls.Certificate) {
	c.m.Store(host, cert)
}
func (c *Cache) Get(host string) *tls.Certificate {
	v, ok := c.m.Load(host)
	if !ok {
		return nil
	}

	return v.(*tls.Certificate)
}
```

####	核心结构

1、`Context`：封装`http.Request`，用于保存最原始的http请求

```GO
// Context 代理上下文
type Context struct {
	Req         *http.Request
	Data        map[interface{}]interface{}
	TunnelProxy bool
	abort       bool
}

```

2、`Proxy`：代理核心结构

```GO
type Proxy struct {
	delegate           Delegate
	clientConnNum      int32
	decryptHTTPS       bool
	websocketIntercept bool
	cert               *cert.Certificate
	transport          *http.Transport		//用于向真实服务器发送https请求
	clientTrace        *httptrace.ClientTrace
	dnsCache           *dnscache.Resolver
}

// Proxy 实现了http.Handler接口
func (p *Proxy) ServeHTTP(rw http.ResponseWriter, req *http.Request){
	//...
}
```

####	MITM：核心流程
从`ServeHTTP`[方法](https://github.com/ouqiang/goproxy/blob/master/proxy.go#L259)入手

1、`CONNECT`隧道代理

```go
// ServeHTTP 实现了http.Handler接口
func (p *Proxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	if req.URL.Host == "" {
		req.URL.Host = req.Host
	}
	atomic.AddInt32(&p.clientConnNum, 1)

	//sync.Pool的典型使用
	ctx := ctxPool.Get().(*Context)
	ctx.Reset(req)

	defer func() {
		p.delegate.Finish(ctx)
		ctxPool.Put(ctx)
		atomic.AddInt32(&p.clientConnNum, -1)
	}()
	p.delegate.Connect(ctx, rw)
	if ctx.abort {
		return
	}
	p.delegate.Auth(ctx, rw)
	if ctx.abort {
		return
	}

	switch {
	case ctx.Req.Method == http.MethodConnect:
		//https隧道代理
		p.tunnelProxy(ctx, rw)
	case websocket.IsWebSocketUpgrade(ctx.Req):
		p.tunnelProxy(ctx, rw)
	default:
		p.httpProxy(ctx, rw)
	}
}
```

2、`tunnelProxy`：隧道代理的完整实现，支持websocket（核心实现见注释），有几个方法需要特别注意：

-	[`ReadRequest`](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/net/http/request.go;l=1023)

```go
// 隧道代理
func (p *Proxy) tunnelProxy(ctx *Context, rw http.ResponseWriter) {
	clientConn, err := hijacker(rw)
	if err != nil {
		p.delegate.ErrorLog(err)
		rw.WriteHeader(http.StatusBadGateway)
		return
	}
	defer func() {
		_ = clientConn.Close()
	}()

	// ......
	
	//下面是核心的MITM代理流程
	var tlsClientConn *tls.Conn
	if p.decryptHTTPS {
		// 为待解密的域名生成临时证书
		tlsConfig, err := p.cert.GenerateTlsConfig(ctx.Req.URL.Host)
		if err != nil {
			p.tunnelConnected(ctx, err)
			p.delegate.ErrorLog(fmt.Errorf("%s - HTTPS解密, 生成证书失败: %s", ctx.Req.URL.Host, err))
			return
		}

		// 构造一个fake-tls-server与客户端完成TLS握手
		tlsClientConn = tls.Server(clientConn, tlsConfig)
		defer func() {
			_ = tlsClientConn.Close()
		}()
		if err := tlsClientConn.Handshake(); err != nil {
			p.tunnelConnected(ctx, err)
			p.delegate.ErrorLog(fmt.Errorf("%s - HTTPS解密, 握手失败: %s", ctx.Req.URL.Host, err))
			return
		}

		// 
		buf := bufio.NewReader(tlsClientConn)
		tlsReq, err := http.ReadRequest(buf)
		if err != nil {
			if err != io.EOF {
				p.tunnelConnected(ctx, err)
				p.delegate.ErrorLog(fmt.Errorf("%s - HTTPS解密, 读取客户端请求失败: %s", ctx.Req.URL.Host, err))
			}
			return
		}
		tlsReq.RemoteAddr = ctx.Req.RemoteAddr
		tlsReq.URL.Scheme = "https"
		tlsReq.URL.Host = tlsReq.Host
		ctx.Req = tlsReq
	}

	targetAddr := ctx.Req.URL.Host
	if parentProxyURL != nil {
		targetAddr = parentProxyURL.Host
	}
	if !strings.Contains(targetAddr, ":") {
		targetAddr += ":443"
	}

	//向真实的server拨号
	targetConn, err := net.DialTimeout("tcp", targetAddr, defaultTargetConnectTimeout)
	if err != nil {
		p.tunnelConnected(ctx, err)
		p.delegate.ErrorLog(fmt.Errorf("%s - 隧道转发连接目标服务器失败: %s", ctx.Req.URL.Host, err))
		return
	}
	defer func() {
		_ = targetConn.Close()
	}()
	if parentProxyURL != nil {
		tunnelRequestLine := makeTunnelRequestLine(ctx.Req.URL.Host)
		_, _ = targetConn.Write([]byte(tunnelRequestLine))
	}

	if p.decryptHTTPS {
		// https代理
		p.httpsProxy(ctx, tlsClientConn)
	} else {
		p.tunnelConnected(ctx, nil)
		p.transfer(clientConn, targetConn)
	}
}
```

3、`httpsProxy`方法：执行HTTP请求，并调用`responseFunc`处理真正的服务器响应，这里最关键的代码是

-	通过`resp, err := p.transport.RoundTrip(newReq)`获取到服务端真实的响应
-	再通过`err = resp.Write(tlsClientConn)`将响应返回给原始的客户端

```go
// HTTPS代理
func (p *Proxy) httpsProxy(ctx *Context, tlsClientConn *tls.Conn) {
	if websocket.IsWebSocketUpgrade(ctx.Req) {
		p.websocketProxy(ctx, NewConnBuffer(tlsClientConn, nil))
		return
	}
	p.DoRequest(ctx, func(resp *http.Response, err error) {
		if err != nil {
			p.delegate.ErrorLog(fmt.Errorf("%s - HTTPS解密, 请求错误: %s", ctx.Req.URL, err))
			_, _ = tlsClientConn.Write(badGateway)
			return
		}
		//最关键的方法
		err = resp.Write(tlsClientConn)
		if err != nil {
			p.delegate.ErrorLog(fmt.Errorf("%s - HTTPS解密, response写入客户端失败, %s", ctx.Req.URL, err))
		}
		_ = resp.Body.Close()
	})
}

// DoRequest 
func (p *Proxy) DoRequest(ctx *Context, responseFunc func(*http.Response, error)) {
	if ctx.Data == nil {
		ctx.Data = make(map[interface{}]interface{})
	}
	p.delegate.BeforeRequest(ctx)
	if ctx.abort {
		return
	}
	newReq := requestPool.Get()
	*newReq = *ctx.Req
	newHeader := headerPool.Get()

	//注意：必须要重新复制一份
	CloneHeader(newReq.Header, newHeader)
	newReq.Header = newHeader
	for _, item := range hopHeaders {
		if newReq.Header.Get(item) != "" {
			newReq.Header.Del(item)
		}
	}
	if p.clientTrace != nil {
		newReq = newReq.WithContext(httptrace.WithClientTrace(newReq.Context(), p.clientTrace))
	}

	resp, err := p.transport.RoundTrip(newReq)
	p.delegate.BeforeResponse(ctx, resp, err)
	if ctx.abort {
		return
	}
	if err == nil {
		for _, h := range hopHeaders {
			resp.Header.Del(h)
		}
	}
	responseFunc(resp, err)	// 调用上面httpsProxy方法中的P.DoRequest参数
	headerPool.Put(newHeader)
	requestPool.Put(newReq)
}
```

核心需要理解`err = resp.Write(tlsClientConn)`这段逻辑是如何[实现](https://cs.opensource.google/go/go/+/refs/tags/go1.21.4:src/net/http/response.go;l=245)的

##  0x0 思考：MITM 防护手段


##  0x06    总结
类似的项目：

-   [mitm - mitm is a SSL-capable man-in-the-middle proxy for use with golang net/http](https://github.com/kr/mitm)
-	[About mitmproxy implemented with golang](https://github.com/lqqyt2423/go-mitmproxy)


##	0x07	
核心原理：不安全的CA导致信任链崩坏

##  0x06    参考
-   [How mitmproxy works](https://docs.mitmproxy.org/stable/concepts-howmitmproxyworks/)
-   [mitmproxy docs](https://docs.mitmproxy.org/stable/)
-   [Subject Key Identifier support for end entity certificate.](https://www.ietf.org/rfc/rfc3280.txt)
-   [Integrate the martian lib in the KrakenD framework](https://github.com/krakend/krakend-martian)
-   [martian-main](https://github.com/google/martian/blob/master/cmd/proxy/main.go)
- [What is HTTPS filtering](https://adguard.com/kb/zh-CN/general/https-filtering/what-is-https-filtering/)
- [Overview](https://adguard.com/kb/zh-CN/)
- [AdGuard Home - Wiki](https://github.com/AdguardTeam/AdGuardHome/wiki)
- [AdGuard content blocking library in golang](https://github.com/AdguardTeam/urlfilter)
- [Simple golang mitm proxy implementation](https://github.com/AdguardTeam/gomitmproxy)
- [Martian is a library for building custom HTTP/S proxies](https://github.com/google/martian)
- [Modifier Reference](https://github.com/google/martian/wiki/Modifier-Reference)
- [TLS 握手期间会发生什么？SSL 握手](https://www.cloudflare-cn.com/learning/ssl/what-happens-in-a-tls-handshake/)
- [All MITM attacks in one place.](https://github.com/frostbits-security/MITM-cheatsheet)
- [Does https prevent man in the middle attacks by proxy server?](https://security.stackexchange.com/questions/8145/does-https-prevent-man-in-the-middle-attacks-by-proxy-server)
- [使用 cert-manager 签发免费证书](https://cloud.tencent.com/document/product/457/49368)