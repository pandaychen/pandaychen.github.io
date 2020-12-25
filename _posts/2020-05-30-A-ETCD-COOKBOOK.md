---
layout:     post
title:      Etcd 开发中的细节梳理（总结）
subtitle:   如何更优雅的使用 Etcd 实现工作项目
date:       2020-05-30
author:     pandaychen
catalog:    true
tags:
    - Etcd
---

##  0x00    前言
本文主要梳理下在使用 Etcd 进行开发中遇到的一些细节问题及解决。必读的文档如下：
-   [etcd3 API](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/api.md)


##  0x01    Etcd 的客户端
Etcd 官方提供了丰富的客户端 [实现](https://github.com/etcd-io/etcd/tree/master/client/v3/experimental/recipes)，值得参考。

####    key.go
[key.go](https://github.com/etcd-io/etcd/blob/v3.3.10/contrib/recipes/key.go)


####    客户端的选项
阅读 etcd 分布式锁 mutex 的源码时，遇到 waitDeletes 函数

##  0x02    Etcd 的 Key-Value
Etcd 的 Key-Value 结构的 PB 定义如下：
```golang
message KeyValue {
  bytes key = 1;
  int64 create_revision = 2;
  int64 mod_revision = 3;
  int64 version = 4;
  bytes value = 5;
  int64 lease = 6;
}
```

##  0x02    各种 Version 的含义

##  0x03    Etcd 的 Resp 的字段含义

##  0x04    Etcd 的 Txn
Etcd 使用 `Txn` 提供了事务处理的机制。使用 Txn 特性，可以一次性操作多条语句（在一个 `Revision` 中完成）：
```golang
func main() {
        endpoints := []string{"127.0.0.1:2379"}
        cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
        if err != nil {
                log.Fatal(err)
        }
        defer cli.Close()

        _, err = cli.Txn(context.TODO()).If().Then(
                clientv3.OpPut("key1", "value1"),
                clientv3.OpPut("key2", "value2"),
                clientv3.OpPut("key3", "value3"),
        ).Commit()
        if err != nil {
                log.Fatal(err)
        }
        resp, err := cli.Txn(context.TODO()).If().Then(
                clientv3.OpGet("key1"),
                clientv3.OpGet("key2"),
                clientv3.OpGet("key3"),
        ).Commit()
        if err != nil {
                log.Fatal(err)
        }
        for _, rp := range resp.Responses {
                for _, ev := range rp.GetResponseRange().Kvs {
                        fmt.Printf("%s->%s,create revision=%d,version=%d,mod revivison=%d\n",
                        string(ev.Key), string(ev.Value), ev.CreateRevision, ev.Version, ev.ModRevision)
                }
        }
}
```
上面输入如下，三个key具有相同的`Create Revision`值：
```javascript
key1->value1,create revision=47,version=5,mod revivison=51
key2->value2,create revision=47,version=5,mod revivison=51
key3->value3,create revision=47,version=5,mod revivison=51
```

##  参考
-   [what is different about Revision, ModRevision and Version?](https://github.com/etcd-io/etcd/issues/6518)
-   [重要文档：etcd3 API](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/api.md)
-   [etcd:recipes](https://github.com/etcd-io/etcd/tree/v3.3.10/contrib/recipes)