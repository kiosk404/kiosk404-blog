---
title: 'k8s 部署 HTTP3  Nginx '
author: kiosk
date: 2021-08-04 22:56:00
categories:
  - k8s
  - Nginx
tags:
  - k8s
  - Nginx
---
在之前的文章中介绍了如何编译[支持HTTP3的Nginx](https://kiosk007.top/2021/07/31/%E5%B0%9D%E8%AF%95-Nginx-%E4%B8%8A%E4%BD%BF%E7%94%A8HTTP3/) 为了一键将H3 Nginx上云，一劳永逸，本文会记录一下容器化Nginx的过程。

<!--more-->

# 制作镜像

Docker Image 的制作两种方法，支持 HTTP3 的 Nginx 在上面的文章中已经阐述(不过需要加上 `nginx-upstream-dynamic-servers` 这个模块，后面会用到)，镜像的制作是将编译好的二进制文件打包放进 Docker 镜像当中，并保证其可运行。

```
方法 1：docker commit #保存 container 的当前状态到 image 后，然后生成对应的 image
方法 2：docker build #使用 Dockerfile 文件自动化制作 image
```

## 方法一: docker commit

获取 ubuntu 镜像

```
$ docker run -it ubuntu:latest /bin/bash
# 进入到容器后，创建 nginx 家目录
mkdir nginx
```

Nginx 的编译产物会全部放进 ${NGINX_QUIC_PATH}/objs 目录下。所以创建镜像需要将这里面的必要文件copy进镜像中。

``` bash
➜  nginx-quic ls objs 
addon  autoconf.err  Makefile  nginx  nginx.8  ngx_auto_config.h  ngx_auto_headers.h  ngx_modules.c  ngx_modules.o  src

➜  docker cp ./obj/nginx 9d129eb7c4a6:/nginx/

```

因为Nginx依赖 Lua 动态库所以还需要将 luajit 等动态库拷贝进镜像中。并指明相关的动态库加载地址。


``` bash
➜  docker cp /usr/local/include/luajit-2.0/ 9d129eb7c4a6:/usr/local/include
➜  docker cp /usr/local/lib/libluajit-5.1.a 9d129eb7c4a6:/usr/local/lib
➜  docker cp /usr/local/lib/libluajit-5.1.so.2.0.5 9d129eb7c4a6:/usr/local/lib

➜  docker cp /usr/local/lib/libpcre.so.1.2.12 9d129eb7c4a6:/usr/local/lib/
➜  docker cp /usr/lib/x86_64-linux-gnu/libunwind.so.8.0.1  9d129eb7c4a6:/usr/local/lib/
➜  docker cp /usr/lib/x86_64-linux-gnu/libprofiler.so.0.4.18 9d129eb7c4a6:/usr/local/lib/

# 进入到容器中，创建软连
cd /usr/local/lib
ln -sv libluajit-5.1.so.2.0.5 libluajit-5.1.so
ln -sv libluajit-5.1.so.2.0.5 libluajit-5.1.so.2

ln -sv libpcre.so.1.2.12 libpcre.so
ln -sv libpcre.so.1.2.12 libpcre.so.1

ln -sv libprofiler.so.0.4.18 libprofiler.so
ln -sv libprofiler.so.0.4.18 libprofiler.so.0

ln -sv libunwind.so.8.0.1 libunwind.so
ln -sv libunwind.so.8.0.1 libunwind.so.8


# 在容器内的 ~/.bashrc 生成环境变量
vim ~/.bashrc

export LUAJIT_LIB=/usr/local/lib
export LUAJIT_INC=/usr/local/include/luajit-2.0


# 执行 exit 退出容器
exit
```

根据容器当前状态做一个 image 镜像
> -m :提交时的说明文字； -a :提交的镜像作者；

``` bash
➜  docker commit -m "nginx http3 quic" -a "kiosk007" 158c3352b0da weijiaxiang007/nginx-http3:v0.1
sha256:322e9c9b5fa886294ca478440ccfe972fcef2416f1d99b3eb9a20552454246fa

➜  docker images
REPOSITORY                                                    TAG       IMAGE ID       CREATED              SIZE
http3_nginx                                                   v0.1      47f04055cd4e   About a minute ago   201MB

```



## 方法二：DockerFile

使用 docker build 创建镜像时，需要使用 Dockerfile 文件自动化制作 image 镜像
Dockerfile 有点像源码编译时./configure 后产生的 Makefile

- 创建工作目录 和 DockerFile 文件

``` bash
➜  mkdir nginx-http3
➜  cd nginx-http3
➜  touch Dockerfile
```

将 Nginx-http3 的编译产物和依赖动态库 copy 至该目录中

``` bash
➜  cp -rf /usr/local/include/luajit-2.0 .
➜  cp -rf /usr/local/lib/libluajit-5.1.so.2.0.5 .
➜  cp -rf /usr/local/lib/libluajit-5.1.a .       
➜  cp -rf /usr/local/lib/libpcre.so.1.2.12 .
➜  cp -rf /usr/lib/x86_64-linux-gnu/libunwind.so.8.0.1 .
➜  cp -rf /usr/lib/x86_64-linux-gnu/libprofiler.so.0.4.18 .
➜  cp -rf ~/Project/Nginx/nginx-quic/objs/nginx .
➜  cp -rf ~/Project/Nginx/nginx-quic/conf/nginx.conf .
➜  cp -rf ~/Project/Nginx/nginx-quic/conf/mime.types . 

```


- 编写DockerFile

``` bash
# This dockerfile uses the ubuntu image
# VERSION 2 - EDITION 1
# Author: kiosk007

# FROM 基于哪个镜像
FROM ubuntu:latest

# MAINTAINER 镜像创建者
MAINTAINER kiosk007

# 设置环境变量
ENV LUAJIT_LIB=/usr/local/lib LUAJIT_INC=/usr/local/include/luajit-2.0

# COPY 文件至容器中
COPY luajit-2.0 /usr/local/include/luajit-2.0
COPY lib* /usr/local/lib/
COPY nginx /sbin


# RUN 
RUN ln -sv /usr/local/lib/libluajit-5.1.so.2.0.5 /usr/local/lib/libluajit-5.1.so && \
ln -sv /usr/local/lib/libluajit-5.1.so.2.0.5 /usr/local/lib/libluajit-5.1.so.2 && \
ln -sv /usr/local/lib/libpcre.so.1.2.12 /usr/local/lib/libpcre.so && \
ln -sv /usr/local/lib/libpcre.so.1.2.12 /usr/local/lib/libpcre.so.1 && \
ln -sv /usr/local/lib/libprofiler.so.0.4.18 /usr/local/lib/libprofiler.so && \
ln -sv /usr/local/lib/libprofiler.so.0.4.18 /usr/local/lib/libprofiler.so.0 && \
ln -sv /usr/local/lib/libunwind.so.8.0.1 /usr/local/lib/libunwind.so && \
ln -sv /usr/local/lib/libunwind.so.8.0.1 /usr/local/lib/libunwind.so.8 && mkdir -p /nginx/logs && mkdir -p /nginx/conf && ldconfig


# EXPOSE 暴露端口
EXPOSE 80/tcp 443/tcp 443/udp

# 设置挂载路径 /nginx
VOLUME /nginx

# CMD
CMD ["/sbin/nginx", "-p", "/nginx", "-g", "daemon off;"]

```

<p class='div-border grey'>Dockerfile 的指令每执行一次都会在 docker 上新建一层，所以特别是 RUN 指令尽量通过 && 连在一起</p>

关于Docker CMD 和 ENTRYPOINT ，可参考[这篇文章](https://www.jb51.net/article/136264.htm), Dockerfile 语法参考 [菜鸟](https://www.runoob.com/docker/docker-dockerfile.html)，
关于为什么 CMD 要写成 `daemon off` 参考 [Docker，为什么你容器刚启动就停了？](https://z.itpub.net/article/detail/45E0117B462C4053A0FA82AC0FBED1FB)

- 构建镜像文件

```
➜  docker build -t kiosk007/nginx-quic:v0.2 .
```

--tag, -t: 镜像的名字及标签，通常 name:tag 或者 name 格式；可以在一次构建中为一个镜像设置多个标签.

镜像打包好后，通过 `docker images ` 命令可以看到当前的镜像id

- 运行容器

```
➜  docker run --name http3_nginx -v ~/Project/Nginx/nginx-quic:/nginx -p 8080:80 -p 8443:443/udp -p 8443:443/tcp -d 35edbb5f8be1
```

> -p 8443:443 映射端口
> -d 守护进程运行
> -name http3_nginx 指定容器的名称
> -v xxxx:/nginx 将宿主机的Nginx工作目录映射进容器中的工作目录 

OK, 既然容器已经都起来了，那么访问一下吧，我本地的nginx conf 已经配置了 `quic.kiosk007.top 的 quic 配置`，使用 quic 客户端访问一下吧。

``` bash
http_client -H quic.kiosk007.top:8443 -p /ping   
HTTP/1.1 200 OK
server: nginx/1.21.1
date: Sat, 07 Aug 2021 08:57:50 GMT
content-type: text/html
content-length: 4
strict-transport-security: max-age=63072000; includeSubDomains; preload
alt-svc: h3-27=":8443"; h3-28=":8443"; h3-29=":8443"; ma=86400; quic=":8443"

pong%                                                  
```



## 提交镜像

登录到 `https://hub.docker.com/`

``` bash
➜  docker login
```

上传到 docker hub
``` bash
➜  docker push weijiaxiang007/nginx-http3:v0.1
```
在 docker hub 官网上可以找到自己上传上去的镜像。

<img src="https://img1.kiosk007.top/static/images/docker/docker_hub.png" >


大家如果想要获取我制作的 http3-nginx 镜像，可直接拉取
``` bash
docker push weijiaxiang007/nginx-http3:latest
```



```
docker container prune  # 删除所有停止的容器
docker image prune -a   # 删除没有用到的镜像
```

# kubernetes

## minikube

Minikube是由Kubernetes社区维护的单机版的Kubernetes集群。并支持Kubernetes的大部分功能，从基础的容器编排管理，到高级特性如负载均衡、Ingress，权限控制等。


安装 minikube ，可参考官方文档 [minikube start](https://minikube.sigs.k8s.io/docs/start/), 另外还需要安装 kubectl 另外做一些配置（如关掉 swap ），具体可参考我之前的[文章](https://kiosk007.top/2020/07/24/ubuntu20-04-%E9%83%A8%E7%BD%B2-Kubernetes-k8s/#%E5%AE%89%E8%A3%85k8s-master) 。

minikube 中安装 cni 是有版本依赖，老旧版本没有 --cni 选项，新版本可以支持
``` bash
➜  minikube version
minikube version: v1.22.0
commit: a03fbcf166e6f74ef224d4a63be4277d017bb62e
```

cni 插件安装：

flannel 服务在安装运行过程中依赖二进制文件 /opt/cni/bin/portmap ，没有会报错

安装 cni 后会在对应目录生成二进制文件

``` bash
➜  sudo apt install kubernetes-cni -y
```

启动 minikube 集群

```bash
➜  minikube start --cni=flannel
```

安装完成后, 查看状态 `minikube status`

``` bash
➜  minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

```

- 使用minikube

``` bash
kubectl get nodes -o wide ## 可以查看到有了一个单节点的集群，IP地址
kubectl get pods -o wide  ## 可以看到创建的pods，当然现在是空的
kubectl cluster-info      ## 查看集群Master信息
kubectl get all --namespace=kube-system	## 查看部署组件

##现在我们ssh到我们的master节点
ssh -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip)
## 也可以直接执行 minikube ssh
```

- minikube 网络控制台

``` bash
minikube dashboard
```

dashboard 是 kubernetes 的一个web UI，提供了友好的图形化界面，和对集群交互的基本操作功能。



# Nginx 

**议题**

1. K8S集群中为什么需要负载均衡器
2. 在K8S集群的边缘节点运行Nginx
3. Nginx如何发现K8S中的服务
4. K8S中的Ingress

**基础概念介绍：**

- **K8S Deployments 介绍**

为了实现在Kubernetes集群上部署容器化应用程序。需要创建一个Kubernetes Deployment，Deployment负责创建和更新应用。

典型的应用场景包括：
> a. 定义Deployment来创建Pod和ReplicaSet
> b. 滚动升级和回滚应用
> c. 扩容和缩容
> d. 暂停和继续Deployment

参考：https://www.kubernetes.org.cn/deployment


- **K8S Service 介绍**

Kubernetes Service 定义了这样一种抽象：一个 Pod 的逻辑分组，也就是为一组Pod提供统一的访问方式。通常 Pod 通过 Label Selector 实现的与 Service 绑定。

一个 Service 在 Kubernetes 中是一个 REST 对象，和 Pod 类似。 像所有的 REST 对象一样， Service 定义可以基于 POST 方式，请求 apiserver 创建新的实例。 例如，假定有一组 Pod，它们对外暴露了 9376 端口，同时还被打上 "app=MyApp" 标签。

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

参考: https://kubernetes.io/zh/docs/concepts/services-networking/service/



## 如何获取K8S集群中的服务?

一般来说暴露服务的三种方式
- **<span class="inline-tag blue">NodePort</span>**: 将服务的类型设置成NodePort-每个集群节点都会在节点上打开一个端口，并将在该端口上接收到的流量重定向到基础服务。该服务仅在内部集群 IP 和端口上才可访间， 但也可通过所有节点上的专用端口访问。**缺点是端口一般是奇怪的端口且端口肯定面临不够用的境地**

- **<span class="inline-tag blue">LoadBalane</span>**: 将服务的类型设置成LoadBalance, NodePort类型的一 种扩展，这使得服务可以通过一个专用的负载均衡器来访问， 这是由Kubernetes中正在运行的云基础设施提供的。 负载均衡器将流量重定向到跨所有节点的节点端口。客户端通过负载均衡器的 IP 连接到服务。**缺点是只是云厂商才能玩得起的（他们有路由器、有SDN）**

- **<span class="inline-tag blue">Ingress</span>**: 创建一个Ingress资源， 这是一个完全不同的机制， 通过一个IP地址公开多个服务，就是一个网关入口。通过域名即可访问。**推荐！！**

> 虽然推荐 Ingress 的形式，但是也会先以 Headless Service 的方式搭建一个Nginx集群。

**为何获取K8S中的服务是比较困难的？**

- 每一个pod都有一个由网络层提供的私有地址，在K8S集群中的任一节点上可以可达。K8S集群外部不能直接访问。
- 一组相同功能的pod构成service，K8S赋予每个Service一个cluster IP地址。Service可以从cluster IP地址访问。
- Cluster IP地址只在K8S集群内有效，不能从外部直接访问。


**外部应用如何才能访问K8S中的service?**

- K8S集群中要有一个或者多个public IP边缘节点
- 外部要访问K8S集群内的Service必须通过边缘节点的public IP地址进行访问
- 有public IP地址的边缘节点需要部署如Nginx，HAProxy等反向代理将请求转发给K8S中的service

参考上图也就是边缘节点（Node）需要部署一台Nginx，Nginx有一个 public ip，用来访问k8s中的 service。那么Nginx的部署方式就有了2种。一种是部署在 K8S 集群内，一种是部署在K8S集群外。

<img src="https://img1.kiosk007.top/static/images/k8s/k8s_nginx_deploy.png
">

2种方案有各自的优缺点。
1. Nginx 部署在K8S 外需要在Nginx的配置文件中指定nameserver；另外Nginx本身需要自己手动维护。

2. Nginx 部署在K8S 内则在Nginx的配置文件中不用指定nameserver，nameserver已由K8S配置好；K8S管理Nginx的启停；Nginx监听端口要映射到host的端口上或者使用host网络(hostNetwork:true)。

> 所以还是推荐使用部署在K8S内的方式。

 **通过NodePort发现服务**

在k8s上可以给Service设置成NodePort类型，这样的话可以让Kubernetes在其所有节点上开放一个端口给外部访问（所有节点上都使用相同的端口号）， 并将传入的连接转发给作为Service服务对象的pod。这样我们的pod就可以被外部请求访问到。至于服务都在哪些节点上，就需要动态更新Nginx的配置文件了（ngx_dynamic_upstream）

<img src="https://img1.kiosk007.top/static/images/k8s/service_nodeport_1.png">

Nginx 配置如下
``` nginx
upstream my_services {
  zone my_zone 1m;   # my_zone 提供有service pod 的节点的 node ip
}

server {
  location / {  
    proxy_passhttp://my_services;
  }
}
```

不过问题还是因为端口有限，暴露的服务也是有限的。

**通过Headless service发现服务(推荐）**

Service 是 Kubernetes 项目中用来将一组 Pod 暴露给外界访问的一种机制，如果不是 VIP 的方式访问，那么即是以 DNS 的方式访问，DNS访问又分两种，第一种，DNS的解析是VIP，那后面的访问形式和上述无异，另一种是DNS解析到POD上。

这就是 **Headless Service**，这里的区别在于，Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址。

这样的话Nginx需支持upstream动态后端服务器地址注册(nginx-upstream-dynamic-servers）

``` nginx
   upstream my_services{
      server my_services:8080 resolve;
   }
   
   server {
      location / {
         proxy_passhttp://my_services;
      }
   }
   
   # 在K8S中发布hello 服务配置nginx:
```


Nginx 可以部署在一个有外网ip（集群外部可访问ip）上。具体可以给Nginx的编排配置上加 `Label Selector`

当然，我们当前是Minikube单机模式，如果是正式的K8S环境，可以执行以下命令加标签

``` bash
kubectllabel node node3 node-type=edge
```


> 参考 [deployment 模板](https://www.cnblogs.com/wzs5800/p/13534942.html)

## 配置 minikube

minikube 和 标准的 k8s 还是有很多不一样。这里有很多坑其实我还没有踩明白。

搭建访问tunnel ip
```bash
➜  minikube tunnel

Status:	
	machine: minikube
	pid: 1158570
	route: 10.96.0.0/12 -> 192.168.49.2
	minikube: Running
	services: []
    errors: 
		minikube: no errors
		router: no errors
		loadbalancer emulator: no errors

```

挂载宿主机目录

<p class='div-border red'>minikube 需要额外执行这一步，才能将宿主机的目录挂载到Pod中的容器中 </p>

``` bash
minikube mount /home/weijiaxiang/Project/Nginx/nginx:/host
```

## 配置 k8s

可参考： [ubuntu20.04 部署 Kubernetes (k8s)](https://kiosk007.top/2020/07/24/ubuntu20-04-%E9%83%A8%E7%BD%B2-Kubernetes-k8s/)

## Nginx depolyment

新建文件: `nginx_http3-deployment.yaml`

``` ymal
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-http3-deployment
  namespace: default    #必填，Pod所属的命名空间
  labels:               #自定义标签
    app: nginx        #自定义标签名字<key: value>
  annotations:          #自定义注释列表
    imageregistry: "https://hub.docker.com/"
spec:
  selector:
    matchLabels:
      app: nginx-http3  # 必填，通过此标签匹配对应pod
  replicas: 1           # 副本集个数
  template:             # 必填，应用容器模版定义
    metadata:
      labels:           # 必填，遇上面matchLabels的标签相同
        app: nginx-http3
    spec:
      hostIPC: true  
      hostNetwork: true   # 使用主机网络
      #Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Never表示不再重启该Pod
      restartPolicy: Always   
      shareProcessNamespace: true  # Pod 里的容器共享 PID Namespace
      containers:
      - name: nginx
        image: weijiaxiang007/nginx-http3:v0.2
        imagePullPolicy: IfNotPresent   # [Always | Never | IfNotPresent] 获取镜像的策略 Alawys表示下载镜像;IfnotPresent表示优先使用本地镜像，否则下载镜像;Nerver表示仅使用本地镜像
        workingDir: "/nginx/"            # 选填，容器的工作目录
        volumeMounts:                   # 挂载到容器内部的存储卷配置
        - mountPath: "/nginx/"
          name: nginx-vol               # 引用pod定义的共享存储卷的名称
        livenessProbe:                  # 对Pod内个容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket
          httpGet:       
            path: /_check_domain_health      
            port: 80
            scheme: HTTP       
          initialDelaySeconds: 30       # 容器启动完成后首次探测的时间，单位为秒
          periodSeconds: 1200             # 对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
          timeoutSeconds: 1            # 对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
        resources:
          requests:
            cpu: 50m
            memory: 200Mi
          limits:
            cpu: 500m
            memory: 500Mi
        env:
          - name: NGINX_VERSION
            value: "1.21.0"
          - name: RUN_DEV
            value: "boe"
      - name: shell
        image: ubuntu    
        stdin: true    
        tty: true
        workingDir: "/nginx/"            # 选填，容器的工作目录
        command: ["/bin/bash","-c","sleep 100000000"]
        volumeMounts:                   # 挂载到容器内部的存储卷配置
          - mountPath: "/nginx/"
            name: nginx-vol               # 引用pod定义的共享存储卷的名称
        lifecycle:      
          postStart:        
            exec:          
              command: ["/bin/sh", "-c", "echo Nginx HTTP3 Server Start from the postStart handler > /usr/share/message"]      
          preStop:        
            exec:          
              command: ["/sbin/nginx","-s","quit"]
      volumes:                # 在该pod上定义共享存储卷列表
      - name: nginx-vol       # 共享存储卷名称 （volumes类型有很多种）
        hostPath:
          path: /host

```

创建 deployment
``` bash
kubectl create -f nginx_http3-deployment.yaml
```

查看 deployment 状态
``` bash
➜  ~ kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-http3-deployment-7cf4c654b5-6km4p   2/2     Running   0          31m
➜  ~ kubectl get deployment
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
nginx-http3-deployment   1/1     1            1           32m

```

验证
```
➜  curl https://webtransport.kiosk007.top:443/ping --resolve webtransport.kiosk007.top:443:192.168.49.2
pong
```


































