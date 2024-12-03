---
title: 网速变慢？你可能需要先设置好 DNS
author: kiosk
tags:
  - web
categories:
  - network
date: 2024-10-24 16:08:00
autoCollapseToc: true
outdatedInfoWarning: true
draft: true
weight: 10
contentCopyright: MIT
mathjax: true
---

有没有觉得 `kiosk007.top` 最近访问变快了？没错，我的主站最近接入了 腾讯云的 [EO(EdegeOne)](https://console.cloud.tencent.com/edgeone)，当然只是14天体验版，体验过后还是会改会直连模式，毕竟是要收费的。不过能使得访问质量得到加速，第一个重要原因就是我们的请求被指向了就近节点，实现了就近访问。其中就依赖一个重要的底层技术 - **DNS**。

<!--more-->

<br/>

## DNS：网络世界的指南针

DNS，英文全称「Domain Name Server」，中文全称「域名服务器」。这种服务器不像网站使用的服务器为用户提供内容，而是将人类易于理解的「域名」转换为机器易于理解的「IP 地址」。

```bash
# dig kiosk007.top +short
kiosk007.top.eo.dnse1.com.
101.72.224.113 
119.188.123.202
```

这块的逻辑大家应该有基础网络知识的应该都了解。这里就不多做介绍了。

具体还可以参考: [DNS 原理入门](https://ruanyifeng.com/blog/2016/06/dns.html)

<br/>

## 智能解析

DNS（Domain Name System）智能解析是一种更高级的域名解析技术。它可以根据用户的地理位置、网络运营商、访问时间等多种因素，智能地将域名解析到不同的 IP 地址。传统的 DNS 解析通常是将域名固定地解析到一个或几个 IP 地址，而智能解析能够动态地调整解析结果，以优化用户的访问体验。

<br/>

举个例子，本站 kiosk007.top 接入了 腾讯云 EO。

从 [boce.aliyun.com](boce.aliyun.com) 上可以看到 kiosk007.top 这个域名从不同地方解析到的结果是不一样的。 

<img src="https://img1.kiosk007.top/static/images/blog/20241127095934-dnszhinengjiexi.png" />



- 在贵阳移动解析到了 [221.178.3.68](https://ipinfo.io/221.178.3.68) 和 [36.147.64.154](https://ipinfo.io/36.147.62.154) ，从  [ipinfo.io](https://ipinfo.io/) 可以查看到两个IP都是重庆移动的IP，在地理位置上都属于 西南省份。
- 在 甘肃移动解析出的 2个IP 分别属于 宁夏移动和甘肃移动的，在地理位置上都属于 西北省份。



<br/>

能做到这点，其实是因为 EO 动态加速将域名做到了就近接入。

kiosk007.top 被 CNAME 到了 `kiosk007.top.eo.dnse1.com.` , 最后再解析出真正的 IP 。

<img src="https://img1.kiosk007.top/static/images/blog/20241127100012-kioskdnseo.png"/>



其实 CNAME 是大多数厂家用做调度的核心技术。这里以一个网上找到一个域名的调度说起，从 boce.aliyun.com 可以得出，这个域名是有接入动态加速技术的。

```bash
$ dig mon.snssdk.com

; <<>> DiG 9.18.28-0ubuntu0.24.04.1-Ubuntu <<>> mon.snssdk.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40247
;; flags: qr rd ra; QUERY: 1, ANSWER: 8, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;mon.snssdk.com.			IN	A

;; ANSWER SECTION:
mon.snssdk.com.		227	IN	CNAME	mon.snssdk.com.bytedns1.com.
mon.snssdk.com.bytedns1.com. 33	IN	CNAME	mon.snssdk.com.c.dsa.cdnbuild.net.
mon.snssdk.com.c.dsa.cdnbuild.net. 106 IN CNAME	l7-online-self-max-v6.s.dsa.cdnbuild.net.
l7-online-self-max-v6.s.dsa.cdnbuild.net. 17 IN	A 111.161.204.165
l7-online-self-max-v6.s.dsa.cdnbuild.net. 17 IN	A 111.161.204.160
l7-online-self-max-v6.s.dsa.cdnbuild.net. 17 IN	A 111.161.204.161
l7-online-self-max-v6.s.dsa.cdnbuild.net. 17 IN	A 111.161.204.162
l7-online-self-max-v6.s.dsa.cdnbuild.net. 17 IN	A 111.161.204.164

;; Query time: 10 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Wed Nov 27 09:50:27 CST 2024
;; MSG SIZE  rcvd: 246

```

这里有 三层 CNAME ，分别是

`mon.snssdk.com.bytedns1.com.`                          -- 接入调度

`mon.snssdk.com.c.dsa.cdnbuild.net.`              --  内部调度管理

`l7-online-self-max-v6.s.dsa.cdnbuild.net.`  -- 调度域

<br/>

前2层一般是调度的内部管理CNAME，最后一层是 调度域名。动态加速厂商管理的一般是调度域。很多域名可以统一 CNAME 到调度域名。

<br/>

回到 kiosk007.top ，这里只有一层 CNAME ，我怀疑是 EO 将 调度域隐藏了，这可以通过 [CNAME Flattening](https://blog.cloudflare.com/introducing-cname-flattening-rfc-compliant-cnames-at-a-domains-root/) 来实现, 后面会再说。

```bash
$ dig kiosk007.top +short  
kiosk007.top.eo.dnse1.com.
119.188.123.202
101.72.224.113

```



下面就借此机会说一下 DNS 都有哪些特性，如果希望网络优化从 DNS 入手，都还有哪些手段。

<br/>



## 胶水记录（Glue Record）

胶水记录是一种特殊的记录类型，来对比下它和普通的 DNS 记录有什么差别

- 普通的 DNS 记录是保存在权威服务器上的
  - 权威服务器一般由域名注册商提供，当然也有由第三方提供的独立解析服务。
  - 用户通过域名解析后台设置解析记录，值是保存在权威服务器上的。
- 胶水记录保存在注册局的DNS服务器上
  - 注册局（Domain Registry）是管理某个后缀所有域名的机构。
  - 用户通过域名注册商后台设置胶水记录时，值是保存到注册局的服务器上的。

详细介绍之前先回顾一下 DNS 查询过程。

我们查询 DNS 的时候一般是 递归查询 PublicDNS 或者 LocalDNS 。它们会直接输出 DNS解析结果 ，而 LocalDNS 或者 PublicDNS 会做迭代查询，层层迭代直到查到结果，当然有缓存的话直接返回结果。

![ABUIABAEGAAg94nTjAYo-Ya5kwQwtwU4nQM!600x600](https://img1.kiosk007.top/static/images/blog/20241201123531-ABUIABAEGAAg94nTjAYo-Ya5kwQwtwU4nQM!600x600.png)



> LocalDNS 一般由 DHCP 分发的 DNS Server
>
> PublicDNS 是一些公共大厂的DNS Server，常见的有 [119.29.29.29](https://www.dnspod.cn/products/publicdns) (腾讯云的PublicDNS) 、[223.5.5.5](https://alidns.com/) (阿里云的PublicDNS)、[180.184.1.1](https://www.volcengine.com/docs/6758/177901)  (火山引擎的 PublicDNS)



以查询 `www.qq.com` 为例。我们直接查询可以看到有1层 CNAME ，3个IP。 

```bash
# dig www.qq.com +short
ins-r23tsuuf.ias.tencent-cloud.net.
221.198.70.47
```

其实我们会以 递归服务器的工作方式来逐级查询 DNS 记录。

**一、获取根服务器列表**

因为所有域名的 DNS 记录查询入口都在这里，就好像你要进入一个多级的目录，你得一级一级地进入，而根服务器就是入口。

全球共有 13 组公认的 DNS 服务器，可在 IANA 网站上查询到：

https://www.iana.org/domains/root/servers

<br/>

**二、向根服务器发送查询**

下面的过程是这样的：

Public DNS：“www.qq.com 的地址你有吗？” 

root权威DNS：“没有，我只有 .com 顶级域的 权威地址， 你可以找他们问, .com 顶级域权威DNS是 d.gtld-servers.net. 等 , 对了他们的地址我也有，就是 192.5.6.30  这些”

```bash
# dig qq.com @m.root-servers.net

; <<>> DiG 9.18.28-0ubuntu0.24.04.1-Ubuntu <<>> qq.com @m.root-servers.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50679
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 13, ADDITIONAL: 27
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;qq.com.				IN	A

;; AUTHORITY SECTION:
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.

;; ADDITIONAL SECTION:
m.gtld-servers.net.	172800	IN	A	192.55.83.30
l.gtld-servers.net.	172800	IN	A	192.41.162.30
k.gtld-servers.net.	172800	IN	A	192.52.178.30
j.gtld-servers.net.	172800	IN	A	192.48.79.30
i.gtld-servers.net.	172800	IN	A	192.43.172.30
h.gtld-servers.net.	172800	IN	A	192.54.112.30
g.gtld-servers.net.	172800	IN	A	192.42.93.30
f.gtld-servers.net.	172800	IN	A	192.35.51.30
e.gtld-servers.net.	172800	IN	A	192.12.94.30
d.gtld-servers.net.	172800	IN	A	192.31.80.30
c.gtld-servers.net.	172800	IN	A	192.26.92.30
b.gtld-servers.net.	172800	IN	A	192.33.14.30
a.gtld-servers.net.	172800	IN	A	192.5.6.30
m.gtld-servers.net.	172800	IN	AAAA	2001:501:b1f9::30
l.gtld-servers.net.	172800	IN	AAAA	2001:500:d937::30
k.gtld-servers.net.	172800	IN	AAAA	2001:503:d2d::30
j.gtld-servers.net.	172800	IN	AAAA	2001:502:7094::30
i.gtld-servers.net.	172800	IN	AAAA	2001:503:39c1::30
h.gtld-servers.net.	172800	IN	AAAA	2001:502:8cc::30
g.gtld-servers.net.	172800	IN	AAAA	2001:503:eea3::30
f.gtld-servers.net.	172800	IN	AAAA	2001:503:d414::30
e.gtld-servers.net.	172800	IN	AAAA	2001:502:1ca1::30
d.gtld-servers.net.	172800	IN	AAAA	2001:500:856e::30
c.gtld-servers.net.	172800	IN	AAAA	2001:503:83eb::30
b.gtld-servers.net.	172800	IN	AAAA	2001:503:231d::2:30
a.gtld-servers.net.	172800	IN	AAAA	2001:503:a83e::2:30

;; Query time: 74 msec
;; SERVER: 202.12.27.33#53(m.root-servers.net) (UDP)
;; WHEN: Sun Dec 01 12:23:50 CST 2024
;; MSG SIZE  rcvd: 831

```

**三、向 .com 权威服务器发送查询**

这里选了一个 .com 的权威服务器查询。下面的对话过程如下：

PublicDNS ：“你有 www.qq.com 的记录吗?”

.com 权威：“没有，但我知道 qq.com 的权威DNS是 ns1.qq.com. 他应该有 www.qq.com 的记录，对了 ns1.qq.com 权威地址的 结尾是 .com ，他的地址，你就不必问我了， 我这里也有，顺便给你哦， 是 101.227.218.144 等”

```bash
dig www.qq.com @192.55.83.30

; <<>> DiG 9.18.28-0ubuntu0.24.04.1-Ubuntu <<>> www.qq.com @a.gtld-servers.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35510
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 4, ADDITIONAL: 17
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.qq.com.			IN	A

;; AUTHORITY SECTION:
qq.com.			172800	IN	NS	ns1.qq.com.
qq.com.			172800	IN	NS	ns2.qq.com.
qq.com.			172800	IN	NS	ns3.qq.com.
qq.com.			172800	IN	NS	ns4.qq.com.

;; ADDITIONAL SECTION:
ns1.qq.com.		172800	IN	A	101.227.218.144
ns1.qq.com.		172800	IN	A	14.116.136.7
ns1.qq.com.		172800	IN	A	157.255.246.101
ns1.qq.com.		172800	IN	A	203.205.220.251
ns1.qq.com.		172800	IN	AAAA	2402:4e00:8030::111
ns2.qq.com.		172800	IN	A	111.230.158.42
ns2.qq.com.		172800	IN	A	211.100.32.218
ns2.qq.com.		172800	IN	AAAA	240e:9f:c600::8
ns2.qq.com.		172800	IN	A	43.129.131.210
ns3.qq.com.		172800	IN	A	112.60.1.69
ns3.qq.com.		172800	IN	A	117.184.232.216
ns3.qq.com.		172800	IN	A	49.51.76.77
ns4.qq.com.		172800	IN	A	170.106.32.66
ns4.qq.com.		172800	IN	A	218.68.91.143
ns4.qq.com.		172800	IN	A	58.144.154.100
ns4.qq.com.		172800	IN	A	59.36.132.142

;; Query time: 333 msec
;; SERVER: 192.5.6.30#53(a.gtld-servers.net) (UDP)
;; WHEN: Sun Dec 01 12:25:40 CST 2024
;; MSG SIZE  rcvd: 391


```

`.com` 权威服务器告诉PublicDNS，`qq.com` 由 `ns*.qq.com` 这组服务器管理，并且还顺便告诉了他们的 IP 地址。

**第四步：向 qq.com 域名权威服务器发送查询**

Public 选择了 ns1.qq.com. 这个权威的地址，并使用 101.227.218.144 这个IP访问.

下面是和 qq.com 的权威服务器 ns1.qq.com 进行查询的对话：

PublicDNS: "你这里有 www.qq.com 的地址吗？"

qq.com 的权威："没有，但我知道 www.qq.com 的权威是 ns-cnc1.qq.com. 对了 ns-cnc1.qq.com 的地址, 我也有，拿去就是 157.255.6.102 "

> 题外话：为什么这里由出现一层权威？？？ 原因是 www.qq.com 不是直接有 A 记录的，他有一层 CNAME ，所以这里由有一个 NS 。

```bash
dig www.qq.com @101.227.218.144 

; <<>> DiG 9.18.28-0ubuntu0.24.04.1-Ubuntu <<>> www.qq.com @101.227.218.144
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13872
;; flags: qr rd ad; QUERY: 1, ANSWER: 0, AUTHORITY: 2, ADDITIONAL: 7
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4304
; COOKIE: fe39971c695d1887 (echoed)
;; QUESTION SECTION:
;www.qq.com.			IN	A

;; AUTHORITY SECTION:
www.qq.com.		86400	IN	NS	ns-cnc1.qq.com.
www.qq.com.		86400	IN	NS	ns-cnc2.qq.com.

;; ADDITIONAL SECTION:
ns-cnc1.qq.com.		3600	IN	A	157.255.6.102
ns-cnc1.qq.com.		3600	IN	A	218.68.91.139
ns-cnc1.qq.com.		3600	IN	A	140.207.180.96
ns-cnc1.qq.com.		3600	IN	AAAA	240e:e1:aa00:2002::3
ns-cnc2.qq.com.		3600	IN	A	157.255.6.102
ns-cnc2.qq.com.		3600	IN	A	218.68.91.139

;; Query time: 35 msec
;; SERVER: 101.227.218.144#53(101.227.218.144) (UDP)
;; WHEN: Sun Dec 01 12:47:46 CST 2024
;; MSG SIZE  rcvd: 203

```



**余下步骤**:

看来解析出一个域名真的是太麻烦了，下面的流程不再详细赘述了，我通过解析操作展示

```bash
1. 问 ns-cnc1.qq.com 询问结果

# dig www.qq.com @157.255.6.102       

; <<>> DiG 9.18.28-0ubuntu0.24.04.1-Ubuntu <<>> www.qq.com @157.255.6.102
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 54329
;; flags: qr aa rd ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4304
; COOKIE: cbd86af4a7b28ff0 (echoed)
;; QUESTION SECTION:
;www.qq.com.			IN	A

;; ANSWER SECTION:
www.qq.com.		300	IN	CNAME	ins-r23tsuuf.ias.tencent-cloud.net.

;; Query time: 43 msec
;; SERVER: 157.255.6.102#53(157.255.6.102) (UDP)
;; WHEN: Sun Dec 01 13:01:34 CST 2024
;; MSG SIZE  rcvd: 99
```



查询 ins-r23tsuuf.ias.tencent-cloud.net. 又要经历上述一层逻辑，这里我们直接给出结果吧。

```bash
dig ins-r23tsuuf.ias.tencent-cloud.net +short
221.198.70.47
```







<br/>

## CNAME 记录打平 (CNAME Flattening)









## ANAME 





## DNS 解析优化调优



## CoreDNS 验证

> CoreDNS是一个灵活、可扩展的开源DNS服务器，它支持多种协议和后端存储，使其成为构建现代、可定制的DNS基础设施的理想选择。下面将介绍如何安装和配置CoreDNS，以便你可以快速搭建一个强大而稳定的DNS服务。



### CroeDNS 部署



<br/>

1. 下载 CoreDNS 的二进制文件，放置到 bin 文件夹下。

```bash
wget https://github.com/coredns/coredns/releases/download/v1.12.0/coredns_1.12.0_linux_amd64.tgz

tar -xvf coredns_1.12.0_linux_amd64.tgz -C .

sudo cp coredns /usr/local/bin/coredns
```

<br/>

2. 创建system文件，使用 systemd 管理

```bash
vi /etc/systemd/system/coredns.service
```

内容如下：

```bash
[Unit]
Description=CoreDNS DNS server
Documentation=https://coredns.io
After=network.target

[Service]
PermissionsStartOnly=true
LimitNOFILE=1048576
LimitNPROC=512
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
User=root
WorkingDirectory=~
ExecStart=/usr/local/bin/coredns -conf=/etc/coredns/Corefile
ExecReload=/bin/kill -SIGUSR1 
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

保存完成后重载 systemd 配置：`systemctl daemon-reload`

<br/>

3. 创建配置文件

首先创建文件夹用于存放配置文件

```bash
mkdir /etc/coredns
```

新建配置文件

```bash
vim /etc/coredns/Corefile
```

这是一个简单的配置:

```
.:53 {
	bind 127.0.0.1 ::1
        errors
        health {
            lameduck 5s
        }
        ready
	forward . 223.5.5.5 129.29.29.29
        cache 30
        reload
        loadbalance
    }   
```

<br/>

4. 确保 53 端口未被占用

```
lsof -i :53
```

systemd-resolved 占用53端口（Ubuntu Server常见） 首先修改 /etc/systemd/resolved.conf,内容如下:

```
[Resolve]
DNS=114.114.114.114
DNSStubListener=no
```

DNS 是配置本机指向的DNS服务器，建议配置运营商或者公共DNS。 DNSStubListener=no 是指不监听53端口。 配置完成后执行命令重启系统服务： `sudo systemctl daemon-reload` `sudo systemctl restart systemd-resolved` 查看/etc/resolv.conf 软链位置，执行命令 `ll /etc/resolv.conf`，如果软链显示如下，则不需要改动: `/etc/resolv.conf -> /run/systemd/resolve/resolv.conf`，否则设置软链 `sudo ln -s -f /run/systemd/resolve/resolv.conf /etc/resolv.conf`。

<br/>

5. 设置开机自启动并启动 CoreDNS

```
systemctl enable coredns
systemctl start coredns
```
