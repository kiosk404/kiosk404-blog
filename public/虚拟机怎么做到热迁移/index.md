# 虚拟机怎么做到热迁移


之前买腾讯云服务的时候，有2大问题一直困扰着我。

1. 我的虚机经常CPU使用率不到5%，它会不会它拿着我的资源去偷偷接济同物理机上的其他高负载业务。反正我也不知道。
2. 我一次性买了3年的虚机，一台物理机能一直开机3年吗？总得重启一下吧，在上家公司的做运维的时候，物理机真的是不停有物理机宕机

这里就对第二个疑问做个终结，因为我知道答案了，云厂商有热迁移技术。

<!--more--> 

# 概述

当虚拟机在物理机上运行时，物理机可能存在资源分配不均，造成负载过重或过轻的情况。另外物理机本身也存在硬件的更换、组网调整、故障处理等操作，需要在不中断业务的情况下完成这些操作。

<br/>

# 应用场景

共享存储和非共享存储迁移的共同应用场景有：

- 当物理机故障或者负载过重时，可以将运行的虚拟机迁移到另一台物理机上，以避免业务中断，保证业务的正常运行。
- 当多数的物理机负载过轻时，可以将虚拟机迁移整合，以减少物理机数量，提高资源的利用率。
- 当物理服务器硬件设备成为瓶颈，比如CPU、内存、硬盘等，需要更换性能更好的硬件，或者需要增加设备，但是又不能关闭虚拟机或者停止业务。
- 服务器软件升级，比如虚拟化平台升级，就可以把虚拟机从旧版本虚拟化平台热迁移到新版本虚拟化平台。

对于非共享存储热迁移，还可以应用在如下场景：

- 当物理机故障存储空间不足，需要将运行的虚拟机迁移到另一台物理机上，可以避免业务中断，保证业务的正常运行。
- 当物理机存储设备老化，性能不能支撑当前业务数据处理，成为系统性能的瓶颈，需要更换性能更强的存储，但是又不能关闭虚拟机或者停止虚拟机，这需要将运行的虚拟机迁移到一个具有更好性能的物理机上。



<br/>

# 虚拟机迁移

虚拟机热迁移是指把虚拟机从一个物理主机（源主机）迁移到另一个物理主机（目的主机）。我们所熟知的开源 QEMU\KVM 虚拟机便支持这种技术。

KVM 虚拟机的迁移分两种方式

1. **静态迁移（冷迁移）**：对于冷迁移，就是在虚拟机关闭的状态下，将虚拟机的磁盘文件及 .xml 配置文件（这两个文件定义了一个虚拟机）复制到要迁移的目标主机上，然后在目标主机上使用 “virsh define *.xml” 重新定义这个虚拟机即可。
2. **动态迁移（热迁移）**：对于热迁移，就比较常用，通常是这台服务器上一直跑着的一些业务，而这些业务又不能中断，那么就需要使用热迁移了。

<br/>

## 冷迁移

通常是存放虚拟机的磁盘的目录是挂载一个NFS文件系统的磁盘，而这个磁盘通常是 LVM 文件系统，需要进行冷迁移，只要在目标主机上挂载这个 NFS 文件系统，就可以看到要迁移的那个虚拟机的磁盘文件，通常以 .qcow2 或 .raw 结尾。然后只需要将虚拟机的 .xml 配置文件发送到目标服务器上，重新你定义一下，就可以看到迁移过来的虚拟机。

<br/>

## 热迁移

如果原宿主机和目的宿主机是共享存储系统的，那么只需要通过网络发送客户机的 vCPU 执行状态、内存中的内容、虚拟机设备的状态到目的主机上。

KVM 动态迁移的具体过程为：

1. 迁移开始时，客户机依然在宿主机上运行，于此同时，客户机的内存页被传输到目的主机上。<font color="red">小结</font>：*[全量迁移阶段]*
2. QEMU/KVM 会监控并记录下迁移过程中所有已被传输的内存页上的修改，并在所有内存页都传输完成之后开始传输在前面过程中内存页的更改内容。<font color="red">小结</font>：*[增量迭代迁移阶段]，该阶段对内存进行多轮迭代，使得剩余脏数据逐渐减少*
3. QEMU/KVM 会估计迁移过程中的传输速度，当剩余内存数据量能够在一个可以设定的时间周期（默认30毫秒）内传输完成时，QEMU/KVM 会关闭宿主机上的客户机，再将剩余的数据量传输到目的主机上，最后传输过来的内存内容在目的宿主机上恢复客户机的运行状体。<font color="red">小结</font>：*[停机迁移阶段]，当剩余内存脏数据能够在停机时间内完成迁移，就会暂停虚拟机，将剩余的内存一次性迁移到目的主机*
4. 至此，KVM虚拟机的动态迁移操作就完成了。其虚拟机上的运行的业务不会受到影响。

<br/>

## 注意事项和约束限制

- 热迁移过程中，需要保证网络状态的良好，如果发生网络中断，热迁移会暂停，直到网络恢复后才会继续，当发生超时，热迁移会失败
- 迁移过程中，不允许对虚拟机进行生命周期和管理虚拟机硬件设备等操作
- 只支持同构热迁移，即源端和目的端CPU型号需要相同。
- 跨业务网段虚拟机迁移可以成功，但是到目的端后会出现网络异常，为了防止该情况的发生，需要用户保证迁移业务的网段一致。

<br/>

# virsh 虚拟化热迁移

Libvirt工具采用XML格式的文件描述一个虚拟机特征，包括虚拟机名称、CPU、内存、磁盘、网卡、鼠标、键盘等信息。用户可以通过修改配置文件，对虚拟机进行管理。

## 安装虚拟化依赖

1. 安装libvirt

```bash
$ sudo apt install qemu-kvm \
libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager
```

- <font color="7CFC00">qemu-kvm</font> - 为 KVM 管理程序提供硬件模拟的软件程序
- <font color="7CFC00">libvirt-daemon-system</font> - 将 libvirt 守护程序作为系统服务运行的配置文件
- <font color="7CFC00">libvirt-clients</font> - 用来管理虚拟化平台的软件
- <font color="7CFC00">bridge-utils</font> - 用来配置网络桥接的[命令行工具](https://cloud.tencent.com/product/cli?from=10680)
- <font color="7CFC00">virtinst</font> - 用来创建虚拟机的命令行工具
- <font color="7CFC00">virt-manager</font> - 提供一个易用的图形界面，并且通过libvirt 支持用于管理虚拟机的命令行工具

<br/>

2. 启动并验证

```bash
$ systemctl start libvirtd
$ systemctl is-active libvirtd
active
```

验证内核是否支持KVM虚拟化

```bash
$ ls /dev/kvm
/dev/kvm
$ ls /sys/module/kvm
parameters uevent
```



<br/>

## 准备虚拟机环境

- **制作镜像**

虚拟机镜像是一个文件，包含了已经完成安装并且可启动操作系统的虚拟磁盘。虚拟机镜像具有不同格式，常见的有raw格式和qcow2格式。

```bash
$ qemu-img create -f qcow2 ubuntu-image.qcow2 5G
Formatting 'ubuntu-image.qcow2', fmt=qcow2 size=5368709120 cluster_size=65536 lazy_refcounts=off refcount_bits=16

```

<br/>

- **准备虚拟机网络**

KVM 虚拟化支持Linux网桥、Open vSwitch 网桥等多种类型的网桥。

{{< image src="https://img1.kiosk007.top/static/images/blog/virtual-network-structure.png" caption="虚拟网络结构图" src_s="https://img1.kiosk007.top/static/images/blog/virtual-network-structure.png" src_l="https://img1.kiosk007.top/static/images/blog/virtual-network-structure.png">}}



上述安装 libvirtd 时会自动启动一个 virbr0 网桥。

```bash
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:1e:17:9a brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.202/24 brd 192.168.122.255 scope global enp1s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe1e:179a/64 scope link 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:db:7b:e0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.123.1/24 brd 192.168.123.255 scope global virbr0
       valid_lft forever preferred_lft forever

```

<br/>

- **定义 Domain XML**

定义一个基本元素及总线元素 x86_64 架构虚拟机的 XML 配置文件

```xml
<domain type='kvm'>
  <name>Ubuntu-VM</name>
  <memory unit='KiB'>2097152</memory>
  <currentMemory unit='KiB'>2097152</currentMemory>
  <vcpu placement='static'>1</vcpu>
  <iothreads>1</iothreads>
  <os>
    <type arch="x86_64" machine="pc-q35-4.2">hvm</type>
  </os>
  <features>
    <acpi/>
  </features>
  <cpu mode='host-passthrough' check='none' migratable='on' />
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' discard='unmap'/>
      <source file='/mnt/ubuntu-image.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
    </disk>
    <disk type='file' device='cdrom'> 
      <driver name='qemu' type='raw'/>
      <source file='/root/kvm/ubuntu-20.04-live-server-amd64.iso'/>
      <boot order='1'/>
      <backingStore/>
      <target dev='sdb' bus='sata'/>
      <readonly/>
      <alias name='sata-cdrom'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <controller type='scsi' index='0' model='virtio-scsi'>
    </controller>
    <controller type='virtio-serial' index='0'>
    </controller>
    <controller type='usb' index='0' model='ehci'>
    </controller>
    <controller type='sata' index='0'>
    </controller>
    <controller type='pci' index='0' model='pcie-root'/>
    <interface type='bridge'>
      <mac address='52:54:00:db:7b:e0'/>
      <source bridge='virbr0'/>
      <model type='virtio'/>
    </interface>
    <serial type='pty'>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
    </input>
    <input type='keyboard' bus='usb'>
      <address type='usb' bus='0' port='2'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='vga' vram='16384' heads='1' primary='yes'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
    </memballoon>
  </devices>
</domain>

```

<br/>

## 创建虚拟机

- **创建虚拟机**

虚拟机XML配置文件为 ubuntu-VM.xml

```bash
$ virsh define ubuntu-vm.xml 
Domain 'Ubuntu-VM' defined from ubuntu-vm.xml
```

- **启动虚拟机**

```bash
$ virsh start Ubuntu-VM
Domain 'Ubuntu-VM' started
```

- **添加ISO**

如果忘记在 XML 添加CDROM设备。可以使用 virt-xml 命令添加 CDROM 。（上面的XML我后来加上了）

```bash
$ virt-xml Ubuntu-VM --add-device \
--disk /root/kvm/ubuntu-20.04-live-server-amd64.iso,device=cdrom
```



- **使用VNC Viewer连上之后进行安装**

<img src="https://img1.kiosk007.top/static/images/blog/virsh_kvm.png" alt="virsh_kvm" style="zoom:40%;clear:both;display:block;margin:auto;" />

- **安装完成卸载 ISO**

```bash
$ virsh change-media Ubuntu-VM sdb --eject
Successfully ejected media.
```



更多 virsh 相关的命令 [管理虚拟机](https://docs.openeuler.org/zh/docs/22.03_LTS/docs/Virtualization/%E7%AE%A1%E7%90%86%E8%99%9A%E6%8B%9F%E6%9C%BA.html)

<br/>



## 热迁移虚拟机

前提条件：

> - 进行热迁移之间需要确保源端和目的端宿主机之间的网络是互通的，并且可获取资源是对等的
> - 在执行虚拟机热迁移之前应当对虚拟机进行健康检查，确保目的端主机有足够的CPU、内存和存储资源

<br/>

- **热迁移脏页率预测(可选)**

用户在迁移前可以使用 dirtyrate 功能，获取热迁移的内存脏页变化速率，根据虚拟机内存使用情况评估虚拟机是否适合迁移或配置合理的迁移参数。

不过我本地的 libvirtd 执行报错，这里不掩饰了。



- **设置热迁移参数(可选)**

在执行热迁移之前，可以通过使用过 virsh migrate-setmaxdowntime 来指定虚拟机热迁移过程中可以容忍的最大停机时间，这是一个可选项配置

如配置最大停机时间为 500ms

```bash
$ virsh migrate-setmaxdowntime Ubuntu-VM 500
```

<br/>

同时可以通过调用 virsh migrate-setspeed 来限制虚拟机热迁移占用的带宽大小，防止虚拟机热迁移时占用带宽过大，对宿主机上的其他虚拟机或业务造成影响，同样是一个可选项

如配置虚拟机热迁移带宽为 500Mbps

```bash
$ virsh migrate-setspeed Ubuntu-VM --bandwidth 500

# 查看
$ virsh migrate-getspeed Ubuntu-VM
```

<br/>

用户可以使用 migrate-set-parameters 来设置热迁移时的相关参数，与热迁移压缩的参数如下：

1. compress-level: 压缩级别，默认：1
2. compress-threads: 压缩线程数目，默认：8
3. compress-wait-thread: 是否等待压缩线程，默认：true
4. decompress-threads: 解压缩线程数目，默认：2
5. compress-method: 压缩算法选择（zlib、zstd），默认：zlib

如将压缩级别改成1

```bash
$ virsh qemu-monitor-command Ubuntu-VM '{ "execute": "migrate-set-parameters", "arguments": {"compress-level": "1"}}'

```

<br/>

最后查看整体的热迁移相关参数。

```bash
virsh qemu-monitor-command Ubuntu-VM '{ "execute": "query-migrate-parameters"}' --pretty
{
  "return": {
    "xbzrle-cache-size": 67108864,
    "cpu-throttle-initial": 20,
    "announce-max": 550,
    "decompress-threads": 2,
    "compress-threads": 8,
    "compress-level": 1,
    "multifd-channels": 2,
    "announce-initial": 50,
    "block-incremental": false,
    "compress-wait-thread": true,
    "downtime-limit": 500,
    "tls-authz": "",
    "announce-rounds": 5,
    "announce-step": 100,
    "tls-creds": "",
    "max-cpu-throttle": 99,
    "max-postcopy-bandwidth": 0,
    "tls-hostname": "",
    "max-bandwidth": 524288000,
    "x-checkpoint-delay": 20000,
    "cpu-throttle-increment": 10
  },
  "id": "libvirt-383"
}

```

- **热迁移操作（非共享存储场景）**

首先，现在目标宿主机上创建一个虚拟磁盘文件。磁盘格式和大小必须保持一致。

```bash
$ qemu-img create -f qcow2 /mnt/ubuntu-vm.qcow2 5G
```

- 在源宿主机使用 virsh migrate 进行迁移

```bash
$ virsh migrate --live --unsafe \
--copy-storage-all --migrate-disks vda Ubuntu-VM qemu+ssh://192.168.122.203/system

```

注意需要提前配置好 ssh 免密登陆，而且对应主机名的 解析需要提前写进 /etc/hosts 文件中，否则会报错`error: internal error: unable to execute QEMU command 'blockdev-add': address resolution failed for instance004:49153: Name or service not known`

- **热迁移完成**

我们在目标主机 instance004 上可以看到，虚拟机已经完整的迁移过来了。

```bash
root@instance004:/mnt# virsh list --all
 Id   Name        State
---------------------------
 9    Ubuntu-VM   running

```





<br/>

# 高负载虚拟机热迁移遇到的挑战

## QEMU 热迁移压缩算法效率低

原生 QEMU 热迁移内存数据使用的压缩算法是 zlib，其存在的如下问题：

1. 单核的压缩性能只有 100MB/s 左右
2. 满足万兆带宽场景，QEMU 压缩线程的 CPU 消耗高达 1000% 

由此引发的后果是性能无法满足高负载场景下的需求，在带宽受限的场景下可能无法完成热迁移，导致功能失效。且严重消耗物理主机的CPU资源，导致物理主机其他业务系统性能下降

目前业界已有的解决方案包括优化 QEMU 的 zlib 压缩算法的性能或者将压缩算法卸载到硬件（加速卡、INTEL QAT 等）

<br/>

## 磁盘热迁移流程中的CPU空耗

QEMU 进行跨存储热迁移时，在迁移数据库时，会通过 BITMAP 来记录磁盘脏数据块。原生 QEMU 的磁盘BITMAP 大小粒度是 1M，后续就要迁移这1M数据，这就带来了磁盘脏数据的放大问题。假设其实只有4K数据发生了修改，但是这1M都需要进行迁移。这就大量消耗在干净数据块的无效遍历上。



<br/>

## QUME 的 CPU 节流算法导致业务性能受影响时间较长

当进入到 *增量迭代迁移阶段* 时，如果内存脏数据的生成速率大于迁移速率，迁移则永远无法完成，原生的 QEMU 通过 CPU 节流策略来解决此问题。具体如下：

1. 如果本轮内存脏数据量超过上轮迁移的 50% ，QEMU 会开始对虚拟机进行 CPU 节流，先削减 20% 的 vCPU （虚拟机CPU）执行时间
2. 在每轮内存迭代迁移开始时，QEMU 会根据之前的削减情况，决定后续的削减粒度
3. 如果新增脏数据仍然过多，QEMU 会继续削减 10% 的 vCPU 执行时间
4. QEMU 最高会削减 99% 的 vCPU 执行时间

原生QEMU热迁移的 CPU 节流策略较为保守和呆板，在高业务负载场景下，CPU 节流持续时间长，导致 业务性能持续受到影响。

<br/>

## 热点内存的重复无效迁移

根据内存的局部性原理，虚拟机业务在运行过程中，会存在大量的热点内存，在短时间内被频繁修改，虚拟机热迁移进入到增量迭代迁移阶段时，热点内存由于被频繁更高，导致每轮迭代迁移都会变脏而重复迁移。这就需要额外的一个算法去识别热点内存区域，先迁移非热点内存区域，最后再高效迁移热点区域。

<br/><br/>





- [Live Migrating QEMU-KVM Virtual Machines](https://developers.redhat.com/blog/2015/03/24/live-migrating-qemu-kvm-virtual-machines)
- [OpenEuler Docs -- 认识虚拟化](https://docs.openeuler.org/zh/docs/22.09/docs/Virtualization/%E8%AE%A4%E8%AF%86%E8%99%9A%E6%8B%9F%E5%8C%96.html)
