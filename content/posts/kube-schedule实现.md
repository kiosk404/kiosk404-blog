---
title: kube-schedule 调度实现
author: kiosk
tags: []
categories:
  - k8s
date: 2024-01-05 00:05:00
---

kubernetest 中有很多核心的组件，其中一个非常重要的组件是 kube-scheduler。 kube-scheduler 负责将新创建的 Pod 调度到合适的节点上运行。



<img src="https://img1.kiosk007.top/static/images/blog/kube-schedule.jpg" alt="vm1" style="zoom: 53%;clear:both;display:block;margin:auto;" />



<!--more-->

# kube-scheduler 的设计

scheduler 在整个系统中承担了 “承上启下” 的重要功能，“承上” 是指负责接受 “Controller Manager” 创建的新 Pod， 为其安排 Node；“启下” 是指安置工作完成后，目标Node 上的 kubelet 服务进程接管后续的绑定创建工作，Pod 是 Kubernetes 中的最小调度单元。



<img src="https://img1.kiosk007.top/static/images/blog/scheduler-sequence.png" alt="vm1" style="zoom: 53%;clear:both;display:block;margin:auto;" />

在上图的调度流程中：

- 第一步 通过 apiserver 的 REST API 创建一个 Pod
- 然后 apiserver 接收到数据后，将数据写入到 etcd 中。
- 由于 kube-scheduler 通过 apiserver watch API 一直在监听资源的变化，发现有一个新的 Pod ，但是这个 Pod 还没有和任何的 Node 节点进行绑定，所以就会加入到 调度队列中，kube-scheduler 就会进行调度，选择出一个合适的 Node 节点，将该 Pod 和该目标 Node 进行绑定，绑定后再更新消息到 etcd 中。
- 目标节点上的 kubelet 通过 apiserver watch API 检测到有一个新的 Pod 调度过来了，他就将该 Pod 的数据传递给后面的容器运行时（container runtime），比如 Docker，让他们去运行该 Pod。

- 而且 kubelet 还会通过 container runtime 获取 Pod 的状态，然后更新到 apiserver 中，当然最后也是写入到 etcd 中的。

这个过程最重要的就是 apiserver watch API 和 kube-scheduler 的调度策略。

总之，kube-scheduler 的功能就是根据 预选策略（Predicates）和优选 （Priorities）两个步骤。

1. 预选（Predicates）：kube-scheduler 根据预选策略过滤掉不满足策略的 Node 节点。（比如磁盘不足，CPU 不足等）
2. 优选（Priorities）：优选会根据优选策略为通过 预选的 Nodes 进行打分排名，选择得分最高的Node，例如，资源越丰富，负载越小等插件plugins 来选出评分最高的 Node。



<!--more-->

# kube-scheduler 应用源码

> 以下源码均以 1.20.1 版本为例 

## 调度器设置

服务在调用 `Setup(ctx, opts, registryOptions...)`  函数来进行调度器的设置工作。

```go
func Setup(ctx context.Context, opts *options.Options, outOfTreeRegistryOptions ...Option) (*schedulerserverconfig.CompletedConfig, *scheduler.Scheduler, error) {
	if errs := opts.Validate(); len(errs) > 0 {
		return nil, nil, utilerrors.NewAggregate(errs)
	}

	c, err := opts.Config()
	if err != nil {
		return nil, nil, err
	}

    // Get the completed config (获取完整的配置)
	cc := c.Complete()

    // 初始化外部的调度器插件
	outOfTreeRegistry := make(runtime.Registry)
	for _, option := range outOfTreeRegistryOptions {
		if err := option(outOfTreeRegistry); err != nil {
			return nil, nil, err
		}
	}

    // 获取事件记录的工厂
	recorderFactory := getRecorderFactory(&cc)
	completedProfiles := make([]kubeschedulerconfig.KubeSchedulerProfile, 0)
    // Create the scheduler. (创建调度器实例)
	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory,
		recorderFactory,
		ctx.Done(),
        // 调度器的设置
		scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
		scheduler.WithAlgorithmSource(cc.ComponentConfig.AlgorithmSource),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
		scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
		scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
		scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
		scheduler.WithParallelism(cc.ComponentConfig.Parallelism),
		scheduler.WithBuildFrameworkCapturer(func(profile kubeschedulerconfig.KubeSchedulerProfile) {
			// Profiles are processed during Framework instantiation to set default plugins and configurations. Capturing them for logging
			completedProfiles = append(completedProfiles, profile)
		}),
	)
	if err != nil {
		return nil, nil, err
	}
    // 打印或者记录调度器的配置项
	if err := options.LogOrWriteConfig(opts.WriteConfigTo, &cc.ComponentConfig, completedProfiles); err != nil {
		return nil, nil, err
	}

	return &cc, sched, nil
}
```

上述代码，创建了一个可用的 `*schedulerserverconfig.CompletedConfig` 类型的应用配置，后续的 调度服务都会以此配置为基础创建调度器。

<br/>

其返回了一个 `*scheduler.Scheduler` 类型的 `sched` 变量，`sched` 变量提供的 `Run` 方法，可以启动 `kube-scheduler` 的核心逻辑。

<br/>

`runCommand` 函数最终调用 `Run` 函数来启动 `kube-scheduler` 服务，其内容很多。其中包含

1. 启动事件广播器

2. 等待选主成功

3. 启动 Informer (存储各种元数据，比如节点信息, PVC 等等，详细参考 [kubernetes 中 informer 的使用](https://cloud.tencent.com/developer/article/1555864)), 并等待缓存同步完成。`kube-scheduler` 会启动2类 Informer：

   - InformerFactory：用于创建动态 SharedInformer，可以监视任何 Kubernetes 资源对象，不需要提前生成对应的 clientset。

   - DynInformerFactory：用于创建针对特定 API 组的 SharedInformer，需要提供对应 API 组的 clientset。

   

<img src="https://img1.kiosk007.top/static/images/blog/informer-1.png" alt="vm1" style="zoom: 53%;clear:both;display:block;margin:auto;" />



```go
	// Start all informers.
	cc.InformerFactory.Start(ctx.Done())
	// DynInformerFactory can be nil in tests.
	if cc.DynInformerFactory != nil {
		cc.DynInformerFactory.Start(ctx.Done())
	}

	// Wait for all caches to sync before scheduling.
	cc.InformerFactory.WaitForCacheSync(ctx.Done())
	// DynInformerFactory can be nil in tests.
	if cc.DynInformerFactory != nil {
		cc.DynInformerFactory.WaitForCacheSync(ctx.Done())
	}
```

可以看到 Run 中 会启动 Informer，并等待 Informer 的缓存全部完成。

4. 运行调度器，最后调用 `sched.Run(ctx)` 运行调度器，调度器在运行期间，`kube-scheduler` 主进程会一直阻塞在 `sched.Run(ctx)` 函数的调用处。

<br/>

# kube-scheduler 调度原理



整个 `kube-scheduler` 核心的调度逻辑是通过 **sched.Run(ctx)** 来启动，代码如下



```go
// Run begins watching and scheduling. It starts scheduling and blocked until the context is done.
func (sched *Scheduler) Run(ctx context.Context) {
    // 启动调度队列，调度队列会 Watch kube-apiserver , 并存储 待调度的 Pod
	sched.SchedulingQueue.Run()

	// We need to start scheduleOne loop in a dedicated goroutine,
	// because scheduleOne function hangs on getting the next item
	// from the SchedulingQueue.
	// If there are no new pods to schedule, it will be hanging there
	// and if done in this goroutine it will be blocking closing
	// SchedulingQueue, in effect causing a deadlock on shutdown.
    // 不断的轮询，执行 sched.scheduleOne 函数，sched.scheduleOne 函数会消费调度队列中待调度的 Pod     // 执行调度流程，完成 Pod 的调度
	go wait.UntilWithContext(ctx, sched.scheduleOne, 0)

	<-ctx.Done()
    // 清理并释放资源
	sched.SchedulingQueue.Close()
}
```



## kube-scheduler 调度模型





<img src="https://img1.kiosk007.top/static/images/blog/kube-sched-framework.png" alt="vm1" style="zoom: 53%;clear:both;display:block;margin:auto;" />

**kube-scheduler** 的调度原理如上图所示，主要分为 3 大部分：

- Policy：Scheduler 的调度策略启动配置目前支持三种方式，配置文件 / 命令行参数 / ConfigMap。  调度策略可以配置指定的调度主流程要用哪些 过滤器 （Predicates）、打分器（Priorities）、外部扩展的调度器（Extenders），以及最新支持的 SchedulerFramework 的自定义扩展点（Plugins）
- Informer：Scheduler 在启动的时候，以 List + Watch 的从 kube-apiserver 获取调度需要的数据，例如Pods、 Nodes、Persistant Volume（PV），Persistant Volume Claim (PVC) 等等，并将这些数据做一定的预处理，作为调度器的 Cache
- 调度流水线：通过 Informer 将需要调度的 Pod 插入 Queue 中，Pipeline 会循环从 Queue Pop 等待调度的 Pod 放入 Pipeline 执行。调度流水线（Schedule Pipeline） 主要有三个阶段：Scheduler Thread, Wait Thread, Bind Thread 。
  - **Scheduler Thread 阶段：** 从如上的架构图可以看到 Scheduler Thread 会经历 Pre Filter -> Filter -> Post Filter -> Score -> Reserve ，可以简单理解为 Filter -> Score -> Reserve 。 
    - Filter 阶段用于选择符合 Pod Spec 描述的 Nodes
    - Score 阶段用于从 Filter 过后的 Nodes 进行打分和 排序
    - Reserve 阶段将 Pod 跟排序后最优 Node 的 NodeCache 中，表示这个 Pod 已经分配到这个 Node 上，让下一个等待调度的 Pod 对这个 Node 进行 Filter 和 Score 的时候能看到刚才分配的 Pod
  - **Wait Thread 阶段**：这个阶段可以用来等待 Pod 关联的资源的 Ready 等待，例如等待 PVC 的 PV 创建成功。
  - **Bind Thread 阶段：** 用于将 Pod 和 Node 的关联持久化 Kube APIServer



整个 调度流水线只有在 Scheduler Thread 阶段是串行的一个 Pod 一个 Pod 的进行调度，在 Wait 和 Bind 阶段Pod 都是异步并行执行。

<br/>

## Scheduling Framework 调度框架

kube-scheduler 从 v1.15 版本开始，引入了一种非常灵活的调度框架。

调度框架是面向 Kubernetes 调度器的一种插件架构，它由一组直接编译到调度程序中的“插件”API 组成。这些 API 允许大多数调度功能以插件的形式实现，同时使调度 “核心” 保持简单且可维护。每个插件支持不同的调度扩展点，一个插件可以在多个扩展点注册，以执行更复杂或有状态的任务。



每次调度一个 Pod 的尝试分为 两个阶段，即 **调度周期** 和 **绑定周期** 。调度周期为 Pod 选择一个节点，绑定周期将一个Pod绑定到一个Node上。调度周期和绑定周期一起被称为 “调度上下文” 。 调度周期是串行运行的，而调度周期可能是同时运行的。如果确定 Pod 不可调度或者存在内部错误，则可以终止调度周期或者绑定周期，Pod 将返回队列并重试。



<img src="https://img1.kiosk007.top/static/images/blog/schedule-cycle.png" alt="vm1" style="zoom: 53%;clear:both;display:block;margin:auto;" />

调度扩展点功能描述如下：

| 调度扩展点     | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| QueueSort      | Sort 扩展用于对 Pod 的待调度队列进行排序，以决定先调度哪个 Pod，Sort 扩展本质上只需要一个 方法 Less（Pod1,Pod2 谁更优先得到调度） |
| PreFilter      | PreFilter 扩展用于对 Pod 的信息进行预处理，或者检查一些集群或 Pod 必须满足的前提条件，然后将其存入缓存中等待 Filter 扩展用 |
| Filter         | Filter 扩展用于排除那些不能运行该 Pod 的节点，对于每个节点，调度器将按顺序执行 filter 扩展 |
| PostFilter     | PostFilter 扩展会对 Score 扩展点的数据做一些预处理操作，然后将其存入缓存中，等待 Score 扩展点执行的时候使用 |
| PreScore       | PreScore 扩展会对 Score 扩展点的数据做一些预处理操作，然后将其存入缓存中等待 Score 扩展点执行的时候使用 |
| Score          | Score 扩展用于为所有的节点进行打分，调度器将针对每一个节点调用 Score 扩展，最终调度器会把每个 Score 打分器对具体某个节点的评分结果和该扩展的权重合并起来，作为最终的评分结果 |
| NormalizeScore | Normalize score 扩展在调度器对节点进行最终排序之前修改每个节点的评分结果，注册到该扩展点的扩展在被调用时，将获得同 score 扩展的评分结果作为参数，调度框架每执行一次调度，都会调用所有的插件中的一个 normalize score 扩展一次 |
| Reserve        | Reserve 是一个通知性质的扩展点，有状态的插件可以用使用该扩展点来获得节点上为 Pod 预留的资源，该事件发生在调度器将Pod绑定到节点之前，目的是避免调度器在等待 Pod 与节点绑定的过程中调度新的 Pod 到节点上时，发生实际使用的资源超出可用资源的情况。这是调度过程的最后一步骤，Pod 进入 reserved 状态以后，要么在绑定失败时触发 Unreserve 扩展，要么在绑定成功时，由 Post-bind 扩展结束绑定过程。 |
| Permit         | Permit 扩展在每个 Pod 调度周期的最后调用，用于阻止或者延迟 Pod 与节点的绑定。Permit 扩展可以做下面三件事的一项：1. approve（批准）：当所有的permit 扩展都 approve 了 Pod 与 节点的绑定，调度器将继续执行绑定过程 。2. deny（拒绝）：如果任何一个 permit 扩展 deny 了 Pod 与节点的绑定，Pod 将被放回到调度队列，此时将触发 Unreserve 扩展 。3. wait（等待）：如果一个permit 扩展返回了 wait，则 Pod 将保持在 permit 阶段，直到被其他扩展 approve ，如果超时事件发生，wait 状态变成 deny，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展。 |
| WaitOnPermit   | WaitOnPermit 扩展 与 Permit 扩展点配合使用实现延时调度功能（内部默认实现） |
| PreBind        | PreBind 扩展用于在 Pod 绑定之前执行某些逻辑，例如，将一个基于网络的数据卷挂载到节点上 |
| Bind           | Bind 扩展用于将 Pod 绑定到节点上                             |
| PostBind       | PostBind 是一个同志性质的扩展，可以用来执行资源清理的动作    |
| Unreserve      | Unreserve 是一个通知性质的扩展，如果为 Pod 预留了资源，Pod 又在绑定过程中被拒绝绑定，则 unreserve 扩展将被调用，Unreserve 扩展应该释放已经为 Pod 预留的节点上的计算资源 |


在 Kube-scheduler 的 Framework 中拥有多个 scheduler.WithXXX 来进行设置

```go
type schedulerOptions struct {
    // KubeSchedulerConfiguration 的 APIVersion，例如：kubescheduler.config.k8s.io/v1，没有实际作用。
    componentConfigVersion string
    // 访问 kube-apiserver 的 REST 客户端
    kubeConfig             *restclient.Config
    // Overridden by profile level percentageOfNodesToScore if set in v1.
    // 节点得分所使用的节点百分比，如果在 v1 中设置了 profile 级别的 percentageOfNodesToScore，则会被覆盖
    percentageOfNodesToScore          int32
    // Pod 的初始退避时间
    podInitialBackoffSeconds          int64
    // Pod 的最大退避时间
    podMaxBackoffSeconds              int64
    // 最大不可调度 Pod 的持续时间
    podMaxInUnschedulablePodsDuration time.Duration
    // Contains out-of-tree plugins to be merged with the in-tree registry.
    // 包含了外部插件，用于与内部注册表进行合并
    frameworkOutOfTreeRegistry frameworkruntime.Registry
    // 调度器的配置文件
    profiles                   []schedulerapi.KubeSchedulerProfile
    // 调度器的扩展程序
    extenders                  []schedulerapi.Extender
    // 用于捕获构建调度框架的函数
    frameworkCapturer          FrameworkCapturer
    // 调度器的并行度
    parallelism                int32
    // 表示是否应用默认配置文件
    applyDefaultProfile        bool
}
```

通过 `scheduler.New` 函数来创建一个 `*Scheduler` 实例，其代码如下：

```go
// New returns a Scheduler
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

	stopEverything := stopCh
	if stopEverything == nil {
		stopEverything = wait.NeverStop
	}

	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}

    // 设置默认的调度策略
	if options.applyDefaultProfile {
		var versionedCfg configv1.KubeSchedulerConfiguration
		scheme.Scheme.Default(&versionedCfg)
		cfg := schedulerapi.KubeSchedulerConfiguration{}
		if err := scheme.Scheme.Convert(&versionedCfg, &cfg, nil); err != nil {
			return nil, err
		}
		options.profiles = cfg.Profiles
	}

    // 创建一个 In-Tree Registry 用来保存 kube-scheduler 自带的调度插件
	registry := frameworkplugins.NewInTreeRegistry()
    // 将 In-Tree 调度插件和 Out-Of-Tree 调度插件进行合并
	if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
		return nil, err
	}

	metrics.Register()

    // 创建 Extender 调度器插件，
	extenders, err := buildExtenders(options.extenders, options.profiles)
	if err != nil {
		return nil, fmt.Errorf("couldn't build extenders: %w", err)
	}

    // 创建 Pod、Node 的Lister,用来 List & Watch Pod 和 Node 资源
	podLister := informerFactory.Core().V1().Pods().Lister()
	nodeLister := informerFactory.Core().V1().Nodes().Lister()

	// The nominator will be passed all the way to framework instantiation.
    // 根据调度器策略和调度插件，创建调度框架的集合。
	nominator := internalqueue.NewPodNominator(podLister)
	snapshot := internalcache.NewEmptySnapshot()
	clusterEventMap := make(map[framework.ClusterEvent]sets.String)

	profiles, err := profile.NewMap(options.profiles, registry, recorderFactory, stopCh,
		frameworkruntime.WithComponentConfigVersion(options.componentConfigVersion),
		frameworkruntime.WithClientSet(client),
		frameworkruntime.WithKubeConfig(options.kubeConfig),
		frameworkruntime.WithInformerFactory(informerFactory),
		frameworkruntime.WithSnapshotSharedLister(snapshot),
		frameworkruntime.WithPodNominator(nominator),
		frameworkruntime.WithCaptureProfile(frameworkruntime.CaptureProfile(options.frameworkCapturer)),
		frameworkruntime.WithClusterEventMap(clusterEventMap),
		frameworkruntime.WithParallelism(int(options.parallelism)),
		frameworkruntime.WithExtenders(extenders),
	)
	if err != nil {
		return nil, fmt.Errorf("initializing profiles: %v", err)
	}

	if len(profiles) == 0 {
		return nil, errors.New("at least one profile is required")
	}

    // 创建调度队列
	podQueue := internalqueue.NewSchedulingQueue(
		profiles[options.profiles[0].SchedulerName].QueueSortFunc(),
		informerFactory,
		internalqueue.WithPodInitialBackoffDuration(time.Duration(options.podInitialBackoffSeconds)*time.Second),
		internalqueue.WithPodMaxBackoffDuration(time.Duration(options.podMaxBackoffSeconds)*time.Second),
		internalqueue.WithPodNominator(nominator),
		internalqueue.WithClusterEventMap(clusterEventMap),
		internalqueue.WithPodMaxInUnschedulablePodsDuration(options.podMaxInUnschedulablePodsDuration),
	)
	// 创建缓存，主要用来缓存 Node、Pod 等信息，用来提高调度性能
	schedulerCache := internalcache.New(durationToExpireAssumedPod, stopEverything)

	// Setup cache debugger.
	debugger := cachedebugger.New(nodeLister, podLister, schedulerCache, podQueue)
	debugger.ListenForSignal(stopEverything)

	sched := newScheduler(
		schedulerCache,
		extenders,
		internalqueue.MakeNextPodFunc(podQueue),
		stopEverything,
		podQueue,
		profiles,
		client,
		snapshot,
		options.percentageOfNodesToScore,
	)

    // 添加 EventHandlers, 根据 Pod,Node 资源的更新情况，将资源放入合适的 Cache 中，根据 CSINode、
    // CSIDriver、PersistentVolume 等资源的更新状态将 PreEnqueueCheck中的 Pod 放入到 
    // BackoffQueue 、 ActiveQueue。
	addAllEventHandlers(sched, informerFactory, dynInformerFactory, unionedGVKs(clusterEventMap))

	return sched, nil
}
```



## 调度插件设置

kube-scheduler 是通过一系列的调度插件最终完成 Pod 调度的，在启动 kube-scheduler 时，首先要加载调度插件，调度插件分为 2种，分别是 **in-tree** 和 **out-of-tree** 。

- In-tree 插件（内建插件）：这些插件是作为 Kubernetes 核心组件的一部分直接编译和交付的，它们与 Kubernetes 的源代码一起维护，并与 Kubernetes 版本保持同步。这些插件以静态库形式打包到 kube-scheduler 的二进制文件中，一些常见的 in-tree 插件包含默认的调度算法，Packed Scheduling 等。
- Out-of-tree 插件（外部插件）：这些插件是作为独立项目开发和维护的，它们与 Kubernetes 核心代码分开，并且可以单独部署和更新。本质上，out-of-tree 插件是基于 kubernetes 的调度器扩展点进行开发的。这些插件以独立的二进制文件的形式存在。



### Out-Of-Tree 插件初始化

kube-scheduler 首先加载的是 out-of-tree 插件。在 main 文件中，就会调用 app.NewSchedulerCommand 来创建一个 Scheduler Application 。例如 [scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins/blob/v0.27.8/cmd/scheduler/main.go) 的 scheduler 实现。

```go
func main() {
	// Register custom plugins to the scheduler framework.
	// Later they can consist of scheduler profile(s) and hence
	// used by various kinds of workloads.
	command := app.NewSchedulerCommand(
		app.WithPlugin(capacityscheduling.Name, capacityscheduling.New),
		app.WithPlugin(coscheduling.Name, coscheduling.New),
		app.WithPlugin(loadvariationriskbalancing.Name, loadvariationriskbalancing.New),
		app.WithPlugin(networkoverhead.Name, networkoverhead.New),
		app.WithPlugin(topologicalsort.Name, topologicalsort.New),
		app.WithPlugin(noderesources.AllocatableName, noderesources.NewAllocatable),
		app.WithPlugin(noderesourcetopology.Name, noderesourcetopology.New),
		app.WithPlugin(preemptiontoleration.Name, preemptiontoleration.New),
		app.WithPlugin(targetloadpacking.Name, targetloadpacking.New),
		app.WithPlugin(lowriskovercommitment.Name, lowriskovercommitment.New),
		// Sample plugins below.
		// app.WithPlugin(crossnodepreemption.Name, crossnodepreemption.New),
		app.WithPlugin(podstate.Name, podstate.New),
		app.WithPlugin(qos.Name, qos.New),
	)

	code := cli.Run(command)
	os.Exit(code)
}
```

在调用 `app.NewSchedulerCommand` 时，通过 `app.WithPlugin` 选项模式，传入了期望加载到 `kube-scheduler` 中的 out-of-tree 插件。

- 开发 out-of-tree 插件时，为了避免改动 kubernetes 源码仓库中的 `kube-scheduler` 源码，我们一般会另启动一个项目，例如：[scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins/blob/v0.27.8/cmd/scheduler/main.go) 。在新项目中我们调用 Kubernetes 源码仓库中的 app 包，来创建一个跟 kube-scheduler 完全一致的调度组件。
- 因为创建应用时，直接调用的是 `k8s.io/kubernetes/cmd/kube-scheduler/app.NewSchedulerCommand` 函数，所以创建的调度器跟 `kube-scheduler` 能够完全保持兼容，也就是说从 配置、使用方式、逻辑等等各个方面，都跟 kubernetes 的调度器完全一样，唯一不同就是加载了指定的 out-of-tree 插件。



一个具体的外部插件实现可以参考：[PodState](https://github.com/kubernetes-sigs/scheduler-plugins/blob/v0.27.8/pkg/podstate/pod_state.go#L100) 调度插件

<br/>

### In-Tree 插件初始化

在 kube-scheduler 中用 **scheduler.New** 方法中，通过 `registry := frameworkplugins.NewInTreeRegistry() ` 创建了 in-tree 插件。 **NewInTreeRegistry** 函数实现如下：

```go
func NewInTreeRegistry() runtime.Registry {
    fts := plfeature.Features{
        EnableDynamicResourceAllocation:              feature.DefaultFeatureGate.Enabled(features.DynamicResourceAllocation),
        EnableReadWriteOncePod:                       feature.DefaultFeatureGate.Enabled(features.ReadWriteOncePod),
        EnableVolumeCapacityPriority:                 feature.DefaultFeatureGate.Enabled(features.VolumeCapacityPriority),
        EnableMinDomainsInPodTopologySpread:          feature.DefaultFeatureGate.Enabled(features.MinDomainsInPodTopologySpread),
        EnableNodeInclusionPolicyInPodTopologySpread: feature.DefaultFeatureGate.Enabled(features.NodeInclusionPolicyInPodTopologySpread),
        EnableMatchLabelKeysInPodTopologySpread:      feature.DefaultFeatureGate.Enabled(features.MatchLabelKeysInPodTopologySpread),
        EnablePodSchedulingReadiness:                 feature.DefaultFeatureGate.Enabled(features.PodSchedulingReadiness),
        EnablePodDisruptionConditions:                feature.DefaultFeatureGate.Enabled(features.PodDisruptionConditions),
        EnableInPlacePodVerticalScaling:              feature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling),
        EnableSidecarContainers:                      feature.DefaultFeatureGate.Enabled(features.SidecarContainers),
    }

    registry := runtime.Registry{
        dynamicresources.Name:                runtime.FactoryAdapter(fts, dynamicresources.New),
        imagelocality.Name:                   imagelocality.New,
        tainttoleration.Name:                 tainttoleration.New,
        nodename.Name:                        nodename.New,
        nodeports.Name:                       nodeports.New,
        nodeaffinity.Name:                    nodeaffinity.New,
        podtopologyspread.Name:               runtime.FactoryAdapter(fts, podtopologyspread.New),
        nodeunschedulable.Name:               nodeunschedulable.New,
        noderesources.Name:                   runtime.FactoryAdapter(fts, noderesources.NewFit),
        noderesources.BalancedAllocationName: runtime.FactoryAdapter(fts, noderesources.NewBalancedAllocation),
        volumebinding.Name:                   runtime.FactoryAdapter(fts, volumebinding.New),
        volumerestrictions.Name:              runtime.FactoryAdapter(fts, volumerestrictions.New),
        volumezone.Name:                      volumezone.New,
        nodevolumelimits.CSIName:             runtime.FactoryAdapter(fts, nodevolumelimits.NewCSI),
        nodevolumelimits.EBSName:             runtime.FactoryAdapter(fts, nodevolumelimits.NewEBS),
        nodevolumelimits.GCEPDName:           runtime.FactoryAdapter(fts, nodevolumelimits.NewGCEPD),
        nodevolumelimits.AzureDiskName:       runtime.FactoryAdapter(fts, nodevolumelimits.NewAzureDisk),
        nodevolumelimits.CinderName:          runtime.FactoryAdapter(fts, nodevolumelimits.NewCinder),
        interpodaffinity.Name:                interpodaffinity.New,
        queuesort.Name:                       queuesort.New,
        defaultbinder.Name:                   defaultbinder.New,
        defaultpreemption.Name:               runtime.FactoryAdapter(fts, defaultpreemption.New),
        schedulinggates.Name:                 runtime.FactoryAdapter(fts, schedulinggates.New),
    }

    return registry
}
```



kube-scheduler 支持的 In-Tree 有诸如 PrioritySort（按优先级排序）、DefaultPreemption（抢占调度，高优先级踢掉低优先级 Pod ）、InterPodAffinity（根据Pod之间的亲和性关系，调度具有相关性的 Pod 到同一节点上）等等。



如 NodeName In-Tree 插件的实现:

`NodeName` 插件实现位于：[pkg/scheduler/framework/plugins/nodename/node_name.go](https://github.com/kubernetes/kubernetes/blob/v1.29.3/pkg/scheduler/framework/plugins/nodename/node_name.go) 文件中。`node_name.go` 文件内容如下：

```go
// NodeName is a plugin that checks if a pod spec node name matches the current node.
type NodeName struct{}

var _ framework.FilterPlugin = &NodeName{}
var _ framework.EnqueueExtensions = &NodeName{}

const (
	// Name is the name of the plugin used in the plugin registry and configurations.
	Name = names.NodeName

	// ErrReason returned when node name doesn't match.
	ErrReason = "node(s) didn't match the requested node name"
)

// EventsToRegister returns the possible events that may make a Pod
// failed by this plugin schedulable.
func (pl *NodeName) EventsToRegister() []framework.ClusterEventWithHint {
	return []framework.ClusterEventWithHint{
		{Event: framework.ClusterEvent{Resource: framework.Node, ActionType: framework.Add | framework.Update}},
	}
}

// Name returns name of the plugin. It is used in logs, etc.
func (pl *NodeName) Name() string {
	return Name
}

// Filter invoked at the filter extension point.
func (pl *NodeName) Filter(ctx context.Context, _ *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {

	if !Fits(pod, nodeInfo) {
		return framework.NewStatus(framework.UnschedulableAndUnresolvable, ErrReason)
	}
	return nil
}

// Fits actually checks if the pod fits the node.
func Fits(pod *v1.Pod, nodeInfo *framework.NodeInfo) bool {
	return len(pod.Spec.NodeName) == 0 || pod.Spec.NodeName == nodeInfo.Node().Name
}

// New initializes a new plugin and returns it.
func New(_ context.Context, _ runtime.Object, _ framework.Handle) (framework.Plugin, error) {
	return &NodeName{}, nil
}
```



