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
RPC 方法的第一个参数 `ctx context.Context` 中，到底存储了什么？
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
        //fmt.Printf("%v: Receive is %s\n", time.Now(), in.Name)
        return &pb.HelloReply{Message: "Hello" + in.Name}, nil
}
```


##  参考
