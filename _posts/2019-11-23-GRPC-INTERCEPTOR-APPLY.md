---
layout:     post
title:      gRPC 之 Interceptor 实战篇
subtitle:   如何优雅的使用 gRPC 拦截器
date:       2019-11-23
author:     pandaychen
header-img:
catalog: true
tags:
    - gRPC
---

##  0x00    背景
在实现 RPC 服务时，如果想在调用 RPC 方法的前 or 后做某些（如服务器登录鉴权、Tracing、耗时统计等等）事情，比如一个典型的应用场景是，当客户端进行 RPC 请求时，先对请求中的某些字段（`Metadata`）进行验证，验证通过再执行相应的 RPC 方法。怎么实现？<br>

gRPC 提供了拦截器（Interceptor）机制，可以完成这个功能。<br>

拦截器（Interceptor） 类似于 HTTP 应用的中间件（Middleware），能够让你在真正调用 RPC 方法前，进行身份认证、日志、限流、异常捕获、参数校验等通用操作，和 Python 的装饰器（Decorator） 的作用基本相同。<br>

在实际项目开发中，可以将许多共性的功能放在拦截器的逻辑中实现，此外利用 `defer` 关键字还能够方便的实现 RPC 拦截器计时的功能。

注意：新版本的 gRPC 已经提供了内置的链式 Interceptor 实现，即 `WithChainUnaryInterceptor` 和 `WithChainStreamInterceptor`
-	[WithChainStreamInterceptor](https://pkg.go.dev/google.golang.org/grpc#WithChainStreamInterceptor)
-	[WithChainUnaryInterceptor](https://pkg.go.dev/google.golang.org/grpc#WithChainUnaryInterceptor)

##  0x01    Interceptor 分析
gRPC 中使用 `UnaryInterceptor` 来实现 Unary RPC 一元拦截器，使用 `StreamInterceptor` 来实现 Stream RPC 流式的拦截器，而且既可以在客户端进行拦截，也可以对服务器端进行拦截。

![type](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/grpc-interceptor-type2.png)

####	一元拦截器（grpc.UnaryInterceptor）
gRPC 的一元拦截器包含服务端 `UnaryServerInterceptor` 和客户端 `UnaryClientInterceptor`，这里我们分析 `UnaryServerInterceptor`:

从 `grpc.NewServer(opts...)` 注册开始，`grpc.UnaryInterceptor` 函数的唯一参数是一个 `grpc.UnaryServerInterceptor`，`UnaryInterceptor` 返回的 `ServerOption` 作为 `grpc.NewServer` 的参数。
```golang
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {
    return func(o *options) {
        if o.unaryInt != nil {
            panic("The unary server interceptor was already set and may not be reset.")
        }
        o.unaryInt = i  // 设置 UnaryServerInterceptor
    }
}
```

从上面的定义看，要实现一个一元拦截器 Interceptor，必须实现 `i UnaryServerInterceptor`，`UnaryServerInterceptor` 的作用是在服务端对于一次 RPC 调用进行拦截。<br>
下面看下 `UnaryServerInterceptor` 的定义，它是一个函数指针，当客户端进行 RPC 调用的时候，首先并不执行用户的 RPC 方法，而是先执行 `UnaryServerInterceptor` 所指的函数，随后再进入真正要执行的函数 `handler UnaryHandler`，如下：
```golang
// UnaryServerInterceptor provides a hook to intercept the execution of a unary RPC on the server. info
// contains all the information of this RPC the interceptor can operate on. And handler is the wrapper
// of the service method implementation. It is the responsibility of the interceptor to invoke handler
// to complete the RPC.
type UnaryServerInterceptor func(
            ctx context.Context,        // 请求上下文
            req interface{},            //RPC 方法的请求参数，这里为 interface{}
            info *UnaryServerInfo,      //RPC 方法的所有信息 (本次调用)
            handler UnaryHandler        //RPC 方法本身 (客户端此次实际要调用的函数)
    )(resp interface{}, err error)
```

在上面 `UnaryServerInterceptor` 的定义中，其中 `handler UnaryHandler` 是具体的 RPC 方法，会在拦截器最后调用，其定义为如下，可以看出 `UnaryHandler` 与 `UnaryServerInterceptor` 相比，少了 `info` 和 `handler` 两个参数。
```golang
type UnaryHandler func(ctx context.Context, req interface{}) (interface{}, error)
```

上面 `UnaryServerInterceptor` 的参数 `UnaryServerInfo` 的结构如下：
```golang
// UnaryServerInfo consists of various information about a unary RPC on
// server side. All per-rpc information may be mutated by the interceptor.
type UnaryServerInfo struct {
    // Server is the service implementation the user provides. This is read-only.
    //Server 是客户编写的服务器端的服务实现，这个成员是只读的
	Server interface{}
    // FullMethod is the full RPC method string, i.e., /package.service/method.
    //FullMethod 成员是要调用的方法名，这个方法名 interceptor 可以进行修改。
	FullMethod string
}
```

所以说，如果需要在服务端 RPC 方法之前或之后实现某些功能的话，Interceptor 可以这么写：
```golang
// 定义一个 grpc.UnaryServerInterceptor
var interceptor grpc.UnaryServerInterceptor

// 实现 grpc.UnaryServerInterceptor
interceptor = func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
    // 前置逻辑, 实现我们需要的通用逻辑，如对 req 中的签名进行验证，成功继续，失败返回 error
	fmt.Printf("Before RPC handling. Info: %+v", info)
	//handler 是客户端原来打算调用的方法，如果验证成功（在上面的逻辑），执行真正的方法
	resp, err := handler(ctx, req)
	fmt.Printf("After RPC handling. resp: %+v", resp)
	// 后置逻辑，如计算耗时，Metrics 上报等等
    return resp, err
}
```

相应的 gRPC 服务端的启动代码大致如下：
```golang
listener, err := net.Listen("tcp", fmt.Sprintf(":%s", port))
if err != nil {
    panic(err)
    return
}
var opts []grpc.ServerOption
// 注册我们实现的 interceptor
opts = append(opts, grpc.UnaryInterceptor(interceptor))
server := grpc.NewServer(opts...)
chatprt.RegisterHelloServer()
server.Serve(listener)
```

####	流式拦截器（grpc.StreamInterceptor）
适用于 gRPC 的流式拦截器定义，`StreamServerInterceptor` 与 `UnaryServerInterceptor` 使用基本一样，这里不过多描述。

####  	Warning
注意，在 gRPC 中，服务器本身只能设置一个 `UnaryServerInterceptor` 和 `StreamServerInterceptor`。客户端亦是如此，虽然不会报错，但是只有最后一个才起作用。那么如何配置多个拦截器呢？<br>

不难想到我们可以把拦截器串联起来，利用 Golang 的闭包 / 递归特性，将第一个位置的拦截器传给 `grpc.UnaryInterceptor`，作为选项 `grpc.ServerOption` 传入。<br>

下面我们分析下如何实现拦截器链。


##		0x02	拦截器链
基于开发的经验，不难想到，多个 Interceptor 的模式可以如下图所示去实现（经典的洋葱模式）：
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/grpc-onion.png)

设想下，如何设计链式的 Interceptor？我脑海中浮现两个方案：
-   闭包方式调用：类似 `f = interceptor1(ctx,f1);f1 = interceptor2(ctx,f2);....;fn = RPC(ctx)` 这样的方式
-   数组依次调用：将拦截器组织成 `[]interceptor` 方式，按照数组的顺序依次执行下去，完成拦截器的功能

简单来说，对于普通的一元拦截器都是在拦截处理的代码之后，再对 handler 进行了调用：
```golang
func OneInterceptor() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		defer func() {
			if err := recover(); err != nil {
				fmt.Println("fatal error:", err,string(debug.Stack()))
			}
		}()
		fmt.Println("before handler")
		// do real RPC
		res, err := handler(ctx, req)
		fmt.Println("after handler")
		return res, err
	}
}
```
那么根据以上特点，我们可以将 `handler` 继续传入拦截器，再利用 Golang 的闭包性质将拦截器打包成新的 `handler`，然后再将新的 `handler` 传入下一个拦截器，下一个拦截器继续打包成 `handler`，再依次传递下去，形成链式关系（interceptor chain）。通过这种方式可以将单个拦截器扩展为链式拦截器，实现与 HTTP 中间件数组（如 `gin`）相同的效果。

![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/grpc-interceptor-order1.png)

下面给出两个开源的实现：
####  go-grpc-interceptor 的实现
mercari 的 [go-grpc-interceptor](https://github.com/mercari/go-grpc-interceptor/tree/master/multiinterceptor) 项目给出了基于数组方式的 [实现](https://github.com/mercari/go-grpc-interceptor/blob/master/multiinterceptor/interceptor.go)，主要实现代码如下：

```golang
type multiUnaryServerInterceptor struct {
	// 存储拦截器的数组
	uints []grpc.UnaryServerInterceptor
}

func NewMultiUnaryServerInterceptor(uints ...grpc.UnaryServerInterceptor) grpc.UnaryServerInterceptor {
    // 返回 gen 方法的结果
	return (&multiUnaryServerInterceptor{uints: uints}).gen
}

func (m *multiUnaryServerInterceptor) gen(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	return m.chain(0, ctx, req, info, handler)
}

func (m *multiUnaryServerInterceptor) chain(i int, ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	if i == len(m.uints) {
		return handler(ctx, req)
	}
	return m.uints[i](ctx, req, info, func(ctx2 context.Context, req2 interface{}) (interface{}, error) {
		return m.chain(i+1, ctx2, req2, info, handler)
	})
}
```
它的调用顺序如下图所示：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/grpc-interceptor-order.png)

使用该拦截器链的方式如下，例子来源于此 [server.go](https://github.com/pandaychen/grpc_in_action/blob/master/chain_interceptor/server.go)：
```golang
import (
        "fmt"
        multiint "github.com/mercari/go-grpc-interceptor/multiinterceptor"
        "golang.org/x/net/context"
        "google.golang.org/grpc"
)

func fooUnaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	fmt.Println("foo")
	newctx := context.WithValue(ctx, "foo_key", "foo_value")
	rsp, err := handler(newctx, req)
	fmt.Printf("after foo,rsp=%v,err=%v\n", rsp, err)
	return rsp, err
}

func barUnaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	fmt.Println("bar")
	newctx := context.WithValue(ctx, "bar_key", "bar_value")
	rsp, err := handler(newctx, req)
	fmt.Printf("after bar,rsp=%v,err=%v\n", rsp, err)
	return rsp, err
}

func main() {
        uIntOpt := grpc.UnaryInterceptor(multiint.NewMultiUnaryServerInterceptor(
                fooUnaryInterceptor,
                barUnaryInterceptor,
        ))
        grpc.NewServer(uIntOpt)
}
```
上面这个例子输出的结果如下，和我们预期一致（先进后出）：
```javascript
foo
bar
context.Background.WithCancel.WithValue(type peer.peerKey, val <not Stringer>).WithValue(type metadata.mdIncomingKey, val <not Stringer>).WithValue(type grpc.streamKey, val <not Stringer>).WithValue(type string, val foo_value).WithValue(type string, val bar_value)
foo_value bar_value
after bar,rsp=message:"Hello pandaychen." ,err=<nil>
after foo,rsp=message:"Hello pandaychen." ,err=<nil>
```

####	go-grpc-middleware 的实现
首先，看下调用方式，和第一种方式大同小异：
```golang
import "github.com/grpc-ecosystem/go-grpc-middleware"

myServer := grpc.NewServer(
    grpc.StreamInterceptor(grpc_middleware.ChainStreamServer(
        ...
    )),
    grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
       ...
    )),
)
```

再看看 `grpc_middleware.ChainUnaryServer` 方法的实现，在 [chain.go](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/chain.go) 中：
```golang
// ChainUnaryServer creates a single interceptor out of a chain of many interceptors.
//
// Execution is done in left-to-right order, including passing of context.
// For example ChainUnaryServer(one, two, three) will execute one before two before three, and three
// will see context changes of one and two.
func ChainUnaryServer(interceptors ...grpc.UnaryServerInterceptor) grpc.UnaryServerInterceptor {
	n := len(interceptors)

    //return 里面包含了一个完整的回调
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
        // 定义 chainer 调用
		chainer := func(currentInter grpc.UnaryServerInterceptor, currentHandler grpc.UnaryHandler) grpc.UnaryHandler {
			return func(currentCtx context.Context, currentReq interface{}) (interface{}, error) {
				return currentInter(currentCtx, currentReq, info, currentHandler)
			}
		}

		chainedHandler := handler
		for i := n - 1; i >= 0; i-- {
			chainedHandler = chainer(interceptors[i], chainedHandler)
		}

		return chainedHandler(ctx, req)
	}
}
```
从实现上看，它采用了回调函数调用的方式，调用链为 `chainedHandler[n-1]`-->`chainedHandler[n-2]`-->`...`-->`chainedHandler[0]`-->`handler`

##	0x03  拦截器链的执行顺序
拦截器的执行顺序与请求处理和响应处理的顺序相反，
```javescript
const promiseClient = new MyServicePromiseClient(
    host, creds, {'unaryInterceptors': [interceptor1, interceptor2, interceptor3]});


const client = new MyServiceClient(
    host, creds, {'streamInterceptors': [interceptor1, interceptor2, interceptor3]});
```
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/grpc-web-interceptors.png)
我们从下面的例子来看下这个执行流程：

####    服务端
```golang
func main() {
	flag.Parse()

	lis, err := net.Listen("tcp", *port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer(grpc.StreamInterceptor(StreamServerInterceptor),
		grpc.UnaryInterceptor(UnaryServerInterceptor))
	pb.RegisterGreeterServer(s, &server{})

	reflection.Register(s)
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

func UnaryServerInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	log.Printf("before handling. Info: %+v", info)
	resp, err := handler(ctx, req)
	log.Printf("after handling. resp: %+v", resp)
	return resp, err
}

// StreamServerInterceptor is a gRPC server-side interceptor that provides Prometheus monitoring for Streaming RPCs.
func StreamServerInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
	log.Printf("before handling. Info: %+v", info)
	err := handler(srv, ss)
	log.Printf("after handling. err: %v", err)
	return err
}
```

####    客户端
```golang

func main() {
	flag.Parse()

	// 连接服务器
	conn, err := grpc.Dial(*address, grpc.WithInsecure(), grpc.WithUnaryInterceptor(UnaryClientInterceptor),
		grpc.WithStreamInterceptor(StreamClientInterceptor))

	if err != nil {
		log.Fatalf("faild to connect: %v", err)
	}
	defer conn.Close()

	c := pb.NewGreeterClient(conn)

	r, err := c.SayHello(context.Background(), &pb.HelloRequest{Name: *name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.Message)
}

func UnaryClientInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
	log.Printf("before invoker. method: %+v, request:%+v", method, req)
	err := invoker(ctx, method, req, reply, cc, opts...)
	log.Printf("after invoker. reply: %+v", reply)
	return err
}

func StreamClientInterceptor(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
	log.Printf("before invoker. method: %+v, StreamDesc:%+v", method, desc)
	clientStream, err := streamer(ctx, desc, cc, method, opts...)
	log.Printf("before invoker. method: %+v", method)
	return clientStream, err
}
```

##	0x04	拦截器链路径上对错误的处理
通常，一个服务的调用流程如下，其中某个服务采用了 gRPC 的拦截器链实现，包含了服务端，服务端又内置了客户端去调用其他的 gRPC 服务，那么这个时候需要注意对错误的传递及处理：
![kraots-cs-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/kratos/kratos-cs-flow-interceptor.png)

![kratos-cs-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/kratos/blog-kratos-2.png)

以服务端的拦截器执行顺序（真正执行了 `handler` 方法）为例：
-	RPC 逻辑：通过运行业务逻辑或者是 RPC 客户端获取结果。获取 `resp3`，以及 `err3`，返回到拦截器 2 的 `handler`
-	拦截器 2：获取到 `resp3` 和 `err3`，同时视情况是否需要对 `resp3` 和 `err3` 进行加工 / 记录或者处理（比如熔断、限流、日志、metrics 等），处理完得到 `resp2` 及 `err2` 返回给拦截器 1 的 `handler`
-	拦截器 1：同样，获取到 `resp2` 和 `err2` 后，加工生成 `resp1` 和 `err1` 返回给客户端

客户端收到服务端的 `resp1` 和 `err1` 后，假设客户端也有拦截器链，那么按照相同的方式，将 `resp1` 和 `err1` 在各个拦截器中流转并做相应的处理，处理完成得到最终的结果。

所以，这里各个拦截器的放置顺序就显得非常重要，比如为啥 `panic` 拦截器要放在第一个位置？如果放在后面的位置，那么假设前面拦截器运行过程中发生了 `panic`，就无法捕获到异常了。同理，需要对错误进行处理的拦截器，如果放置的位置不合适，获取不到相关的错误，那么该拦截器就无意义了（比如对于客户端的 breaker 熔断 / 超时重试 / 参数检查拦截器的位置，一般而言，是按照先检查输入参数 -> 熔断拦截器 -> 超时重试的顺序，因为熔断需要检查超时错误，但是对于参数校验错误就不关心）

##  0x05	参考
-   [gRPC 之 Interceptors](https://www.do1618.com/archives/1467/grpc-%E4%B9%8B-interceptors/)
-   [gRPC server insterceptor for golang](https://github.com/mercari/go-grpc-interceptor)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权