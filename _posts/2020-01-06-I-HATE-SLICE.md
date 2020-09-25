---
layout:     post
title:      Golang 中的 Slice 那些事
subtitle:	Slice 踩坑
date:       2020-01-06
author:     pandaychen
header-img:	
tags:
	- Golang
---

> `Golang` 是宣扬实用主义的语言，很多时候都把 `C` 中的最佳实践直接规定成语法了。Slice 就是其一，简单但是及易踩坑

> Slice 第一法则：防止共享数据

> `Golang` 里面所有的类型都是值类型，之所以有些用起来像引用，是在于该类型内部是用指针实现的，但是其本质就是包含指针的结构体

##	0x00	Slice 与 Array
Slice 与 Array 的区别在于，Slice 是 <font color="#dd0000"> 指针 </font>，Array 存储实体

####	Array 类型
Array 是固定长度的（数组），使用前必须确定数组长度
-	将一个数组赋值给另外一个数组，那么，实际上是整个数组拷贝了一份
-	如果 golang 中的数组作为函数的参数，那么实际传递的参数是一份数组的拷贝，而不是数组的指针
-	array 的长度也是 Type 的一部分，`[10]int` 和 `[20]int` 是不一样的

####	Slice 类型
Slice 是一个动态的指向数组切片的指针，Slice 是不定长的，总是指向底层的数组 Array 的数据结构


##	0x02 Slice 的 "坑"
Google 搜索 <font color="#dd0000">golang Slice 的坑 </font>，会有非常多的文章，如果不清楚 Slice 底层的实现原理，在使用 Slice 就会出现一些无法理解的结果。

先看几个例子：
####	例子 1
```golang
func main() {
        s := make([]int, 0)
        s = append(s, 7)
        s = append(s, 9)
        x := append(s, 11)
        y := append(s, 12)
        fmt.Println(s, x, y)
}
//[7 9] [7 9 11] [7 9 12]
```

####	例子 2

```golang
func main() {
        s := make([]int, 2)
        s = append(s, 5)
        x := append(s, 11)
        y := append(s, 12)
        fmt.Println(s, x, y)
}
//[0 0 5] [0 0 5 12] [0 0 5 12]
```

####	例子 3

```golang
func main() {
        s := []int{5}
        //s := make([]int, 0)
        s = append(s, 7)
        s = append(s, 9)
        x := append(s, 11)
        y := append(s, 12)
        fmt.Println(s, x, y)
}
//[5 7 9] [5 7 9 12] [5 7 9 12]
```

####	例子 4

看起来'正常'的测试代码：
```golang
func main() {
        s := []int{5, 7, 9}
        x := append(s, 11)
        y := append(s, 12)
        fmt.Println(s, x, y)
}
//[5 7 9] [5 7 9 11] [5 7 9 12]
```

##	0x03	Slice 技巧

本文翻译自SliceTricks。我会追加一些我的理解。官方给出的例子代码很漂亮，建议大家多看看，尤其是利用多个切片共享底层数组的功能就地操作很有意思。
1、Append another Slice：`append`支持两个参数，第一个是被追加的 slice，第二个参数是追加到后面的数据。第二个参数是变长参数，可以传多值：

```golang
a = append(a, b...)
```

2、copy复制
`copy`方法把数据 a 复制到 b 中。它有个坑，复制数据的长度取决于 b 的当前长度，如果 b 没有初始化，那么并不会发生复制操作。所以复制的第一行需要初始化长度。

复制还有一种替代方案，利用append的多值情况来追加。但是这样会有一个问题，追加的时候是追加到[]T(nil)里面，默认初始长度是0，每追加一个元素需要检测当前长度是否满足，如果不满足就要扩容，每次扩容扩当前容量的一倍（详细原理可以查看 slice 的内部实现）。这么操作的话如果 a 长度是3，第一种方法复制出来长度和容量都是3，而第二种方法长度是3，容量却是4。如果只是单纯复制我推荐第一种。
```golang
b = make([]T, len(a))
copy(b, a)
// or
b = append([]T(nil), a...)
```

例子如下：
```golang
func main() {
        a := []int{1, 2, 3, 4}
        b := make([]int, len(a))
        copy(b, a)
        fmt.Println(a, b, &a[0], &b[0])
}
//[1 2 3 4] [1 2 3 4] 0xc000014320 0xc000014340
```

注意，当复制的目标`b`未初始化时，则不会复制：
```golang
func main() {
        a := []int{1, 2, 3, 4}
        var b []int
        copy(b, a)
        fmt.Println(a, b, &a[0])
}
//[1 2 3 4] [] 0xc000014320
```

3、Cut：Slice拼接（去掉某个连续位置的元素）：把 `[i, j)`中间的元素剪切掉。Slice 的切片也遵循<font color="#dd0000">前开后闭</font>原则。
```golang
//a = append(a[:i], a[j:]...)
func main() {
        a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}
        a = append(a[:2], a[4:]...)
        fmt.Println(a)
}
//[1 2 5 6 7 8 9]
```


##	0x04	参考
-	[对 Go 的 Slice 进行 Append 的一个坑](http://sharecore.net/post/%E5%AF%B9go%E7%9A%84slice%E8%BF%9B%E8%A1%8Cappend%E7%9A%84%E4%B8%80%E4%B8%AA%E5%9D%91/)
-	[Slice 小技巧](https://blog.cyeam.com/golang/2018/06/18/slicetricks)
-	[Go 语言的 slice 为啥有这样的奇怪问题呢？ - 达达的回答 - 知乎](https://www.zhihu.com/question/27161493/answer/35485751)