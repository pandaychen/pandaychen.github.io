---
layout: post
title: Kubernetes CRONJOB && kubebuilder CRONJOB 实现分析
subtitle:  
date: 2025-04-30
author: pandaychen
catalog: true
tags:
  - Kubernetes
---

##	0x00	前言
CronJob 管理基于时间的 Job，即：

-	在给定时间点只运行一次
-	周期性地在给定时间点运行。创建周期性运行的 Job，例如：数据库备份、发送邮件

参考使用[文档](https://www.zhaowenyu.com/kubernetes-doc/workloads/cronjob.html)

举个例子，如下的CRONJOB模版：

```YAML
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-hello-job
spec:
  schedule: "*/1 * * * *" # 每分钟一次
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox # 使用什么镜像
            # 在这里定义要执行的代码（命令）
            command: ["/bin/sh", "-c"]
            args: ["date; echo 'Hello from Kubernetes CronJob'"]
          restartPolicy: OnFailure
```

首先，明确一个概念，在 Kubernetes CronJob 的实现中，真正执行任务的代码并不在 Kubernetes 项目本身的代码库里。比如，对于上面的cronjob配置，真正的任务代码是 `image: busybox`和 `command: ["/bin/sh", "-c", "date; echo 'Hello from Kubernetes CronJob'"]`。这段代码被打包在 busybox这个容器镜像里。Kubernetes 是一个编排系统，它的核心职责是管理和管理工作负载，而不是直接执行这些工作负载的业务逻辑。它遵循一种声明式 API和控制器模式：

-	（用户）声明期望状态，用户提供一个 YAML 文件，告诉 Kubernetes 想要一个每小時运行一次的任务，这个任务的具体内容是运行 `curl http://example.com`，控制器确保实际状态匹配期望状态
-	CronJob 控制器（核心计划者：实现位于`pkg/controller/cronjob`）监听到用户（YAML）的声明，它的工作就是在正确的时间点，创建一个 Job 对象来代表这次任务运行
-	Job 控制器接手：Job 控制器发现有一个新的 Job 被创建了，它的工作是创建一个或多个 Pod 来执行这个任务
-	kubelet 最终执行：节点上的 kubelet服务监听到有 Pod 被调度到它所在的节点，它的工作是拉取指定的容器镜像并在容器内启动进程
-	任务的执行最终发生在 Pod 里的容器中

从代码层面上来说：
-	CronJob 控制器的代码（计划者）：只负责计时和创建 Job 对象。它包含一个无限循环，不断地检查是否有 CronJob 到了该触发的时间点。如果到了，它就调用 Kubernetes API 创建了一个 Job 资源。这里不包含任何与具体任务相关的逻辑。
-	Job 对象的定义（任务蓝图）：定义在CronJob YAML 文件的 `spec.jobTemplate.spec.template`中，即定义任务代码的地方。它实际上是一个 Pod 模板，其中包含了容器镜像、命令、参数等，真正的任务代码 `image: busybox`和 `command: ["/bin/sh", "-c", "date; echo 'Hello from Kubernetes CronJob'"]`。这段代码被打包在 busybox的容器镜像里
-	Job 控制器的代码（任务执行者）：位于 `pkg/controller/job`，这里的代码负责监听 Job 对象的创建。当它发现一个新的 Job 被创建（比如由 CronJob 控制器创建的），它就根据 Job 对象中的 Pod 模板，调用 Kubernetes API 创建一个或多个 Pod。它也不包含任务逻辑
-	最终执行地点（运行时）：集群的 Node 节点上，在由 Job 控制器创建的 Pod 中，执行者是节点上的 kubelet服务根据 Pod 的定义，拉取指定的容器镜像（如 busybox），并启动一个容器，在容器内执行命令（如 `/bin/sh -c "date; echo ...`）。任务代码最终是在这个容器内部运行的。它可以是任何能在容器里运行的东西：一个 Shell 脚本、一个 Python 程序、一个编译好的 Go 二进制文件等

Kubernetes 自身的代码（CronJob/Job 控制器）只是确保这个蓝图（YAML）能在正确的时间被正确地实例化和运行

##	0x01	k8s的cronjob controller实现
核心controller实现位于[cronjob_controllerv2](https://github.com/kubernetes/kubernetes/blob/release-1.28/pkg/controller/cronjob/cronjob_controllerv2.go)

####	工作流程
和其他controller实现机制一样，cronjob也遵循kubernetes的controller的一般设计模式

![k8s-controller]()

![cronjob-controller-flow]()

####	核心代码分析
1、`Controller`控制器定义，`CronJobController`结构体定义了控制器对象，Controller 控制器对象是实现 CronJob 功能的核心对象

```GO
type ControllerV2 struct {
	// 队列中存储发生了变化 (需要同步) 的 CronJob 对象
	queue workqueue.RateLimitingInterface

	// kubelet 客户端对象：用于执行各项类似 "kubectl ..." 操作
	kubeClient  clientset.Interface

	// Job 列表
	jobLister     batchv1listers.JobLister
	// CronJob 列表
	cronJobLister batchv1listers.CronJobLister
}
```

2、Controller初始化，`NewControllerV2` 方法用于 CronJob 控制器对象的初始化工作，并返回一个实例化对象

```GO
func NewControllerV2(ctx context.Context, ...) (*ControllerV2, error) {
	jm := &ControllerV2{
		queue:       workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "cronjob"),
		kubeClient:  kubeClient,
		broadcaster: eventBroadcaster,
		recorder:    eventBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{Component: "cronjob-controller"}),

		jobControl:     realJobControl{KubeClient: kubeClient},
		cronJobControl: &realCJControl{KubeClient: kubeClient},

		jobLister:     jobInformer.Lister(),
		cronJobLister: cronJobsInformer.Lister(),

		jobListerSynced:     jobInformer.Informer().HasSynced,
		cronJobListerSynced: cronJobsInformer.Informer().HasSynced,
		now:                 time.Now,
	}

	// 增加 Job informer 监听回调方法
	jobInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    jm.addJob,		//细节1
		UpdateFunc: jm.updateJob,
		DeleteFunc: jm.deleteJob,
	})
	
	// 增加 CronJob informer 监听回调方法
	cronJobsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
			// cronJobsInformer的addfunc
			jm.enqueueController(obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			jm.updateCronJob(logger, oldObj, newObj)
		},
		DeleteFunc: func(obj interface{}) {
			jm.enqueueController(obj)
		},
	})

	return jm, nil
}
```

TODO

3、启动控制器Controller，根据控制器的初始化方法 `NewControllerV2` 的调用链路，可以找到控制器开始启动和执行的地方

```GO
// cmd/kube-controller-manager/app/batch.go
func startCronJobController(ctx context.Context, ...) (controller.Interface, bool, error) {
	cj2c, err := cronjob.NewControllerV2(ctx, controllerContext.InformerFactory.Batch().V1().Jobs(),
		......
	)
    
    // 启动单独的 goroutine 来完成控制器的 {初始化 && 运行}
	go cj2c.Run(ctx, int(controllerContext.ComponentConfig.CronJobController.ConcurrentCronJobSyncs))
	
	return nil, true, nil
}
```

`ControllerV2.Run` 方法执行控制器具体的启动逻辑，注意`UntilWithContext`方法的参数`jm.worker`

```GO
func (jm *ControllerV2) Run(ctx context.Context, workers int) {
	...

	// 根据参数配置个数，启动多个 goroutine 处理逻辑
	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, jm.worker /*jm.work*/, time.Second)
	}

	<-ctx.Done()
}
```

`Controller.worker` [方法](https://github.com/kubernetes/kubernetes/blob/release-1.28/pkg/controller/cronjob/cronjob_controllerv2.go#L105)本质上就是一个无限循环轮询器，不断从队列中取出 `CronJob` 对象，然后进行对应的操作，内部无限循环调用 `processNextWorkItem` 方法

cronjob最核心的实现都在`jm.sync`中

```GO
func (jm *ControllerV2) worker(ctx context.Context) {
	for jm.processNextWorkItem(ctx) {
	}
}

func (jm *ControllerV2) processNextWorkItem(ctx context.Context) bool {
	// 从队列获取 CronJob 对象
	key, quit := jm.queue.Get()
    
	...... 

	// 和其他几个常用的控制器不同
	// CronJob 控制器不是通过字段注入的方式设置同步的回调方法
	// 而是直接调用 sync 方法进行同步
	requeueAfter, err := jm.sync(ctx, key.(string))
	
	switch {
	case err != nil:
		// 如果同步回调方法执行异常
		// 将当前 CronJob 对象重新放入队列
		jm.queue.AddRateLimited(key)
	case requeueAfter != nil:
		// 如果同步回调方法执行成功
		//  并且 
		// CronJob 对象的下次执行时间不为空
		// 第一步: 先将当前 CronJob 对象踢出队列
		jm.queue.Forget(key)
		// 第二步: 将当前 CronJob 对象延迟放入队列
		jm.queue.AddAfter(key, *requeueAfter)
	}
	
	return true
}
```

4、CronJob 同步：`ControllerV2` 的回调处理方法是 `sync` 方法，该方法是所有 CronJob 对象同步操作的入口方法

```GO
func (jm *ControllerV2) sync(ctx context.Context, cronJobKey string) (*time.Duration, error) {
	// 通过 key 解析出 CronJob 对象对应的 命名空间和名称
	ns, name, err := cache.SplitMetaNamespaceKey(cronJobKey)

	// 获取 CronJob 对象
	cronJob, err := jm.cronJobLister.CronJobs(ns).Get(name)
	
	......

	// 获取 CronJob 对象关联的 Job 对象列表
	jobsToBeReconciled, err := jm.getJobsToBeReconciled(cronJob)

	// 深度拷贝一个 CronJob 对象 (用户优化操作)
	// 拷贝的对象用于计算源对象的所有更新操作
	// 并且仅在需要的时候执行一次更新
	cronJobCopy := cronJob.DeepCopy()

	// 删除完成的 Job
    updateStatusAfterCleanup := jm.cleanupFinishedJobs(ctx, cronJobCopy, jobsToBeReconciled)
	
	// 获取 CronJob 对象下次执行的时间和同步 (更新) 状态
	requeueAfter, updateStatusAfterSync, syncErr := jm.syncCronJob(ctx, cronJobCopy, jobsToBeReconciled)
	
    // 更新 CronJob 对象状态
	if updateStatusAfterCleanup || updateStatusAfterSync {
		if _, err := jm.cronJobControl.UpdateStatus(ctx, cronJobCopy); err != nil {
			......
			return 
		}
	}

	// 返回下次 CronJob 的下次执行时间
	if requeueAfter != nil {
		return requeueAfter, nil
	}
	
	return nil, syncErr
}
```

##	0x0	kubebuilder的 cronjob controller 实现


##  0x0 参考
-	[Kubernetes CronJob 设计与实现](https://dbwu.tech/posts/k8s/source_code/cronjob_controller/)