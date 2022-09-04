---
layout:     post
title:      Golang如何实现配置文件优雅的热更新
subtitle:	配置文件的优雅热加载实现以及viper库的机制分析
date:       2021-03-02
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 热更新
    - Golang
---

##	0x00	前言
Nginx中有个`reload`指令，提供了不重启服务下针对配置文件的热加载能力，本文讨论下在golang下的文件热加载实现。
1.	通过signal手动触发
2.	通过文件监听notify自动触发

##	0x01	go中的实现

####	手动触发
要点有如下几点：
-	配置文件指针化，并且加锁保护（配置的单例模式）
-	捕获信号，重新读配置
-	在代码逻辑引用配置文件的核心路径上，读出配置文件的字段

1、初始化配置<br>
通常来说，在一个进程周期，配置文件仅需解析一次，建议使用单例模式实现config的解析，go中可以使用`sync.Once`实现
```golang
var (
        cfg * tomlConfig
        once sync.Once
        cfgLock = new(sync.RWMutex)
)

func Config() *tomlConfig {
	once.Do(ReloadConfig)
	cfgLock.RLock()
	defer cfgLock.RUnlock()
	return cfg
}

func ReloadConfig() {
        filePath, err := filepath.Abs(configPath)
        if err != nil {
                panic(err)
        }
        config := new(tomlConfig)
        if _ , err := toml.DecodeFile(filePath, config); err != nil {
                panic(err)
        }
        cfgLock.Lock()
        defer cfgLock.Unlock()
        cfg = config
}
```

2、捕获信号<br>
```golang
func handleSignals(){
	s := make(chan os.Signal, 1)
	signal.Notify(s, syscall.SIGUSR1)
	for {
			<-s
			ReloadConfig()
	}
}
```

3、重载后，在程序的路径中使用配置<br>

示例代码参见[此]()

##	0x02	viper库的动态监听实现

##	0x03	参考
-	[Go 每日一库之 viper](https://juejin.cn/post/6844904051369312264)
-	[go基于viper实现配置文件热更新及其源码分析](https://blog.csdn.net/HYZX_9987/article/details/103924392)
-	[golang进阶:怎么使用viper管理配置](https://ljlu1504.github.io/2018/12/26/how-to-use-viper-configuration-in-golang)