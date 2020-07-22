---
layout:     post
title:      Kratos 源码分析：Warden 之 gRPC-Server 封装
subtitle:   分析 Warden 的 Server 端封装
date:       2020-07-20
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
Kratos 的 Warden 框架 [server.go](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/server.go) 封装了 gRPC 的服务端启动的核心逻辑。
-   服务端的启动 & 配置流程
-   拦截器链的 "安装" 顺序
-   `tracer`、`metrics` 以及 `limiter` 与 `grpc.Server` 的结合
-	过载保护实现

##  0x01	Server 端拦截器
服务端的拦截器的顺序如下图所示：
![image](https://wx1.sbimg.cn/2020/05/12/grpc-server-.png)

##  0x02    Server 配置
默认配置如下，需要注意的是这里的 Keepalive 相关的配置：
```golang
defaultSerConf = &ServerConfig{
    Network:           "tcp",
    Addr:              "0.0.0.0:9000",
    Timeout:           xtime.Duration(time.Second),
    IdleTimeout:       xtime.Duration(time.Second * 180),
    MaxLifeTime:       xtime.Duration(time.Hour * 2),
    ForceCloseWait:    xtime.Duration(time.Second * 20),
    KeepAliveInterval: xtime.Duration(time.Second * 60),
    KeepAliveTimeout:  xtime.Duration(time.Second * 20),
}
```
对应于下面这个 `ServerConfig` 的配置结构：
```golang
// ServerConfig is rpc server conf.
type ServerConfig struct {
	// Network is grpc listen network,default value is tcp
	Network string `dsn:"network"`
	// Addr is grpc listen addr,default value is 0.0.0.0:9000
	Addr string `dsn:"address"`
	// Timeout is context timeout for per rpc call.
	Timeout xtime.Duration `dsn:"query.timeout"`
	// IdleTimeout is a duration for the amount of time after which an idle connection would be closed by sending a GoAway.
	// Idleness duration is defined since the most recent time the number of outstanding RPCs became zero or the connection establishment.
	IdleTimeout xtime.Duration `dsn:"query.idleTimeout"`
	// MaxLifeTime is a duration for the maximum amount of time a connection may exist before it will be closed by sending a GoAway.
	// A random jitter of +/-10% will be added to MaxConnectionAge to spread out connection storms.
	MaxLifeTime xtime.Duration `dsn:"query.maxLife"`
	// ForceCloseWait is an additive period after MaxLifeTime after which the connection will be forcibly closed.
	ForceCloseWait xtime.Duration `dsn:"query.closeWait"`
	// KeepAliveInterval is after a duration of this time if the server doesn't see any activity it pings the client to see if the transport is still alive.
	KeepAliveInterval xtime.Duration `dsn:"query.keepaliveInterval"`
	// KeepAliveTimeout  is After having pinged for keepalive check, the server waits for a duration of Timeout and if no activity is seen even after that
	// the connection is closed.
	KeepAliveTimeout xtime.Duration `dsn:"query.keepaliveTimeout"`
	// LogFlag to control log behaviour. e.g. LogFlag: warden.LogFlagDisableLog.
	// Disable: 1 DisableArgs: 2 DisableInfo: 4
	LogFlag int8 `dsn:"query.logFlag"`
}
```

##  0x02	Server运行分析
`warden.Server` 结构如下，其封装了 `grpc.Server`，值得注意的是 `handlers []grpc.UnaryServerInterceptor` 结构，这里存储了**服务端的拦截器数组**：
```golang
// Server is the framework's server side instance, it contains the GrpcServer, interceptor and interceptors.
// Create an instance of Server, by using NewServer().
type Server struct {
	conf  *ServerConfig
	mutex sync.RWMutex

	server   *grpc.Server
	handlers []grpc.UnaryServerInterceptor
}
```

而 `mutex` 是用于更新 `conf` 时的并发保护：
```golang
......
// SetConfig hot reloads server config
func (s *Server) SetConfig(conf *ServerConfig) (err error) {
    s.mutex.Lock()
	s.conf = conf
	s.mutex.Unlock()
......
}
```

####    初始化 Server
初始化包含如下几个步骤：
1.	解析 `DSN` 配置
2.	初始化 `grpc.Server` 配置：`s.server = grpc.NewServer(opt...)`
3.	使用 `s.Use(xxx)` 初始化服务端拦截器数组（一定要注意拦截器的顺序！）

```golang
// NewServer returns a new blank Server instance with a default server interceptor.
func NewServer(conf *ServerConfig, opt ...grpc.ServerOption) (s *Server) {
	if conf == nil {
		if !flag.Parsed() {
			fmt.Fprint(os.Stderr, "[warden] please call flag.Parse() before Init warden server, some configure may not effect\n")
		}
		conf = parseDSN(_grpcDSN)
	} else {
		fmt.Fprintf(os.Stderr, "[warden] config is Deprecated, argument will be ignored. please use -grpc flag or GRPC env to configure warden server.\n")
	}
	s = new(Server)
	if err := s.SetConfig(conf); err != nil {
		panic(errors.Errorf("warden: set config failed!err: %s", err.Error()))
	}
	keepParam := grpc.KeepaliveParams(keepalive.ServerParameters{
		MaxConnectionIdle:     time.Duration(s.conf.IdleTimeout),
		MaxConnectionAgeGrace: time.Duration(s.conf.ForceCloseWait),
		Time:                  time.Duration(s.conf.KeepAliveInterval),
		Timeout:               time.Duration(s.conf.KeepAliveTimeout),
		MaxConnectionAge:      time.Duration(s.conf.MaxLifeTime),
	})
	opt = append(opt, keepParam, grpc.UnaryInterceptor(s.interceptor))
	s.server = grpc.NewServer(opt...)

	//初始化拦截器数组
	//s.recovery()必须要在第一个
	s.Use(s.recovery(), s.handle(), serverLogging(conf.LogFlag), s.stats(), s.validate())
	s.Use(ratelimiter.New(nil).Limit())
	return
}
```

####    warden.Server 实现的方法

启动服务端：
```golang
// Start create a new goroutine run server with configured listen addr
// will panic if any error happend
// return server itself
func (s *Server) Start() (*Server, error) {
	_, err := s.startWithAddr()
	if err != nil {
		return nil, err
	}
	return s, nil
}
```

```golang
func (s *Server) startWithAddr() (net.Addr, error) {
	lis, err := net.Listen(s.conf.Network, s.conf.Addr)
	if err != nil {
		return nil, err
	}
	log.Info("warden: start grpc listen addr: %v", lis.Addr())
	reflection.Register(s.server)
	go func() {
		if err := s.Serve(lis); err != nil {
			panic(err)
		}
	}()
	return lis.Addr(), nil
}

// Serve accepts incoming connections on the listener lis, creating a new
// ServerTransport and service goroutine for each.
// Serve will return a non-nil error unless Stop or GracefulStop is called.
func (s *Server) Serve(lis net.Listener) error {
	return s.server.Serve(lis)
}
```

以 `addr` 直接启动服务器，使用 `Run` 方法：
```golang
// Run create a tcp listener and start goroutine for serving each incoming request.
// Run will return a non-nil error unless Stop or GracefulStop is called.
func (s *Server) Run(addr string) error {
	lis, err := net.Listen("tcp", addr)
	if err != nil {
		err = errors.WithStack(err)
		log.Error("failed to listen: %v", err)
		return err
	}
	reflection.Register(s.server)
	return s.Serve(lis)
}
```

关闭服务器：调用 `server.GracefulStop()` 或者 `s.server.Stop()` 来结束服务端运行：
```golang
// Shutdown stops the server gracefully. It stops the server from
// accepting new connections and RPCs and blocks until all the pending RPCs are
// finished or the context deadline is reached.
func (s *Server) Shutdown(ctx context.Context) (err error) {
	ch := make(chan struct{})
	go func() {
		s.server.GracefulStop()
		close(ch)
	}()
	select {
	case <-ctx.Done():
		s.server.Stop()
		err = ctx.Err()
	case <-ch:
	}
	return
}
```

####	Server的拦截器操作

```golang
// handle return a new unary server interceptor for OpenTracing\Logging\LinkTimeout.
func (s *Server) handle() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		var (
			cancel func()
			addr   string
		)
		s.mutex.RLock()
		conf := s.conf
		s.mutex.RUnlock()
		// get derived timeout from grpc context,
		// compare with the warden configured,
		// and use the minimum one
		timeout := time.Duration(conf.Timeout)
		if dl, ok := ctx.Deadline(); ok {
			ctimeout := time.Until(dl)
			if ctimeout-time.Millisecond*20 > 0 {
				ctimeout = ctimeout - time.Millisecond*20
			}
			if timeout > ctimeout {
				timeout = ctimeout
			}
		}
		ctx, cancel = context.WithTimeout(ctx, timeout)
		defer cancel()

		// get grpc metadata(trace & remote_ip & color)
		var t trace.Trace
		cmd := nmd.MD{}
		if gmd, ok := metadata.FromIncomingContext(ctx); ok {
			t, _ = trace.Extract(trace.GRPCFormat, gmd)
			for key, vals := range gmd {
				if nmd.IsIncomingKey(key) {
					cmd[key] = vals[0]
				}
			}
		}
		if t == nil {
			t = trace.New(args.FullMethod)
		} else {
			t.SetTitle(args.FullMethod)
		}

		if pr, ok := peer.FromContext(ctx); ok {
			addr = pr.Addr.String()
			t.SetTag(trace.String(trace.TagAddress, addr))
		}
		defer t.Finish(&err)

		// use common meta data context instead of grpc context
		ctx = nmd.NewContext(ctx, cmd)
		ctx = trace.NewContext(ctx, t)

		resp, err = handler(ctx, req)
		return resp, status.FromError(err).Err()
	}
}


// interceptor is a single interceptor out of a chain of many interceptors.
// Execution is done in left-to-right order, including passing of context.
// For example ChainUnaryServer(one, two, three) will execute one before two before three, and three
// will see context changes of one and two.
func (s *Server) interceptor(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	var (
		i     int
		chain grpc.UnaryHandler
	)

	n := len(s.handlers)
	if n == 0 {
		return handler(ctx, req)
	}

	chain = func(ic context.Context, ir interface{}) (interface{}, error) {
		if i == n-1 {
			return handler(ic, ir)
		}
		i++
		return s.handlers[i](ic, ir, args, chain)
	}

	return s.handlers[0](ctx, req, args, chain)
}



// Use attachs a global inteceptor to the server.
// For example, this is the right place for a rate limiter or error management inteceptor.
func (s *Server) Use(handlers ...grpc.UnaryServerInterceptor) *Server {
	finalSize := len(s.handlers) + len(handlers)
	if finalSize >= int(_abortIndex) {
		panic("warden: server use too many handlers")
	}
	mergedHandlers := make([]grpc.UnaryServerInterceptor, finalSize)
	copy(mergedHandlers, s.handlers)
	copy(mergedHandlers[len(s.handlers):], handlers)
	s.handlers = mergedHandlers
	return s
}

```

##  服务端调用实例
这里给出一个使用 Etcd 作为服务发现的服务端实例应用：
```golang
import (
        "github.com/bilibili/kratos/pkg/ecode"
        "github.com/bilibili/kratos/pkg/log"
        "github.com/bilibili/kratos/pkg/naming"
        "github.com/bilibili/kratos/pkg/naming/etcd"
        "github.com/bilibili/kratos/pkg/net/rpc/warden"
)

var Gaddr = flag.String("addr", "127.0.0.1:8081", "listen addr")
var Ghostid = flag.String("host", "h1", "service uniq id")

type helloServer struct {
        addr string
}

// AppID your appid, ensure unique.
const AppID = "demo.service" // NOTE: example

func (s *helloServer) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
        if in.Name == "err_detail_test" {
                err, _ := ecode.Error(ecode.AccessDenied, "AccessDenied").WithDetails(&pb.HelloReply{Success: true, Message: "this is test detail"})                return nil, err
        }
        return &pb.HelloReply{Message: fmt.Sprintf("hello %s from %s", in.Name, s.addr)}, nil
}

func (s *helloServer) StreamHello(ss pb.Greeter_StreamHelloServer) error {
        for i := 0; i < 3; i++ {
                in, err := ss.Recv()
                if err == io.EOF {
                        return nil
                }
                if err != nil {
                        return err
                }
                ret := &pb.HelloReply{Message: "Hello" + in.Name, Success: true}
                err = ss.Send(ret)
                if err != nil {
                        return err
                }
        }
        return nil
}

func runServer(addr, svcid string) *warden.Server {
        server := warden.NewServer(&warden.ServerConfig{
                // 服务端每个请求的默认超时时间
                Timeout: xtime.Duration(time.Second),
        })

        //start warden service registry
        config := &clientv3.Config{
                Endpoints:   []string{"127.0.0.1:2379"},
                DialTimeout: time.Second * 5,
                DialOptions: []grpc.DialOption{grpc.WithBlock()},
        }
        etcd_builder, err := etcd.New(config)

        if err != nil {
                fmt.Println(err)
                return nil
        }

        //kratos-etcd-key: ratos_etcd/app1/h1
        var localaddr []string
        localaddr = append(localaddr, fmt.Sprintf("grpc://%s", addr))
        _, err = etcd_builder.Register(context.Background(), &naming.Instance{
                AppID:    "app1",
                Hostname: svcid,
                Zone:     "zone01",
                Addrs:    localaddr,
        })

        // 使用拦截器（优雅）
        //server.Use(middleware())
        server.Use(middleware()).Use(middleware()).Use(stats())
        pb.RegisterGreeterServer(server.Server(), &helloServer{addr: addr})
        go func() {
                err := server.Run(addr)
                if err != nil {
                        panic("run server failed!" + err.Error())
                }
        }()
        return server
}

func main() {
        //./server_with_etcd -addr=127.0.0.1:8082 -host=h2
        //./server_with_etcd -addr=127.0.0.1:8081 -host=h1
        //./server_with_etcd -addr=127.0.0.1:8083 -host=h3
        log.Init(&log.Config{Stdout: true})
        flag.Parse()
        server := runServer(*Gaddr, *Ghostid)
        signalHandler(server)
}

// 类似于中间件
func middleware() grpc.UnaryServerInterceptor {
        return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
                // 记录调用方法
                log.Info("method:%s", info.FullMethod)
                //call chain
                resp, err = handler(ctx, req)
                return
        }
}

func stats() grpc.UnaryServerInterceptor {
        return func(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
                resp, err = handler(ctx, req)
                trailer := metadata.Pairs([]string{"serverinfo", "enjoy"}...)
                // 每次 rpc 请求时，放在 tailer，上报至 discovery
                grpc.SetTrailer(ctx, trailer)
                return
        }
}

func signalHandler(s *warden.Server) {
        var (
                ch = make(chan os.Signal, 1)
        )
        signal.Notify(ch, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT)
        for {
                si := <-ch
                switch si {
                case syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT:
                        log.Info("get a signal %s, stop the consume process", si.String())
                        ctx, cancel := context.WithTimeout(context.Background(), time.Second*3)
                        defer cancel()
                        //gracefully shutdown with timeout
                        s.Shutdown(ctx)
                        return
                case syscall.SIGHUP:
                default:
                        return
                }
        }
}
```

##  参考
-   [warden-quickstart.md](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/warden-quickstart.md)