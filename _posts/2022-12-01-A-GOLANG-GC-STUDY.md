---
layout:     post
title:      Golang 中的 GC 小结
subtitle:   gc 场景及原理总结
date:       2022-12-01
author:     pandaychen
catalog:    true
tags:
    - GC
    - Golang
---


##  0x00    前言
本文的版本基于 `go version go1.21.3 linux/amd64`

##  0x01    gc 典型触发场景汇总
Golang 的 GC 会自动回收所有不可达对象（无引用链可访问的对象）。经典场景包括局部变量、重新赋值的变量、循环临时变量等。仅在需要优化内存敏感代码时，才考虑手动干预（如手动置 `nil` 或使用对象池）

1、局部变量（函数内创建的对象），函数内的局部变量在函数返回后失去所有引用，GC 会标记为不可达对象并回收

```GO
func createMap() {
    m := make(map[string]int) // 局部变量
    m["key"] = 1
    // 函数结束后，m 超出作用域，不再被引用，会被 GC 回收
}
```

2、被重新赋值的变量：变量被重新赋值后，原对象失去唯一引用，GC 会回收旧内存

```GO
func main() { 
    data := make([]byte, 100) // 分配 100 字节的切片
    data = make([]byte, 200)  // 重新赋值后，原 100 字节切片失去引用，会被回收
}
```

```GO
type A struct{
        Map map[int]int
}

func main(){
        var a A
        m1:=make(map[int]int)
        m1[1]=1
        fmt.Println(m1)
        a.Map=m1    

        m2:=make(map[int]int)
        m2[2]=2
        a.Map=m2        //m1失去了引用，会被回收

        fmt.Println(a.Map)
}
```

3、循环中的临时变量：每次循环迭代的 `temp` 都是新对象，迭代结束后失去引用，GC 逐步回收

```GO
func process() {
    for i := 0; i < 1000; i++ {
        temp := make([]int, 1000) // 每次循环创建临时切片
        // 循环结束后，temp 超出作用域，会被回收
    }
}
```

4、channel 操作后的对象：通道传递的对象引用计数归零后（如 `received` 不再使用），GC 回收原数据

```GO
func channelExample() {
    ch := make(chan *int)
    go func() {
        x := 42
        ch <- &x // 发送 x 的指针到通道
    }()
    received := <-ch // 接收后，原 goroutine 中的 x 可能仍被引用
    // 若 received 后续不再使用，x 会被回收
}
```

5、结构体或切片中的元素：结构体或切片的内部元素引用被清除后，GC 会回收这些元素

```GO
type Data struct {
    slice []*int
}

func main() {
    d := Data{}
    for i := 0; i < 10; i++ {
        x := new(int) // 分配内存
        d.slice = append(d.slice, x)
    }
    d.slice = nil // 切片置空后，所有元素失去引用，会被 GC 回收
}
```

6、闭包中未捕获的外部变量：闭包只延长其捕获变量的生命周期，未捕获的变量（如 `b`）会正常回收

```GO
func closureExample() {
    a := 1
    b := 2
    go func() {
        fmt.Println(a) // 闭包捕获了 a，但未捕获 b
    }()
    // b 未被闭包引用，函数返回后会被回收
}
```

##  0x0 gc 原理

##  0x0 参考
-   [从入门到掉坑：Go 内存池 / 对象池技术介绍](https://cloud.tencent.com/developer/article/1638446)
-   [GODEBUG 之 gctrace 干货解析](https://zhuanlan.zhihu.com/p/73183820)