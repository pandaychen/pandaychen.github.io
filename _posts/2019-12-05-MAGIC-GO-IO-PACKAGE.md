---
layout:     post
title:      神奇的Golang-IO包（计划）
subtitle:   
date:       2020-01-01
author:     pandaychen
header-img: 
catalog: true
tags:
    - Golang
---

##	0x01	Golang的io包介绍
在 golang 中，标准库中的package设计的基本原则是职责内聚。通常开发者可以使用自定义Wrapper的方式来封装标准库package中的interface接口，亦或在此基础上扩展，添加自定义的功能。
但是有一点，返回值必须封装的方法保持一致。

##  0x02    神奇的io.Copy
做过服务端开发的同学一定都写过代理 Proxy，代理的本质，是转发两个相同方向路径上的stream（数据流）。例如，一个`A-->B-->C`的代理模式，B作为代理，需要完成下面两件事情：
1.	读取从`A--->B`的数据，转发到`B--->C`
2.	读取从`C--->B`的数据，转发到`B--->A`

在golang中，只需要`io.Copy()`就能轻而易举的完成上面的事情，其[实现代码](https://golang.org/src/io/io.go?s=12796:12856#L353)如下所示：
```go
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	if buf == nil {
		size := 32 * 1024
		if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
			if l.N < 1 {
				size = 1
			} else {
				size = int(l.N)
			}
		}
		buf = make([]byte, size)
	}
	for {
		nr, er := src.Read(buf)
		if nr > 0 {
			nw, ew := dst.Write(buf[0:nr])
			if nw > 0 {
				written += int64(nw)
			}
			if ew != nil {
				err = ew
				break
			}
			if nr != nw {
				err = ErrShortWrite
				break
			}
		}
		if er != nil {
			if er != EOF {
				err = er
			}
			break
		}
	}
	return written, err
}
```


##  0x03	golang  io.Pipe的妙用


##	0x04	参考
-	[Go编程技巧--io.Reader/Writer](https://segmentfault.com/a/1190000013358594)