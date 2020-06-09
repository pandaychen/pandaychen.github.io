---
layout:     post
title:      gRPC 知识点备忘录（持续更新）
subtitle:
date:       2020-04-16
author:     pandaychen
catalog:    true
tags:
    - gRPC
---

##  0x00    前言
本篇文章主要记录在使用 gRPC 中的一些收获及容易混淆的知识点。

##  0x01    原理篇

##  0x02    应用篇

####    关于 context 的问题
1.  RPC 方法的第一个参数 `ctx context.Context` 中，到底存储了什么？以下面的 `SayHello` 为例：
```golang
// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
        // 服务端尝试接收 metadata 数据
        fmt.Println("ctx=",ctx)
        md, ok := metadata.FromIncomingContext(ctx)
        if !ok {
                fmt.Printf("get metadata error")
        }else{
                fmt.Println("metadata=",md)
        }
        if t, ok := md["timestamp"]; ok {
                fmt.Printf("timestamp from metadata:\n")
                for i, e := range t {
                        fmt.Printf("%d. %s\n", i, e)
                }
        }
        return &pb.HelloReply{Message: "Hello" + in.Name}, nil
}
```

这段代码输出的结果如下：
```bash
ctx= context.Background.WithCancel.WithValue(peer.peerKey{}, &peer.Peer{Addr:(*net.TCPAddr)(0xc0000fc1e0), AuthInfo:credentials.AuthInfo(nil)}).WithValue(metadata.mdIncomingKey{}, metadata.MD{":authority":[]string{"127.0.0.1:50001"}, "content-type":[]string{"application/grpc"}, "timestamp":[]string{"Jun  9 01:35:01.069962038"}, "user-agent":[]string{"grpc-go/1.27.0-dev"}}).WithValue(grpc.streamKey{}, <stream: 0xc000142000, /proto.Greeter/SayHello>)
metadata= map[:authority:[127.0.0.1:50001] content-type:[application/grpc] timestamp:[Jun  9 01:35:01.069962038] user-agent:[grpc-go/1.27.0-dev]]
timestamp from metadata:
 0. Jun  9 01:35:01.069962038
```

我们按照 ctx 的结果来画下继承关系（注意：紫色的位置是当前打印的 ctx 所处于的位置），由于可以向上查找，所以 `metadata.FromIncomingContext(ctx)` 可以查找到上一级 `metadata.mdIncomingKey{}` 对应的 Value 值：
![image](https://wx2.sbimg.cn/2020/06/09/grpc-context-1.png)

再看看 `metadata.FromIncomingContext(ctx)` 的 [实现](https://github.com/grpc/grpc-go/blob/master/metadata/metadata.go#L169)，其中 gRPC 封装了取 `mdIncomingKey{}`Value 的接口给到用户（`mdIncomingKey{}` 开头是小写，用户无法直接调用）：
```golang
type mdIncomingKey struct{}

func FromIncomingContext(ctx context.Context) (md MD, ok bool) {
        // 似曾相识
	md, ok = ctx.Value(mdIncomingKey{}).(MD)
	return
}
```

##  0x03        参考
-   [从实践到原理，带你参透 gRPC](https://segmentfault.com/a/1190000019608421)