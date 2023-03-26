---
layout: post
title: Golang 闭包与递归：介绍与应用场景
subtitle: Golang closure 
date: 2023-03-25
author: pandaychen
catalog: true
tags:
  - Golang
  - 闭包
---

## 0x00 前言
闭包是指内层函数引用了外层函数中的变量或称为引用了自由变量的函数，其返回值也是一个函数（方法），先看例子：
```golang
func outer(x int) func(int) int {
        return func(y int) int {
                return x + y
        }
}

func main() {
        f := outer(10)
        fmt.Println(f(100))
}
```

golang的闭包有如下特点：

- 闭包能够访问外层代码中的变量
- `for range`模式与goroutine同时执行
- 所有的goroutine操作的变量都是直接操作外层代码的变量，而外层代码中的变量的值取决于循环执行的节点

##  0x01 应用场景



##  0x02  使用闭包要注意的点

####  变量传递
```GOLANG
func main() {
        var s []string = []string{
                "a",
                "b",
                "c",
        }

        var waitGroup sync.WaitGroup
        waitGroup.Add(len(s))

        for _, item := range s {
                go func(i string) {
                        fmt.Println(i)
                        waitGroup.Done()
                }(item)
        }

        waitGroup.Wait()
}
```

注意由于匿名函数可以访问函数体外部的变量，而 `for range`返回的 `val` 的值是引用的同一个内存地址的数据，所以匿名函数访问的函数体外部的 `val` 值是循环中最后输出的一个值，下面的代码是错误的：
```GOLANG
for _, item := range s {
    go func() {
        fmt.Println(item)
        waitGroup.Done()
    }()
}
```

另外一种规避方便是创建一个新变量，如下：
```golang
func main() {
        var s []string = []string{
                "a",
                "b",
                "c",
        }

        var waitGroup sync.WaitGroup
        waitGroup.Add(len(s))

        for _, item := range s {
                newitem := item
                go func() {
                        fmt.Println(newitem)
                        waitGroup.Done()
                }()
        }

        waitGroup.Wait()
}
```


## 0x0 参考
- [Golang 中关于闭包的坑](https://www.jianshu.com/p/fa21e6fada70)