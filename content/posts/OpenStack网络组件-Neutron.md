---
title: OpenStack网络组件-Neutron
author: kiosk
draft: true
tags: 
  - openstack
categories:
  - devops
date: 2022-09-06 11:23:00
---

Neutron 的设计目标是实现“网络即服务（Networking as a Service）”。为了达到这一目标，在设计上遵循了基于 SDN 实现网络虚拟化的原则，在实现上充分利用了 Linux 系统上的各种网络相关的技术。

SDN 模式服务— NeutronSDN( 软件定义网络 ), 通过使用它，网络管理员和云计算操作员可以通过程序来动态定义虚拟网络设备。Openstack 网络中的 SDN 组件就是 Quantum.但因为版权问题而改名为Neutron 。

<!--more-->

# 基本概念

## Network

Network 是一个隔离的二层广播域。Neutron 支持多种类型的 Network，包含 Local、Flat、VLAN、Vxlan 和 GRE 。

1. **Local**： 网络与其他物理机节点网络不连通。
2. **Flat**：无 Vlan tagging 的网络，Flat 网络下的 instance 可以跨物理节点通信
3. **VLAN**：具有 802.1q tagging 的网络，是一个二层的广播域，同一 vlan 中的instance 可以通信。不同 vlan 的通信只能通过 router 进行。vlan 可以跨节点通信。
4. **Vxlan**：Vxlan是基于隧道技术的 overlay 网络，其通过一个 segmentation ID （VNI）与其他 Vxlan 网络区分，Vxlan 中数据包会通过 VNI 封装成 UDP 包进行传输，因为二层的包通过封装在三层传输，能够克服 VLAN 和物理网络基础设施的限制。
5. **GRE**：与 Vxlan 类似的一种 overlay 网络。主要区别在于使用 IP 包而非 UDP 进行封装。

<br>

## Subnet

是一个 IPv4 或者 IPv6 地址段。Instance 的 IP 从 Subnet 中分配。每个 Subnet 需要定义 IP 地址的范围和掩码。

同一个Network 的 Subnet 可以是不同的 IP 段，但CIDR不能重叠。

```bash
# 有效配置
networkA 
subnetA-a：10.0.1.0/24  10.0.1.1 - 10.0.1.50
subnetB-b：10.0.2.0/24  10.0.2.1 - 10.0.2.50

# 无效配置
networkB
subnetB-a：10.0.1.0/24  10.0.1.1 - 10.0.1.50
subnetB-b：10.0.1.0/24  10.0.1.51- 10.0.1.100
```

<br/>

## Port

虚拟交换机上的一个端口，Port 上定义了Mac地址和IP地址，当 instance 的虚拟网卡 VIF（Virtual Interface）绑定到 Port 时，Port 会将 Mac 和 IP 分配给 VIF

<br/>

## 虚拟路由器（Virtual router）

一个 Virtual router 提供不同网段之间的 IP 包路由功能，本质是一个 Namespace，每个Namespace中维护一张路由表。由 Nuetron L3 agent 负责管理。

<br/>

# Neutron 组成

Neutron 本身只是负责网络抽象的 API，充分采用了**插件化**的设计，只要可以实现 Neutron 网络API就可以，非常灵活。



## 插件

插件总体分为两类：**Core Plugin** 和 **Service Plugin** 。

<img src="https://img1.kiosk007.top/static/images/blog/neutron1.png" alt="neutron1" style="zoom:43%;display:block;clear:both;margin:auto;" />

- Core Plugin：实现了 Core Plugin API，在数据库中维护 network, subnet 和 port 的状态，并负责调用相应的 agent 在 network provider 上执行相关操作，比如创建 network。

- Service Plugin：实现了 Extension Plugin API，在数据库中维护 router, load balance, security group 等资源的状态，并负责调用相应的 agent 在 network provider 上执行相关操作，比如创建 router。

<br/>

## ML2 Core Plugin

Moduler Layer 2 （ML2）：是 Neutron 在 Havana 版本实现的一个新的 core plugin，用于替代原有的 linux bridge plugin 和 openvswitch plugin。它提供了一个框架，允许在 OpenStack 的网络中使用多种 Layer 2 网络技术，不同的节点可以使用不同的网络实现机制。

> Core Plugin/Agent 负责管理核心的实体， net、subnet 和 port。

<br/>

ML2 对二层网络进行了抽象和建模，引入了 **type driver** 和 **mechansim driver**。这两类 driver 解耦了 Neutron 所支持的所有网络类型 （type）与访问这些网络类型的机制（mechansim），使得ML2具有非常好的弹性，易于扩展，能够灵活支持多种 type 和 mechanism 。

- **Type Driver**：Neutron 支持的每一种网络类型都有一个对应的 ML2 type driver。type driver 负责维护网络类型的状态，执行验证，创建网络等。ML2 支持的网络类型包括 local、flat、vlan、vxlan、gre。
- **Mechansim Driver**：Neutron 支持的每一种网络机制都有一个对应的 ML2 Mechansim Driver，Mechanism Driver 负责获取由 type driver 维护的网络状态，并确保在相应的网络设备（虚拟或物理）上正确实现这些状态。
  - Agent-Based 类型：包括 Linux Bridge、OpenvSwitch 等
  - Controller-Based 类型：包括 OpenDaylight、VMWare NSX等
  - 基于物理交换机：包括 Cisco Nexus、Arista、Mellanox 等

<img src="https://img1.kiosk007.top/static/images/blog/neutron3.png" alt="neutron3" style="zoom:43%;display:block;clear:both;margin:auto;" />

<br/>

## Service Plugin / Agent

Service Plugin 及其 Agent 提供更丰富的扩展能力，包括Router、Load Balance、Firewall 等等。其负责在 agent 上的 network provider  执行相关的操作，比如创建路由。

1. DHCP：dhcp agent 通过 dnsmasq 为 instance 提供 DHCP 服务

2. Routing：L3 agent 可以为 Project（租户）创建 router，提供 subnet 之间的路由服务。而路由功能默认由 iptables 实现。

3. Firewall：L3 agent 可以在router 上配置防火墙策略，提供网络安全防护，另外也提供 Security Group，其本质也是 iptables 实现。

4. Load Balance：Neutron 通过 HAProxy 为 project 中的多个 instance 提供 Load Balance 服务。

   <img src="https://img1.kiosk007.top/static/images/blog/neutron4.png" alt="neutron4" style="zoom:43%;display:block;clear:both;margin:auto;" />

<br/>

# 网络模式

在部署openstack 网络组件 neutron 时会看到有两种网络方案，一种是 [provider 模式](https://docs.openstack.org/neutron/yoga/install/controller-install-option1-ubuntu.html)，一种是 [self-service 模式](https://docs.openstack.org/neutron/yoga/install/controller-install-option2-ubuntu.html) 。

在[官网](https://docs.openstack.org/install-guide/overview.html#networking)的介绍中有两张图，可以看到 self-service 模式主要是在控制节点（旧版本的网络节点，新版本将网络节点和控制节点部署在了一起），多了一个 L3 agent。顾名思义，self-service 的三层路由服务由 openstack 提供，而 provider 的三层服务是由物理交换机提供。

<br/>

## Provider network

provider network 又称为运营商网络，self-service network 又称为租户网络。

对 neutron 而言，**provider 这种网络类型** “**没有**” **三层路由功能**，**或者说没有自主的路由功能**。他需要借助外部的网络，才能完成不同网络之间的路由，也就是说他的路由器或者三层网络服务是由 openstack 之外的力量提供，因此被称之为 provider 。

<img src="https://img1.kiosk007.top/static/images/blog/network1-connectivity.png" alt="network1-connectivity" style="zoom:73%;display:block;clear:both;margin:auto;" />

provider是一个半虚拟化的二层网络架构，只能通过桥接的方式实现，处于provider网络模式下vm获取到的ip地址与物理网络在同一网段，可以看成是物理网络的扩展，在该模式下，控制节点不需要安装L3 agent，也不需要网络节点，vm直接通过宿主机的NIC与物理网络通信,provider网络只支持flat和vlan两种模式。

<br/>

其网络的 [配置文件](https://docs.openstack.org/neutron/yoga/install/controller-install-option1-ubuntu.html) 就可以看到一些区别。

在 */etc/neutron/neutron.conf* 中，可以看到

```bash
[DEFAULT]
core_plugin = ml2
service_plugins = 
allow_overlapping_ips = false
```

在 */etc/neutron/plugins/ml2/ml2_conf.ini* 中，可以看到

```bash
# 
[ml2]
type_drivers = flat,vlan
tenant_network_types = 
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[securitygroup]
enable_ipset = true
```

在 */etc/neutron/plugins/ml2/linuxbridge_agent.ini* 可以看到

```bash
# 这里将 provider映射到具体的物理网卡，也就是创建的租户vlan从计算节点路由ens33出，
# 而后进入ens33背后的网络设备，在那里进行路由和交换。
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
# 关闭 vxlan
[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

</br>

## self-service network

self-service 就是 neutron 不依赖外部的网络和三层路由，租户自己通过 ovs 或者 Linux bridge 创建虚拟的交换机来进行互通。

self-service 模式允许租户自己创建网络，最终租户创建的网络借助 provider 网络以 NAT 方式访问外网，所以self-service模式可以看成是网络层级的延伸。要实现self-service模式必须先创建provider网络。self-service网络支持flat、vlan、vxlan、gre模式。

<img src="https://img1.kiosk007.top/static/images/blog/network2-connectivity.png" alt="network2-connectivity" style="zoom:73%;display:block;clear:both;margin:auto;" />

vm 从 self-service获取到的IP地址称为fix ip，vm 从provider 网络获取到的IP地址称为floating IP。不管租户创建的网络类型为 vxlan 或者 vlan，租户的vm 之间通过 fix ip 之间互访的流量称为**东西走向** ，当vm需要通过 SNAT 访问外网或者通过 floating IP 访问vm时，此时的流量称之为**南北走向**。

<br/>

其网络的 [配置文件](https://docs.openstack.org/neutron/yoga/install/controller-install-option2-ubuntu.html) 就可以看到一些区别。

在 */etc/neutron/neutron.conf* 中，可以看到

```bash
[DEFAULT]
core_plugin = ml2
service_plugins = router   # service_plugin 加载了 路由
allow_overlapping_ips = true
```

在 */etc/neutron/plugins/ml2/ml2_conf.ini* 中

```bash
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population

[ml2_type_flat]
flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 1:1000
```

在 */etc/neutron/plugins/ml2/linuxbridge_agent.ini* 中

```bash
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true
```

在 */etc/neutron/l3_agent.ini* 中

```bash
[DEFAULT]
interface_driver = linuxbridge
```



# 东西流量 VS 南北流量

一个 tenant 可以创建多个 subnet，此时 vm 之间的通信就分为同一个 subnet 和不同subnet 之间两种情况，区别是不同的subnet之间的通信需要经过网关，而同 subnet 之间的通信不需要经过网络节点。

<br/>

## 东西流量

- **不同 subnet 之间的通信**

无论 tenant 创建的网络是 隧道(vxlan\gre) 还是 vlan，不同的subnet之间的通信必须借助 L3 Agent 完成，而在集中式网络节点架构中，只有网络节点部署了该角色，vm 之间的流量如下：

（红线是隧道的通信方式、绿线是 vlan）

<img src="https://img1.kiosk007.top/static/images/blog/neutron5.png" alt="neutron5" style="zoom:150%;" />

1. vm1 向 vm2 发送通信请求，根据目的IP地址得知 vm2 和自己不再同一个网段，故将数据包发送到网关。
2. 数据包经过 linux bridge 通过其上的 iptables 安全策略检查，送往 br-int 并打上内部的 vlan 号
3. 数据包脱掉 br-int的内部vlan号，进入 br-tun
4. 进入 br-tun 的数据包此时的vxlan封装并打上 vni 号，通过NIC离开 compute 1
5. 数据包进入网络节点，经过br-tun的数据包完成vxlan的解封装去掉vni号。
6. 数据包进入br-int打上内部 vlan 号
7. 进入 router namespace路由空间寻找网关，vm1 的网关配置在路由器接口 qr-1 上
8. 数据包路由到vm2的网关 qr-2 上，送回 br-int并打上内部vlan号
9. 数据包脱掉br-int的内部vlan号进入 br-tun/br-vlan
10. 进入 br-tun 的数据包此时的vxlan封装并打上 vni 号，进入 br-vlan 的数据包此时打上外部 vlan 号，通过 nic 离开 network node
11. 数据包进入 compute 2 ，经过 br-tun 的数据包完成 vxlan 的解封装去掉vni号
12. 数据包进入br-int，此时被打上内部vlan号
13. 数据包离开br-int并去掉内部vlan号，送往linux brdge 通过其上的 iptables 安全策略检查
14. 最后数据包送到vm2

<br/>

- **相同subnet之间通信**

相同的subnet之间通信不需要借助 L3 Agent ，vm之间的流量如下图所示

<img src="https://img1.kiosk007.top/static/images/blog/neutron6.png" alt="neutron6" style="zoom:150%;" />

1. vm1 向 vm2 发起请求，通过目的IP地址得知vm2 与自己在同一个网段。
2. 数据包经过 linux bridge,进行安全策略检查，进入 br-int 打上内部 vlan 号
3. 数据包脱掉br-int的内部vlan号进入 br-tun。
4. 进入 br-tun 的数据包此时 vxlan 封装并打上 vni 号，进入 br-vlan 的数据包此时打上外部vlan号，通过 nic 离开 compute 1
5. 数据包进入compute2 ，经过 br-tun 的数据包完成 vxlan 的解封去掉 vni 号
6. 数据包进入 br-int,此时打上内部 vlan 号
7. 数据包经过离开br-int并去掉内部vlan号，送往 linux bridge 通过其上的 iptables 安全策略检查
8. 最后数据包送到 vm2

<br/>

## 南北流量

南北流量分为 floating ip 和无 floating ip （fix ip）两种情况，唯一的区别在于 vm 最终离开 network node 访问internet 时，有 floating ip的vm源地址为 floating ip ，而使用 fix ip 的vm通过 snat 方式，源地址为 network node 的ip。vm 南北流量如下图所示：

<img src="https://img1.kiosk007.top/static/images/blog/neutron8.png" alt="neutron8" style="zoom:150%;" />

1. vm1 向公网发出通信请求，数据包被送往网关
2. 数据包经过 linux bridge 通过其上的 iptables 安全策略检查，送往 br-int并打上内部vlan号
3. 数据包脱掉内部的vlan号经过 br-tun，br-tun 的数据包经过vxlan封装打上vni号
4. 数据包进入网络节点，经过br-tun的数据包完成vxlan的解封装去掉vni号。再送往br-int,此时打上内部的vlan号。
5. 进入 router namespace 路由空间查询路由表项，vm1 的网关配置在此路由器的 qr 接口上
6. 数据包在路由器内完成地址转换，源地址变成 qg 口 network node 节点的外网地址（floating ip 在 qg 口使用的是给vm分配的外网地址，需要提前将 fix ip 和 floating ip 绑定），送回br-int并打上内部vlan号。
7. 数据包离开 br-int 进入 br-ex，脱掉内部的vlan号，打上外网ip的vlan号
8. 最后借助 provider 的网络访问公网，此处也印成了 provider 网络只能是vlan 或者 flat 类型。

<br/>

# 常用操作

## 网络、子网、路由、端口管理

<img src="https://img1.kiosk007.top/static/images/blog/neutron9.png" alt="neutron9" style="zoom:150%;" />

<br/>

## 防火墙管理



![neutron10](https://img1.kiosk007.top/static/images/blog/neutron10.png)