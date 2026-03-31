---
layout:     post
title:      基于 Golang HTTP 传输文件的一些思考
subtitle:
date:       2024-01-15
author:     pandaychen
catalog:    true
tags:
    - Golang
---


##  0x00    前言

####    一般做法
如下，将整个文件读到 buf 里，不适用于大文件，使用 `mime/multipart` 包处理

```GO
buf := new(bytes.Buffer)
writer := multipart.NewWriter(buf)
defer writer.Close()
part, err := writer.CreateFormFile("myFile", "foo.txt")
if err != nil {
    return err
}
file, err := os.Open(name)
if err != nil {
    return err
}
defer file.Close()
if _, err = io.Copy(part, file); err != nil {
    return err
}
http.Post(url, writer.FormDataContentType(), buf)
```

####    方法 2：io.Pipe()
参考前文 [神奇的 Golang-IO 包](https://pandaychen.github.io/2020/01/01/MAGIC-GO-IO-PACKAGE/) 提到的 `io.Pipe` 方法，将文件传输强行限制在一个管道中：

```go
r, w := io.Pipe()
m := multipart.NewWriter(w)
go func() {
    defer w.Close()
    defer m.Close()
    part, err := m.CreateFormFile("myFile", "foo.txt")
    if err != nil {
        return
    }
    file, err := os.Open(name)
    if err != nil {
        return
    }
    defer file.Close()
    if _, err = io.Copy(part, file); err != nil {
        return
    }
}()
http.Post(url, m.FormDataContentType(), r)
```
在这个示例中，`io.Pipe` 充当了一个同步的内存管道，它在两个 goroutine 之间传输数据。当主 goroutine 从管道的读取端读取数据时，新 goroutine 将从文件中读取数据并将数据写入管道的写入端，此方法的优点是文件数据的读取和上传是并发进行的，此外由于使用了内存管道，不需要将整个文件加载到内存中，从而减少了内存消耗

如果抓包看，客户端的 HTTP 头部可能是这样，注意 `Transfer-Encoding: chunked`，表示服务器将使用分块传输编码（chunked transfer encoding）来传输响应体数据。分块传输编码是一种动态生成响应内容的方法，允许服务器在不知道整个响应体大小的情况下开始发送数据。这对于处理大文件或生成内容需要很长时间的响应非常有用

```TEXT
POST / HTTP/1.1
...
Transfer-Encoding: chunked
Accept-Encoding: gzip
Content-Type: multipart/form-data; boundary=....
User-Agent: Go-http-client/1.1
```

##  0x02     multipart/form-data 方式



##  0x03    参考
-   [使用 multipart/form-data 实现文件的上传与下载](https://tonybai.com/2021/01/16/upload-and-download-file-using-multipart-form-over-http/)
-   [Sending big file with minimal memory in Golang](https://medium.com/@owlwalks/sending-big-file-with-minimal-memory-in-golang-8f3fc280d2c)