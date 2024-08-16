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
---

##  0x00    前言
写这篇文章的初衷是，在研究 Opentracing 中，出现了大量涉及到 [Metadata](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md) 的代码，特此总结下。

> **gRPC 的 Metadata 简单理解，就是 Http 的 Header 中的 key-value 对**

Metadata 是以 key-value 的形式存储数据的，其中 key 是 string 类型，而 value 是 `[]string`，即一个字符串数组类型。metadata 使得 client 和 server 能够为对方提供关于本次调用的一些信息，就像一次 HTTP 请求的 RequestHeader 和 ResponseHeader 一样。HTTP 中 Header 的生命周周期是一次 HTTP 请求，那么 metadata 的生命周期就是一次 RPC 调用。

##	0x01	结构 && 创建

####	结构
metadata 本质上是一个 map，注意 metadata 的 value 是一个 slice，意味着可以对同一个 key 添加多个 value（可以方便的存储多个同类型的数据）
```golang
type MD map[string][]string
```
Opentracing 中的 `MDReaderWriter` 是如下封装的，`MDReaderWriter` 本质就是 Metadata
```go
//MDReaderWriter metadata Reader and Writer
type MDReaderWriter struct {
	metadata.MD
}
```

####	Metadata 创建
值得关注的是：metadata 中 key 是不区分大小写的，也就是说 `key1` 和 `KEY1` 是同一个 key，这对于下面方法都是一样的。

1、New方法<br>
```golang
md := metadata.New(map[string]string{"key1":"value1","key2":"value2"})
```

2、Pair（相同的 key 自动合并）方法<br>
```golang
md := metadata.Pairs(
    "key1", "value1",
    "key1", "value1.2", // "key1" will have map value []string{"value1", "value1.2"}
    "key2", "value2",
)
```


#### 存储二进制数据
在 metadata 中，key 永远是 string 类型，但是 value 可以是 string 也可以是二进制数据。为了在 metadata 中存储二进制数据，仅仅需要在 key 的后面加上一个 `-bin` 后缀。具有 `-bin` 后缀的 key 所对应的 value 在创建 metadata 时会被编码（base64），收到的时候会被解码：
```golang
md := metadata.Pairs(
    "key", "string value",
    "key-bin", string([]byte{96, 102}),
)
```

##  0x02	客户端处理

####  发送 Metadata
贴一个客户端发送的例子：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/grpc-metadata-send.png)

```golang
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
-	`NewOutgoingContext`：将新创建的 Metadata 添加到 context 中，这样会 **覆盖** 掉原来已有的 metadata
-	`AppendToOutgoingContext`：可以将 key-value 对添加到已有的 context 中。如果对应的 context 没有 metadata，那么就会创建一个；如果已有 metadata 了，那么就将数据添加到原来的 metadata（推荐使用 `AppendToOutgoingContext`，PS：在 Interceptor 中，常常需要给 Metadata 添加 key-value 对）

1、`NewOutgoingContext` 方法的用例<br>
```golang
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

2、`AppendToOutgoingContext` 方法的用例<br>
```golang
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

再简单看下 `NewOutgoingContext` 方法做了啥事情 [源码在此](https://github.com/grpc/grpc-go/tree/master/metadata/metadata.go#L147)，其实就是调用了 `context.WithValue` 方法，生成了一个子 context 而已，这个子 context 中包含了传入的 Metadata。
```golang
// NewOutgoingContext creates a new context with outgoing md attached. If used
// in conjunction with AppendToOutgoingContext, NewOutgoingContext will
// overwrite any previously-appended metadata.
func NewOutgoingContext(ctx context.Context, md MD) context.Context {
	return context.WithValue(ctx, mdOutgoingKey{}, rawMD{md: md})
}
```

####	接收 Metadata（和 Server 发送对应）
客户端如何接收 Metadata？答案是 `grpc.Header()` 和 `grpc.Trailer()`，客户端可以接收的 Metadata 只有 header 和 trailer。此外，针对 Unary Call 和 Streaming 两种 RPC 类型，接收 metadata 的方式也不同。


1、UnaryCall，使用 `grpc.Header()` 和 `grpc.Trailer()` 方法来接收 Metadata<br>
```golang
var header, trailer metadata.MD // variable to store header and trailer
r, err := client.SomeRPCMethod(
    ctx,
    someRequest,
    grpc.Header(&header),    // will retrieve header
    grpc.Trailer(&trailer),  // will retrieve trailer
)
// do something with header and trailer
```

2、Streaming Call，包含 Server、Client 和 Bidirectional 三种 streaming 类型 RPC，相应的 Header 和 Trailer 可以通过调用返回的 ClientStream 接口的 `Header()` 和 `Trailer()` 方法接收<br>
```golang
stream, err := client.SomeStreamingRPC(ctx)
// retrieve header
header, err := stream.Header()
// retrieve trailer
trailer := stream.Trailer()
```

##	0x03	服务端处理
服务端处理 Metadata 的方法和客户端有些细微的区别，服务端一般作为 RPC 的 response 端，使用 `FromIncomingContext` 方法接收 Metadata。先给一个 Server 的例子：
```go
// server is used to implement helloworld.GreeterServer.
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	// 服务端尝试接收 metadata 数据，通过 FromIncomingContext 接收
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		fmt.Printf("get metadata error")
	}else{
		fmt.Println(md)
	}
	if t, ok := md["timestamp"]; ok {
		fmt.Printf("timestamp from metadata:\n")
		for i, e := range t {
			fmt.Printf("%d. %s\n", i, e)
		}
	}
	//fmt.Printf("%v: Receive is %s\n", time.Now(), in.Name)
	return &pb.HelloReply{Message: "Hello" + in.Name}, nil
}
```

####	Server 接收 Metedata
服务器需要在 RPC 调用中的 context 中获取客户端发送的 metadata。如果是一个普通的 RPC 调用，那么就可以直接用 context；如果是一个 Streaming 调用，服务器需要从相应的 stream 里获取 context，然后获取 metadata。
```golang
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

####	Server 发送 Metedata（和 Client 接收 Metadata 对应）
一般而言，Server 发送的环节，一般在 RPC 请求处理完成时。客户端可以接收的 metadata 只有 header 和 trailer，因此 server 也只能发送 header 和 trailer。

1、UnaryCall<br>
服务器直接通过 `grpc.setHeader()` 和 `grpc.SetTrailer()` 向 client 发送 header 和 trailer。这两个方法的第一个参数都是 context：
```golang
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    // create and send header
    header := metadata.Pairs("header-key", "val")
    grpc.SendHeader(ctx, header)
    // create and set trailer
    trailer := metadata.Pairs("trailer-key", "val")
    grpc.SetTrailer(ctx, trailer)
}
```

2、StreamingCall<br>
在流式 RPC 方法中，可以使用 `stream.SendHeader()` 和 `stream.SetTrailer()` 方法，如下：
```golang
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

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权