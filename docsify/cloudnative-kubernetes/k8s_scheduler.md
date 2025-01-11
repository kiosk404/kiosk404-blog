# Scheduler 解读

Kubernetes（K8s）作为一个开源的容器编排平台，已经成为现代分布式系统的核心组件。调度器是 Kubernetes 的重要组成部分，负责将 Pod 分配到最适合的节点上。本文将详细解读 Kubernetes 调度器的架构、原理以及调度过程，为程序员和 K8S 运维专家提供深度技术参考。

以下代码基于 kubernetes [v1.20](https://github.com/kubernetes/kubernetes/tree/release-1.20) 分析 

原文：[k8s-src-analysis](https://github.com/jindezgm/k8s-src-analysis/tree/master/kube-scheduler)



# 什么是 Kubernetes 调度器？

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

# 调度器简介

