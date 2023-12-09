---
layout:     post
title:      内存缓存使用的一些技巧
subtitle:
date:       2022-10-03
author:     pandaychen
catalog:    true
tags:
    - Cache
---


##  0x00    前言
本文梳理下笔者在项目开发中使用过一些内存缓存的技巧

##	0x01	双缓冲：double buffering
有一个场景是，存在某一本地文件配置（修改少），程序初始化时把文件内容读取到本地内存里（假设为 map1），由于程序逻辑需要频繁且高性能的读取 map1（尽量不加锁），在这种背景下如何实现安全的修改文件自动同步到 `map` 里（且不加锁）？有两种思路：

1.	分段锁
2.	使用双缓冲（double buffering）技术，方法是在后台修改一个副本的 `map`，当修改完成后，将其原子性地替换为当前活动的 `map`。这样在修改期间，程序可以继续高性能（原子）读取当前活动的 map，修改完成后，将当前活动的 `map` 指向已经完成新一轮加载读取的 map（ping-pong map）

参考代码如下，`MapHolder` 包含 `2` 个 `map` 和一个 atomic 变量 `idx`。`LoadFile` 方法读取文件内容并将其加载到一个新的 `map` 中，然后原子性地替换当前活动的 `map`。`Get` 方法根据当前活动的 `map` 来获取键值对；如此实现允许在不加锁的情况下实现对 `map` 的高性能访问。但注意该实现仅适用于读密集型场景，如果需要同时进行大量的写操作则不建议使用此方法

```GO
type MapHolder struct {
	maps [2]*map[string]string
	idx  int32
}

func NewMapHolder() *MapHolder {
	m1 := make(map[string]string)
	m2 := make(map[string]string)
	return &MapHolder{maps: [2]*map[string]string{&m1, &m2}}
}

func (h *MapHolder) LoadFile(filename string) error {
	file, err := os.Open(filename)
	if err != nil {
		return err
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	newMap := make(map[string]string)
	for scanner.Scan() {
		parts := strings.Split(scanner.Text(), ":")
		if len(parts) == 2 {
			newMap[parts[0]] = parts[1]
		}
	}

	// 原子性地替换活动 map
	idx := atomic.LoadInt32(&h.idx)
	atomic.StorePointer((*unsafe.Pointer)(unsafe.Pointer(&h.maps[idx^1])), unsafe.Pointer(&newMap))
	atomic.StoreInt32(&h.idx, idx^1)

	return scanner.Err()
}

func (h *MapHolder) Get(key string) (string, bool) {
	idx := atomic.LoadInt32(&h.idx)     // 原子获取当前活动的 index
	m := h.maps[idx]
	value, ok := (*m)[key]
	return value, ok
}

func main() {
	holder := NewMapHolder()
	err := holder.LoadFile("file.txt")
	if err != nil {
		fmt.Println("Error loading file:", err)
		return
	}

	go func() {
		for {
			time.Sleep(1 * time.Second)
			err := holder.LoadFile("file.txt")
			if err != nil {
				fmt.Println("Error reloading file:", err)
			}
		}
	}()

	for {
		time.Sleep(100 * time.Millisecond)
		value, ok := holder.Get("a")
		if ok {
			fmt.Println("Value of a:", value)
		} else {
			fmt.Println("Key a not found")
		}
	}
}
```

##	0x	参考