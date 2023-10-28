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


####	基础回顾：TLS 握手

1、TLS 握手期间会发生的事情

在 TLS 握手过程中，客户端和服务器一同执行以下操作：
-	协商将要使用的 TLS 版本（TLS 1.0、1.2、1.3 等）
-	决定将要使用哪些密码套件
-	通过服务器的公钥和 SSL 证书颁发机构的数字签名来验证服务器的身份
-	生成会话密钥，以在握手完成后使用对称加密

2、TLS 握手的步骤

TLS 握手是由客户端和服务器交换的一系列数据报或消息。TLS 握手涉及多个步骤，因为客户端和服务器要交换完成握手和进行进一步对话所需的信息。TLS 握手中的确切步骤将根据所使用的密钥交换算法的种类和双方支持的密码套件而有所不同。RSA 密钥交换算法虽然现在被认为不安全，但曾在 1.3 之前的 TLS 版本中使用。大致如下：
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


####	TLS 1.3 中的握手有什么不同？
TLS 1.3 不支持 RSA，也不支持易受攻击的其他密码套件和参数。它还缩短了 TLS 握手，使 TLS 1.3 握手更快更安全。

TLS 1.3 握手的基本步骤为：

-	客户端问候：客户端发送客户端问候消息，内含协议版本、客户端随机数和密码套件列表。由于已从 TLS 1.3 中删除了对不安全密码套件的支持，因此可能的密码套件数量大大减少。客户端问候消息还包括将用于计算预主密钥的参数。大体上来说，客户端假设它知道服务器的首选密钥交换方法（由于简化的密码套件列表，它有可能知道）。这减少了握手的总长度——这是 TLS 1.3 握手与 TLS 1.0、1.1 和 1.2 握手之间的重要区别之一
-	服务器生成主密钥：此时，服务器已经接收到客户端随机数以及客户端的参数和密码套件。它已经拥有服务器随机数，因为它可以自己生成。因此，服务器可以创建主密钥
-	服务器问候和完成：服务器问候包括服务器的证书、数字签名、服务器随机数和选择的密码套件。因为它已经有了主密钥，所以它也发送了一个完成消息
-	最后步骤和客户端完成：客户端验证签名和证书，生成主密钥，并发送完成消息
-	实现安全对称加密


####	MITM：模式
通常 MITM 可以有如下几种方式，首先于根证书的信任，必须提前内置网关伪造的根证书到用户的浏览器，下面两种都需要：

1、Explicit 模式

用户需要显式配置浏览器使用的代理，网关（MITM）在接受到 `CONNECT domain` 的 HTTP 代理请求后，执行 MITM 的逻辑

2、Transparent 模式

用户（浏览器）无需任何配置，在路由器（网关）上进行透明代理和 MITM 的操作，用户无感知

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



##  0x0 思考：MITM 防护手段


##  0x06    总结
类似的项目：

-   [mitm - mitm is a SSL-capable man-in-the-middle proxy for use with golang net/http](https://github.com/kr/mitm)
-	[About mitmproxy implemented with golang](https://github.com/lqqyt2423/go-mitmproxy)

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
-	[TLS 握手期间会发生什么？| SSL 握手](https://www.cloudflare-cn.com/learning/ssl/what-happens-in-a-tls-handshake/)
-	[All MITM attacks in one place.](https://github.com/frostbits-security/MITM-cheatsheet)
-	[Does https prevent man in the middle attacks by proxy server?](https://security.stackexchange.com/questions/8145/does-https-prevent-man-in-the-middle-attacks-by-proxy-server)