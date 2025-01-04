# Openstack 网络

在前面的章节中，我们已经搭建出了一个 openstack 服务，并且创建了一个子网 10.0.0.0/24 ，也在上面创建出了一个vm 实例。

![openstack_network_8](https://img1.kiosk007.top/static/images/blog/20241229161432-openstack_network_8.png)

在操作之前先检查一下当前的网络状态。

```bash
root@instance-vm00:~/env# ip netns 
qdhcp-fd5bb1e2-fec4-4bb2-904e-86f666f0af11 (id: 0)
qdhcp-1584f4ec-f5f0-411c-9e03-5c141ac22f48 (id: 1)
qrouter-3456a318-2360-4bdf-95ab-2fea5b8be7a0 (id: 2)

```

第一个 dhcp (为 provider 网络分配 IP 地址)

```bash
root@instance-vm00:~/env# ip netns exec qdhcp-fd5bb1e2-fec4-4bb2-904e-86f666f0af11 ip addr
...
2: ns-349ee2c1-3f@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether fa:16:3e:0e:62:61 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.120.100/24 brd 192.168.120.255 scope global ns-349ee2c1-3f
       valid_lft forever preferred_lft forever
    inet 169.254.169.254/32 brd 169.254.169.254 scope global ns-349ee2c1-3f
       valid_lft forever preferred_lft forever
    inet6 fe80::a9fe:a9fe/128 scope link 
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe0e:6261/64 scope link 
       valid_lft forever preferred_lft forever

```

第二个 dhcp （为 self-network 分配IP地址）

```bash
root@instance-vm00:~/env# ip netns exec qdhcp-1584f4ec-f5f0-411c-9e03-5c141ac22f48 ip addr
...
2: ns-9f835e65-b4@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether fa:16:3e:70:6c:2b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global ns-9f835e65-b4
       valid_lft forever preferred_lft forever
    inet 169.254.169.254/32 brd 169.254.169.254 scope global ns-9f835e65-b4
       valid_lft forever preferred_lft forever
    inet6 fe80::a9fe:a9fe/128 scope link 
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe70:6c2b/64 scope link 
       valid_lft forever preferred_lft forever

```



# 创建第二个子网及虚机

这里只展示命令，详细过程可以在上节找到

```bash
# 创建一个self network
openstack network create selfservice2

# 创建子网
openstack subnet create --network selfservice2 \
  --dns-nameserver 223.5.5.5 --gateway 10.0.1.1 \
  --subnet-range 10.0.1.0/24 selfservice2-net1
  
# 查看创建的网络
root@instance-vm00:~/env# openstack network list
+--------------------------------------+--------------+--------------------------------------+
| ID                                   | Name         | Subnets                              |
+--------------------------------------+--------------+--------------------------------------+
| 1584f4ec-f5f0-411c-9e03-5c141ac22f48 | selfservice1 | e358aae7-c108-44db-9202-2a495daf6742 |
| 22afb410-0da9-45e5-9a12-323be1a77fa8 | selfservice2 | 6094dabd-4904-4ae4-91a7-7c6ee89176fa |
| fd5bb1e2-fec4-4bb2-904e-86f666f0af11 | provider     | 5844a3c9-16e9-4e23-be14-03b272bcb5e4 |
+--------------------------------------+--------------+--------------------------------------+
```

![openstack_network_9](https://img1.kiosk007.top/static/images/blog/20241229162918-openstack_network_9.png)

然后我们看一下控制节点上网络设备的变化情况，如下，又多了一个tap设备（veth-pair类型）、vxlan设备和网桥设备，而且新的tap设备和新的vxlan设备在新的网桥上

```bash
root@instance-vm00:~/env# ip addr |grep vxlan
13: vxlan-337: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master brq1584f4ec-f5 state UNKNOWN group default qlen 1000
18: vxlan-104: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master brq22afb410-0d state UNKNOWN group default qlen 1000


root@instance-vm00:~/env# brctl show
bridge name	bridge id		STP enabled	interfaces
brq1584f4ec-f5		8000.6624cc713d47	no		tap2842389a-4c
							tap9f835e65-b4
							vxlan-337
brq22afb410-0d		8000.12bc994ba180	no		tap8be75b5d-c9
							vxlan-104
brqfd5bb1e2-fe		8000.ee5988bcd2f6	no		enp1s0
							tap349ee2c1-3f
							tapc943e751-29

```



**创建虚机**

这里为 第二个组网里创建一台虚机。

```bash
root@instance-vm00:~/env# openstack network list
+--------------------------------------+--------------+--------------------------------------+
| ID                                   | Name         | Subnets                              |
+--------------------------------------+--------------+--------------------------------------+
| 1584f4ec-f5f0-411c-9e03-5c141ac22f48 | selfservice1 | e358aae7-c108-44db-9202-2a495daf6742 |
| 22afb410-0da9-45e5-9a12-323be1a77fa8 | selfservice2 | 6094dabd-4904-4ae4-91a7-7c6ee89176fa |
| fd5bb1e2-fec4-4bb2-904e-86f666f0af11 | provider     | 5844a3c9-16e9-4e23-be14-03b272bcb5e4 |
+--------------------------------------+--------------+--------------------------------------+

# net-id 填上面 selfservice2 的网络ID
root@instance-vm00:~/env# openstack server create --flavor m1.nano --image cirros \
  --nic net-id=22afb410-0da9-45e5-9a12-323be1a77fa8 --security-group default \
  --key-name mykey selfservice2-cirros1
```

![openstack_network_10](https://img1.kiosk007.top/static/images/blog/20241229163819-openstack_network_10.png)



此时 2 个虚机还无法ping通



# 用已有路由打通两个网络

截止至上一节，我们已经开好了两个网络、子网，并且在两个子网中分别开了一台虚机，但是这两台虚机之间并不能够通信。



**创建路由器打通2个子网**

因为一个子网只能连到一个虚拟路由上，之前为了让节点可以上外网已经将 子网1 连到了 router 路由上，所以这里只需要将 子网2 也连到 router 路由器上。



```bash
# 查看2个子网
root@instance-vm00:~/env# openstack subnet list
+--------------------------------------+-------------------+--------------------------------------+------------------+
| ID                                   | Name              | Network                              | Subnet           |
+--------------------------------------+-------------------+--------------------------------------+------------------+
| 5844a3c9-16e9-4e23-be14-03b272bcb5e4 | provider          | fd5bb1e2-fec4-4bb2-904e-86f666f0af11 | 192.168.120.0/24 |
| 6094dabd-4904-4ae4-91a7-7c6ee89176fa | selfservice2-net1 | 22afb410-0da9-45e5-9a12-323be1a77fa8 | 10.0.1.0/24      |
| e358aae7-c108-44db-9202-2a495daf6742 | selfservice1-net1 | 1584f4ec-f5f0-411c-9e03-5c141ac22f48 | 10.0.0.0/24      |
+--------------------------------------+-------------------+--------------------------------------+------------------+


# 将子网2加入到虚拟路由器中
openstack router add subnet router selfservice2-net1
```



现在可以子网2中的虚机可以ping通子网1中的虚机了。

![openstack_network_11](https://img1.kiosk007.top/static/images/blog/20241229170636-openstack_network_11.png)



# 从 LinuxBridge 到 Openvswitch

下面将 LinuxBridge 改为 Openswitch ，

<br/>

- **一、控制节点**

##### **1.1 清理所有用户下的VM、网络、路由器**

在dashboard把所有的VM、网络、路由器都删掉。这样，我们就不需要重置neutron数据库。

##### **1.2 停止LinuxBridgeAgent**

执行以下命令停止LinuxBridgeAgent

```bash
systemctl stop neutron-linuxbridge-agent
systemctl disable neutron-linuxbridge-agent
```

##### **1.3 安装OpenvswitchAgent**

执行以下命令安装OpenvswitchAgent（libibverbs不装ovs-vsctl会报错）

```bash
sudo apt-get install neutron-openvswitch-agent libibverbs1
```

##### **1.4 配置ML2 插件**

编辑 `/etc/neutron/plugins/ml2/ml2_conf.ini` 文件，在`[ml2]`下找到`mechanism_drivers`，改成如下：

```bash
[ml2]
...
mechanism_drivers = openvswitch,l2population
```

##### **1.5 配置ML3 插件**

编辑`/etc/neutron/l3_agent.ini`，在`[DEFAULT]`下面，更改`interface_driver`这一行，并添加`external_network_bridge`一行，如下：

```bash
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
external_network_bridge = br-ex
...
```

##### **1.6 配置openvswitch_agent**

编辑`/etc/neutron/plugins/ml2/openvswitch_agent.ini`，更改如下三个字段的配置， 如下：

```bash
[ovs]
local_ip = 192.168.91.70
bridge_mappings = provider:br-ex
...

[securitygroup]
enable_security_group = true
firewall_driver =neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
...

[agent]
tunnel_types = vxlan
l2_population = true
...
```

##### **1.7 配置dhcp**

编辑`/etc/neutron/dhcp_agent.ini`，更改interfacedriver，如下：

```bash
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
...
```

##### **1.8 重启neutron所有服务**

```bash
$ systemctl restart neutron-server neutron-openvswitch-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
$ systemctl enable neutron-server neutron-openvswitch-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
```

<br/>

- **二、计算节点**

##### **2.1 停止LinuxBridgeAgent**

执行以下命令停止LinuxBridgeAgent

```bash
$ systemctl stop neutron-linuxbridge-agent
$ systemctl disable neutron-linuxbridge-agent
```

##### **2.2 安装OpenvswitchAgent**

执行以下命令安装OpenvswitchAgent（控制节点计算节点都要）

```bash
sudo apt-get install neutron-openvswitch-agent libibverbs1
```

##### **2.3 配置openvswitch_agent**

编辑`/etc/neutron/plugins/ml2/openvswitch_agent.ini`，更改如下三个字段的配置， 如下：

```bash
[ovs]
# 相比控制节点，这里少了bridge_mappings，因为不通过计算节点出外网
local_ip = 192.168.91.71
...

[securitygroup]
enable_security_group = true
firewall_driver =neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
...

[agent]
tunnel_types = vxlan
l2_population = true
...
```

##### **2.4 启动openvswitch-agent服务**

```bash
$ systemctl start neutron-openvswitch-agent
$ systemctl enable neutron-openvswitch-agent
```

<br/>

- **三、控制节点**

##### **3.1 创建br-ex**

```
$ ovs-vsctl add-br br-ex
$ ovs-vsctl add-port br-ex enp1s0
```

[此文](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux_openstack_platform/7/html/networking_guide/openstack_networking_concepts#provider_networks)指引如何将br-ex持久化，这样主机重启就不需要再手动创建br-ex与add-port。但是实践发现没有这样操作也能持久化，并且发现每台主机上都有ovsdb服务，所以猜测存储在本地的ovsdb数据库上。

##### **3.2 查询网络的类型是否为OVS**

```bash
root@instance-vm00:~/env# openstack network agent list
+--------------------------------------+--------------------+---------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host          | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+---------------+-------------------+-------+-------+---------------------------+
| 1a8740c0-0496-4417-8a5b-a6c7a16e85c7 | Metadata agent     | instance-vm00 | None              | :-)   | UP    | neutron-metadata-agent    |
| 3e3bf6b2-940d-4d89-b583-82f36c624b05 | DHCP agent         | instance-vm00 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 5bd83f43-499e-43bc-8441-20f0b7514fa4 | Linux bridge agent | instance-vm00 | None              | XXX   | UP    | neutron-linuxbridge-agent |
| ab90a3f9-6676-4213-86d7-9fe218f9c81e | Open vSwitch agent | instance-vm01 | None              | :-)   | UP    | neutron-openvswitch-agent |
| b1d5247a-420d-4477-a31f-8c1af26209c7 | L3 agent           | instance-vm00 | nova              | :-)   | UP    | neutron-l3-agent          |
| cfd76abb-1f5e-4ef1-a673-09c34bce7541 | Linux bridge agent | instance-vm01 | None              | XXX   | UP    | neutron-linuxbridge-agent |
| e0460be0-7970-4f7d-9113-256817bdafd3 | Open vSwitch agent | instance-vm00 | None              | :-)   | UP    | neutron-openvswitch-agent |
+--------------------------------------+--------------------+---------------+-------------------+-------+-------+---------------------------+

```

删除以前的LinuxBridgeAgent

```bash
$ openstack network agent delete 5bd83f43-499e-43bc-8441-20f0b7514fa4
$ openstack network agent delete cfd76abb-1f5e-4ef1-a673-09c34bce7541
```



重启服务

```bash
root@instance-vm00:~/env# openstack network agent delete 5bd83f43-499e-43bc-8441-20f0b7514fa4 
root@instance-vm00:~/env# openstack network agent delete cfd76abb-1f5e-4ef1-a673-09c34bce7541

```

##### **3.3 查看ovs网桥**

应该能看到三个ovs网桥

``` bash
root@instance-vm00:~/env# ovs-vsctl show
18e58a9a-0ad0-4950-9a64-2a16f88284ee
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-tun                #### 隧道网络
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port br-tun
            Interface br-tun
                type: internal
        Port vxlan-ac100114
            Interface vxlan-ac100114
                type: vxlan
                options: {df_default="true", egress_pkt_mark="0", in_key=flow, local_ip="172.16.1.10", out_key=flow, remote_ip="172.16.1.20"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    Bridge br-int               #### 内部管理网
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port br-int
            Interface br-int
                type: internal
        Port tap9f835e65-b4
            tag: 3
            Interface tap9f835e65-b4
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
        Port tap8be75b5d-c9
            tag: 1
            Interface tap8be75b5d-c9
                type: internal
        Port tap349ee2c1-3f
            tag: 2
            Interface tap349ee2c1-3f
                type: internal
    Bridge br-ex               ##### 外部网络
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port enp1s0
            Interface enp1s0
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
        Port br-ex
            Interface br-ex
                type: internal
    ovs_version: "3.3.0"

```

<br/>

# 使用OVS创建网络

在创建网络之前，现将之前基于 LinuxBridge 的网络都清除，重新基于 OVS 创建网络。

要删除所有网络及其相关组件，你需要按照以下步骤进行操作。请确保你了解每个命令的影响，并谨慎操作。

#### 删除原有网络

- 步骤1：删除路由器

首先，列出所有路由器并删除它们。路由器必须在删除网络之前删除，因为它们可能与网络相关联。

```bash
# 列出所有路由器
openstack router list

# 列出与路由器关联的所有端口：
openstack port list --router <router-id>

# 如果端口连接到虚拟机，删除相应的虚拟机
openstack server delete <vm-id>

# 查看路由器上连接的子网
openstack router show <router-id>

# 删除对应的子网
openstack router remove subnet <router-id> <subnet-id>

# 断开路由器的外部网关
openstack router remove gateway <router-id>

# 等待虚拟机删除后，检查端口是否自动删除。如果没有，手动删除端口（一般上述操作完成后，这个自动就没有了）
openstack port delete <port-id>

# 删除指定路由器
openstack router delete <router-id>
```

- 步骤2：删除网络和子网

接下来，删除所有网络和子网。子网通常会随着网络的删除而被删除。

```bash
# 列出所有网络
openstack network list

# 删除指定网络
openstack network delete <network-id>
```

- 注意事项

  - **数据丢失**：删除网络和路由器是不可逆的操作，所有相关数据将被永久删除。

  - **检查依赖关系**：确保没有虚拟机或其他资源依赖于这些网络和路由器。

  - **权限**：确保你有足够的权限来执行这些操作。

通过以上步骤，你可以删除OpenStack中的所有网络及其相关组件



#### 新建网络

### 创建2个网络子网及虚机（ovs+vxlan）

**网络拓扑图如下：**

> (DHCP 1 是 10.0.0.2   DHCP 2 是 10.0.1.2  , 下图画错了)

![openstack_network_ovs](https://img1.kiosk007.top/static/images/blog/20250104173900-openstack_network_ovs.png)

先创建网络和实例，预计创建2个子网，分别是 10.0.0.0/24 一个是 10.0.1.0/24 ，

```bash
# 创建子网1
openstack network create selfservice1
openstack subnet create --network selfservice1 \
  --dns-nameserver 223.5.5.5 --gateway 10.0.0.1 \
  --subnet-range 10.0.0.0/24 selfservice1-net1

# 创建子网2
openstack network create selfservice2
openstack subnet create --network selfservice2 \
  --dns-nameserver 223.5.5.5 --gateway 10.0.1.1 \
  --subnet-range 10.0.1.0/24 selfservice2-net1
  
# 查看刚才创建的网络
openstack network list 
openstack subnet list 

```

此时，我们再在这2个子网上各创建一个VM

```bash
# 网络1上的VM
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=d953cf16-af03-45e0-864a-867c0798280c --security-group default \
  --key-name mykey selfservice1-cirros1
  
# 网络2上的VM (创建2个)
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=fea7be8c-424e-4e4b-9dc8-d834dd223d6f --security-group default \
  --key-name mykey selfservice2-cirros1
  
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=fea7be8c-424e-4e4b-9dc8-d834dd223d6f --security-group default \
  --key-name mykey selfservice2-cirros2
  
# 查看刚才创建的VM
openstack server list
+--------------------------------------+----------------------+--------+-------------------------+--------+---------+
| ID                                   | Name                 | Status | Networks                | Image  | Flavor  |
+--------------------------------------+----------------------+--------+-------------------------+--------+---------+
| 266918fb-3a6b-4813-b94a-6f2073fbc3b5 | selfservice2-cirros2 | ACTIVE | selfservice2=10.0.1.113 | cirros | m1.nano |
| 6707eef2-4c82-4212-a10f-ea5b6e7ac22c | selfservice1-cirros1 | ACTIVE | selfservice1=10.0.0.224 | cirros | m1.nano |
| 579b9095-62b5-47e1-81d8-96c69b106da5 | selfservice2-cirros1 | ACTIVE | selfservice2=10.0.1.57  | cirros | m1.nano |
+--------------------------------------+----------------------+--------+-------------------------+--------+---------+


```





1、控制节点

先查看ovs网桥的相关信息（注意port与tag）

```bash
root@instance-vm00:~# ovs-vsctl show
18e58a9a-0ad0-4950-9a64-2a16f88284ee
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port vxlan-ac100114
            Interface vxlan-ac100114
                type: vxlan
                options: {df_default="true", egress_pkt_mark="0", in_key=flow, local_ip="172.16.1.10", out_key=flow, remote_ip="172.16.1.20"}
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port br-int
            Interface br-int
                type: internal
        Port tap9ca0f3d5-e5
            tag: 4
            Interface tap9ca0f3d5-e5
                type: internal
        Port tap76c81432-37
            tag: 5
            Interface tap76c81432-37
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
    Bridge br-ex
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port enp1s0
            Interface enp1s0
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
        Port br-ex
            Interface br-ex
                type: internal
    ovs_version: "3.3.0"

```

从控制节点上看，可以看到2个 ns

```bash
root@instance-vm00:~# ip netns list
qdhcp-d953cf16-af03-45e0-864a-867c0798280c (id: 0)
qdhcp-fea7be8c-424e-4e4b-9dc8-d834dd223d6f (id: 1)

```

查看 linux-bridge (控制节点上无linux-bridge)

```bash
root@instance-vm00:~# brctl show
bridge name	bridge id		STP enabled	interfaces
virbr0		8000.525400d1eeb7	yes		  # 这个和本文无关
```



2. 计算节点

```bash
root@instance-vm01:~# ovs-vsctl show
e04211c1-552c-4883-b6c8-24e18df87d27
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port vxlan-ac10010a
            Interface vxlan-ac10010a
                type: vxlan
                options: {df_default="true", egress_pkt_mark="0", in_key=flow, local_ip="172.16.1.20", out_key=flow, remote_ip="172.16.1.10"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port br-tun
            Interface br-tun
                type: internal
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port qvo271f9531-54
            tag: 1
            Interface qvo271f9531-54
        Port qvo6b5d65c3-82
            tag: 2
            Interface qvo6b5d65c3-82
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port qvo7d4e020e-fd
            tag: 1
            Interface qvo7d4e020e-fd
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "3.3.0"

```

查看网桥

```bash
root@instance-vm01:~# brctl show 
bridge name	bridge id		STP enabled	interfaces
docker0		8000.02425f38a36c	no		
qbr271f9531-54		8000.1a4d53bd9aa8	no		qvb271f9531-54
							tap271f9531-54
qbr6b5d65c3-82		8000.46e63f17f3f2	no		qvb6b5d65c3-82
							tap6b5d65c3-82
qbr7d4e020e-fd		8000.1e663d829f52	no		qvb7d4e020e-fd
							tap7d4e020e-fd
virbr0		8000.525400d1eeb7	yes		

```

上面的网桥可以在 selfservicex 中看到

![openstack_network_ovs_1](https://img1.kiosk007.top/static/images/blog/20250104175802-openstack_network_ovs_1.png)



#### 连通性分析

下面的操作都需要在计算节点上进行。

1. **同网络的连通性分析**

> selfservice2-cirros1 （10.0.1.57）和 selfservice2-cirros2 （10.0.1.113） 

```bash
oot@instance-vm01:~# ovs-ofctl dump-flows br-int
 cookie=0x433f33f9b81f214f, duration=3732.353s, table=0, n_packets=0, n_bytes=0, priority=65535,dl_vlan=4095 actions=drop
 cookie=0x433f33f9b81f214f, duration=73.486s, table=0, n_packets=0, n_bytes=0, priority=10,icmp6,in_port="qvo7d4e020e-fd",icmp_type=136 actions=resubmit(,24)
...
 cookie=0x433f33f9b81f214f, duration=3958.994s, table=62, n_packets=0, n_bytes=0, priority=3 actions=NORMAL
```

流表的匹配规则：流表中有很多rule，一个包进来后，都是从table=0的规则开始匹配。如果table=0中有很多rule，则priority越大，越先匹配；如果priority一样，则顺序匹配。

假设我们从10.0.1.57 ping 10.0.1.113，arp广播包首先从 qbr271f9531-54 这个port进入到br-int。

最后一行规则最为重要，`actions=NORMAL`表示，根据mac地址与vlan的tag进行转发。所以，10.0.1.57 ping 10.0.1.113 的vlan tag是一样的，所以它们是通的。而172.16.100.18的tag与它们不一样，所以是不通的。



- **查看 br-tun 的流表**

```bash
root@instance-vm01:~# ovs-ofctl dump-flows br-tun
 cookie=0x98005a9960fa437, duration=4059.955s, table=0, n_packets=89, n_bytes=6784, priority=1,in_port="patch-int" actions=resubmit(,2)
 cookie=0x98005a9960fa437, duration=3916.321s, table=0, n_packets=9, n_bytes=2396, priority=1,in_port="vxlan-ac10010a" actions=resubmit(,4)
 cookie=0x98005a9960fa437, duration=4059.953s, table=0, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x98005a9960fa437, duration=4059.951s, table=2, n_packets=3, n_bytes=126, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x98005a9960fa437, duration=4059.950s, table=2, n_packets=86, n_bytes=6658, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
 cookie=0x98005a9960fa437, duration=4059.949s, table=3, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x98005a9960fa437, duration=3917.302s, table=4, n_packets=6, n_bytes=1597, priority=1,tun_id=0x3e3 actions=mod_vlan_vid:1,resubmit(,10)
 cookie=0x98005a9960fa437, duration=3887.203s, table=4, n_packets=3, n_bytes=799, priority=1,tun_id=0x1ba actions=mod_vlan_vid:2,resubmit(,10)
 cookie=0x98005a9960fa437, duration=4059.948s, table=4, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x98005a9960fa437, duration=4059.947s, table=6, n_packets=0, n_bytes=0, priority=0 actions=drop
 cookie=0x98005a9960fa437, duration=4059.946s, table=10, n_packets=9, n_bytes=2396, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0x98005a9960fa437,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:OXM_OF_IN_PORT[]),output:"patch-int"
 cookie=0x98005a9960fa437, duration=113.139s, table=20, n_packets=1, n_bytes=42, priority=2,dl_vlan=2,dl_dst=fa:16:3e:4f:41:24 actions=strip_vlan,load:0x1ba->NXM_NX_TUN_ID[],output:"vxlan-ac10010a"
 cookie=0x98005a9960fa437, duration=112.748s, table=20, n_packets=2, n_bytes=84, priority=2,dl_vlan=1,dl_dst=fa:16:3e:d3:95:07 actions=strip_vlan,load:0x3e3->NXM_NX_TUN_ID[],output:"vxlan-ac10010a"
 cookie=0x98005a9960fa437, duration=4059.945s, table=20, n_packets=0, n_bytes=0, priority=0 actions=resubmit(,22)
 cookie=0x98005a9960fa437, duration=113.142s, table=22, n_packets=28, n_bytes=2090, priority=1,dl_vlan=2 actions=strip_vlan,load:0x1ba->NXM_NX_TUN_ID[],output:"vxlan-ac10010a"
 cookie=0x98005a9960fa437, duration=112.751s, table=22, n_packets=52, n_bytes=4060, priority=1,dl_vlan=1 actions=strip_vlan,load:0x3e3->NXM_NX_TUN_ID[],output:"vxlan-ac10010a"
 cookie=0x98005a9960fa437, duration=4059.944s, table=22, n_packets=6, n_bytes=508, priority=0 actions=drop

```

解释一下这里面的一些关键信息：

（1）`dl_dst=00:00:00:00:00:00/01:00:00:00:00:00`表示目的mac地址为单播地址，`dl_dst=01:00:00:00:00:00/01:00:00:00:00:00`表示目的mac地址为多播地址，dl为data link的缩写。
（2）`mod_vlan_vid:1`表示增加或更改二层数据帧的vlan-id为1，一般从tun收到数据帧并转发给br-int时需要添加。
（3）`tun_id=0x3e3`表示vxlan的vni为0x3e3，十进制即为995。我们可以通过如下命令看到network的vni。

```bash
root@instance-vm00:~/env# openstack network show selfservice2
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        | nova                                 |
....
| name                      | selfservice2                         |
| port_security_enabled     | True                                 |
| project_id                | e63b23475dbf4f23b650d0d69e4731dc     |
| provider:network_type     | vxlan                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 995     ## 这里                       |
| qos_policy_id             | None                                 |
...

```

### 创建路由器打通网络（ovs+vxlan）

![openstack_network_ovs_2](https://img1.kiosk007.top/static/images/blog/20250104191050-openstack_network_ovs_2.png)

```bash
# 创建一个路由器
openstack router create router

# 将2个服务子网添加为路由器上的接口
openstack router add subnet router selfservice1-net1
openstack router add subnet router selfservice2-net1
```

- 查看创建的路由器

```bash
root@instance-vm00:~/env# openstack router show dbeeae5f-28ef-468a-98db-7dc2d55b7eda
+-------------------------+----------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                            |
+-------------------------+----------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                               |
| availability_zone_hints |                                                                                                                                  |
...
| enable_ndp_proxy        | None                                                                                                                             |
| external_gateway_info   | null                                                                                                                             |
| flavor_id               | None                                                                                                                             |
| ha                      | False                                                                                                                            |
| id                      | dbeeae5f-28ef-468a-98db-7dc2d55b7eda                                                                                             |
| interfaces_info         | [{"port_id": "8eb656be-cfb6-4a19-ace2-40ae41e747af", "ip_address": "10.0.1.1", "subnet_id":                                      |
|                         | "19350940-ba10-4188-9921-cedcf36c3150"}, {"port_id": "b2ffd9d3-dd90-414b-9a94-5af26a228349", "ip_address": "10.0.0.1",           |
|                         | "subnet_id": "4393c5e9-8b8a-4e04-ba13-c29e9eebe163"}]                                                                            |
| name                    | router                                                                                                                           |
| project_id              | e63b23475dbf4f23b650d0d69e4731dc                                                                                                 |
...
```





### 路由器连接外网(ovs-vxlan)

```bash
# 创建一个外部网络 
openstack network create --share --external \
  --provider-physical-network provider \
  --provider-network-type flat provider

openstack subnet create --network provider \
  --allocation-pool start=192.168.120.100,end=192.168.120.200 \
  --dns-nameserver 223.5.5.5 --gateway 192.168.120.1 \
  --subnet-range 192.168.120.0/24 provider

# 路由器连接到外部网络上
openstack router set --external-gateway provider router

# 创建一个浮动IP地址
openstack floating ip create provider

# 得到IP地址 192.168.120.188, 将浮动IP与实例关联
openstack server add floating ip selfservice1-cirros1 192.168.120.188

```



