---
title: Devops
subtitle: A compound of development (Dev) and operations (Ops)
type: gallery
disallow: true
lightgallery: true
comment: false
date: 2022-06-08 23:41:26
---

> DevOps is the union of people, process, and technology to continually provide value to customers.



## Devops

### CPU 性能分析工具

`top`、`mpstat`、`pidstat`、`uptime`、`vmstate`

关注点：

- 上下文切换（cs context switch）：过多的上下文切换会导致 CPU时间消耗在寄存器、内存栈以及虚拟内存等数据的保存和恢复上，从而缩短真正运行的时间。 
  - **自愿上下文切换**：进程无法获取到所需要的资源导致的上下文切换。比如 I/O 、内存等系统资源不足
  - **非自愿上下文切换**：进程因为时间片到时原因，被系统强制调度。

- 就绪队列长度（r runing or runnable）：是正在运行和等待 CPU 的进程数。
- 不可中断睡眠状态的进程数（b Blocked）：处于不可中断睡眠状态的进程数。

<br/>

## Docker

### Docker 是什么

Docker本身所用到的隔离技术也并不是什么黑科技，都是把已有的功能翻出来拼装了一下而已。**容器的本质是一个“单进程”模型，本质是一个特殊的进程而已**

Docker 容器技术是由 Namespace、Cgroups、rootfs 三种技术构建出



- **Namespace** : Linux很早版本就实现的一个系统调用，他可以实现新创建一个进程的时候，为这个进程创建一个沙盒，有挂载点、UTS（主机名）、共享内存、进程号、网络、用户 几种
- **Cgroups**：用来限制资源使用的一种技术
- **rootfs**：挂载在容器根目录上，用来给容器进程提供隔离后执行环境的文件系统就是 rootfs（根文件系统），其利用了 Union File System 的能力，将多个目录挂载到同一个目录之上。

