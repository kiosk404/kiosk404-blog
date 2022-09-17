---
title: 【译】虚拟机生命周期
author: kiosk
tags: 
  - openstack
categories:
  - cloud_computing
date: 2022-09-12 11:23:00
---

本文翻译自 Libvirt 官方文档 [VM Lifecycle](http://wiki.libvirt.org/page/VM_lifecycle)

<!--more-->

# 虚拟机生命周期

本文介绍虚拟机的生命周期的基本要素，目的在于提供创建、运行、迁移和删除虚拟机的基本信息。

## 术语

知道文档那个中使用的术语及命令和语法的意义一直很重要。如果你不熟悉 Libvirt 的基本概念，如节点（Node）、域（Domain）、以及为了获取Libvirt 项目目标、覆盖范围的概括，请参考这篇[文章](https://libvirt.org/goals.html)。

> 这里我再备注一下：
>
> - Node：一个物理机节点
> - Hypervisor：也称为**虚拟机监控器 VMM ( virtual machine monitor )，可用于创建和运行虚拟机 ( VM) 的进程**。 它是一种中间软件层，运行在基础物理服务器和操作系统之间，可允许多个操作系统和应用共享硬件。 通过让Hypervisor以虚拟化的方式共享其资源（如内存和处理资源），一台主机计算机可以支持多台客户机虚拟机。
> - Domain：相当于一个虚拟机实例

<br/>

# 概念

## Guest Domain 用 XML 来描述

在 Libvirt 中，XML 文件被用来保存所有东西的配置信息，包括 Domain、网络、存储和其他元素。XML 使得用户可以使用舒适的编辑器及易于集成的其他技术和工具。

<br/>

例如，Domain 中的设备（Device）通过赋予XML元素或子元素属性来表示。一个 XML 片段如下：

```xml
<domain type='qemu'>
   <name>demo</name>
   ...
   <devices>
      ...
      <disk type='file' device='disk'> ... </disk>
      <disk type='file' device='cdrom'> ... </disk>
      <input type='mouse' bus='ps2'/>
      ...
   </devices>
</domain>
```

Libvirt 使用了 XPath 技术来选取XML文档中的节点（node）。

<br/>

## 短暂型 Guest Domain 和 持久型 Guest Domain

Libvirt 区分两种不同类型的 Domain：短暂型的（Transient）和持久型的（Persistent）。

- 短暂型 Domain 只存在于 Domain 被关闭或者宿主机重启之前
- 持久型 Domain 会一直存在（直到被删除）

不管什么类型，一旦一个 Domain 创建起来，只要源文件还存在它的状态就能够保存进一个文件和无限次的恢复（restore）。所以即使是一个短暂型 Domain 也是可以一遍又一遍的恢复。

<br/>

创建短暂型Domain 和创建持久型Domain有一些不一样，持久型Domain需要在启动前定义（Define）；短暂型 Domain 只创建和启动一次，处理这两种类型的 Domain 命令也不一样。

虽然短暂型 Domain 可以随时（on the fly）创建和销毁，但它所需的组件（如存储、网络、设备等）都需要提前准备好。

<br/>

## Guest Domain 状态

Domain 可以有以下几个状态：

1. **Undefined** - 这是初始状态（baseline state）。Libvirt 不知道处于这个状态的 Domain 的所有信息，因为 Domain 还没有定义或者创建。
2. **Defined** 或 **Stopped** - Domain 已经定义，但是还没有运行。这个状态也称为 stopped，只有持久型的 Domain 才可以处于这个状态，当一个短暂型 Domain 停止或关闭时，其就不存在了。
3. **Running** - Domain 已经创建和启动的状态，两种 Domain 处于这个状态都是已经在宿主机的 Hypervisor 执行起来。
4. **Paused** - Domain 在 Hypervisor 上的运行被挂起。它的状态会临时地存储起来直到恢复运行。它本身并不知道自己是否被暂停。如果你熟悉操作系统中的进程，你会发现这两者很相似。
5. **Saved** - 类似于 paused 状态，但不同的是 Domain 的状态会被存储到持久性存储上，同样，处于这个状态的Domain 可以被恢复，而且不会察觉到时间的流逝。（虚拟机的快照功能）

<br/>

下图展示了 Domain 状态之间的转化，长方形便是不同的状态，箭头表示将一个状态转化为另一个状态的命令。

<img src="https://img1.kiosk007.top/static/images/blog/vm_lifecycle_graph.png" alt="vm_lifecycle_graph" style="zoom:150%;" />

从上图中，可以看到， shutdown 命令使得 Domain 从 running 状态转化为 defined 或者 undefined 状态，在这种情形下，短暂型 Domain 会变为 undefined 状态（销毁），持久性 Domain 会变为 defined 状态。

<br/>

## 快照

快照是一个虚拟机操作系统及其所有应用在某一时刻的视图（View）。在虚拟机领域，能够创建虚拟机的可恢复快照是一个基本的特性。快照允许用户及时保存虚拟机在某时刻的状态，以及回滚到那个状态。基本的用例包括生成快照、安装新的应用、更新或升级，然后回滚到前一个时间点。

<br/>

显而易见，快照生成之后的任何变化都不会包含在该快照中，快照不会持续更新，他只代表某一时刻虚拟机的状态。

<br/>

## 迁移

一个运行中的 Domain 或者虚拟机可以迁移到另一个虚拟机，只要虚拟机的存储在虚拟机之间共享以及目的宿主机可以支持虚拟机的 CPU 模式（CPU model）。根据类型和应用，虚拟机的迁移不会引起任何服务的中断。

<br/>

Libvirt 支持不同的迁移类型：

- **Standard (标准模式)** - Domain 被挂起，将其资源传送到目的宿主机。一旦传送完成，虚拟机从目的宿主机恢复运行。挂起的时间直接跟 Domain 的内存大小成正比。
- **Peer-to-peer (对等模式)** - 当原宿主机和目的宿主机能够直接通信时会使用该模式。
- **Tunnelled (隧道模式)** - 在原宿主机和目的宿主机之间建立一条隧道，如 SSH 隧道，所有原宿主机和目的宿主机或物理主机之间的网络通信都会通过该隧道。
- **Live vs no-live (实时 VS 非实时模式)** - 当在实时模式时，Domain 不会暂停，其上的所有服务也会继续运行。一开始，目的宿主机上的 Domain 或 虚拟机处于关闭状态，而且在通过网络传输Domain 状态的时候，在目的宿主机上的 Domain 实际是不可见的，实时迁移对应用的负载很敏感。当实时迁移一个 Domain 时，它所分配的内存被发送到目的宿主机，同时在原宿主机监控其变化，在原宿主机上的 Domain 会保持 active 状态直到其在两个宿主机上的内存一致，在此时，目的宿主机的上的 Domain 变为 active 状态，原宿主机上的 Domain 变为 passive 状态或不可见状态。
- **Direct (直接模式)** - Libvirt 利用 Hypervisor 触发迁移，而且整个迁移过程都在 Hypervisor 的监控之下。Hypervisor 通常有互相之间直接通信的特性（如原宿主机上的 Xen 可以跟目的宿主机上的 Xen 直接通信，而不需要 Libvirt 的干预）。

<br/>

迁移的要求：

1. 可以在相同路径和位置访问共享存储，如 iSCSI、NFS 等
2. 双方物理主机使用同一个版本的 Hypervisor
3. 相同的网络配置
4. 相同或更好的 CPU，CPU 必须来自同一个厂商，而且目的宿主机上的 CPU flag 必须是原宿主机的 CPU flag 的超集。

迁移的相关文章可以在[这里](https://www.libvirt.org/migration.html)找到

<br/>

## 删除 Domain 时的数据安全性

一些保存了敏感信息的应用都应该安全的处理，就像文件系统中的任一文件，当虚拟机被删除时，只是把文件系统的指针删除，原先被占用的区块（block）还是继续被占用着，他们只是被文件系统标记为空，实际上，这取决于你的文件系统。

<br/>

# 任务

**开启此章节之前先介绍一下相关工具的准备**



安装相关工具：

```bash
# 安装环境 ubuntu 22.04  
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager
```

- *qemu-kvm* - 为KVM管理程序提供硬件模拟的软件程序
- *libvirt-daemon-system* - 将 libvirt 守护程序作为系统服务运行
- *libvirt-clients* - 用来管理虚拟化平台的客户端命令行工具
- *bridge-utils* - 用来配置网桥的命令行工具
- *virtinst* - 用来创建虚拟机的命令行工具
- *virt-manager* - 提供一个易用的图形界面，并且通过 libvirt 支持用于管理虚拟机的命令行工具

<br/>

验证安装成功

一旦软件包被安装好，libvirt 守护程序将自动启动，通过以下命令去验证。

```bash
➜  sudo systemctl is-active libvirtd
active
```

<br/>

想要创建和管理虚拟机，需要将当前使用用户添加到 *libvirt* 和 *kvm* 用户组中

```bash
➜  sudo usermod -aG libvirt $USER
➜  sudo usermod -aG kvm $USER   
```



<br/>

## 创建 Domain

在运行 Domain 之前需要先创建一个。有很多办法可以创建 Domain。这篇 [文章](https://wiki.libvirt.org/page/CreatingNewVM_in_VirtualMachineManager) 介绍了通过 Virtual Machine Manager GUI 来创建 Domain ，第二种方法是使用 virt-install 命令行工具来创建。

```bash
virt-install \
             --connect qemu:///system \
             --virt-type kvm \
             --name MyNewVM \
             --ram 512 \
             --disk path=/var/lib/libvirt/images/MyNewVM.img,size=8 \
             --vnc \
             --cdrom /var/lib/libvirt/images/Fedora-14-x86_64-Live-KDE.iso \
             --network network=default,mac=52:54:00:9c:94:3b \
             --os-variant fedora14
```

这个命令创建了一个基于 KVM 的 Domain，名为 MyNewVM，RAM 为 512MB，磁盘空间为 8GB。

<br/>

还有一种方法是创建一个 Domain XML 定义文件和存储卷（volume），以及运行 virsh 命令：vol-create 和 define。

存储卷会被加入一个存储池（pool）中。默认地，存在一个叫为 *default* 的存储池。这是一个目录类型的存储池，意味着所有的卷都以文件的方式存储在一个目录中。如果你不是很清楚 Libvirt 的存储管理，请阅读这篇[文章](http://libvirt.org/storage.html)。

<br/>

XML 定义文件（new_volume.xml）例子：

```xml
<volume>
 <name>sparse.img</name>
 <capacity unit="G">10</capacity>
</volume>
```

这定义了一个容量为 10 GB 的卷，在 *default* 存储池中创建卷：

```bash
virsh vol-create default new_volume.xml
```

另一个 XML 定义文件（MyNewVM.xml）例子

```xml
<domain type='kvm'>
  <name>MyNewVM</name>
  <currentMemory>524288</currentMemory>
  <memory>524288</memory>
  <uuid>30d18a08-d6d8-d5d4-f675-8c42c11d6c62</uuid>
  <os>
    <type arch='x86_64'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/><apic/><pae/>
  </features>
  <clock offset="utc"/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <vcpu>1</vcpu>
  <devices>
    <emulator>/usr/bin/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/MyNewVM.img'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <disk type='block' device='cdrom'>
      <target dev='hdc' bus='ide'/>
      <readonly/>
    </disk>
    <interface type='network'>
      <source network='default'/>
      <mac address='52:54:00:9c:94:3b'/>
      <model type='virtio'/>
    </interface>
    <input type='tablet' bus='usb'/>
    <graphics type='vnc' port='-1'/>
    <console type='pty'/>
    <sound model='ac97'/>
    <video>
      <model type='cirrus'/>
    </video>
  </devices>
</domain>
```

定义一个新的持久性 Domain：

```
virsh define MyNewVM.xml
```

Domain XML 文件格式有很多有用的可选元素。所以，请阅读这篇包括样例和大多共性场景的完整 Domain XML 格式参考[文章](http://libvirt.org/formatdomain.html) 。

<br/>

## 编辑 Domain

编辑 Domain 可以使用自己喜欢的编辑器。需要设定 **VISUAL** 或者 **EDITOR** 环境变量，没有设置变量的话，默认使用 vi 编辑器。关闭编辑器后，Libvirt 会自动检查内容变化并应用 （apply）它们，另外，在 Virtual Machine Manager 中也可以编辑 Domain

```
virsh edit <domain>
```

<br/>

## 启动 Domain

一旦一个 Domain 被创建好，就可以开始运行它了，可以通过 Virtual Machine Manager 或者执行 virsh start 命令，如：

```
virsh start MyNewVM
```

这个命令可以执行干净启动 （clean boot up） 或者回复 Domain 之前保存的状态。需要注意的是，其组件如网络等设备也是需要准备好的。

就如前面提到的，短暂型 Domain 可以直接运行而无需定义 （define）动作：

```
virsh create /path/to/MyNewVM.xml
```

<br/>

## 关闭或重启 Domain

关闭 Domain：

```
virsh shutdown <domain>
```

重启一个持久型 Domain:

``` 
virsh reboot <domain>
```

重启一个 短暂型 Domain 是不可能的，因为它在关机（shutdown）后已经不在了（undefined）。

粗野关机（inelegant shutdown），也称为强制关机（hard-stop），这等同于拔电源：

```
virsh destory <domain>
```

<br/>

## 暂停 Domain

暂停 Domain

``` 
virsh suspend <domain>
```

或者在 Virtual Machine Manager 中主工具栏点击 *Pause* 按钮。当虚拟机处于挂起状态（Suspended State），它占用系统 RAM 资源而不是处理器资源；磁盘和网络 I/O 也不会发生。

<br/>

## 继续运行 Domain

任何一个暂停或者挂起的 Domain 都可以恢复运行：

```
virsh resume <domain>
```

或者通过反点击（unclick） Virtual Machine Manager 的 *Pause* 按钮。

<br/>

## 生成快照

生成快照

```
virsh snapshot-create <domain>
```

<br/>

## 列出快照

列出 Domain 快照

```
virsh snapshot-list <domain>
```

输出结果可能看起来像这样子：

```
Name                 Creation Time             State
---------------------------------------------------
 1295973577           2011-01-25 17:39:37 +0100 running
 1295978837           2011-01-25 19:07:17 +0100 running
```

我们可以看到，一个快照是在当地时间 17:39:17 创建的，名为 1295973577，代表着 Unix 时间；另一个快照是在 19:07:17 创建，名为 1295978837。

<br/>

## 从快照恢复 Domain

从一个快照中恢复 Domain ：

```
virsh snapshot-restore <domain> <snapshotname>
```

这就恢复 Domain 到快照保存的状态。注意，Domain 的任何变化都会被销毁。

<br/>

## 移除 Domain 的快照

移除 Domain 的快照

```
virsh snapshot-delete <domain> <snapshotname>
```

<br/>

## 迁移

Libvirt 提供了虚拟机迁移的支持，这意味着可以通过网络来迁移虚拟机，迁移主要有两种模式：

1. Plain migration （简单迁移）：原宿主机新建一条通向目的宿主机的未加密 TCP 连接以传输数据。除非手工指定了端口，Libvirt 会在 49152 - 49215 中选择一个。
2. Tunneled migration（隧道迁移）：原宿主机 libvirtd 进程新建一条通向目的宿主机 libvirtd 的直连通道以传输数据，这允许我们对数据进行加密，但需要 qemu 0.12.0 和 libvirt 0.7.2 及以上版本。

一个成功的迁移有很多事情需要完成，对于虚拟机，存储配置需要匹配得上、待迁移 Domain 使用的存储卷都要放在同一路径下。完整的迁移参考：[Guest migration](https://www.libvirt.org/migration.html)

一旦可以迁移，可以使用 virsh 来迁移虚拟机了：

```
virsh migrate <domain> <remote host URI> --migrateuri tcp://<remote host>:<port>
```

<br/>

## 删除 Domain

我们可以删除一个 inactive Domain：

```
virsh undefine <domain>
```

图形化管理工具 Virtual Machine Manager 可以参考[这篇文章](https://wiki.libvirt.org/page/DeletingVirtualMachine_in_VirtualMachineManager)删除

<br/>

## 擦除 Domain 存储数据

Domain 使用的存储卷可能会包含保密数据，所以在移除它之前需要擦除上面的数据：

```
virsh vol-wipe <volume>
```

这截断（truncate）或扩展（extend）该卷为其原始的大小，实际上是用0来填充文件，这保证之前存储在该卷上的数据不再可读。

最后，可以删除卷了

```
virsh vol-delete <volume>
```

<br/>

<br/>

# Reference

以下网页还能提供更有用的一些信息：

- [Domain XML format](http://libvirt.org/formatdomain.html)
- [RHEL 8 - Configuring and managing virualization](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_virtualization/virtualization-in-rhel-8-an-overview_configuring-and-managing-virtualization)
- [Anatomy of the libvirt virtualization library](http://www.ibm.com/developerworks/linux/library/l-libvirt/index.html?ca=dgr-lnxw97LXlibvirt-APIdth-LX&S_TACT=105AGX59&S_CMP=lnxw97#basic_architecture)