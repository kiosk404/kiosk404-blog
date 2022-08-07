---


title: 分布式存储系统 ceph
author: kiosk
date: 2022-06-18 13:52:18
draft: true
tags:
  - k8s
categories:
  - devops
---







- [部署k8s]](https://kiosk007.top/post/ubuntu20-04-%E9%83%A8%E7%BD%B2-kubernetes-k8s/)



<!--more-->

kubernetes StatefulSet 允许我们为Pod分配一个稳定的标识和持久化存储。Elasticsearch 需要的是稳定的存储来保证 Pod 在重新调度或者重启后的数据依然不变，所以需要 StatefulSet 来管理 Pod。



> 环境：
>
> Kubernetes 版本：minikube v1.25.2 
>
> docker 版本：20.10.12
>
> 服务器：腾讯云轻量应用服务器 一台
>
> 存储：Ceph + Rock



# 分布式存储系统

这里为了为 ES 提供一个数据持久化存储，我们这里选择的 [Ceph](https://ceph.io/en/) 和 [Rock](https://rook.io/)，下面的内容会先介绍一下。



## Ceph 

Ceph是一种高度可扩展的**分布式存储解决方案**，系统设计旨在性能、可靠性和可扩展性上能够提供优秀的存储服务。分布式对象存储是存储的未来，因为它们适应非结构化数据，并且客户端可以同时使用当前及传统的对象接口进行数据存取。

<br/>

**体系架构**

{{< image src="https://img1.kiosk007.top/static/images/blog/ceph.png" caption=" (`ceph`)" src_s="https://img1.kiosk007.top/static/images/blog/ceph.png" src_l="https://img1.kiosk007.top/static/images/blog/ceph.png" >}}



- RADOS （RADO）：RADOS 本身是分布式存储系统，CEPH所有的存储功能都是基于 RADOS 实现。底层的存储服务由多个主机（host）组成，其由 OSD 和 Monitor 组成。
- LIBRADOS：是RADOS存储集群的API，支持多种操作系统语言接口。
- RBD（块存储）：提供一个标准的块设备接口，常用语在虚拟化的场景下为虚拟机创建 volume。RBD与 RADOS GW 对接，可以暴露Rest风格的接口。
- CEPHFS：CEPHFS 提供POSIX 接口，用户直接通过客户端挂载使用，它是内核态程序，它通过内核中的net模块与 Rados 交互。



> 更多关于 Ceph 的文章
>
> - ceph 官方文档：https://docs.ceph.com/en/quincy/
> - [ceph分布式存储简介](https://zhuanlan.zhihu.com/p/161358190)



ceph 是一种高度扩展的分布式存储方案，但是ceph的集群搭建相对还是比较复杂的，既然都上k8s 了，那么肯定需要用 k8s 来管理 ceph集群。

## Rook

**Rook 是一个自管理的分布式存储编排系统**，**可以为 Kubernetes提供便利的存储解决方案**。可以提供 Ceph 集群管理能力的 Operator。Rock 使用 CRD 一个控制器对 Ceph 之类的资源进行部署和管理。Rock 本身不提供存储功能，但是在 k8s 和存储系统之间提供了适配层，简化了存储系统的部署与维护。



Rook 由 Operator 和 Cluster 两部分组成：

- Operator：由一些 CRD 和 一个 All in one 镜像构成，包含启动和监控存储系统的所有功能。
- Cluster：负责创建CRD对象。制定相关参数，包括ceph镜像、元数据持久化位置、磁盘位置等。



# 部署 Rook 和 Ceph

- 下载 Rook

```
git clone https://github.com/rook/rook.git
```

- 启动 rook operator（operator 作为 kubernetes 的插件存在，提供API扩展Rock的一些概念）

```
$ git clone --single-branch --branch v1.9.2 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

在国内访问一些镜像时，正常情况下访问不通，将镜像改为国内阿里云的镜像，这里用了这篇[文章](https://blog.csdn.net/qq_40592377/article/details/110292089) 里的镜像。

```
#在91行下面，修改下面几个配置，改为阿里云的地址，tag不变（默认是注释掉的，先解开注释）

ROOK_CSI_CEPH_IMAGE: "registry.aliyuncs.com/it00021hot/cephcsi:v3.6.1"
  ROOK_CSI_REGISTRAR_IMAGE: "registry.aliyuncs.com/it00021hot/csi-node-driver-registrar:v2.5.0"
  ROOK_CSI_RESIZER_IMAGE: "registry.aliyuncs.com/it00021hot/csi-resizer:v1.4.0"
  ROOK_CSI_PROVISIONER_IMAGE: "registry.aliyuncs.com/it00021hot/csi-provisioner:v3.1.0"
  ROOK_CSI_SNAPSHOTTER_IMAGE: "registry.aliyuncs.com/it00021hot/csi-snapshotter:v5.0.1"
  ROOK_CSI_ATTACHER_IMAGE: "registry.aliyuncs.com/it00021hot/csi-attacher:v3.4.0"
  ROOK_CSI_NFS_IMAGE: "registry.aliyuncs.com/it00021hot/nfsplugin:v3.1.0"

```

- 确保 rock operator 处于 Running 状态后，接下来启动rook cluster，由于使用了 Minikube，所以选择 rook/deploy/examples/cluster-test.yaml 文件进行集群创建。

1. 将 cluster-test.yaml 中的 devieFilter 一项注释去掉，虚拟机的硬盘是 vda, 所以需要在 deviceFilter 处填上 va[\^a]，以忽略创建在vda上的osd
2. 默认的ceph镜像地址 quay.io 在国内访问不了，可替换为 quay.mirrors.ustc.edu.cn/ceph/ceph:v17

```
kubectl create -f cluster-test.yaml
```



- 部署toolbox

toolbox 是一个rook的工具集容器，该容器中的命令可以用来调试、测试rook，对Ceph临时测试

```
kubectl apply -f toolbox.yaml
```



- 测试 rook-ceph

可以添加个别名，就不需要写太多的命令

```
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -- ceph -s
  cluster:
    id:     d2259cda-d11f-4d87-a8c8-08ac16438067
    health: HEALTH_WARN
            Reduced data availability: 1 pg inactive
            OSD count 0 < osd_pool_default_size 1
 
  services:
    mon: 1 daemons, quorum a (age 13h)
    mgr: a(active, since 13h)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     100.000% pgs unknown
             1 unknown
 

```



- 部署 ceph dashboard

```
kubectl apply -f dashboard-external-http.yaml
```

查看service和minikube的ip，利用ip+NodePort访问

```bash
$ kubectl -n rook-ceph get service
NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
csi-cephfsplugin-metrics                ClusterIP   10.108.9.98      <none>        8080/TCP,8081/TCP   14h
csi-rbdplugin-metrics                   ClusterIP   10.101.202.131   <none>        8080/TCP,8081/TCP   14h
rook-ceph-mgr                           ClusterIP   10.107.69.30     <none>        9283/TCP            14h
rook-ceph-mgr-dashboard                 ClusterIP   10.104.245.208   <none>        7000/TCP            14h
rook-ceph-mgr-dashboard-external-http   NodePort    10.111.128.73    <none>        7000:30588/TCP      8m46s
rook-ceph-mon-a                         ClusterIP   10.106.21.92     <none>        6789/TCP,3300/TCP   14h

$ minikube ip
192.168.49.2

// 使用 192.168.49.2:30588 链接访问
```







> 部署参考：
>
> - [rock quick start](https://rook.io/docs/rook/v1.9/quickstart.html)



# 创建 Ceph 块存储

- 创建StorageClass

在提供（Provisioning）块存储之前，需要先创建 StorageClass 和存储池。K8S 需要创建这两类资源，才能和 Rook 交互，进而分配持久卷（PV）。

```
$ cd rook/deploy/examples/csi/rbd
$ kubectl apply -f storageclass.yaml 
cephblockpool.ceph.rook.io/replicapool created
storageclass.storage.k8s.io/rook-ceph-block created
$ kubectl get sc
NAME                 PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-ceph-block      rook-ceph.rbd.csi.ceph.com   Delete          Immediate           true                   9s
standard (default)   k8s.io/minikube-hostpath     Delete          Immediate           false                  22h

```









<!------->

<br/>

<br/>

参考：

- [《大话 Ceph》之RDB 那点事儿](https://cloud.tencent.com/developer/article/1006283#:~:text=RBD%E6%98%AF%E4%BB%80%E4%B9%88%20RBD%20%3A%20Ceph%E2%80%99s%20RADOS%20Block%20Devices%2C%20Ceph,striped%20over%20multiple%20OSDs%20in%20a%20Ceph%20cluster.)