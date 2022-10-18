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

- [Devops](/topic/interview/devops/#devops)
  - [CPU 性能分析工具](/topic/interview/devops/#cpu-性能分析工具)

- [Docker](/topic/interview/devops/#docker)
  - [Docker是什么](/topic/interview/devops/#docker-%E6%98%AF%E4%BB%80%E4%B9%88)

<br/>
<br/>

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

### Linux 启动顺序
- **第一阶段：硬件引导启动**
 1. Power ON 加电自检：主要检查外围设备CPU、内存等
 2. BIOS POST初始化硬件：
 3. 加载MBR到内存阶段：BIOS 读取并执行启动设备的MBR中的Bootloader
- **第二阶段：GRUB2启动引导阶段**
 1. 解析grub的配置文件 /boot 分区下 /grub/grub.conf, 显示操作系统启动菜单。
 2. 加载内存镜像到内存
 3. 通过 /boot/initrd 开头文件建立虚拟 DAM DISK 虚拟文件系统，转交给内核
- **第三阶段：内核引导阶段**
 1. 加载解压内核：执行核心的程序，为了让内核足够小，硬件驱动并没有放在内核文件里面
 2. 内核初始化：启动systemd程序，先执行initrd.target, 挂载 /etc/fstab 文件中的文件系统。
 3. 从 initramfs 根文件系统切换到磁盘的根目录

- **第四阶段：systemd初始化阶段** 
 1. systemd 执行默认target配置

<br/>

## Docker

### Docker 是什么

Docker本身所用到的隔离技术也并不是什么黑科技，都是把已有的功能翻出来拼装了一下而已。**容器的本质是一个“单进程”模型，本质是一个特殊的进程而已**

Docker 容器技术是由 Namespace、Cgroups、rootfs 三种技术构建出



- **Namespace** : Linux很早版本就实现的一个系统调用，他可以实现新创建一个进程的时候，为这个进程创建一个沙盒，有挂载点、UTS（主机名）、共享内存、进程号、网络、用户 几种
- **Cgroups**：用来限制资源使用的一种技术
- **rootfs**：挂载在容器根目录上，用来给容器进程提供隔离后执行环境的文件系统就是 rootfs（根文件系统），其利用了 Union File System 的能力，将多个目录挂载到同一个目录之上。
