---
layout: post
title: Golang httprouter 库分析
subtitle: 高效的路由框架 httprouter 分析
date: 2021-10-01
author: pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category: false
tags:
  - Golang
  - 基数树
  - Radix
---

## 0x00 前言

[httprouter](https://github.com/julienschmidt/httprouter) 非常高效的一个 http 路由框架，gin 框架的路由也基于此库。使用比较简单：

```golang
func main() {
    router := httprouter.New()

    // 注册路由
    router.GET("/", Index)
    router.GET("/hello/:name", Hello)

    log.Fatal(http.ListenAndServe(":8080", router))
}
```

理解 httprouter 的核心在于理解 [基数树](https://zh.m.wikipedia.org/zh-hans/%E5%9F%BA%E6%95%B0%E6%A0%91) 的实现，路由搜索就是基数树搜索。


## 0x01 基数树 Radix Tree 的实现

httprouter 基于 Radix Tree 实现，Radix Tree 的特点是拥有共同前缀的节点也共享同一个父节点，是一种更节省空间的 Trie 树（前缀树），其中作为唯一子节点的每个节点都与其父节点合并，边既可以表示为元素序列又可以表示为单个元素。 因此每个内部节点的子节点数最多为基数树的基数 `r` ，其中 `r` 为正整数，`x` 为 `2` 的幂（`x>=1`），这使得基数树更适用于对于较小的集合（尤其是字符串很长的情况下）和有很长相同前缀的字符串集合。

经典例子：

```text
Priority   Path             Handle
9          \                *<1>
3          ├s               nil
2          |├earch\         *<2>
1          |└upport\        *<3>
2          ├blog\           *<4>
1          |    └:post      nil
1          |         └\     *<5>
2          ├about-us\       *<6>
1          |        └team\  *<7>
1          └contact\        *<8>
```

对应于下面的 GET 请求，`*<num>` 是方法（handler）对应的指针，从根节点遍历到叶子节点我们就能得到完整的路由表：

```golang
GET("/", func1)
GET("/search/", func2)
GET("/support/", func3)
GET("/blog/", func4)
GET("/blog/:post/", func5)
GET("/about-us/", func6)
GET("/about-us/team/", func7)
GET("/contact/", func8)
```

####    Radix 树 VS hash 表
在少量长整型数据、普通字符串等 key-value 存储场景，Radix 和 hash 表相比，有如下优势：
-   不需要考虑 Hash 冲突
-   简单，可能会节省空间
-   不需要设计 hash 映射函数

借助于 Radix 树，我们可以 实现对于长整型数据类型的路由，可以根据一个长整型（比如一个长 ID）快速查找到其对应的 Value 指针

####    Radix 树 VS Trie 树
![radix-vs-trie](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/datastructure/radix-tree/trie2radixTree.png)

####    搜索
下图展示了在基数树中进行字符串查找的示例，通过基数树可以提高字符顺序匹配的效率，对于 URL 之类的字符使用基数树来进行归类、匹配非常适合。

![search](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/datastructure/radix_tree.png)

####   Radix 树的实现
以项目 [go-radix](https://github.com/armon/go-radix/) 为例，先窥探下 radix 树的结构：

```golang
// Tree implements a radix tree. This can be treated as a
// Dictionary abstract data type. The main advantage over
// a standard hash map is prefix-based lookups and
// ordered iteration,
type Tree struct {
	root *node
	size int
}

type node struct {
	// leaf is used to store possible leaf
	leaf *leafNode

	// prefix is the common prefix we ignore
	prefix string

	// Edges should be stored in-order for iteration.
	// We avoid a fully materialized slice to save memory,
	// since in most cases we expect to be sparse
	edges edges
}

type edges []edge

// edge is used to represent an edge node
type edge struct {
	label byte
	node  *node
}

// leafNode is used to represent a value
type leafNode struct {
	key string
	val interface{}
}
```

如上面代码：
-   `Tree`：radixTree 开始节点
-   `node`：radixTree 路径上的节点
-   `edge`：指向下一层节点的指针
-   `edges`：把 `edge` 封装为 slice，便于操作
-   `leafNode`：叶子节点

用下图可以表示 go-radix 的结构关系：

![node-relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/datastructure/radix-tree/radix-tree-node-struct.png)

`node`的结构如下图：
![node](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/datastructure/radix-tree/radix-node-relation.png)


####    节点间的关系


##  0x02    go-radix 的主要操作解读

####    查找
查找需要按照关键字一层一层匹配即可，如果在某一层的所有 child 节点中都没有匹配到则查找失败，本库做了 [优化](https://github.com/armon/go-radix/blob/master/radix.go#L379)，返回匹配到最长的 key：
```golang
// LongestPrefix is like Get, but instead of an
// exact match, it will return the longest prefix match.
func (t *Tree) LongestPrefix(s string) (string, interface{}, bool) {
	var last *leafNode
	n := t.root
	search := s
	for {
		// Look for a leaf node
		if n.isLeaf() {
			last = n.leaf
		}

		// Check for key exhaution
		if len(search) == 0 {
			break
		}

		// Look for an edge
		n = n.getEdge(search[0])
		if n == nil {
			break
		}

		// Consume the search prefix
		if strings.HasPrefix(search, n.prefix) {
			search = search[len(n.prefix):]
		} else {
			break
		}
	}
	if last != nil {
		return last.key, last.val, true
	}
	return "", nil, false
}

// longestPrefix finds the length of the shared prefix
// of two strings
func longestPrefix(k1, k2 string) int {
	max := len(k1)
	if l := len(k2); l < max {
		max = l
	}
	var i int
	for i = 0; i < max; i++ {
		if k1[i] != k2[i] {
			break
		}
	}
	return i
}

```

####    插入
插入流程如下 [代码](https://github.com/armon/go-radix/blob/master/radix.go#L147)：
1.  开始，从根 root 节点往下找
2.  如果在某个 child 节点中找到相同前缀则继续往下找（需匹配完本节点的所有 prefix，看上面的 `longestPrefix` 方法）
3.  直到查找到某个节点处停止；如果某个 child 节点部分前缀是字符串前缀，则需要将公共前缀作为新节点，原来孩子节点的孩子作为新节点的孩子（分裂的目的是确保了一个节点的孩子数目不会超过总的字符串的个数）

插入的用例，假设以空字符串作为结尾：
![]()

```golang
// Insert is used to add a newentry or update
// an existing entry. Returns if updated.
func (t *Tree) Insert(s string, v interface{}) (interface{}, bool) {
	var parent *node
	n := t.root
	search := s
	for {
		// Handle key exhaution
		if len(search) == 0 {
			if n.isLeaf() {
				old := n.leaf.val
				n.leaf.val = v
				return old, true
			}

			n.leaf = &leafNode{
				key: s,
				val: v,
			}
			t.size++
			return nil, false
		}

		// Look for the edge
		parent = n
		n = n.getEdge(search[0])

		// No edge, create one
		if n == nil {
			e := edge{
				label: search[0],
				node: &node{
					leaf: &leafNode{
						key: s,
						val: v,
					},
					prefix: search,
				},
			}
			parent.addEdge(e)
			t.size++
			return nil, false
		}

		// Determine longest prefix of the search key on match
		commonPrefix := longestPrefix(search, n.prefix)
		if commonPrefix == len(n.prefix) {
			search = search[commonPrefix:]
			continue
		}

		// Split the node
		t.size++
		child := &node{
			prefix: search[:commonPrefix],
		}
		parent.updateEdge(search[0], child)

		// Restore the existing node
		child.addEdge(edge{
			label: n.prefix[commonPrefix],
			node:  n,
		})
		n.prefix = n.prefix[commonPrefix:]

		// Create a new leaf node
		leaf := &leafNode{
			key: s,
			val: v,
		}

		// If the new key is a subset, add to to this node
		search = search[commonPrefix:]
		if len(search) == 0 {
			child.leaf = leaf
			return nil, false
		}

		// Create a new edge for the node
		child.addEdge(edge{
			label: search[0],
			node: &node{
				leaf:   leaf,
				prefix: search,
			},
		})
		return nil, false
	}
}
```

## 0x03 重点：HTTPROUTER 分析

1.	路由是是如何注册？如何保存的？-- 对应于radix树的构造
2.	当请求到来之后，路由是如何匹配，如何查找的？-- 对应于radix树的查询


再回归下radix树的要点，对于radix树的每个节点，如果该节点是唯一的子树的话，就和父节点合并。


#### Path

#### Router

```golang
// Router is a http.Handler which can be used to dispatch requests to different
// handler functions via configurable routes
type Router struct {
    trees map[string]*node  //Map 存储了 HTTP 不同方法的根节点
    paramsPool sync.Pool    //WITH efficiency
    maxParams  uint16
    SaveMatchedRoutePath bool
    RedirectTrailingSlash bool
    RedirectFixedPath bool
    HandleMethodNotAllowed bool
    HandleOPTIONS bool
    GlobalOPTIONS http.Handler  //
    globalAllowed string
    NotFound http.Handler   //
    MethodNotAllowed http.Handler   //
    PanicHandler func(http.ResponseWriter, *http.Request, interface{})
}
```

`Router` 也实现了 `ServeHTTP` 方法，因此符合 `net/http.Handler` 接口，`ServeHTTP` 方法为处理请求时，查找路由树及 `Handler` 的逻辑：

```golang
// ServeHTTP makes the router implement the http.Handler interface.
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    if r.PanicHandler != nil {
        defer r.recv(w, req)
    }

    path := req.URL.Path

    // 获取本次请求路由（path）的 handle 方法
    if root := r.trees[req.Method]; root != nil {
    	if handle, ps, tsr := root.getValue(path, r.getParams); handle != nil {
            //root.getValue 查找路由树的具体方法
    		if ps != nil {
    			handle(w, req, *ps)
    			r.putParams(ps)
    		} else {
    			handle(w, req, nil)
    		}
    		return
    .....
}
```

#### Tree

## 0x03 路由注册

## 0x04 路由查找

## 0x05 查找算法

## 0x06 参考

- [基数树 (Radix Tree)](https://juejin.cn/post/6933244263241089037)
- [Radix tree](https://en.wikipedia.org/wiki/Radix_tree)
- [hashicorp：An immutable radix tree implementation in Golang](https://github.com/hashicorp/go-immutable-radix)
- [6.2. router 请求路由](https://books.studygolang.com/advanced-go-programming-book/ch6-web/ch6-02-router.html)
- [Wikipedia - 基数树](https://zh.m.wikipedia.org/zh-hans/%E5%9F%BA%E6%95%B0%E6%A0%91)
- [httprouter 源码分析](https://learnku.com/articles/27591)