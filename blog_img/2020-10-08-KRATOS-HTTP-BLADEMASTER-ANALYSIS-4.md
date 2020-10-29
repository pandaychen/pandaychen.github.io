---
layout:     post
title:      Kratos 源码分析：CGI 框架 BM （四）
subtitle:   分析基于 gin 改造的框架 blademaster：路由树构造算法
date:       2020-10-08
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---


##  0x00    前言
从前几篇文章已知，bm 框架也是使用了 [Trie 树](https://zh.wikipedia.org/wiki/Trie) 作为路由 path 的存储结构，本文分析根据设置的路由请求构建路由 Trie 树的代码实现 [细节](https://github.com/go-kratos/kratos/blob/master/pkg/net/http/blademaster/tree.go)。


##	0x01	路由
在 `bm.Engine` 成员中，`trees` 是一个数组，针对框架支持的每一种方法，都会创建一个节点。例如 GET、POST 是 trees 的两个元素。
在 Gin 框架中，路由规则被分成了最多 9 棵前缀树，每一个 HTTP Method 对应一棵「前缀树」，树的节点按照 URL 中的 / 符号进行层级划分，URL 支持 :name 形式的名称匹配，还支持 *subpath 形式的路径通配符 。

每个节点都会挂接若干请求处理函数构成一个请求处理链 HandlersChain。当一个请求到来时，在这棵树上找到请求 URL 对应的节点，拿到对应的请求处理链来执行就完成了请求的处理。

```golang
type Engine struct {
		......
		trees            methodTrees
		// 不同的 HTTP 方法（GET/POST/DELETE 等）都对应一棵树
		......
}

type methodTrees []methodTree

// 方法树的定义
type methodTree struct {
        method string
        root   *node
}
```

####	node 的定义
每个 tree 的节点代表了一条实际注册的路由 PATH，节点（node）的定义如下。根节点 root 通过 node 及包含 children 的 node 数组的结构形成了整个框架的路由树结构。
```golang
type node struct {
        path      string	// 当前节点的 path
        indices   string	//
        children  []*node
        handlers  HandlersChain
        priority  uint32
        nType     nodeType
        maxParams uint8
        wildChild bool
        fullPath  string
}
```

path：表示当前节点的 path
indices：通常情况下维护了 children 列表的 path 的各首字符组成的 string，之所以是通常情况，是在处理包含通配符的 path 处理中会有一些例外情况；
priority：代表了有几条路由会经过此节点，用于在节点进行排序时使用；
nType：是节点的类型，默认是 static 类型，还包括了 root 类型，对于 path 包含冒号通配符的情况，nType 是 param 类型，对于包含 * 通配符的情况，nType 类型是 catchAll 类型；wildChild：默认是 false，当 children 是 通配符类型时，wildChild 为 true；
fullPath：是从 root 节点到当前节点的全部 path 部分；如果此节点为终结节点 handlers 为对应的处理链，否则为 nil；maxParams 是当前节点到各个叶子节点的包含的通配符的最大数量。




##	0x03	路由初始化
路由初始化过程中构建 Trie 树的核心流程如下：
![add-trietree-route](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gin-trietree-add-route.png)


##  0x05    参考
-   [blademaster 官方文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster.md)
-   [blademaster](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/blademaster-mid.md)
-   [go 语言框架 gin 的中文文档](https://github.com/skyhee/gin-doc-cn)
