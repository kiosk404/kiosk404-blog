---
title: minikube 实践
author: kiosk
tags: 
  - k8s
categories:
  - k8s
date: 2020-09-22 09:53:00
lastmod: 2024-06-18 19:16:20
---



最近又在折腾 k8s 了，minikube 无疑是单机环境下最好用的 k8s 解决方案了。网上的资源太老旧了，让安装的版本是 1.2.0 - 1.13.0 之间，而且给提供的 minikube 可执行文件在 aliyun 的 oss 上还没有其他的版本，所以决定自己手动安装并且记录一下。

<br/>

>  **Minikube**可以实现一种轻量级的Kubernetes集群，通过在本地计算机上创建虚拟机并部署只包含单个节点的简单集群

<br/>

本文于 24.6 月重新更新过，最新的 minikube 版本是 1.32.0 , kubernetes 最新版本 1.33.0 

<br/>

如果需要更新 minikube ，需要更新 minikube 安装包

- 如需更新minikube，需要更新 minikube 安装包
  - `minikube delete` 删除现有虚机，删除 `~/.minikube` 目录缓存的文件
  - 重新创建 minikube 环境



<br/>

# 配置

## 先决条件

Minikube在不同操作系统上支持不同的驱动

- macOS
  - [Docker](https://minikube.sigs.k8s.io/docs/drivers/docker/) 缺省驱动
  - [xhyve driver](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md#xhyve-driver) , [VirtualBox](https://www.virtualbox.org/wiki/Downloads) 或 [VMware Fusion](https://www.vmware.com/products/fusion)
- Linux
  - [VirtualBox](https://www.virtualbox.org/wiki/Downloads) 或 [KVM2](https://minikube.sigs.k8s.io/docs/drivers/kvm2/)
  - [Docker](https://minikube.sigs.k8s.io/docs/drivers/docker/) 缺省驱动
- Windows
  - [VirtualBox](https://www.virtualbox.org/wiki/Downloads) 或 [Hyper-V](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md#hyperV-driver) 

<br/>

注：

- 由于minikube复用了docker-machine，在其软件包中已经支持了相应的VirtualBox, VMware Fusion驱动
- 如果是 Linux 环境下，可以使用 Docker 当底层驱动

<br/>

## 安装 minikube

这里建议直接从 [github release](https://github.com/kubernetes/minikube/releases) 页面下载，什么版本的都可以找到。避免网上全是上古版本的minikube。

这里我下载的是 1.33.0 , 最新版本的了

https://github.com/kubernetes/minikube/releases/download/v1.33.0/minikube_1.33.0-0_amd64.deb

```bash
sudo apt dpkg -u minikube_1.33.0-0_amd64.deb
```



<br/>

## 安装 kubectl 

kubectl 建议也是安装最新的（要和 kubenetes apiserver 的版本不要差太多）

这里我会下载最新的 稳定版本，这里安装的实际是 1.30

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo chmod +x kubectl
sudo cp kubectl /usr/local/bin
```



<br/>

# 启动

> 为了方便大家开发和体验Kubernetes，社区提供了可以在本地部署的开发环境 [Minikube](https://github.com/kubernetes/minikube)。由于网络访问原因，很多朋友无法直接使用minikube进行实验。在v1.24.0的官方 Minikube 中，已经合并了由阿里云团队支持的方案，可以帮助大家利用阿里云的服务来获取所需Docker镜像，
>
> 但不幸的是 --image-mirror-country='cn' 参数已经失效，亲自验证过

好吧，本质上 minikube 在启动的时候需要安装一个  kicbase 的镜像。这个镜像非常大，预估1.2G

<br/>

> kicbase 通常指的是 Kubernetes Image Collection (KIC) 的基础镜像。KIC 项目是一个由社区驱动的 Kubernetes 镜像集合，旨在提供一种简化的方式来使用和管理 Kubernetes 集群中所需的各种镜像。



<br/>

这里我们采取先拉下来镜像，直接从镜像里启动的方式。（默认是启动时 minikube 从 docker.io 去拉取镜像，但是由于不可描述的原因，这个是行不通的）

这里推荐 https://docker.aityp.com/ 这个上面去搜

## 预拉取镜像

```bash
# 拉取 kicbase 镜像
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/kicbase/stable:v0.0.44
docker tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/kicbase/stable:v0.0.44  docker.io/kicbase/stable:v0.0.44

# 启动 minikube
minikube start --base-image=kicbase/stable:v0.0.44 --registry-mirror=https://bmtb46e4.mirror.aliyuncs.com
```



```bash
# 启动期间又卡住了, storage-provisioner:v5 镜像是从 gcr.io 拉取的，所以因为不可描述的原因，这个镜像是获取失败的。
minikube start --base-image=kicbase/stable:v0.0.44 --registry-mirror=https://bmtb46e4.mirror.aliyuncs.com
😄  Ubuntu 22.04 上的 minikube v1.33.1
✨  自动选择 docker 驱动。其他选项：kvm2, qemu2, none, ssh
📌  使用具有 root 权限的 Docker 驱动程序
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.44 ...
🔥  Creating docker container (CPUs=2, Memory=3900MB) ...
❗  The image 'gcr.io/k8s-minikube/storage-provisioner:v5' was not found; unable to add it to cache.
❗  The image 'registry.k8s.io/etcd:3.5.12-0' was not found; unable to add it to cache.
❗  The image 'registry.k8s.io/kube-scheduler:v1.30.0' was not found; unable to add it to cache.
❗  The image 'registry.k8s.io/kube-controller-manager:v1.30.0' was not found; unable to add it to cache.
❗  The image 'registry.k8s.io/kube-apiserver:v1.30.0' was not found; unable to add it to cache.

```



只能按照之前的办法继续了 先下好镜像，再加载的方式了

```bash
# 提前加载 gcr.io/k8s-minikube/storage-provisioner:v5
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/gcr.io/k8s-minikube/storage-provisioner:v5
docker tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/gcr.io/k8s-minikube/storage-provisioner:v5  gcr.io/k8s-minikube/storage-provisioner:v5
minikube image load gcr.io/k8s-minikube/storage-provisioner:v5

# 提前加载 registry.k8s.io/etcd:3.5.12-0
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/etcd:3.5.12-0
docker tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/etcd:3.5.12-0  registry.k8s.io/etcd:3.5.12-0
minikube image load registry.k8s.io/etcd:3.5.12-0

# 提前加载 registry.k8s.io/kube-scheduler:v1.30.2
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/kube-scheduler:v1.30.2
docker tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/kube-scheduler:v1.30.2  registry.k8s.io/kube-scheduler:v1.30.2
minikube image load registry.k8s.io/kube-scheduler:v1.30.2

# 提前加载 registry.k8s.io/kube-controller-manager:v1.30.2
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/kube-controller-manager:v1.30.2
docker tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/kube-controller-manager:v1.30.2  registry.k8s.io/kube-controller-manager:v1.30.2
minikube image load registry.k8s.io/kube-controller-manager:v1.30.2

# 提前加载 registry.k8s.io/kube-apiserver:v1.30.2
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/kube-apiserver:v1.30.2
docker tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/kube-apiserver:v1.30.2  registry.k8s.io/kube-apiserver:v1.30.2
minikube image load registry.k8s.io/kube-apiserver:v1.30.2
```



## minkube 启动 !!

```bash
✗ minikube start --base-image=kicbase/stable:v0.0.44  
😄  Ubuntu 22.04 上的 minikube v1.33.1
✨  根据现有的配置文件使用 docker 驱动程序
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.44 ...
💾  正在下载 Kubernetes v1.30.0 的预加载文件...
    > preloaded-images-k8s-v18-v1...:  342.90 MiB / 342.90 MiB  100.00% 30.36 K
🏃  正在更新运行中的 docker "minikube" container ...
🐳  正在 Docker 26.1.1 中准备 Kubernetes v1.30.0…
    ▪ 正在生成证书和密钥...
    ▪ 正在启动控制平面...
    ▪ 配置 RBAC 规则 ...
🔗  配置 bridge CNI (Container Networking Interface) ...
🔎  正在验证 Kubernetes 组件...
    ▪ 正在使用镜像 gcr.io/k8s-minikube/storage-provisioner:v5
🌟  启用插件： storage-provisioner, default-storageclass
🏄  完成！kubectl 现在已配置，默认使用"minikube"集群和"default"命名空间
```



<br/>

# 使用 minikube

## minikube 插件

用户使用Minikube CLI管理虚拟机上的Kubernetes环境，比如：启动，停止，删除，获取状态等。一旦Minikube虚拟机启动，用户就可以使用熟悉的Kubectl CLI在Kubernetes集群上执行操作。

Minikube 也提供了丰富的 Addon 组件

```bash
minikube addons list  
|-----------------------------|----------|--------------|--------------------------------|
|         ADDON NAME          | PROFILE  |    STATUS    |           MAINTAINER           |
|-----------------------------|----------|--------------|--------------------------------|
| ambassador                  | minikube | disabled     | 3rd party (Ambassador)         |
| auto-pause                  | minikube | disabled     | minikube                       |
| cloud-spanner               | minikube | disabled     | Google                         |
| csi-hostpath-driver         | minikube | disabled     | Kubernetes                     |
| dashboard                   | minikube | disabled     | Kubernetes                     |
| default-storageclass        | minikube | enabled ✅   | Kubernetes                     |
| efk                         | minikube | disabled     | 3rd party (Elastic)            |
| freshpod                    | minikube | disabled     | Google                         |
| gcp-auth                    | minikube | disabled     | Google                         |
| gvisor                      | minikube | disabled     | minikube                       |
| headlamp                    | minikube | disabled     | 3rd party (kinvolk.io)         |
| helm-tiller                 | minikube | disabled     | 3rd party (Helm)               |
| inaccel                     | minikube | disabled     | 3rd party (InAccel             |
|                             |          |              | [info@inaccel.com])            |
| ingress                     | minikube | disabled     | Kubernetes                     |
...
```

