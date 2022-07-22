---
layout:     post
title:      Golang 中的 sync.Pool
subtitle:   Golang 优化系列
date:       2020-03-11
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Golang
---

##  前言

##  实现复用 []byte
```golang
const BuffSize = 10 * 1024

var buffPool sync.Pool

func GetBuff() *bytes.Buffer {
	var buffer *bytes.Buffer
	item := buffPool.Get()
	if item == nil {
		var byteSlice []byte
		byteSlice = make([]byte, 0, BuffSize)
		buffer = bytes.NewBuffer(byteSlice)

	} else {
		buffer = item.(*bytes.Buffer)
	}
	return buffer
}

func PutBuff(buffer *bytes.Buffer) {
	buffer.Reset()
	buffPool.Put(buffer)
}

func main(){
	var fy_buf bytes.Buffer
	PutBuff(&fy_buf)
	p:=GetBuff()
	p.WriteString("aaaaa")
	fmt.Println(p.String())
}
```

##  参考
-   [Go 1.13 中 sync.Pool 是如何优化的?](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/)