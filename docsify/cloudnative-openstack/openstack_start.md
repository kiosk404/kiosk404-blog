# 创建虚拟机实例

**创建过程**

1. 创建虚拟网络

2. 创建m1.nano规格的主机（相等于定义虚拟机的硬件配置）

3. 生成一个密钥对（openstack的原理是不使用密码连接，而是使用密钥对进行连接）

4. 增加安全组规则（用iptables做的安全组）

5. 启动一个实例（启动虚拟机有三种类型：1.命令CLI 2.api 3.Dashboard）实际上Dashboard也是通过api进行操作

6. 虚拟网络分为提供者网络和私有网络，提供者网络就是跟主机在同一个网络里，私有网络自定义路由器等，跟主机不在一个网络

<br/>

---------

<br/>

下面是我们要创建的集群， Openstack 架构图

![openstack_deploy_draw](https://img1.kiosk007.top/static/images/blog/20250104121503-openstack_deploy_draw.png)

三种网络平面说明：

管理网络（management/API网络）：
提供系统管理相关功能，用于节点之间各服务组件内部通信以及对数据库服务的访问，所有节点都需要连接到管理网络，这里管理网络也承载了API网络的流量，将API网络和管理网络合并，OpenStack各组件通过API网络向用户暴露API服务。

隧道网络（tunnel网络或self-service网络）：
提供租户虚拟网络的承载网络（VXLAN or GRE）。openstack里面使用gre或者vxlan模式，需要有隧道网络；隧道网络采用了点到点通信协议代替了交换连接，在openstack里，这个tunnel就是虚拟机走网络数据流量用的。这个网络所承载的网络和官方文档Networking Option 2: Self-service networks相对应。

外部网络(external网络或者provider网络)：
openstack网络至少要包括一个外部网络，这个网络能够访问OpenStack安装环境之外的网络，并且非openstack环境中的设备能够访问openstack外部网络的某个IP。另外外部网络为OpenStack环境中的虚拟机提供浮动IP，实现openstack外部网络对内部虚拟机实例的访问。这个网络和官方文档Networking Option 1: Provider networks相对应。



# 创建网络

为配置Neutron时选择的网络选项创建虚拟网络。 如果您选择选项1，则只创建提供商网络。 如果您选择了选项2，请创建提供商和自助服务网络。

- [x] Provider network
- [x] Self-service network

提供者网络-provider网络
在启动实例之前，您必须创建必要的虚拟网络基础结构,管理员或其他特权用户必须创建此网络，因为它直接连接到物理网络基础结构。



## 创建 provider 外部网络

在控制节点上，获取admin用户凭证以访问仅管理员的CLI命令：

```
. admin-openrc.sh
```

创建虚拟网络（网络名为 provider）

```bash
openstack network create --share --external \
  --provider-physical-network provider \
  --provider-network-type flat provider
```

执行结果：

```bash
root@instance-vm00:~/env# openstack network create --share --external \
  --provider-physical-network provider \
  --provider-network-type flat provider
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
...

```

参数说明：
- -share选项允许所有项目使用虚拟网络。
- -external选项将虚拟网络定义为外部。 如果你想创建一个内部网络，你可以使用–internal代替。 默认值是内部的。
–provider-physical-network 提供者和 –provider-network-type 平面选项使用来自以下文件的信息将扁平虚拟网络连接到主机上eth1接口上的扁平（本地/非标记）物理网络：

```bash
ml2_conf.ini:
[ml2_type_flat]
flat_networks = provider
linuxbridge_agent.ini:
[linux_bridge]
physical_interface_mappings = provider:ens38
```



使用命令查看创建的网络：

```bash
root@instance-vm00:~/env# openstack network list
+--------------------------------------+----------+---------+
| ID                                   | Name     | Subnets |
+--------------------------------------+----------+---------+
| fd5bb1e2-fec4-4bb2-904e-86f666f0af11 | provider |         |
+--------------------------------------+----------+---------+
```

以 admin 用户登录 dashboard 查看创建的网络：

![openstack_network_1](https://img1.kiosk007.top/static/images/blog/20241229140403-openstack_network_1.png)

查看网络拓扑图：

![openstack_network_2](https://img1.kiosk007.top/static/images/blog/20241229140410-openstack_network_2.png)

<br/>

---------

<br/>

### 网络中创建子网

在外部网络上创建一个子网：

```bash
$ openstack subnet create --network provider \
  --allocation-pool start=192.168.120.100,end=192.168.120.200 \
  --dns-nameserver 223.5.5.5 --gateway 192.168.120.1 \
  --subnet-range 192.168.120.0/24 provider
```

执行结果：

```bash
root@instance-vm00:~/env# openstack subnet create --network provider \
  --allocation-pool start=192.168.120.100,end=192.168.120.200 \
  --dns-nameserver 223.5.5.5 --gateway 192.168.120.1 \
  --subnet-range 192.168.120.0/24 provider
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 192.168.120.100-192.168.120.200      |
| cidr                 | 192.168.120.0/24                     |
...
```



**参数说明**
用CIDR表示法将PROVIDER_NETWORK_CIDR替换为提供商物理网络上的子网。
将START_IP_ADDRESS和END_IP_ADDRESS替换为要为实例分配的子网内范围的第一个和最后一个IP地址。 该范围不得包含任何现有的活动IP地址。
将DNS_RESOLVER替换为DNS解析器的IP地址。 在大多数情况下，您可以使用主机上/etc/resolv.conf文件中的一个。
将PROVIDER_NETWORK_GATEWAY替换为提供商网络上的网关IP地址，通常为“.1”IP地址。

- -network，指定创建的子网名称
- -subnet-range 后边的provider为要创建子网的网络（要跟上面创建网络的名称对应起来）



<br/>

查看创建的子网

```bash
root@instance-vm00:~/env# openstack subnet list
+--------------------------------------+----------+--------------------------------------+------------------+
| ID                                   | Name     | Network                              | Subnet           |
+--------------------------------------+----------+--------------------------------------+------------------+
| 5844a3c9-16e9-4e23-be14-03b272bcb5e4 | provider | fd5bb1e2-fec4-4bb2-904e-86f666f0af11 | 192.168.120.0/24 |
+--------------------------------------+----------+--------------------------------------+------------------+
```



以 admin 用户登录 dashboard 查看创建的子网：

![openstack_network_3](https://img1.kiosk007.top/static/images/blog/20241229141719-openstack_network_3.png)



查看网络的拓扑变化：

![openstack_network_4](https://img1.kiosk007.top/static/images/blog/20241229141727-openstack_network_4.png)



### 查看节点的网卡变化

OpenStack中创建的实例想要访问外网必须要创建外部网络（即provider network），然后通过虚拟路由器连接外部网络和租户网络，Neutron网桥的方式实现外网的访问，当Neutron创建外部网络并创建子网后会创建一个新的网桥，并且将enp1s0 这块外部网卡加入网桥，执行 ifconfig 可以看到，多了一个 brqfd5bb1e2-fe 的网桥。

```bash
root@instance-vm00:~/env# ifconfig 
brqfd5bb1e2-fe: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.120.10  netmask 255.255.255.0  broadcast 192.168.120.255
        inet6 fe80::ec59:88ff:febc:d2f6  prefixlen 64  scopeid 0x20<link>
        ether ee:59:88:bc:d2:f6  txqueuelen 1000  (Ethernet)
        RX packets 304  bytes 11813 (11.8 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 25  bytes 1842 (1.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp1s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::5054:ff:fe44:8b86  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:44:8b:86  txqueuelen 1000  (Ethernet)
        RX packets 127109  bytes 279727346 (279.7 MB)
        RX errors 0  dropped 5  overruns 0  frame 0
        TX packets 47511  bytes 3422012 (3.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp7s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.0.10  netmask 255.255.255.0  broadcast 172.16.0.255
        inet6 fe80::5054:ff:fe9a:dac  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:9a:0d:ac  txqueuelen 1000  (Ethernet)
        RX packets 615175  bytes 100272895 (100.2 MB)
        RX errors 0  dropped 5  overruns 0  frame 0
        TX packets 561905  bytes 253418238 (253.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp8s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.1.10  netmask 255.255.255.0  broadcast 172.16.1.255
        inet6 fe80::5054:ff:fe0f:6ec9  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:0f:6e:c9  txqueuelen 1000  (Ethernet)
        RX packets 34726  bytes 1869951 (1.8 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 36  bytes 2588 (2.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

查看该网桥的信息，其中该网桥的 `tap349ee2c1-3f` 连接了 enp1s0

```bash
root@instance-vm00:~/env# brctl show
bridge name	bridge id		STP enabled	interfaces
brqfd5bb1e2-fe		8000.ee5988bcd2f6	no		enp1s0
							tap349ee2c1-3f

```

<br/>

## 创建租户网络

如果选择联网选项2，则还可以创建通过NAT连接到物理网络基础结构的自助服务（专用）网络。该网络包括一个为实例提供IP地址的DHCP服务器。此网络上的实例可以自动访问外部网络，如Internet。但是，从外部网络（例如Internet）访问此网络上的实例需要浮动IP地址
这个demo或其他非特权用户可以创建这个网络，因为它仅提供与demo项目内实例的连接。

> 创建租户网络之前，需要先创建提供商网络



### 创建 self-service 网络

1. 创建网络：

```bash
root@instance-vm00:~/env# openstack network create selfservice1
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
...

```

非特权用户通常不能为该命令提供额外的参数。该服务使用来自以下文件的信息自动选择参数：

```bash
# cat /etc/neutron/plugins/ml2/ml2_conf.ini
ml2_conf.ini:
[ml2]
tenant_network_types = vxlan
[ml2_type_vxlan]
vni_ranges = 1:1000
```

创建的内部网络类型是由tenant_network_types中指定，为vxlan。该配置能指定内部网络类型，如flat，vlan，gre等。
查看创建的网络：

```bash
root@instance-vm00:~/env# openstack network list
+--------------------------------------+--------------+--------------------------------------+
| ID                                   | Name         | Subnets                              |
+--------------------------------------+--------------+--------------------------------------+
| 1584f4ec-f5f0-411c-9e03-5c141ac22f48 | selfservice1 |                                      |
| fd5bb1e2-fec4-4bb2-904e-86f666f0af11 | provider     | 5844a3c9-16e9-4e23-be14-03b272bcb5e4 |
+--------------------------------------+--------------+--------------------------------------+
```

在dashboard上查看创建的网络：

![openstack_network_5](https://img1.kiosk007.top/static/images/blog/20241229143009-openstack_network_5.png)

### 网络中创建子网

```bash
openstack subnet create --network selfservice1 \
  --dns-nameserver 223.5.5.5 --gateway 10.0.0.1 \
  --subnet-range 10.0.0.0/24 selfservice1-net1
```

执行结果：

```bash
root@instance-vm00:~/env# openstack subnet create --network selfservice1 \
  --dns-nameserver 223.5.5.5 --gateway 10.0.0.1 \
  --subnet-range 10.0.0.0/24 selfservice1-net1
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 10.0.0.2-10.0.0.254                  |
| cidr                 | 10.0.0.0/24                          |
...
```

查看创建的子网：

```bash
root@instance-vm00:~/env# openstack subnet list
+--------------------------------------+-------------------+--------------------------------------+------------------+
| ID                                   | Name              | Network                              | Subnet           |
+--------------------------------------+-------------------+--------------------------------------+------------------+
| 5844a3c9-16e9-4e23-be14-03b272bcb5e4 | provider          | fd5bb1e2-fec4-4bb2-904e-86f666f0af11 | 192.168.120.0/24 |
| e358aae7-c108-44db-9202-2a495daf6742 | selfservice1-net1 | 1584f4ec-f5f0-411c-9e03-5c141ac22f48 | 10.0.0.0/24      |
+--------------------------------------+-------------------+--------------------------------------+------------------+

```

查看网络拓扑：

![openstack_network_6](https://img1.kiosk007.top/static/images/blog/20241229144747-openstack_network_6.png)

此时计算节点的网卡会创建一个网桥。

隧道网卡 enp8s0 会桥接在网桥上

```bash
root@instance-vm01:/var/log/neutron# ifconfig 
brq1584f4ec-f5: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        ether 66:24:cc:71:3d:47  txqueuelen 1000  (Ethernet)
        RX packets 11  bytes 1222 (1.2 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp1s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.120.20  netmask 255.255.255.0  broadcast 192.168.120.255
        inet6 fe80::5054:ff:fe3f:e643  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:3f:e6:43  txqueuelen 1000  (Ethernet)
        RX packets 45306  bytes 27696987 (27.6 MB)
        RX errors 0  dropped 6  overruns 0  frame 0
        TX packets 5812  bytes 446050 (446.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp7s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.0.20  netmask 255.255.255.0  broadcast 172.16.0.255
        inet6 fe80::5054:ff:fe92:356c  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:92:35:6c  txqueuelen 1000  (Ethernet)
        RX packets 173015  bytes 58415752 (58.4 MB)
        RX errors 0  dropped 6  overruns 0  frame 0
        TX packets 127213  bytes 53779068 (53.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp8s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.1.20  netmask 255.255.255.0  broadcast 172.16.1.255
        inet6 fe80::5054:ff:fe5a:13e  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:5a:01:3e  txqueuelen 1000  (Ethernet)
        RX packets 35898  bytes 1934419 (1.9 MB)
        RX errors 0  dropped 5  overruns 0  frame 0
        TX packets 79  bytes 8113 (8.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 72953  bytes 4543434 (4.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 72953  bytes 4543434 (4.5 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

tapfb6352fb-35: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet6 fe80::fc16:3eff:feb1:465c  prefixlen 64  scopeid 0x20<link>
        ether fe:16:3e:b1:46:5c  txqueuelen 1000  (Ethernet)
        RX packets 38  bytes 3443 (3.4 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 39  bytes 3900 (3.9 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

vxlan-337: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        ether fa:45:ec:63:3c:88  txqueuelen 1000  (Ethernet)
        RX packets 26  bytes 2530 (2.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 38  bytes 2911 (2.9 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```





### 创建路由器

自助服务网络使用通常执行双向NAT的虚拟路由器连接到提供商网络。每个路由器至少包含一个自助服务网络上的接口和提供商网络上的网关。

提供商网络必须包含router:external选项以使自助服务路由器能够使用它来连接到外部网络，例如互联网。这个admin或其他特权用户必须在网络创建期间包含此选项或稍后添加它。在这种情况下，该 router:external选项–external在创建provider网络时通过使用该参数进行设置。

1. **创建路由器**

```bash
openstack router create router
```

2. **查看创建的路由器**

```bash
root@instance-vm00:~/env# openstack route list
+--------------------------------------+--------+--------+-------+----------------------------------+-------------+-------+
| ID                                   | Name   | Status | State | Project                          | Distributed | HA    |
+--------------------------------------+--------+--------+-------+----------------------------------+-------------+-------+
| 3456a318-2360-4bdf-95ab-2fea5b8be7a0 | router | ACTIVE | UP    | e63b23475dbf4f23b650d0d69e4731dc | False       | False |
+--------------------------------------+--------+--------+-------+----------------------------------+-------------+-------+
```

3. **租户网络添加到路由器**

将自助服务网络子网添加为路由器上的接口

```bash
openstack router add subnet router selfservice1-net1
```

4. **路由器连接到外部网络**

在路由器上的提供商网络上设置网关：

```bash
openstack router set --external-gateway provider router
```

5. **验证操作**

列出网络名称空间。你应该看到一个qrouter命名空间和两个 qdhcp命名空间。

```bash
root@instance-vm00:~/env# ip netns
qdhcp-fd5bb1e2-fec4-4bb2-904e-86f666f0af11 (id: 0)
qdhcp-1584f4ec-f5f0-411c-9e03-5c141ac22f48 (id: 1)
qrouter-3456a318-2360-4bdf-95ab-2fea5b8be7a0 (id: 2)

# 列出路由器上的端口以确定提供商网络上的网关IP地址：
root@instance-vm00:~/env# openstack port list --router router
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                             | Status |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------+--------+
| 2842389a-4cb1-45f3-936a-984187695860 |      | fa:16:3e:87:d4:4b | ip_address='10.0.0.1', subnet_id='e358aae7-c108-44db-9202-2a495daf6742'        | ACTIVE |
| c943e751-2983-49fc-8d20-35849c9b9c25 |      | fa:16:3e:56:ee:17 | ip_address='192.168.120.180', subnet_id='5844a3c9-16e9-4e23-be14-03b272bcb5e4' | ACTIVE |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------+--------+
```

从控制节点或物理提供商网络上的任何主机ping此IP地址：

```bash
root@instance-vm00:~/env# ping 192.168.120.180
PING 192.168.120.180 (192.168.120.180) 56(84) bytes of data.
64 bytes from 192.168.120.180: icmp_seq=1 ttl=64 time=1.66 ms
```



<br/>

# 创建实例类型

最小的默认flavor消耗每个实例512 MB的内存。 对于包含少于4 GB内存的计算节点的环境，我们建议创建每个实例仅需要64 MB的m1.nano特征。 为了测试目的，请仅将CirrOS图像用于此flavor。

```bash
root@instance-vm00:~/env# openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano

```

查看创建的实例类型：

```bash
root@instance-vm00:~/env# openstack flavor list
+----+---------+-----+------+-----------+-------+-----------+
| ID | Name    | RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+---------+-----+------+-----------+-------+-----------+
| 0  | m1.nano |  64 |    1 |         0 |     1 | True      |
+----+---------+-----+------+-----------+-------+-----------+

```



# 生成秘钥对

大多数云镜像支持公钥认证，而不是传统的密码认证。 在启动实例之前，您必须将公钥添加到Compute服务。

```bash
root@instance-vm00:~/env# ssh-keygen -q -N ""
Enter file in which to save the key (/root/.ssh/id_ed25519): 
```

创建秘钥对，并将生成的公钥文件添加到秘钥对：

```bash
root@instance-vm00:~/env# openstack keypair create --public-key ~/.ssh/id_ed25519.pub mykey
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| created_at  | None                                            |
| fingerprint | c7:45:95:21:30:53:b0:21:7a:70:91:3a:2d:71:d3:ab |
| id          | mykey                                           |
| is_deleted  | None                                            |
| name        | mykey                                           |
| type        | ssh                                             |
| user_id     | 5406c29ffb89473e9e56c43e9421e9ad                |
+-------------+-------------------------------------------------+

```

验证密钥对是否添加成功：

```bash
root@instance-vm00:~/env# openstack keypair list  
+-------+-------------------------------------------------+------+
| Name  | Fingerprint                                     | Type |
+-------+-------------------------------------------------+------+
| mykey | c7:45:95:21:30:53:b0:21:7a:70:91:3a:2d:71:d3:ab | ssh  |
+-------+-------------------------------------------------+------+
```



# 添加安全组规则

默认情况下，默认安全组适用于所有实例，并包含拒绝对实例进行远程访问的防火墙规则。 对于像CirrOS这样的Linux映像，我们建议至少允许ICMP（ping）和安全shell（SSH）。
向default安全组添加规则：
1.允许ICMP（ping）：

```bash
root@instance-vm00:~/env# openstack security group rule create --proto icmp default
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| belongs_to_default_sg   | True                                 |
| created_at              | 2024-12-29T07:16:41Z                 |
| description             |                                      |
| direction               | ingress                              |
...

```

允许安全shell（SSH）访问：

```bash
root@instance-vm00:~/env# openstack security group rule create --proto tcp --dst-port 22 default
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| belongs_to_default_sg   | True                                 |
| created_at              | 2024-12-29T07:17:17Z                 |
| description             |                                      |
...
```

查看安全组以及创建的安全组规则：

```
root@instance-vm00:~/env# openstack security group list
+--------------------------------------+---------+------------------------+----------------------------------+------+
| ID                                   | Name    | Description            | Project                          | Tags |
+--------------------------------------+---------+------------------------+----------------------------------+------+
| 78d3f572-5e31-47ff-b383-490cb6d8e088 | default | Default security group | e63b23475dbf4f23b650d0d69e4731dc | []   |
+--------------------------------------+---------+------------------------+----------------------------------+------+
root@instance-vm00:~/env# openstack security group rule list
+----------------------+-------------+-----------+-----------+------------+-----------+-----------------------+----------------------+------------------------+
| ID                   | IP Protocol | Ethertype | IP Range  | Port Range | Direction | Remote Security Group | Remote Address Group | Security Group         |
+----------------------+-------------+-----------+-----------+------------+-----------+-----------------------+----------------------+------------------------+
| 3a3ccdd6-d8be-445d-  | tcp         | IPv4      | 0.0.0.0/0 | 22:22      | ingress   | None                  | None                 | 78d3f572-5e31-47ff-    |
| 8e9b-e20d80b1c1ea    |             |           |           |            |           |                       |                      | b383-490cb6d8e088      |
| 73ba5fab-5a3c-4ce9-  | None        | IPv4      | 0.0.0.0/0 |            | ingress   | 78d3f572-5e31-47ff-   | None                 | 78d3f572-5e31-47ff-    |
| 9864-60a5449f616c    |             |           |           |            |           | b383-490cb6d8e088     |                      | b383-490cb6d8e088      |
| 7c68531a-9e2b-40c0-  | icmp        | IPv4      | 0.0.0.0/0 |            | ingress   | None                  | None                 | 78d3f572-5e31-47ff-    |
| 9bb3-2ad85a43a1de    |             |           |           |            |           |                       |                      | b383-490cb6d8e088      |
| 91705a9d-8178-4c5f-  | None        | IPv6      | ::/0      |            | ingress   | 78d3f572-5e31-47ff-   | None                 | 78d3f572-5e31-47ff-    |
| b9c8-f29134e0dced    |             |           |           |            |           | b383-490cb6d8e088     |                      | b383-490cb6d8e088      |
| 94593d65-ca83-4cb8-  | None        | IPv4      | 0.0.0.0/0 |            | egress    | None                  | None                 | 78d3f572-5e31-47ff-    |
| aa7c-3c1e63a127f3    |             |           |           |            |           |                       |                      | b383-490cb6d8e088      |
| ece7f925-1146-4800-  | None        | IPv6      | ::/0      |            | egress    | None                  | None                 | 78d3f572-5e31-47ff-    |
| a051-c5a0b8bb2131    |             |           |           |            |           |                       |                      | b383-490cb6d8e088      |
+----------------------+-------------+-----------+-----------+------------+-----------+-----------------------+----------------------+------------------------+

```



# 创建实例前的确认

``` bash
# 1. 列出可用的flavor
openstack flavor list

# 2. 列出可用的镜像
openstack image list

# 3. 列出可用的网络
openstack network list

# 4. 列出可用的安全组
openstack security group list

# 5. 列出可用的密钥对
openstack keypair list  

```



# 创建实例

租户网络 self-service-1 上创建实例：

```bash
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=1584f4ec-f5f0-411c-9e03-5c141ac22f48 --security-group default \
  --key-name mykey selfservice1-cirros1
```

执行结果如下：

```bash
root@instance-vm00:/var/log/nova# openstack server create --flavor m1.nano --image cirros   --nic net-id=1584f4ec-f5f0-411c-9e03-5c141ac22f48 --security-group default   --key-name mykey selfservice1-cirros1
+--------------------------------------+-----------------------------------------------+
| Field                                | Value                                         |
+--------------------------------------+-----------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                        |
| OS-EXT-AZ:availability_zone          |                                               |
| OS-EXT-SRV-ATTR:host                 | None                                          |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | None                                          |
| OS-EXT-SRV-ATTR:instance_name        |                                 ...

```

检查创建出实例的状态：

```bash
root@instance-vm00:/var/log/nova# openstack server list
+--------------------------------------+----------------------+--------+------------------------+--------+---------+
| ID                                   | Name                 | Status | Networks               | Image  | Flavor  |
+--------------------------------------+----------------------+--------+------------------------+--------+---------+
| b709970b-b099-449b-9667-0c434a334ac9 | selfservice1-cirros1 | ACTIVE | selfservice1=10.0.0.55 | cirros | m1.nano |
+--------------------------------------+----------------------+--------+------------------------+--------+---------+
```



# 虚拟控制台访问实例

为您的实例获取虚拟网络计算（VNC）会话URL并从Web浏览器访问它：

```
root@instance-vm00:/var/log/nova# openstack console url show selfservice1-cirros1
+----------+-------------------------------------------------------------------------------------------+
| Field    | Value                                                                                     |
+----------+-------------------------------------------------------------------------------------------+
| protocol | vnc                                                                                       |
| type     | novnc                                                                                     |
| url      | http://controller:6080/vnc_auto.html?path=%3Ftoken%3D2c1459d6-ccf2-4a62-a36a-6b7514dfb969 |
+----------+-------------------------------------------------------------------------------------------+

```

![openstack_network_7](https://img1.kiosk007.top/static/images/blog/20241229155300-openstack_network_7.png)

可以看到创建出的 server ，通过浏览器的 web console 可以登录进去，而且已经分配了一个 ip 地址 10.0.0.55

可以做以下测试

```
ping 10.0.0.1          # 租户网络网关
ping 192.168.120.1     # 外部网络网关
ping www.kiosk007.top  # 外部网络
```



# 为实例分配一个浮动 IP 地址

如果想通过外网远程连接到实例，需要在外部网络上创建浮动IP地址，并将浮动ip地址关联到实例上，然后通过访问外部的浮动ip地址来访问实例。

1. 在外部网络上生成浮动IP地址

```bash
root@instance-vm00:/var/log/nova# openstack floating ip create provider
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| created_at          | 2024-12-29T08:01:01Z                 |
| description         |                                      |
| dns_domain          | None                                 |
...

```

2. 将浮动IP地址与实例关联

```bash
root@instance-vm00:/var/log/nova# openstack server add floating ip selfservice1-cirros1 192.168.120.105
```

3. 检查浮动IP地址的关联关系

```bash
root@instance-vm00:/var/log/nova# openstack server list
+--------------------------------------+----------------------+--------+-----------------------------------------+--------+---------+
| ID                                   | Name                 | Status | Networks                                | Image  | Flavor  |
+--------------------------------------+----------------------+--------+-----------------------------------------+--------+---------+
| b709970b-b099-449b-9667-0c434a334ac9 | selfservice1-cirros1 | ACTIVE | selfservice1=10.0.0.55, 192.168.120.105 | cirros | m1.nano |
+--------------------------------------+----------------------+--------+-----------------------------------------+--------+---------+
```

在控制节点或者外部网络上通过floating IP验证对实例的访问是否正常：
通过来自控制器节点或供应商物理网络上的任何主机的浮动IP地址验证与实例的连接性：

```bash
root@instance-vm00:/var/log/nova# ping 192.168.120.105
PING 192.168.120.105 (192.168.120.105) 56(84) bytes of data.
64 bytes from 192.168.120.105: icmp_seq=1 ttl=63 time=2.35 ms
```

在我们实际的物理宿主机上，理论上现在也可以访问这个虚拟机了。

```bash
➜  ~ ssh root@192.168.120.105
The authenticity of host '192.168.120.105 (192.168.120.105)' can't be established.
ECDSA key fingerprint is SHA256:/h3er/I/fvolXLWtFM0ig4zX7jNoJulyFcTpMmHp7tQ.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes

```



# 网卡变化

创建好内部网络和实例之后，vxlan隧道就建立起来。系统会在控制节点创建一个vxlan 的VTEP，在计算节点创建一个vxlan的VTEP。

```bash
# 控制节点上观察
root@instance-vm00:~/env# brctl show
bridge name	bridge id		STP enabled	interfaces
brq1584f4ec-f5		8000.6624cc713d47	no		tap2842389a-4c
							tap9f835e65-b4
							vxlan-337


# 计算节点上观察
root@instance-vm01:~# brctl show
bridge name	bridge id		STP enabled	interfaces
brq1584f4ec-f5		8000.6624cc713d47	no		tapfb6352fb-35
							vxlan-337

```

