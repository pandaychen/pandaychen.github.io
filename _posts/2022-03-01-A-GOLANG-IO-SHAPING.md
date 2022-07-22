---
layout:     post
title:      Golang Io shaping
subtitle:   基于限流器的流量整形实现
date:       2022-03-01
author:     pandaychen
header-img:
catalog: true
tags:
    - Golang
---

##  0x00    前言
一个现实的问题，在调用 `io.Copy()` 进行数据传输时，能够有效的控制传输的速率吗？答案是可以，原生库提供了 `io.LimitReader` 方法，满足这种场景

`Reader` 使用指针接收者，`Writer` 使用值接收者

##  0x01    `io.LimitReader` 分析
`io.LimitReader` 的 [实现](https://cs.opensource.google/go/go/+/go1.18.4:src/io/io.go;l=464) 如下，从功能上看，`LimitedReader` 是限制了读的最大字节数。

`LimitReader(r Reader, n int64)`：Reader 返回一个内容长度受限的 Reader，当从这个 Reader 中读取了 `n` 个字节一会就会遇到 `EOF`，该机制提供了一个保护的功能，其它和普通的 `io.Reader` 没有两样。

```golang
//初始化LimitReader
func LimitReader(r Reader, n int64) Reader { 
    return &LimitedReader{r, n}
}

type LimitedReader struct {
  R Reader // underlying reader
  N int64  // max bytes remaining
}

func (l *LimitedReader) Read(p []byte) (n int, err error) {
  if l.N <= 0 {
    return 0, EOF
  }
  if int64(len(p)) > l.N {
    //最大只能读到N长度
    p = p[0:l.N]
  }
  //此时，len(p)必然是<=N的
  n, err = l.R.Read(p)

  //每次Read完之后，都更新l.N的值？
  l.N -= int64(n)
  return
}
```

疑问：使用 `io.LimitedReader`封装器，你可以从底层Reader中读取设定的字节数，因为它创建了一个新的 `io.Reader`限于固定数量的`N` 字节，那么如果数据超过了`N`字节，怎么处理呢？


##  参考
-   [从 io.Reader 中读数据](https://colobu.com/2019/02/18/read-data-from-net-Conn/)
