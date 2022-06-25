---
layout: post
title: 使用 Golang 实现 SSH 和 SSHD（二）
subtitle: golang-ssh 库使用 && gliderlabs/ssh 分析（实战篇）
date: 2020-11-01
author: pandaychen
catalog: true
tags:
  - OpenSSH
---

## 0x00 前言
先记录一下自己实现sshd（proxy）相关系统中需要搞懂的几个重要的ssh数据结构

## 0x01 重要结构

1、Channel<br>
A Channel is an ordered, reliable, flow-controlled, duplex stream that is multiplexed over an SSH connection.（注，个人理解是channel最大的作用是给开发者提供了读取SSH明文的手段）

```golang
type Channel interface {
	// Read reads up to len(data) bytes from the channel.
	Read(data []byte) (int, error)

	// Write writes len(data) bytes to the channel.
	Write(data []byte) (int, error)

	// Close signals end of channel use. No data may be sent after this
	// call.
	Close() error

	// CloseWrite signals the end of sending in-band
	// data. Requests may still be sent, and the other side may
	// still send data
	CloseWrite() error

	// SendRequest sends a channel request.  If wantReply is true,
	// it will wait for a reply and return the result as a
	// boolean, otherwise the return value will be false. Channel
	// requests are out-of-band messages so they may be sent even
	// if the data stream is closed or blocked by flow control.
	// If the channel is closed before a reply is returned, io.EOF
	// is returned.
	SendRequest(name string, wantReply bool, payload []byte) (bool, error)

	// Stderr returns an io.ReadWriter that writes to this channel
	// with the extended data type set to stderr. Stderr may
	// safely be read and written from a different goroutine than
	// Read and Write respectively.
	Stderr() io.ReadWriter
}
```

2、Request 和 Reply<br>
协议交互的收发包协议结构：

Request is a request sent outside of the normal stream of data. Requests can either be specific to an SSH channel, or they can be global.

```golang
type Request struct {
	Type      string
	WantReply bool
	Payload   []byte
	// contains filtered or unexported fields
}
```

<br>
Reply sends a response to a request. It must be called for all requests where WantReply is true and is a no-op otherwise. The payload argument is ignored for replies to channel-specific requests.<br>

```golang
func (r *Request) Reply(ok bool, payload []byte) error
```

3、核心结构 Session<br>
文档：[Session](https://pkg.go.dev/golang.org/x/crypto/ssh#Session)
A Session represents a connection to a remote command or shell.

```golang
type Session struct {
	// Stdin specifies the remote process's standard input.
	// If Stdin is nil, the remote process reads from an empty
	// bytes.Buffer.
	Stdin io.Reader

	// Stdout and Stderr specify the remote process's standard
	// output and error.
	//
	// If either is nil, Run connects the corresponding file
	// descriptor to an instance of ioutil.Discard. There is a
	// fixed amount of buffering that is shared for the two streams.
	// If either blocks it may eventually cause the remote
	// command to block.
	Stdout io.Writer
	Stderr io.Writer
	// contains filtered or unexported fields
}
```

4、Conn<br>
Conn 是一个`interface{}`
Conn represents an SSH connection for both server and client roles. Conn is the basis for implementing an application layer, such as ClientConn, which implements the traditional shell access for clients.<br>

```golang
type Conn interface {
	ConnMetadata

	// SendRequest sends a global request, and returns the
	// reply. If wantReply is true, it returns the response status
	// and payload. See also RFC4254, section 4.
	SendRequest(name string, wantReply bool, payload []byte) (bool, []byte, error)

	// OpenChannel tries to open an channel. If the request is
	// rejected, it returns *OpenChannelError. On success it returns
	// the SSH Channel and a Go channel for incoming, out-of-band
	// requests. The Go channel must be serviced, or the
	// connection will hang.
	OpenChannel(name string, data []byte) (Channel, <-chan *Request, error)

	// Close closes the underlying network connection
	Close() error

	// Wait blocks until the connection has shut down, and returns the
	// error causing the shutdown.
	Wait() error
}
```

5、ConnMetadata<br>
ConnMetadata holds metadata for the connection.<br>

```golang
type ConnMetadata interface {
	// User returns the user ID for this connection.
	User() string

	// SessionID returns the session hash, also denoted by H.
	SessionID() []byte

	// ClientVersion returns the client's version string as hashed
	// into the session ID.
	ClientVersion() []byte

	// ServerVersion returns the server's version string as hashed
	// into the session ID.
	ServerVersion() []byte

	// RemoteAddr returns the remote address for this connection.
	RemoteAddr() net.Addr

	// LocalAddr returns the local address for this connection.
	LocalAddr() net.Addr
}
```

6、
ServerConn is an authenticated SSH connection, as seen from the server<br>

```golang
type ServerConn struct {
	Conn

	// If the succeeding authentication callback returned a
	// non-nil Permissions pointer, it is stored here.
	Permissions *Permissions
}
```

7、Session<br>
A Session represents a connection to a remote command or shell.
```golang
type Session struct {
	// Stdin specifies the remote process's standard input.
	// If Stdin is nil, the remote process reads from an empty
	// bytes.Buffer.
	Stdin io.Reader

	// Stdout and Stderr specify the remote process's standard
	// output and error.
	//
	// If either is nil, Run connects the corresponding file
	// descriptor to an instance of ioutil.Discard. There is a
	// fixed amount of buffering that is shared for the two streams.
	// If either blocks it may eventually cause the remote
	// command to block.
	Stdout io.Writer
	Stderr io.Writer
	// contains filtered or unexported fields
}
```

## 0x02 认证、配置相关

1、Signer<br>
A Signer can create signatures that verify against a public key.<br>

```golang
type Signer interface {
	// PublicKey returns an associated PublicKey instance.
	PublicKey() PublicKey

	// Sign returns raw signature for the given data. This method
	// will apply the hash specified for the keytype to the data.
	Sign(rand io.Reader, data []byte) (*Signature, error)
}
```

2、PublicKey<br>
PublicKey is an abstraction of different types of public keys.<br>

```golang
type PublicKey interface {
	// Type returns the key's type, e.g. "ssh-rsa".
	Type() string

	// Marshal returns the serialized key data in SSH wire format,
	// with the name prefix. To unmarshal the returned data, use
	// the ParsePublicKey function.
	Marshal() []byte

	// Verify that sig is a signature on the given data using this
	// key. This function will hash the data appropriately first.
	Verify(data []byte, sig *Signature) error
}
```

3、CertChecker<br>
CertChecker does the work of verifying a certificate. Its methods can be plugged into ClientConfig.HostKeyCallback and ServerConfig.PublicKeyCallback. For the CertChecker to work, minimally, the IsAuthority callback should be set.

```golang
type CertChecker struct {
	// SupportedCriticalOptions lists the CriticalOptions that the
	// server application layer understands. These are only used
	// for user certificates.
	SupportedCriticalOptions []string

	// IsUserAuthority should return true if the key is recognized as an
	// authority for the given user certificate. This allows for
	// certificates to be signed by other certificates. This must be set
	// if this CertChecker will be checking user certificates.
	IsUserAuthority func(auth PublicKey) bool

	// IsHostAuthority should report whether the key is recognized as
	// an authority for this host. This allows for certificates to be
	// signed by other keys, and for those other keys to only be valid
	// signers for particular hostnames. This must be set if this
	// CertChecker will be checking host certificates.
	IsHostAuthority func(auth PublicKey, address string) bool

	// Clock is used for verifying time stamps. If nil, time.Now
	// is used.
	Clock func() time.Time

	// UserKeyFallback is called when CertChecker.Authenticate encounters a
	// public key that is not a certificate. It must implement validation
	// of user keys or else, if nil, all such keys are rejected.
	UserKeyFallback func(conn ConnMetadata, key PublicKey) (*Permissions, error)

	// HostKeyFallback is called when CertChecker.CheckHostKey encounters a
	// public key that is not a certificate. It must implement host key
	// validation or else, if nil, all such keys are rejected.
	HostKeyFallback HostKeyCallback

	// IsRevoked is called for each certificate so that revocation checking
	// can be implemented. It should return true if the given certificate
	// is revoked and false otherwise. If nil, no certificates are
	// considered to have been revoked.
	IsRevoked func(cert *Certificate) bool
}
```

4、ClientConfig<br>
A ClientConfig structure is used to configure a Client. It must not be modified after having been passed to an SSH function.<br>

```golang
type ClientConfig struct {
	// Config contains configuration that is shared between clients and
	// servers.
	Config

	// User contains the username to authenticate as.
	User string

	// Auth contains possible authentication methods to use with the
	// server. Only the first instance of a particular RFC 4252 method will
	// be used during authentication.
	Auth []AuthMethod

	// HostKeyCallback is called during the cryptographic
	// handshake to validate the server's host key. The client
	// configuration must supply this callback for the connection
	// to succeed. The functions InsecureIgnoreHostKey or
	// FixedHostKey can be used for simplistic host key checks.
	HostKeyCallback HostKeyCallback

	// BannerCallback is called during the SSH dance to display a custom
	// server's message. The client configuration can supply this callback to
	// handle it as wished. The function BannerDisplayStderr can be used for
	// simplistic display on Stderr.
	BannerCallback BannerCallback

	// ClientVersion contains the version identification string that will
	// be used for the connection. If empty, a reasonable default is used.
	ClientVersion string

	// HostKeyAlgorithms lists the key types that the client will
	// accept from the server as host key, in order of
	// preference. If empty, a reasonable default is used. Any
	// string returned from PublicKey.Type method may be used, or
	// any of the CertAlgoXxxx and KeyAlgoXxxx constants.
	HostKeyAlgorithms []string

	// Timeout is the maximum amount of time for the TCP connection to establish.
	//
	// A Timeout of zero means no timeout.
	Timeout time.Duration
}
```

5、ServerConfig<br>
ServerConfig holds server specific configuration data.<br>

```golang
type ServerConfig struct {
	// Config contains configuration shared between client and server.
	Config

	// NoClientAuth is true if clients are allowed to connect without
	// authenticating.
	NoClientAuth bool

	// MaxAuthTries specifies the maximum number of authentication attempts
	// permitted per connection. If set to a negative number, the number of
	// attempts are unlimited. If set to zero, the number of attempts are limited
	// to 6.
	MaxAuthTries int

	// PasswordCallback, if non-nil, is called when a user
	// attempts to authenticate using a password.
	PasswordCallback func(conn ConnMetadata, password []byte) (*Permissions, error)

	// PublicKeyCallback, if non-nil, is called when a client
	// offers a public key for authentication. It must return a nil error
	// if the given public key can be used to authenticate the
	// given user. For example, see CertChecker.Authenticate. A
	// call to this function does not guarantee that the key
	// offered is in fact used to authenticate. To record any data
	// depending on the public key, store it inside a
	// Permissions.Extensions entry.
	PublicKeyCallback func(conn ConnMetadata, key PublicKey) (*Permissions, error)

	// KeyboardInteractiveCallback, if non-nil, is called when
	// keyboard-interactive authentication is selected (RFC
	// 4256). The client object's Challenge function should be
	// used to query the user. The callback may offer multiple
	// Challenge rounds. To avoid information leaks, the client
	// should be presented a challenge even if the user is
	// unknown.
	KeyboardInteractiveCallback func(conn ConnMetadata, client KeyboardInteractiveChallenge) (*Permissions, error)

	// AuthLogCallback, if non-nil, is called to log all authentication
	// attempts.
	AuthLogCallback func(conn ConnMetadata, method string, err error)

	// ServerVersion is the version identification string to announce in
	// the public handshake.
	// If empty, a reasonable default is used.
	// Note that RFC 4253 section 4.2 requires that this string start with
	// "SSH-2.0-".
	ServerVersion string

	// BannerCallback, if present, is called and the return string is sent to
	// the client after key exchange completed but before authentication.
	BannerCallback func(conn ConnMetadata) string

	// GSSAPIWithMICConfig includes gssapi server and callback, which if both non-nil, is used
	// when gssapi-with-mic authentication is selected (RFC 4462 section 3).
	GSSAPIWithMICConfig *GSSAPIWithMICConfig
	// contains filtered or unexported fields
}
```

##	0x03	gliderlabs/ssh库实现分析
这个库封装了golang-ssh的接口，使得可以通过该库直接开发我们需要的ssh应用，如sshd、ssh交互式命令行等等。此外，该库在重要的位置都预留了用户钩子，开发者可以将自己的逻辑内嵌进去。

####	代码组织
-	[server.go](https://github.com/gliderlabs/ssh/blob/master/server.go)：封装了sshd实现相关的结构及接口
-	[session.go](https://github.com/gliderlabs/ssh/blob/master/session.go)：封装了ssh-session相关的结构及接口
-	[ssh.go](https://github.com/gliderlabs/ssh/blob/master/ssh.go)：提供了供用户调用的外部接口，如`Serve`、`ListenAndServe`等等
-	[tcpip.go](https://github.com/gliderlabs/ssh/blob/master/tcpip.go)：提供了端口转发的实现，以及`DirectTCPIPHandler`方法（可直接调用）
-	[agent.go](https://github.com/gliderlabs/ssh/blob/master/agent.go)：提供了ssh-agent相关的接口及实现
-	[context.go](https://github.com/gliderlabs/ssh/blob/master/context.go)：封装`context.Context`，加入ssh的属性，使得每条连接都关联一个唯一的context


####	Session结构
`ssh.Session`抽象了一条ssh会话，以及该会话可以获取到的所有关联的属性
```golang
type Session interface {
	gossh.Channel

	// User returns the username used when establishing the SSH connection.
	User() string

	// RemoteAddr returns the net.Addr of the client side of the connection.
	RemoteAddr() net.Addr

	// LocalAddr returns the net.Addr of the server side of the connection.
	LocalAddr() net.Addr

	// Environ returns a copy of strings representing the environment set by the
	// user for this session, in the form "key=value".
	Environ() []string

	// Exit sends an exit status and then closes the session.
	Exit(code int) error

	// Command returns a shell parsed slice of arguments that were provided by the
	// user. Shell parsing splits the command string according to POSIX shell rules,
	// which considers quoting not just whitespace.
	Command() []string

	// RawCommand returns the exact command that was provided by the user.
	RawCommand() string

	// Subsystem returns the subsystem requested by the user.
	Subsystem() string

	// PublicKey returns the PublicKey used to authenticate. If a public key was not
	// used it will return nil.
	PublicKey() PublicKey

	// Context returns the connection's context. The returned context is always
	// non-nil and holds the same data as the Context passed into auth
	// handlers and callbacks.
	//
	// The context is canceled when the client's connection closes or I/O
	// operation fails.
	Context() Context

	// Permissions returns a copy of the Permissions object that was available for
	// setup in the auth handlers via the Context.
	Permissions() Permissions

	// Pty returns PTY information, a channel of window size changes, and a boolean
	// of whether or not a PTY was accepted for this session.
	Pty() (Pty, <-chan Window, bool)

	// Signals registers a channel to receive signals sent from the client. The
	// channel must handle signal sends or it will block the SSH request loop.
	// Registering nil will unregister the channel from signal sends. During the
	// time no channel is registered signals are buffered up to a reasonable amount.
	// If there are buffered signals when a channel is registered, they will be
	// sent in order on the channel immediately after registering.
	Signals(c chan<- Signal)

	// Break regisers a channel to receive notifications of break requests sent
	// from the client. The channel must handle break requests, or it will block
	// the request handling loop. Registering nil will unregister the channel.
	// During the time that no channel is registered, breaks are ignored.
	Break(c chan<- bool)
}
```

上面公共接口对应的实例化实现如下：
```golang
type session struct {
	sync.Mutex
	gossh.Channel
	conn              *gossh.ServerConn	//指向上层的连接
	handler           Handler
	subsystemHandlers map[string]SubsystemHandler
	handled           bool
	exited            bool
	pty               *Pty
	winch             chan Window
	env               []string
	ptyCb             PtyCallback
	sessReqCb         SessionRequestCallback
	rawCmd            string
	subsystem         string
	ctx               Context
	sigCh             chan<- Signal
	sigBuf            []Signal
	breakCh           chan<- bool
}
```

这里有个比较重要的成员`gossh.Channel`，见第一节，该数据结构是SSH数据层面通信的核心结构（对应原生库的实例化[结构](https://cs.opensource.google/go/x/crypto/+/793ad666:ssh/channel.go;l=150)是`channel`），其核心方法如下：
-	`SendRequest`：request类型的请求发送
-	`Read`：读取明文数据
-	`Write`：写入明文数据

封装`gossh.Channel`的意义是，一是可以直接调用`gossh.Channel`暴露的方法来完成ssh数据层面的转发/发送；二是可以基于此来实现ssh会话数据的劫持（即SSH会话审计）

####	默认的session处理方法：DefaultSessionHandler
项目提供了默认的ssh session处理方法`DefaultSessionHandler`，在`DefaultSessionHandler`方法中，只提供了常见的`reqeusts`的处理，并未提供`pty/tty`之类的对接（需要由开发者自行实现），比如：
-	基于`top`命令的带tty的命令行[实现](https://github.com/pandaychen/openssh-magic-change/blob/master/sshd/top.go)
-	基于`/bin/bash`的bash命令交互式[实现](https://github.com/pandaychen/openssh-magic-change/blob/master/sshd/tty.go)

```golang
func DefaultSessionHandler(srv *Server, conn *gossh.ServerConn, newChan gossh.NewChannel, ctx Context) {
	ch, reqs, err := newChan.Accept()		//获取ssh的channel及requests
	if err != nil {
		return
	}
	sess := &session{
		Channel:           ch,
		conn:              conn,
		handler:           srv.Handler,
		ptyCb:             srv.PtyCallback,
		sessReqCb:         srv.SessionRequestCallback,
		subsystemHandlers: srv.SubsystemHandlers,
		ctx:               ctx,
	}
	sess.handleRequests(reqs)
}
```

`handleRequests`方法，提供了不少标准协议的`requests`类型实现，参见RFC[文档](https://datatracker.ietf.org/doc/html/rfc4250#section-4.9.3)，这里注意两点：
1.	数据需要按照ssh的协议格式编码，如`gossh.Unmarshal`
2.	需要应答的请求，一定要调用`req.Reply`进行响应

```golang
func (sess *session) handleRequests(reqs <-chan *gossh.Request) {
	for req := range reqs {
		switch req.Type {
		case "shell", "exec":
			if sess.handled {
				req.Reply(false, nil)
				continue
			}

			var payload = struct{ Value string }{}
			gossh.Unmarshal(req.Payload, &payload)
			sess.rawCmd = payload.Value

			// If there's a session policy callback, we need to confirm before
			// accepting the session.
			if sess.sessReqCb != nil && !sess.sessReqCb(sess, req.Type) {
				sess.rawCmd = ""
				req.Reply(false, nil)
				continue
			}

			sess.handled = true
			req.Reply(true, nil)

			go func() {
				sess.handler(sess)
				sess.Exit(0)
			}()
		case "subsystem":
			if sess.handled {
				req.Reply(false, nil)
				continue
			}

			var payload = struct{ Value string }{}
			gossh.Unmarshal(req.Payload, &payload)
			sess.subsystem = payload.Value

			// If there's a session policy callback, we need to confirm before
			// accepting the session.
			if sess.sessReqCb != nil && !sess.sessReqCb(sess, req.Type) {
				sess.rawCmd = ""
				req.Reply(false, nil)
				continue
			}

			handler := sess.subsystemHandlers[payload.Value]
			if handler == nil {
				handler = sess.subsystemHandlers["default"]
			}
			if handler == nil {
				req.Reply(false, nil)
				continue
			}

			sess.handled = true
			req.Reply(true, nil)

			go func() {
				handler(sess)
				sess.Exit(0)
			}()
		case "env":
			if sess.handled {
				req.Reply(false, nil)
				continue
			}
			var kv struct{ Key, Value string }
			gossh.Unmarshal(req.Payload, &kv)
			sess.env = append(sess.env, fmt.Sprintf("%s=%s", kv.Key, kv.Value))
			req.Reply(true, nil)
		case "signal":
			var payload struct{ Signal string }
			gossh.Unmarshal(req.Payload, &payload)
			sess.Lock()
			if sess.sigCh != nil {
				sess.sigCh <- Signal(payload.Signal)
			} else {
				if len(sess.sigBuf) < maxSigBufSize {
					sess.sigBuf = append(sess.sigBuf, Signal(payload.Signal))
				}
			}
			sess.Unlock()
		case "pty-req":
			if sess.handled || sess.pty != nil {
				req.Reply(false, nil)
				continue
			}
			ptyReq, ok := parsePtyRequest(req.Payload)
			if !ok {
				req.Reply(false, nil)
				continue
			}
			if sess.ptyCb != nil {
				ok := sess.ptyCb(sess.ctx, ptyReq)
				if !ok {
					req.Reply(false, nil)
					continue
				}
			}
			sess.pty = &ptyReq
			sess.winch = make(chan Window, 1)
			sess.winch <- ptyReq.Window
			defer func() {
				// when reqs is closed
				close(sess.winch)
			}()
			req.Reply(ok, nil)
		case "window-change":
			if sess.pty == nil {
				req.Reply(false, nil)
				continue
			}
			win, ok := parseWinchRequest(req.Payload)
			if ok {
				sess.pty.Window = win
				sess.winch <- win
			}
			req.Reply(ok, nil)
		case agentRequestType:
			// TODO: option/callback to allow agent forwarding
			SetAgentRequested(sess.ctx)
			req.Reply(true, nil)
		case "break":
			ok := false
			sess.Lock()
			if sess.breakCh != nil {
				sess.breakCh <- true
				ok = true
			}
			req.Reply(ok, nil)
			sess.Unlock()
		default:
			// TODO: debug log
			req.Reply(false, nil)
		}
	}
}
```


## 0x04 参考
-	[ssh包](https://pkg.go.dev/golang.org/x/crypto/ssh)
-	[gliderlabs-ssh包](https://pkg.go.dev/github.com/gliderlabs/ssh)
-	[Understanding SSH](https://containerssh.io/development/containerssh/ssh/)
