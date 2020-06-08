---
layout:     post
title:      Kratos 源码分析（1-2）：Kratos 中的 Metadata 元数据
subtitle:   一种全局变量的存储方式
date:       2020-06-10
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
看看下面这个例子，将 `map[string]interface{}` 定义成一个类型，这样直接 `newmap:=MD{}` 生成的对象就是一个 map（虽然这种语法看起来很奇怪）。理解此用法是分析Metadata的基础。
```golang
type mdKey struct{}

type MD map[string]interface{}

func main(){
        fmt.Println(mdKey{})        //是一个空struct
        newmap:=MD{}
        newmap["1"]=1
        newmap["2"]=2
        fmt.Println(newmap,len(newmap))
}
```

该例子输出的结果是：
```bash
{}
map[1:1 2:2] 2
```


##  0x01    Context之传值
利用Context可以方便的进行Value传递和向上查找（虽然不推荐这么做），一棵使用了Withvalue的Context-Tree可能像下面这样：

```golang
func main() {
        ctx := context.Background()
        process(ctx)

        //左子树
        lctx := context.WithValue(ctx, "key1", "val1")
        process(lctx)
        lctx = context.WithValue(lctx, "key2", "val2")
        process(lctx)
        lctx = context.WithValue(lctx, "key3", "val3")
        process(lctx)

        //右子树
        rctx:=context.WithValue(lctx, "key4", "val4")
        process(rctx)
        rctx =context.WithValue(rctx, "key5", "val5")
        process(rctx)
}

func process(ctx context.Context) {
        traceId, ok := ctx.Value("traceid1").(string)
        if ok {
                fmt.Printf("find trace_id=%s\n", traceId)
        } else {
                fmt.Printf("no trace_id\n")
        }
}
```

![img](https://wx2.sbimg.cn/2020/06/08/golang-context-withvalue.png)


##      0x02



##      参考
