---
layout:     post
title:      gRPC 中的 Metadata
subtitle:
date:       2020-02-22
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - gRPC
    - Metadata
---

##  0x00    前言
写这篇文章的初衷是，在研究 Opentracing 中，出现了大量以 Metadata 的代码，特此总结下。

> **gRPC 的 Metadata 简单理解，就是 Http 的 Header 中的 key-value 对 **

Metadata 是以 key-value 的形式存储数据的，其中 key 是 string 类型，而 value 是 []string，即一个字符串数组类型。metadata 使得 client 和 server 能够为对方提供关于本次调用的一些信息，就像一次 http 请求的 RequestHeader 和 ResponseHeader 一样。http 中 header 的生命周周期是一次 http 请求，那么 metadata 的生命周期就是一次 RPC 调用。

##	0x01	结构 && 创建

####	结构

metadata 其实就是一个 map，注意 metadata 的 value 是一个 slice，意味着我们可以对同一个 key 添加多个 value。
```go
type MD map[string][]string
```
Opentracing 中的 MDReaderWriter 是如下封装的，MDReaderWriter 本质就是 Metadata
```go
//MDReaderWriter metadata Reader and Writer
type MDReaderWriter struct {
	metadata.MD
}
```

####	Metadata 创建
值得关注的是：metadata 中 key 是不区分大小写的，也就是说 key1 和 KEY1 是同一个 key，这对于下面方法都是一样的。

-	New：
```go
md := metadata.New(map[string]string{"key1":"value1","key2":"value2"})
```
-	Pair（相同的 key 自动合并）：
```
md := metadata.Pairs(
    "key1", "value1",
    "key1", "value1.2", // "key1" will have map value []string{"value1", "value1.2"}
    "key2", "value2",
)
```


#### 存储二进制数据
在 metadata 中，key 永远是 string 类型，但是 value 可以是 string 也可以是二进制数据。为了在 metadata 中存储二进制数据，我们仅仅需要在 key 的后面加上一个 - bin 后缀。具有 - bin 后缀的 key 所对应的 value 在创建 metadata 时会被编码（base64），收到的时候会被解码：
```go
md := metadata.Pairs(
    "key", "string value",
    "key-bin", string([]byte{96, 102}),
)
```

##  0x02	客户端处理
在 Opentracing 的分析中，我们之前是用 StartSpanFromContext 来模拟从 context 中启动一个子 span，但是 StartSpanFromContext 或者 SpanFromContext 只能在同一个服务内使用，grpc 中 client 的 context 和 server 的 context 并不是同一个 context，无法使用这两个函数。（参考 https://github.com/grpc/grpc-go/issues/130）
如果想通过 grpc 的 context 传递 span 的信息，就需要使用 grpc 的 metadata 来传递（一个简单的例子：https://medium.com/@harlow/grpc-context-for-client-server-metadata-91cec8729424）。
同时 grpc-client 端提供了 Inject 函数，可以将 span 的 context 信息注入到 carrier 中，再将 carrier 写入到 metadata 中，即可完成 span 信息的传递。

####  发送 Metadata
贴一个客户端发送的例子：
![image](https://s2.ax1x.com/2020/02/26/3aljER.jpg)

```go

var timestampFormat = time.StampNano // "Jan _2 15:04:05.000"

client := pb.NewGreeterClient(conn)
// 生成 metadata 数据
md := metadata.Pairs("timestamp", time.Now().Format(timestampFormat))
ctx := metadata.NewOutgoingContext(context.Background(), md)

resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "hello, world"})
if err == nil {
        fmt.Printf("Reply is %s\n", resp.Message)
} else {
        fmt.Printf("call server error:%s\n", err)
}
```

####	两个方法：`AppendToOutgoingContext` 和 `NewOutgoingContext`
-	`NewOutgoingContext`：将新创建的 Metadata 添加到 context 中，这样会 ** 覆盖 ** 掉原来已有的 metadata
-	`AppendToOutgoingContext`：可以将 key-value 对添加到已有的 context 中。如果对应的 context 没有 metadata，那么就会创建一个；如果已有 metadata 了，那么就将数据添加到原来的 metadata（推荐使用 `AppendToOutgoingContext`，PS：在 Interceptor 中，常常需要给 Metadata 添加 key-value 对）

NewOutgoingContext 方法：
```go
// create a new context with some metadata
md := metadata.Pairs("k1", "v1", "k1", "v2", "k2", "v3")
ctx := metadata.NewOutgoingContext(context.Background(), md)

// later, add some more metadata to the context (e.g. in an interceptor)
md, _ := metadata.FromOutgoingContext(ctx)
newMD := metadata.Pairs("k3", "v3")
ctx = metadata.NewContext(ctx, metadata.Join(metadata.New(send), newMD))

// make unary RPC
response, err := client.SomeRPC(ctx, someRequest)

// or make streaming RPC
stream, err := client.SomeStreamingRPC(ctx)
```

AppendToOutgoingContext 方法：
```go
// create a new context with some metadata
ctx := metadata.AppendToOutgoingContext(ctx, "k1", "v1", "k1", "v2", "k2", "v3")

// later, add some more metadata to the context (e.g. in an interceptor)
ctx := metadata.AppendToOutgoingContext(ctx, "k3", "v4")	
// 在拦截器中，常常需要给 Metadata 添加 key-value 对

// make unary RPC
response, err := client.SomeRPC(ctx, someRequest)

// or make streaming RPC
stream, err := client.SomeStreamingRPC(ctx)
```

我们简单看下 `NewOutgoingContext` 方法做了啥事情 [源码在此](https://github.com/grpc/grpc-go/tree/master/metadata/metadata.go#L147)，其实就是调用了 `context.WithValue` 方法，生成了一个子 context 而已，这个子 context 中包含了传入的 Metadata。
```go
// NewOutgoingContext creates a new context with outgoing md attached. If used
// in conjunction with AppendToOutgoingContext, NewOutgoingContext will
// overwrite any previously-appended metadata.
func NewOutgoingContext(ctx context.Context, md MD) context.Context {
	return context.WithValue(ctx, mdOutgoingKey{}, rawMD{md: md})
}

```

####	接收 Metadata
客户端如何接收 Metadata？答案是 `grpc.Header()` 和 `grpc.Trailer()`，客户端可以接收的 Metadata 只有 header 和 trailer。此外，针对 Unary Call 和 Streaming 两种 RPC 类型，接收 metadata 的方式也不同。

1.	UnaryCall，使用 `grpc.Header()` 和 `grpc.Trailer()` 方法来接收 Metadata

```go
var header, trailer metadata.MD // variable to store header and trailer
r, err := client.SomeRPCMethod(
    ctx,
    someRequest,
    grpc.Header(&header),    // will retrieve header
    grpc.Trailer(&trailer),  // will retrieve trailer
)

// do something with header and trailer
```

2.	Streaming Call，Server/Client/Bidirectional streaming RPC，相应的 Header 和 Trailer 可以通过调用返回的 ClientStream 接口的 Header() 和 Trailer() 方法接收

```go
stream, err := client.SomeStreamingRPC(ctx)

// retrieve header
header, err := stream.Header()

// retrieve trailer
trailer := stream.Trailer()
```

##	0x03	服务端处理
服务端处理 Metadata 的方法和客户端有些细微的区别。先给一个Server的例子：
```go
// server is used to implement helloworld.GreeterServer.
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	// 服务端尝试接收metadata数据，通过FromIncomingContext接收
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		fmt.Printf("get metadata error")
	}else{
		fmt.Println(md)
	}
	if t, ok := md["timestamp"]; ok {
		fmt.Printf("timestamp from metadata:\n")
		for i, e := range t {
			fmt.Printf(" %d. %s\n", i, e)
		}
	}
	//fmt.Printf("%v: Receive is %s\n", time.Now(), in.Name)
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}
```

####	Server 接收Metedata
服务器需要在 RPC 调用中的 context 中获取客户端发送的 metadata。如果是一个普通的 RPC 调用，那么就可以直接用 context；如果是一个 Streaming 调用，服务器需要从相应的 stream 里获取 context，然后获取 metadata。
```go
//Unary Call
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    // do something with metadata
}

//Streaming Call
func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
    md, ok := metadata.FromIncomingContext(stream.Context()) // get context from stream
    // do something with metadata
}
```

####	Server 发送Metedata
一般而言，Server 发送的环节，一般在 RPC 请求处理完成时。客户端可以接收的 metadata 只有 header 和 trailer，因此 server 也只能发送 header 和 trailer。

1.	UnaryCall
服务器直接通过 `grpc.setHeader()` 和 `grpc.SetTrailer()` 向 client 发送 header 和 trailer。这两个方法的第一个参数都是 context：
```go
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    // create and send header
    header := metadata.Pairs("header-key", "val")
    grpc.SendHeader(ctx, header)
    // create and set trailer
    trailer := metadata.Pairs("trailer-key", "val")
    grpc.SetTrailer(ctx, trailer)
}
```

2.	StreamingCall
在流式 RPC 方法中，可以使用 `stream.SendHeader()` 和 `stream.SetTrailer()` 方法，如下：
```go
func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
    // create and send header
    header := metadata.Pairs("header-key", "val")
    stream.SendHeader(header)
    // create and set trailer
    trailer := metadata.Pairs("trailer-key", "val")
    stream.SetTrailer(trailer)
}
```

##	0x04	参考
-	[Dive into gRPC(6)：metadata](https://www.jianshu.com/p/863dad87d16f)