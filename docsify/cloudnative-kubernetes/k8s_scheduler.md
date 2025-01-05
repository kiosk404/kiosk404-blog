# Scheduler 解读

Kubernetes（K8s）作为一个开源的容器编排平台，已经成为现代分布式系统的核心组件。调度器是 Kubernetes 的重要组成部分，负责将 Pod 分配到最适合的节点上。本文将详细解读 Kubernetes 调度器的架构、原理以及调度过程，为程序员和 K8S 运维专家提供深度技术参考。

以下代码基于 kubernetes [v1.20](https://github.com/kubernetes/kubernetes/tree/release-1.20) 分析 

原文：[k8s-src-analysis](https://github.com/jindezgm/k8s-src-analysis/tree/master/kube-scheduler)



## 什么是 Kubernetes 调度器？

Kubernetes 调度器（Scheduler）是负责将未绑定节点的 Pod 分配到集群中合适节点上的组件。其核心目标是根据各种策略、资源约束和调度算法，优化资源利用率并确保应用高效运行。



![kube-scheduler](https://img1.kiosk007.top/static/images/blog/20250105134502-kube-scheduler.png)

1. 利用SharedIndedInformer过滤未调度的Pod放入[调度队列](/k8s_scheduler?id=调度队列-schedulingqueue)
2. 利用SharedIndexInformer过滤已调度的Pod更新[调度缓存](/k8s_scheduler?id=调度缓存-cache)
3. 从 调度队列 取出一个待调度的Pod，通过Pod.Spec.SchedulerName获取[调度框架](/k8s_scheduler?id=调度框架-framework)
4. 调度框架 是配置好的[调度插件](/k8s_scheduler?id=调度插件-plugin)集合，既可以通过扩展 调度插件 的方式扩展调度能力，也可以通过[调度扩展器](/k8s_scheduler?id=调度扩展-extender)
5. [调度算法](/k8s_scheduler?id=调度算法-schedulealgorithm)利用 调度缓存 的快照以及输入的 调度框架 ，为Pod选择最优的节点
6. 如果 调度算法 执行失败，将Pod放入 调度队列 的不可调度自队列；如果 调度算法 执行成功，通知 调度缓存 假定调度Pod后异步绑定，绑定成功执行2
7. 其实 调度算法 并不是执行失败就将Pod放入不可调度队列，而是通过 调度插件 执行抢占调度，抢占成功的Pod会被[提名](/k8s_scheduler?id=pod提名-podnominator)并等待被强占调度Pod退出，只有抢占失败的Pod才会放入不可调度队列
8. 而且Pod假定调度完成后，不是立刻执行绑定，而是需要经过 调度插件 的ReservePlugin和PermitPlugin，如果PermitPlugin返回等待，则Pod需要[等待](/k8s_scheduler?id=pod等待-watingpods)直到批准或超时
9. 向SharedIndedInformer注册了Pod、Node、Service、CSINode、PV、PVC、StorageClass[事件处理函数](/k8s_scheduler?id=事件处理函数-enventhandler)
10. 以上调度流程通过[调度器](/k8s_scheduler?id=调度器-scheduler)对象实现，而 调度器 是通过[配置器](k8s_scheduler?id=配置器-configuration)构造的， 配置器 的配置主要来自[配置API](/k8s_scheduler?id=配置api-configurator)









<br/><br/><br/>

<!---->





# 调度队列 (SchedulingQueue)

- **功能**：管理待调度的 Pod。
- **实现**：主要包括活动队列（Active Queue）和未满足条件队列（Backoff Queue）。活动队列存储所有可以立即调度的 Pod，而后备队列则用于存储那些暂时无法调度的 Pod。

#### SchedulingQueue 的运作机制

在 Kubernetes 调度器中，SchedulingQueue 是调度流程的核心组件之一，用于管理待调度 Pod 的队列。它的主要作用是组织和优先排序待调度的 Pod，从而优化调度器的性能和响应速度。

1. **活动队列（Active Queue）**
   - **定义**：存储所有可以立即调度的 Pod。
   - **特性**：基于优先级排序，优先调度高优先级的 Pod。
2. **未满足条件队列（Backoff Queue）**
   - **定义**：存储暂时无法调度的 Pod，例如因为资源不足或节点不可用。
   - **特性**：通过延迟机制（Backoff）控制 Pod 的重试频率，避免调度器陷入忙等待。
3. **不可调度 Pod 集合（Unschedulable Pods）**
   - **定义**：用于存储目前无法满足调度条件的 Pod。
   - **作用**：当集群状态变化（如资源释放或节点加入）时，这些 Pod 会被重新评估。



这里提供一张简单的图总结一下调度队列：

<img src="https://img1.kiosk007.top/static/images/blog/20250105181540-SchedulingQueue.png" style="display:block;margin:0 auto;" />

1. 虽然名字上叫做调度队列，但是实际上都是按照map组织的，其实map也是一种队列，只是序是按照对象的key排列的（需要注意的是golang中的map是无序的），此处需要与数据结构中介绍的queue区分开。在调度过程中会频繁的判断对象是否已经存在（有一个高大上的名字叫幂等）、是否不存在，从一个子队列挪到另一个子队列，伴随添加、删除、更新等操作，map相比于其他数据结构是有优势的。
2. 调度队列虽然用map组织了Pod，同时用Pod的key的slice对Pod进行排序，这样更有队列的样子了。
3. 调度队列分为三个自队列：
   1. activeQ：即ready队列，里面都是准备调度的Pod，新添加的Pod都会被放到该队列，从调度队列中弹出需要调度的Pod也是从该队列获取；
   2. unschedulableQ：不可调度队列，里面有各种原因造成无法被调度Pod，至于有哪些原因，笔者在分析调度器的文章中会说明；
   3. backoffQ：退避队列，里面都是需要可以调度但是需要退避一段时间后才能调度的Pod；
4. 新创建的Pod通过Add()接口将Pod放入activeQ。 调度器每调度一个Pod都需要调用Pop()接口获取一个Pod，如果调度失败需要调用AddUnschedulableIfNotPresent()把Pod返回到调度队列中，如果调度成功了调用AssignedPodAdded()把依赖的Pod从unschedulableQ挪到activeQ或者backoffQ。
5. 调度队列后台每30秒刷一次unschedulableQ，如果入队列已经超过unschedulableQTimeInterval(60秒)，则将Pod从unschedulableQ移到activeQ，当然，如果还在退避时间内，只能放入backoffQ。
6. 调度队列后台每1秒刷一次backoffQ，把退避完成的所有Pod从backoffQ移到activeQ。
7. 虽然调度队列的实现是优先队列，但是基本没看到优先级相关的内容，是因为创建优先队列的时候需要传入lessFunc，该函数决定了activeQ的顺序，也就是调度优先级。所以调度优先级是可自定义的。
8. Add()和AddUnschedulableIfNotPresent()会设置Pod入队列时间戳，这个时间戳会用来计算Pod的退避时间、不可调度时间。







#### 调度队列中的Pod

虽然kubernetes定义了Pod的API对象 ( https://github.com/kubernetes/api/blob/release-1.20/core/v1/types.go#L3664 )，但是它只定义了Pod本身的属性，而Pod在调度队列中的属性是没有的，所以就有了QueuedPodInfo（代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/framework/types.go#L43 ）：

```go
// QueuedPodInfo是在Pod的API对象基础上增加了一些与调度队列相关的变量，所以在调度队列中管理的Pod对象是QueuedPodInfo类型的
type QueuedPodInfo struct {
    // 继承Pod的API类型
    Pod *v1.Pod
    // Pod添加到调度队列的时间。因为Pod可能会频繁的从调度队列中取出(用于调度)然后再放入调度队列(不可调度)
    // 所以每次进入队列时都会记录入队列的时间，这个时间作用很大，后面在分析调度队列的实现的时候会提到。
    Timestamp time.Time
    // Pod尝试调度的次数。应该说，正常的情况下Pod一次就会调度成功，但是在一些异常情况下（比如资源不足），Pod可能会被尝试调度多次
    Attempts int
    // Pod第一次添加到调度队列的时间，Pod调度成功前可能会多次加回队列，这个变量可以用来计算Pod的调度延迟（即从Pod入队到最终调度成功所用时间）
    InitialAttemptTimestamp time.Time
}
```

#### 堆（Heap）

这里提到的堆与排序有关，总所周知，golang的快排采用的是堆排序。所以不要与内存管理中"堆"概念混淆。

在kube-scheduler中，堆既有map的高效检索能力，有具备slice的顺序，这对于调度队列来说非常关键。因为调度对象随时可能添加、删除、更新，需要有高效的检索能力快速找到对象，map非常适合。但是golang中的map是无序的，访问map还有一定的随机性（每次range的第一个对象是随机的）。而调度经常会因为优先级、时间、依赖等原因需要对对象排序，slice非常合适，所以就有了堆这个类型。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/heap/heap.go#L127

```go
// Heap定义
type Heap struct {
    // Heap继承了data
    data *data
    // 监控相关，与本文内容无关
    metricRecorder metrics.MetricRecorder
}
// data是Heap核心实现
type data struct {
    // 用map管理所有对象
    items map[string]*heapItem
    // 在用slice管理所有对象的key，对象的key就是Pod的namespace+name，这是在kubernetes下唯一键
    queue []string
 
    // 获取对象key的函数，毕竟对象是interface{}类型的
    keyFunc KeyFunc
    // 判断两个对象哪个小的函数，了解sort.Sort()的读者此时应该能够猜到它是用来排序的。
    // 所以可以推断出来queue中key是根据lessFunc排过序的，而lessFunc又是传入进来的，
    // 这让Heap中对象的序可定制，这个非常有价值，非常有用
    lessFunc lessFunc
}
// 堆存储对象的定义
type heapItem struct {
    obj   interface{} // 对象，更准确的说应该是指针，应用在调度队列中就是*QueuedPodInfo
    index int         // Heap.queue的索引
}
```

本文不会对堆做详细的介绍，从堆的定义基本能够看出它的功能, 其本质就是实现了一个队列

#### 不可调度队列(UnschedulablePodsMap)



不可调度队列(UnschedulablePodsMap)管理暂时无法被调度（比如没有满足要求的Node）的Pod。虽然叫队列，其实是map实现，队列本质就是排队的缓冲，他与数据结构中的queue不是一个概念。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L665

```go
// 不可调度Pod的队列，就是对map的一种封装，可以简单的把它理解为map就可以了
type UnschedulablePodsMap struct {
    // 本质就是一个map，keyFunc与堆一样，用来获取对象的key
    podInfoMap map[string]*framework.QueuedPodInfo
    keyFunc    func(*v1.Pod) string
    // 监控相关，与本文内容无关
    metricRecorder metrics.MetricRecorder
}
```

#### 调度队列的抽象

golang开发习惯是用interface抽象一种接口，然后在用struct实现该接口。kube-scheduler对于调度队列也是有它的抽象，如下代码所示(https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L67)：

```go
type SchedulingQueue interface {
    // PodNominator其实与调度队列的功能关系不大，但是Pod在调度队列中的状态变化需要同步给PodNominator。
    // 本文不对PodNominator做详细说明，在其他后续分析调度器的文章中引用时再做说明。
    framework.PodNominator
    // 向队列中添加待调度的Pod，比如通过kubectl创建一个Pod时，kube-scheduler会通过该接口放入队列中.
    Add(pod *v1.Pod) error
    // 返回队列头部的pod，如果队列为空会被阻塞直到新的pod被添加到队列中.Add()和Pop()的组合有点数据结构中queue的感觉了，
    // 可能不是先入先出，这要通过lessFunc对Pod的进行排序，也就是本文后面提到的优先队列，按照Pod的优先级出队列。
    Pop() (*framework.QueuedPodInfo, error)
    // 首先需要理解什么是调度周期，kube-scheduler没调度一轮算作一个周期。向调度队列添加Pod不能算作一个调度周期，因为没有执行调度动作。
    // 只有从调度队列中弹出（Pop）才会执行调度动作，当然可能因为某些原因调度失败了，但是也算调度了一次，所以调度周期是在Pop()接口中统计的。
    // 每次pop一个pod就会加一，可以理解为调度队列的一种特殊的tick。用该接口可以获取当前的调度周期。
    SchedulingCycle() int64
    // 把无法调度的Pod添加回调度队列，前提条件是Pod不在调度队列中。podSchedulingCycle是通过调用SchedulingCycle（）返回的当前调度周期号。
    AddUnschedulableIfNotPresent(pod *framework.QueuedPodInfo, podSchedulingCycle int64) error
    // 更新pod
    Update(oldPod, newPod *v1.Pod) error
    // 删除pod
    Delete(pod *v1.Pod) error
    // 首选需要知道调度队列中至少包含activeQ和backoffQ，activeQ是所有ready等待调度的Pod，backoffQ是所有退避Pod。
    // 什么是退避？与退避三舍一个意思，退避的Pod不会被调度，即便优先级再高也没用。那退避到什么程度呢？调度队列用时间来衡量，比如1秒钟。
    // 对于kube-scheduler也是有退避策略的，退避时间按照尝试次数指数增长，但不是无限增长，有退避上限，默认的退避上限是10秒。
    // 在kube-scheduler中Pod退避的原因就是调度失败，退避就是为了减少无意义的频繁重试。
    // 把所有不可调度的Pod移到activeQ或者backoffQ中，至于哪些放到activeQ哪些放入backoffQ后面章节会有说明。
    MoveAllToActiveOrBackoffQueue(event string)
    // 当参数pod指向的Pod被调度后，把通过标签选择该Pod的所有不可调度的Pod移到activeQ，是不是有点绕？
    // 说的直白点就是当Pod1依赖(通过标签选择)Pod2时，在Pod2没有被调度前Pod1是不可调度的，当Pod2被调度后调度器就会调用该接口。
    AssignedPodAdded(pod *v1.Pod)
    // 与AssignedPodAdded一样，只是发生在更新时
    AssignedPodUpdated(pod *v1.Pod)
    // 获取所有挂起的Pod，其实就是队列中所有的Pod，因为调度队列中都是未调度（pending）的Pod
    PendingPods() []*v1.Pod
    // 关闭队列
    Close()
    // 获取队列中不可调度的pod数量
    NumUnschedulablePods() int
    // 启动协程管理队列
    Run()
}
```

#### 优先队列(PriorityQueue)

优先队列（PriorityQueue）实现了调度队列（SchedulingQueue），优先队列的头部是优先级最高的挂起（Pending）Pod。优先队列有三个子队列：一个子队列包含准备好调度的Pod，称为activeQ（是堆类型）；另一个队列包含已尝试并且确定为不可调度的Pod，称为unschedulableQ（UnschedulablePodsMap）；第三个队列是backoffQ,包含从unschedulableQ移出的Pod，退避完成后的Pod将其移到activeQ。源码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L113

```go
type PriorityQueue struct {
    // 与本文无关
    framework.PodNominator
    // 这两个不解释了，比较简单
    stop  chan struct{}
    clock util.Clock
 
    // Pod的初始退避时间，默认值是1秒，可配置.当Pod调度失败后第一次的退避时间就是podInitialBackoffDuration。
     // 第二次尝试调度失败后的退避时间就是podInitialBackoffDuration*2，第三次是podInitialBackoffDuration*4以此类推
    podInitialBackoffDuration time.Duration
    // Pod的最大退避时间，默认值是10秒，可配置。因为退避时间随着尝试次数指数增长，这个变量就是退避时间的上限值
    podMaxBackoffDuration time.Duration
 
    lock sync.RWMutex
    cond sync.Cond
 
    // activeQ，是Heap类型，前面解释过了
    activeQ *heap.Heap
    // backoffQ，也是Heap类型
    podBackoffQ *heap.Heap
    // unschedulableQ，详情见前面章节
    unschedulableQ *UnschedulablePodsMap
    // 调度周期，在Pop()中自增，SchedulingCycle()获取这个值
    schedulingCycle int64
    // 这个变量以调度周期为tick，记录上一次调用MoveAllToActiveOrBackoffQueue()的调度周期。
    // 因为调用AddUnschedulableIfNotPresent()接口的时候需要提供调度周期，当调度周期小于moveRequestCycle时，
    // 说明该不可调度Pod应该也被挪走，只是在调用接口的时候抢锁晚于MoveAllToActiveOrBackoffQueue()。具体后面会有代码注释。
    moveRequestCycle int64
 
    closed bool
}
```

#### Add

源码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L248



```go
func (p *PriorityQueue) Add(pod *v1.Pod) error {
    // 全程在锁的保护范围内，后续代码中都有相关的代码，不再赘述
    p.lock.Lock()
    defer p.lock.Unlock()
    // 把v1.Pod转成QueuedPodInfo，然后加入到activeQ。
    // 需要注意的是newQueuedPodInfo()会用当前时间设置pInfo.Timestamp和InitialAttemptTimestamp
    pInfo := p.newQueuedPodInfo(pod)
    if err := p.activeQ.Add(pInfo); err != nil {
        klog.Errorf("Error adding pod %v to the scheduling queue: %v", nsNameForPod(pod), err)
        return err
    }
    // 如果Pod在不可调度队列，那么从其中删除
    if p.unschedulableQ.get(pod) != nil {
        klog.Errorf("Error: pod %v is already in the unschedulable queue.", nsNameForPod(pod))
        p.unschedulableQ.delete(pod)
    }
    // 如果Pod在退避队列，那么从其中删除
    if err := p.podBackoffQ.Delete(pInfo); err == nil {
        klog.Errorf("Error: pod %v is already in the podBackoff queue.", nsNameForPod(pod))
    }
    // metrics和PodNominator与本文无关，后面章节不在注释
    metrics.SchedulerQueueIncomingPods.WithLabelValues("active", PodAdd).Inc(
    p.PodNominator.AddNominatedPod(pod, "")
    // 因为有新的Pod，所以需要唤醒所有被Pop()阻塞的协程，在注释Pop()的源码的时候会看到阻塞的部分。
    p.cond.Broadcast()
 
    return nil
}
```

#### AddUnschedulableIfNotPresent

代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L297

```go
// 如果调度队列中没有指定的不可调度的Pod，则将其加入调度队列。通常，PriorityQueue将无法调度的Pod放入`unschedulableQ`中。
// 但是，如果最近调用过MoveAllToActiveOrBackoffQueue()，则将pod放入`podBackoffQ`中。
func (p *PriorityQueue) AddUnschedulableIfNotPresent(pInfo *framework.QueuedPodInfo, podSchedulingCycle int64) error {
    p.lock.Lock()
    defer p.lock.Unlock()
    // 如果已经在不可调度队列存在，则返回错误
    pod := pInfo.Pod
    if p.unschedulableQ.get(pod) != nil {
        return fmt.Errorf("pod: %v is already present in unschedulable queue", nsNameForPod(pod))
    }
 
    // 更新Pod加入队列的时间戳，因为被重新加入队列。有没有感觉比较奇怪，为什么在此处更新时间戳，而不是在上面(判断不可调度队列之前)或者下面(判断退避队列之后)？
    // 在判断不可调度队列之前更新的话,如果已经在不可调度队列会有问题；在此处更新如果Pod已经在activeQ或者backoffQ会有问题，以backoffQ为例，
    // 因为更新后会让Pod退避继续延后，原本backoffQ中是按照退避完成时间排序，此处更新了相当于修改了退避时间，但是没有更新backoffQ中的排序。
    // 造成该Pod后面本应该退避完成了也只能等该Pod退避完后才能放到activeQ，具体可以参看flushBackoffQCompleted()的实现，后面章节会有这部分源码说明。
    // 笔者已经向社区提交BUG了！PR连接：https://github.com/kubernetes/kubernetes/pull/97302，并且已经被社区合并到了master分支
    pInfo.Timestamp = p.clock.Now()
    // 只要在activeQ或者podBackoffQ存在也返回错误，完全符合接口中IfNotPresent的定义
    if _, exists, _ := p.activeQ.Get(pInfo); exists {
        return fmt.Errorf("pod: %v is already present in the active queue", nsNameForPod(pod))
    }
    if _, exists, _ := p.podBackoffQ.Get(pInfo); exists {
        return fmt.Errorf("pod %v is already present in the backoff queue", nsNameForPod(pod))
    }
 
    // 此处就是判断该Pod是不是属于上次MoveAllToActiveOrBackoffQueue()的范围，如果是就把Pod放到退避队列，否则放到不可调度队列。
    // 为什么不判断是否应该放入activeQ？因为刚刚更新了入队时间，是不可能退避完成的，所以直接放入backoffQ没毛病。
    if p.moveRequestCycle >= podSchedulingCycle {
        if err := p.podBackoffQ.Add(pInfo); err != nil {
            return fmt.Errorf("error adding pod %v to the backoff queue: %v", pod.Name, err)
        }
        metrics.SchedulerQueueIncomingPods.WithLabelValues("backoff", ScheduleAttemptFailure).Inc()
    } else {
        p.unschedulableQ.addOrUpdate(pInfo)
        metrics.SchedulerQueueIncomingPods.WithLabelValues("unschedulable", ScheduleAttemptFailure).Inc()
    }
 
    p.PodNominator.AddNominatedPod(pod, "")
    return nil
}
```

#### Pop

代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L378



```go
// 从调度队列中弹出一个需要调度的Pod
func (p *PriorityQueue) Pop() (*framework.QueuedPodInfo, error) {
    p.lock.Lock()
    defer p.lock.Unlock()
    // 如果activeQ中没有Pod，则阻塞协程，当然如果调度队列已经关闭了除外。
    for p.activeQ.Len() == 0 {
        if p.closed {
            return nil, fmt.Errorf(queueClosed)
        }
        p.cond.Wait()
        // 为什么不再判断一下p.closed？必要性不大，因为后续的代码不会阻塞协程。
    }
    // 从activeQ弹出第一个Pod
    obj, err := p.activeQ.Pop()
    if err != nil {
        return nil, err
    }
    // 尝试调度的计数加1，调度周期加1，这些都比较容易理解
    pInfo := obj.(*framework.QueuedPodInfo)
    pInfo.Attempts++
    p.schedulingCycle++
    return pInfo, err
}
```



#### Update

代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L417

```go
func (p *PriorityQueue) Update(oldPod, newPod *v1.Pod) error {
    p.lock.Lock()
    defer p.lock.Unlock()
    // 更新Pod就需要先找到他在哪个子队列，但只有oldPod不为空才会在activeQ或者backoffQ查找，这是为什么？
    // 既然是更新oldPod就不应该为nil，笔者猜测是版本更新历史遗留问题，当前版本调用该接口的地方保证了oldPod肯定不为nil，详情参看下面如下代码链接
    // https://github.com/kubernetes/kubernetes/blob/release-1.19/pkg/scheduler/eventhandlers.go#L184
    if oldPod != nil {
        oldPodInfo := newQueuedPodInfoNoTimestamp(oldPod)
        // 如果Pod在activeQ中就更新它
        if oldPodInfo, exists, _ := p.activeQ.Get(oldPodInfo); exists {
            p.PodNominator.UpdateNominatedPod(oldPod, newPod)
            err := p.activeQ.Update(updatePod(oldPodInfo, newPod))
            return err
        }
        // 如果Pod在backoffQ中，就把他从backoffQ中移除，更新后放到activeQ中，并唤醒所有因为Pop()阻塞的协程。
        // 直接更新不行么？为什么需要从backoffQ移到activeQ？很简单，造成Pod退避的问题更新后可能就不存在了，所以立即再尝试调度一次。
        if oldPodInfo, exists, _ := p.podBackoffQ.Get(oldPodInfo); exists {
            p.PodNominator.UpdateNominatedPod(oldPod, newPod)
            p.podBackoffQ.Delete(oldPodInfo)
            err := p.activeQ.Add(updatePod(oldPodInfo, newPod))
            if err == nil {
                p.cond.Broadcast()
            }
            return err
        }
    }
 
    // 如果Pod在unschedulableQ中，则更新它。
    if usPodInfo := p.unschedulableQ.get(newPod); usPodInfo != nil {
        p.PodNominator.UpdateNominatedPod(oldPod, newPod)
        // 此处判断了Pod是否有更新，如果确实更新了就把他从unschedulableQ中移除，更新后放到activeQ中，道理和上面一样，不多说了。
        // 那么问题来了，为什么此处需要判断Pod是否确实更新了？为什么在backoffQ的处理代码中没有相应的判断？
        // 这就需要关联调度逻辑的上下文，等笔者分析调度器实现的文章再说明，此处先留一个坑。至少这个问题不影响本文的理解。
        if isPodUpdated(oldPod, newPod) {
            p.unschedulableQ.delete(usPodInfo.Pod)
            err := p.activeQ.Add(updatePod(usPodInfo, newPod))
            if err == nil {
                p.cond.Broadcast()
            }
            return err
        }
        // Pod没有实质的更新，那么就直接在unschedulableQ更新好了
        p.unschedulableQ.addOrUpdate(updatePod(usPodInfo, newPod))
        return nil
    }
    // 如果Pod没有在任何子队列中，那么就当新的Pod处理，为什么要有这个逻辑？
    // 1.正常的逻辑可能不会运行到这里，但是可能有某些异常情况，这里可以兜底，保证状态一致，没什么毛病。
    // 2.因为调度队列是支持并发的，在调用Update()的同时可能另一个协程刚刚删除了该Pod，更新其实就是覆盖，也约等于删除后添加。
    err := p.activeQ.Add(p.newQueuedPodInfo(newPod))
    if err == nil {
        p.PodNominator.AddNominatedPod(newPod, "")
        p.cond.Broadcast()
    }
    return err
}
```



#### Delete

```go
func (p *PriorityQueue) Delete(pod *v1.Pod) error {
    p.lock.Lock()
    defer p.lock.Unlock()
    p.PodNominator.DeleteNominatedPodIfExists(pod)
    // 删除实现比较简单，就是先从activeQ删除，如果删除失败多半是不存在，那么再从backoffQ和unschedulableQ删除。
    // 其实还可以判断一次backoffQ的删除错误代码，如果不存在再从unschedulableQ删除，毕竟Pod只可能出现在3个子队列中的一个队列中。
    // 当前的实现也没有问题，代码显得更简练一点，无非是效率上差一点点而已，可以忽略
    err := p.activeQ.Delete(newQueuedPodInfoNoTimestamp(pod))
    if err != nil { 
        p.podBackoffQ.Delete(newQueuedPodInfoNoTimestamp(pod))
        p.unschedulableQ.delete(pod)
    }
    return nil
}
```

#### MoveAllToActiveOrBackoffQueue

代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L500

```go
func (p *PriorityQueue) MoveAllToActiveOrBackoffQueue(event string) {
    p.lock.Lock()
    defer p.lock.Unlock()
    // 把map变成slice
    unschedulablePods := make([]*framework.QueuedPodInfo, 0, len(p.unschedulableQ.podInfoMap))
    for _, pInfo := range p.unschedulableQ.podInfoMap {
        unschedulablePods = append(unschedulablePods, pInfo)
    }
    // 调用movePodsToActiveOrBackoffQueue把所有不可调度的Pod移到activeQ或者backoffQ.
    p.movePodsToActiveOrBackoffQueue(unschedulablePods, event)
}
// 把不可调度的Pod移到activeQ或者backoffQ
func (p *PriorityQueue) movePodsToActiveOrBackoffQueue(podInfoList []*framework.QueuedPodInfo, event string) {
    for _, pInfo := range podInfoList {
        pod := pInfo.Pod
        // 判断Pod当前是否需要退避(判断方法见下面)，如果退避就放在backoffQ，当然放失败了还得放回unschedulableQ
        if p.isPodBackingoff(pInfo) {
            if err := p.podBackoffQ.Add(pInfo); err != nil {
                klog.Errorf("Error adding pod %v to the backoff queue: %v", pod.Name, err)
            } else {
                metrics.SchedulerQueueIncomingPods.WithLabelValues("backoff", event).Inc()
                p.unschedulableQ.delete(pod)
            }
        } else {
            // Pod无需退避就放在activeQ，放失败了还得放回unschedulableQ
            if err := p.activeQ.Add(pInfo); err != nil {
                klog.Errorf("Error adding pod %v to the scheduling queue: %v", pod.Name, err)
            } else {
                metrics.SchedulerQueueIncomingPods.WithLabelValues("active", event).Inc()
                p.unschedulableQ.delete(pod)
            }
        }
    }
    // 记录上一次移动的调度周期，在AddUnschedulableIfNotPresent()中用到了，到这里就比较清晰了。
    p.moveRequestCycle = p.schedulingCycle
    // 唤醒所有被Pop阻塞的协程，因为可能有Pod放入了activeQ。
    // 笔者认为这个实现不是很理想，应该增加一个bool型的局部变量，当有任何Pod放入activeQ成功后将该变量设置为true，
    // 然后根据该bool变量决定是否环境阻塞的协程，这样可以避免无效的唤醒。
    p.cond.Broadcast()
}
// 判断Pod是否需要退避
func (p *PriorityQueue) isPodBackingoff(podInfo *framework.QueuedPodInfo) bool {
    // 获取Pod需要退避的时间(见下面)，如果退避完成时间比当前还要晚，那么就需要退避
    boTime := p.getBackoffTime(podInfo)
    return boTime.After(p.clock.Now())
}
// 获取Pod的退避时间
func (p *PriorityQueue) getBackoffTime(podInfo *framework.QueuedPodInfo) time.Time {
    // 计算退避时长(见下面)
    duration := p.calculateBackoffDuration(podInfo)
    // 在加入队列的时间基础上+退避时长就是Pod的退避完成时间
    backoffTime := podInfo.Timestamp.Add(duration)
    return backoffTime
}
// 计算Pod退避时长
func (p *PriorityQueue) calculateBackoffDuration(podInfo *framework.QueuedPodInfo) time.Duration {
    // 在初始退避时长基础上，每尝试调度一次就在退避时长基础上乘以2，如果超过最大退避时长则按照最大值退避。
    // 这种方法比较常见，比如网络通信中连接失败，每次失败都等待一段时间再尝试连接，等待时间就是按照2的指数增长直到最大值。
    // 但是有没有感觉代码实现有点呆呆的？为什么不用移位(p.podInitialBackoffDuration<<podInfo.Attempts)的方法计算？
    // 因为很可能podInfo.Attempts是一个非常大的值，移位后就变成0了。但是这种实现还是不优雅，笔者准备向社区提交一个优雅的实现~
    duration := p.podInitialBackoffDuration
    for i := 1; i < podInfo.Attempts; i++ {
        duration = duration * 2
        if duration > p.podMaxBackoffDuration {
            return p.podMaxBackoffDuration
        }
    }
    return duration
}
```

#### PendingPods

代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L560

```go
// PendingPods的实现就是把三个子队列的Pod放到一个slice里面返回，比较简单
func (p *PriorityQueue) PendingPods() []*v1.Pod {
    p.lock.RLock()
    defer p.lock.RUnlock()
    var result []*v1.Pod
    // 先导出activeQ中的Pod
    for _, pInfo := range p.activeQ.List() {
        result = append(result, pInfo.(*framework.QueuedPodInfo).Pod)
    }
    // 再导出backoffQ中的Pod
    for _, pInfo := range p.podBackoffQ.List() {
        result = append(result, pInfo.(*framework.QueuedPodInfo).Pod)
    }
    // 最后导出unschedulableQ中的Pod
    for _, pInfo := range p.unschedulableQ.podInfoMap {
        result = append(result, pInfo.Pod)
    }
    return result
}
```



#### Run

代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L241

```go
func (p *PriorityQueue) Run() {
    // 启动两个协程，一个定时刷backoffQ，一个定时刷unschedulableQ，至于wait.Until的实现读者自行了解，比较简单.
    // 为什么是两个协程，1个协程不行么？这个从两个协程的定时周期可以知道，不可调度的Pod刷新周期要远远低于退避协程.
    // 合成一个协程实现需要取最小刷新周期，如果不可调度Pod比较多，会带来很多无效的计算。
    go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop)
    go wait.Until(p.flushUnschedulableQLeftover, 30*time.Second, p.stop)
}
// 把所有退避完成的Pod放到activeQ
func (p *PriorityQueue) flushBackoffQCompleted() {
    p.lock.Lock()
    defer p.lock.Unlock()
    for {
        // 此处需要知道的是，退避队列是按照退避完成时间排序的队列，还记得Heap中的lessFunc么？退避队列的lessFunc就是比较退避完成时间
        // Peek是返回队列第一个Pod但不从队列中删除（类似于C++中std::vector::front()），如果队列中没有Pod就可以返回了
        rawPodInfo := p.podBackoffQ.Peek()
        if rawPodInfo == nil {
            return
        }
        // 如果第一个Pod的退避完成时间还没到，那么队列中的所有Pod也都没有完成退避，因为队列是按照退避完成时间排序的，这种队列比较常见，没什么解释的了。
        pod := rawPodInfo.(*framework.QueuedPodInfo).Pod
        boTime := p.getBackoffTime(rawPodInfo.(*framework.QueuedPodInfo))
        if boTime.After(p.clock.Now()) {
            return
        }
        // 弹出第一个Pod，因为Pod退避时间结束了
        _, err := p.podBackoffQ.Pop()
        if err != nil {
            klog.Errorf("Unable to pop pod %v from backoff queue despite backoff completion.", nsNameForPod(pod))
            return
        }
        // 把Pod加入activeQ
        p.activeQ.Add(rawPodInfo)
        metrics.SchedulerQueueIncomingPods.WithLabelValues("active", BackoffComplete).Inc()
        // 至少有一个Pod放入到了activeQ，所以函数退出时需要唤醒被阻塞的协程。
        // 问：p.cond.Broadcast()会被执行多少次？答：当然是循环到这里几次就执行几次。
        // 问：需要唤醒几次？答：一次就可以
        // 当然循环多次意味着一秒内有多个Pod可能完成退避，这个概率不大，所以也没有啥问题，即便唤醒多次也没什么大不了的。
        // 笔者是完美主义者，得向社区提交点东西刷刷存在感~
        defer p.cond.Broadcast()
    }
}
// 不可调度队列并不意味着Pod不再调度，kube-scheduler有一个不可调度的间隔，如果Pod放入unschedulableQ超过该时间间隔将会重新尝试调度
// 说的直白点，不可调度的Pod每隔unschedulableQTimeInterval(60秒，可以理解刷新周期是30秒了吧)还会再尝试调度的.
func (p *PriorityQueue) flushUnschedulableQLeftover() {
    p.lock.Lock()
    defer p.lock.Unlock()
 
    // 取出在unschedulableQ中呆的时间超过不可调度间隔的所有Pod
    var podsToMove []*framework.QueuedPodInfo
    currentTime := p.clock.Now()
    for _, pInfo := range p.unschedulableQ.podInfoMap {
        lastScheduleTime := pInfo.Timestamp
        if currentTime.Sub(lastScheduleTime) > unschedulableQTimeInterval {
            podsToMove = append(podsToMove, pInfo)
        }
    }
    // 然后将这些Pod放到activeQ或者backoffQ中
    if len(podsToMove) > 0 {
        // 这个函数在前面已经介绍了
        p.movePodsToActiveOrBackoffQueue(podsToMove, UnschedulableTimeout)
    }
}
```

#### AssignedPodAdded/AssignedPodUpdated

代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L482

```go
// 这两个函数最终都是调用movePodsToActiveOrBackoffQueue把需要的Pod放入activeQ或者backoffQ的。
// 什么是需要的？getUnschedulablePodsWithMatchingAffinityTerm()给出了答案，简单说就是Pod的亲和性与指定另一个Pod有关，
// 比如Pod1不能与指定标签的Pod2在同一个Node上，那么Pod2没调度，Pod1是不能调度的，比较好理解吧。
func (p *PriorityQueue) AssignedPodAdded(pod *v1.Pod) {
    p.lock.Lock()
    p.movePodsToActiveOrBackoffQueue(p.getUnschedulablePodsWithMatchingAffinityTerm(pod), AssignedPodAdd)
    p.lock.Unlock()
}
func (p *PriorityQueue) AssignedPodUpdated(pod *v1.Pod) {
    p.lock.Lock()
    p.movePodsToActiveOrBackoffQueue(p.getUnschedulablePodsWithMatchingAffinityTerm(pod), AssignedPodUpdate)
    p.lock.Unlock()
}
```





<br/><br/><br/>

<!---->



# 调度缓存 (Cache)

调度队列（SchedulingQueue）中都是Pending状态的Pod，也就是未调度的Pod。而本文分析的Cache中都是已经调度的Pod（包括假定调度的Pod）。而Cache并不是仅仅为了存储已调度的Pod方便查找，而是为调度提供能非常重要的状态信息，甚至已经超越了Cache本身定义范畴。

既然定义为Cache，需要回答如下几个问题：

1. cache谁？kubernetes的信息都存储在etcd中，而访问kubernetes的etcd的唯一方法是通过apiserver，所以准确的说是缓存etcd的信息。
2. cache哪些信息？调度器需要将Pod调度到满足需求的Node上，所以cache至少要缓存Pod和Node信息，这样才能提高kube-scheduler访问apiserver的性能。
3. 为什么要cache？因为client-go 已经提供了cache能力，kube-scheduler增加一层 cache的目的是什么呢？答案很简单，为了调度。本文的Cache不仅缓存了Pod和Node信息，更关键的是聚合了调度结果，让调度变得更容易，也就是本文重点内容。



#### Cache 的定义

1. Cache缓存了Pod和Node信息，并且Node信息聚合了运行在该Node上所有Pod的资源量和镜像信息；Node有虚实之分，已删除的Node，Cache不会立刻删除它，而是继续维护一个虚的Node，直到Node上的Pod清零后才会被删除；但是nodeTree中维护的是实际的Node，调度使用nodeTree就可以避免将Pod调度到虚Node上；
2. kube-scheduler利用client-go监控(watch)Pod和Node状态，当有事件发生时调用Cache的AddPod，RemovePod，UpdatePod，AddNode，RemoveNode，UpdateNode更新Cache中Pod和Node的状态，这样kube-scheduler开始新一轮调度的时候可以获得最新的状态；
3. kube-scheduler每一轮调度都会调用UpdateSnapshot更新本地(局部变量)的Node状态，因为Cache中的Node按照最近更新排序，只需要将Cache中Node.Generation大于kube-scheduler本地的快照generation的Node更新到snapshot中即可，这样可以避免大量不必要的拷贝；
4. kube-scheduler找到合适的Node调度Pod后，需要调用Cache.AssumePod假定Pod已调度，然后启动协程异步Bind Pod到Node上，当Pod完成Bind后，调用Cache.FinishBinding通知Cache；
5. kube-scheudler调用Cache.AssumePod后续的所有造作一旦有错误就会调用Cache.ForgetPod删除假定的Pod，释放资源；
6. 完成Bind的Pod默认超时为30秒，Cache有一个协程定时(1秒)清理超时的Bind超时的Pod，如果超时依然没有收到Pod确认消息(调用AddPod)，则将删除超时的Pod，进而释放出Cache.AssumePod占用的资源;
7. Cache的核心功能就是统计Node的调度状态(比如累加Pod的资源量、统计镜像)，然后以镜像的形式输出给kube-scheduler，kube-scheduler从调度队列(SchedulingQueue)中取出等待调度的Pod，根据镜像计算最合适的Node；

此时再来看看源码中关于Pod状态机的注释就非常容易理解了：

```bash
// State Machine of a pod's events in scheduler's cache:
//
//
//   +-------------------------------------------+  +----+
//   |                            Add            |  |    |
//   |                                           |  |    | Update
//   +      Assume                Add            v  v    |
//Initial +--------> Assumed +------------+---> Added <--+
//   ^                +   +               |       +
//   |                |   |               |       |
//   |                |   |           Add |       | Remove
//   |                |   |               |       |
//   |                |   |               +       |
//   +----------------+   +-----------> Expired   +----> Deleted
//         Forget             Expire
//
```

上面总结中描述了kube-scheduler大致调度一个Pod的流程，其实kube-scheduler调度一个Pod的流程非常复杂



#### Cache 的抽象

源码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/interface.go#L58

```go
type Cache interface {
    // 获取node的数量，用于单元测试使用，本文不做说明
    NodeCount() int
    // 获取Pod的数量，用于单元测试使用，本文不做说明
    PodCount() (int, error)
 
    // 此处需要给出一个概念：假定Pod,就是将Pod假定调度到指定的Node,但还没有Bind完成。
    // 为什么要这么设计？因为kube-scheduler是通过异步的方式实现Bind，在Bind完成前，
    // 调度器还要调度新的Pod，此时就先假定Pod调度完成了。至于什么是Bind？为什么Bind？
    // 怎么Bind?笔者会在其他文章中解析，此处简单理解为：需要将Pod的调度结果写入etcd,
    // 持久化调度结果，所以也是相对比较耗时的操作。
    // AssumePod会将Pod的资源需求累加到Node上，这样kube-scheduler在调度其他Pod的时候,
    // 就不会占用这部分资源。
    AssumePod(pod *v1.Pod) error
 
    // 前面提到了，Bind是一个异步过程，当Bind完成后需要调用这个接口通知Cache，
    // 如果完成Bind的Pod长时间没有被确认(确认方法是AddPod)，那么Cache就会清理掉假定过期的Pod。
    FinishBinding(pod *v1.Pod) error
 
    // 删除假定的Pod，kube-scheduler在调用AssumePod后如果遇到其他错误，就需要调用这个接口
    ForgetPod(pod *v1.Pod) error
 
    // 添加Pod既确认了假定的Pod，也会将假定过期的Pod重新添加回来。
    AddPod(pod *v1.Pod) error
 
    // 更新Pod，其实就是删除再添加
    UpdatePod(oldPod, newPod *v1.Pod) error
 
    // 删除Pod.
    RemovePod(pod *v1.Pod) error
 
    // 获取Pod.
    GetPod(pod *v1.Pod) (*v1.Pod, error)
 
    // 判断Pod是否假定调度
    IsAssumedPod(pod *v1.Pod) (bool, error)
 
    // 添加Node的全部信息
    AddNode(node *v1.Node) error
 
    // 更新Node的全部信息
    UpdateNode(oldNode, newNode *v1.Node) error
 
    // 删除Node的全部信息
    RemoveNode(node *v1.Node) error
 
    // 其实就是产生Cache的快照并输出到nodeSnapshot中，那为什么是更新呢？
    // 因为快照比较大，产生快照也是一个比较重的任务，如果能够基于上次快照把增量的部分更新到上一次快照中，
    // 就会变得没那么重了，这就是接口名字是更新快照的原因。文章后面会重点分析这个函数，
    // 因为其他接口非常简单，理解了这个接口基本上就理解了Cache的精髓所在。
    UpdateSnapshot(nodeSnapshot *Snapshot) error
 
    // Dump会快照Cache，用于调试使用，不是重点，所以本文不会对该函数做说明。
    Dump() *Dump
}
```

从Cache的接口设计上可以看出，Cache只缓存了Pod和Node信息，而Pod和Node信息存储在etcd中(可以通过kubectl增删改查)，所以可以确认Cache缓存了etcd中的Pod和Node信息。

#### NodeInfo 的定义

在SchedulingQueue中，调度队列定义了QueuedPodInfo类型，在Pod API基础上扩展了与调度队列相关的属性。同样的道理，Node API只是Node的公共属性，而Cache中的Node需要扩展与Cache相关的属性，所以就有了NodeInfo这个类型。源码连接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/framework/types.go#L189

```go
// NodeInfo是Node层的汇聚信息
type NodeInfo struct {
    // Node API对象，无需过多解释
    node *v1.Node
    // 运行在Node上的所有Pod，PodInfo的定义读者自己查看，本文不再扩展了
    Pods []*PodInfo
    // PodsWithAffinity是Pods的子集，所有的Pod都声明了亲和性
    PodsWithAffinity []*PodInfo
    // PodsWithRequiredAntiAffinity是Pods子集，所有的Pod都声明了反亲和性
    PodsWithRequiredAntiAffinity []*PodInfo
    // 本文无关，忽略
    UsedPorts HostPortInfo
    // 此Node上所有Pod的总Request资源，包括假定的Pod，调度器已发送该Pod进行绑定，但可能尚未对其进行调度。
    Requested *Resource
    // Pod的容器资源请求有的时候是0，kube-scheduler为这类容器设置默认的资源最小值，并累加到NonZeroRequested.
    // 也就是说，NonZeroRequested等于Requested加上所有按照默认最小值累加的零资源
    // 这并不反映此节点的实际资源请求，而是用于避免将许多零资源请求的Pod调度到一个Node上。
    NonZeroRequested *Resource
    // Node的可分配的资源量
    Allocatable *Resource
    // 镜像状态，比如Node上有哪些镜像，镜像的大小，有多少Node相应的镜像等。
    ImageStates map[string]*ImageStateSummary
    // 与本文无关，忽略
    TransientInfo *TransientSchedulerInfo
    // 类似于版本，NodeInfo的任何状态变化都会使得Generation增加，比如有新的Pod调度到Node上
    // 这个Generation很重要，可以用于只复制变化的Node对象，后面更新镜像的时候会详细说明
    Generation int64
}
```



#### nodeTree

nodeTree是按照区域(zone)将Node组织成树状结构，当需要按区域列举或者全量列举按照区域排序，nodeTree就会用的上。为什么有这个需求，还是那句话，调度需要。举一个可能不恰当的例子：比如多个Pod的副本需要部署在同一个区域亦或是不同的区域。

源码连接：https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/internal/cache/node_tree.go#L32

```go
type nodeTree struct {
    // map的键是zone名字，map的值是该区域内所有Node的名字。
    tree     map[string][]string 
    // 所有的zone的名字
    zones    []string
    // Node的数量
    numNodes int
}
```

nodeTree只是把Node名字组织成树状，如果需要NodeInfo还需要根据Node的名字查找NodeInfo。

#### 快照

快照是对Cache某一时刻的复制，随着时间的推移，Cache的状态在持续更新，kube-scheduler在调度一个Pod的时候需要获取Cache的快照。相比于直接访问Cache，快照可以解决如下几个问题：

快照不会再有任何变化，可以理解为只读，那么访问快照不需要加锁保证保证原子性； 快照和Cache让读写分离，可以避免大范围的锁造成Cache访问性能下降； 虽然快照的状态从创建开始就落后于(因为Cache可能随时都会更新)Cache，但是对于kube-scheduler调度一个Pod来说是没问题的，至于原因笔者会在解析调度流程中加以说明。

源码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/snapshot.go#L29

```go
// 从定义上看，快照只有Node信息，没有Pod信息，其实Node信息中已经有Pod信息了，这个在NodeInfo中已经说明了
type Snapshot struct {
    // nodeInfoMap用于根据Node的key(NS+Name)快速查找Node
    nodeInfoMap map[string]*framework.NodeInfo
    // nodeInfoList是Cache中Node全集列表（不包含已删除的Node），按照nodeTree排序.
    nodeInfoList []*framework.NodeInfo
    // 只要Node上有任何Pod声明了亲和性，那么该Node就要放入havePodsWithAffinityNodeInfoList。
    // 为什么要有这个变量？当然是为了调度，比如PodA需要和PodB调度在一个Node上。
    havePodsWithAffinityNodeInfoList []*framework.NodeInfo
    // havePodsWithRequiredAntiAffinityNodeInfoList和havePodsWithAffinityNodeInfoList相似，
    // 只是Pod声明了反亲和，比如PodA不能和PodB调度在一个Node上
    havePodsWithRequiredAntiAffinityNodeInfoList []*framework.NodeInfo
    // generation是所有NodeInfo.Generation的最大值，因为所有NodeInfo.Generation都源于一个全局的Generation变量，
    // 那么Cache中的NodeInfo.Gerneraion大于该值的就是在快照产生后更新过的Node。
    // kube-scheduler调用Cache.UpdateSnapshot的时候只需要更新快照之后有变化的Node即可
    generation                                   int64
}
```

#### Cache 的实现

前面铺垫了已经足够了，现在开始进入重点内容，先来看看Cache实现类schedulerCache的定义。源码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L57

```go
// schedulerCache实现了Cache接口
type schedulerCache struct {
    // 这个比较好理解，用来通知schedulerCache停止的chan，说明schedulerCache有自己的协程
    stop   <-chan struct{}
    // 假定Pod一旦完成绑定，就要在指定的时间内确认，否则就会超时，ttl就是指定的过期时间，默认30秒
    ttl    time.Duration
    // 定时清理“假定过期”的Pod，period就是定时周期，默认是1秒钟
    // 前面提到了schedulerCache有自己的协程，就是定时清理超时的假定Pod.
    period time.Duration

    // 锁，说明schedulerCache利用互斥锁实现协程安全，而不是用chan与其他协程交互。
    // 这一点实现和SchedulingQueue是一样的。
    mu sync.RWMutex
    // 假定Pod集合，map的key与podStates相同，都是Pod的NS+NAME，值为true就是假定Pod
    // 其实assumedPods的值没有false的可能，感觉assumedPods用set类型(map[string]struct{}{})更合适
    assumedPods map[string]bool
    // 所有的Pod，此处用的是podState，后面有说明，与SchedulingQueue中提到的QueuedPodInfo类似，
    // podState继承了Pod的API定义，增加了Cache需要的属性
    podStates map[string]*podState
    // 所有的Node，键是Node.Name，值是nodeInfoListItem，后面会有说明，只需要知道map类型就可以了
    nodes     map[string]*nodeInfoListItem
    // 所有的Node再通过双向链表连接起来
    headNode *nodeInfoListItem
    // 节点按照zone组织成树状，前面提到用nodeTree中Node的名字再到nodes中就可以查找到NodeInfo.
    nodeTree *nodeTree
    // 镜像状态，本文不做重点说明，只需要知道Cache还统计了镜像的信息就可以了。
    imageStates map[string]*imageState
}
 
// podState与继承了Pod的API类型定义，同时扩展了schedulerCache需要的属性.
type podState struct {
    pod *v1.Pod
    // 假定Pod的超时截止时间，用于判断假定Pod是否过期。
    deadline *time.Time
    // 调用Cache.AssumePod的假定Pod不是所有的都需要判断是否过期，因为有些假定Pod可能还在Binding
    // bindingFinished就是用于标记已经Bind完成的Pod，然后开始计时，计时的方法就是设置deadline
    // 还记得Cache.FinishBinding接口么？就是用来设置bindingFinished和deadline的，后面代码会有解析
    bindingFinished bool
}
 
// nodeInfoListItem定义了nodeInfoList双向链表的item,nodeInfoList的实现非常简单，不多解释。
type nodeInfoListItem struct {
    info *framework.NodeInfo
    next *nodeInfoListItem
    prev *nodeInfoListItem
}
```

问题来了，既然已经有了nodes(map类型)变量，为什么还要再加一个headNode(list类型)的变量？这不是多此一举么？其实不然，nodes可以根据Node的名字快速找到Node，而headNode则是根据某个规则排过序的。这一点和SchedulingQueue中介绍的用map/slice实现队列是一个道理，至于为什么用list而不是slice，肯定是排序方法链表的效率高于slice，后面在更新headNode的地方再做说明，此处先排除疑虑。

从schedulerCache的定义基本可以猜到大部分Cache接口的实现，本文对于比较简单的接口实现只做简要说明，将文字落在一些重点的函数上。PodCount和NodeCount两个函数因为用于单元测试使用，本文不做说明。

#### AssumePod

当kube-scheduler找到最优的Node调度Pod的时候会调用AssumePod假定Pod调度，在通过另一个协程异步Bind。假定其实就是预先占住资源，kube-scheduler调度下一个Pod的时候不会把这部分资源抢走，直到收到确认消息AddPod确认调度成功，亦或是Bind失败ForgetPod取消假定调度。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L361

```go
func (cache *schedulerCache) AssumePod(pod *v1.Pod) error {
    // 获取Pod的唯一key，就是NS+Name，因为kube-scheduler调度整个集群的Pod
    key, err := framework.GetPodKey(pod)
    if err != nil {
        return err
    }
 
    cache.mu.Lock()
    defer cache.mu.Unlock()
    // 如果Pod已经存在，则不能假定调度。因为在Cache中的Pod要么是假定调度的，要么是完成调度的
    if _, ok := cache.podStates[key]; ok {
        return fmt.Errorf("pod %v is in the cache, so can't be assumed", key)
    }
    
    // 见下面代码注释
    cache.addPod(pod)
    ps := &podState{
        pod: pod,
    }
    // 把Pod添加到map中，并标记为assumed
    cache.podStates[key] = ps
    cache.assumedPods[key] = true
    return nil
}
 
func (cache *schedulerCache) addPod(pod *v1.Pod) {
    // 查找Pod调度的Node，如果不存在则创建一个虚Node，虚Node只是没有Node API对象。
    // 为什么会这样？可能kube-scheduler调度Pod的时候Node被删除了，可能很快还会添加回来
    // 也可能就彻底删除了，此时先放在这个虚的Node上，如果Node不存在后期还会被迁移。
    n, ok := cache.nodes[pod.Spec.NodeName]
    if !ok {
        n = newNodeInfoListItem(framework.NewNodeInfo())
        cache.nodes[pod.Spec.NodeName] = n
    }
    // AddPod就是把Pod的资源累加到NodeInfo中，本文不做详细说明，感兴趣的读者自行查看源码
    // 但需要知道的是n.info.AddPod(pod)会更新NodeInfo.Generation，表示NodeInfo是最新的
    n.info.AddPod(pod)
    // 将Node放到schedulerCache.headNode队列头部，因为NodeInfo当前是最新的，所以放在头部。
    // 此处可以解答为什么用list而不是slice，因为每次都是将Node直接放在第一个位置，明显list效率更高
    // 所以headNode是按照最近更新排序的
    cache.moveNodeInfoToHead(pod.Spec.NodeName)
}
```

#### ForgetPod

假定Pod预先占用了一些资源，如果之后的操作(比如Bind)有什么错误，就需要取消假定调度，释放出资源。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L406

```go
func (cache *schedulerCache) ForgetPod(pod *v1.Pod) error {
    // 获取Pod唯一key
    key, err := framework.GetPodKey(pod)
    if err != nil {
        return err
    }
 
    cache.mu.Lock()
    defer cache.mu.Unlock()
    // 这里有意思了，也就是说Cache假定Pod的Node名字与传入的Pod的Node名字不一致，则返回错误
    // 这种情况会不会发生呢?有可能，但是可能性不大，毕竟多协程修改Pod调度状态会有各种可能性。
    // 此处留挖一个坑，在解析kube-scheduler调度流程的时候看看到底什么极致的情况会触发这种问题。
    currState, ok := cache.podStates[key]
    if ok && currState.pod.Spec.NodeName != pod.Spec.NodeName {
        return fmt.Errorf("pod %v was assumed on %v but assigned to %v", key, pod.Spec.NodeName, currState.pod.Spec.NodeName)
    }
 
    switch {
    // 只有假定Pod可以被Forget，因为Forget就是为了取消假定Pod的。
    case ok && cache.assumedPods[key]:
        // removePod()就是把假定Pod的资源从NodeInfo中减去，见下面代码注释
        err := cache.removePod(pod)
        if err != nil {
            return err
        }
        // 删除Pod和假定状态
        delete(cache.assumedPods, key)
        delete(cache.podStates, key)
    // 要么Pod不存在，要么Pod已确认调度，这两者都不能够被Forget
    default:
        return fmt.Errorf("pod %v wasn't assumed so cannot be forgotten", key)
    }
    return nil
}
 
func (cache *schedulerCache) removePod(pod *v1.Pod) error {
    // 找到假定Pod调度的Node
    n, ok := cache.nodes[pod.Spec.NodeName]
    if !ok {
        klog.Errorf("node %v not found when trying to remove pod %v", pod.Spec.NodeName, pod.Name)
        return nil
    }
    // 减去假定Pod的资源，并从NodeInfo的Pod列表移除假定Pod
    // 和n.info.AddPod相同，也会更新NodeInfo.Generation
    if err := n.info.RemovePod(pod); err != nil {
        return err
    }
    // 如果NodeInfo的Pod列表没有任何Pod并且Node被删除，则Node从Cache中删除
    // 否则将NodeInfo移到列表头，因为NodeInfo被更新，需要放到表头
    // 这里需要知道的是，Node被删除Cache不会立刻删除该Node，需要等到Node上所有的Pod从Node中迁移后才删除，
    // 具体实现逻辑后续文章会给出，此处先知道即可。
    if len(n.info.Pods) == 0 && n.info.Node() == nil {
        cache.removeNodeInfoFromList(pod.Spec.NodeName)
    } else {
        cache.moveNodeInfoToHead(pod.Spec.NodeName)
    }
    return nil
}
```



#### FinishBinding

当假定Pod绑定完成后，需要调用FinishBinding通知Cache开始计时，直到假定Pod过期如果依然没有收到AddPod的请求，则将过期假定Pod删除。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L382

```go
func (cache *schedulerCache) FinishBinding(pod *v1.Pod) error {
    // 取当前时间
    return cache.finishBinding(pod, time.Now())
}
 
func (cache *schedulerCache) finishBinding(pod *v1.Pod, now time.Time) error {
    // 获取Pod唯一key
    key, err := framework.GetPodKey(pod)
    if err != nil {
        return err
    }
 
    cache.mu.RLock()
    defer cache.mu.RUnlock()
 
    klog.V(5).Infof("Finished binding for pod %v. Can be expired.", key)
    // Pod存在并且是假定状态才行
    currState, ok := cache.podStates[key]
    if ok && cache.assumedPods[key] {
        // 标记为完成Binding，并且设置过期时间，还记得ttl默认是多少么？30秒。
        dl := now.Add(cache.ttl)
        currState.bindingFinished = true
        currState.deadline = &dl
    }
    return nil
}
```

#### AddPod

当Pod Bind成功，kube-scheduler会收到消息，然后调用AddPod确认调度结果。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L476

```go
func (cache *schedulerCache) AddPod(pod *v1.Pod) error {
    // 获取Pod唯一key
    key, err := framework.GetPodKey(pod)
    if err != nil {
        return err
    }
 
    cache.mu.Lock()
    defer cache.mu.Unlock()
    
    // 以下是根据Pod在Cache中的状态决定需要如何处理
    currState, ok := cache.podStates[key]
    switch {
    // Pod是假定调度
    case ok && cache.assumedPods[key]:
        // Pod实际调度的Node和假定的不一致？
        if currState.pod.Spec.NodeName != pod.Spec.NodeName {
            klog.Warningf("Pod %v was assumed to be on %v but got added to %v", key, pod.Spec.NodeName, currState.pod.Spec.NodeName)
            // 如果不一致，先从假定调度的NodeInfo中减去Pod占用的资源，然后在累加到新NodeInfo中
            // 这种情况会在什么时候发生？还是留给后续文章分解吧
            if err = cache.removePod(currState.pod); err != nil {
                klog.Errorf("removing pod error: %v", err)
            }
            cache.addPod(pod)
        }
        // 删除假定状态
        delete(cache.assumedPods, key)
        // 清空假定过期时间，理论上从cache.assumedPods删除，假定过期时间自然也就失效了
        cache.podStates[key].deadline = nil
        // 这里有意思了，为什么要在赋值一次？currState中不是已经在AssumePod的时候设置了么？
        // 道理很简单，这是同一个Pod的两个副本，而当前参数‘pod’版本更新
        cache.podStates[key].pod = pod
    // Pod不存在
    case !ok:
        // Pod可能已经假定过期被删除了，需要重新添加回来
        cache.addPod(pod)
        ps := &podState{
            pod: pod,
        }
        cache.podStates[key] = ps
    // Pod已经执行过AddPod，有句高大上名词叫什么来着？对了，幂等！
    default:
        return fmt.Errorf("pod %v was already in added state", key)
    }
    return nil
}
```



#### RemovePod

kube-scheduler收到删除Pod的请求，如果Pod在Cache中，就需要调用RemovePod。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L541

```go
func (cache *schedulerCache) RemovePod(pod *v1.Pod) error {
    // 获取Pod唯一key
    key, err := framework.GetPodKey(pod)
    if err != nil {
        return err
    }
 
    cache.mu.Lock()
    defer cache.mu.Unlock()
 
    // 根据Pod在Cache中的状态执行相应的操作
    currState, ok := cache.podStates[key]
    switch {
    // 只有执行AddPod的Pod才能够执行RemovePod，假定Pod是不会执行RemovePod的，为什么？
    // 我只能说就是这么设计的，假定Pod是不会执行这个函数的，这涉及到Pod删除的全流程，
    // 已经超纲了。。。，我肯定会有文章解析，此处再挖一个坑。
    case ok && !cache.assumedPods[key]:
        // 卧槽，Pod的Node和AddPod时的Node不一样？这回的选择非常直接，奔溃，已经超时异常解决范围了
        // 如果再继续下去可能会造成调度状态的混乱，不如重启再来。
        if currState.pod.Spec.NodeName != pod.Spec.NodeName {
            klog.Errorf("Pod %v was assumed to be on %v but got added to %v", key, pod.Spec.NodeName, currState.pod.Spec.NodeName)
            klog.Fatalf("Schedulercache is corrupted and can badly affect scheduling decisions")
        }
        // 从NodeInfo中减去Pod的资源
        err := cache.removePod(currState.pod)
        if err != nil {
            return err
        }
        // 从Cache中删除Pod
        delete(cache.podStates, key)
    default:
        return fmt.Errorf("pod %v is not found in scheduler cache, so cannot be removed from it", key)
    }
    return nil
}
```



#### AddNode

有新的Node添加到集群，kube-scheduler调用该接口通知Cache。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L605

```go
func (cache *schedulerCache) AddNode(node *v1.Node) error {
    cache.mu.Lock()
    defer cache.mu.Unlock()
 
    n, ok := cache.nodes[node.Name]
    if !ok {
        // 如果NodeInfo不存在则创建
        n = newNodeInfoListItem(framework.NewNodeInfo())
        cache.nodes[node.Name] = n
    } else {
        // 已存在，先删除镜像状态，因为后面还会在添加回来
        cache.removeNodeImageStates(n.info.Node())
    }
    // 将Node放到列表头
    cache.moveNodeInfoToHead(node.Name)
 
    // 添加到nodeTree中
    cache.nodeTree.addNode(node)
    // 添加Node的镜像状态，感兴趣的读者自行了解，本文不做重点
    cache.addNodeImageStates(node, n.info)
    // 只有SetNode的NodeInfo才是真实的Node，否则就是前文提到的虚的Node
    return n.info.SetNode(node)
}
```

#### RemoveNode

Node从集群中删除，kube-scheduler调用该接口通知Cache。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L648

```go
func (cache *schedulerCache) RemoveNode(node *v1.Node) error {
    cache.mu.Lock()
    defer cache.mu.Unlock()
 
    // 如果Node不存在返回错误
    n, ok := cache.nodes[node.Name]
    if !ok {
        return fmt.Errorf("node %v is not found", node.Name)
    }
    // RemoveNode就是将*v1.Node设置为nil，此时Node就是虚的了
    n.info.RemoveNode()
    // 当Node上没有运行Pod的时候删除Node，否则把Node放在列表头，因为Node状态更新了
    // 熟悉etcd的同学会知道，watch两个路径(Node和Pod)是两个通道，这样会造成两个通道的事件不会按照严格时序到达
    // 这应该是存在虚Node的原因之一。
    if len(n.info.Pods) == 0 {
        cache.removeNodeInfoFromList(node.Name)
    } else {
        cache.moveNodeInfoToHead(node.Name)
    }
    // 虽然nodes只有在NodeInfo中Pod数量为零的时候才会被删除，但是nodeTree会直接删除
    // 说明nodeTree中体现了实际的Node状态，kube-scheduler调度Pod的时候也是利用nodeTree
    // 这样就不会将Pod调度到已经删除的Node上了。
    if err := cache.nodeTree.removeNode(node); err != nil {
        return err
    }
    cache.removeNodeImageStates(node)
    return nil
}
```

#### 后期清理协程函数run

前文提到过，Cache有自己的协程，就是用来清理假定到期的Pod。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L724

```go
func (cache *schedulerCache) run() {
    // 定时1秒钟执行一次cleanupExpiredAssumedPods
    go wait.Until(cache.cleanupExpiredAssumedPods, cache.period, cache.stop)
}
 
func (cache *schedulerCache) cleanupExpiredAssumedPods() {
    // 取当前时间
    cache.cleanupAssumedPods(time.Now())
}
 
func (cache *schedulerCache) cleanupAssumedPods(now time.Time) {
    cache.mu.Lock()
    defer cache.mu.Unlock()
    defer cache.updateMetrics()
 
    // 遍历假定Pod
    for key := range cache.assumedPods {
        // 获取Pod
        ps, ok := cache.podStates[key]
        if !ok {
            klog.Fatal("Key found in assumed set but not in podStates. Potentially a logical error.")
        }
        // 如果Pod没有标记为结束Binding，则忽略，说明Pod还在Binding中
        // 说白了就是没有调用FinishBinding的Pod不用处理
        if !ps.bindingFinished {
            klog.V(5).Infof("Couldn't expire cache for pod %v/%v. Binding is still in progress.",
                ps.pod.Namespace, ps.pod.Name)
            continue
        }
        // 如果当前时间已经超过了Pod假定过期时间，说明Pod假定时间已过期
        if now.After(*ps.deadline) {
            // 此类情况属于异常情况，所以日志等级是waning
            klog.Warningf("Pod %s/%s expired", ps.pod.Namespace, ps.pod.Name)
            // 清理假定过期的Pod
            if err := cache.expirePod(key, ps); err != nil {
                klog.Errorf("ExpirePod failed for %s: %v", key, err)
            }
        }
    }
}
 
func (cache *schedulerCache) expirePod(key string, ps *podState) error {
    // 从NodeInfo中减去Pod资源、镜像等状态
    if err := cache.removePod(ps.pod); err != nil {
        return err
    }
    // 从Cache中删除Pod
    delete(cache.assumedPods, key)
    delete(cache.podStates, key)
    return nil
}
```

其实这里有一个比较严重的问题：如果假定过期的Pod资源刚刚会被释放，又有新Pod调度到了与刚刚假定过期Pod相同的Node上，此后Pod被AddPod添加回来，可能会让Node的资源过载。

#### UpdateSnapshot

好了，前文那么多的铺垫，都是为了UpdateSnapshot，因为Cache存在的核心目的就是给kube-scheduler提供Node镜像，让kube-scheduler根据Node的状态调度新的Pod。而Cache中的Pod是为了计算Node的资源状态存在的，毕竟二者在etcd中是两个路径。话不多说，直接上代码。代码链接：https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/cache/cache.go#L203

```go
// UpdateSnapshot更新的是参数nodeSnapshot，不是更新Cache.
// 也就是Cache需要找到当前与nodeSnapshot的差异，然后更新它，这样nodeSnapshot就与Cache状态一致了
// 至少从函数执行完毕后是一致的。
func (cache *schedulerCache) UpdateSnapshot(nodeSnapshot *Snapshot) error {
    cache.mu.Lock()
    defer cache.mu.Unlock()
    // 与本文关系不大，鉴于不增加复杂性原则，先忽略他，从命名上看很容易立理解
    balancedVolumesEnabled := utilfeature.DefaultFeatureGate.Enabled(features.BalanceAttachedNodeVolumes)
 
    // 获取nodeSnapshot的版本，笔者习惯叫版本，其实就是版本的概念。
    // 此处需要多说一点：kube-scheudler为Node定义了全局的generation变量，每个Node状态变化都会造成generation+=1然后赋值给该Node
    // nodeSnapshot.generation就是最新NodeInfo.Generation，就是表头的那个NodeInfo。
    snapshotGeneration := nodeSnapshot.generation
 
    // 介绍Snapshot的时候提到了，快照中有三个列表，分别是全量、亲和性和反亲和性列表
    // 全量列表在没有Node添加或者删除的时候，是不需要更新的
    updateAllLists := false
    // 当有Node的亲和性状态发生了变化(以前没有任何Pod有亲和性声明现在有了，抑或反过来)，
    // 则需要更新快照中的亲和性列表
    updateNodesHavePodsWithAffinity := false
    // 同上
    updateNodesHavePodsWithRequiredAntiAffinity := false
 
    // 遍历Node列表，为什么不遍历Node的map？因为Node列表是按照Generation排序的
    // 只要找到大于nodeSnapshot.generation的所有Node然后把他们更新到nodeSnapshot中就可以了
    for node := cache.headNode; node != nil; node = node.next {
        // 说明Node的状态已经在nodeSnapshot中了，因为但凡Node有任何更新，那么NodeInfo.Generation 
        // 肯定会大于snapshotGeneration，同时该Node后面的所有Node也不用在遍历了，因为他们的版本更低
        if node.info.Generation <= snapshotGeneration {
            break
        }
        // 先忽略
        if balancedVolumesEnabled && node.info.TransientInfo != nil {
            // Transient scheduler info is reset here.
            node.info.TransientInfo.ResetTransientSchedulerInfo()
        }
        // node.info.Node()获取*v1.Node，前文说了，如果Node被删除，那么该值就是为nil
        // 所以只有未被删除的Node才会被更新到nodeSnapshot，因为快照中的全量Node列表是按照nodeTree排序的
        // 而nodeTree都是真实的node
        if np := node.info.Node(); np != nil {
            // 如果nodeSnapshot中没有该Node，则在nodeSnapshot中创建Node，并标记更新全量列表，因为创建了新的Node
            existing, ok := nodeSnapshot.nodeInfoMap[np.Name]
            if !ok {
                updateAllLists = true
                existing = &framework.NodeInfo{}
                nodeSnapshot.nodeInfoMap[np.Name] = existing
            }
            // 克隆NodeInfo，这个比较好理解，肯定不能简单的把指针设置过去，这样会造成多协程读写同一个对象
            // 因为克隆操作比较重，所以能少做就少做，这也是利用Generation实现增量更新的原因
            clone := node.info.Clone()
            // 如果Pod以前或者现在有任何亲和性声明，则需要更新nodeSnapshot中的亲和性列表
            if (len(existing.PodsWithAffinity) > 0) != (len(clone.PodsWithAffinity) > 0) {
                updateNodesHavePodsWithAffinity = true
            }
            // 同上，需要更新非亲和性列表
            if (len(existing.PodsWithRequiredAntiAffinity) > 0) != (len(clone.PodsWithRequiredAntiAffinity) > 0) {
                updateNodesHavePodsWithRequiredAntiAffinity = true
            }
            // 将NodeInfo的拷贝更新到nodeSnapshot中
            *existing = *clone
        }
    }
    // Cache的表头Node的版本是最新的，所以也就代表了此时更新镜像后镜像的版本了
    if cache.headNode != nil {
        nodeSnapshot.generation = cache.headNode.info.Generation
    }
 
    // 如果nodeSnapshot中node的数量大于nodeTree中的数量，说明有node被删除
    // 所以要从快照的nodeInfoMap中删除已删除的Node，同时标记需要更新node的全量列表
    if len(nodeSnapshot.nodeInfoMap) > cache.nodeTree.numNodes {
        cache.removeDeletedNodesFromSnapshot(nodeSnapshot)
        updateAllLists = true
    }
 
    // 如果需要更新Node的全量或者亲和性或者反亲和性列表，则更新nodeSnapshot中的Node列表
    if updateAllLists || updateNodesHavePodsWithAffinity || updateNodesHavePodsWithRequiredAntiAffinity {
        cache.updateNodeInfoSnapshotList(nodeSnapshot, updateAllLists)
    }
 
    // 如果此时nodeSnapshot的node列表与nodeTree的数量还不一致，需要再做一次node全列表更新
    // 此处应该是一个保险操作，理论上不会发生，谁知道会不会有Bug发生呢？多一些容错没有坏处
    if len(nodeSnapshot.nodeInfoList) != cache.nodeTree.numNodes {
        errMsg := fmt.Sprintf("snapshot state is not consistent, length of NodeInfoList=%v not equal to length of nodes in tree=%v "+
            ", length of NodeInfoMap=%v, length of nodes in cache=%v"+
            ", trying to recover",
            len(nodeSnapshot.nodeInfoList), cache.nodeTree.numNodes,
            len(nodeSnapshot.nodeInfoMap), len(cache.nodes))
        klog.Error(errMsg)
        // We will try to recover by re-creating the lists for the next scheduling cycle, but still return an
        // error to surface the problem, the error will likely cause a failure to the current scheduling cycle.
        cache.updateNodeInfoSnapshotList(nodeSnapshot, true)
        return fmt.Errorf(errMsg)
    }
 
    return nil
}
 
// 先思考一个问题：为什么有Node添加或者删除需要更新快照中的全量列表？如果是Node删除了，
// 需要找到Node在全量列表中的位置，然后删除它，最悲观的复杂度就是遍历一遍列表，然后再挪动它后面的Node
// 因为快照的Node列表是用slice实现，所以一旦快照中Node列表有任何更新，复杂度都是Node的数量。
// 那如果是有新的Node添加呢？并不知道应该插在哪里，所以重新创建一次全量列表最为简单有效。
// 亲和性和反亲和性列表道理也是一样的。
func (cache *schedulerCache) updateNodeInfoSnapshotList(snapshot *Snapshot, updateAll bool) {
    // 快照创建亲和性和反亲和性列表
    snapshot.havePodsWithAffinityNodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
    snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
    // 如果更新全量列表
    if updateAll {
        // 创建快照全量列表
        snapshot.nodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
        nodesList, err := cache.nodeTree.list()
        if err != nil {
            klog.Error(err)
        }
        // 遍历nodeTree的Node
        for _, nodeName := range nodesList {
            // 理论上快照的nodeInfoMap与nodeTree的状态是一致，此处做了判断用来检测BUG，下面的错误日志也是这么写的
            if nodeInfo := snapshot.nodeInfoMap[nodeName]; nodeInfo != nil {
                // 追加全量、亲和性(按需)、反亲和性列表(按需)
                snapshot.nodeInfoList = append(snapshot.nodeInfoList, nodeInfo)
                if len(nodeInfo.PodsWithAffinity) > 0 {
                    snapshot.havePodsWithAffinityNodeInfoList = append(snapshot.havePodsWithAffinityNodeInfoList, nodeInfo)
                }
                if len(nodeInfo.PodsWithRequiredAntiAffinity) > 0 {
                    snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = append(snapshot.havePodsWithRequiredAntiAffinityNodeInfoList, nodeInfo)
                }
            } else {
                klog.Errorf("node %q exist in nodeTree but not in NodeInfoMap, this should not happen.", nodeName)
            }
        }
    } else {
        // 如果更新全量列表，只需要遍历快照中的全量列表就可以了
        for _, nodeInfo := range snapshot.nodeInfoList {
            // 按需追加亲和性和反亲和性列表
            if len(nodeInfo.PodsWithAffinity) > 0 {
                snapshot.havePodsWithAffinityNodeInfoList = append(snapshot.havePodsWithAffinityNodeInfoList, nodeInfo)
            }
            if len(nodeInfo.PodsWithRequiredAntiAffinity) > 0 {
                snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = append(snapshot.havePodsWithRequiredAntiAffinityNodeInfoList, nodeInfo)
            }
        }
    }
}
```





<br/><br/><br/>

<!---->

# 调度框架 (Framework)



<br/><br/><br/>

<!---->

# 调度插件 (Plugin)

<br/><br/><br/>

<!---->

# 调度扩展 (Extender)

<br/><br/><br/>

<!---->

# 调度算法 (ScheduleAlgorithm)

<br/><br/><br/>

<!---->

# 调度器 (Scheduler)

<br/><br/><br/>

<!---->

# Pod提名 (PodNominator)



<br/><br/><br/>

<!---->



# 事件处理函数 (EnventHandler)

<br/><br/><br/>

<!---->

# Pod等待 (WatingPods)

<br/><br/><br/>

<!---->

# 配置器 (Configuration)

<br/><br/><br/>

<!---->

# 配置API (Configurator)

