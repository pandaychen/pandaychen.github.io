---
layout: post
title: Golang httprouter 库分析
subtitle: 高效的路由框架 julienschmidt/httprouter 分析
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

通过上面的示例可以看出：

1.	`*<数字>` 代表一个 handler 函数的内存地址（指针）
2.	search 和 support 拥有共同的父节点 s ，并且 s 是没有对应的 handle 的，只有叶子节点（就是最后一个节点，下面没有子节点的节点）才会注册 handler
3.	从根开始，一直到叶子节点，才是路由的实际路径
4.	路由搜索的顺序是从上向下，从左到右的顺序，为了快速找到尽可能多的路由，包含子节点越多的节点，优先级越高

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
基于`httprouter v1.3.0`分析，重点考虑下面两点：

1.	路由是是如何注册？如何保存的？-- 对应于radix树的构造
2.	当请求到来之后，路由是如何匹配，如何查找的？-- 对应于radix树的查询


再回归下radix树的要点，对于radix树的每个节点，如果该节点是唯一的子树的话，就和父节点合并。


####	Param
用于设置路由的方法`Handle`中的`Params`作何用处？
```golang
func (r *Router) GET(path string, handle Handle) {
    r.Handle("GET", path, handle)
}
// DELETE is a shortcut for router.Handle("DELETE", path, handle)
func (r *Router) DELETE(path string, handle Handle) {
    r.Handle("DELETE", path, handle)
}

//调用handle
type Handle func(http.ResponseWriter, *http.Request, Params)
```

看下`Handle`和`http.HandlerFunc` 多了`Params` 参数，该参数就是用来支持路径中的参数绑定，例如：`/hello/:name`可通过 `Params.ByName("name")` 来获取路径绑定参数的值；`Params` 就是一个 Param 的slice，定义如下：

```golang
// Parram 是一个 URL 参数，包含了一个 Key 和 一个 Value
// Key 就是注册路由时候的使用的字符串，如： "/hello/:name" ，这个 Key 就是 name
// Value 就是访问路径中获取到的值， 如： localhost:8080/hello/xxxxx ，
// 此时 Value 就是 xxxxx
type Param struct {
    Key   string
    Value string
}

// Params 就是一个 Param 的切片，这样就可以看出来， URL 参数可以设置多个了。
// 它是在 tree 的 GetValue() 方法调用的时候设置的，一会分析树的时候可以看到。
// 这个切片是有顺序的，第一个设置的参数就是切片的第一个值，所以通过索引获取值是安全的
type Params []Param

// ByName 返回传入的 name 的值，如果没有找到就会返回一个空字符串
// 这里就是做了个循环，一担找到就将值返回。
func (ps Params) ByName(name string) string {
    for i := range ps {
        if ps[i].Key == name {
            return ps[i].Value
        }
    }
    return ""
}
```

#### Path

#### Router

```golang
// Router 是一个 http.Handler 可以通过定义的路由将请求分发给不同的函数
type Router struct {

	//注册的路由，按照HTTP方法，每个方法对应一棵radix tree
    trees map[string]*node

    // 这个参数是否自动处理当访问路径最后带的 /，一般为 true 就行。
    // 例如： 当访问 /foo/ 时， 此时没有定义 /foo/ 这个路由，但是定义了 
    // /foo 这个路由，就对自动将 /foo/ 重定向到 /foo (GET 请求
    // 是 http 301 重定向，其他方式的请求是 http 307 重定向）。
    RedirectTrailingSlash bool

    // 是否自动修正路径， 如果路由没有找到时，Router 会自动尝试修复。
    // 首先删除多余的路径，像 ../ 或者 // 会被删除。
    // 然后将清理过的路径再不区分大小写查找，如果能够找到对应的路由， 将请求重定向到
    // 这个路由上 ( GET 是 301， 其他是 307 ) 。
    RedirectFixedPath bool

    // 用来配合下面的 MethodNotAllowed 参数。 
    HandleMethodNotAllowed bool

    // 如果为 true ，会自动回复 OPTIONS 方式的请求。
    // 如果自定义了 OPTIONS 路由，会使用自定义的路由，优先级高于这个自动回复。
    HandleOPTIONS bool

    // 路由没有匹配上时调用这个 handler 。
    // 如果没有定义这个 handler ，就会返回标准库中的 http.NotFound 。
    NotFound http.Handler

    // 当一个请求是不被允许的，并且上面的 HandleMethodNotAllowed 设置为 ture 的时候，
    // 如果这个参数没有设置，将使用状态为 with http.StatusMethodNotAllowed 的 http.Error
    // 在 handler 被调用以前，为允许请求的方法设置 "Allow" header 。
    MethodNotAllowed http.Handler

    // 当出现 panic 的时候，通过这个函数来恢复。会返回一个错误码为 500 的 http error 
    // (Internal Server Error) ，这个函数是用来保证出现 painc 服务器不会崩溃。
    PanicHandler func(http.ResponseWriter, *http.Request, interface{})
}
```

`Router` 也实现了 `ServeHTTP` 方法，因此符合 `net/http.Handler` 接口，`ServeHTTP` 方法为处理请求时，查找路由树及 `Handler` 的逻辑：

```golang
// ServeHTTP makes the router implement the http.Handler interface.
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // 在最开始，就做了 panic 的处理，这样可以保证业务代码出现问题的时候，不会导致服务器崩溃
    // 这里就是简单的在浏览器返回一个 500 错误，如果想在 panic 的时候自己处理
    // 只需要将 r.PanicHandler = func(http.ResponseWriter, *http.Request, interface{}) 
    // 重写， 添加上自己的处理逻辑即可。
    if r.PanicHandler != nil {
        defer r.recv(w, req)
    }

    // 通过 Request 获取当前请求的路径
    path := req.URL.Path

    // 到基数树中去查找匹配的路由
    // 首先看是否有请求的方法（比如 GET）是否存在路由，如果存在就继续寻找
    if root := r.trees[req.Method]; root != nil {

        // 如果路由匹配上了，从基数树中将路由取出
        if handle, ps, tsr := root.getValue(path); handle != nil {
            // 获取的是一个函数，签名 func(http.ResponseWriter, *http.Request, Params)
            // 这个就是我们向 httprouter 注册的函数
            handle(w, req, ps)
            // 处理完成后 return ，此次生命周期就结束了
            return

            // 当没有找到的时候，并且请求的方法不是 CONNECT 并且 路径不是 / 的时候 
        } else if req.Method != "CONNECT" && path != "/" {
            // 这里就要做重定向处理， 默认是 301
            code := 301 // Permanent redirect, request with GET method

            // 如果请求的方式不是 GET 就将 http 的响应码设置成 307
            if req.Method != "GET" {
                // Temporary redirect, request with same method
                // As of Go 1.3, Go does not support status code 308.
                // 看上面的注释的意思貌似是作者想使用 308（永久重定向），但是 Go 1.3 
                // 不支持 308 ，所以使用了一个临时重定向。 为什么不能用 301 呢？
                // 因为 308 和 307 不允许请求方法从 POST 更改为 GET
                code = 307
            }

            // tsr 返回值是一个 bool 值，用来判断是否需要重定向, getValue 返回来的 
            // RedirectTrailingSlash 这个就是初始化时候定义的，只有为 true 才会处理 
            if tsr && r.RedirectTrailingSlash {
                // 如果 path 的长度大于 1，只有大于 1 才会出现这种情况，如 p/，path/
                // 并且路径的最后是 / 的时候将最后的 / 去除
                if len(path) > 1 && path[len(path)-1] == '/' {
                    req.URL.Path = path[:len(path)-1]
                } else {
                    // 如果不是 / 结尾的，给结尾添加一个 /
                    // 假设定义了一个路由 '/foo/' ，并且没有定义路由 /foo ,
                    // 实际访问的是 '/foo'， 将 /foo 重定向到 /foo/
                    req.URL.Path = path + "/"
                }

                // 将处理过的路由重定向， 这个是一个 http 标准包里面的方法
                http.Redirect(w, req, req.URL.String(), code)
                return
            }

            // Try to fix the request path
            // 路由没有找到，重定向规则也不符合，这里会尝试修复路径
            // 需要在初始化的时候定义 RedirectFixedPath 为 true，允许修复
            if r.RedirectFixedPath {
                // 这里就是在处理 Router 里面说的，将路径通过 CleanPath 方法去除多余的部分
                // 并且 RedirectTrailingSlash 为 ture 的时候，去匹配路由 
                // 比如： 定义了一个路由 /foo , 但实际访问的是 ////FOO ，就会被重定向到 /foo
                fixedPath, found := root.findCaseInsensitivePath(
                    CleanPath(path),
                    r.RedirectTrailingSlash,
                )
                if found {
                    req.URL.Path = string(fixedPath)
                    http.Redirect(w, req, req.URL.String(), code)
                    return
                }
            }
        }
    }

    // 路由也没有匹配，重定向也没有找到，修复后仍然没有匹配，就到了这里

    // 如果没有任何路由匹配，请求方式又是 OPTIONS， 并且允许响应 OPTIONS
    // 就会给 Header 设置一个 Allow 头，返回去
    if req.Method == "OPTIONS" && r.HandleOPTIONS {
        // Handle OPTIONS requests
        if allow := r.allowed(path, req.Method); len(allow) > 0 {
          w.Header().Set("Allow", allow)
            return
        }
    } else {
        // 如果不是 OPTIONS 或者 不允许 OPTIONS 时 
        // Handle 405
        // 如果初始化的时候 HandleMethodNotAllowed 为 ture
        if r.HandleMethodNotAllowed {
            // 返回 405 响应，通过 allowed() 方法来处理 405 时 allow的值。
            // 大概意思是这样的，比如定义了一个 POST 方法的路由 POST("/foo",...)
            // 但是调用却是通过 GET 方式，这是就会给调用者返回一个包含 POST 的 405
            if allow := r.allowed(path, req.Method); len(allow) > 0 {
                w.Header().Set("Allow", allow)
                if r.MethodNotAllowed != nil {
                    // 这里默认就会返回一个字符串 Method Not Allowed
                    // 可以自定义 r.MethodNotAllowed = 自定义的 http.Handler 实现
                    // 来按照需求响应， allowd 方法也不太难，就是各种判断和查找，就不分析了。
                    r.MethodNotAllowed.ServeHTTP(w, req)
                } else {
                    http.Error(w,
                        http.StatusText(http.StatusMethodNotAllowed),
                        http.StatusMethodNotAllowed,
                    )
                }
                return
            }
        }
    }

    // Handle 404
    // 如果什么的没有找到，就只会返回一个 404 了
    // 如果定义了 NotFound ，就会调用自定义的
    if r.NotFound != nil {
        // 如果需要自定义，在初始化之后给 NotFound 赋一个值就可以了。
        // 可以简单的通过 http.HandlerFunc 包装一个 handler ，例如：
        // router.NotFound = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        //         w.Write([]byte("什么都没有找到"))
        // })
        // 
        r.NotFound.ServeHTTP(w, req)
    } else {
        // 如果没有定义，就返回一个 http 标准库的  NotFound 函数，返回一个 404 page not found
        http.NotFound(w, req)
    }
}
```


#### node结构：radix树中的节点
`node`结构代表了Radix树的一个节点，注意到`Router`结构的定义：
```golang
type Router struct {
    trees map[string]*node	//每个HTTP方法对应一课树
	//...
}
```

node的定义如下：
```GOLANG
type node struct {
    // 当前节点的 URL 路径
    // 如上面图中的例子的首先这里是一个 /
    // 然后 children 中会有 path 为 [s, blog ...] 等的节点 
    // 然后 s 还有 children node [earch,upport] 等，就不再说明了
    path      string

    // 判断当前节点路径是不是含有参数的节点, 上图中的 :post 的上级 blog 就是wildChild节点
    wildChild bool

    // 节点类型: static, root, param, catchAll
    // static: 静态节点, 如上图中的父节点 s （不包含 handler 的)
    // root: 如果插入的节点是第一个, 那么是root节点
    // catchAll: 有*匹配的节点
    // param: 参数节点，比如上图中的 :post 节点
    nType     nodeType

    // path 中的参数最大数量，最大只能保存 255 个（超过这个的情况貌似太难见到了）
    // 这里是一个非负的 8 进制数字，最大也只能是 255 了
    maxParams uint8

    // 和下面的 children 对应，保留的子节点的第一个字符
    // 如上图中的 s 节点，这里保存的就是 eu （earch 和 upport）的首字母 
    indices   string

    // 当前节点的所有直接子节点
    children  []*node

    // 当前节点对应的 handler
    handle    Handle

    // 优先级，查找的时候会用到,表示当前节点加上所有子节点的数目
    priority  uint32
}
```

一共有`4`种类型的node：
-	`static`：静态节点, 如上图中的父节点 s （不包含 handler 的)
-	`root`：如果插入的节点是第一个, 那么是root节点
-	`catchAll`：有`*`匹配的节点
-	`param`：参数节点，比如上图中的 `:post` 节点

一开始树为空，注册路由的过程就是建树的过程

#### Tree

## 0x04 路由注册
路由注册的底层都是调用`Handle`方法，看下其实现，最后是调用`addRoute`方法添加路由：
```golang
// 通过给定的路径和方法，注册一个新的 handle 请求，对于 GET, POST, PUT, PATCH 和 DELETE
// 请求，相对应的方法，直接调用这个方法也是可以的，只需要多传入第一个参数即可。
// 官方给的建议是： 在批量加载或使用非标准的自定义方法时候使用。
// 
// 它有三个参数，第一个是请求方式（GET，POST……）， 第二个是路由的路径， 前两个参数都是字符串类型
// 第三个参数就是我们要注册的函数，比如上面的 Index 函数，可以追踪一下，在这个文件的最上面
// 有 Handle 的定义，它是一个函数类型，只要和这个 Handle 的签名一直的都可以作为参数
func (r *Router) Handle(method, path string, handle Handle) {

    // 验证路径是否是 / 开头的，如： /foo 就是可以的， foo 就会 panic，编译的时候机会出错
    if path[0] != '/' {
        panic("path must begin with '/' in path '" + path + "'")
    }

    // 因为 map 是一个指针类型，必须初始化才可以使用，这里做一个判断，
    // 如果从来没有注册过路由，要先初始化 tress 属性的 map
    if r.trees == nil {
        r.trees = make(map[string]*node)
    }

    // 因为路由是一个基数树，全部是从根节点开始，如果第一次调用注册方法的时候跟是不存在的，
    // 就注册一个根节点， 这里是每一种请求方法是一个根节点，会存在多个树。
    root := r.trees[method]

    // 根节点存在就直接调用，不存在就初始化一个
    if root == nil {
        // 需要注意的是，这里使用的 new 来初始化，所以 root 是一个指针。
        root = new(node)
        r.trees[method] = root
    }

    // 向 root 中添加路由，树的具体操作在后面单独去分析。
    root.addRoute(path, handle)
}
```

####	核心1：addRouter方法
入口是走根节点的`root.addRoute()`进入的，特别注意，此方法非线程安全，此方法的最终目的是把某个路由（`/a/bb/cccc`）插入Radix树：

node的`addRoute`方法分为两步：
1.	第一步是根据path和已有的前缀树的匹配，找到剩余的path需要插入的孩子节点的位置
2.	第二步是插入到该节点。当一颗树是空树时，增加第一条path的过程便退化为只有第二步

```golang
// addRoute 将传入的 handle 添加到路径中
func (n *node) addRoute(path string, handle Handle) {
    fullPath := path
    // 请求到这个方法，就给当前节点的权重 + 1
    n.priority++

    // 计算传入的路径参数的数量，即路径中包含:/*特殊字符的总数
    numParams := countParams(path)

    // 如果树不是空的
    // 判断的条件是当前节点的 path 的字符串长度和子节点的数量全部都大于 0
    // 就是说如果当前节点是一个空的节点，或者当前的节点是一个叶子节点，就直接
    // 进入 else 在当前节点下面添加子节点
    if len(n.path) > 0 || len(n.children) > 0 {
    // 定义一个 lable ，循环里面可以直接 break 到这里，适合这种嵌套的比较深的
    walk:
        for {
            // 如果传入的节点的最大参数的数量大于当前节点记录的数量，替换
            if numParams > n.maxParams {
                n.maxParams = numParams
            }

            // 查找最长的共同的前缀
            // 公共前缀不包含 ":" 或 "*"
            i := 0

            // 将最大值设置成长度较小的路径的长度
            max := min(len(path), len(n.path))

            // 循环，计算出当前节点和添加的节点共同前缀的长度
            for i < max && path[i] == n.path[i] {
                i++
            }

            // 如果相同前缀的长度比当前节点保存的 path 短
            // 比如当前节点现在的 path 是 sup ，添加的节点的 path 是 search
            // 它们相同的前缀就变成了 s ， s 比 sup 要短，符合 if 的条件，要做处理
            if i < len(n.path) {
                // 将当前节点的属性定义到一个子节点中，没有注释的属性不变，保持原样
                child := node{
                    // path 是当前节点的 path 去除公共前缀长度的部分
                    path:      n.path[i:],
                    wildChild: n.wildChild,
                    // 将类型更改为 static ，默认的，没有 handler 的节点
                    nType:     static,
                    indices:   n.indices,
                    children:  n.children,
                    handle:    n.handle,
                    // 权重 -1 
                    priority:  n.priority - 1,
                }

                // 遍历当前节点的所有子节点（当前节点变成子节点之后的节点），
                // 如果最大参数数量大于当前节点的数量，更新
                for i := range child.children {
                    if child.children[i].maxParams > child.maxParams {
                        child.maxParams = child.children[i].maxParams
                    }
                }

                // 在当前节点的子节点定义为当前节点转换后的子节点
                n.children = []*node{&child}

                // 获取子节点的首字母,因为上面分割的时候是从 i 的位置开始分割
                // 所以 n.path[i] 可以去除子节点的首字母，理论上去 child.path[0] 也是可以的
                // 这里的 n.path[i] 取出来的是一个 uint8 类型的数字（代表字符），
                // 先用 []byte 包装一下数字再转换成字符串格式
                n.indices = string([]byte{n.path[i]})

                // 更新当前节点的 path 为新的公共前缀
                n.path = path[:i]

                // 将 handle 设置为 nil
                n.handle = nil

                // 肯定没有参数了，已经变成了一个没有 handle 的节点了
                n.wildChild = false
            }

            // 将新的节点添加到此节点的子节点， 这里是新添加节点的子节点
            if i < len(path) {
                // 截取掉公共部分，剩余的是子节点
                path = path[i:]

                // 如果当前路径有参数
                // 如果进入了上面 if i < len(n.path) 这个条件，这里就不会成立了
                // 因为上一个 if 中将 n.wildChild 重新定义成了 false 
                // 什么情况会进入到这里呢 ? 
                // 1. 上面的 if 不生效，也就是说不会有新的公共前缀， n.path = i 的时候
                // 2. 当前节点的 path 是一个参数节点就是像这种的 :post
                // 就是定义路由时候是这种形式的： blog/:post/update
                // 
                if n.wildChild {
                    // 如果进入到了这里，证明这是一个参数节点，类似 :post 这种
                    // 不会这个节点进行处理，直接将它的子节点赋值给当前节点
                    // 比如： :post/ ，只要是参数节点，必有子节点，哪怕是
                    // blog/:post 这种，也有一个 / 的子节点
                    n = n.children[0]

                    // 又插入了一个节点，权限再次 + 1
                    n.priority++

                    // 更新当前节点的最大参数个数
                    if numParams > n.maxParams {
                        n.maxParams = numParams
                    }

                    // 更改方法中记录的参数个数，个人猜测，后面的逻辑还会用到
                    numParams--

                    // 检查通配符是否匹配
                    // 这里的 path 已经变成了去除了公共前缀的后面部分，比如 
                    // :post/update ， 就是 /update
                    // 这里的 n 也已经是 :post 这种的下一级的节点，比如 / 或者 /u 等等
                    // 如果添加的节点的 path >= 当前节点的 path && 
                    // 当前节点的 path 长度和添加节点的前面相同数量的字符是相等的， &&
                    if len(path) >= len(n.path) && n.path == path[:len(n.path)] &&
                        // 简单更长的通配符，
                        // 当前节点的 path >= 添加节点的 path ，其实有第一个条件限制，
                        // 这里也只有 len(n.path) == len(path) 才会成立，
                        // 就是当前节点的 path 和 添加节点的 path 相等 ||
                        // 添加节点的 path 减去当前节点的 path 之后是 /
                        // 例如： n.path = name, path = name 或 
                        // n.path = name, path = name/ 这两种情况
                        (len(n.path) >= len(path) || path[len(n.path)] == '/') {

                        // 跳出当前循环，进入下一次循环
                        // 再次循环的时候 
                        // 1. if i < len(n.path) 这里就不会再进入了，现在 i == len(n.path)
                        // 2. if n.wildChild 也不会进入了，当前节点已经在上次循环的时候改为 children[0]
                        continue walk
                    } else {
                        // 当不是 n.path = name, path = name/ 这两种情况的时候，
                        // 代表通配符冲突了，什么意思呢？
                        // 简单的说就是通配符部分只允许定义相同的或者 / 结尾的
                        // 例如：blog/:post/update，再定义一个路由 blog/:postabc/add，
                        // 这个时候就会冲突了，是不被允许的，blog 后面只可以定义 
                        // :post 或 :post/ 这种，同一个位置不允许使用多种通配符
                        // 这里的处理是直接 panic 了，如果想要支持，可以尝试重写下面部分代码

                        // 下面做的事情就是组合 panic 用到的提示信息
                        var pathSeg string

                        // 如果当前节点的类型是有*匹配的节点
                        if n.nType == catchAll {
                            pathSeg = path
                        } else {
                            // 如果不是，将 path 做字符串分割 
                            // 这个是通过 / 分割，最多分成两个部分,然后取第一部分的值
                            // 例如： path = "name/hello/world"
                            // 分割两部分就是 name 和 hello/world , pathSeg = name
                            pathSeg = strings.SplitN(path, "/", 2)[0]
                        }

                        // 通过传入的原始路径来处理前缀, 可以到上面看下，方法进入就定义了这个变量
                        // 在原始路径中提取出 pathSeg 前面的部分在拼接上 n.path
                        // 例如： n.path = ":post" , fullPath="/blog/:postnew/add"
                        // 这时的 prefix = "/blog/:post"
                        prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path

                        // 最终的提示信息就会生成类似这种： 
                        // panic: ':postnew' in new path '/blog/:postnew/update/' \
                        // conflicts with existing wildcard ':post' in existing \
                        // prefix '/blog/:post'
                        // 就是说已经定义了 /blog/:post 这种规则的路由，
                        // 再定义 /blog/:postnew 这种就不被允许了
                        panic("'" + pathSeg +
                            "' in new path '" + fullPath +
                            "' conflicts with existing wildcard '" + n.path +
                            "' in existing prefix '" + prefix +
                            "'")
                    }
                }

                // 如果没有进入到上面的参数节点，当前节点不是一个参数节点 :post 这种
                c := path[0]

                // slash after param
                // 如果当前节点是一个参数节点，如 /:post && 是 / 开头的 && 只有一个子节点
                if n.nType == param && c == '/' && len(n.children) == 1 {
                    // /:post 这种节点不做处理，直接拿这个节点的子节点去匹配
                    n = n.children[0]
                    // 权重 + 1 ， 因为新的节点会变成这个节点的子节点
                    n.priority++

                    // 结束当前循环，再次进行匹配
                    // 比如： /:post/add 当前节点是 /:post 有个子节点 /add
                    // 新添加的节点是 /:post/update ，再次循环的时候将会用
                    // /add 和 /update 进行匹配， 就相当于是两次新的匹配，
                    // 由于 node 本身是一个指针，所以它们一直都会在 /:post 下面搞事情
                    continue walk
                }

                // 检查添加的 path 的首字母是否保存在在当前节点的 indices 中
                for i := 0; i < len(n.indices); i++ {
                    // 如果存在
                    if c == n.indices[i] {
                        // 这里处理优先级和排序的问题，把这个方法看完再去查看这个方法干了什么
                        i = n.incrementChildPrio(i)

                        // 将当前的节点替换成它对应的子节点
                        n = n.children[i]

                        // 结束当前循环，继续下一次
                        // 此时的当前 node n，已经变成了它的子字节，下次循环就从这个子节点开始处理了
                        // 比如：
                        // 当前节点是 s , 包含两个节点 earch 和 upport
                        // 这时 indices 就会有字母 e 和 u 并且是和子节点 earch 和 uppor 相对应
                        // 新添加的节点如果叫 subject ， 与当前节点匹配去除公共前缀 s 后， 就变成了
                        // ubject ，这时 n = upport 这个节点了，path = ubject 
                        // 下一次循环就拿 upport 这个 n 和 ubject 这个 path 去开始下次的匹配
                        continue walk
                    }
                }

                // 如果上面 for 中也没有匹配上，就将新添加的节点插入

                // 如果第一个字符不是通配符 : 并且不是 *
                if c != ':' && c != '*' {
                    // []byte for proper unicode char conversion, see #65
                    // 将 path 的首字母 c 拼接到 indices 中
                    n.indices += string([]byte{c})

                    // 初始化一个新的节点
                    child := &node{
                        maxParams: numParams,
                    }

                    // 将这个新初始化的节点添加到当前节点的子节点中
                    n.children = append(n.children, child)

                    // 这个应该还是在处理优先级，一会再去看这个方法
                    n.incrementChildPrio(len(n.indices) - 1)

                    // 将当前节点替换成新生成的节点
                    n = child
                }

                // 用当前节点发起插入子节点的动作
                // 注意这个 n 已经替换成了上面新初始化的 child 了，只初始化了 maxParams
                // 属性，相当于是一个空的节点。insertChild 这个方法一会再去具体查看，
                // 现在只要知道它在做具体的插入子节点的动作就行，先把这个方法追踪完。
                n.insertChild(numParams, path, fullPath, handle)
                return

            // 如果公共前缀和当前添加的路径长度相等
            } else if i == len(path) { // Make node a (in-path) leaf
                // 如果当前节点不是 static 类型的， 就是已经注册了 handle 的节点
                // 就证明已经注册过这个路由，直接 panic 了
                if n.handle != nil {
                    panic("a handle is already registered for path '" + fullPath + "'")
                }

                // 如果是 nil 的，就是没有注册，证明这个节点是之前添加别的节点时候拆分出来的共同前缀
                // 例如： 添加过两个截点 /submit 和 /subject ，处理后就会有一个 static 的前缀 sub
                // 这是再添加一个 /sub 的路由，就是这里的情况了。

                n.handle = handle
            }

            // 这个新的节点被添加了， 出现了 return ， 只有出现这个才会正常退出循环，一次添加完成。
            return
        }
    } else { // Empty tree
        // 如果 n 是一个空格的节点，就直接调用插入子节点方法
        n.insertChild(numParams, path, fullPath, handle)

        // 并且它只有第一次插入的时候才会是空的，所以将 nType 定义成 root
        n.nType = root
    }
}
```


####	子方法1：incrementChildPrio
`incrementChildPrio`方法主要处理节点之前的关系，添加或者修改已经存在的，拼接出树的结构，真正写入插入节点的数据是在方法`子方法1：incrementChildPrio`处理完关系后，调用 `insertChild` 方法完成。
```golang
// increments priority of the given child and reorders if necessary
// 通过之前两次的调用，我们知道，这个 pos 都是 n.indices 中指定字符的索引，也就是位置
func (n *node) incrementChildPrio(pos int) int {
    // 因为 children 和 indices 是同时添加的，所以索引是相同的
    // 可以通过 pos 代表的位置找到， 将对应的子节点的优先级 + 1
    n.children[pos].priority++
    prio := n.children[pos].priority

    // 调整位置（向前移动）
    newPos := pos

    // 这里的操作就是将 pos 位置的子节点移动到最前面，增加优先级，比如：
    // 原本的节点是 [a, b, c, d], 传入的是 2 ,就变成 [c, a, b, d]
    // 这只是个示例，实际情况还要考虑 n.children[newPos-1].priority < prio ，不一定是移动全部
    for newPos > 0 && n.children[newPos-1].priority < prio {
        // swap node positions
        n.children[newPos-1], n.children[newPos] = n.children[newPos], n.children[newPos-1]

        newPos--
    }

    // build new index char string
    // 重新组织 n.indices 这个索引字符串，和上面做的是相同的事，只不过是用切割 slice 方式进行的
    // 从代码本身来说上面的部分也可以通过切割 slice 完成，不过移动应该是比切割组合 slice 效率高吧
    // 感觉是这样，没测试过，有兴趣的可以测试下。
    if newPos != pos {
        n.indices = n.indices[:newPos] + // unchanged prefix, might be empty
            n.indices[pos:pos+1] + // the index char we move
            n.indices[newPos:pos] + n.indices[pos+1:] // rest without char at 'pos'
    }

    // 最后返回的是调整后的位置，保证外面调用这个方法之后通过索引找到的对应的子节点是正确的
    return newPos
}
```

####	字方法2：insertChild
`insertChild`最终在叶子节点写入注册的数据（path 和 handler)
```GOLANG
// 传入的几个参数：
// numParams 参数个数
// path 插入的子节点的路径
// fullPath 完整路径，就是注册路由时候的路径，没有被处理过的
// 注册路由对应的 handle 函数
func (n *node) insertChild(numParams uint8, path, fullPath string, handle Handle) {
    var offset int // 已经处理过的路径的所有字节数

    // find prefix until first wildcard (beginning with ':'' or '*'')
    // 查找前缀，知道第一个通配符（ 以 ':' 或 '*' 开头
    // 就是要将 path 遍历，提取出参数
    // 只要不是通配符开头的就不做处理，证明这个路由是没有参数的路由
    for i, max := 0, len(path); numParams > 0; i++ {
        c := path[i]

        // 如果不是 : 或 * 跳过本次循环，不做任何处理
        if c != ':' && c != '*' {
            continue
        }

        // 查询通配符后面的字符，直到查到 '/' 或者结束
        end := i + 1

        for end < max && path[end] != '/' {
            switch path[end] {
            // 通配符后面的名称不能包含 : 或 * ， 如 ::name 或 :*name 不允许定义
            case ':', '*':
                panic("only one wildcard per path segment is allowed, has: '" +
                    path[i:] + "' in path '" + fullPath + "'")
            default:
                end++
            }
        }

        // 检查通配符所在的位置，是否已经有子节点，如果有，就不能再插入
        // 例如： 已经定义了 /hello/name ， 就不能再定义 /hello/:param
        if len(n.children) > 0 {
            panic("wildcard route '" + path[i:end] +
                "' conflicts with existing children in path '" + fullPath + "'")
        }

        // 检查通配符是否有一个名字
        // 上面定义 end = i+1 ， 后面的 for 又执行了 ++ 操作，所以通配符 : 或 * 后面最少
        // 要有一个字符, 如： :a 或 :name ， :a 的时候 end 就是 i+2
        // 所以如果 end - i < 2 ，就是通配符后面没有对应的名称， 就会 panic
        if end-i < 2 {
            panic("wildcards must be named with a non-empty name in path '" + fullPath + "'")
        }

        // 如果 c 是 : 通配符的时候
        if c == ':' { // param
            // split path at the beginning of the wildcard
            // 从 offset 的位置，到查询到通配符的位置分割 path
            // 并把分割出来的路径定义到节点的 path 属性
            if i > 0 {
                n.path = path[offset:i]
                // 开始的位置变成了通配符所在的位置
                offset = i
            }

            // 将参数部分定义成一个子节点
            child := &node{
                nType:     param,
                maxParams: numParams,
            }

            // 用新定义的子节点初始化一个 children 属性
            n.children = []*node{child}
            // 标记上当前这个节点是一个包含参数的节点的节点
            n.wildChild = true

            // 将新创建的节点定义为当前节点，这个要想一下，到这里这种操作已经有不少了
            // 因为一直都是指针操作，修改都是指针的引用，所以定义好的层级关系不会被改变
            n = child
            // 新的节点权重 +1
            n.priority++
            // 最大参数个数 - 1
            numParams--

            // if the path doesn't end with the wildcard, then there
            // will be another non-wildcard subpath starting with '/'
            // 这个 end 有可能是结束或者下一个 / 的位置
            // 如果小于路径的最大长度，代表还包含子路径（也就是说后面还有子节点）
            if end < max {
                // 将通配符提取出来，赋值给 n.path， 现在 n.path 是 :name 这种格式的字符串
                n.path = path[offset:end]
                // 将起始位置移动到 end 的位置
                offset = end

                // 定义一个子节点，无论后面还有没有子节点 :name 这种格式的路由后面至少还有一个 /
                // 因为参数类型的节点不会保存 handler
                child := &node{
                    maxParams: numParams,
                    priority:  1,
                }

                // 将初始化的子节点赋值给当前节点
                n.children = []*node{child}

                // 当前节点又变成了新的子节点（到此时的 n 的身份已经转变了几次了，看这段代表的时候
                // 脑中要有一颗树，实在想不出来的话可以按照开始的图的结构，将节点的变化记录到一张纸上，
                // 然后将每一次的转变标记出来，就完全能明白了）
                n = child

                // 如果进入倒了这个 if ，执行到这里，就还会进入到下一次循环，将可能生成出来的参数再次去匹配
            }

            // 如果走到了这里，并且是循环的最后一次，就是已经将当前节点 n 定义成了叶子节点
            // 就可以进入到最下面代码部分，进行插入了
            // （上面的注释说明的位置是 else 上面，不是为下面的 else 做的注释）

        // 进入到 else ，就是包含 * 号的路由了 
        } else { // catchAll
            // 这里的意思是， * 匹配的路径只允许定义在路由的最后一部分 
            // 比如 : /hello/*world 是允许的， /hello/*world/more 这种就会 painc
            // 这种路径就是会将 hello/ 后面的所有内容变成 world 的变量
            // 比如地址栏输入： /hello/one/two/more ，获取到的参数 world = one/twq/more
            // 不会再将后面的 / 作为路径处理了
            if end != max || numParams > 1 {
                panic("catch-all routes are only allowed at the end of the path in path '" + fullPath + "'")
            }

            // 这种情况是，新定义的 * 通配符路由和其他已经定义的路由冲突了
            // 例如已经定义了一个 /hello/bro ， 又定义了一个 /hello/*world ，此时就会 panic 了
            if len(n.path) > 0 && n.path[len(n.path)-1] == '/' {
                panic("catch-all conflicts with existing handle for the path segment root in path '" + fullPath + "'")
            }

            // currently fixed width 1 for '/'
            i--
            // 这里是查询通配符前面是否有 / 没有 / 是不行的，panic
            if path[i] != '/' {
                panic("no / before catch-all in path '" + fullPath + "'")
            }

            n.path = path[offset:i]

            // first node: catchAll node with empty path
            // 后面的套路基本和之前看到的类似，就是定义一个子节点，保存通配符前面的路径，
            // 有变化的就是将 nType 定义为 catchAll，就是说代表这是一个  * 号匹配的路由
            child := &node{
                wildChild: true,
                nType:     catchAll,
                maxParams: 1,
            }
            n.children = []*node{child}
            n.indices = string(path[i])
            n = child
            n.priority++

            // second node: node holding the variable
            // 将下面的节点再添加到上面，不过 * 号路由不会再有下一级的节点了，因为它会将后面的
            // 的所有内容当做变量，即使它是个 / 符号
            child = &node{
                path:      path[i:],
                nType:     catchAll,
                maxParams: 1,
                handle:    handle,
                priority:  1,
            }
            n.children = []*node{child}

            // 这里 return 了，看下上面，因为已经将 handle 保存，查到了叶子节点，所以就直接结束了当前方法
            return
        }
    }

    // insert remaining path part and handle to the leaf
    // 这里给所有的处理的完成后的节点（不包含 * 通配符方式）的 handle 和 path 赋值
    n.path = path[offset:]
    n.handle = handle
}
```

## 0x05 路由查找
前文`ServeHTTP`方法中，使用`getValue`来查找路由对应的handler，如下：

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
    //.....
}
```

####	getValue方法
```golang
// 通过传入的参数的路径，寻找指定的路由
// 参数： 字符串类型的路径
// 返回值：
// Handle 通过路径找到的 handler ， 如果没有找到，会返回一个 nil
// Params 如果路由注册的是参数路由，这里会将参数及值返回，是一个 Param 结构的 slice 
// tsr 是一个 boolean 类型的标志，告诉调用者这个路由是否可以被重定向，配合 Router
// 里面的 RedirectTrailingSlash 属性
func (n *node) getValue(path string) (handle Handle, p Params, tsr bool) {
walk: // outer loop for walking the tree
    for {
        // 如果要找的路径长度大于节点的长度
        // 入了根目录 / 一般情况都会满足这个条件，进入到这里走一圈
        if len(path) > len(n.path) {
            // 如果寻找的路径和节点保存的路径有相同的前缀
            if path[:len(n.path)] == n.path {
                // 将寻找路径的前缀部分去除
                path = path[len(n.path):]
                // 如果当前的节点不包含通配符（子节点有 ： 或者 * 这种的），就可以直接去子节点继续搜索。
                if !n.wildChild {
                    // 通过查找路径的首字符，快速找到包含这个字符的子节点（性能提升的地方就体现出来了）
                    c := path[0]
                    for i := 0; i < len(n.indices); i++ {
                        // 如果找到了，就继续去 indices 中对应的子节点去找
                        // 并且进入下一次循环
                        if c == n.indices[i] {
                            n = n.children[i]
                            continue walk
                        }
                    }

                    // 如果进入到了这里，代表没有找到对应的子节点

                    // 如果寻找路径正好是 / ，因为去除了公共路径
                    // 假设寻找的是 /hello/ ， n.path = hello， 这时 path 就是 /
                    // 当前的节点如果注册了 handle ，就证明这个是一个路由，tsr 就会变成 true
                    // 调用者就知道可以将路由重定向到 /hello 了，下次再来就可以找到这个路由了
                    // 可以去 Router.ServeHTTP 方法看下，是不是会发起一个重定向
                    tsr = (path == "/" && n.handle != nil)

                    // 返回，此时会返回 (nil,[],true|false) 这 3 个值
                    return
                }

                // 处理是通配符节点（包含参数或*）的情况
                // 如果当前节点是通配符节点，子节点肯定只有一个，因为 :name 这种格式的只允许注册一个
                // 所以可以把当前节点替换成 :name 或 *name 这种子节点
                n = n.children[0]
                switch n.nType {
                // 当子节点是参数类型的时候，:name 这种格式的
                case param:
                    // find param end (either '/' or path end)
                    // 查找出寻找路径中到结束或者 / 以前的部分所在的位置
                    // 比如是 :name 或者 :name/more ，end 就会等于 4
                    end := 0
                    for end < len(path) && path[end] != '/' {
                        end++
                    }

                    // save param value
                    // 保存参数的值， 因为参数是一个 slice ，使用前要初始化一下
                    if p == nil {
                        // lazy allocation
                        p = make(Params, 0, n.maxParams)
                        // 现在 p 的值是 []Param
                    }
                    // 下面两行是将 slice 的长度扩展
                    // 如果第一次来， p 是一个空格， i = 0
                    // p = p[:0+1] p 取的是一个位置 0 到 1 的切片，因为切片前开后闭，
                    // 所以它的长度就变成了 1， 下面就可以向切片添加一个值了
                    // 第二次来就是在原有的长度再扩展 1 的长度，然后再添加一个值，
                    // 这样这个参数就会永远是一个有效的最小长度，应该可以提高性能 ?
                    // 不清楚作者为什么这样做，直接 append 不可以吗？ 有兴趣的可以去确定下。
                    i := len(p)
                    p = p[:i+1] // expand slice within preallocated capacity

                    // 这个就是给 Parm 赋值， Key 就是去掉 : 后的值，比如 :name 就是 name
                    // Value 就是路径到结束位置 end 的值
                    p[i].Key = n.path[1:]
                    p[i].Value = path[:end]

                    // we need to go deeper!
                    // 上面已经获取了第一个值，但是有可能还会有两个或者更多，所以还要继续处理
                    // 如果之前找到第一个值的位置比寻找路径的长度小，就是还有 :name 后面还有内容需要去匹配
                    if end < len(path) {
                        // 如果还存在子节点， 就将 path 和 node 替换成子节点的，跳出当前循环再找一遍
                        if len(n.children) > 0 {
                            path = path[end:]
                            n = n.children[0]
                            continue walk
                        }

                        // ... but we can't
                        tsr = (len(path) == end+1)
                        // 这里返回的是 (nil, [Parm{name, 路径上的值}], true|false)
                        return
                    }

                    // 如果当前节点保存了 handle ，就将 handle 返回给调用者，去发起业务逻辑
                    if handle = n.handle; handle != nil {
                        return
                    } else if len(n.children) == 1 {
                        // No handle found. Check if a handle for this path + a
                        // trailing slash exists for TSR recommendation
                        // 确认它是否有一个 / 的子节点，如果有 tsr = true
                        n = n.children[0]
                        tsr = (n.path == "/" && n.handle != nil)
                    }

                    // 这里返回的是 [nil, [找到的参数...], true|false]
                    return

                // 如果是 * 这种方式的路由, 就是直接处理参数，并返回，因为它肯定是最后一个节点
                // 所以也不会出现子节点，及时路径上后面根再多的 / 也都只是参数的值
                case catchAll:
                    // save param value
                    if p == nil {
                        // lazy allocation
                        p = make(Params, 0, n.maxParams)
                    }
                    i := len(p)
                    p = p[:i+1] // expand slice within preallocated capacity
                    p[i].Key = n.path[2:]
                    p[i].Value = path

                    handle = n.handle
                    return

                default:
                    // 没想到什么情况会出现无效的类型， 如果出现了就直接 panic
                    // 现在这个 panic 不会导致服务器崩溃，因为在 ServeHTTP 中做了 Recovery
                    panic("invalid node type")
                }
            }
        // 如果查询的路径和节点保存的路径相同
        } else if path == n.path {
            // We should have reached the node containing the handle.
            // Check if this node has a handle registered.
            // 检查下这个节点是否包含 handle ，如果包含就返回
            // 会有不包含的情况，比如恰巧这个就是一个相同前缀的节点
            if handle = n.handle; handle != nil {
                return
            }

            // 如果查找的路径是 / 又不是根节点，还是个参数节点，就允许重定向
            if path == "/" && n.wildChild && n.nType != root {
                tsr = true
                return
            }

            // No handle found. Check if a handle for this path + a
            // trailing slash exists for trailing slash recommendation
            // 如果没有找到，但是有一个 / 的子节点，也允许它重定向
            // 就是这个意思，比如定义了一个 /hello/ 路由，但是访问的是 /hello
            // 也允许它重定向到 /hello/ 上
            for i := 0; i < len(n.indices); i++ {
                if n.indices[i] == '/' {
                    n = n.children[i]
                    tsr = (len(n.path) == 1 && n.handle != nil) ||
                        (n.nType == catchAll && n.children[0].handle != nil)
                    return
                }
            }

            return
        }

        // Nothing found. We can recommend to redirect to the same URL with an
        // extra trailing slash if a leaf exists for that path
        // 是否允许重定向的一个验证
        tsr = (path == "/") ||
            (len(n.path) == len(path)+1 && n.path[len(path)] == '/' &&
                path == n.path[:len(n.path)-1] && n.handle != nil)
        return
    }
}
```

## 0x06 查找算法


##	0x07	举例



####	

##	0x08	其他一些细节

1、编译验证<br>
如下面代码：
```GOLANG
var _ http.Handler = New()
```
上面代码通过 `New()` 函数初始化一个 `Router` ，并且指定 `Router` 实现了 http.Handler 接口 ，如果 `New()` 没有实现 `http.Handler` 接口，在编译的时候就会报错了。这里只是为了验证一下， `New()` 函数的返回值并不需要，所以就把它赋值给 `_`；常用于编译检查


## 0x09 参考

- [基数树 (Radix Tree)](https://juejin.cn/post/6933244263241089037)
- [Radix tree](https://en.wikipedia.org/wiki/Radix_tree)
- [hashicorp：An immutable radix tree implementation in Golang](https://github.com/hashicorp/go-immutable-radix)
- [6.2. router 请求路由](https://books.studygolang.com/advanced-go-programming-book/ch6-web/ch6-02-router.html)
- [Wikipedia - 基数树](https://zh.m.wikipedia.org/zh-hans/%E5%9F%BA%E6%95%B0%E6%A0%91)
- [httprouter 源码分析](https://learnku.com/articles/27591)
- [gin框架之路由前缀树初始化分析](https://mp.weixin.qq.com/s/lLgeKMzT4Q938Ij0r75t8Q)