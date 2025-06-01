---
title: Linux流量控制-TC框架
author: kiosk
tags: 
  - openstack
categories:
  - Cloud Computing
date: 2022-08-22 11:23:00
---

TC（Traffic Control）是Linux操作系统中的流量控制器，用于Linux内核的流量控制。它通过在输出端口处建立一个队列来实现流量控制，可以限速、整形和策略控制网络流量的带宽、延迟、丢包等参数，从而实现网络流量的优化和管理



<!--more-->

<br/>

TC是Linux自带的模块，一般情况下不需要另行安装，可以用 man tc 查看tc 相关命令细节，tc 要求内核 2.4.18 以上

<br/>

Linux中的QoS分为入口(Ingress)部分和出口(Egress)部分，入口部分主要用于进行入口流量限速(policing)，出口部分主要用于队列调度(queuingscheduling)。大多数排队规则(qdisc)都是用于输出方向的，输入方向只有一个排队规则，即ingressqdisc。ingressqdisc本身的功能很有限，输入方向只有一个排队规则，即ingressqdisc（因为没有缓存只能实现流量的drop）但可用于重定向incomingpackets。通过Ingressqdisc把输入方向的数据包重定向到虚拟设备ifb，而ifb的输出方向可以配置多种qdisc，就可以达到对输入方向的流量做队列调度的目的。

<br/>



<img src="https://img1.kiosk007.top/static/images/blog/20240908115320-linux-tc-cccc.png" />

<br/>

即：

- 我们能控制出方向，通过 Shaping 将出的流量控制成我们自己想要的模样。如 QOS（Quality of Service）
- 至于入方向， 我们无法控制，只能通过 Policy 将包丢弃。



Ingress 限速只能对整个网卡入流量限速，无队列之分

```bash
tc qdisc add dev eth0 ingress

tc filter add dev eth0 parent ffff: protocol ip prio 10 u32 match ipsrc 0.0.0.0/0 police rate 2048kbps burst 1m drop flowid :1
```

而对于 Egress 利用队列规定建立处理数据包的队列，并定义队列中的数据包被发送的方式，从而实现对流量的控制。TC 模块实现流量控制功能使用的队列规定分为两类，一类是无类队列规定，另一类是分类队列规定。无类队列规定相对简单，而分类队列规定则引出了分类和过滤器等概念，使其流量控制功能增强。

<br/>

# TC 规则

流量控制包括以下的几种方式。

## 流量控制方式

- **Shaping（限制）**：当流量被限制时，它的传输速率就被控制在某个值之下，限制值可以远远小于有效带宽，这样可以平滑掉突发的数据流量，使得网络更为稳定。

- **SCHEDULING（调度）**：通过调度数据包的传输，可以在带宽范围内，按照优先级分配带宽，SCHEDULING（调度）也只适用于 Egress 流量。

- **POLICING（策略）**：SHAPPING 用于处理外向的流量，而 POLICING（策略）处于接收到数据

- **DROPPING（丢弃）**：如果流量超过某个设定的带宽，就丢弃数据包，不管是向内还是向外

<br/>

## 流量控制处理的对象

- **qdisc（排队规则）**：流量控制(traffic control)的基础。无论何时，内核如果需要通过某个网络接口发送数据包，它都需要按照为这个接口配置的qdisc(排队规则)把数据包加入队列。然后，内核会尽可能多地从qdisc里面取出数据包，把它们交给网络适配器驱动模块。最简单的QDisc是pfifo它不对进入的数据包做任何的处理，数据包采用先入先出的方式通过队列。不过，它会保存网络接口一时无法处理的数据包。下面会
- **class（类别）**：某些QDisc(排队规则)可以包含一些类别，不同的类别中可以包含更深入的QDisc(排队规则)，通过这些细分的QDisc还可以为进入的队列的数据包排队。通过设置各种类别数据包的离队次序，QDisc可以为设置网络数据流量的优先级。
- **filter（过滤器）**：决定它们按照何种QDisc进入队列。无论何时数据包进入一个划分子类的类别中，都需要进行分类。分类的方法可以有多种，使用fileter(过滤器)就是其中之一。使用filter(过滤器)分类时，内核会调用附属于这个类(class)的所有过滤器，直到返回一个判决。如果没有判决返回，就作进一步的处理，而处理方式和QDISC有关。需要注意的是，filter(过滤器)是在QDisc内部，它们不能作为主体。



<br/>

# TC 命令

tc 命令使用以下

```bash
tc  qdisc [ add | change | replace | link ] dev DEV [ parent qdisc-id | root ] [ handle qdisc-id ] qdisc [ qdisc specific parameters ]  
      
tc class [ add | change | replace ] dev DEV parent qdisc-id  [  classid class-id ] qdisc [ qdisc specific parameters ]  
      
tc filter [ add | change | replace ] dev DEV [ parent qdisc-id | root ] protocol protocol prio priority filtertype [ filtertype specific param‐eters ] flowid flow-id  
      
tc [ FORMAT ] qdisc show [ dev DEV ]  
      
tc [ FORMAT ] class show dev DEV  
       
tc filter show dev DEV  
      
    ###### FORMAT := { -s[tatistics] | -d[etails] | -r[aw] | -p[retty] | i[ec] }  
```



<br/>



# 控制网络的 QoS 的方式

在 Linux 下，可以通过 TC 控制网络的 QoS，主要就是通过队列的方式。

## 无类别排队规则

 **无类别排队规则(Classless Queuing Disciplines)** ，这是一种把不同的网络包分类的技术。

### pfifo_fast 



<img src="https://img1.kiosk007.top/static/images/blog/20240907212331-linux-tc-pfifo_fast.png">

pfifo_fast 是三个先入先出的队列，根据网络报的 TOS ，看到这个包到底应该进入哪个队列，TOS 有 4 位组成，

0 - F 其实就是 0x00 - 0xFF , 表示不同的 TOS 进入哪种队列

```bash
# tc qdisc show dev ens33
qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap 1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
```



<br/>

- **扩展知识 qdisc**

qdisc 全称是 queuing discipline，中文叫 排队规则。如果需要通过某个网络接口发送数据包，他都需要为这个接口配置 qdisc（排队规则）把数据包加入队列。

最简单的 qdisc 是 pfifo，他不对进入的数据包做任何的处理，数据包采用先入先出的方式通过队列。pfifo_fast 稍微复杂一点。它的队列包括三个波段（band），在每个波段里面，使用先进先出的规则。

(如下，执行 `ip a` 命令可以看到 ens33 有个 qdisc pfifo_fast )

<br/>

```bash
# ip a
... 
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:0d:72:3a brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    inet 172.16.80.129/24 metric 100 brd 172.16.80.255 scope global dynamic ens33
       valid_lft 946sec preferred_lft 946sec
    inet6 fe80::20c:29ff:fe0d:723a/64 scope link 
       valid_lft forever preferred_lft forever
```

<br/>

三个波段（band）的优先级也不相同，band 0 优先级最高。band 2 优先级最低。

数据包是按照服务类型（Type of Service, TOS） 分配到三个波段 （band）中的。TOS 是 IP 头里面的一个字段的，代表了当前的包是最高优先级还是最低优先级的。

<img src="https://img1.kiosk007.top/static/images/blog/20240908000645-linux-tc-ip.png" />

<br/>

### 随机公平队列（SFQ，Stochastic Fair Queuing）

<img src="https://img1.kiosk007.top/static/images/blog/20240908002216-linux_tc_sfq.png" />

<br/>

会建立很多的 FIFO 队列， TCP Session 会计算 hash 值，通过 hash 值分配到某个队列，在队列的另一端，网络报会通过轮询的策略从各个队列取出发送。

<br/>

### 令牌桶规则（TBF， Token Bucket Filte）

所有的网络报排成队列进行发送 ，但不是到了队头才能发送，而是必须拿到 令牌才可以发送，所以 TBF 可以用来进行限速。



<br/>

## 基于类别的队列规则

基于**类别的队列规则（Classful Queuing Disciplines）** , 其中典型的为 **分层令牌桶（HTB， Hierarchical Token Bucket）**。

<img src="https://img1.kiosk007.top/static/images/blog/20240908003011-linux_tc_htb.png" />

<br/>

HTB 往往是一棵树， 可以使用 TC 为Linux 下的网卡 ens33 创建一个 HTB 的队列规则，需要付给他一个句柄（:1）。

<br/>

这是整课树的根节点，接下来会有分支，例如图中的三个分支。分别为 (:10)、(:11)、(:12) 。最后的参数default 12, 表示默认发送给 1:12 ，也即发送给第三个分支。

```bash
tc qdisc add dev ens33 root handle 1: htb default 12 
```

<br/>

对于这个网卡，需要规定发送的速度，一般有2个速度可以配置，一个是**rate**，表示一般情况下的速度，一个是 **ceil**，表示最高情况下的速度，对于根节点来讲，这两个速度是一样的，于是创建一个 root class，速度为 (rate=100kbps, ceil=100kbps)

```bash
tc class add dev ens33 parent 1: classid 1:1 htb rate 100kbps ceil 100kbps
```

<br/>

接下来创建3个分支，也即创建3个子class。每个子 class 统一有2个速度，三个分支分别是 (rate=30kbps, ceil=100kbps)、(rate=10kbps, ceil=100kbps)、（rate=60kbps, ceil=100kbps）。

```bash
tc class add dev ens33 parent 1:1 classid 1:10 htb rate 30kbps ceil 100kbps
tc class add dev ens33 parent 1:1 classid 1:11 htb rate 10kbps ceil 100kbps
tc class add dev ens33 parent 1:1 classid 1:12 htb rate 60kbps ceil 100kbps
```

<br/>

可以看到三个 rate 加起来，是整个网卡允许的最大速度。（30+10+60）

<br/>

HTB 有一个很好的特性，同一个 root class 下的子类可以相互借流量，如果不直接在队列规则下面创建一个 root class。而是直接创建三个 class，它们是不能直接互相借流量的。

最后创建 叶子队列规则，分别是 fifo 和 sfq。

```bash
tc qdisc add dev ens33 parent 1:10 handle 20: pfifo limit 5
tc qdisc add dev ens33 parent 1:11 handle 30: pfifo limit 5
tc qdisc add dev ens33 parent 1:12 handle 40: sfq perturb 10
```

<br/>

基于这个队列规则，可以设置 TC 设定的发送规则：从 150.158.101.167 来的包，发送给 80 端口的包，从第一个分支 1:10 走；其他从  150.158.101.167  发送来的包从第二个分支 1:11 走，其他的走默认分支。

```bash
tc filter add dev ens33 protocol ip parent 1:0 prio 1 u32 match ip src 150.158.101.167 match ip dport 80 0xffff flowid 1:10
tc filter add dev ens33 protocol ip parent 1:0 prio 1 u32 match ip src 150.158.101.167 flowid 1:11
```



<br/>

# 实际使用

## 针对端口限速

在使用git拉取代码时很容易跑满带宽，为了控制带宽的使用，配置如下：

```bash
#查看现有的队列
tc -s qdisc ls dev ens33

#查看现有的分类
tc -s class ls dev ens33

#创建队列
tc qdisc add dev ens33 root handle 1:0 htb default 1 
#添加一个tbf队列，ens33，命名为1:0 ，默认归类为1
#handle：为队列命名或指定某队列

#创建分类
tc class add dev ens33 parent 1:0 classid 1:1 htb rate 10Mbit burst 15k
#为ens33下的root队列1:0添加一个分类并命名为1:1，类型为htb，带宽为10M
#rate: 是一个类保证得到的带宽值.如果有不只一个类,请保证所有子类总和是小于或等于父类.
#ceil: ceil是一个类最大能得到的带宽值.

#创建一个子分类
tc class add dev ens33 parent 1:1 classid 1:10 htb rate 10Mbit ceil 10Mbit burst 15k
#为1:1类规则添加一个名为1:10的类，类型为htb，带宽为10M

#为了避免一个会话永占带宽,添加随即公平队列sfq.
tc qdisc add dev ens33 parent 1:10 handle 10: sfq perturb 10
#perturb：是多少秒后重新配置一次散列算法，默认为10秒
#sfq,他可以防止一个段内的一个ip占用整个带宽

#使用u32创建过滤器
tc filter add dev ens33 protocol ip parent 1:0 prio 1 u32 match ip sport 22 flowid 1:10

#删除队列
tc qdisc del dev ens33 root
```

<br/>

## 针对IP进行限速

因为带宽资源有限（20Mbit≈2Mbyte），使用git上传代码的时候导致带宽资源告警，所以对git进行限速，要求：内网不限速；外网上传速度为1M左右。

```bash
#!/bin/bash
#针对不同的ip进行限速

#清空原有规则
tc qdisc del dev ens33 root

#创建根序列
tc qdisc add dev ens33 root handle 1: htb default 1

#创建一个主分类绑定所有带宽资源（20M）
tc class add dev ens33 parent 1:0 classid 1:1 htb rate 20Mbit burst 15k

#创建子分类
tc class add dev ens33 parent 1:1 classid 1:10 htb rate 20Mbit ceil 10Mbit burst 15k
tc class add dev ens33 parent 1:1 classid 1:20 htb rate 20Mbit ceil 20Mbit burst 15k

#避免一个ip霸占带宽资源（上面有提到）
tc qdisc add dev ens33 parent 1:10 handle 10: sfq perturb 10
tc qdisc add dev ens33 parent 1:20 handle 20: sfq perturb 10

#创建过滤器
#对所有ip限速
tc filter add dev ens33 protocol ip parent 1:0 prio 2 u32 match ip dst 0.0.0.0/0 flowid 1:10
#对内网ip放行
tc filter add dev ens33 protocol ip parent 1:0 prio 1 u32 match ip dst 10.0.0.0/8 flowid 1:20
```



> 下面是限速的效果
>
> 限速前：
>
> ```bash
> # speedtest-cli
> Retrieving speedtest.net configuration...
> Testing from China Unicom (123.112.69.232)...
> Retrieving speedtest.net server list...
> Selecting best server based on ping...
> Hosted by China Unicom (Mianyang) [1413.88 km]: 118.96 ms
> Testing download speed................................................................................
> Download: 7.84 Mbit/s
> Testing upload speed......................................................................................................
> Upload: 10.59 Mbit/s
> ```
>
> 限速后：(为了演示效果，上传限速1Mbit / s)
>
> ```bash
> # speedtest-cli
> Retrieving speedtest.net configuration...
> Testing from China Unicom (123.112.69.232)...
> Retrieving speedtest.net server list...
> Selecting best server based on ping...
> Hosted by China Unicom 5G (Shanghai) [1072.57 km]: 57.737 ms
> Testing download speed................................................................................
> Download: 8.68 Mbit/s
> Testing upload speed......................................................................................................
> Upload: 1.39 Mbit/s
> 
> ```
>
> 

<br/>

## 针对IP进行丢包

```bash
sudo tc qdisc del dev ens33 root
sudo tc qdisc add dev ens33 root handle 1: htb default 10    # 创建htb规则, 并创建一个分支, 默认发给 :10
sudo tc class add dev ens33 parent 1: classid 1:1 htb rate 100mbps ceil 200mbps # 创建 root class
sudo tc qdisc add dev ens33 parent 1:1 handle 10: netem loss random 50%         # 丢包50%
sudo tc filter add dev ens33 protocol ip parent 1:0 u32 match ip dst 110.242.68.66/24 flowid 1:1
```

> 执行后的效果
>
> ```bash
> # ping 110.242.68.66
> PING 110.242.68.66 (110.242.68.66) 56(84) bytes of data.
> 64 bytes from 110.242.68.66: icmp_seq=1 ttl=128 time=15.2 ms
> ...
> 64 bytes from 110.242.68.66: icmp_seq=25 ttl=128 time=19.7 ms
> ^C
> --- 110.242.68.66 ping statistics ---
> 28 packets transmitted, 14 received, 50% packet loss, time 27337ms
> rtt min/avg/max/mdev = 15.145/49.997/123.787/31.840 ms
> ```
>
> 

更多的 netem 模拟网络波动的方法：[使用 TC 和 Netem 模拟网络异常](https://www.hi-linux.com/posts/35699.html#vip-container)

<br/>

## 限速脚本

```bash
#!/usr/bin/env bash
NBQ={Number of TX queues on NIC}
HANDLE_ID=100
FQ="fq buckets 4096"
HTB="htb default 2"
for ETH in ens33 ens37
do
        tc qd del dev $ETH root 2>/dev/null
        tc qd add dev $ETH root handle ${HANDLE_ID}: mq
        for i in `seq 1 $NBQ`
        do
                slot=$( printf %x $(( i )) )
                echo $slot
                tc qd add dev $ETH handle $slot: parent 100:$slot ${HTB}
                echo "tc class add dev $ETH parent $slot: classid $slot:1 htb rate {limit} ceil (m/k)bit"
                tc class add dev $ETH parent $slot: classid $slot:1 htb rate {limit} ceil {limit}
                tc class add dev $ETH parent $slot: classid $slot:2 htb rate {limit} ceil {limit}
                tc class add dev $ETH parent $slot: classid $slot:3 htb rate 10gbit
                tc filter add dev $ETH protocol ip parent $slot: prio 1 u32 match ip dst {VIP1}/32 flowid $slot:1
                tc filter add dev $ETH protocol ip parent $slot: prio 1 u32 match ip dst {VIP2}/32 flowid $slot:2
        done
done
```

> limit \ VIP1 \ VIP2 需要自行补全才能执行