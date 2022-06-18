---
title: Service Mesh 是什么？
author: kiosk
date: 2021-09-19 11:40:25
categories:
  - k8s
tags:
  - service mesh
---
下面是摘自维基百科的一段话。

> 微服务 (Microservices) 是一种软件架构风格，它是以专注于单一责任与功能的小型功能区块 (Small Building Blocks) 为基础，利用模块化的方式组合出复杂的大型应用程序，各功能区块使用与语言无关 (Language-Independent/Language agnostic) 的 API 集相互通信。

看起来过于抽象，Service Mesh 在我看来，就是如何处理服务于服务之间的关系，如服务之间的相互调用关系，核心服务的熔断等，这不仅不含一个普通的 A 服务对 B 服务的一次 RPC 调用，也涉及RPC调用时具体的超时时间、重试次数等细节。最终目的是让整个服务形成一张网一般。而开发人员只需要注重自己的业务逻辑开发，而服务之间的联系全部交给 Service Mesh。

想象一下，你的代码不再需要考虑各种超时逻辑、也不需要考虑熔断和集成各种底层网络库。你的代码只有业务本身。

<!--more-->

# Service Mesh 的概念

微服务的架构特性

1. 特点一：围绕业务构建团队

在最开始的时候，可能公司分为 前端RD、后端RD、数据库DBA、运维OPS，整个公司的全部前端服务全部交由前端RD开发，后端代码逻辑全部交由后端RD开发。比如互联网一部负责整个商城系统开发，商城系统服务本身错综复杂，但是大家写的代码都会部署到一起，是个单体服务。

微服务的诞生打破了这个结构。比如商城系统被拆分成了 售卖组、商品直播组、安全组、售后组等等，每个组自身有后端和前端开发，每个组专注于自身业务。每个组的业务本身也是分离的，互不影响。

这样的好处是服务彼此独立，独立部署，没有依赖，比如售卖组的业务量大，售卖组的服务可以多部署几份。售后组的服务少部署几份。

<img src="https://img1.kiosk007.top/static/images/service_mesh/feature1.png">



2. 特点二：去中心化的数据管理

由于业务部署的独立，数据库也会面临一定的拆分，比如每个业务用到的数据库 database 是不同的库，业务自身使用的是不同的表。

<img src="https://img1.kiosk007.top/static/images/service_mesh/feature2.png">


为了满足上述的业务服务越拆越细的需求。微服务的概念便出现，随之微服务的几个核心便是`服务注册\发现`,`路由\流量转移`,`弹性能力、熔断、超时`,`安全`,`可链路追踪\监控`，而这些核心的实现便是 **Service Mesh**


Service Mesh 发展至今，从最开始的“原始通信时代” 到依托于K8S的“SideCar”模式才发展成了真正的 Service Mesh

<img src="https://img1.kiosk007.top/static/images/service_mesh/history.png">

参考：
- [什么是 Service Mesh](https://zhuanlan.zhihu.com/p/61901608)


# istlo

istlo 是什么？

> Istio 提供一种简单的方式来为已部署的服务建立网络，该网络具有负载均衡、服务间认证、监控等功能，而不需要对服务的代码做任何改动。


- istio 适用于容器或虚拟机环境（特别是 k8s），兼容异构架构。
- istio 使用 sidecar（边车模式）代理服务的网络，不需要对业务代码本身做任何的改动。
- HTTP、gRPC、WebSocket 和 TCP 流量的自动负载均衡。
- istio 通过丰富的路由规则、重试、故障转移和故障注入，可以对流量行为进行细粒度控制；支持访问控制、速率限制和配额。
- istio 对出入集群入口和出口中所有流量的自动度量指标、日志记录和跟踪。

## 安装

安装教程参考官方 (https://istio.io/latest/zh/docs/setup/getting-started/), 不过网站似乎国内打不开，下面演示一下 。

- **准备 Kubernetes 环境**

kubernetes 环境版本：v1.21.4

安装方法参考：[ubuntu20.04 部署 Kubernetes (k8s)](https://kiosk007.top/2020/07/24/ubuntu20-04-%E9%83%A8%E7%BD%B2-Kubernetes-k8s/)

- **下载 istio**

下载针对你操作系统的安装文件
```
$ curl -L https://raw.githubusercontent.com/istio/istio/master/release/downloadIstioCandidate.sh | sh -
```
如果想要指定版本、不同处理器体系则可以使用

```
$ curl -L https://raw.githubusercontent.com/istio/istio/master/release/downloadIstioCandidate.sh | ISTIO_VERSION=1.6.8 TARGET_ARCH=x86_64  sh -
```

转到istio的目录。例如 istio-1.11.2
```
cd istio-1.11.2
```
安装目录包含: `samples/` 目录下的示例应用程序 和 `bin/` 目录下的 `istioctl`客户端二进制工具。可将 `istioctl` 加入到 PATH 中。


- **安装 istio**

istio本身提供不同模式的 `Configuration Profile`

<img src="https://img1.kiosk007.top/static/images/service_mesh/configuration.png">

对于本次安装，我们选择 `demo` 模式，这个是专门为了展示和学习而准备的。

```
$ istioctl install --set profile=demo -y
✔ Istio core installed                                                                                                                                                          
✔ Istiod installed                                                                                                                                                              
✔ Ingress gateways installed                                                                                                                                                    
✔ Egress gateways installed                                                                                                                                                     
✔ Installation complete                                                                                                                                                         
Thank you for installing Istio 1.11.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/kWULBRjUv7hHci7T6

```

给 namespace 添加标签，指示istio在部署应用时自动注入 envoy sidecar 代理。可以简单的理解为当部署应用清单时，它会帮你改写清单，把具体的sidecar配置给你添加进去。

```
kubectl label namespace default istio-injection=enabled
```

查看istio部署的pods

```
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-6cb7bdc7fb-8h2n6    1/1     Running   0          6m15s
istio-ingressgateway-694d8d7656-z4ccc   1/1     Running   0          6m15s
istiod-6c68579c55-lmp8r                 1/1     Running   0          7m

```


# 部署 bookinfo 应用

bookinfo 是官方推荐的一个微服务案例。主要包含以下4个微服务，主页productpage服务是由python开发的，主要会调用评论和详细内容两个服务。评论服务有三个版本，reviews 微服务有 3 个版本：

> v1 版本不会调用 ratings 服务。
> v2 版本会调用 ratings 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
> v3 版本会调用 ratings 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

<img src="https://img1.kiosk007.top/static/images/service_mesh/bookinfo.png">

```
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created

```

应用很快会起来，等到应用就绪后，istio 的sidecar 代理将一起被部署。
```
$ kubectl get service
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.80.140.94   <none>        9080/TCP   61s
kubernetes    ClusterIP   10.80.0.1      <none>        443/TCP    24d
productpage   ClusterIP   10.80.6.178    <none>        9080/TCP   61s
ratings       ClusterIP   10.80.32.218   <none>        9080/TCP   61s
reviews       ClusterIP   10.80.60.33    <none>        9080/TCP   61s
```

和 

``` bash
kubectl get pods
NAME                                      READY   STATUS            RESTARTS   AGE
details-v1-79f774bdb9-kl2dg               0/2     PodInitializing   0          2m40s
nfs-client-provisioner-6d9cb7bf7d-wclzl   1/1     Running           38         24d
productpage-v1-6b746f74dc-xt7qh           0/2     PodInitializing   0          2m39s
ratings-v1-b6994bb9-r8pqp                 2/2     Running           0          2m40s
reviews-v1-545db77b95-l6ctg               0/2     PodInitializing   0          2m40s
reviews-v2-7bf8c9648f-72k95               2/2     Running           0          2m40s
reviews-v3-84779c7bbc-zkp5r               0/2     PodInitializing   0          2m40s

```


- **创建 Ingress**

此时的bookinfo应用已经部署，但是还不能被外界访问，想要访问则需要创建 istio 入站网关 Ingress，他会在网格边缘把一个路径映射到路由。

``` bash
$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

- **确定入站 IP 和 端口**

访问网关有2个变量控制，<span style="color:red">INGRESS_HOST </span> 和 <span style="color:red">INGRESS_PORT</span> 

``` bash
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.80.53.173   <pending>     15021:32628/TCP,80:30554/TCP,443:30266/TCP,31400:30402/TCP,15443:30959/TCP   83m

```
设置 <span style="color:red">EXTERNAL-IP</span> 的值之后， 你的环境就有了一个外部的负载均衡，可以用它做入站网关。 但如果 <span style="color:red">EXTERNAL-IP</span> 的值为 <none> (或者一直是 <pending> 状态)， 则你的环境则没有提供可作为入站流量网关的外部负载均衡。 在这个情况下，你还可以用服务（Service）的 [节点端口](https://kubernetes.io/zh/docs/concepts/services-networking/service/#nodeport) 访问网关 (但是k8s的NodePort设置要求的范围是 30000-32767)。

由于我本地的测试环境没有 Loadblance ，所以就用NodePort方式访问吧。

``` bash
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
$ export INGRESS_HOST=172.16.101.135
$ export GATEWAT_URL=$INGRESS_HOST:$INGRESS_PORT
```


浏览器访问 `http://172.16.101.135:30554/productpage` 可以看到这个微服务。

- **查看仪表盘**

istio 可以可以添加 kiali 仪表盘、Prometheus、grafana 还有 Jaeger 等。

1. 安装 并 等待其部署完成
``` bash
kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system
```
2. 访问 kiali 仪表盘

kiali 在希腊语中是望远镜的意思，其官方定义是 istio 的可观察性控制台，通过服务拓扑帮助你理解服务网格的结构。通过服务拓扑帮助你理解服务网格的结构，并且具有配置网格的功能。

<img src="https://img1.kiosk007.top/static/images/service_mesh/kiali_funcation.png">

``` bash
istioctl dashboard kiali
```
> 我因为是在远端虚拟机搭建的k8s集群，所以将 kiali 改为 NodePort 模式

3. 在左侧的导航栏菜单选择 `Graph`, 然后在Namespace下拉列表选择 default。kiali 仪表盘提供了网络的概览，以及 bookinfo 示例应用的各个服务的关系，它还提供过滤器可视化流量的流动。


<img src="https://img1.kiosk007.top/static/images/service_mesh/kiali.png">

## Virtual Service 动态路由

要路由到一个版本，需要为微服务进行默认模板的设置，在这种情况下，Virtual Service 将所有的流量路由到每一个v1版本的pod上。

**Virtual Service 的作用正是 1. 定义路由规则。 2. 满足描述条件的请求去哪里**

这里涉及到一个满足描述条件，这正是**目标规则 (Destination Rule) 的功能，即 1. 定义子集策略。 2. 描述到达目标的子集怎么处理**

1. 执行以下命令，使所有的 review 流量都打到 review v1 版本上

``` bash
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```
<img src="https://img1.kiosk007.top/static/images/service_mesh/virtualservice.png">

``` yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
...
```

这个清单定义了4个 Virtual Service ，可以看到 所有的服务都被打向了 v1 版本。v1 版本是怎么定义出来的呢？在 `destination-rule-all.yaml` , `subsets` 中，将不同的labels中定义的版本进行标记，如  **subsets.name.labels.version: v1**, 可以使用 `kubectl get destinationrules` 查看

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
...
```

## Gateway 管理进入网格的流量

网关是运行在网格边缘的负载均衡器，接收外部请求转发到网格内部的服务，配置对外端口与内部服务的对应关系。


<img src="https://img1.kiosk007.top/static/images/service_mesh/gateway.png">


在之前搭建 bookinfo 时，其实我们已经创建了一个网关，这个网关暴露了bookinfo 这个微服务的对外应用 `productpage` 。


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080

```

网关的 VirtualService 有一个 gateways, 正是上面创建 `Gateway` ，这个服务有5个被暴露出来的 uri，只有匹配的 uri 的流量才被打到 productpage 服务里。

## ServiceEntry 服务入口

服务入口的作用是添加外部服务到内部网格内，是希望将管理内部服务队外部服务的请求，起到扩展网格的功能。

一般默认情况下，整个微服务是可以访问外部服务的。所以需要我们先禁止以下微服务访问外部。

 Istio有一个安装选项`global.outboundTrafficPolicy.mode`用来配置sidecar对外部服务(未在istio内部服务注册中定义的服务)的处理方式。 如果选项值为<span style="color:red">ALLOW_ANY</span>，sidecar将允许调用未知的服务，如果选项值为<span style="color:red">REGISTRY_ONLY</span>，那么Istio代理将会阻止调用任何未在服务网格注册中定义的服务或ServiceEntry的host。

查看当前的规则则可以使用以下命令，没有任何输出表示默认的 `ALLOW_ANY`
``` bash
kubectl get istiooperator installed-state -n istio-system -o jsonpath='{.spec.meshConfig.outboundTrafficPolicy.mode}'
```

在禁止直接访问之前。安装一个带 `curl` 命令的pod，访问微服务外的服务。

``` bash
$ kubectl apply -f samples/sleep/sleep.yaml
$ kubectl exec  sleep-557747455f-svc9k -- curl http://httpbin.org/headers
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/7.78.0-DEV", 
    "X-Amzn-Trace-Id": "Root=1-6147eadc-3c6f1cdb3884436b76fca6f1", 
    "X-B3-Sampled": "1", 
    "X-B3-Spanid": "b0c944c749e6dcd8", 
    "X-B3-Traceid": "5822ffcb03b68b61b0c944c749e6dcd8", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Envoy-Peer-Metadata": "ChkKDkFQUF9DT05UQUlORVJTEgcaBXNsZWVwChoKCkNMVVNURVJfSUQSDBoKS3ViZXJuZXRlcwoZCg1JU1RJT19WRVJTSU9OEggaBjEuMTEuMgrEAQoGTEFCRUxTErkBKrYBCg4KA2FwcBIHGgVzbGVlcAohChFwb2QtdGVtcGxhdGUtaGFzaBIMGgo1NTc3NDc0NTVmCiQKGXNlY3VyaXR5LmlzdGlvLmlvL3Rsc01vZGUSBxoFaXN0aW8KKgofc2VydmljZS5pc3Rpby5pby9jYW5vbmljYWwtbmFtZRIHGgVzbGVlcAovCiNzZXJ2aWNlLmlzdGlvLmlvL2Nhbm9uaWNhbC1yZXZpc2lvbhIIGgZsYXRlc3QKGgoHTUVTSF9JRBIPGg1jbHVzdGVyLmxvY2FsCiAKBE5BTUUSGBoWc2xlZXAtNTU3NzQ3NDU1Zi1zdmM5awoWCglOQU1FU1BBQ0USCRoHZGVmYXVsdApJCgVPV05FUhJAGj5rdWJlcm5ldGVzOi8vYXBpcy9hcHBzL3YxL25hbWVzcGFjZXMvZGVmYXVsdC9kZXBsb3ltZW50cy9zbGVlcAoXChFQTEFURk9STV9NRVRBREFUQRICKgAKGAoNV09SS0xPQURfTkFNRRIHGgVzbGVlcA==", 
    "X-Envoy-Peer-Metadata-Id": "sidecar~10.100.1.55~sleep-557747455f-svc9k.default~default.svc.cluster.local"
  }
}

```


将 istio改为禁止调用未在网格内注册过的服务。
``` bash
$ istioctl install --set profile=demo -y --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY

$ kubectl get istiooperator installed-state -n istio-system -o jsonpath='{.spec.meshConfig.outboundTrafficPolicy.mode}'
REGISTRY_ONLY
```

再访问已经无法访问
...

使用ServiceEntry将一个外部服务注册到服务网格中

``` bash
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
EOF

```

此时便可再访问外部服务。这里的 `location: MESH_EXTERNAL` 便是指定访问网格外的服务白名单

参考：https://blog.frognew.com/2021/07/learning-istio-1.10-11.html



## 流量转移：灰度发布

istio 可以实现改写路由的权重，可以完成 灰度发布、蓝绿部署、A/B 测试 等功能。

其本质还是利用 Virtual Service 进行服务权重的调整，bookinfo 里的 reviews 服务有3个版本，我们即可利用 reviews 服务进行测试。

``` yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```

其本质是修改路由的权重值。

## Egress 控制出口流量

查看 egress 网关已安装。

``` bash
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-68cc7d6d78-bnwc8                1/1     Running   0          13h
istio-egressgateway-6cb7bdc7fb-8h2n6    1/1     Running   0          17h
istio-ingressgateway-694d8d7656-z4ccc   1/1     Running   0          17h

```
查看之前的 Service Entry 已配置

``` bash
$ kubectl get se
NAME          HOSTS             LOCATION        RESOLUTION   AGE
httpbin-ext   ["httpbin.org"]   MESH_EXTERNAL   DNS          49m
```

配置 Egress 网关、Destination 和 Virtual Service （Destination 和 Virtual Service 配置略）。

``` bash
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - httpbin.org

```

详见： https://cloud.tencent.com/developer/article/1469811


## 超时重试

一般一个微服务架构里，最常见的就是服务之间的相互调用。这种调用关系是基于网络的，网络是不可靠的，另外被调用方也有可能因为服务故障导致处理请求慢，调用方对其他服务的调用必然涉及到超时重试，一般来说，业务的服务应该关注业务而不是在这些调用超时、重试之类的细枝末节上，所以微服务的超时重试便由此诞生。

<img src="https://img1.kiosk007.top/static/images/service_mesh/timeout_retry.png">


演示过程：
- 给 ratings 服务添加延迟
- 给 reviews 服务添加超时策略
- 给 ratings 服务添加重试策略


先将 review 切换到 v2 版本
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2

```

为 rating 服务添加延迟 2s

``` yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percent: 100.0
        fixedDelay: 2s
    route:
    - destination:
        host: ratings
        subset: v1

```

为 review 服务添加超时时间 1s 

``` yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
    timeout: 1s

```

此时review 服务将无法响应，页面上可以看到挂掉了。下来验证重试和超时。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 2
      perTryTimeout: 1s

```

## 熔断

熔断是一种过载保护的手段, 目的是避免服务的级联失败。

一个熔断器可以有三种状态：关闭、打开和半开，默认情况下处于关闭状态。在关闭状态下，无论请求成功或失败，到达预先设定的故障数量阈值前，都不会触发熔断。而当达到阈值时，熔断器就会打开。当调用处于打开状态的服务时，熔断器将断开请求，这意味着它会直接返回一个错误，而不去执行调用。通过在客户端断开下游请求的方式，可以在生产环境中防止级联故障的发生。在经过事先配置的超时时长后，熔断器进入半开状态，这种状态下故障服务有时间从其中断的行为中恢复。如果请求在这种状态下继续失败，则熔断器将再次打开并继续阻断请求。否则熔断器将关闭，服务将被允许再次处理请求。


<img src="https://img1.kiosk007.top/static/images/service_mesh/circuit_breaking.png">

设置熔断器

``` bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

参考: https://www.servicemesher.com/blog/istio-circuit-breaking/

## 日志

系统的运行状态，除了通过 istio 自带的 `prometheus`、`kiali`、`grafna` 查看相关的日志。但是总是需要看更详细的文本日志输出。

原始的调用结构是 服务A 直接调用 服务B ，但是现在服务A 的sidecar 调用服务B 的sidecar然后才能调用服务B 。

那么想要定位问题，就是查看envoy的日志了。 


首先确保envoy 日志是打开的
``` bash
$kubectl describe configmap istio -n istio-system | grep "accessLogFile"
accessLogFile: /dev/stdout

```
envoy 的日志是定制的，在服务的pod里叫做 `istio-proxy` 这个contianer。

``` bash
$ kubectl logs -f productpage-v1-6b746f74dc-xt7qh istio-proxy

[2021-09-20T14:12:23.277Z] "GET /details/0 HTTP/1.1" 200 - via_upstream - "-" 0 178 24 23 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36" "768f9804-849c-9aca-b4f0-f85c838304e4" "details:9080" "10.100.1.49:9080" outbound|9080|v1|details.default.svc.cluster.local 10.100.1.51:54256 10.80.140.94:9080 10.100.1.51:46546 - -
[2021-09-20T14:12:23.310Z] "GET /reviews/0 HTTP/1.1" 200 - via_upstream - "-" 0 295 41 41 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36" "768f9804-849c-9aca-b4f0-f85c838304e4" "reviews:9080" "10.100.0.19:9080" outbound|9080|v1|reviews.default.svc.cluster.local 10.100.1.51:54950 10.80.60.33:9080 10.100.1.51:55736 - -
[2021-09-20T14:12:23.253Z] "GET /productpage HTTP/1.1" 200 - via_upstream - "-" 0 4288 106 105 "10.100.0.0" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36" "768f9804-849c-9aca-b4f0-f85c838304e4" "172.16.101.135:30554" "10.100.1.51:9080" inbound|9080|| 127.0.0.6:33335 10.100.1.51:9080 10.100.0.0:0 outbound_.9080_._.productpage.default.svc.cluster.local default
[2021-09-20T14:12:23.411Z] "GET /static/bootstrap/css/bootstrap.min.css HTTP/1.1" 304 - via_upstream - "-" 0 0 32 29 "10.100.0.0" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36" "8df34a40-a462-99b3-afe3-ea90df5f09d4" "172.16.101.135:30554" "10.100.1.51:9080" inbound|9080|| 127.0.0.6:53397 10.100.1.51:9080 10.100.0.0:0 outbound_.9080_._.productpage.default.svc.cluster.local default
[2021-09-20T14:12:23.429Z] "GET /static/bootstrap/css/bootstrap-theme.min.css HTTP/1.1" 304 - via_upstream - "-" 0 0 20 19 "10.100.0.0" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36" "11b436e3-bad2-9608-b389-9eb8f422ca68" "172.16.101.135:30554" "10.100.1.51:9080" inbound|9080|| 127.0.0.6:48115 10.100.1.51:9080 10.100.0.0:0 outbound_.9080_._.productpage.default.svc.cluster.local default
[2021-09-20T14:12:23.436Z] "GET /static/jquery.min.js HTTP/1.1" 304 - via_upstream - "-" 0 0 18 17 "10.100.0.0" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36" "105258af-c512-9df4-b8cd-7063eb29ee70" "172.16.101.135:30554" "10.100.1.51:9080" inbound|9080|| 127.0.0.6:50457 10.100.1.51:9080 10.100.0.0:0 outbound_.9080_._.productpage.default.svc.cluster.local default
[2021-09-20T14:12:23.436Z] "GET /static/bootstrap/js/bootstrap.min.js HTTP/1.1" 304 - via_upstream - "-" 0 0 26 24 "10.100.0.0" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36" "adfefa36-12a2-90a2-b223-11c829cc2133" "172.16.101.135:30554" "10.100.1.51:9080" inbound|9080|| 127.0.0.6:52945 10.100.1.51:9080 10.100.0.0:0 outbound_.9080_._.productpage.default.svc.cluster.local default

```

envoy 日志配置项


| 配置项        | 说明   |
| --------   | -----:  |
| global.proxy.accessLogFile |  日志输出文件，空为关闭输出 |
| global.proxy.accessLogEncoding |  日志编码格式：JSON、TEXT |
| global.proxy.accessLogFormat | 配置显示在日志中的字段，空为默认格式 |
| global.proxy.logLevel  | 日志级别，空为 warning |























