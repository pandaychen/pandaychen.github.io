---
layout:     post
title:      Golang如何实现配置文件优雅的热更新
subtitle:   配置文件的优雅热加载实现以及viper库的机制分析
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

示例代码参见[这里](https://github.com/pandaychen/golang_in_action/tree/master/hotreload)

##	0x02	viper库的动态监听实现（v1.13.0）
笔者项目中常用的配置文件解析库是viper，其热更新机制基于[fsnotify](https://github.com/fsnotify/fsnotify)实现，通常使用如下代码，其中`ReloadConf`是需要开发者自行实现的配置文件解析逻辑：

```golang
viperInstance := viper.New()    // viper实例
viperInstance.WatchConfig()
viperInstance.OnConfigChange(func(e fsnotify.Event) {
    log.Print("Config file updated.")
    ReloadConf(viperInstance)  // 加载配置的方法
})
```


####    viper结构
viper结构[定义](https://github.com/spf13/viper/blob/v1.13.0/viper.go#L218)如下，注意有个`onConfigChange`成员：
```golang
// Note: Vipers are not safe for concurrent Get() and Set() operations.
type Viper struct {
	// Delimiter that separates a list of keys
	// used to access a nested value in one go
	keyDelim string

	// A set of paths to look for the config file in
	configPaths []string

	// The filesystem to read config from.
	fs afero.Fs

	// A set of remote providers to search for the configuration
	remoteProviders []*defaultRemoteProvider

	// Name of file to look for inside the path
	configName        string
	configFile        string
	configType        string
	configPermissions os.FileMode
	envPrefix         string

	// Specific commands for ini parsing
	iniLoadOptions ini.LoadOptions

	automaticEnvApplied bool
	envKeyReplacer      StringReplacer
	allowEmptyEnv       bool

	config         map[string]interface{}
	override       map[string]interface{}
	defaults       map[string]interface{}
	kvstore        map[string]interface{}
	pflags         map[string]FlagValue
	env            map[string][]string
	aliases        map[string]string
	typeByDefValue bool

	onConfigChange func(fsnotify.Event)

        //...
}
```


####    `WatchConfig`方法
`WatchConfig`[方法](https://github.com/spf13/viper/blob/v1.13.0/viper.go#L431)开启事件监听，确定用户操作文件后该文件是否可正常读取，并将内容注入到viper实例的`config`字段。任何文件的变更事件都可以从`Watcher.Events`这个channel中获取到，即`event, ok := <-watcher.Events`，这里的`event`有如下属性：
- `event.Name`：监听文件的绝对路径，如`/root/xxxx.yaml`
- `event.Op`：对文件的操作

`WatchConfig`关注的文件变更有两种：
-	if the config file was modified or created：（对文件进行了创建操作或写操作）
-	f the real path to the config file changed (eg: k8s ConfigMap replacement)  


```GOLANG
func (v *Viper) WatchConfig() {
	initWG := sync.WaitGroup{}
	initWG.Add(1)
	go func() {
                //用来建立新的文件监听处理，会开启一个协程开始等待读取事件，完成 从I/O完成端口读取任务，将事件注入到Event对象中
		watcher, err := newWatcher()
		if err != nil {
			log.Fatal(err)
		}
		defer watcher.Close()
		// we have to watch the entire directory to pick up renames/atomic saves in a cross-platform way
		filename, err := v.getConfigFile()
		if err != nil {
			log.Printf("error: %v\n", err)
			initWG.Done()
			return
		}

		configFile := filepath.Clean(filename)
		configDir, _ := filepath.Split(configFile)
		realConfigFile, _ := filepath.EvalSymlinks(filename)    

		eventsWG := sync.WaitGroup{}
		eventsWG.Add(1)
		go func() {
			for {
				select {
				case event, ok := <-watcher.Events:
					if !ok { // 'Events' channel is closed
						eventsWG.Done()
						return
					}
					currentConfigFile, _ := filepath.EvalSymlinks(filename)
					// we only care about the config file with the following cases:
					// 1 - if the config file was modified or created
					// 2 - if the real path to the config file changed (eg: k8s ConfigMap replacement)
					const writeOrCreateMask = fsnotify.Write | fsnotify.Create
					//用来判断操作的文件是否是configFile，确保是同一个文件
					if (filepath.Clean(event.Name) == configFile &&
						event.Op&writeOrCreateMask != 0) ||
						(currentConfigFile != "" && currentConfigFile != realConfigFile) {
						realConfigFile = currentConfigFile
						err := v.ReadInConfig()
						if err != nil {
							log.Printf("error reading config file: %v\n", err)
						}
						if v.onConfigChange != nil {
							//调用开发者设置的回调方法，处理变更event
							v.onConfigChange(event)
						}
					} else if filepath.Clean(event.Name) == configFile &&
						event.Op&fsnotify.Remove != 0 {
						eventsWG.Done()
						return
					}

				case err, ok := <-watcher.Errors:
					if ok { // 'Errors' channel is not closed
						log.Printf("watcher error: %v\n", err)
					}
					eventsWG.Done()
					return
				}
			}
		}()
		watcher.Add(configDir)
		initWG.Done()   // done initializing the watch in this go routine, so the parent routine can move on...
		eventsWG.Wait() // now, wait for event loop to end in this go-routine...
	}()
	initWG.Wait() // make sure that the go routine above fully ended before returning
}
```

`newWatcher`创建文件监听器：
```go
import "github.com/fsnotify/fsnotify"

type watcher = fsnotify.Watcher

func newWatcher() (*watcher, error) {
	return fsnotify.NewWatcher()
}
```

####    fsnotify
`WatchConfig`开始时，会调用fsnotify的[NewWatcher](https://github.com/fsnotify/fsnotify/blob/main/backend_inotify.go#L126)方法创建一个file监听器（以UNIX为例）：

当监听的文件发生改变时，会通过`sendEvent`方法将`events`发送到channel `w.Events` 中

```golang
// NewWatcher creates a new Watcher.
func NewWatcher() (*Watcher, error) {
	// Create inotify fd
	// Need to set the FD to nonblocking mode in order for SetDeadline methods to work
	// Otherwise, blocking i/o operations won't terminate on close
	fd, errno := unix.InotifyInit1(unix.IN_CLOEXEC | unix.IN_NONBLOCK)
	if fd == -1 {
		return nil, errno
	}

	w := &Watcher{
		fd:          fd,
		inotifyFile: os.NewFile(uintptr(fd), ""),
		watches:     make(map[string]*watch),
		paths:       make(map[int]string),
		Events:      make(chan Event),
		Errors:      make(chan error),
		done:        make(chan struct{}),
		doneResp:    make(chan struct{}),
	}
        //https://github.com/fsnotify/fsnotify/blob/main/backend_inotify.go#L335
	go w.readEvents()
	return w, nil
}

// Returns true if the event was sent, or false if watcher is closed.
func (w *Watcher) sendEvent(e Event) bool {
	select {
	case w.Events <- e:
		return true
	case <-w.done:
	}
	return false
}
```

####    `OnConfigChange`方法
`OnConfigChange`[方法](https://github.com/spf13/viper/blob/v1.13.0/viper.go#L424)是调用用户传入的方法，即执行`v.onConfigChange(event)`逻辑，通常我们不care这个`event`的内容（特殊情况下比如对文件做增量监控需要对`event`进行细化处理），直接读取所有配置再重新reload就好
```GOLANG
func OnConfigChange(run func(in fsnotify.Event)) { 
        v.OnConfigChange(run) 
}
func (v *Viper) OnConfigChange(run func(in fsnotify.Event)) {
	v.onConfigChange = run
}
```

如果想完整的处理`events`事件，可以尝试如下代码：
```golang
//监听配置变化
viper.WatchConfig()
//配置改变时的回调
viper.OnConfigChange(func(in fsnotify.Event) {
   switch in.Op {
   case fsnotify.Create:
      fmt.Println("监听到文件Create")
   case fsnotify.Remove:
      fmt.Println("监听到文件Remove")
   case fsnotify.Rename:
      fmt.Println("监听到文件Rename")
   case fsnotify.Write:
      fmt.Println("监听到文件Write")
   case fsnotify.Chmod:
      fmt.Println("监听到文件Chmod")
   }
})
```


##	0x03	ConfigMap/secrets的热更新

##	0x04	参考
-	[Go 每日一库之 viper](https://juejin.cn/post/6844904051369312264)
-	[go基于viper实现配置文件热更新及其源码分析](https://blog.csdn.net/HYZX_9987/article/details/103924392)
-	[golang进阶:怎么使用viper管理配置](https://ljlu1504.github.io/2018/12/26/how-to-use-viper-configuration-in-golang)
-	[viper 读取配置文件](https://learnku.com/articles/33908)
-	[Kubernetes ConfigMap hot-reload in action with Viper](https://medium.com/@xcoulon/kubernetes-configmap-hot-reload-in-action-with-viper-d413128a1c9a)
-	[记录Viper加载远程配置填坑过程](https://blog.huoding.com/2020/08/10/832)
-	[Kubernetes Pod 中的 ConfigMap 配置更新](https://aleiwu.com/post/configmap-hotreload/)