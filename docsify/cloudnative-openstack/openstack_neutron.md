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
