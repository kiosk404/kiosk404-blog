---
title: Nginx Ingress
author: kiosk
tags:
  - nginx
  - k8s
categories:
  - k8s
date: 2021-09-05 11:37:00
---
在K8S的中， Service 暴露给外界的三种方法。其中有一个叫作 LoadBalancer 类型的 Service，它会为你在 Cloud Provider（比如：Google Cloud 或者 OpenStack）里创建一个与该 Service 对应的负载均衡服务。

但是，由于每个 Service 都要有一个负载均衡服务，所以这个做法实际上既浪费成本又高。作为用户，我其实更希望看到 Kubernetes 为我内置一个全局的负载均衡器。然后，通过我访问的 URL，把请求转发给不同的后端 Service。

**这种全局的、为了代理不同后端 Service 而设置的负载均衡服务，就是 Kubernetes 里的 Ingress 服务。**

<!--more-->


# 简介

 **Kubernetes** 内置一个全局的负载均衡器。然后，通过访问 URL，把请求转发给不同的后端 Service。这就是这种全局的、为了代理不同后端 Service 而设置的负载均衡服务，就是 Kubernetes 里的 Ingress 服务。所以，Ingress 的功能其实很容易理解：**所谓 Ingress，就是 Service 的“Service”。**
 
<img src="https://img1.kiosk007.top/static/images/k8s/ingress/ingress.png"> 
 
 
# 安装 nginx-ingress

参考: [NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/)

1. 创建 `Nginx Controller`
 - 创建 `Namespace` 与 `账号`
 - 创建角色并绑定账号
 - 创建 default server 的秘钥
 - 创建存放 `nginx.conf` 的 `Config Map`
 - 创建 `ingress-class`
 - 创建 `Nginx Controller Pod`
2. 暴露 `Nginx Controller` 服务
3. 创建 `Ingress` 规则
 - `Host` 精准与通配符匹配
 - `Path` 前缀或精确匹配
 - `Backend`
 
第一步：创建 `Nginx Controller`

``` bash
$ git clone https://github.com/nginxinc/kubernetes-ingress/
$ cd kubernetes-ingress/deployments
$ git checkout v1.12.0
```

Namespace 与 账号

``` bash
$ kubectl apply -f common/ns-and-sa.yaml
$ kubectl apply -f rbac/rbac.yaml
```

default server 秘钥证书 与 Config Map

``` bash
$ kubectl apply -f common/default-server-secret.yaml
$ kubectl apply -f common/nginx-config.yaml
$ kubectl apply -f common/ingress-class.yaml
```

创建 `Nginx-Ingress` 当一个新的 Ingress 对象由用户创建后，nginx-ingress-controller 就会根据 Ingress 对象里定义的内容，生成一份对应的 Nginx 配置文件（/etc/nginx/nginx.conf），并使用这个配置文件启动一个 Nginx 服务。

``` bash
$ kubectl apply -f deployment/nginx-ingress.yaml

```

为了让用户能够用到这个 Nginx，我们就需要创建一个 Service 来把 Nginx Ingress Controller 管理的 Nginx 服务暴露出去。创建 `Node Port` , 暴露服务。Kubernetes将在集群的每个节点上随机分配两个端口。可以使用 **任意节点** + **分配的特殊端口** 即可访问服务。


``` bash
$ kubectl create -f service/nodeport.yaml

$ kubectl get svc -n nginx-ingress
NAME            TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
nginx-ingress   NodePort   10.80.23.67   <none>        80:30595/TCP,443:30920/TCP   4m32s

```

这个 Service 的唯一工作，就是将所有携带 ingress-nginx 标签的 Pod 的 80 和 433 端口暴露出去。


# 测试

Nginx 官方 ingress 提供了 cafe 、 tea 的例子，参考: https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/complete-example

``` bash
IC_IP=172.16.101.135   # 宿主机ip
IC_HTTPS_PORT=30920    # nodeport
```

部署

``` bash
kubectl create -f cafe.yaml
kubectl create -f cafe-secret.yaml
kubectl create -f cafe-ingress.yaml
```

验证：

``` bash
$ curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/coffee --insecure
Server address: 10.100.1.37:8080
Server name: coffee-6f4b79b975-59d8f
Date: 04/Sep/2021:15:36:01 +0000
URI: /coffee
Request ID: 1873881292193359134a0619136b2780

```

# 配置


 
每次文件变化后，执行 `kubectl apply -f cafe-ingress.yaml`，实际会对Nginx进行一次reload。


``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        pathType: Prefix
        backend:
          service:
            name: tea-svc
            port:
              number: 80
      - path: /coffee
        pathType: Prefix
        backend:
          service:
            name: coffee-svc
            port:
              number: 80
```

执行上述命令后，会生成一个 ingress 规则。

``` bash
$ kubectl get ingress 
NAME           CLASS   HOSTS              ADDRESS   PORTS     AGE
cafe-ingress   nginx   cafe.example.com             80, 443   14h
```

而这个ingress 规则实质上是在 `ingress controller` 的 deployment pod 中的 `/etc/nginx/conf.d` 中生成一个规则。

``` bash
$ kubectl exec -it -n nginx-ingress nginx-ingress-c7cd9948d-r8hxz -- /bin/bash
nginx@nginx-ingress-c7cd9948d-r8hxz:/$ cd /etc/nginx/conf.d/
nginx@nginx-ingress-c7cd9948d-r8hxz:/etc/nginx/conf.d$ ls
default-cafe-ingress.conf

```


`nginx.conf` 相关配置遵从 configmap 模板配置, 同样更改会reload。
详见：`https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/`

> k8s 官方的 ingress-nginx 是利用 balance by lua 的方式进行负载均衡。nginx 官方是利用 nginx reload 实现。


# ingress 工作原理

K8S 调度 `Nginx Ingress Controller Pod`, Go 语言基于Nginx 模板生成的 `nginx.conf` 

- 模板语法 `import "text/template"`
- 模板位置 Nginx 官方:
 - `/nginx.ingress.tmpl` 和 `nginx.tmpl`
- 填充模板的数据：
 - `ConfigMap`, `Ingress`, `Service` 和  `Secrets`

配置文件改变后，**Nginx 官方的 Ingress (https://github.com/nginxinc/kubernetes-ingress) ** 的做法是每次 reload。而 **kubernetes 官方的 Ingress **是基于 lua 实现的类似openresty 的 `balance by lua` 来实现配置重载。

而配置文件本身是全部存在的 ETCD 中的。ETCD 基于 raft 协议实现高可用的一致性。具体可以参考我之前的文章 [etcd 的基本入门](https://kiosk007.top/2020/05/30/go-etcd-%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%93%8D%E4%BD%9C/)


Nginx 官方的Ingress 和 K8S 官方的Ingress 性能对比。

- 静态部署：即 Nginx 配置文件不变, 可以看到开源的Nginx（蓝线）在压力逐渐增加时，时延变高的最慢。

<img src="https://img1.kiosk007.top/static/images/k8s/ingress/ingress_static_stress.png">

- 动态部署: 即 Nginx 配置总是变，如 upstream 变化导致需要配置变化。官方Nginx的性能就拉垮了，压力到90多就时延上升一个台阶，到99.9时时延已经1分钟以上了。

<img src="https://img1.kiosk007.top/static/images/k8s/ingress/ingress_dynamic_stress.png">


参考：[K8S Ingress Controller技术细节探讨](https://www.nginx.org.cn/course/detail/12)


## Ingress Controller

Ingress Controller 完成了包含 `负载均衡/会话保持`、`协议转换`、`TLS 卸载、认证`、`文本压缩`、`请求认证`、`限流限速`、`Waf`、`全链路跟踪`、`日志监控` 等等功能。

## **第三方模块启用**

Nginx 有大量的第三方模块，Ingress 复用 Nginx 时这些大量的第三方模块是用 `annotations` 的方式加载这些第三方模块。详见 [K8S官方文档 Annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) 和 [Nginx 官方文档](https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/advanced-configuration-with-annotations/)

``` yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
  annotations:
    nginx.org/proxy-connect-timeout: "30s"
    nginx.org/proxy-read-timeout: "20s"
    nginx.org/client-max-body-size: "4m"
spec:
  # ingressClassName: nginx # use only with k8s version >= 1.18.0
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80

```


<img src="https://img1.kiosk007.top/static/images/k8s/ingress/ingress_annotations.png" >



- opentrace 全链路追踪

Nginx 本身可以遵从全链路追踪 `opentracing` 架构规范实现（https://opentracing.io/）。通过为流行的平台提供一致的，富有表现力的，供应商中立的API，OpenTracing使开发人员可以轻松地通过O（1）配置更改添加（或切换）跟踪实现。OpenTracing还为OSS检测和特定于平台的跟踪帮助程序库提供了通用语言。

比如全链路追踪的 `jaeger trace`。jaeger 本身搭建比较简单，但是其数据源需要 ingress Nginx 按照 opentracing 规范导入到 jaeger 中最使用 UI 查看。

<img src="https://img1.kiosk007.top/static/images/k8s/ingress/jaeger_trace.png" />

`nginx ingress` 中的nginx 支持如 `https://github.com/opentracing-contrib/nginx-opentracing.git` 等模块。

详见: [采用 NGINX Ingress Controller for Kubernetes 支持 OpenTracing](https://www.nginx.org.cn/article/detail/517)







## **nginx.conf 配置加载**

nginx 的 config 配置文件是通过 `ConfigMap` 搭配 go 语言的 template 模板来更新 `nginx.conf` 。


``` yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-config
  namespace: nginx-ingress
data:
  worker-processes: "2"

```

如下是进入到 `nginx ingress controller` pod 里看到的内容，`ConfigMap` 里data中的内容，`worker-processes: "2"` 会对应到 pod 里的 `nginx.tmpl` 的 `{{.WorkerProcesses}}`


``` bash
nginx@nginx-ingress-c7cd9948d-r8hxz:/$ ls
bin   dev		   docker-entrypoint.sh  home  lib64  mnt	     nginx.ingress.tmpl  nginx.transportserver.tmpl  opt   root  sbin  sys  usr
boot  docker-entrypoint.d  etc			 lib   media  nginx-ingress  nginx.tmpl		 nginx.virtualserver.tmpl    proc  run	 srv   tmp  var
nginx@nginx-ingress-c7cd9948d-r8hxz:/$ cat nginx.tmpl 

worker_processes  {{.WorkerProcesses}};
{{- if .WorkerRlimitNofile}}
worker_rlimit_nofile {{.WorkerRlimitNofile}};{{end}}
{{- if .WorkerCPUAffinity}}
worker_cpu_affinity {{.WorkerCPUAffinity}};{{end}}
{{- if .WorkerShutdownTimeout}}
worker_shutdown_timeout {{.WorkerShutdownTimeout}};{{end}}
daemon off;

```


## k8s与Nginx 官方 Ingres 区别

进程架构

- K8S 官方
  - /usr/bin/dumb-init
  - nginx-ingress-controller
  - /usr/local/nginx/sbin/nginx -c /etc/nginx/nginx.conf 
- Nginx 官方
  - nginx-ingress
  - /usr/sbin/nginx
  
- **K8S 官方优势**

可以看出 K8S 官方是对容器的理解非常深，容器是单进程架构，这并不意味着容器里面只能跑一个进程，而是指容器内只能管理一个进程。K8S 的这一个进程就是 `/usr/bin/dumb-init` 仿照了操作系统中的 `systemd` 进程对这个POD容器进行管理。

而Nginx官方的Ingress 是 `nginx-ingress` 进程。

另一个就是K8S官方的负载均衡配置变更是用的 `Balance By Lua` ，这个在配置频繁变更时性能会很高。

- **Nginx 官方优势**

但是 Nginx 官方的 Ingress 明显是对 `nginx` 的理解更加深刻的。Ingress 默认是不支持正则匹配，Nginx 官方则对此有优化，这个就是 `VirtualServer`、`VirtualServerRoute`。

<img src="https://img1.kiosk007.top/static/images/k8s/ingress/virtual_server.png" >


另外，Nginx 官方还提供了 `snippets` 。支持更高级的匹配模式，但其实也是一种全局的匹配。 具体参考 [Advanced Configuration with Snippets](https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/advanced-configuration-with-snippets/)

支持添加 302 跳转，支持全局加 Header 。

``` yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress-with-snippets
  annotations:
    nginx.org/server-snippets: |
      location / {
          return 302 /coffee;
      }      
    nginx.org/location-snippets: |
            add_header my-test-header test-value;
spec:
  rules:
  - host: cafe.example.com
  ....
```


另外Nginx官方支持自定义模板。具体参考 [Custom Annotations](https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/custom-annotations/)






