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

##  0x02    MTTM 介绍：Transparent 模式（透明劫持）
该模式即透明代理劫持 HTTPS，区别于依赖 HTTP Proxy 协议 / 功能的代理劫持，自然就是不需要设置代理，通常在路由器把流量设置到目标服务器上，目标服务器来进行 HTTPS 劫持。

![transparent](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/mitm/how-mitmproxy-works-transparent-https.png)

transparent 模式下的数据流程如下：

1.  客户端连接服务器
2.  路由器重定向客户端连接到中间人
3.  中间人通过 SNI 获知需要连接哪个具体的目标网站
4.  接下来的流程和 Explicit 模式下的几乎一样

通常可以在路由器上通过 iptables 来重定向客户端连接，或者使用 tun 这种高级模式等方式来实现透明代理劫持



##  0x03    参考实现 1：GOOGLE/Martian

GOOGLE/Martian 是一个开源的 HTTP/HTTPS 代理库，它可以在代理服务器上拦截和修改 HTTP/HTTPS 请求和响应，以实现一些高级功能，例如请求重定向、请求修改、响应修改、请求和响应记录等。Martian 的主要应用场景是在开发和测试过程中，帮助开发人员进行 HTTP/HTTPS 请求和响应的调试和测试。

Martian 的实现原理是通过在代理服务器上设置 HTTP/HTTPS 代理，拦截和修改 HTTP/HTTPS 请求和响应。Martian 使用 Go 语言编写，支持多种代理服务器，例如 Go 自带的 net/http、goproxy、mitmproxy 等。Martian 还支持自定义规则和过滤器，可以根据用户需求进行扩展。

Martian 的主要特点如下：

支持 HTTP/HTTPS 代理：Martian 可以拦截和修改 HTTP/HTTPS 请求和响应，实现一些高级功能。

支持多种代理服务器：Martian 支持多种代理服务器，例如 Go 自带的 net/http、goproxy、mitmproxy 等。

支持自定义规则和过滤器：Martian 支持自定义规则和过滤器，可以根据用户需求进行扩展。

支持请求重定向、请求修改、响应修改、请求和响应记录等高级功能：Martian 可以实现一些高级功能，例如请求重定向、请求修改、响应修改、请求和响应记录等。


####    Explicit 模式：关键代码
该模式，martian 实现的核心 [代码](https://github.com/google/martian/blob/master/proxy.go#L442C17-L442C23) 如下：


核心结构如下：

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


##  0x05 思考：MITM 防护手段


##  0x06    总结
类似的项目：

-   [mitm - mitm is a SSL-capable man-in-the-middle proxy for use with golang net/http](https://github.com/kr/mitm)

##  0x06    参考
-   [How mitmproxy works](https://docs.mitmproxy.org/stable/concepts-howmitmproxyworks/)
-   [mitmproxy docs](https://docs.mitmproxy.org/stable/)
-   [Subject Key Identifier support for end entity certificate.](https://www.ietf.org/rfc/rfc3280.txt)
-   [Integrate the martian lib in the KrakenD framework](https://github.com/krakend/krakend-martian)
-   [martian-main](https://github.com/google/martian/blob/master/cmd/proxy/main.go)