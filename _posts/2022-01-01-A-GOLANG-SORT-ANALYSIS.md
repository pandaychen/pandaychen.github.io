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
本文分析下 golang 的 sort 算法实现

##  0x01    sort 的典型使用
Golang 的 `sort.Sort` 使用了一个接口类型 `sort.Interface` 来指定通用的排序算法和可能被排序到的序列类型之间的约定。这个接口的实现由序列的具体表示和它希望排序的元素决定，通常排序的对象是 slice

####    case1：`[]int` 排序
对 `[]int` slice 序列按照从小到大升序排序
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
可以利用 sort.Search 方法对有序数据进行查找，注意要遵循：使用 `sort.Search` 时，入参条件是根据要查找的序列是升序序列还是降序序列来决定的，如果是升序序列，则传入的条件应该是 **>= 目标元素值**；如果是降序序列，则传入的条件应该是 **<= 目标元素值**
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
-   堆排序（heapSort）：借助了堆数据结构的特性，将数据放到堆以后再取出，便有了顺序
-   快速排序 （quickSort）： 每次选取一个哨兵，然后将比它更小的放左边，比它更大的放右边， 然后递归对左右两侧的子数组进行快排的操作，最终数组便会是一个有序的数组

使用 Golang 原生的排序方法，首先得让要排序的对象实现这个接口：
```GO
// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package. The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

首先，以 `sort.Ints` 的实现为例：

```GO
func Ints(a []int) { Sort(IntSlice(a)) }

func Sort(data Interface) {
	n := data.Len()
	quickSort(data, 0, n, maxDepth(n))
}
```

接着看 `quickSort` 的实现：

```go
func quickSort(data Interface, a, b, maxDepth int) {
	for b-a > 12 { // Use ShellSort for slices <= 12 elements
		if maxDepth == 0 {
			heapSort(data, a, b)
			return
		}
		maxDepth--
		mlo, mhi := doPivot(data, a, b)
		// Avoiding recursion on the larger subproblem guarantees
		// a stack depth of at most lg(b-a).
		if mlo-a < b-mhi {
			quickSort(data, a, mlo, maxDepth)
			a = mhi // i.e., quickSort(data, mhi, b)
		} else {
			quickSort(data, mhi, b, maxDepth)
			b = mlo // i.e., quickSort(data, a, mlo)
		}
	}
	if b-a > 1 {
		// Do ShellSort pass with gap 6
		// It could be written in this simplified form cause b-a <= 12
		for i := a + 6; i < b; i++ {
			if data.Less(i, i-6) {
				data.Swap(i, i-6)
			}
		}
		insertionSort(data, a, b)
	}
}
```

在不同的条件下，`quickSort` 会使用不同的算法：
1.	当元素数量小于 `12` 时，使用 shell sort
2.	当元素数量大于 `12` 且 `maxDepth` 不为 `0` 时，使用原地快排
3.	`maxDepth = 0` 时，转而使用 heap sort

##	0x03	insertionSort（Shell sort）分析

##	0x04	heapSort分析

##	0x05	quickSort分析

##	0x06	总结


##  0x07 参考
-   [一文搞懂 Go 语言中的切片排序](https://tonybai.com/2020/11/26/slice-sort-in-go/)
-   [Go sort](https://www.jianshu.com/p/a7317f1a4e50)
- [Go 标准库 sort.Search 是这样实现的二分查找](https://segmentfault.com/a/1190000040178984)