---
title: Kubernetes网络方案
author: kiosk
draft: true
tags: [k8s]
categories:
  - k8s
date: 2025-03-12 11:23:00
---

Kubernetes 用来在集群上运行分布式系统。分布式系统的本质使得网络组件在 Kubernetes 中是至关重要也不可或缺的。

<!--more-->

网络是一个很宽泛的领域，其中有许多成熟的技术。对于不熟悉网络整体背景的人而言，要将各种新的概念、旧的概念放到一起来理解（例如，网络名称空间、虚拟网卡、IP forwarding、网络地址转换等），并融汇贯通，是一个非常困难的事情。

<img src="https://img1.kiosk007.top/static/images/blog/20250216121743-OIP-C.jpeg" style="display: block; margin: 0 auto; width:50%;" />

# 网络模型

关于 Pod 如何接入网络这件事情，Kubernetes 做出了明确的选择。具体来说，Kubernetes 要求所有的网络插件实现必须满足如下要求：

- 所有的 Pod 可以与任何其他 Pod 直接通信，无需使用 NAT 映射（network address translation）
- 所有节点可以与所有 Pod 直接通信，无需使用 NAT 映射
- Pod 内部获取到的 IP 地址与其他 Pod 或节点与其通信时的 IP 地址是同一个

在这些限制条件下，需要解决如下四种完全不同的网络使用场景的问题：

1. [Container-to-Container 的网络](#container-to-container-的网络)
2. [Pod-to-Pod 的网络](#pod-to-pod-的网络)
3. [Pod-to-Service 的网络](#service-to-pod-的网络)
4. [External-to-Service 的网络](#external-to-service-的网络)

<br/>

## Container-to-Container 的网络

在 Kubernetes 中，Pod 是一组 docker 容器的集合，这一组 docker 容器将共享一个 network namespace。Pod 中所有的容器都：

- 使用该 network namespace 提供的同一个 IP 地址以及同一个端口空间
- 可以通过 localhost 直接与同一个 Pod 中的另一个容器通信

Kubernetes 为每一个 Pod 都创建了一个 network namespace。具体做法是，把一个 Docker 容器当做 “Pod Container” 用来获取 network namespace，在创建 Pod 中新的容器时，都使用 docker run 的 `--network:container` 功能来加入该 network namespace，参考 [docker run reference (opens new window)](https://docs.docker.com/engine/reference/run/#network-settings)。如下图所示，每一个 Pod 都包含了多个 docker 容器（`ctr*`），这些容器都在同一个共享的 network namespace 中。

<img src="https://img1.kiosk007.top/static/images/blog/20250216112313-pod-namespace.5098bb9c.png" style="display: block; margin: 0 auto; width:70%;" />





<br/>

## Pod-to-Pod 的网络

在 Kubernetes 中，每一个 Pod 都有一个真实的 IP 地址，并且每一个 Pod 都可以使用此 IP 地址与 其他 Pod 通信。

> 下面介绍一种全虚拟化的 Pod-to-Pod 的网络通信方式。

从 Pod 的视角来看，Pod 是在其自身所在的 network namespace 与同节点上另外一个 network namespace 进程通信。在Linux上，不同的 network namespace 可以通过 [Virtual Ethernet Device (opens new window)](http://man7.org/linux/man-pages/man4/veth.4.html)或 ***veth pair*** (两块跨多个名称空间的虚拟网卡)进行通信。为连接 pod 的 network namespace，可以将 ***veth pair*** 的一段指定到 root network namespace，另一端指定到 Pod 的 network namespace。每一组 ***veth pair*** 类似于一条网线，连接两端，并可以使流量通过。节点上有多少个 Pod，就会设置多少组 ***veth pair***。

此时，我们的 Pod 都有了自己的 network namespace，从 Pod 的角度来看，他们都有自己的以太网卡以及 IP 地址，并且都连接到了节点的 root network namespace。为了让 Pod 可以互相通过 root network namespace 通信，我们将使用 network bridge（网桥）。

Linux Ethernet bridge 是一个虚拟的 Layer 2 网络设备，可用来连接两个或多个网段（network segment）。网桥的工作原理是，在源于目标之间维护一个转发表（forwarding table），通过检查通过网桥的数据包的目标地址（destination）和该转发表来决定是否将数据包转发到与网桥相连的另一个网段。桥接代码通过网络中具备唯一性的网卡MAC地址来判断是否桥接或丢弃数据。

<img src="https://img1.kiosk007.top/static/images/blog/20250216113334-pods-connected-by-bridge.8f775095.png" style="display: block; margin: 0 auto; width:70%;" />

<br/>

### 数据包的传递 (Pod-to-Pod, 同节点)

在 network namespace 将每一个 Pod 隔离到各自的网络堆栈的情况下，虚拟以太网设备（virtual Ethernet device）将每一个 namespace 连接到 root namespace，网桥将 namespace 又连接到一起，此时，Pod 可以向同一节点上的另一个 Pod 发送网络报文了。下图演示了同节点上，网络报文从一个Pod传递到另一个Pod的情况。

<img src="https://img1.kiosk007.top/static/images/blog/20250216113652-pod-to-pod-same-node.90e4d5a2.gif" style="display: block; margin: 0 auto; width:70%;" />

Pod1 发送一个数据包到其自己的默认以太网设备 `eth0`。

1. 对 Pod1 来说，`eth0` 通过虚拟以太网设备（veth0）连接到 root namespace
2. 网桥 `cbr0` 中为 `veth0` 配置了一个网段。一旦数据包到达网桥，网桥使用[ARP (opens new window)](https://en.wikipedia.org/wiki/Address_Resolution_Protocol)协议解析出其正确的目标网段 `veth1`
3. 网桥 `cbr0` 将数据包发送到 `veth1`
4. 数据包到达 `veth1` 时，被直接转发到 Pod2 的 network namespace 中的 `eth0` 网络设备。

在整个数据包传递过程中，每一个 Pod 都只和 `localhost` 上的 `eth0` 通信，且数包被路由到正确的 Pod 上。与开发人员正常使用网络的习惯没有差异。

<br/>

### 数据包的传递 (Pod-to-Pod, 跨节点)

Kubernetes 网络模型要求 Pod 的 IP 在整个网络中都可访问，但是并不指定如何实现这一点。实际上，这是所使用网络插件相关的，但是，仍然有一些模式已经被确立了。

通常，集群中每个节点都被分配了一个 CIDR 网段，指定了该节点上的 Pod 可用的 IP 地址段。一旦发送到该 CIDR 网段的流量到达节点，就由节点负责将流量继续转发给对应的 Pod。下图展示了两个节点之间的数据报文传递过程。



<img src="https://img1.kiosk007.top/static/images/blog/20250216114056-pod-to-pod-different-nodes.4187b249.gif" style="display: block; margin: 0 auto; width:70%;" />

目标 Pod（以绿色高亮）与源 Pod（以蓝色高亮）在不同的节点上，数据包传递过程如下：

1. 数据包从 Pod1 的网络设备 `eth0`，该设备通过 `veth0` 连接到 root namespace
2. 数据包到达 root namespace 中的网桥 `cbr0`
3. 网桥上执行 ARP 将会失败，因为与网桥连接的所有设备中，没有与该数据包匹配的 MAC 地址。一旦 ARP 失败，网桥会将数据包发送到默认路由（root namespace 中的 `eth0` 设备）。此时，数据包离开节点进入网络
4. 假设网络可以根据各节点的CIDR网段，将数据包路由到正确的节点
5. 数据包进入目标节点的 root namespace（VM2 上的 `eth0`）后，通过网桥路由到正确的虚拟网络设备（`veth1`）
6. 最终，数据包通过 `veth1` 发送到对应 Pod 的 `eth0`，完成了数据包传递的过程

<br/>

通常来说，每个节点知道如何将数据包分发到运行在该节点上的 Pod。一旦一个数据包到达目标节点，数据包的传递方式与同节点上不同Pod之间数据包传递的方式就是一样的了。

此处，我们直接跳过了如何配置网络，以使得数据包可以从一个节点路由到匹配的节点。这些是与具体的网络插件实现相关的，如果感兴趣，可以深入查看某一个网络插件的具体实现。例如，AWS上，亚马逊提供了一个 [Container Network Interface(CNI) plugin (opens new window)](https://github.com/aws/amazon-vpc-cni-k8s)使得 Kubernetes 可以在 Amazon VPC 上执行节点到节点的网络通信。

<br/>

Container Network Interface(CNI) plugin 提供了一组通用 API 用来连接容器与外部网络。具体到容器化应用开发者来说，只需要了解在整个集群中，可以通过 Pod 的 IP 地址直接访问 Pod；网络插件是如何做到跨节点的数据包传递这件事情对容器化应用来说是透明的。AWS 的 CNI 插件通过利用 AWS 已有的 VPC、IAM、Security Group 等功能提供了一个满足 Kubernetes 网络模型要求的，且安全可管理的网络环境。

> 关于 CNI 网络插件下面会再做介绍

<br/>



## Service-to-Pod 的网络

我们已经了解了如何在 Pod 的 IP 地址之间传递数据包。然而，Pod 的 IP 地址并非是固定不变的，随着 Pod 的重新调度（例如水平伸缩、应用程序崩溃、节点重启等），Pod 的 IP 地址将会出现又消失。此时，Pod 的客户端无法得知该访问哪一个 IP 地址。Kubernetes 中，Service 的概念用于解决此问题。

一个 Kubernetes Service 管理了一组 Pod 的状态，可以追踪一组 Pod 的 IP 地址的动态变化过程。一个 Service 拥有一个 IP 地址，并且充当了一组 Pod 的 IP 地址的“虚拟 IP 地址”。任何发送到 Service 的 IP 地址的数据包将被负载均衡到该 Service 对应的 Pod 上。在此情况下，Service 关联的 Pod 可以随时间动态变化，客户端只需要知道 Service 的 IP 地址即可（该地址不会发生变化）。

从效果上来说，Kubernetes 自动为 Service 创建和维护了集群内部的分布式负载均衡，可以将发送到 Service IP 地址的数据包分发到 Service 对应的健康的 Pod 上。接下来我们讨论一下这是怎么做到的。

<br/>



### netfilter and iptables

Kubernetes 利用 Linux 内建的网络框架 - `netfilter` 来实现负载均衡。Netfilter 是由 Linux 提供的一个框架，可以通过自定义 handler 的方式来实现多种网络相关的操作。Netfilter 提供了许多用于数据包过滤、网络地址转换、端口转换的功能，通过这些功能，自定义的 handler 可以在网络上转发数据包、禁止数据包发送到敏感的地址等。

`iptables` 是一个 user-space 应用程序，可以提供基于决策表的规则系统，以使用 netfilter 操作或转换数据包。在 Kubernetes 中，kube-proxy 控制器监听 apiserver 中的变化，并配置 iptables 规则。当 Service 或 Pod 发生变化时（例如 Service 被分配了 IP 地址，或者新的 Pod 被关联到 Service），kube-proxy 控制器将更新 iptables 规则，以便将发送到 Service 的数据包正确地路由到其后端 Pod 上。iptables 规则将监听所有发向 Service 的虚拟 IP 的数据包，并将这些数据包转发到该Service 对应的一个随机的可用 Pod 的 IP 地址，同时 iptables 规则将修改数据包的目标 IP 地址（从 Service 的 IP 地址修改为选中的 Pod 的 IP 地址）。当 Pod 被创建或者被终止时，iptables 的规则也被对应的修改。换句话说，iptables 承担了从 Service IP 地址到实际 Pod IP 地址的负载均衡的工作。

在返回数据包的路径上，数据包从目标 Pod 发出，此时，iptables 规则又将数据包的 IP 头从 Pod 的 IP 地址替换为 Service 的 IP 地址。从请求的发起方来看，就好像始终只是在和 Service 的 IP 地址通信一样。

------

<br/>

### IPVS

Kubernetes v1.11 开始，提供了另一个选择用来实现集群内部的负载均衡：[IPVS](https://kuboard.cn/learning/k8s-intermediate/service/service-details.html#ipvs-代理模式)。 IPVS（IP Virtual Server）也是基于 netfilter 构建的，在 Linux 内核中实现了传输层的负载均衡。IPVS 被合并到 LVS（Linux Virtual Server）当中，充当一组服务器的负载均衡器。IPVS 可以转发 TCP / UDP 请求到实际的服务器上，使得一组实际的服务器看起来像是只通过一个单一 IP 地址访问的服务一样。IPVS 的这个特点天然适合与用在 Kubernetes Service 的这个场景下。

当声明一个 Kubernetes Service 时，你可以指定是使用 iptables 还是 IPVS 来提供集群内的负载均衡工鞥呢。IPVS 是转为负载均衡设计的，并且使用更加有效率的数据结构（hash tables），相较于 iptables，可以支持更大数量的网络规模。当创建使用 IPVS 形式的 Service 时，Kubernetes 执行了如下三个操作：

- 在节点上创建一个 dummy IPVS interface
- 将 Service 的 IP 地址绑定到该 dummy IPVS interface
- 为每一个 Service IP 地址创建 IPVS 服务器

将来，IPVS 有可能成为 kubernetes 中默认的集群内负载均衡方式。这个改变将只影响到集群内的负载均衡，本文后续讨论将以 iptables 为例子，所有讨论对 IPVS 是同样适用的。

-----



<br/>

### 数据包的传递 (Pod-to-Service)

<img src="https://img1.kiosk007.top/static/images/blog/20250216184009-pod-to-service.6718b584.gif" style="display: block; margin: 0 auto; width:70%;" />

在 Pod 和 Service 之间路由数据包时，数据包的发起和以前一样：

1. 数据包首先通过 Pod 的 `eth0` 网卡发出
2. 数据包经过虚拟网卡 `veth0` 到达网桥 `cbr0`
3. 网桥上的 APR 协议查找不到该 Service，所以数据包被发送到 root namespace 中的默认路由 - `eth0`
4. 此时，在数据包被 `eth0` 接受之前，数据包将通过 iptables 过滤。iptables 使用其规则（由 kube-proxy 根据 Service、Pod 的变化在节点上创建的 iptables 规则）重写数据包的目标地址（从 Service 的 IP 地址修改为某一个具体 Pod 的 IP 地址）
5. 数据包现在的目标地址是 Pod 4，而不是 Service 的虚拟 IP 地址。iptables 使用 Linux 内核的 `conntrack` 工具包来记录具体选择了哪一个 Pod，以便可以将未来的数据包路由到同一个 Pod。简而言之，iptables 直接在节点上完成了集群内负载均衡的功能。数据包后续如何发送到 Pod 上，其路由方式与 [Pod-to-Pod的网络](https://kuboard.cn/learning/k8s-intermediate/service/network.html#Pod-to-Pod的网络) 中的描述相同。

<br/>



### 数据包的传递 (Service-to-Pod)

<img src="https://img1.kiosk007.top/static/images/blog/20250216121136-service-to-pod.4393f600.gif" style="display: block; margin: 0 auto; width:70%;" />

1. 接收到此请求的 Pod 将会发送返回数据包，其中标记源 IP 为接收请求 Pod 自己的 IP，目标 IP 为最初发送对应请求的 Pod 的 IP
2. 当数据包进入节点后，数据包将经过 iptables 的过滤，此时记录在 `conntrack` 中的信息将被用来修改数据包的源地址（从接收请求的 Pod 的 IP 地址修改为 Service 的 IP 地址）
3. 然后，数据包将通过网桥、以及虚拟网卡 `veth0`
4. 最终到达 Pod 的网卡 `eth0`

### 使用 DNS

Kubernetes 也可以使用 DNS，以避免将 Service 的 cluster IP 地址硬编码到应用程序当中。Kubernetes DNS 是 Kubernetes 上运行的一个普通的 Service。每一个节点上的 `kubelet` 都使用该 DNS Service 来执行 DNS 名称的解析。集群中每一个 Service（包括 DNS Service 自己）都被分配了一个 DNS 名称。DNS 记录将 DNS 名称解析到 Service 的 ClusterIP 或者 Pod 的 IP 地址。[SRV 记录](https://kuboard.cn/learning/k8s-intermediate/service/dns.html#srv-记录) 用来指定 Service 的已命名端口。

DNS Pod 由三个不同的容器组成：

- `kubedns`：观察 Kubernetes master 上 Service 和 Endpoints 的变化，并维护内存中的 DNS 查找表
- `dnsmasq`：添加 DNS 缓存，以提高性能
- `sidecar`：提供一个健康检查端点，可以检查 `dnsmasq` 和 `kubedns` 的健康状态

DNS Pod 被暴露为 Kubernetes 中的一个 Service，该 Service 及其 ClusterIP 在每一个容器启动时都被传递到容器中（环境变量及 /etc/resolves），因此，每一个容器都可以正确的解析 DNS。DNS 条目最终由 `kubedns` 解析，`kubedns` 将 DNS 的所有信息都维护在内存中。`etcd` 中存储了集群的所有状态，`kubedns` 在必要的时候将 `etcd` 中的 key-value 信息转化为 DNS 条目信息，以重建内存中的 DNS 查找表。

CoreDNS 的工作方式与 `kubedns` 类似，但是通过插件化的架构构建，因而灵活性更强。自 Kubernetes v1.11 开始，CoreDNS 是 Kubernetes 中默认的 DNS 实现。

------



<br/>

## External-to-Service 的网络

前面我们已经了解了 Kubernetes 集群内部的网络路由。下面，我们来探讨一下如何将 Service 暴露到集群外部：

- 从集群内部访问互联网
- 从互联网访问集群内部

### 出方向 - 从集群内部访问互联网

将网络流量从集群内的一个节点路由到公共网络是与具体网络以及实际网络配置紧密相关的。为了更加具体地讨论此问题，本文将使用 AWS VPC 来讨论其中的具体问题。

在 AWS，Kubernetes 集群在 VPC 内运行，在此处，每一个节点都被分配了一个内网地址（private IP address）可以从 Kubernetes 集群内部访问。为了使访问外部网络，通常会在 VPC 中添加互联网网关（Internet Gateway），以实现如下两个目的：

- 作为 VPC 路由表中访问外网的目标地址
- 提供网络地址转换（NAT Network Address Translation），将节点的内网地址映射到一个外网地址，以使外网可以访问内网上的节点

在有互联网网关（Internet Gateway）的情况下，虚拟机可以任意访问互联网。但是，存在一个小问题：Pod 有自己的 IP 地址，且该 IP 地址与其所在节点的 IP 地址不一样，并且，互联网网关上的 NAT 地址映射只能够转换节点（虚拟机）的 IP 地址，因为网关不知道每个节点（虚拟机）上运行了哪些 Pod （互联网网关不知道 Pod 的存在）。接下来，我们了解一下 Kubernetes 是如何使用 iptables 解决此问题的。

<br/>

### 数据包的传递: Node-to-Internet

下图中：

1. 数据包从 Pod 的 network namespace 发出
2. 通过 `veth0` 到达虚拟机的 root network namespace
3. 由于网桥上找不到数据包目标地址对应的网段，数据包将被网桥转发到 root network namespace 的网卡 `eth0`。在数据包到达 `eth0` 之前，iptables 将过滤该数据包。
4. 在此处，数据包的源地址是一个 Pod，如果仍然使用此源地址，互联网网关将拒绝此数据包，因为其 NAT 只能识别与节点（虚拟机）相连的 IP 地址。因此，需要 iptables 执行源地址转换（source NAT），这样子，对互联网网关来说，该数据包就是从节点（虚拟机）发出的，而不是从 Pod 发出的
5. 数据包从节点（虚拟机）发送到互联网网关
6. 互联网网关再次执行源地址转换（source NAT），将数据包的源地址从节点（虚拟机）的内网地址修改为网关的外网地址，最终数据包被发送到互联网

在回路径上，数据包沿着相同的路径反向传递，源地址转换（source NAT）在对应的层级上被逆向执行。

<img src="https://img1.kiosk007.top/static/images/blog/20250216184355-pod-to-internet.986cf745.gif" style="display: block; margin: 0 auto; width:70%;" />









# 网络插件CNI

容器网络接口（Container Network Interface，CNI）是一套用于容器网络配置的规范和库，旨在为容器化应用提供简单、可插拔的网络解决方案。它主要应用于容器编排系统（如 Kubernetes）中，确保容器能够与其他容器、宿主机及外部网络进行通信。

## 其解决的问题

- **容器网络的动态创建与管理**

在容器化环境中，容器的创建和销毁是动态的过程。传统网络配置方式难以满足这种动态需求，而 CNI 允许在容器创建时自动配置网络接口，在容器销毁时自动清理相关网络资源，大大提高了容器网络管理的效率。

- **多容器间的通信**

容器通常需要与同一 Pod 内的其他容器以及不同 Pod 之间进行通信。CNI 提供了多种网络模型和插件，支持容器之间实现高效的通信，例如通过桥接网络、overlay 网络等方式，使容器能够像运行在同一物理网络中一样相互访问。

- **容器与外部网络的连接**

容器化应用往往需要与外部世界交互，如访问数据库、调用外部 API 等。CNI 解决了容器如何接入外部网络，如何获取外部可访问的 IP 地址等问题，确保容器能够与外部网络进行无缝通信。

----



## 组成部分

1. **基础插件**：负责创建和管理容器的网络接口，如 `bridge` 插件，它通过创建网桥将容器连接到宿主机网络；`loopback` 插件为容器创建回环接口，用于容器内部的网络通信。
2. **IPAM（IP Address Management）插件**：专门负责 IP 地址的分配和管理。例如，`host - local` 插件基于本地配置文件或 DHCP 服务器为容器分配 IP 地址，确保每个容器都能获得唯一可用的 IP 地址。
3. **高级插件**：提供更复杂的网络功能，如网络策略实施、服务发现等。例如，`flannel` 插件通过 overlay 网络为 Kubernetes 集群提供了一个扁平的二层网络，实现 Pod 之间的跨主机通信；`calico` 插件不仅支持网络策略的实施，还能提供高性能的网络转发能力。

----



## 配置网络容器接口的常用方法

不同的容器虚拟化网络解决方案中为 Pod 的网络名称空间创建虚拟接口设备的方式也会有所不同，目前主流的方式有三种：

1. **veth设备**：**这是最常见的一种方式**，上述的网络模型章节中提到的也是该方案，其是创建一个网桥，并为每个容器创建一对 虚拟以太网（veth），一个接入容器内部，另一个留置在宿主机的网络命名空间中添加为Linux内核桥接功能或者 OpenvSwtich (OVS) 的从设备
2. **多路复用**: 多路复用可以有一个中间网络设备组成，它暴露多个虚拟接口，使用数据包转发规则来控制每个数据包转发到对应的接口上。（通常使用 MACVLAN 或者 IPVLAN 技术）
   - MACVLAN 技术为每个虚拟接口配置一个 MAC 地址并基于此地址完成二层报文的转发，IPVLAN 则是分配一个IP地址，并共享单个MAC并根据目标IP完成容器的报文转发。（参考：[Docker跨主机通信之macvlan](http://dockeradv.baoshu.red/advanced_network/macvlan.html)）

3. **硬件交换**: 现今市面上有相当数量的 NIC 都支持单根 I/O 虚拟化（SR-IOV），它是创建虚拟设备的一种实现方式，每个虚拟设备自身表现为一个独立的PCI设备，并有着自己的 VLAN及硬件强制关联的QoS；SR-IOV 提供了接近硬件级别的性能。



<br/>

上面说到，Pod 之间的通信如果是同宿主机的话，其通信非常简单，只需要经过网桥绕行，就可以连接2个不同的 Pod 网络。但是如果要跨宿主机，怎么判断要访问的Pod 在哪个宿主上呢？



# 搭建一个简单的 K8S 集群

因为环境限制，这里我们使用 [minikube](https://minikube.sigs.k8s.io/docs/)  起一个三节点的 K8S 集群。

```bash
minikube start --nodes 3 --driver=kvm2 --image-mirror-country=cn 
```

> 建议下次启动加上 minikube start --registry-mirror=https://xxxxx.mirror.aliyuncs.com  ，不然即便minikube 启动起来了，但是里面的容器因为众所周知网络原因无法拉下来后续的镜像。

启动完成之后，可以使用如下的命令查看, 1个控制节点，2个work节点。

```bash
kubectl get nodes
NAME           STATUS   ROLES           AGE     VERSION
minikube       Ready    control-plane   3m56s   v1.32.0
minikube-m02   Ready    <none>          3m      v1.32.0
minikube-m03   Ready    <none>          116s    v1.32.0
```

启用 dashboard 。

```bash
kubectl proxy --address='0.0.0.0' --disable-filter=true &
minikube dashboard
```





minikube 默认使用的  kindnet (kubernetes in Docker) KinD 项目，这是一个利用 Docker 容器模拟 Kubernetes 节点，进而在本地快速搭建一个 kubernetes 的工具， 主要用于测试和开发的场景，kindnet 为 KinD 集群提供网络支持，确保 Pod 之间、Pod 与服务之间能够正常通信。



1. 准备一个测试的 deployment，3个副本集， 一个 service ，用于暴露这个服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-echo-service
spec:
  selector:
    app: demo-echo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: ClusterIP
--- 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-echo-app
  labels:
    app: demo-echo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-echo
  template:
    metadata:
      labels:
        app: demo-echo
    spec:
      containers:
      - name: demo-echo
        image: python:3.11-slim-buster
        ports:
        - containerPort: 8000
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        command: ["/bin/sh", "-c"]
        args:
          - |
            cat <<EOF > app.py
            import http.server
            import socketserver
            import os

            class MyHandler(http.server.SimpleHTTPRequestHandler):
                def do_GET(self):
                    self.send_response(200)
                    self.send_header('Content-type', 'text/plain')
                    self.end_headers()
                    pod_name = os.getenv('POD_NAME', 'Unknown')
                    pod_ip = os.getenv('POD_IP', 'Unknown')
                    node_ip = os.getenv('NODE_IP', 'Unknown')
                    message = f"POD Name: {pod_name}\nPOD IP: {pod_ip}\nNode IP: {node_ip}\n"
                    self.wfile.write(message.encode())

            PORT = 8000
            with socketserver.TCPServer(("", PORT), MyHandler) as httpd:
                print(f"Serving at port {PORT}")
                httpd.serve_forever()
            EOF
            python app.py
  
```



2. 验证网络的连通性

```bash
# 查找 pod 的 IP 地址
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
test-pod-m02   1/1     Running   0          3m10s
test-pod-m03   1/1     Running   0          3m10s

# 设置 POD 的 IP 地址
$ POD_M02_IP=$(kubectl get pod test-pod-m02 -o jsonpath='{.status.podIP}')   # 10.224.1.4
$ POD_M03_IP=$(kubectl get pod test-pod-m03 -o jsonpath='{.status.podIP}')   # 10.224.2.3

# 从 pod2 ping pod3
$ kubectl exec -it test-pod-m02 -- ping $POD_M03_IP

PING 10.244.2.3 (10.244.2.3): 56 data bytes
64 bytes from 10.244.2.3: seq=0 ttl=62 time=0.328 ms
64 bytes from 10.244.2.3: seq=1 ttl=62 time=0.402 ms

```



3. 验证其 跨 POD 访问方式，其通过为每个节点设置静态路由 (NextHop 只想内部节点的IP) 来建立可达性，每 10 s 检查一次这些路由是否有变化。

通过以下方式得到了验证

```bash
$ minikube node list 
minikube       192.168.49.2
minikube-m02   192.168.49.3
minikube-m03   192.168.49.4

$ minikube ssh -n minikube-m02
$ docker@minikube-m02:~$ ip route show
default via 192.168.49.1 dev eth0 proto static metric 100
10.244.0.0/24 via 192.168.49.2 dev eth0
10.244.1.2 dev veth0855796c scope host
10.244.1.3 dev veth46d0113f scope host
10.244.1.4 dev vethe6d0de64 scope host
10.244.2.0/24 via 192.168.49.4 dev eth0   # 目标为 10.244.2.0/24 都直接发给了
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
192.168.49.0/24 dev eth0 proto kernel scope link src 192.168.49.3
```



https://www.tkng.io/cni/kindnet/

<img src="https://img1.kiosk007.top/static/images/blog/20250316214947-kindnet.png">



# CNI 插件实验

## cilium (基于 eBPF 的网络)

Cilium 为基于 Kubernetes 的 Linux 容器管理平台上部署的服务，透明地提供服务间的网络和 API 连接及安全。

Cilium 底层是基于 Linux 内核的新技术 [eBPF](https://jimmysong.io/kubernetes-handbook/GLOSSARY.html#ebpf)，可以在 Linux 系统中动态注入强大的安全性、可见性和网络控制逻辑。 Cilium 基于 [eBPF](https://jimmysong.io/kubernetes-handbook/GLOSSARY.html#ebpf) 提供了多集群路由、替代 kube-proxy 实现负载均衡、透明加密以及网络和服务安全等诸多功能。除了提供传统的网络安全之外，[eBPF](https://jimmysong.io/kubernetes-handbook/GLOSSARY.html#ebpf) 的灵活性还支持应用协议和 DNS 请求/响应安全。同时，Cilium 与 Envoy 紧密集成，提供了基于 Go 的扩展框架。因为 [eBPF](https://jimmysong.io/kubernetes-handbook/GLOSSARY.html#ebpf) 运行在 Linux 内核中，所以应用所有 Cilium 功能，无需对应用程序代码或容器配置进行任何更改。

基于微服务的应用程序分为小型独立服务，这些服务使用 **HTTP**、**gRPC**、**Kafka** 等轻量级协议通过 API 相互通信。但是，现有的 Linux 网络安全机制（例如 iptables）仅在网络和传输层（即 IP 地址和端口）上运行，并且缺乏对微服务层的可见性。

Cilium 为 Linux 容器框架（如 [**Docker**](https://www.docker.com/) 和 [**Kubernetes）**](https://kubernetes.io/) 带来了 API 感知网络安全过滤。使用名为 **[eBPF](https://jimmysong.io/kubernetes-handbook/GLOSSARY.html#ebpf)** 的新 Linux 内核技术，Cilium 提供了一种基于容器 / 容器标识定义和实施网络层和应用层安全策略的简单而有效的方法。

| 标准容器网络                                                 | Clilum eBPF 容器网络                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| <img src="https://img1.kiosk007.top/static/images/blog/20250216132738-With-KubeProxy.webp" /> | <img src="https://img1.kiosk007.top/static/images/blog/20250216132813-Without-KubeProxy.webp" /> |
| Kube-Proxy 对 iptables 的依赖增加了显著的开销                |                                                              |





<br/>

-----



### 基本支持

一个简单的扁平第 3 层网络能够跨越多个集群连接所有应用程序容器。使用主机范围分配器可以简化 IP 分配。这意味着每个主机可以在主机之间没有任何协调的情况下分配 IP。

支持以下多节点网络模型：

- **Overlay**：基于封装的虚拟网络产生所有主机。目前 VXLAN 和 Geneve 已经完成，但可以启用 Linux 支持的所有封装格式。

  何时使用此模式：此模式具有最小的基础架构和集成要求。它几乎适用于任何网络基础架构，唯一的要求是主机之间可以通过 IP 连接。

- **本机路由**：使用 Linux 主机的常规路由表。网络必须能够路由应用程序容器的 IP 地址。

  何时使用此模式：此模式适用于高级用户，需要了解底层网络基础结构。此模式适用于：

  - 本地 IPv6 网络
  - 与云网络路由器配合使用
  - 如果您已经在运行路由守护进程



Cilium 是一个基于 **eBPF 技术** 的 Kubernetes CNI 插件，提供以下核心网络特性：

| 特性                          | 描述                                                         |
| :---------------------------- | :----------------------------------------------------------- |
| **服务发现与负载均衡**        | 支持 L3/L4 和 L7（如 HTTP、gRPC）的负载均衡，替代 kube-proxy |
| **网络策略（NetworkPolicy）** | 基于身份（Identity）的策略，支持 L3-L7 的细粒度控制（例如限制 HTTP 路径） |
| **多集群通信**                | 通过 Cluster Mesh 实现跨集群的服务发现和通信                 |
| **透明加密**                  | 使用 WireGuard 或 IPsec 对 Pod 间通信自动加密                |
| **可观测性**                  | 基于 Hubble 的流量监控，支持 Prometheus 和 Grafana 集成      |
| **带宽管理**                  | 基于 eBPF 的带宽控制和 QoS                                   |
| **IPv4/IPv6 双栈**            | 支持双协议栈网络                                             |



### 验证安装

1. **先清理现有的 CNI**

首先清理当前Minikube集群正在使用的CNI插件，可以通过删除对应的 POD 来做，不过更简单的方式是删除并重建整个 minikube 环境。

```bash
minikube delete
```

2. **启动 minikube 集群**，不过在启动的时候，不使用默认的CNI插件，可以使用以下命令启动一个新的 Minikube 集群。

```bash
minikube start --nodes 3 --driver=kvm2 --cni=false --image-mirror-country=cn 
```

3. **添加 cilium Helm 仓库**

使用 Helm 添加 Cilium 仓库，方便后续的安装

```bash
helm repo add cilium https://helm.cilium.io/

# 添加完成后，更新一下
helm repo update
```

4. **安装 cilium**

``` bash
helm install cilium cilium/cilium --version 1.17.2 \
   --namespace kube-system \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true \
   --set kubeProxyReplacement=true \
   --set hostServices.enabled=false \
   --set externalIPs.enabled=true \
   --set nodePort.enabled=true \
   --set hostPort.enabled=true \
   --set bpf.masquerade=false \
   --set image.pullPolicy=IfNotPresent \
   --set ipam.mode=kubernetes
```

5. **验证 Cilium 安装**

安装完成后可以验证是否安装成功

查看 Cilium Pod 状态

```bash
$ kubectl get pods -n kube-system -l k8s-app=cilium
```

或者使用 Cliium CLI 工具验证，安装 Cilium CLI 工具，使用以下命令验证

```bash
$ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium-envoy             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator          Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium                   Running: 3
                       cilium-envoy             Running: 3
                       cilium-operator          Running: 2
                       clustermesh-apiserver
                       hubble-relay
Cluster Pods:          1/1 managed by Cilium
Helm chart version:    1.17.2
Image versions         cilium             quay.io/cilium/cilium:v1.17.2@sha256:3c4c9932b5d8368619cb922a497ff2ebc8def5f41c18e410bcc84025fcd385b1: 3
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.31.5-1741765102-efed3defcc70ab5b263a0fc44c93d316b846a211@sha256:377c78c13d2731f3720f931721ee309159e782d882251709cb0fac3b42c03f4b: 3
                       cilium-operator    quay.io/cilium/operator-generic:v1.17.2@sha256:81f2d7198366e8dec2903a3a8361e4c68d47d19c68a0d42f0b7b6e3f0523f249: 2
```

从 POD 中查看更详细的内容，这里可以看到 `Routing` 是基于 Vxlan 的。

```bash
$ kubectl exec -it cilium-f7qwr -n kube-system -- cilium status
Defaulted container "cilium-agent" out of: cilium-agent, config (init), mount-cgroup (init), apply-sysctl-overwrites (init), mount-bpf-fs (init), clean-cilium-state (init), install-cni-binaries (init)
KVStore:                 Disabled
Kubernetes:              Ok         1.32 (v1.32.0) [linux/arm64]
Kubernetes APIs:         ["EndpointSliceOrEndpoint", "cilium/v2::CiliumClusterwideNetworkPolicy", "cilium/v2::CiliumEndpoint", "cilium/v2::CiliumNetworkPolicy", "cilium/v2::CiliumNode", "cilium/v2alpha1::CiliumCIDRGroup", "core/v1::Namespace", "core/v1::Pods", "core/v1::Service", "networking.k8s.io/v1::NetworkPolicy"]
KubeProxyReplacement:    True   [eth0   192.168.49.2 fe80::2c41:c5ff:fe9f:e465 (Direct Routing)]
Host firewall:           Disabled
SRv6:                    Disabled
CNI Chaining:            none
CNI Config file:         successfully wrote CNI configuration file to /host/etc/cni/net.d/05-cilium.conflist
Cilium:                  Ok   1.17.2 (v1.17.2-fb3ab54f)
NodeMonitor:             Listening for events on 2 CPUs with 64x4096 of shared memory
Cilium health daemon:    Ok
IPAM:                    IPv4: 3/254 allocated from 10.244.0.0/24,
IPv4 BIG TCP:            Disabled
IPv6 BIG TCP:            Disabled
BandwidthManager:        Disabled
Routing:                 Network: Tunnel [vxlan]   Host: Legacy
Attach Mode:             TCX
Device Mode:             veth
Masquerading:            IPTables [IPv4: Enabled, IPv6: Disabled]
Controller Status:       26/26 healthy
Proxy Status:            OK, ip 10.244.0.6, 0 redirects active on ports 10000-20000, Envoy: external
Global Identity Range:   min 256, max 65535
Hubble:                  Ok              Current/Max Flows: 4095/4095 (100.00%), Flows/s: 1.50   Metrics: Disabled
Encryption:              Disabled
Cluster health:          3/3 reachable   (2025-03-19T15:41:27Z)
Name                     IP              Node   Endpoints
Modules Health:          Stopped(0) Degraded(0) OK(54)
```



<br/>



### 基本特性(一) - 跨节点通信

```bash
$ kubectl get service
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
demo-echo-service   ClusterIP   10.110.229.85   <none>        80/TCP    3h26m
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP   2d8h
kiosk@ubuntuvm1:~$ minikube ssh      # 登录到 minikube 的宿主机上
docker@minikube:~$ curl 10.110.229.85  # 访问 demo 的 cluster IP ，可以正常访问，访问的结果是负载均衡的
POD Name: demo-echo-app-5d5558fcbd-mmpt4
POD IP: 10.244.0.83
Node IP: 192.168.49.2
```

登录到pod中查看

```bash
$ kubectl exec -it -n -kube-system cilium-559pk -- bash
$ hubble observe --from-namespace default --follow=true
```



<img src="https://img1.kiosk007.top/static/images/blog/20250323232540-cilium-hubble-observe.webp">





<br/>

### 基本特性(二) - Hubble 可视化追踪

Cilium 官方提供了一个流量测试的部署方案，可以直接使用官方部署的业务进行测试。

执行命令 `cilium connectivity test`，Cilium 会自动创建 `cilium-test` 的 Namespace，同时在 cilium-test 下部署测试业务。

```bash
# kubectl get all -n cilium-test
NAME                                  READY   STATUS    RESTARTS   AGE
pod/client-7df6cfbf7b-z5t2j           1/1     Running   0          21s
pod/client2-547996d7d8-nvgxg          1/1     Running   0          21s
pod/echo-other-node-d79544ccf-hl4gg   2/2     Running   0          21s
pod/echo-same-node-5d466d5444-ml7tc   2/2     Running   0          21s

NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/echo-other-node   NodePort   10.109.58.126   <none>        8080:32269/TCP   21s
service/echo-same-node    NodePort   10.108.70.32    <none>        8080:32490/TCP   21s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/client            1/1     1            1           21s
deployment.apps/client2           1/1     1            1           21s
deployment.apps/echo-other-node   1/1     1            1           21s
deployment.apps/echo-same-node    1/1     1            1           21s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/client-7df6cfbf7b           1         1         1       21s
replicaset.apps/client2-547996d7d8          1         1         1       21s
replicaset.apps/echo-other-node-d79544ccf   1         1         1       21s
replicaset.apps/echo-same-node-5d466d5444   1         1         1       21s
```

部署 Hubble Relay 后，Hubble 可以提供完整的集群范围的网络流量观测。

- **配置端口转发**

为了能正常访问 Hubble API，需要创建端口转发，将本地请求转发到 Hubble Service。可以执行 `kubectl port-forward deployment/hubble-relay -n kube-system 4245:4245` 命令，在当前终端开启端口转发。

端口转发配置可以参考 [端口转发](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)。

`kubectl port-forward` 命令不会返回，需要打开另一个终端来继续测试。

配置完端口转发之后，在终端执行 `hubble status` 命令，如果有类似如下输出，则端口转发配置正确，可以使用命令行进行流量观测。

```
# hubble status
Healthcheck (via localhost:4245): Ok
Current/Max Flows: 8,190/8,190 (100.00%)
Flows/s: 22.86
Connected Nodes: 2/2
```

执行命令 `cilium hubble ui` 可以自动创建端口转发，将 `hubble-ui service` 映射到本地端口。 正常情况下，执行完命令后，会自动打开本地的浏览器，跳转到 Hubble UI 界面。如果没有自动跳转，在浏览器中输入 `http://localhost:12000` 打开 UI 观察界面。

在界面左上角，选择 `cilium-test` namespace，查看 Cilium 提供的测试流量信息。

<img src="https://img1.kiosk007.top/static/images/blog/20250323233419-cilium-test-ui.webp">



### 基本特性(三) - 网络策略

https://kube-ovn.readthedocs.io/zh-cn/latest/advance/cilium-networkpolicy/







- 参考：[最Cool Kubernetes网络方案Cilium入门](https://cilium.io/blog/2020/05/04/guest-blog-kubernetes-cilium/)

- 参考: [Cilium 网络流量观测](https://kube-ovn.readthedocs.io/zh-cn/latest/advance/cilium-hubble-observe/)



## flannel





## calico









参考文章：

- [kubernetes 网络模型](https://kuboard.cn/learning/k8s-intermediate/service/network.html)
- [基于 eBPF 的网络 Cilium](https://jimmysong.io/kubernetes-handbook/concepts/cilium.html)