# Kubernetes 架构及组件设计

Kubernetes 架构中的组件主要有 kubectl、kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy和container 等
不同组件之间松耦合架构，各组件之间各司其职，保证整个集群的稳定运行。
<br/>

<img src="https://img1.kiosk007.top/static/images/docsify/cloudnative/k8s_arch1.png" alt="kubernetes arch" style="zoom:43%;clear:both;display:block;margin:auto">

- **Api Server** 作为整个Kubernetes集群操作etcd的唯一入口，负责Kubernetes各资源的认证&鉴权，校验以及CRUD等操作，提供RESTful APIs，供其它组件调用.

  > 比如通过 kubectl 创建了一个 Pod 资源对象，请求通过kube-apiserver 的 HTTP 接口将 Pod 资源对象存储至 ETCD集群中

- **Controller Manager** 作为集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的管理，当某个Node意外宕机时，Controller Manager会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。 

  > 其负责确保 Kubernetes 系统的实际状态收敛到所需状态，其默认提供了一些控制器（Controller），例如 DeploymentControllers 控制器、StatefulSet 控制器、Namespace 控制器及 PersistentVolume 控制器，每个控制器通过 kube-apiserver 组件提供的接口实时监控整个集群的当前状态，当系统发生故障时，会尝试系统状态尝试修复到 “期望状态”。

- **Scheduler** 作为整个 Kubernetes 的关键模块，为Pod提供调度服务，例如基于资源的公平调度、调度Pod到指定节点、或者通信频繁的Pod调度到同一节点等。

  > 调度算法分两种，预选调度算法和优选调度算法，除调度策略外，Kubernetes 还支持优先级调度、抢占机制以及亲和性调度等功能。

- **kubelet** 用于管理节点，kubelet 组件用来接收、处理、上报 kube-apiserver 组件下发的任务。kubelet 进程启动时会向kube-apiserver注册节点自身信息，主要负责所在节点上Pod资源对象的管理。例如 Pod 资源对象的创建、修改、监控、删除、驱逐以及 Pod 生命周期管理

  > kubelet 开放接口
  >
  > - Container Runtime interface：简称 CRI（容器运行时接口），提供容器运行时通用插件接口服务，CRI 定义了容器和镜像服务的接口，CRI 将kubelete 组件与容器运行时进行解耦
  > - Contianer Network Interface：简称 CNI（容器网络接口），提供网络通用插件接口服务，CNI 定义了 Kubernetes 网络插件的基础，容器创建时通过 CNI 插件配置网络。
  > - Contianer Storage Interface：简称 CSI（容器存储接口）：提供存储通用插件接口服务，CSI 定义了容器存储卷标准规范，容器创建时通过 CSI 插件配置存储卷。

- **kube-proxy**：作为节点上的网络代理，运行在每个 Kubernetes 节点之上，监控 kube-apiserver 的服务和端点资源的变化，通过 iptables / ipvs 等配置负载均衡器，为一组 Pod 提供统一的 TCP/UDP 流量转发和负载均衡功能。



# 开发组件
## cobra

kubectl 的命令行交互实现，对 Kubernetes API Server 进行操作，通信协议使用 HTTP/JSON ，kubectl 发送相应的 HTTP 请求。

> https://github.com/spf13/cobra

## client-go

kubernetes 提供了通过编程的方式与 kubernetes API Server 交互进行通信。client-go 是从 kubernetes 是从 kubernetes 代码中单独抽离出来的包，并作为官网的 k8s sdk 。

在大部分的基于 kubernetes 的二次开发中，都是通过 client-go进行的交互操作。

> https://github.com/kubernetes/client-go
>
> [K8s二开之 client-go 初探](https://juejin.cn/post/6962869412785487909)

``` go
fmt.Println("hello world")
```