# Linux容器网络-vxlan


 `Linux` 是支持 `VXLAN` 的，我们可以使用 Linux 搭建基于 `VXLAN` 的 overlay 网络，下面的文章将记录相关的一些内容。



<!--more-->

# VXLAN 技术

## VXLAN 技术概述

- 定义：VXLAN（Virtual Extensible Local Area Network）即虚拟可扩展局域网，是一种网络虚拟化技术，标准文档在 [RFC7348](https://tools.ietf.org/html/rfc7348)，他通过现有的三层网络之上构建二层网络，将二层网络的范围扩展到三层网络之上，实现跨越物理网络边界的虚拟局域网通信。
- 诞生背景：
  - **VLAN ID 的数量限制**。随着数据中心规模的不断扩大，传统 VLAN 技术再扩展性、灵活性等方面面临着诸多的限制，比如VLAN的标签空间 ( 4096个 ) 有限，难以满足大规模多租户环境下的网络隔离和灵活部署的需求。
  - **交换机MAC 地址表限制**。越来越多的数据中心（尤其是公有云服务）需要创建大量的 VM，VM 的数量与原有的物理机器相比呈指数级的增长，而交换机的内存有限，存不下如此多的MAC地址，VXLAN 借助 `VTEP` 将二层以太网帧封装在 `UDP` 中，一个 `VTEP` 就可以被一个物理机上的所有 VM 共用，交换机的角度来看，没有VM，他看到的全部是所有的物理机。
  - **虚拟机或容器迁移范围受限**。VLAN 如果和 物理网络融合在一起，不存在Overlay 网络，就会存在虚拟网络不能打破物理网络的限制，比如 VLAN 100 部署虚拟机，必须在 VLAN 100 的物理机上部署。（VLAN 为了避免广播域过大）。因为 VXLAN 是 Overlay 网络，所以就无所谓部署在哪台物理设备上了。



<br/>

简单来讲，`VXLAN` 是在底层物理网络（underlay）之上使用隧道技术，借助 `UDP` 层构建的 Overlay 的逻辑网络，使逻辑网络与物理网络解耦，实现灵活的组网需求。它对原有的网络架构几乎没有影响，不需要对原网络做任何改动，即可架设一层新的网络。也正是因为这个特性，很多 CNI 插件（Kubernetes 集群中的容器网络接口，这个大家应该都知道了吧，如果你不知道，现在你知道了）才会选择 `VXLAN` 作为通信网络。

`VXLAN` 不仅支持一对一，也支持一对多，一个 `VXLAN` 设备能通过像网桥一样的学习方式学习到其他对端的 IP 地址，还可以直接配置静态转发表。



<img src="https://img1.kiosk007.top/static/images/blog/20241222230723-wireshark-vxlan-003.png" />

<br/>

## VXLAN 技术原理

vxlan 这类隧道网络的一个特点是对原有的网络架构影响小，原来的网络不需要做任何改动，在原来网络基础上架设一层新的网络。

vxlan 自然会引入一些新的概念。

- **VTEP（VXLAN Tunnel Endpoints, VXLAN 隧道端点）**：vxlan 网络的边缘设备，用来进行 vxlan 报文的处理（封包和解包）。vtep 可以是网络设备（比如交换机），也可以是一台机器（比如虚拟化集群中的宿主机）
- **VNI（VXLAN Network Identifier, VXLAN 网络标识符）**：VNI 是每个 vxlan 的标识，是个 24 位整数，一共有 2^24 = 16,777,216（一千多万），一般每个 VNI 对应一个租户，也就是说使用 vxlan 搭建的公有云可以理论上可以支撑千万级别的租户
- **Tunnel (VXLAN 隧道)**：隧道是一个逻辑上的概念，在 vxlan 模型中并没有具体的物理实体想对应。隧道可以看做是一种虚拟通道，vxlan 通信双方（图中的虚拟机）认为自己是在直接通信，并不知道底层网络的存在。从整体来说，每个 vxlan 网络像是为通信的虚拟机搭建了一个单独的通信通道，也就是隧道

<br/>

<img src="https://img1.kiosk007.top/static/images/blog/20241221170405-vxlan_packet_header.png" alt="vxlan_packet_header">

<br/>

封装与解封装：

- VXLAN 的核心原理是对原始以太网帧进行封装，再源端（数据发送端），VXLAN 会在原始以太网帧外面添加 VXLAN头和UDP头、IP头和MAC头，将其封装成一个可以在三层网络中传输的数据包。VXLAN头中包含了关键的 VNI 信息，用于标识该数据包所属的虚拟网络
- 在目的端（数据包接收端），则会进行相反的解封装操作，去除添加的头部信息，还原出原始的以太网帧，然后将其发送到目标设备上，例如，一个容器发送的数据需要跨越不同的物理服务器到达另一个容器，数据在离开开源容器所在的服务器时会被封装，经过三层网络传输之后，再在目的服务器上解封装并发送到目标容器上。

<br/>

## VXLAN 网络架构



- 单播模式和多播模式：
  - 单播模式：VTEP 之间通过预先配置的单播路由来建立隧道并传输数据，当一个 VTEP 需要向另一个 VTEP 发送数据包时，他会直接将封装后的数据包发送到目标 VTEP 的IP地址，这种模式适合网络拓扑相对简单，VTEP 数量相对较少且易于配置单播路由的场景。
  - 多播模式：利用多播协议来传播VXLAN数据包，当一个 VTEP 需要发送数据包时，他会将数据包发送到一个多播组，同一多播组中的其他VTEP可以接收到该数据包，多播模式在大规模的VXLAN网络中可以减少配置工作量，但需要网络支持多播模式并且对多播组的管理要求较高。
- VXLAN 网络中的设备：
  - 除了 VTEP 外，VXLAN 网络中还可能包含其他设备，例如：网关设备用于连接 VXLAN 网络和外部网络，实现VXLAN 内部网络和外部网络之间的通信，在容器网络环境中，容器运行时或者容器编排平台（如 kubernetes） 可以与 VTEP 集成，实现容器之间的跨越物理主机之间的通信。

<br/>



## Linux 下的 VXLAN

`Linux` 对 VXLAN 协议的支持时间并不久，`2012` 年 Stephen Hemminger 才把相关的工作合并到 kernel 中，并最终出现在 `kernel 3.7.0` 版本。为了稳定性和很多的功能，可能会看到某些软件推荐在 `3.9.0` 或者 `3.10.0` 以后版本的 kernel 上使用 VXLAN。

到了 `kernel 3.12` 版本，Linux 对 VXLAN 的支持已经完备，支持单播和组播，IPv4 和 IPv6。利用 `man` 查看 ip 的 `link` 子命令，可以查看是否有 VXLAN type：



- **管理 VXLAN 接口**

1. 创建点对点的 VXLAN 接口:

```bash
$ ip link add vxlan0 type vxlan id 4100 remote 192.168.1.101 local 192.168.1.100 dstport 4789 dev eth0
```

其中 `id` 为 VNI，`remote` 为远端主机的 IP，`local` 为你本地主机的 IP，`dev` 代表 VXLAN 数据从哪个接口传输。

在 VXLAN 中，**一般将 VXLAN 接口（本例中即 vxlan0）叫做 VTEP**。

2. 创建多播模式的VXLAN接口：

```bash
$ ip link add vxlan0 type vxlan id 4100 group 224.1.1.1 dstport 4789 dev eth0
```

多播组主要通过 `ARP` 泛洪来学习 `MAC` 地址，即在 VXLAN 子网内广播 `ARP` 请求，然后对应节点进行响应。`group` 指定多播组的地址。

3. 查看 VXLAN 接口详细信息

```bash
$ ip -d link show vxlan0
```

<br/>

- **FDB 表**

`FDB`（Forwarding Database entry，即转发表）是 Linux 网桥维护的一个二层转发表，用于保存远端虚拟机/容器的 MAC地址，远端 VTEP IP，以及 VNI 的映射关系，可以通过 `bridge fdb` 命令来对 `FDB` 表进行操作：

1. 条目添加：

```bash
$ bridge fdb add <remote_host_mac> dev <vxlan_interface> dst <remote_host_ip>
```

2. 条目删除：

```bash
$ bridge fdb del <remote_host_mac> dev <vxlan_interface>
```

3. 条目更新：

```bash
$ bridge fdb del <remote_host_mac> dev <vxlan_interface>
```

4. 条目查询:

```bash
$ bridge fdb show
```





<br/>

# VXLAN 网络搭建

| 信息      | 节点A             | 节点B             |
| --------- | ----------------- | ----------------- |
| 宿主机    | 192.168.120.10    | 192.168.120.20    |
| mac地址   | 52:54:00:44:8b:86 | 52:54:00:3f:e6:43 |
| VXLAN IP  | 10.0.0.1          | 10.0.1.1          |
| VXLAN Mac | da:af:e1:9b:30:73 | da:af:e1:9b:30:74 |
| 容器网段  | 10.0.0.0/24       | 10.0.1.0/24       |

<br/>

## 容器环境构建

这里使用 Linux net namespace 构建容器。

在节点A上进行创建，节点B执行的流程一致。这里就只展示节点A上的操作

```bash
# 创建容器 net namespace
ip netns add net1
ip netns exec net1 ip link set lo up

# 创建 veth
ip link add type veth

# 创建 veth0
ip link set veth0 netns net1
ip netns exec net1 ip link set veth0 name eth0
ip netns exec net1 ip addr add 10.0.0.10 dev eth0
ip netns exec net1 ip link set eth0 up

# 设置veth1
# 启用代理 ARP 功能。代理 ARP 允许一台主机代替其他主机响应 ARP 请求
# 有助于在网络中隐藏实际的主机地址或者实现更灵活的网络地址转换等功能
# 在这里是为了辅助容器与外部网络更好地通信。
echo 1 > /proc/sys/net/ipv4/conf/veth1/proxy_arp
ip link set veth1 address ee:ee:ee:ee:ee:ee
ip link set veth1 up

# 设置容器内路由
# 在net1网络命名空间内，为通过eth0接口访问169.254.1.1这个链路本地地址添加一条路由
# 限定其为链路本地范围（scope link），意味着这个地址的通信仅在本地链路内进行，
# 不需要经过网关转发，便于在本地范围内进行一些特定的通信操作。
ip netns exec net1 ip route add 169.254.1.1 dev eth0 scope link
ip netns exec net1 ip route add default via 169.254.1.1 dev eth0

# 设置主机路由
# 在宿主机上添加一条路由，使得宿主机知道要访问容器内10.0.0.10这个 IP 地址时
# 通过veth1接口在链路本地范围内进行转发
# 保证宿主机与容器之间可以基于这个 IP 地址进行通信。
ip route add 10.0.0.10 dev veth1 scope link

# 测试容器内网络 到 宿主机 的联通性
ip netns exec eth1 ping 192.168.120.10

```

<br/>

## vxlan 环境搭建

vxlan 设置内容，其中节点A上执行动作如下：

```bash
# 创建 vxlan 设备，指明该设备的出口为enp1s0,端口 4789, VNI 为 4096
ip link add vxlan0 type vxlan id 4096 dev enp1s0 dstport 4789
# 设置 vxlan 设备
ip addr add 10.0.0.1/32 dev vxlan0
# 启动 vxlan
ip link set vxlan0 up
# 添加 fdb 信息，ip 为远端的 enps1s0 ip, mac 为远端的 vxlan mac 
# (这个时候将上述操作在 节点B 上也操作一边，就可以得到 B xlan 的Mac地址)
bridge fdb add da:af:e1:9b:30:74 dev vxlan0 dst 192.168.120.20
# 设置远端 POD 需要经过 vxlan0 传输
ip route add 10.0.1.0/24 via 10.0.1.1 dev vxlan0 onlink
# 设置 vxlan mac ip 的arp信息
arp -s 10.0.1.1 da:af:e1:9b:30:74 -i vxlan0
```

<br/>

节点B上执行的动作如下：

```bash
# 创建 vxlan 设备，指明该设备的出口为enp1s0,端口 4789, VNI 为 4096
ip link add vxlan0 type vxlan id 4096 dev enp1s0 dstport 4789
# 设置 vxlan 设备
ip addr add 10.0.1.1/32 dev vxlan0
# 启动 vxlan
ip link set vxlan0 up
# 添加 fdb 信息，ip 为远端的 enps1s0 ip, mac 为远端的 vxlan mac 
# (这个时候将上述操作在 节点B 上也操作一边，就可以得到 B xlan 的Mac地址)
bridge fdb add da:af:e1:9b:30:73 dev vxlan0 dst 192.168.120.10
# 设置远端 POD 需要经过 vxlan0 传输
ip route add 10.0.0.0/24 via 10.0.0.1 dev vxlan0 onlink
# 设置 vxlan mac ip 的arp信息
arp -s 10.0.0.1 da:af:e1:9b:30:73 -i vxlan0
```

<br/>

测试2个容器之间的联通性 。`ip netns exec net1 ping 10.0.1.10`

```bash
root@instance-vm00:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:44:8b:86 brd ff:ff:ff:ff:ff:ff
    inet 192.168.120.10/24 brd 192.168.120.255 scope global enp1s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe44:8b86/64 scope link 
       valid_lft forever preferred_lft forever
...
6: veth1@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns net1
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
7: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether da:af:e1:9b:30:73 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/32 scope global vxlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::d8af:e1ff:fe9b:3074/64 scope link 
       valid_lft forever preferred_lft forever


root@instance-vm00:~# ip netns exec net1 ping 10.0.1.10
PING 10.0.1.10 (10.0.1.10) 56(84) bytes of data.
64 bytes from 10.0.1.10: icmp_seq=1 ttl=62 time=0.528 ms
64 bytes from 10.0.1.10: icmp_seq=2 ttl=62 time=0.351 ms
64 bytes from 10.0.1.10: icmp_seq=3 ttl=62 time=1.11 ms
64 bytes from 10.0.1.10: icmp_seq=4 ttl=62 time=1.31 ms

--- 10.0.1.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3036ms
rtt min/avg/max/mdev = 0.351/0.824/1.314/0.396 ms
```

<br/>

通过抓包可以看到其中的内容

`ssh root@192.168.120.10 tcpdump not port 22 -w - | wireshark -k -S -i -`

- 从 veth1 上看到的抓包

<img src="https://img1.kiosk007.top/static/images/blog/20241221191647-wireshark-vxlan000.png" />

> 注意看 Mac地址，原 Mac地址是 容器内 veth0 的 Mac ，目的 Mac 地址是 veht1 的 Mac “ee:ee:ee:ee:ee:ee”

<br/>

- 从 vxlan0 上看到的抓包

<img src="https://img1.kiosk007.top/static/images/blog/20241221195429-wireshark-vxlan002.png" />

> 从 Mac 地址上看，是源地址 是 主机A vxlan 的Mac 地址，目的地址是 主机B vxlan 的 Mac地址 

<br/>

- 从 enp1s0 上看到的抓包

<img src="https://img1.kiosk007.top/static/images/blog/20241221192002-wireshark-vxlan-001.png" />

> 可以看到有 vxlan 的封装, 2层的封装看是 主机A 的 Mac 地址，UDP 之上是 vxlan 的封装，vxlan 层有封装 是主机A 和 主机B vxlan 的IP地址



<br/>

# 网络流量分析

首先从 节点A 上可以看到

```bash
# arp
Address                  HWtype  HWaddress           Flags Mask            Iface
_gateway                 ether   52:54:00:f4:e3:7f   C                     enp1s0
192.168.120.20           ether   52:54:00:3f:e6:43   C                     enp1s0
10.0.1.1                 ether   da:af:e1:9b:30:74   CM                    vxlan0
10.0.0.10                ether   b2:37:3c:94:5b:59   C                     veth1
```

他不仅学习到 本机上 容器的 mac 地址，也学习到跨主机的 容器上的mac地址。

- **从主机A veth1 跳转到主机的 vxlan**

1. 流量从 veth1 出来是 2 层Packet。

2. Linux 协议栈从 veth1 中取出 packet，释放其 Mac 层地址转交给路由器(3层)

3. 路由层通过下一跳地址，在这期间， packet 只有 mac 地址发生了变化，其他不变。
4. packet 到达 vxlan0 ，源地址 10.0.0.10 ，目的地址 10.0.1.10 ，源和目的 Mac 分别是 发送端 和 接收端 vxlan0 Mac 地址。

<br/>

- **从 vxlan0 跳转到主机的 enp1s0**

这里有一层封装，将应用层数据封装在 vxlan 里

- **从主机A enp1s0 跳转到 主机 vxlan0**

主机的 vxlan0 直接监听4789端口，eth0 获取到 packet 之后，会直接发给 vxlan0 

<br/>

# 拓展

目前的 vxlan 常见的 有 点对点通信、VXLAN + Bridge 、多播模式的 VXLAN ,

- **点对点通信**：点对点 `VXLAN` 网络通信双方只有一个 `VTEP`，且只有一个通信实体，而在实际生产中，每台主机上都有几十台甚至上百台虚拟机或容器需要通信。所以很难大范围使用。
- **VXLAN + Bridge**：Linux Bridge 就可以将多块虚拟网卡连接起来，因此可以选择使用 `Bridge` 将多个虚拟机或容器放到同一个 VXLAN 网络中
- **多播模式**：上面两种模式只能点对点连接，也就是说同一个 VXLAN 网络中只能有两个节点，这怎么能忍。。。使用多播，把网络中的某些节点组成一个虚拟的整体。事先知道 `MAC` 地址和 `VTEP IP` 信息，直接把 `ARP` 和 `FDB` 信息告诉发送方 VTEP。一般是通过外部的分布式控制中心来收集这些信息，收集到的信息会分发给同一个 VXLAN 网络的所有节点。

<br/>

和上述实验有明显区别的是 FDB 表项的内容：

```bash
# 本实验
$ bridge fdb show 
da:af:e1:9b:30:74 dev vxlan0 dst 192.168.120.20 self permanent

# 如果是多播模式
00:00:00:00:00:00 dev vxlan0 dst 224.1.1.1 self permanent
```

`dst` 字段的值变成了多播地址 `224.1.1.1`，而不是之前对方的 VTEP 地址，VTEP 会通过 [IGMP（Internet Group Management Protocol）](https://zh.wikipedia.org/wiki/因特网组管理协议) 加入同一个多播组 `224.1.1.1`。

<br/>

来分析下多播模式下 `VXLAN` 通信的全过程：

1. 发送 ping 报文到 `10.0.1.10`，查看路由表，报文会从 `vxlan0` 发出去。
2. 内核发现 `vxlan0` 的 IP 是 `10.0.1.0/24`，和目的 IP 在同一个网段，所以在同一个局域网，需要知道对方的 MAC 地址，因此会发送 `ARP` 报文查询。
3. `ARP` 报文源 MAC 地址为 `vxlan0` 的 MAC 地址，目的 MAC 地址为全 1 的广播地址（ff:ff:ff:ff:ff:ff）。
4. `VXLAN` 根据配置（VNI 42）添加上头部。
5. **到这一步就和之前不一样了**，由于不知道对端 `VTEP` 在哪台主机，根据多播配置，`VTEP` 会往多播地址 `224.1.1.1` 发送多播报文。
6. 多播组中的所有主机都会收到这个报文，内核发现是 `VXLAN` 报文，就会根据 `VNI` 发送给相应的 `VTEP`。
7. 收到报文的所有主机的 `VTEP` 会去掉 `VXLAN` 的头部，取出真正的 `ARP` 请求报文。同时，`VTEP` 会记录源 `MAC` 地址和 IP 地址信息到 `FDB` 表中，这便是一次学习过程。如果发现 `ARP` 不是发送给自己的，就直接丢弃；如果是发送给自己的，则生成 `ARP` 应答报文。
8. 后面的步骤就和上面的实验相同了。

<br/>

整个通信过程和之前比较类似，只是 `Underlay` 采用组播的方式发送报文，对于多节点的 `VXLAN` 网络来说比较简单高效。但多播也是有它的问题的，并不是所有网络设备都支持多播（比如公有云），再加上多播方式带来的报文浪费，在实际生成中很少被采用。

一般来说，公有云厂商都是通过 中心分布式控制中心来自动发现 `VTEP` 和 `MAC` 地址。



### 参考

- [VXLAN 基础教程：在 Linux 上配置 VXLAN 网络](https://www.cnblogs.com/ryanyangcs/p/12742922.html)

- [VXLAN 基础教程：VXLAN 协议原理介绍](https://icloudnative.io/posts/vxlan-protocol-introduction)

