---
layout: post
title: Golang 闭包与递归：介绍与应用场景
subtitle: Golang closure
date: 2023-03-25
author: pandaychen
catalog: true
tags:
  - Golang
  - 闭包
---

## 0x00 前言
笔者近期项目中，实现了通过 JSON 协议模拟会话目录及文件的树型结构的操作，用到了 Golang 的递归与闭包。闭包是指内层函数引用了外层函数中的变量或称为引用了自由变量的函数，其返回值也是一个函数（方法），先看例子：
```golang
func outer(x int) func(int) int {
        return func(y int) int {
                return x + y
        }
}

func main() {
        f := outer(10)
        fmt.Println(f(100))
}
```

golang 的闭包有如下特点：

- 闭包能够访问外层代码中的变量
- `for range` 模式与 goroutine 同时执行
- 所有的 goroutine 操作的变量都是直接操作外层代码的变量，而外层代码中的变量的值取决于循环执行的节点


而递归没啥好说的，唯一的要点是递归一定要有退出条件

##  0x01 应用场景

`path/filepath` 包的 `Walk` 方法是典型的应用场景：
```golang
Walk(root stirng, walkFn WalkFunc) error
```

> walk 方法会遍历 root 下的所有文件 (包含 root) 并对每一个目录和文件都调用 `walkFunc` 方法。在访问文件和目录时发
> 生的错误都会通过 error 参数传递给 `WalkFunc` 方法。文件是按照词法顺序进行遍历的，这个通常让输出更漂亮，但是也会导致处理非常大的目录时效率会降低

`Walk` 需要传入一个自定义的 `WalkFunc` 方法，用于在递归遍历目录中处理各个节点，其 [定义](https://cs.opensource.google/go/go/+/refs/tags/go1.20.2:src/path/filepath/path.go;l=443;drc=06264b740e3bfe619f5e90359d8f0d521bd47806) 如下：

```golang
type WalkFunc func(path string, info fs.FileInfo, err error) error
```

先看两个 `WalkFunc` 的实现：
```golang
func explorer(path string, info os.FileInfo, err error) error {
        if err != nil {
                return err
        }
        if info.IsDir() {
                fmt.Printf("dir: %s\n", path)
        } else {
                fmt.Printf("file: %s\n", path)
        }
        return nil
}

func main() {
        filepath.Walk("/data/", explorer)
}
```

```GOLANG
type Walker struct {
    directories []string
    files       []string
}

func main() {
    walker := new (Walker)
    path := "/xxxx/"
    filepath.Walk(path, func(path string, info os.FileInfo, err error) error {
            if err != nil { return err }
            if info.IsDir() {
                walker.directories = append(walker.directories, path)
            } else {
                walker.files = append(walker.files, path)
            }
            return nil
        })
}
```

##      0x02    代码分析

####    filepath.Walk 方法
```golang
// Walk walks the file tree rooted at root, calling fn for each file or
// directory in the tree, including root.
//
// All errors that arise visiting files and directories are filtered by fn:
// see the WalkFunc documentation for details.
//
// The files are walked in lexical order, which makes the output deterministic
// but requires Walk to read an entire directory into memory before proceeding
// to walk that directory.
//
// Walk does not follow symbolic links.
//
// Walk is less efficient than WalkDir, introduced in Go 1.16,
// which avoids calling os.Lstat on every visited file or directory.
func Walk(root string, fn WalkFunc) error {
	info, err := os.Lstat(root)
	if err != nil {
                // Lstat获取root对应的路径（文件）信息出错,返回err由定义的walkFn函数处理，相当于退出递归方法（case1）
		err = fn(root, nil, err)
	} else {
                //开启递归处理，第一层
		err = walk(root, info, fn)
	}
	if err == SkipDir || err == SkipAll {
		return nil
	}
	return err
}
```

####    递归方法：[walk方法](https://cs.opensource.google/go/go/+/refs/tags/go1.20.2:src/path/filepath/path.go;drc=a0c9d153e0c177677701b8a4e6e5eba5a6c44a4f;l=482)
```golang
// walk recursively descends path, calling walkFn.
func walk(path string, info fs.FileInfo, walkFn WalkFunc) error {
	if !info.IsDir() {
                //非目录直接退出递归（调用用户方法walkFn）
		return walkFn(path, info, nil)
	}

        // 读取该path下的所有目录和文件
	names, err := readDirNames(path)
	err1 := walkFn(path, info, err)
	// If err != nil, walk can't walk into this directory.
	// err1 != nil means walkFn want walk to skip this directory or stop walking.
	// Therefore, if one of err and err1 isn't nil, walk will return.
	if err != nil || err1 != nil {
		// The caller's behavior is controlled by the return value, which is decided
		// by walkFn. walkFn may ignore err and return nil.
		// If walkFn returns SkipDir or SkipAll, it will be handled by the caller.
		// So walk should return whatever walkFn returns.
                //发生错误，退出递归
		return err1
	}

        //遍历文件和目录列表
	for _, name := range names {
                // 组装子目录的路径
		filename := Join(path, name)
		fileInfo, err := lstat(filename)
		if err != nil {
                        //出错，退出递归
			if err := walkFn(filename, fileInfo, err); err != nil && err != SkipDir {
				return err
			}
		} else {
                        //递归调用
			err = walk(filename, fileInfo, walkFn)
			if err != nil {
				if !fileInfo.IsDir() || err != SkipDir {
                                        //退出递归
					return err
				}
			}
		}
	}
	return nil
}
```

`walk`方法提供了一个写递归的非常标准的模板，代码逻辑也足够清晰了。特别注意的是最下层的这段逻辑，这里隐含的意思是，如果当前的`fileInfo`为文件（非目录），那么不需要再递归了，直接退出即可，文件相当于到达叶子节点了，也没法再递归下去。
```golang
//....
err = walk(filename, fileInfo, walkFn)
if err != nil {
        if !fileInfo.IsDir() || err != SkipDir {
                //退出递归
                return err
        }
}
//.....
```


##  0x03  使用闭包要注意的点

####  变量传递
```GOLANG
func main() {
        var s []string = []string{
                "a",
                "b",
                "c",
        }

        var waitGroup sync.WaitGroup
        waitGroup.Add(len(s))

        for _, item := range s {
                go func(i string) {
                        fmt.Println(i)
                        waitGroup.Done()
                }(item)
        }

        waitGroup.Wait()
}
```

注意由于匿名函数可以访问函数体外部的变量，而 `for range` 返回的 `val` 的值是引用的同一个内存地址的数据，所以匿名函数访问的函数体外部的 `val` 值是循环中最后输出的一个值，下面的代码是错误的：
```GOLANG
for _, item := range s {
    go func() {
        fmt.Println(item)
        waitGroup.Done()
    }()
}
```

另外一种规避方便是创建一个新变量，如下：
```golang
func main() {
        var s []string = []string{
                "a",
                "b",
                "c",
        }

        var waitGroup sync.WaitGroup
        waitGroup.Add(len(s))

        for _, item := range s {
                newitem := item
                go func() {
                        fmt.Println(newitem)
                        waitGroup.Done()
                }()
        }

        waitGroup.Wait()
}
```


## 0x04 参考
- [Golang 中关于闭包的坑](https://www.jianshu.com/p/fa21e6fada70)