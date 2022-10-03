---
layout: post
title: 数据结构与算法回顾（五）：golang 的 container 包
subtitle: list、heap 和 ring
date: 2022-05-30
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 数据结构
  - Golang
---

## 0x00 前言
golang 的标准库 `container` 中，提供了 heap/list/ring 的实现。本文简单分析下实现和应用。

##	0x01	heap
以 minheap 最小二叉堆为例，堆具有以下特性：
-	任意节点小于它的所有子孙节点，最小节点在堆的根上（堆序性）
-	堆是一棵完全树（complete tree）：即除了最底层，其他层的节点都被元素填满，且最底层尽可能地从左到右填入
-	由于堆是完全二叉树，所以可以用顺序数组来表示，如下图

![heap-as-array](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/heap/heap-as-array.png)

![heap-as-array2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/heap/heap-sort.png)

####  	源码分析
标准库的 heap 是一个 `interface`，因此开发者需要完成三件事：

1、实现相关的接口（共 `5` 个，包含了 `sort.Interface` 的 `3` 个），在源码分析时，特别注意这几个公共接口的嵌入位置<br>
```golang
type Interface interface {
    sort.Interface      // 继承了 sort.Interface，包含了Less/Len/Swap三个方法
    Push(x interface{}) // add x as element Len()
    Pop() interface{}   // remove and return element Len() - 1.
}
```

2、提供承载数据的slice<br>
这里注意两件事：
-	对于原生结构，如`[]int`等，开发者实现的`Swap`比较简单，直接`h[i], h[j] = h[j], h[i]`交换即可
-	对于符合结构，如`[]*Item`，开发者实现`Swap`机制有两种选择：
	1.	按照原生方式实现
	2.	只利用`[]*Item`作为存储，在`Item`加上`index`成员，充当堆的下标（参考优先级队列的实现），这时`Swap`就需要考虑`Item.index`的交换，对应下图

![swap](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/heap/heap-swap-pointer.png)

3、以上完成后，可以调用heap库暴露的方法来操作heap（PS：接口都需要传入上述接口的实例化对象）<br>
另外，这里特别注意的是不要混淆`heap.Push`和自己slice实现的`Push`方法：
-	开发者实现的`Push`方法仅仅是对slice操作
-	`heap.Push`调用了slice的`Push`操作，还需要额外的调整用以维护heap性质

```golang
// 建堆， 对heap进行初始化，生成小根堆（或大根堆）
func Init(h Interface)
// 插入元素
func Push(h Interface, x interface{})
// 弹出root元素
func Pop(h Interface) interface{}
// Update元素(包括优先级)，从i位置数据发生改变后，对堆再平衡，优先级队列的实现会使用此方法
func Fix(h Interface, i int)
// 删除，从指定位置删除数据，并返回删除的数据，同时亦涉及到堆的再平衡
func Remove(h Interface, i int) interface{}
```

回想下，minheap 的插入 / 删除过程：
- 向 minheap 中插入 `Push` 一个元素的时候，将此元素插入到最右子树的最后一个节点中，然后调用 `up` 向上调整保证最小堆性质
- 从 minheap 中取出堆顶元素（最小的）时，先把该元素和右子树最后一个节点交换，然后 `Pop` 出最后一个节点，然后对根节点调用 `down` 方法向下调整保证最小堆性质
- 从 minheap 的任意位置取数据都类似上面的做法


####	一个容易忽略的点：`Swap`方法
上面已经讨论，主要涉及到非标准型结构的交换问题


####  主要方法分析

1、Init[方法](https://cs.opensource.google/go/go/+/refs/tags/go1.18.3:src/container/heap/heap.go;l=42)<br>
`Init` 为初始化建立 heap 的方法，关键在 `heap.down` 方法：
- 在 `Init` 方法中，调整的位置，第 `1` 个的元素的位置是 `n/2-1` 个，符合 minheap 的特性；最后一个位置是堆顶的位置 `0`，保证每个元素都不漏下
- `heap.down` 方法的作用是，任选一个元素 `i`，将与其子节点 `2i+1` 和 `2i+2` 比较，如果 `i` 比它的子节点小，则将 `i` 与两个子节点中较小的节点交换（代码中的 `j`）；子节点 `j` 再与它的子节点，继续比较、交换，直到数组末尾、或者待比较的元素比它的两个子节点都小，跳出当前的 `heap.down` 循环

```golang
// A heap must be initialized before any of the heap operations
// can be used. Init is idempotent with respect to the heap invariants
// and may be called whenever the heap invariants may have been invalidated.
// Its complexity is O(n) where n = h.Len().
//
func Init(h Interface) {
	// heapify
	n := h.Len()	//堆长度，下标从0 ~ n-1
	for i := n/2 - 1; i >= 0; i-- {
		// 从长度的一半开始，一直到第0个数据，每个位置都调用down方法，down方法实现的功能是保证从该位置往下保证形成堆
		down(h, i, n)
	}
}

// 给定类型，需要调整的元素在数组中的索引以及 heap 的长度
// 将该元素下沉到该元素对应的子树合适的位置，从而满足该子树为最小堆
func down(h Interface, i0, n int) bool {
	i := i0	// 中间变量，初始化保存为：往下调整为heap所在的节点位置
	for {
		j1 := 2*i + 1	// i节点的左子孩子
		if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
			// 如果j1 越界了，说明已经调整完成了，可以退出循环
			break
		}
		j := j1 // left child

		//中间变量j先赋值为左子孩子，之后j将被赋值为左右子孩子中最小（大）的一个孩子的位置
		if j2 := j1 + 1; j2 <n && h.Less(j2, j1) {
			j = j2 // = 2*i + 2  // right child
		}
		//j被赋值为两个孩子中的最小（大）孩子的位置（由开发者实现的Less方法决定）
		if !h.Less(j, i) {
      		// 比较孩子和当前的父亲节点，如果满足堆的要求了，退出循环（注意：j在前，i在后，结果取非）
			break
		}
		h.Swap(i, j) // 否则交换i和j位置的值，继续比较
		i = j		// 保存j的位置，继续向下调整，保证j位置的子树是heap结构
	}

	//这个结果有点意思：如果i>i0，说明调整了，返回true；否则，未调整返回false
	return i > i0
}
```

![adjust](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/heap/heap-sort-adjust.png)

2、`Push` 方法 <br>
`Push` 方法保证插入新元素时，顺序数组 `h` 仍然是一个 heap；和上面的描述一致，将 `x` 元素插入到了数组的末尾位置，再调用 `up` 方法自下而上进行调整，使其满足 heap 的性质：

`heap.up` 方法也较易理解：依此（`for loop`）查找元素 `j` 的父节点（`i`），如果 `j` 比父节点 `i` 要小，则交换这两个节点，并继续向再上一级的父节点比较，直到根节点，或者元素 `j` 大于 父节点 `i`（调整完毕，无需再继续进行）
```golang
// Push pushes the element x onto the heap. The complexity is
// O(log(n)) where n = h.Len().
func Push(h Interface, x interface{}) {
	// 将新插入进来的节点放到最后（调用开发者封装的Push）
	h.Push(x)
	// 自下而上调整
	up(h, h.Len()-1)
}

func up(h Interface, j int) {
	for {
		i := (j - 1) / 2 // parent（j节点的父节点）
		if i == j || !h.Less(j, i) {
			// 如果越界，或者满足堆的条件（使用开发者实现的Less方法），则结束for循环
			break
		}
		// 否则将该节点和父节点交换，继续下一轮比较
		h.Swap(i, j)
		j = i	// 交换当前位置，对父节点继续进行检查直到根节点
	}
}
```

3、`Pop` 方法 <br>
`heap.Pop` 方法是取出堆顶位置的数据（minheap 为最小），取完数据之后，heap 肯定不平衡。所以通常的做法是：将根节点（`0` 位置）与末尾节点的元素交换，并将新的根节点的元素（先前的最后一个元素）down 自上而下调整到合适的位置，满足最小堆的要求；

最后再调用用户自定义的 Pop 函数获取最后一个元素即可

**这里需要区分 heap 包的 Pop 方法和用户自定义实现的 Pop 方法的根本区别**，用户的 `Pop` 方法只是用来获取数据的
```golang
// Pop removes the minimum element (according to Less) from the heap
// and returns it. The complexity is O(log(n)) where n = h.Len().
// It is equivalent to Remove(h, 0).
func Pop(h Interface) interface{} {
	// 把最后一个节点和第一个节点进行交换，之后，从根节点开始重新保证堆结构，最后把最后那个节点数据丢出并返回
	n := h.Len() - 1
	h.Swap(0, n)
	down(h, 0, n)
	return h.Pop()
}
```

4、`Remove` 方法 <br>
`Remove` 方法提供了删除指定位置 index 元素的实现，即先将要删除的节点 `i` 与末尾节点 `n` 交换，然后将新的节点 `i` 下沉或上浮到合适的位置（通俗的说，由于新数据调整，原先末尾的位置升到了它不该在的位置，需要调整这个元素，先一路 `down` 到底，然后再一路 `up` 到最终的位置）
```golang
// Remove removes the element at index i from the heap.
// The complexity is O(log(n)) where n = h.Len().
//
func Remove(h Interface, i int) interface{} {
	n := h.Len() - 1
	if n != i {
		//Pop只是Remove的特例
		//Remove是把i位置的节点和最后一个节点进行交换，之后保证从i节点往下及往上都保证堆结构，最后把最后一个节点的数据返回
		h.Swap(i, n)
		if !down(h, i, n) {
			up(h, i)
		}
	}
	return h.Pop()
}
```

5、`Fix` 方法 <br>
`Fix` 方法的意义是在优先级队列的场景（从 `i` 位置数据发生改变后，对 heap 再平衡，优先级队列会使用本方法）。即当`i`节点的**比较值**发生改变后，需要保证heap的再平衡：**先调用down保证该节点下面的堆结构，如果有位置交换，则需要保证该节点往上的堆结构，否则就不需要往上保证堆结构（没有调整影响另一侧的话，肯定是平衡的）**，这里画个图，非常容易理解。
```golang
// Fix re-establishes the heap ordering after the element at index i has changed its value.
// Changing the value of the element at index i and then calling Fix is equivalent to,
// but less expensive than, calling Remove(h, i) followed by a Push of the new value.
// The complexity is O(log(n)) where n = h.Len().
func Fix(h Interface, i int) {
	if !down(h, i, h.Len()) {
		up(h, i)
	}
}
```


####  heap 实例化
下面使用 `[]int` 的 slice 结构来实现一个 heap，注意调用方式采用 heap 包的方法进行调用：
```golang
type IntHeap []int

func main() {
    h := &IntHeap{2, 1, 5, 13,11, 4, 3, 7, 9, 8, 0}
    heap.Init(h)                                // 将数组切片进行堆化
    fmt.Println(*h)
    fmt.Println(heap.Pop(h))                    // 调用 pop 0 返回移除的顶部最小元素
    heap.Push(h, 6)                             // 添加一个元素进入堆中进行堆化
    for len(*h) > 0 {
      fmt.Printf("%d \n", heap.Pop(h))
    }
}

func (h IntHeap) Len() int { return len(h) }

func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }   //minheap

//交换两个元素位置
func (h IntHeap) Swap(i, j int) {		
        h[i], h[j] = h[j], h[i]
}

func (h *IntHeap) Pop() interface{} {
    old := *h
    n := len(old)
    // 将顶小堆元素与最后一个元素交换位置，在进行堆排序的结果
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}

// 实现 Push 方法，插入新元素
func (h *IntHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}
```


####	heap 的应用
1.	定时器
2.	优先级队列：比如kubernetes中的[实现](https://dev.to/chuck_ha/data-types-in-kubernetes-priorityqueue-38d2)，[FIFO-PriorityQueue](https://github.com/kubernetes/kubernetes/blob/v1.13.2/pkg/scheduler/internal/queue/scheduling_queue.go)
3.	heap排序


这里使用 heap 实现优先级队列，如下定义，调用代码[在此](https://github.com/pandaychen/golang_in_action/blob/master/datastruct/heap/heap-app2.go#L59)：
```golang
// An Item is something we manage in a priority queue.
type Item struct {
    value    string  // 优先级队列中的数据
    priority int    // The priority of the item in the queue.
    // The index is needed by update and is maintained by the heap.Interface methods.
	// index是该节点在堆中的位置，这里采用我们所说的复合结构，注意Swap操作
	index int // The index of the item in the heap.
}

// A PriorityQueue implements heap.Interface and holds Items.
type PriorityQueue []*Item

func (pq PriorityQueue) Len() int { return len(pq) }

func (pq PriorityQueue) Less(i, j int) bool {
    // We want Pop to give us the highest, not lowest, priority so we use greater than here.
    return pq[i].priority > pq[j].priority
}

func (pq PriorityQueue) Swap(i, j int) {
    pq[i], pq[j] = pq[j], pq[i]
    // pq[i].index = i
    // pq[j].index = j
	//pq[j].index,pq[i].index =  pq[i].index,pq[j].index 
	pq[i].index, pq[j].index = i, j
}

func (pq *PriorityQueue) Push(x interface{}) {
    n := len(*pq)
    item := x.(*Item)
    item.index = n
    *pq = append(*pq, item)
}

func (pq *PriorityQueue) Pop() interface{} {
    old := *pq
    n := len(old)
    item := old[n-1]
	//将index置为-1是为了标识该数据已经出了优先级队列
    item.index = -1 // for safety
    *pq = old[0 : n-1]
    return item
}

// update modifies the priority and value of an Item in the queue.
// 更新优先级队列中某个指定item的优先级，涉及到heap的再平衡，本操作修改了优先级和值的item在优先级队列中的位置
func (pq *PriorityQueue) update(item *Item, value string, priority int) {
    item.value = value
    item.priority = priority
    heap.Fix(pq, item.index)
}
```

与上例子不同的是，优先级队列有个`update`方法，该方法用以实时调整heap某个元素的priority，调整之后会触发`heap.Fix`方法对heap进行重排序以达到heap的特性

####	小结
使用标准库的heap，需要明确这几项：
1.	定义自己的接口，实现heap所需要的方法
2.	注意package的方法，和结构体的方法，虽然同名，但是功能完全不一样
3.	`heap.Fix`、`heap.Remove`方法的使用场景

##	0x02	list包
`container/list`包，[https://go.dev/src/container/list/list.go]有两个暴露对外部的结构：
-	`List`：List 实现了一个双向链表
-	`Element`：Element 代表了链表中元素的结构

注意，List是可以做到开箱即用的，看下面List对外的方法，参数均为`interface{}` 类型；

```golang
// List represents a doubly linked list.
// The zero value for List is an empty list ready to use.
type List struct {
	//链表的根元素
	root Element // sentinel list element, only &root, root.prev, and root.next are used

	//链表的长度
	len  int     // current list length excluding (this) sentinel element
}

// Element is an element of a linked list.
type Element struct {
	// Next and previous pointers in the doubly-linked list of elements.
	// To simplify the implementation, internally a list l is implemented
	// as a ring, such that &l.root is both the next element of the last
	// list element (l.Back()) and the previous element of the first list
	// element (l.Front()).
	next, prev *Element	// 前/后元素指针

	// The list to which this element belongs.
	list *List	//元素所在的链表

	// The value stored with this element.
	Value any		//这个any是interface{}，用来存储元素值
}
```

####	list的使用
```golang
func main() {
	//创建List
	var l = list.New()
	e0 := l.PushBack(1)
	e1 := l.PushFront(3)
	e2 := l.PushBack(7)

	l.InsertBefore(3, e0)
	l.InsertAfter(196, e1)
	l.InsertAfter(829, e2)

	for e := l.Front(); e != nil; e = e.Next() {
		fmt.Printf("%#v\n", e.Value.(int))
	}
}
```

####	Element的方法

```golang
func (e *Element) Next() *Element // 返回该元素的下一个元素，如果没有下一个元素则返回 nil
func (e *Element) Prev() *Element // 返回该元素的前一个元素，如果没有前一个元素则返回 nil
```

####	List的方法

```golang
func New() *List                                                    // 构造一个初始化的list
func (l *List) Back() *Element                                      // 获取list l的最后一个元素
func (l *List) Front() *Element                                     // 获取list l的最后一个元素
func (l *List) Init() *List                                         // list l 初始化或者清除 list l
func (l *List) InsertAfter(v interface{}, mark *Element) *Element   // 在 list l 中元素 mark 之后插入一个值为 v 的元素，并返回该元素，如果 mark 不是list中元素，则 list 不改变
func (l *List) InsertBefore(v interface{}, mark *Element) *Element  // 在 list l 中元素 mark 之前插入一个值为 v 的元素，并返回该元素，如果 mark 不是list中元素，则 list 不改变
func (l *List) Len() int                                            // 获取 list l 的长度
func (l *List) MoveAfter(e, mark *Element)                          // 将元素 e 移动到元素 mark 之后，如果元素e 或者 mark 不属于 list l，或者 e==mark，则 list l 不改变
func (l *List) MoveBefore(e, mark *Element)                         // 将元素 e 移动到元素 mark 之前，如果元素e 或者 mark 不属于 list l，或者 e==mark，则 list l 不改变
func (l *List) MoveToBack(e *Element)                               // 将元素 e 移动到 list l 的末尾，如果 e 不属于list l，则list不改变             
func (l *List) MoveToFront(e *Element)                              // 将元素 e 移动到 list l 的首部，如果 e 不属于list l，则list不改变             
func (l *List) PushBack(v interface{}) *Element                     // 在 list l 的末尾插入值为 v 的元素，并返回该元素              
func (l *List) PushBackList(other *List)                            // 在 list l 的尾部插入另外一个 list，其中l 和 other 可以相等               
func (l *List) PushFront(v interface{}) *Element                    // 在 list l 的首部插入值为 v 的元素，并返回该元素              
func (l *List) PushFrontList(other *List)                           // 在 list l 的首部插入另外一个 list，其中 l 和 other 可以相等              
func (l *List) Remove(e *Element) interface{}                       // 如果元素 e 属于list l，将其从 list 中删除，并返回元素 e 的值
```

####	List的问题（注意）
1.	不要使用自己构造的Element结构，作为参数传入List的方法
2.	`Remove`[方法](https://cs.opensource.google/go/go/+/refs/tags/go1.19.1:src/container/list/list.go;drc=54182ff54a687272dd7632c3a963e036ce03cb7c;l=108)是传入指定位置的元素（list的`Remove`实现），复杂度是`O(1)`，需要开发者保存对应的element的指针（地址）
3.	若在goroutine并发环境下使用`container/list`链表，那么需要加锁
4.	List并未提供`Pop`类方法，需要自行组合实现，不过需要加锁
5.	List如何正确删除所有元素？
6.	List包中大部分对于`e *Element`进行操作的元素都可能会导致程序崩溃，其根本原因是`e`是一个`Element`类型的指针，当然其也可能为`nil`，但是golang中list包中函数没有对其进行是否为`nil`的检查，变默认其非`nil`进行操作，所以这种情况下，便可能出现程序崩溃

1、`Remove`的实现<br>
```GOLANG
// Remove removes e from l if e is an element of list l.
// It returns the element value e.Value.
// The element must not be nil.
func (l *List) Remove(e *Element) any {
	if e.list == l {
		// if e.list == l, l must have been initialized when e was inserted
		// in l or l == nil (e is a zero Element) and l.remove will crash
		l.remove(e)
	}
	return e.Value
}

// remove removes e from its list, decrements l.len
func (l *List) remove(e *Element) {
	e.prev.next = e.next
	e.next.prev = e.prev
	e.next = nil // avoid memory leaks
	e.prev = nil // avoid memory leaks
	e.list = nil
	l.len--
}
```


####	List的典型应用场景
-	时间轮的[任务链表定义](https://pandaychen.github.io/2022/05/28/A-TIMEWHEEL-ANALYSIS/#0x02-go-zero-%E7%9A%84%E6%97%B6%E9%97%B4%E8%BD%AE)
-	Cache中实现LRU机制的[链式结构](https://pandaychen.github.io/2022/06/02/A-GOLANG-LRUCACHE-ANALYSIS-2/#keylru-%E7%9A%84%E5%AE%9E%E7%8E%B0)


####	List的坑


##	0x03	ring包
`container/ring` 实现了环形链表的功能，其典型应用场景是构造定长环形队列，比如用来保存固定size的元素，如最近`N`条日志等。Ring的结构如下：

```golang
type Ring struct {
    next *Ring
	prev  *Ring
    Value       interface{}
}
```

####	ring的典型用法
```golang
func main() {
	// 创建一个环, 包含 3 个元素
	r := ring.New(3)
	fmt.Printf("ring: %+v\n", *r)

	// 初始化
	for i := 1; i <= 3; i++ {
		r.Value = i
		r = r.Next()
	}
	fmt.Printf("init ring: %+v\n", *r)

	// sum
	s := 0
	r.Do(func(i interface{}) {
        fmt.Println(i)
		s += i.(int)
	})
	fmt.Printf("sum ring: %d\n", s)
}
```

####	Ring可调用的方法
```golang
func New(n int) *Ring               // 用于创建一个新的 Ring, 接收一个整形参数，用于初始化 Ring 的长度  
func (r *Ring) Len() int            // 环长度

func (r *Ring) Next() *Ring         // 返回当前元素的下个元素
func (r *Ring) Prev() *Ring         // 返回当前元素的上个元素
func (r *Ring) Move(n int) *Ring    // 指针从当前元素开始向后移动或者向前(n 可以为负数)

// Link & Unlink 组合起来可以对多个链表进行管理
func (r *Ring) Link(s *Ring) *Ring  // 将两个 ring 连接到一起 (r 不能为空)
func (r *Ring) Unlink(n int) *Ring  // 从当前元素开始，删除 n 个元素

func (r *Ring) Do(f func(interface{}))  // Do 会依次将每个节点的 Value 当作参数调用这个函数 f, 实际上这是策略方法的引用，通过传递不同的函数以在同一个 ring 上实现多种不同的操作。
```

####	Ring数据结构的应用场景
1.	构造定长环回队列，如保存固定长度的数据等
2.	用作固定长度的对象缓冲区（参见goim的数据结构分析）


####	Ring VS List
Ring 和 List 的区别如下：
-	Ring 类型的数据结构仅由它自身即可代表，而 List 类型则需要由它以及 Element 类型联合表示
-	一个 Ring 类型的值严格来讲，只代表了其所属的循环链表中的一个元素，而一个 List 类型的值则代表了一个完整的链表
-	在创建并初始化一个 Ring 时，可以指定它包含的元素数量，但是对于一个 List 值来说却不需要。循环链表一旦被创建，其长度是不可变的
-	通过 `var r ring.Ring` 声明的 `r` 将会是一个长度为 `1` 的循环链表，而 List 类型的零值则是一个长度为 `0` 的链表。)（List 中的根元素不会持有实际元素的值）
-	Ring 的 `Len` 方法的算法复杂度是 `O(N)`，而 List 的 `Len` 算法复杂度是 `O(1)`

####	ring的典型应用场景


##	0x04	番外：go-zero提供的数据结构

####	queue（FIFO）
[Queue](https://github.com/zeromicro/go-zero/blob/master/core/collection/fifo.go)是go-zero提供的先进先出的安全队列

1、[结构](https://github.com/zeromicro/go-zero/blob/master/core/collection/fifo.go#L6)<br>
```GOLANG
// A Queue is a FIFO queue.
type Queue struct {
	lock     sync.Mutex
	elements []interface{}	//存储
	size     int
	head     int
	tail     int
	count    int
}
```

2、操作方法：`Put`和`Take`<br>
```GOLANG
// Put puts element into q at the last position.
func (q *Queue) Put(element interface{}) {
	q.lock.Lock()
	defer q.lock.Unlock()

	if q.head == q.tail && q.count > 0 {
		nodes := make([]interface{}, len(q.elements)+q.size)
		copy(nodes, q.elements[q.head:])
		copy(nodes[len(q.elements)-q.head:], q.elements[:q.head])
		q.head = 0
		q.tail = len(q.elements)
		q.elements = nodes
	}

	q.elements[q.tail] = element
	q.tail = (q.tail + 1) % len(q.elements)
	q.count++
}

// Take takes the first element out of q if not empty.
func (q *Queue) Take() (interface{}, bool) {
	q.lock.Lock()
	defer q.lock.Unlock()

	if q.count == 0 {
		return nil, false
	}

	element := q.elements[q.head]
	q.head = (q.head + 1) % len(q.elements)
	q.count--

	return element, true
}
```


####	set：集合
[Set](https://github.com/zeromicro/go-zero/blob/master/core/collection/set.go)提供了集合的实现
`Set`结构主要是map为基准，核心在于实现存储key值`interface{}`的通用性，此外还需要实现交集、并集、补集、差集等实现
```GO
// Set is not thread-safe, for concurrent use, make sure to use it with synchronization.
type Set struct {
	data map[interface{}]lang.PlaceholderType
	tp   int
}
```


## 0x05  参考
-  [Wikipedia - 堆](https://zh.wikipedia.org/wiki/%E5%A0%86%E7%A9%8D)
-	[goim 关于 ring 的 issue](https://github.com/Terry-Mao/goim/issues/109)
-	[go 标准库 container 支持 multi goroutine 吗？](https://groups.google.com/g/golang-china/c/JdbR_CGo3ao/m/apyVG5grRVEJ)
-	[Usage of the Heap Data Structure in Go (Golang), with Examples](https://www.tugberkugurlu.com/archive/usage-of-the-heap-data-structure-in-go-golang-with-examples)
-	[Data types in Kubernetes: PriorityQueue](https://dev.to/chuck_ha/data-types-in-kubernetes-priorityqueue-38d2)