# Kubernets 源码学习

> Kubernetes, also known as K8s, is an open-source system for automating deployment, scaling, and management of containerized applications.

本项目主要记录 Kubernets 相关云原生相关知识，并加以自己的分析与注解。


# 内容更新
- 本人博客: [**kiosk's Blog**](https://kiosk007.top) 
    - [ubuntu20.04 部署kubernets](https://kiosk007.top/post/ubuntu20-04-%E9%83%A8%E7%BD%B2-kubernetes-k8s/)
    - [K8S 架构拓扑](https://kiosk007.top/post/k8s%E6%9E%B6%E6%9E%84%E6%8B%93%E6%89%91/)
    - [Kubernetes 调度对象](https://kiosk007.top/post/kubernetes-%E8%B0%83%E5%BA%A6%E5%AF%B9%E8%B1%A1/)

- Kubernetes 官方源码：https://github.com/kubernetes/kubernetes

# 重新认识 Kubernets 
1. 为什么需要 Kubernetes,它能做什么？
容器是打包和运行应用程序的最好方式，他极大的解决了开发环境和生产环境不相同的情况。
Kubernets 提供了
- **服务发现和负载均衡**：使用 DNS 或者自己的 IP 地址公开容器访问，如果进入容器的流量很大，Kubernets 可以负载均衡并分配网络流量，从而使得部署稳定
- **存储编排**：允许自动挂载选择的储存系统，如本地提供、云厂商提供的云盘
- **自动部署和回滚**: 可以受控的速率进行灰度发布上线。
- **自动完成装箱计算**：允许指定每个容器所需的 CPU 和内存（RAM）
- **自我修复**：重启失败的容器、替换容器、杀死有问题的容器
- **弹性扩缩容**：自动根据当前负载情况进行扩容操作
- **密钥与配置管理**：允许存储和管理敏感信息，如 密码、OAuth 令牌等，无需在操作系统或代码中暴露。