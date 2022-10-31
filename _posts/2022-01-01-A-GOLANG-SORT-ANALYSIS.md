---
layout: post
title: 数据结构与算法回顾（七）：Golang 中的 Sort 算法实现解析
subtitle:   排序算法回顾
date: 2021-08-01
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 数据结构
  - Sort
---

## 0x00 前言
本文分析下golang的Sort算法实现

##  0x01    sort的典型使用
Golang 的 `sort.Sort` 使用了一个接口类型 `sort.Interface` 来指定通用的排序算法和可能被排序到的序列类型之间的约定。这个接口的实现由序列的具体表示和它希望排序的元素决定，通常排序的对象是Slice

####    case1：`[]int`排序
对 `[]int` slice序列按照从小到大升序排序
```GOLANG
type IntSlice []int

func (s IntSlice) Len() int           { return len(s) }
func (s IntSlice) Less(i, j int) bool { return s[i] < s[j] }
func (s IntSlice) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }
func main() {
        s := []int{4, 5, 1, 7, 2, 9}
        sort.Sort(IntSlice(s))
        fmt.Println(s)
}
```

####  case2：查找
可以利用sort.Search方法对有序数据进行查找，注意要遵循：使用`sort.Search`时，入参条件是根据要查找的序列是升序序列还是降序序列来决定的，如果是升序序列，则传入的条件应该是**>=目标元素值**；如果是降序序列，则传入的条件应该是**<=目标元素值**
```golang
func main() {
        arrInts := []int{13, 35, 57, 79}
        findPos := sort.Search(len(arrInts), func(i int) bool {
                //return arrInts[i] >= 35 //true
                return arrInts[i]==35 //wrong! 输出不符合预期
        })

        fmt.Println(findPos)
        return
}
```

##  0x02    分析
sort 包内部实现了四种基本的排序算法：
-   插入排序（insertionSort）
-   归并排序（symMerge）
-   堆排序（heapSort）
-   快速排序 （quickSort）


##  0x0 参考
-   [一文搞懂Go语言中的切片排序](https://tonybai.com/2020/11/26/slice-sort-in-go/)
-   [Go sort](https://www.jianshu.com/p/a7317f1a4e50)
- [Go标准库sort.Search是这样实现的二分查找](https://segmentfault.com/a/1190000040178984)