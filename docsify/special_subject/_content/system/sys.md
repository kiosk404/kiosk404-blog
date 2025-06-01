## NUMA

简单来说，**NUMA （Non-Uniform Memory Access，非统一内存访问）** 是一种多处理器计算机体系结构。

在 **NUMA** 架构中，计算机的处理器（CPU）被分组到多个“节点”中，每个节点都有自己的一组 **CPU** 和直接连接的本地内存。

**核心特点**：

1. **本地内存访问快**： 一个 CPU 访问自己节点上的内存（本地内存）速度非常快。
2. **远程内存访问慢**： 一个 CPU 访问其他节点上的内存（远程内存）速度会比较慢，延迟更高。

**为什么会出现 NUMA？**

随着 CPU 核心数量的增加，传统的 UMA (Uniform Memory Access，统一内存访问) 架构（所有 CPU 共享同一块内存，访问速度一致）遇到了瓶颈。所有 CPU 都通过一个共享的总线访问内存，这个总线会成为性能瓶颈。

NUMA 的出现就是为了解决这个内存带宽瓶颈问题。通过将内存分散到多个节点，并让每个节点上的 CPU 优先访问本地内存，可以提高整体的内存带宽和系统可扩展性。

**对程序的影响：**

对于不了解 NUMA 的程序，如果一个进程或线程的数据被分配到远程内存节点，而其执行的 CPU 位于另一个节点，就会导致性能下降。
现代操作系统和应用程序（特别是数据库、高性能计算等）会尽量优化，让 CPU 访问其本地内存，以获得最佳性能。

**查看本机支持情况**
```bash
$ numactl --hardware
available: 1 nodes (0)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19
node 0 size: 31852 MB
node 0 free: 3147 MB
node distances:
node   0 
  0:  10 
```

我的机器只有一个 NUMA 节点（`NUMA node(s): 1` 和 `available: 1 nodes (0)`）。
这种架构实际上就是传统的 `UMA (Uniform Memory Access)` 架构。

> 因为我的机器是笔记本。**NUMA 的概念只存在于拥有多个独立内存控制器和内存区域的系统（通常是多路服务器）上**。

**NUMA 的概念只存在于拥有多个独立内存控制器和内存区域的系统（通常是多路服务器）上**

### K8S的NUMA优化

Kubernetes (K8s) 的 NUMA 优化对于运行高性能、延迟敏感或资源密集型工作负载（如数据库、高性能计算 HPC、NFV 等）至关重要。在多路服务器（拥有多个 CPU 插槽，每个插槽通常对应一个 NUMA 节点）上，如果 Pod 的 CPU 和内存没有被调度到同一个 NUMA 节点，就会导致显著的性能下降，因为跨节点访问内存会增加延迟。

K8s 本身并没有一个全局的、自动的 NUMA 感知调度器，但它提供了一些机制和组件来帮助实现 NUMA 优化。

K8s 的 NUMA 优化主要围绕资源隔离和拓扑感知调度展开。

**CPU Manager 实现 CPU 亲和性和 NUMA 优化的原理**

`CPU Manager` 是 `kubelet` 的一个核心组件，它负责在 K8s 节点上对 CPU 资源进行精细化管理。当其配置策略设置为 `static` 时，它能够为满足特定条件的 Pod 提供独占的 CPU 核心，并尽可能地将这些核心与 NUMA 节点对齐，从而实现 CPU 亲和性和 NUMA 优化。

其实现原理主要涉及以下几个方面：

1.  **资源管理器的注册：**
    * `CPU Manager` 作为 `kubelet` 中的一个资源管理器模块，在 `kubelet` 启动时会注册自己。这使得它能够在 Pod 的生命周期管理中（尤其是在 Pod 被调度到该节点并准备启动时）介入 CPU 资源的分配过程。

2.  **独占 CPU 核心池的维护：**
    * 当 `static` 策略被激活时，`CPU Manager` 会在节点上识别并维护一个**可用于独占分配的 CPU 核心池**。通常，CPU 0 会被保留用于处理操作系统和 `BestEffort`/`Burstable` QoS 级别 Pod 的任务，以避免核心争用。
    * 这个池中的每个 CPU 核心都有其唯一的 ID，`CPU Manager` 能够精确地追踪哪些核心是空闲的、哪些已经被分配。

3.  **QoS 级别和整数 CPU 请求的限定：**
    * `static` 策略并非对所有 Pod 都生效。它只作用于以下类型的 Pod：
        * **QoS 级别为 `Guaranteed`：** 即 Pod 的 `requests.cpu` 必须等于 `limits.cpu`，并且 `requests.memory` 必须等于 `limits.memory`。这保证了 Pod 获得最高优先级的资源保障。
        * **请求整数个 CPU：** Pod 的 `requests.cpu` 和 `limits.cpu` 必须是整数值（例如 `"1"`、`"2"`），而不是分数（例如 `"0.5"`）。只有整数请求才能对应到独占的物理或逻辑核心。

4.  **CPU 分配与 NUMA 感知逻辑：**
    * 当一个满足上述条件的 Pod 被调度到当前 `kubelet` 所在的节点上时，`CPU Manager` 会为其分配 CPU 核心：
        * **NUMA 感知优先：** `CPU Manager` 会查询底层的 NUMA 拓扑信息。这些信息通常通过 Linux 内核在 `/sys/devices/system/node/` 等路径下暴露。`CPU Manager` 会优先尝试从**单个 NUMA 节点**上选择足够数量的可用 CPU 核心，以确保 Pod 的所有 CPU 都可以在同一个 NUMA 域内。
        * **最佳匹配：** 如果一个 NUMA 节点无法提供所有所需的 CPU 核心，`CPU Manager` 可能会根据内部逻辑（例如，是否允许跨 NUMA 节点分配，或者根据 `Topology Manager` 的策略）来决定下一步。
        * **绑定 (Pinning)：** 一旦为 Pod 确定了 CPU 核心集合，`CPU Manager` 会利用 Linux 内核的 **`cgroup` (Control Group) 机制**。具体来说，它会操作 Pod 对应的 `cgroup` 中 `cpuset` 控制器下的 `cpuset.cpus` 文件，将 Pod 内所有进程和线程的执行限制在这些特定的 CPU 核心上。这种绑定确保了 Pod 的 CPU 亲和性，避免了进程在不同核心间的频繁迁移导致的缓存失效。

5.  **与 Topology Manager 的协作：**
    * `Topology Manager` 是 Kubernetes 1.16+ 引入的关键组件，它负责协调 `kubelet` 内的多个资源管理器（包括 `CPU Manager`、`HugePages` 管理器、设备插件等）。
    * 当 `Topology Manager` 被启用（特别是设置为 `single-numa-node` 策略）时，它会在资源分配前，先进行全局的拓扑对齐检查。它会告诉 `CPU Manager`：“请尝试从这个特定的 NUMA 节点分配 CPU。”
    * 这样，`CPU Manager` 就会收到来自 `Topology Manager` 的提示，并据此在指定的 NUMA 节点内分配 CPU 核心。同时，内存管理器也会尝试从该 NUMA 节点分配内存，从而实现 CPU 和内存的**双重 NUMA 亲和性**，最大化性能收益。

<br/>
------------------------------------------------------------------------------------------------

**源码链接和相关文件**
CPU Manager 的核心实现在 kubelet 内部。你可以在 Kubernetes 的 GitHub 仓库中找到相关的代码。

- **Kubernetes GitHub 仓库**： https://github.com/kubernetes/kubernetes
具体实现路径：

1. kubelet 目录： `pkg/kubelet/`
2. cpumanager 包： `pkg/kubelet/cm/cpumanager/`
    - manager.go： 这是 `CPU Manager` 的主要入口和接口定义。
    - policy_static.go： 实现了 `static` 策略的逻辑。你会在这里找到核心的 `CPU` 分配和绑定逻辑，以及 `NUMA` 节点选择的尝试。
    - topology 包： `pkg/kubelet/cm/topologymanager/` 包含了 `Topology Manager` 的实现，它与 `CPU Manager` 协同工作。

