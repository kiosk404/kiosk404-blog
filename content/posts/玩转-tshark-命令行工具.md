---
title: 玩转 tshark 命令行工具
author: kiosk
tags:
  - wireshark
categories:
  - network
date: 2020-11-07 22:26:00
---
玩转TShark（Wireshark的命令行版）
wireshark 是一个伟大的网络问题分析工具，当然它也是有终端命令行工具的。**tshark**就是wireshark的命令行之一。WireShark的功能基本都有，还能组合grep/awk等编程处理分析抓包文件。

<!--more-->
# wireshark 自带命令集

Wireshark除了能够手动的分析报文之外，还额外的提供了几个命令行工具，方便开发者日常的报文处理需求，比如批量的合并以及编辑报文。这几个命令都是安装wireshark之后能够直接使用的，同时有的有对应的wireshark GUI 的操作。这些命令分别有`tshark`、`tcpdump`、`capinfos`、`dumpcap`、`text2cap`、`editcap`、`reordercap`、`rawshark`、`mergecap`、`pcapfix(需单独安装)`。

- tshark：基于终端的Wireshark
- tcpdump：使用“tcpdump”捕获以便使用Wireshark查看
- dumpcap：捕获“dumpcap”以便使用Wireshark查看
- capinfos：打印有关捕获文件的信息
- rawshark：转储和分析网络流量。
- editcap：编辑捕获文件
- mergecap：将多个捕获文件合并为一个
- text2pcap：将ASCII hexdumps转换为网络捕获
- reordercap：重新排序捕获文件
- pcapfix: 修复pcap文件

# pcap

## 认识 pcap

尝试查找网络问题的根源时，有助于查看可能是症状的数据包。为了查看这些数据包，必须首先捕获它们。

Wireshark默认的存储方式是pcap格式，最新版本的wireshark默认存储方式是pcapng。ng是next generation 的缩写，pcap和pcapng格式文件有是存在一定的差异。

pcap报文文件结构示意图
![pcap_format](https://img1.kiosk007.top/static/images/tshark/pcap_file_format.png)

1. Global Header是整个文件的文件头，包含文件格式标识，pcap格式版本号等文件指示信息。
2. Packet Header是每一片数据报文的头部信息，这些信息都是在形成pcap 报文的过程中由抓包软件wireshark添加的额外信息，例如报文捕获时间等。
3. Packet Data是抓取通信过程中的实际数据，包括协议数据和内容数据。

Wireshark 中对 Global Header 的定义

``` c
typedef struct pcap_hdr_s {
        guint32 magic_number;   /* magic number */
        guint16 version_major;  /* major version number */
        guint16 version_minor;  /* minor version number */
        gint32  thiszone;       /* GMT to local correction */
        guint32 sigfigs;        /* accuracy of timestamps */
        guint32 snaplen;        /* max length of captured packets, in octets */
        guint32 network;        /* data link type */
} pcap_hdr_t;

```

Wireshark 中对 Packet Header 的定义

``` c
typedef struct pcaprec_hdr_s {
        guint32 ts_sec;         /* timestamp seconds */
        guint32 ts_usec;        /* timestamp microseconds */
        guint32 incl_len;       /* number of octets of packet saved in file */
        guint32 orig_len;       /* actual length of packet */
} pcaprec_hdr_t;

```

## 获取 pcap

- **服务器** :推荐使用 tcpdump or tshark (tshark --color 可以染色哦) 命令行
- **个人电脑**: 推荐使用 wireshark 
- **Android 移动设备**: 推荐使用 [Pcap Remote](https://github.com/egorovandreyrm/pcap-remote) (需要在 Google Play 下载) 
- **iOS 移动设备**: 推荐使用 `rvictl -s [设备udid]` 方式抓包，参考 [Wireshark 抓包iOS设备](https://www.jianshu.com/p/c67baf5fce6d)

更多获取 pcap 方法参考 [捕获pcap](https://tshark.dev/capture/)


# tshark 

[tshark官方文档](https://www.wireshark.org/docs/man-pages/tshark.html)

## 介绍

TShark是一个网络分析工具。它能帮你在实时网络中捕获数据包，或是从预先保存好的捕获文件中读取数据包，或是打印出这些数据包的解码形式到标准输出，再或是把数据包写入到一个文件中。TShark的本地捕获文件格式是pcapng格式，这种pcapng格式也被wireshark和多种其他工具使用。


## 参数
- [-o ${key:value} ](https://tshark.dev/packetcraft/arcana/profiles/#-o-keyvalue) : 配置首选项中的settings, 一般可配置显示时间戳、是否相关seq number，或者TLS、WPA 解密等。

<img src="/images/tshark/tshark_o.png" style="height:200px" />

- [-z ${statistics} ](https://www.wireshark.org/docs/man-pages/tshark.html) : 集各种类型的统计信息, 这里面有一些信息是很有用的，如专家信息、时序统计飞行中的报文、时序丢包都需要这个参数来

<img src="https://img1.kiosk007.top/static/images/tshark/tshark_z.png" style="height:350px" >

**其余参数如下**

``` bash
# 抓包接口类
-i 设置抓包的网络接口，不设置则默认为第一个非自环接口。
-D 列出当前存在的网络接口。在不了解OS所控制的网络设备时，一般先用“tshark -D”查看网络接口的编号以供-i参数使用。
-f 设定抓包过滤表达式（capture filter expression）。抓包过滤表达式的写法雷同于tcpdump
-s 设置每个抓包的大小，默认为65535，多于这个大小的数据将不会被程序记入内存、写入文件
-p 设置网络接口以非混合模式工作，即只关心和本机有关的流量

# 停止抓包参数
-c 抓取的packet数，在处理一定数量的packet后，停止抓取，程序退出。
-a 设置tshark抓包停止向文件书写的条件，事实上是tshark在正常启动之后停止工作并返回的条件。条件写为test:value的形式，如“-a duration:5”表示tshark启动后在5秒内抓包然后停止；“-a filesize:10”表示tshark在输出文件达到10kB后停止；“-a files:n”表示tshark在写满n个文件后停止。

# 处理类
-R 设置读取（显示）过滤表达式（read filter expression）。不符合此表达式的流量同样不会被写入文件。
-n 禁止所有地址名字解析（默认为允许所有）。
-N 启用某一层的地址名字解析。“m”代表MAC层，“n”代表网络层，“t”代表传输层，“C”代表当前异步DNS查找。如果-n和-N参数同时存在，-n将被忽略。如果-n和-N参数都不写，则默认打开所有地址名字解析。
-d 将指定的数据按有关协议解包输出。如要将tcp 8888端口的流量按http解包，应该写为“-d tcp.port==8888,http”。注意选择子和解包协议之间不能留空格。

# 输出类
-w 设置raw数据的输出文件。这个参数不设置，tshark将会把解码结果输出到stdout。“-w-”表示把raw输出到stdout。如果要把解码结果输出到文件，使用重定向“>”而不要-w参数。
-F 设置输出raw数据的格式，默认为libpcap。“tshark -F”会列出所有支持的raw格式。
-V 设置将解码结果的细节输出，否则解码结果仅显示一个packet一行的summary。
-x 设置在解码输出结果中，每个packet后面以HEX dump的方式显示具体数据。
-T 设置解码结果输出的格式，包括text,ps,psml和pdml，默认为text。
-E: -E <fieldsoption>=<value>如果-T fields选项指定，使用-E来设置一些属性，比如
　　　　header=y|n
　　　　separator=/t|/s|<char>
　　　　occurrence=f|l|a
　　　　aggregator=,|/s|<char>
-t 设置解码结果的时间格式。“ad”表示带日期的绝对时间，“a”表示不带日期的绝对时间，“r”表示从第一个包到现在的相对时间，“d”表示两个相邻包之间的增量时间（delta）。
-S 在向raw文件输出的同时，将解码结果打印到控制台。
-l 在处理每个包时即时刷新输出。
-X 扩展项。
-q 设置安静的stdout输出（例如做统计时）
-z 设置统计参数。

# 其他
-h 显示命令行帮助。
-v 显示tshark的版本信息。
-o 重载选项
```


## 利用 tshark 打印pcap

我们通过tcpdump或者wireshark抓到 pcap 文件，接下来就可以利用 `tshark` 这个强大的命令行工具进行抓包。其中 `-o `的几个选项可以指定 ssl 解密（需要sslkeylog），`-T` 指定输出格式，`-e` 指定都需要输出哪些字段（字段列表参考 https://www.wireshark.org/docs/dfref/）。输出是将每个Package中指定的字段输出。
``` bash
tshark -o ssl.keylog_file:./sslkeylog.txt \
-o "ssl.desegment_ssl_records: TRUE" \
-o "ssl.desegment_ssl_application_data: TRUE" \
-e ssl.handshake.ciphersuite \  
-e tcp.analysis.zero_window \
-e http.host \
-e dns.time \
-e tcp.flags.urg \
-e http.request.line \
-e dns.qry.name \
-e ip.version \
-e tcp.analysis.window_full \
-e ipv6.dst \
-e http.request.version \
-e udp.dstport \
-e dns.flags.response \
.... \
-T json -r "./capturedump.pcap"
```

其余更多的信息

``` bash
//打印http协议流相关信息
tshark -s 512 -i eth0 -n -f 'tcp dst port 80' -Y 'http.host and http.request.uri' -T fields -e http.host -e http.request.uri -l

//实时打印当前mysql查询语句
tshark -s 512 -i eth0 -n -f 'tcp dst port 3306' -Y 'mysql.query' -T fields -e mysql.query

//解析MySQL协议
tshark -r ./mysql-compress.cap -o tcp.calculate_timestamps:true \
-T fields -e mysql.caps.cp -e frame.number \
-e frame.time_epoch  -e frame.time_delta_displayed  \
-e ip.src -e tcp.srcport -e tcp.dstport -e ip.dst \
-e tcp.time_delta -e frame.time_delta_displayed \
-e tcp.stream -e tcp.len -e mysql.query

//抓取详细SQL语句, 快速确认client发过来的具体SQL内容：
sudo tshark -i any -f 'port 3306' -s 0 -l -w - |strings
sudo tshark -i eth0 -d tcp.port==3306,mysql -T fields -e mysql.query 'port 3306'
sudo tshark -i eth0 -Y "ip.addr==11.163.182.137" -d tcp.port==3306,mysql -T fields -e mysql.query 'port 3306'
sudo tshark -i eth0 -Y "tcp.srcport==62877" -d tcp.port==3001,mysql -T fields -e tcp.srcport -e mysql.query 'port 3001'
```


## 查看请求目标信息


``` bash
$ tshark -r wechat.pcap -o 'gui.column.format:"Source Net Addr","%uns","Dest Net Addr", "%und"' -Y "ip" | sort | uniq
111.206.101.25 → 192.168.100.115
111.206.4.92 → 192.168.100.115
119.167.215.208 → 192.168.100.115
122.14.230.129 → 192.168.100.115
123.125.102.19 → 192.168.100.115
192.168.100.115 → 111.161.111.119
192.168.100.115 → 111.206.101.25
192.168.100.115 → 111.206.4.92
192.168.100.115 → 119.167.215.208
```

## 查看丢包、带宽、吞吐、延迟

``` bash
// 统计 ip 情况
$ tshark -r wechat.pcap -q -z conv,ip

// 跟踪一条流打印 16进制数据
$ tshark -r wechat.pcap -q -z "follow,tcp,hex,0" 
```
![conv](https://img1.kiosk007.top/static/images/tshark/tshark_conv.png)


**计算丢包情况**

``` bash
$ tshark -r wechat.pcap -q \
-z io,stat,1,"COUNT(tcp.analysis.retransmission) tcp.analysis.retransmission","COUNT(tcp.analysis.fast_retransmission) tcp.analysis.fast_retransmission","COUNT(tcp.analysis.duplicate_ack) tcp.analysis.duplicate_ack","COUNT(tcp.analysis.lost_segment) tcp.analysis.lost_segment"
```

**计算上行带宽**
``` bash
$ tshark -o tcp.desegment_tcp_streams:TRUE -n \
-q -r  ./wechat.pcap \
-z io,stat,99999999,"BYTES()(ip.src!=10.0.0.0/8 and ip.src!=172.16.0.0/12 and ip.src!=192.168.0.0/16)"
```

**计算 TCP RTT**

``` bash
$ tshark -o tcp.desegment_tcp_streams:TRUE -n \
-q -r  ./wechat.pcap  -z \
io,stat,99999999,"AVG(tcp.analysis.ack_rtt)tcp.analysis.ack_rtt"
```



**分析 `tcp.stream` id 为 0 的传输带宽 。**

```
$ tshark -o tcp.desegment_tcp_streams:FALSE -n \
-q -r wechat.pcapng -z io,stat,1,"BYTES()tcp.stream==0"

===============================
| IO Statistics               |
|                             |
| Duration: 50.712225 secs    |
| Interval:  1 secs           |
|                             |
| Col 1: BYTES()tcp.stream==0 |
|-----------------------------|
|          |1       |         |
| Interval |  BYTES |         |
|-------------------|         |
|  0 <>  1 |      0 |         |
|  1 <>  2 |      0 |         |
...
|  8 <>  9 | 128301 |         |
|  9 <> 10 | 127132 |         |
| 10 <> 11 | 128667 |         |
| 11 <> 12 | 127749 |         |

...
```
wireshark 的TCP流图中的吞吐量

<img src="https://img1.kiosk007.top/static/images/tshark/wireshark_throughput.png" style="height:300px" />

对比通过 tshark的 io graph 绘出的图。

<img src="https://img1.kiosk007.top/static/images/tshark/tshark_throughput.png" style="weight:300px" />


同理可以绘 丢包重传、飞行中的报文等。

对于排查网络延时/应用问题有一些过滤条件是非常有用的：

- `tcp.analysis.lost_segment`：表明已经在抓包中看到不连续的序列号。报文丢失会造成重复的ACK，这会导致重传。
- `tcp.analysis.duplicate_ack`：显示被确认过不止一次的报文。大量的重复ACK是TCP端点之间高延时的迹象。
- `tcp.analysis.retransmission`：显示抓包中的所有重传。如果重传次数不多的话还是正常的，过多重传可能有问题。这通常意味着应用性能缓慢和/或用户报文丢失。
- `tcp.analysis.window_update`：将传输过程中的TCP window大小图形化。如果看到窗口大小下降为零，这意味着发送方已经退出了，并等待接收方确认所有已传送数据。这可能表明接收端已经不堪重负了。
- `tcp.analysis.bytes_in_flight`：某一时间点网络上未确认字节数。未确认字节数不能超过你的TCP窗口大小（定义于最初3此TCP握手），为了最大化吞吐量你想要获得尽可能接近TCP窗口大小。如果看到连续低于TCP窗口大小，可能意味着报文丢失或路径上其他影响吞吐量的问题。
- `tcp.analysis.ack_rtt`：衡量抓取的TCP报文与相应的ACK。如果这一时间间隔比较长那可能表示某种类型的网络延时（报文丢失，拥塞，等等）。

## 专家信息
可统计重传、TLS 会话复用、HTTP 会话、TCP RST 等等。功能强大

```
tshark -r refresh_video.pcap -q -z expert

```

<img src="https://img1.kiosk007.top/static/images/tshark/tshark_expert.png" />


## 高阶使用

- 分析SQL查询的时间分布

``` bash
tshark -r gege_drds.pcap \
-Y "mysql.query or (tcp.srcport==3306  and tcp.len>60)" \
-o tcp.calculate_timestamps:true \
-T fields \
-e frame.number -e frame.time_epoch  \
-e frame.time_delta_displayed  \
-e ip.src -e tcp.srcport -e tcp.dstport -e ip.dst \
-e tcp.time_delta -e tcp.stream -e tcp.len \
| awk 'BEGIN { \
sum0=0;sum3=0;sum10=0;sum30=0;sum50=0; \
sum100=0;sum300=0;sum500=0;sum1000=0;\
sumo=0;count=0;sum=0
} \
{ \
rt=$8; \
if(rt>=0.000) sum=sum+rt; count=count+1; \
if(rt<=0.000) sum0=sum0+1; \
else if(rt<0.003) sum3=sum3+1 ;\
else if(rt<0.01) sum10=sum10+1; \
else if(rt<0.03) sum30=sum30+1; \
else if(rt<0.05) sum50=sum50+1; \
else if(rt < 0.1) sum100=sum100+1; \
else if(rt < 0.3) sum300=sum300+1; \
else if(rt < 0.5) sum500=sum500+1; \
else if(rt < 1) sum1000=sum1000+1; \
else sum=sum+1 ; \
} \
END { printf "-------------\n3ms:\t%s \n10ms:\t%s \n30ms:\t%s \n50ms:\t%s \n100ms:\t%s \n300ms:\t%s \n500ms:\t%s \n1000ms:\t%s \n>1s:\t %s\n-------------\navg: %.6f \n" , sum3,sum10,sum30,sum50,sum100,sum300,sum500,sum1000,sumo,sum/count;}'

 -------------
3ms:    145037 
10ms:    78811 
30ms:    7032 
50ms:    2172 
100ms:    1219 
300ms:    856 
500ms:    449 
1000ms:118
>1s:    0
-------------
avg: 0.005937 

```

- 分析每个包的response time

``` bash
$ tshark -r rsb2.cap -o tcp.calculate_timestamps:true \
-T fields -e frame.number -e frame.time_epoch \
-e ip.src -e ip.dst -e tcp.stream -e tcp.len \
-e tcp.analysis.initial_rtt -e tcp.time_delta

1481	1465269331.308138000	100.98.199.36	10.25.92.13	302	0		0.002276000
1482	1465269331.308186000	10.25.92.13	    100.98.199.36	361	11	0.000063000
1483	1465269331.308209000	100.98.199.36	10.25.92.13	496	0		0.004950000
1484	1465269331.308223000	100.98.199.36	10.25.92.13	513	0		0.000000000
1485	1465269331.308238000	100.98.199.36	10.25.92.13	326	0		0.055424000
1486	1465269331.308246000	100.98.199.36	10.25.92.13	514	0		0.000000000
1487	1465269331.308261000	10.25.92.71	    10.25.92.13	48	0		0.000229000
1488	1465269331.308277000	100.98.199.36	10.25.92.13	254	0		0.055514000
```

- 分析rtt时间

``` bash
$ tshark -r wechat.pcapng -q -z \
io,stat,5,"MIN(tcp.analysis.ack_rtt)tcp.analysis.ack_rtt",\
"MAX(tcp.analysis.ack_rtt)tcp.analysis.ack_rtt",\
"AVG(tcp.analysis.ack_rtt)tcp.analysis.ack_rtt"

========================================================
| IO Statistics                                        |
|                                                      |
| Duration: 50.712225 secs                             |
| Interval:  5 secs                                    |
|                                                      |
| Col 1: MIN(tcp.analysis.ack_rtt)tcp.analysis.ack_rtt |
|     2: MAX(tcp.analysis.ack_rtt)tcp.analysis.ack_rtt |
|     3: AVG(tcp.analysis.ack_rtt)tcp.analysis.ack_rtt |
|------------------------------------------------------|
|          |1         |2         |3         |          |
| Interval |    MIN   |    MAX   |    AVG   |          |
|-------------------------------------------|          |
|  0 <>  5 | 0.000000 | 0.000000 | 0.000000 |          |
|  5 <> 10 | 0.000987 | 0.358817 | 0.293383 |          |
| 10 <> 15 | 0.001537 | 1.125008 | 0.336217 |          |
| 15 <> 20 | 0.001598 | 0.745323 | 0.632126 |          |
| 20 <> 25 | 0.002196 | 1.454920 | 0.584168 |          |
| 25 <> 30 | 0.002674 | 0.892343 | 0.771408 |          |
| 30 <> 35 | 0.001505 | 1.406873 | 1.066937 |          |
| 35 <> 40 | 0.001333 | 1.372204 | 1.267557 |          |
| 40 <> 45 | 0.001366 | 1.410311 | 1.204430 |          |
| 45 <> 50 | 0.001513 | 1.360609 | 1.008420 |          |
| 50 <> Dur| 0.001609 | 1.378431 | 0.956597 |          |
========================================================


```


# tshark 抓包

最后让我们来用伟大的tshark抓包吧，快放弃古老的 tcpdump。

执行 `sudo tshark -Y 'ip.addr == 8.8.8.8' --color`
会在终端以wireshark的风格开始抓包。
![](https://img1.kiosk007.top/static/images/tshark/tshark_capture.png)



参考：
- [就是让你懂抓包--WireShark之命令行版tshark](https://plantegg.github.io/2019/06/21/%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82%E6%8A%93%E5%8C%85--WireShark%E4%B9%8B%E5%91%BD%E4%BB%A4%E8%A1%8C%E7%89%88tshark/)