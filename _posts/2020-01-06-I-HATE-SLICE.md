---
layout:     post
title:      Golang Slice 那些事
subtitle:   Slice 最佳实践
date:       2020-01-06
author:     pandaychen
catalog:    true
tags:
    - Golang
---

> `Golang` 是宣扬实用主义的语言，很多时候都把 `C` 中的最佳实践直接规定成语法了。Slice 就是其一，简单但是及易踩坑

> Slice 第一法则：防止共享数据

> `Golang` 里面所有的类型都是值类型，之所以有些用起来像引用，是在于该类型内部是用指针实现的，但是其本质就是包含指针的结构体

##	0x00	Slice 与 Array
Slice 与 Array 的区别在于，Slice 是 <font color="#dd0000"> 指针 </font>，Array 是存储实体

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

####    例子5
`x`与`y`均指向同一块存储区域，修改`y`的指向的值<br>
```go
func main() {
        x := []int{2, 3, 5, 7, 11}
        y := x[1:3]
        fmt.Println(x, y)
        y[0] = 10
        y[1] = 11
        fmt.Println(x, y)
}
//[2 3 5 7 11] [3 5]
//[2 10 11 7 11] [10 11]
```
![slice-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/slice/slice-1.png)

####    例子6

##	0x03	Slice 技巧
1、Append another Slice（AppendVector）<br>
`append`支持两个参数，第一个是被追加的 slice，第二个参数是追加到后面的数据。第二个参数是变长参数，可以传多值：

```golang
a = append(a, b...)
```

2、copy复制<br>

`copy`方法把数据 `a` 复制到 `b` 中。它有个坑，复制数据的长度取决于 `b` 的当前长度，如果 `b` 没有初始化，那么并不会发生复制操作。所以复制的第一行需要初始化长度。

复制的一种替代方案，利用`append`的多值情况来追加。但是这样会有一个问题，追加的时候是追加到`[]T(nil)`里面，默认初始长度是`0`，每追加一个元素需要检测当前长度是否满足，如果不满足就要扩容，每次扩容扩当前容量的一倍。如此操作的话如果 a 长度是`3`，第一种方法复制出来长度和容量都是`3`，而第二种方法长度是`3`，容量却是`4`。

如果只是单纯复制推荐第一种方式

```golang
b = make([]T, len(a))
copy(b, a)
// 或者
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

3、Cut：Slice拼接（去掉某个连续位置的元素）<br>
Cut操作把 `[i, j)`中间的元素剪切掉。Slice 的切片也遵循**前开后闭**原则

```GOLANG
a = append(a[:i], a[j:]...)
```

示例如下：
```golang
//a = append(a[:i], a[j:]...)
func main() {
        a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}
        a = append(a[:2], a[4:]...)
        fmt.Println(a)
}
//[1 2 5 6 7 8 9]
```

4、Delete操作<br>
删除第`a[i]`位置的数据，提供了两种方式

方式一：利用剪切的方式删除。**因为只是在删除，这个`append`操作并不会引起底层数据的扩容。只不过 `i` 位置之后的数据发生了更新，此时长度减小`1`，容量capacity不变**

方式二：利用`copy`方法实现。将 `i` 位置后面所有的数据迁移，然后删除最后一位数据。将 `i+1` 到 `len(a)` 的数据复制到 `i` 到 `len(a)-1`的位置上。`copy`方法的返回值是复制的元素长度，所以这里又被直接用来截断，将 `a` 的最后一位没有被删除的数据删除

上述两种操作的底层结果一致，即被删除元素的后面元素都需要被复制。


```GOLANG
a = append(a[:i], a[i+1:]...)
// or
a = a[:i+copy(a[i:], a[i+1:])]
```

5、Delete without preserving order：不保留顺序的删除<br>
```GOLANG
a[i] = a[len(a)-1] 
a = a[:len(a)-1]
```
删除第 `i` 位置的元素，把最后一位放到第 `i` 位上，然后把最后一位元素删除。这种方式底层并没有发生复制操作


注意：如果 slice 的类型是指向结构体的指针，或者是结构体 slice 里面包含指针，这些数据在被删除后需要进行GC来释放内存。所以上面的`Cut`和`Delete`方法有可能引起内存泄漏：被删除的数据依然被 `a` 所引用（底层数据中引用）导致无法进行GC。通常用下面的代码解决这个问题：

`Cut`操作优化：相比于之前多了一步操作，将被删除的位置置为 `nil`。这样指针就没有被引用的地方了，可以被GC

```GOLANG
copy(a[i:], a[j:])
for k, n := len(a)-j+i, len(a); k < n; k++ {
        //设置为nil
	a[k] = nil // or the zero value of T
}
a = a[:len(a)-j+i]
```

`Delete`操作优化：
```GOLANG
copy(a[i:], a[i+1:])
a[len(a)-1] = nil // or the zero value of T
a = a[:len(a)-1]
```

Delete without preserving order操作优化：
```GOLANG
a[i] = a[len(a)-1]
a[len(a)-1] = nil
a = a[:len(a)-1]
```

6、Expand<br>
在中间位置 `i` 扩展长度为 `j` 的 slice
```GOLANG
a = append(a[:i], append(make([]T, j), a[i:]...)...)
```

示例：
```GOLANG
func main() {
        a := []int{1, 2, 3, 4}

        j := []int{6, 7, 8}
        i := 2

        a = append(a[:i], append(j, a[i:]...)...)

        fmt.Println(a)
}
//[1 2 6 7 8 3 4]
```

7、Extend<br>
在最后延伸长度是 `j` 的 slice
```GOLANG
a = append(a, make([]T, j)...)
```

8、Insert<br>
在位置 `i` 插入元素 `x`
```GOLANG
a = append(a[:i], append([]T{x}, a[i:]...)...)
```

注意上面通过第二个`append`把 `a[i:]` 追加到 `x` 后面（这个操作会引起`[]T{x}`发生多次扩容），然后通过第一个`append`方法把这个新的 slice 追加到 `a[:i]`后面（这个操作会引起 `a` 发生一次扩容）。此两个操作创建了一个新的 slice（这样相当于创建了内存垃圾）。

如何优化呢？实际上第二个复制也可以被避免，如下代码所示，首先通过`append`将 slice 扩容，然后把 `i` 后面的元素后移，最后复制。整个操作一次扩容。

```GOLANG
s = append(s, 0)
copy(s[i+1:], s[i:])
s[i] = x
```

9、InsertVector<br>
在位置 `i` 插入 slice `b`
```GOLANG
a = append(a[:i], append(b, a[i:]...)...)
```

10、Pop/Shift<br>
一行实现 pop 操作，出队列头
```golang
x, a = a[0], a[1:]
```

11、Pop Back<br>
一行实现 pop 操作，出队列尾
```GOLANG
x, a = a[len(a)-1], a[:len(a)-1]
```

12、Push<br>
push `x` 到队列尾
```GO
a = append(a, x)
```

13、Push Front/Unshift<br>
push `x` 到队列头
```GOLANG
a = append([]T{x}, a...)
```

14、Additional Tricks 附加技巧<br>

####    Filtering without allocating：不申请内存过滤数据
多个slice切片引用的底层数组是有可能是同一个，利用这个原理可以实现复用底层数组实现数据过滤。当然，过滤之后底层数组内容会被修改

```GOLANG
b := a[:0]
for _, x := range a {
	if f(x) {
		b = append(b, x)
	}
}
```
上面代码实现了就地过滤。首先申明切片 `b`，和 `a` 共享底层数组。遍历 `a` 进行过滤，过滤后到加入 `b` 中。这样 `a` 和 `b` 同时被修改了。`b` 是过滤后正确的 slice，而 `a` 的数据会错乱。


####    Reversing 反转
将 slice 的数据顺序反转：
```GOLANG
for i := len(a)/2-1; i >= 0; i-- {
	opp := len(a)-1-i
	a[i], a[opp] = a[opp], a[i]
}
```

上述代码再简化为（可以将反转用到的索引省略），离谱：
```GOLANG
for left, right := 0, len(a)-1; left < right; left, right = left+1, right-1 {
	a[left], a[right] = a[right], a[left]
}
```

####    Shuffling 随机
Fisher–Yates 算法（需要Go `1.10` 以上 `math/rand.Shuffle`）：每个数据随机一个新位置出来

```GOLANG
for i := len(a) - 1; i > 0; i-- {
    j := rand.Intn(i + 1)
    a[i], a[j] = a[j], a[i]
}
```


##	0x04	参考
-	[对 Go 的 Slice 进行 Append 的一个坑](http://sharecore.net/post/%E5%AF%B9go%E7%9A%84slice%E8%BF%9B%E8%A1%8Cappend%E7%9A%84%E4%B8%80%E4%B8%AA%E5%9D%91/)
-	[Slice 小技巧](https://blog.cyeam.com/golang/2018/06/18/slicetricks)
-	[Go 语言的 slice 为啥有这样的奇怪问题呢？ - 达达的回答 - 知乎](https://www.zhihu.com/question/27161493/answer/35485751)
-       [理解 Go 中的 Slice](https://sanyuesha.com/2018/07/31/go-slice/)
-       [Go Slice Tricks Cheat Sheet](https://ueokande.github.io/go-slice-tricks/)
-       [SliceTricks](https://github-wiki-see.page/m/golang/go/wiki/SliceTricks)
-       [SliceTricks](https://github.com/golang/go/wiki/SliceTricks)