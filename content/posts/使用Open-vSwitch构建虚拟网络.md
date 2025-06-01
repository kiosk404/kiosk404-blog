---
title: 使用Open-vSwitch创建虚拟网络
author: kiosk
fontawesome: true
tags:
  - openstack
categories:
  - Cloud Computing
date: 2022-08-22 11:23:00
---

前面的文章 [虚拟化-创建一个虚拟机](https://kiosk007.top/post/%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E8%99%9A%E6%8B%9F%E6%9C%BA/) 介绍了如何创建一个虚拟机，但是整个Iaas生态，网络的虚拟化也是非常大的一块。我们需要将同一台物理机上的虚拟机连通，虚拟机可以上网，甚至还需要将不同物理机上的虚拟机连通。

而 OpenvSwitch 就是为了上述需求而诞生的SDN软件。



# OpenvSwtich 

OpenvSwitch (简称 OVS) 是一个用 C 语言开发的多层虚拟交换机。现如今基本上已经成为了开源 SDN（软件定义网络）基础设施层的事实标准。



## OVS 支持功能

- 支持标准 802.1Q VLAN协议，允许端口配置trunk模式。
- 支持组播
- 支持多种隧道协议（GRE、VXLAN、STT、Geneve 和 IPsec）
- 支持内核和用户空间的转发引擎选项

<br/>

## OVS的术语解释

* **Bridge**

中文名称**网桥**，一个 Bridge 代表一个以太网交换机（Switch），一台主机可以创建一个或多个 Bridge，Bridge 可以根据一定的规则，把某一个端口接收到的数据报文转发到另一个或多个端口上，也可以修改或丢弃数据报文。

- **Port**

交换机上的插口，可以接水晶头，Port隶属于 Bridge，必须先添加了 Bridge 才能在 Bridge 上添加 Port。

> ​     **Normal**：用户可以把操作系统中已有的网卡添加到 OVS 上，OVS会自动生成一个同名的Port。此类型的 Port 常用于 VLAN 模式的多台物理主机相连的口，交换机的一端属于 Trunk 模式

> ​     **Internal**：当Port的类型是 Internal 时，OVS会自动创建一个虚拟网卡（Interface），此端口收到的数据报文都会转发到这个网卡。

> ​    **Patch**：Patch Port 和 veth pair 功能相同，总是成双成对的出现，在其中一端收到的数据报文会被转发到另一个 Patch Port 上，就像是一根网线一样，Patch Port 常用于链接两个 Bridge，使两个网桥合并成为一个网桥

> ​     **Tunnel**：OVS 支持 GRE、VXLAN、IPsec 隧道协议，这些隧道协议就是 overlay 网络的基础协议，通过对物理网络做的一层封装和扩展，解决跨二层网络的问题。

**Interface**

接口是OVS与操作系统交换数据报文的组件，一个接口即是操作系统上的一块网卡，这个网卡可能是 OVS 生成的虚拟网卡，也可能是挂载在 OVS 上的物理网卡，操作系统上的虚拟网卡（TAP/TUN）也可以被挂载在 OVS 上。

**Controller**

OpenFlow 控制器，OVS 可以接受一个或者多个 OpenFLow 控制器的管理。功能主要是下发流表、控制转发规则。

**Flow**

流表是OVS进行数据转发的核心功能，定义了端口之间的转发数据报文的规则，一条流表规则主要分为匹配和动作两部分，匹配部分决定哪些数据报文需要被处理，动作决定了匹配到的数据报文该如何处理。

<br/>

## 网络术语

1. **什么是VLAN**

VLAN（Virtual Local Area Network）即虚拟局域网，是将一个物理的LAN在逻辑上划分成多个广播域的通信技术。
每个VLAN是一个广播域，VLAN内的主机间可以直接通信，而VLAN间则不能直接互通。这样，广播报文就被限制在一个VLAN内。

详见：[什么是VLAN](https://info.support.huawei.com/info-finder/encyclopedia/zh/VLAN.html)

2. **什么是VXLAN**

VXLAN（Virtual eXtensible Local Area Network，虚拟扩展局域网），是对传统VLAN协议的一种扩展。VXLAN的特点是将L2的以太帧封装到UDP报文（即L2 over L4）中，并在L3网络中传输。

VXLAN本质上是一种隧道技术，在源网络设备与目的网络设备之间的IP网络上，建立一条逻辑隧道，将用户侧报文经过特定的封装后通过这条隧道转发。从用户的角度来看，接入网络的服务器就像是连接到了一个虚拟的二层交换机的不同端口上

详见：[什么是VXLAN](https://support.huawei.com/enterprise/zh/doc/EDOC1100087027)

3. **什么是访问链接和汇聚链接**

访问链接（access link）:指的是“只属于一个VLAN，且仅向该VLAN转发数据帧”的端口。一般用于链接计算机端口。

汇聚链接（trunk link）：指的是能够转发多个不同VLAN的通信的端口。汇聚链路上流通的数据帧，都被附加了用于识别分属于哪个VLAN的特殊信息。一般用于交换机与交换机之间的相关接口。

4. **VLAN的汇聚方式**

通过汇聚链路时附加的 VLAN 识别信息，这个VLAN 识别信息有 IEEE 802.1Q、Cisco 产品独有的"ISL（Inter Switch Link）"

<br/>

# 准备实验环境

VMware WorkStation 

ubuntu 22.04 虚拟机 2 台，每台主机2块网卡，一块配置为Nat，一块配置为 Host-only 模式。

| 虚拟机       | 网卡一                   | 网卡二                            | 网卡三                        |
| ------------ | ------------------------ | --------------------------------- | ----------------------------- |
| instance_001 | ens33 Nat 172.16.101.130 | ens37  Host-only (192.16.100.10)  | -                             |
| instance_002 | ens33 Nat 172.16.101.131 | ens37  Host-only (192.168.100.11) | -                             |
| instance_003 | ens33 Nat 172.16.101.132 | ens37 Host-only (192.168.100.12)  | ens38  Bridge (192.168.0.106) |

<img src="https://img1.kiosk007.top/static/images/blog/openvswitch_vmware_network.png" alt="openvswitch_vmware_network" style="zoom:33%;clear:both;display:block;margin:auto;" />



<i class="fa-solid fa-network-wired"></i>   安装 OpenvSwtich 。

```bash
sudo apt-get install openvswitch-switch

systemctl start openvswitch-switch.service
systemctl enable openvswitch-switch.service
```

<i class="fa-solid fa-circle-check"></i>    验证 OpenvSwitch

```bash
ovs-vsctl show
4b671582-0aae-4dcb-bb89-763ac8a004cf
    ovs_version: "2.13.8"
```



<i class="fa-solid fa-terminal"></i> 支持的命令

- 添加网桥：`ovs-vsctl add-br br0`
- 列出所有网桥：`ovs-vsctl list-br`
- 判断网桥是否存在：`ovs-vsctl br-exists br0`
- 将物理网卡挂接到网桥：`ovs-vsctl add-port br0 eth0`
- 列出网桥中的所有端口：`ovs-vsctl list-ports br0`
- 列出所有挂接到网卡上的网桥：`ovs-vsctl port-to-br eth0`
- 查看OVS 状态：`ovs-vsctl show`
- 查看OVS 的所有Interface、Port 等：`ovs-vsctl list (Interface|Port)` 或 `ovs-vsctl list Port ens37`
- 删除网桥上已经挂接的网口：`vs-vsctl del-port br0 eth0`
- 删除网桥：`ovs-vsctl del-br br0`

<br/>

# 实验一: 同台宿主机下的虚拟机相互通信

> 本实验均在 instance_001上进行

<img src="https://img1.kiosk007.top/static/images/blog/network1.png" alt="network1" style="zoom:37%;clear:both;display:block;margin:auto;" />

- 创建一个网桥

```bash
$ ovs-vsctl add-br br-in
$ ovs-vsctl show
cd41a1cd-3c5e-40ed-9b96-cc46522b69fd
    Bridge br-in
        Port br-in
            Interface br-in
                type: internal
    ovs_version: "2.13.8"

```

- br 网桥上添加一个端口

```bash
$ ovs-vsctl add-port br-in ens37
$ ovs-vsctl list-ports br-in
ens37
$ ovs-vsctl show 
cd41a1cd-3c5e-40ed-9b96-cc46522b69fd
    Bridge br-in
        Port ens37
            Interface ens37
        Port br-in
            Interface br-in
                type: internal
    ovs_version: "2.13.8"
```

- 网卡配置脚本

```bash
# 启动网卡脚本
touch /root/virtual-machine/if-up
chmod +x /root/virtual-machine/if-up
vim /root/virtual-machine/if-up 
#!/bin/bash
 
bridge=br-in
 
if [ -n $1 ];then
        ip link set $1 up
        sleep 1
        ovs-vsctl add-port $bridge $1
        [ $? -eq 0 ] && exit 0 || exit 1
else
        echo 'Error:no port specified'
        exit 2
fi

# 关闭网卡脚本
touch /root/virtual-machine/if-down
chmod +x /root/virtual-machine/if-down
vim /root/virtual-machine/if-down
#!/bin/bash
 
bridge=br-in
 
if [ -n $1 ];then
        ip link set $1 down
        sleep 1
        ovs-vsctl del-port $bridge $1
        [ $? -eq 0 ] && exit 0 || exit 1
else
        echo 'Error:no port specified'
        exit 2
fi
```



- 启动虚拟机 vm1 vm2

```bash
# 启动vm1
$ qemu-system-x86_64 -enable-kvm -name "vm1" -m 256 -smp 1 \
-drive file=/root/images/cirros-0.4.0-x86_64-nocloud-1.img,media=disk,if=virtio \
-net nic,model=virtio,macaddr=52:54:00:00:00:01 \
-net tap,ifname=vif0.0,script=/root/virtual-machine/if-up,downscript=/root/virtual-machine/if-down \
--nographic

# 启动vm2
$ qemu-system-x86_64 -enable-kvm -name "vm2" -m 256 -smp 1 \
-drive file=/root/images/cirros-0.4.0-x86_64-nocloud-2.img,media=disk,if=virtio \
-net nic,model=virtio,macaddr=52:54:00:00:00:02 \
-net tap,ifname=vif1.0,script=/root/virtual-machine/if-up,downscript=/root/virtual-machine/if-down \
--nographic
```

- 配置vm的ip地址

```bash
# vm1 上的操作
$ ifconfig eth0 10.0.3.1 netmask 255.255.255.0 up

# vm2 上的操作
$ ifconfig eth0 10.0.3.2 netmask 255.255.255.0 up


# 此时vm1就可以 ping通vm2 了
$ ping 10.0.3.2
PING 10.0.3.2 (10.0.3.2): 56 data bytes
64 bytes from 10.0.3.2: seq=0 ttl=64 time=1.072 ms
64 bytes from 10.0.3.2: seq=1 ttl=64 time=0.755 ms

```



<br/>

# 实验二: 划分 VLAN 隔离网络

> 本实验均在 instance_001上进行

<img src="https://img1.kiosk007.top/static/images/blog/network2.png" alt="network2" style="zoom:37%;clear:both;display:block;margin:auto;" />

VLan 的划分有很多方法，有基于Mac地址、有基于IP地址，本次基于 Tag 的方式（给端口加标签的方式）

```bash
$ ovs-vsctl set port vif0.0 tag=10
$ ovs-vsctl set port vif1.0 tag=11
```

此时 vm1 无法再 ping 通 vm2 

```
$ ovs-vsctl list Port vif0.0
_uuid               : 83d88e1f-c234-4666-aba1-ece7b3d278b9
...
fake_bridge         : false
interfaces          : [f07ff627-90e3-4650-a0da-588b722e9407]
lacp                : []
mac                 : []
name                : vif0.0
other_config        : {}
protected           : false
...
tag                 : 10
trunks              : []
vlan_mode           : []

```



- 在 instance_001 上再启动一个 ovs 交换机 br1-in。

```
$ ovs-vsctl add-br br1-in
$ ovs-vsctl list-br
br-in
br1-in
```

- 启动一个新的虚拟机 vm3，并将vm3 连接在 br1-in 上，并配置 ip。

```bash
$ cp if-up if-up1
$ cp ip-down if-down1

# 启动vm3
$ qemu-system-x86_64 -enable-kvm -name "vm3" -m 256 -smp 1 \
-drive file=/root/images/cirros-0.4.0-x86_64-nocloud-3.img,media=disk,if=virtio \
-net nic,model=virtio,macaddr=52:54:00:00:00:03 \
-net tap,ifname=vif2.0,script=/root/virtual-machine/if-up1,downscript=/root/virtual-machine/if-down1 \
--nographic

# vm3 上配置 ip
$ ifconfig eth0 10.0.3.3 netmask 255.255.255.0 up
```

- 此时的 vm3 因为和 vm1、vm2 连在一个交换机上，肯定无法ping通。将两个交换机连起来

```bash
# 创建一个网卡对
$ ip link add s0 type veth peer name s1 
$ ip link set s0 up
$ ip link set s1 up

# 连接两个交换机
$ ovs-vsctl add-port br-in s0
$ ovs-vsctl add-port br1-in s1

# 将vm3的 vif2.0 的vlan设为10
$ ovs-vsctl set port vif2.0 tag=10
```

- 此时 vm3 由于和 vm1 同出于 tag=10 的Vlan，vm3 和 vm2 无法ping 通，但是 vm3 和 vm1 可以ping 通。

```
$ ping 10.0.3.2
PING 10.0.3.2 (10.0.3.2): 56 data bytes
^C
--- 10.0.3.2 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss

$ ping 10.0.3.1
PING 10.0.3.1 (10.0.3.1): 56 data bytes
64 bytes from 10.0.3.1: seq=0 ttl=64 time=11.117 ms
64 bytes from 10.0.3.1: seq=1 ttl=64 time=1.852 ms

```

> 实验结束，删除 VLAN tag 配置可使用
>
> ovs-vsctl remove Port vif0.0 tag 10



<br/>

# 实验三: 自动分配IP地址

> 本实验在 instance_001 上进行，并且清空之前 instance_001 上的网桥相关信息，重新开始



为了让启动的虚拟机自动获得 IP 地址，我们使用 dnsmasq 搭建一个 DHCP 服务。

- 使用网络空间创建一个 Namespace

```
$ ip netns add r0   #添加namespace
$ ip link add sif0 type veth peer name rif0 #添加一对网卡
$ ip link set sif0 up
$ ip link set rif0 up
$ ip link set rif0 netns r0  #把网卡rif0给r0
$ ovs-vsctl add-port ovs-br1 sif0 #把网卡sif0给br-in
```

- 激活r0上的网卡

```bash
$ ip netns exec r0 ip link set rif0 up
$ ip netns exec r0 ip addr add 10.0.0.1/24 dev rif0
```

- 安装dnsmasq并配置DHCP服务

```bash
$ sudo apt install dnsmasq
# 配置DHCP
$ ip netns exec r0 dnsmasq -F 10.0.0.200,10.0.0.230,86400 -i rif0
$ ip netns exec r0 ss -unl
State               Recv-Q              Send-Q                           Local Address:Port                            Peer Address:Port              Process              
UNCONN              0                   0                                      0.0.0.0:53                                   0.0.0.0:*                                      
UNCONN              0                   0                                      0.0.0.0:67                                   0.0.0.0:*                                      
UNCONN              0                   0                                         [::]:53                                      [::]:*            
```

- 启动一个虚拟机，并查看 IP地址和路由信息

```shell
# 启动 vm4 
$ qemu-system-x86_64 -enable-kvm -name "vm4" -m 256 -smp 1 \
-drive file=/root/images/cirros-0.4.0-x86_64-nocloud.img,media=disk,if=virtio \
-net nic,model=virtio,macaddr=52:54:00:00:00:04 \
-net tap,ifname=vif4.0,script=/root/virtual-machine/if-up,downscript=/root/virtual-machine/if-down \
--nographic

# 查看信息
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 52:54:00:00:00:04 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.211/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe00:4/64 scope link 
       valid_lft forever preferred_lft forever
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.1        0.0.0.0         UG    0      0        0 eth0
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0


```



<br/>

# 实验四: 跨宿主机下虚拟机通信

> 本实验在 instance_001 和 instance_002 上进行

- 创建网桥

```bash
# 节点1添加网桥(instance_001 上执行)
$ ovs-vsctl add-br ovs-br1

# 节点2添加网桥 (instance_002 上执行)
$ ovs-vsctl add-br ovs-br2 
```

- 配置 gre 隧道

```bash
# 添加 gre （instance_001 上执行）
$ ovs-vsctl add-port ovs-br1 gre1
$ ovs-vsctl set interface gre1 type=gre options:remote_ip=192.168.100.11

# 添加 gre （instance_002 上执行）
$ ovs-vsctl add-port ovs-br2 gre2
$ ovs-vsctl set interface gre2 type=gre options:remote_ip=192.168.100.10

# 查看 002
$ ovs-vsctl show
4b671582-0aae-4dcb-bb89-763ac8a004cf
    Bridge ovs-br2
        Port gre2
            Interface gre2
                type: gre
                options: {remote_ip="192.168.100.10"}
        Port ovs-br2
            Interface ovs-br2
                type: internal
    ovs_version: "2.13.8"

```

- 启动虚拟机，自动从 001 上的DHCP 服务获取IP地址

```bash
# 启动虚拟机 (instance_002 上执行)
$ qemu-system-x86_64 -enable-kvm -name "vm5" -m 256 -smp 1 \
-drive file=/root/images/cirros-0.4.0-x86_64-nocloud.img,media=disk,if=virtio \
-net nic,model=virtio,macaddr=52:54:00:00:00:05 \
-net tap,ifname=vif5.0,script=/root/virtual-machine/if-up,downscript=/root/virtual-machine/if-down \
--nographic

# vm5 的虚拟机内执行
ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 52:54:00:00:00:05 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.212/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe00:5/64 scope link 
       valid_lft forever preferred_lft forever
# ping 10.0.0.211
PING 10.0.0.211 (10.0.0.211): 56 data bytes
64 bytes from 10.0.0.211: seq=0 ttl=64 time=5.710 ms
64 bytes from 10.0.0.211: seq=1 ttl=64 time=1.604 ms
^C
--- 10.0.0.211 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss

```

instance_001 的 ens37 网卡上抓报可以看到，其使用的GRE 隧道技术，IP地址之上又封装了一层IP地址。（*截图中的 192.168.0.10 可以认为是 192.168.100.10，后面换过网段*）

<img src="https://img1.kiosk007.top/static/images/blog/network3.png" alt="network3" style="zoom:67%;" />

<br/>

# 实验五: 跨宿主机的 VLAN 划分

> 本实验在 instance_001 和 instance_002 上进行

<img src="https://img1.kiosk007.top/static/images/blog/network5.png" alt="network4" style="zoom:40%;clear:both;display:block;margin:auto;" />

- 再启动两个虚拟机

```bash
# instance_001 上再启动一个虚拟机
$ qemu-system-x86_64 -enable-kvm -name "vm6" -m 256 -smp 1 \
-drive file=/root/images/cirros-0.4.0-x86_64-nocloud-1.img,media=disk,if=virtio \
-net nic,model=virtio,macaddr=52:54:00:00:00:06 \
-net tap,ifname=vif6.0,script=/root/virtual-machine/if-up,downscript=/root/virtual-machine/if-down \
--nographic

# instance_002 上再启动一个虚拟机
$ qemu-system-x86_64 -enable-kvm -name "vm7" -m 256 -smp 1 \
-drive file=/root/images/cirros-0.4.0-x86_64-nocloud-1.img,media=disk,if=virtio \
-net nic,model=virtio,macaddr=52:54:00:00:00:07 \
-net tap,ifname=vif7.0,script=/root/virtual-machine/if-up,downscript=/root/virtual-machine/if-down \
--nographic

```

- 划分VLAN

```bash
# instance_001 上对 vm4 和 vm6 划分
ovs-vsctl set port vif4.0 tag=10
ovs-vsctl set port vif6.0 tag=11

# instance_002 上对 vm5 和 vm7 划分
ovs-vsctl set port vif5.0 tag=10
ovs-vsctl set port vif7.0 tag=11
```

此时 vm7 (10.0.0.214) 可以ping 通 vm6 (10.0.0.213) 但是无法ping通 vm4 (10.0.0.211)



<br/>

# 实验六: 虚拟机访问外网（OpenStack 模式）

> 本实验在 instance_001 、 instance_002 和 instance_003 上进行（001 、002 取消所有vlan tag\删除001 和 002 中间的 gre）
>
> $ ovs-vsctl remove Port vif4.0 tag 10 -- remove Port vif6.0 tag 11
>
> $ ovs-vsctl del-port ovs-br1 gre1

<img src="https://img1.kiosk007.top/static/images/blog/network6.png" alt="network6" style="zoom:107%;" />

- instance_003 创建 网桥

```bash
$ ovs-vsctl add-br ovs-br3
```

- 配置网络命名空间

```bash
# instance_003 上操作
# 添加一个 netns
ip netns add r0
ip link add sin0 type veth peer name rin0
ip link add sex0 type veth peer name rex0
ip link set sin0 up
ip link set sex0 up
ip link set rin0 netns r0
ip link set rex0 netns r0

# 将 rin0 设置一个ip
ip netns exec r0 ip addr add 10.0.0.2/24 dev rin0
ip netns exec r0 ifconfig rin0 up

# 将 sin0 添加到 ovs-br3
ovs-vsctl add-port ovs-br3 sin0
```

- 连通 instance_001 和 instance_003

```bash
# instance_001 上执行
ovs-vsctl add-port ovs-br1 gre3
ovs-vsctl set interface gre3 type=gre options:remote_ip=192.168.100.12

# instance_002 上执行
ovs-vsctl add-port ovs-br2 gre4
ovs-vsctl set interface gre4 type=gre options:remote_ip=192.168.100.12

# instance_003 上执行
ovs-vsctl add-port ovs-br3 gre5
ovs-vsctl set interface gre5 type=gre options:remote_ip=192.168.100.10
ovs-vsctl add-port ovs-br3 gre6
ovs-vsctl set interface gre6 type=gre options:remote_ip=192.168.100.11

# 此时 r0 已经可以和众多虚拟机 ping 通
$ ip netns exec r0 ping 10.0.0.211
PING 10.0.0.211 (10.0.0.211) 56(84) bytes of data.
64 bytes from 10.0.0.211: icmp_seq=1 ttl=64 time=3.91 ms

```

- instance_003 上创建网桥，并桥接 物理网卡 ens38

```bash
# 创建网桥
brctl addbr br-ext
# 绑定 ens38
brctl addif br-ext ens38
# 删除 ens38 上的 IP 地址
ip addr del dev ens38 172.16.101.133/24
# 将IP地址配置到网桥之上
ifconfig br-ext 172.16.101.133/24 up
# 重新加入默认网关
route add default gw 172.16.101.2

# 将 sex0 也加入 br-ext
brctl addif br-ext sex0
ip netns exec r0 ifconfig rex0 192.168.0.200/24 up
```

- 虚拟机上配置默认路由

```bash
# 随便选一台vm,这里选择 instance_001 上的 vm1
route add default gw 10.0.0.2

$ ping 172.16.101.150
PING 172.16.101.150 (172.16.101.150): 56 data bytes
64 bytes from 172.16.101.150: seq=0 ttl=64 time=2.931 ms

```

- 配置 instance_003 上的 r0 iptables

```bash
# 开启ip转发
ip netns exec r0 sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.ip_forward=1

# 配置 r0 的 route 网关
ip netns exec r0 route add default 192.168.0.106 

ip netns exec r0 iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -j SNAT --to-source 192.168.0.106
ip netns exec r0 iptables -t nat -A PREROUTING -d 192.168.0.106 -j DNAT --to-destination 10.0.0.2
```

- 在 vm 4 上 ping 外网

```bash
ping 114.114.114.114
PING 114.114.114.114 (114.114.114.114): 56 data bytes
64 bytes from 114.114.114.114: seq=0 ttl=89 time=24.016 ms
```





<br>

# 更多实战

这里看到知乎上一位博主的文章，对 OVS 的总结还是算比较全的。

- [Open vSwitch 入门实践（1）简介](https://zhuanlan.zhihu.com/p/336487371)
- [Open vSwitch 入门实践（2）使用OVS构建隔离网络](https://zhuanlan.zhihu.com/p/336692269)

- [Open vSwitch 入门实践（3）使用OVS构建分布式隔离网络](https://zhuanlan.zhihu.com/p/337210249)
- [Open vSwitch 入门实践（4）使用OVS配置端口镜像](https://zhuanlan.zhihu.com/p/337436733)

<br/>

**更多参考**

- [OVS初级教程](https://www.sdnlab.com/sdn-guide/14747.html)

