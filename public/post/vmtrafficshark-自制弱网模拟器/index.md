# VMTrafficShark 自制弱网模拟器

VMTrafficShark是参考 [TrafficShark](http://geekluo.com/contents/2017/05/02/38-trafficshark-tool-for-network.html) 改造后的弱网模拟和抓包工具。其本质是参考Facebook开源的弱网模拟器 [ATC](https://github.com/facebookarchive/augmented-traffic-control) 。方便应用开发测试人员模拟弱网环境下的用户体验。VMTrafficShark是基于Vmware/VirtualBox 虚拟机，方便移动部署，一键开机，配置灵活，功能强大。

<!--more-->

----------------------------------------

# 功能

<img src="https://img1.kiosk007.top/static/images/network/VMTrafficShark/vmtrafficshark.png" />


VMTrafficShark 可以模拟各种复杂网络场景.

- [x] 低带宽、高丢包、高时延.
- [x] DNS空解析、DNS劫持.
- [x] 7层HTTP内容劫持、模拟服务端故障
- [x] 中间人抓包、方便导出 pcap、sslkey.log （设备只要应用层没有做 SSL Pinning 等反中间人措施的APP均可嗅探TLS加密后的应用层内容）
- [ ] 可视化前端操作、多场景模拟一键操作
- [ ] 接入设备管理


# 环境搭建与安装

## 环境

**硬件**

- 笔记本电脑
- USB无线网卡

> 本文使用的是 [COMFAST CF-812AC双频千兆无线网卡 (不是打广告)](https://item.jd.com/100006507594.html)

**软件**

- vmware workstation
- ubuntu 19.10



## 安装
### 安装必须软件包
``` bash
➜ sudo apt-get update
➜ sudo apt install git gcc make vim libssl-dev net-tools -y
```
### 连接USB网卡，安装驱动
- 连接USB网卡

<img src="https://img1.kiosk007.top/static/images/network/VMTrafficShark/netinterface_driver_install.png
" >
> 最好将USB改称USB 3.0 ，2.0好像兼容性不是特别好
<img src="/images/network/VMTrafficShark/net_interface_usb.png
" style="height:250px">

- 安装驱动

``` bash
➜ git clone https://github.com/cilynx/rtl88x2BU_WiFi_linux_v5.3.1_27678.20180430_COEX20180427-5959.git 
➜ cd rtl88x2BU_WiFi_linux_v5.3.1_27678.20180430_COEX20180427-5959/
➜ make
➜ sudo make install
➜ sudo modprobe 88x2bu
➜ sudo lsmod | grep 88x2bu
➜ sudo reboot
```

### 配置网卡
- 添加一块 host-only 的网卡 `eth1` ，用于我们ssh登陆 虚拟机使用
<img src="https://img1.kiosk007.top/static/images/network/VMTrafficShark/eth1.png
" style="height:350px">
右键虚拟机tab，`settings` -> `Add` -> `Network Adapter` -> `Host-only`

- 修改网络内核参数

``` bash
# 默认开启端口转发
➜ sudo echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
# 修改网络接口命名方式（默认为ens33规则，修改为eth0、wlan0规则，与上面对应）
➜ sudo vim /etc/default/grub
# Modify GRUB_CMDLINE_LINUX
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
sudo grub-mkconfig -o /boot/grub/grub.cfg
➜ sudo reboot
```
> 重启后就ssh不上了...  另外注意 eth0 必须是桥接模式，nat的话会导致tcp option丢失。

- 配置网卡

ubuntu从17.10开始，已放弃在/etc/network/interfaces里固定IP的配置，即使配置也不会生效，而是改成netplan方式

refer: https://netplan.io/examples

``` bash
➜ sudo vim /etc/netplan/50-cloud-init.yaml 
network:
    ethernets:
        eth0:
            dhcp4: true
        eth1:
            dhcp4: true
        wlan0:
            dhcp4: no
            dhcp6: no
            addresses: [192.168.100.1/24]
    version: 2
    
➜ sodu netplan apply
```

- 配置完成

```
➜ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:1d:c7:31 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.104/24 brd 192.168.0.255 scope global dynamic eth0
       valid_lft 6009sec preferred_lft 6009sec
    inet6 fe80::20c:29ff:fe1d:c731/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:1d:c7:3b brd ff:ff:ff:ff:ff:ff
    inet 172.16.246.128/24 brd 172.16.246.255 scope global dynamic eth1
       valid_lft 1510sec preferred_lft 1510sec
    inet6 fe80::20c:29ff:fe1d:c73b/64 scope link 
       valid_lft forever preferred_lft forever
4: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 20:0d:b0:38:d2:ea brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/24 brd 192.168.100.255 scope global wlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::220d:b0ff:fe38:d2ea/64 scope link 
       valid_lft forever preferred_lft forever


```

### 安装dhcp服务

``` bash
➜ sudo apt install isc-dhcp-server -y
# 备份
➜ sudo cp /etc/default/isc-dhcp-server{,.bak} 
# 更改DHCP生效网卡
➜ sudo vim /etc/default/isc-dhcp-server
DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
INTERFACESv4="wlan0"
# 更改dhcp配置文件
➜ sudo vim /etc/dhcp/dhcpd.cnf
subnet 192.168.100.0 netmask 255.255.255.0 {
    range 192.168.100.100 192.168.100.200;
    option domain-name-servers 192.168.100.1;
    option routers 192.168.100.1;
}
➜ sudo systemctl restart isc-dhcp-server
➜
```

### 安装hostapd服务
``` bash
➜ sudo apt install hostapd -y
➜ sudo sed -i 's|#DAEMON_CONF=""|DAEMON_CONF="/etc/hostapd/hostapd.conf"|' /etc/default/hostapd
➜ sudo vim /etc/hostapd/hostapd.conf
interface=wlan0
driver=nl80211
ssid=Traffic Shark
hw_mode=g
ieee80211n=1
wmm_enabled=1
channel=7
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=12345678
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP 
➜ sudo systemctl unmask hostapd
➜ sudo systemctl enable hostapd
➜ sudo systemctl start hostapd
```

### 安装dnsmasq

由于ubuntu 18.04 开始，多了一个系统的systemd-resolvd服务，与我们要安装的dnsmasq都监听在 53 端口，妨碍了我们启用dnsmasq，所以需要停掉。

- 停止 systemd-resolved

``` bash
➜ sudo systemctl disable systemd-resolved
➜ sudo systemctl stop systemd-resolved
➜ ll /etc/resolv.conf 
lrwxrwxrwx 1 root root 39 Oct 17  2019 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
➜ rm -f /etc/resolv.conf 
➜ sudo tee /etc/resolv.conf <<EOF
nameserver 127.0.0.1
nameserver 114.114.114.114
EOF

```

- 安装dnsmasq

``` bash
➜ sudo apt-get install dnsmasq
➜ touch /etc/hosts.chat.freenode.net
➜ vim /etc/dnsmasq.conf
interface=wlan0
listen-address=192.168.100.1
dhcp-range=192.168.100.100,192.168.100.199,255.255.255.0,24h

address=/www.kiosk007.top/127.0.0.1
addn-hosts=/etc/hosts.chat.freenode.net
➜ systemctl enable dnsmasq
➜ systemctl start dnsmasq
```

### 安装中间人

``` bash
➜ wget https://snapshots.mitmproxy.org/5.1.1/mitmproxy-5.1.1-linux.tar.gz
➜ tar -xf mitmproxy-5.1.1-linux.tar.gz
```

### 安装Nginx

安装Nginx是为了模拟服务端故障和7层劫持，如果配置了证书同样可以对https进行劫持

``` bash
➜ sudo apt install nginx -y
➜ sudo tee /etc/nginx/nginx.conf <<EOF
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;
events {
        worker_connections 768;
        multi_accept on;
}
http {
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; 
        ssl_prefer_server_ciphers on;
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
        gzip on;
        include /etc/nginx/conf.d/*.conf;
}
EOF
```

## 测试

使用 iptables 命令转发流量
``` bash
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -F
iptables -t nat -F
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
```

- 手机搜索 AP “Traffic Shark”
- 密码 12345678

<img src="https://img1.kiosk007.top/static/images/network/VMTrafficShark/wifi.png" style="height:550px">


下行带宽可以达到93.35Mbps
上行带宽可以达到30.50Mbps （主要还得看物理网卡和运营商猛不猛）
<img src="https://img1.kiosk007.top/static/images/network/VMTrafficShark/traffic_test.png" style="height:550px">

## 弱网模拟使用
- 弱网模拟使用的是tc
- DNS劫持使用的dnsmasq
- 应用层劫持使用的Nginx和iptables
- 中间人使用的 mitmproxy

### 上行限速
``` bash
# Uplink
sudo tc qdisc del dev eth0 root
sudo tc qdisc add dev eth0 root handle 1: htb default 2
sudo tc class add dev eth0 parent 1:1 classid 1:2 htb rate 100mbit
sudo tc class add dev eth0 parent 1:1 classid 1:3 htb rate 100mbit
sudo tc qdisc add dev eth0 parent 1:2 handle 2: sfq perturb 10
sudo tc qdisc add dev eth0 parent 1:3 handle 3: netem rate 2mbit
sudo tc filter add dev eth0 protocol ip parent 1:0 u32 match ip src 192.168.100.0/24 flowid 1:3
```

### 下行限速
```bash
# Downlink
sudo tc qdisc del dev wlan0 root
sudo tc qdisc add dev wlan0 root handle 1: htb default 2
sudo tc class add dev wlan0 parent 1:1 classid 1:2 htb rate 100mbit
sudo tc class add dev wlan0 parent 1:1 classid 1:3 htb rate 100mbit
sudo tc qdisc add dev wlan0 parent 1:2 handle 2: sfq perturb 10
sudo tc qdisc add dev wlan0 parent 1:3 handle 3: netem rate 20mbit
sudo tc filter add dev wlan0 protocol ip parent 1:0 u32 match ip dst 192.168.100.0/24 flowid 1:3
```

### 模拟丢包、高时延等弱网环境
``` bash
sudo tc qdisc del dev wlan0 root
sudo tc qdisc add dev wlan0 root handle 1: htb default 2
sudo tc class add dev wlan0 parent 1:1 classid 1:2 htb rate 100mbps
sudo tc class add dev wlan0 parent 1:1 classid 1:3 htb rate 100mbps
sudo tc qdisc add dev wlan0 parent 1:2 handle 2: netem rate 3000kbit
sudo tc qdisc add dev wlan0 parent 1:3 handle 3: netem loss random 50%
sudo tc filter add dev wlan0 protocol ip parent 1:0 u32 match ip dst 192.168.0.0/24 flowid 1:2  # ip 是 eth0 的出网ip
```

### 封禁DNS请求
``` bash
sudo tc qdisc del dev wlan0 root
sudo tc qdisc add dev wlan0 root handle 1: htb default 2
sudo tc class add dev wlan0 parent 1: classid 1:2 htb rate 100mbps
sudo tc class add dev wlan0 parent 1: classid 1:3 htb rate 100mbps
sudo tc qdisc add dev wlan0 parent 1:2 handle 2: sfq perturb 10
sudo tc qdisc add dev wlan0 parent 1:3 handle 3: netem loss 100%
sudo tc filter add dev wlan0 protocol ip parent 1: prio 1 u32 match ip protocol 17 ff match ip sport 53 0xffff flowid 1:3
```

### 封禁IP段
``` bash
sudo tc qdisc del dev wlan0 root
sudo tc qdisc add dev wlan0 root handle 1: htb default 2
sudo tc class add dev wlan0 parent 1: classid 1:2 htb rate 100mbps
sudo tc class add dev wlan0 parent 1: classid 1:3 htb rate 100mbps
sudo tc qdisc add dev wlan0 parent 1:2 handle 2: sfq perturb 10
sudo tc qdisc add dev wlan0 parent 1:3 handle 3: netem loss 100%
sudo tc filter add dev wlan0 protocol ip parent 1: u32 match ip src 8.8.0.0/16 flowid 1:3
```

### 组合弱网场景
``` bash
sudo tc qdisc del dev wlan0 root
sudo tc qdisc add dev wlan0 root handle 1: htb default 2
sudo tc class add dev wlan0 parent 1:1 classid 1:2 htb rate 100mbps
sudo tc class add dev wlan0 parent 1:1 classid 1:3 htb rate 100mbps
sudo tc class add dev wlan0 parent 1:1 classid 1:4 htb rate 100mbps
sudo tc qdisc add dev wlan0 parent 1:2 handle 2:  sfq perturb 10
sudo tc qdisc add dev wlan0 parent 1:3 handle 3: netem loss random 1% rate 20mbps
sudo tc qdisc add dev wlan0 parent 1:4 handle 4: netem loss 100%
sudo tc filter add dev wlan0 protocol ip parent 1:0 u32 match ip dst 192.168.100.0/24 flowid 1:3  # 限制下行带宽
sudo tc filter add dev wlan0 protocol ip parent 1: prio 1 u32 match ip protocol 17 ff match ip sport 443 0xffff flowid 1:4  #封禁 udp 443 端口 
```

### dns 劫持
``` bash
➜ vim /etc/dnsmasq.conf
address=/www.kiosk007.top/127.0.0.1

如果想要解析多个ip
➜ vim /etc/hosts.chat.freenode.net
127.0.0.1 www.kiosk007.top
127.0.0.2 www.kiosk007.top

```

### 7层内容劫持

- 1、利用 DNS 劫持先将流量劫持到 Nginx 上
- 2、自签名泛域名证书，参考 https://github.com/Fishdrowned/ssl
- 3、配置Nginx完成劫持

### mitm中间人
``` bash
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables  -t nat -F
iptables  -t nat -A PREROUTING -i wlan0 -p tcp -j REDIRECT --to-port 8080
ip6tables -t nat -A PREROUTING -i wlan0 -p tcp --dport 443 -j REDIRECT --to-port 8080
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
SSLKEYLOGFILE="/root/sslkey/sslkeylogfile.txt" mitmproxy --mode transparent --showhost

```

<img src="https://img1.kiosk007.top/static/images/network/VMTrafficShark/mitmproxy.png">


### 抓包
为了能将抓到的包在宿主机上实时显示，需要配置root用户ssh免密登陆,配合上面的mitm导出中间人密钥，再将sslkey_log所在目录挂载到宿主机导入wireshark中，可以做到实时查看tls内容。

``` bash
➜ vim /etc/ssh/sshd_config
PermitRootLogin yes

➜ ssh root@172.16.246.128 "tcpdump -i wlan0 -U -w - 2>/dev/null"| wireshark -k -S -i -

```
