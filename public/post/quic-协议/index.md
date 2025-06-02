# 认识 QUIC 协议

HTTP 3 ，它来了，[QUIC（quick udp internet connection “快速 UDP 互联网连接”）](https://www.fastvue.co/fastvue/blog/googles-quic-protocols-security-and-reporting-implications/)正如其名一样，它就是快。其正在标准化为新一代的互联网传输协议。是由google提出的使用udp进行多路并发的传输协议。

如果想要网站在弱网情况下提升网络访问速度，那么QUIC将是一个不错的选择。
<!--more-->


<img src="https://img1.kiosk007.top/static/images/network/QUIC/gquic-stack.png" style="width:600px;height:400px" />

## QUIC 概述
QUIC解决了什么问题呢？从上世纪90年代至今，互联网一直按照一成不变的模式发展着。使用ipv4进行路由，使用tcp进行连接层面的拥塞控制，并保证其传输的可靠性，使用tls层进行安全加密和身份验证，使用http进行应用的数据传输。
这么多年的发展，这些协议只是小部分或者细节上有了改变，tcp提出了几个新的拥塞控制算法，tls改变了加密方式，http层进化出了h2。但是互联网发展至今，网络传输的内容越来越大，用户对传输的时延，带宽提出越来越大的要求。
tcp虽然也在拥塞控制上提出了一些优秀的拥塞控制算法，如BBR但是限制于其对操作系统内核版本的要求，windows操作系统又支持不好等。导致想要切换成更高效的协议成本巨大。

这里列出几个主要的矛盾。
1. 协议历史悠久导致中间设备僵化。
2. 依赖于操作系统的实现导致协议本身僵化。
3. 建立连接的握手延迟大。
4. 队头阻塞。

QUIC孕育而生，其抛开了TCP直接采用UDP，如一些拥塞算法，可靠性保证机制，不再依赖操作系统内核，而是可以自定义。
QUIC 相比现在广泛应用的 http2+tcp+tls 协议有如下优势：
1. 快速（0-RTT）建连，类似于TLS的false start和TCP的TFO。
2. 集成的拥塞控制算法，可插拔。
3. 高安全性，类似于TLS，QUIC的报文始终被加密和认证。 
4. 避免队头阻塞的多路复用。
5. 基于Connection ID 的连接迁移，减少与移动端的重复建联。
6. 前向冗余纠错（using FEC – Forward Error Correction）。

### 现有问题
#### 中间设备僵化和操作系统老旧
有些防火墙只允许通过 80 和 443，不放通其他端口。因为通信协议栈都是固化到操作系统中的，只能通过内核参数进行调整，但是这样的调整极其有限，如果想要新加协议，只能重新编译内核。而现实是，可能还有一些Centos5 还作为某些古董公司的线上服务器。另外，windows xp 可能还是某些事业单位的办公电脑上装的操作系统，尽管xp的时代已经过去20年了。
#### 建立连接的握手延迟大
知道一个首次https网站的访问都要有哪些步骤吗？dns解析需要1个RTT，TCP三次握手，HTTP 302 跳转 HTTPS又需要RTT，TCP重新握手。TLS握手再消耗2个。解析CA的DNS（因为浏览器获取到证书后，有可能需要发起 OCSP 或者 CRL 请求，查询证书状态）再对CA进行TCP握手，OCSP响应。密钥协商又是RTT。当然这种情况是最极端的，大部分情况下不是所有流程都需要走一遍的。

More info: [大型网站的 HTTPS 实践（二）-- HTTPS 对性能的影响](https://developer.baidu.com/resources/online/doc/security/https-pratice-2.html)
#### 队头阻塞
- `HTTP1.1的队头阻塞` HTTP1.1 是一发一收，也就是第一个数据没响应之前不能发第二个请求。
- `TCP的队头阻塞` 一个数据包丢了，后面的数据就算都收到了，内核也不会将缓冲区的数据交给应用层。所以基于TCP的HTTP2 在处理4层的队头阻塞问题上也是无能为力。
- `TLS的队头阻塞` TLS 协议都是按照 record 来处理数据的，如果一个 record 中丢失了数据，也会导致整个 record 无法正确处理。这个目前也没有很好的解决方法。

基于以上的原因，QUIC选择了UDP。没有了三次握手，在应用层面完成了传输的可靠性，拥塞控制还有TLS的安全性。只要应用软件的客户端和服务端支持就行，绕开了操作系统内核版本这个硬骨头。

## 协议解析
最早这一实验性协议由 Google 推出，并命名为 gQUIC，因此，IETF 草案中仍然保留了 QUIC 概念

- QUIC 层由https://tools.ietf.org/html/draft-ietf-quic-transport-29 描述，它定义了连接、报文的可靠传输、有序字节流的实现；
- TLS 协议会将 QUIC 层的部分报文头部暴露在明文中，方便代理服务器进行路由。https://tools.ietf.org/html/draft-ietf-quic-tls-29 规范定义了 QUIC 与 TLS 的结合方式；
- 丢包检测、RTO 重传定时器预估等功能由https://tools.ietf.org/html/draft-ietf-quic-recovery-29 定义，目前拥塞控制使用了类似TCP New RENO 的算法，未来有可能更换为基于带宽检测的算法（例如BBR）；
- 基于以上 3 个规范，https://tools.ietf.org/html/draft-ietf-quic-http-29 定义了 HTTP 语义的实现，包括服务器推送、请求响应的传输等；
- 在 HTTP/2 中，由 HPACK 规范定义 HTTP 头部的压缩算法。由于 HPACK 动态表的更新具有时序性，无法满足 HTTP/3 的要求。在 HTTP/3 中，QPACK 定义 HTTP 头部的编码：https://tools.ietf.org/html/draft-ietf-quic-qpack-16。

``` bash
UDP Header
Packet Header     -----
QUIC Frame Header    --  TLS1.3
HTTP3 Frame Header   -- 
HTTP Message      -----
```

为了实现 多路复用、链接迁移、非队头阻塞等特性，QUIC必须做到多层定义，另外层与层之间要做到非强依赖。
这 3 层 Header 实现的功能各不相同：
- Packet Header 实现了可靠的连接。当 UDP 报文丢失后，通过 Packet Header 中的 Packet Number 实现报文重传。连接也是通过其中的 Connection ID 字段定义的；
- QUIC Frame Header 不可跨越Packet，在无序的 Packet 报文中，基于 QUIC Stream 概念实现了有序的字节流，这允许 HTTP 消息可以像在 TCP 连接上一样传输；
- HTTP/3 Frame Header 可跨越多个Packet,定义了 HTTP Header、Body 的格式，以及服务器推送、QPACK 编解码流等功能。
- HTTP Message

### 如何实现连接迁移

Packet Header 可以细分为两种：Long Packet Header 用于首次建立连接；Short Packet Header 用于日常传输数据。

**Long Packet Header**
其中，Long Packet Header 的格式如下图所示：
![quic_longpacket](https://img1.kiosk007.top/static/images/network/QUIC/quic_longpacket.png)

![quic_longpacket_pcap](https://img1.kiosk007.top/static/images/network/QUIC/quic_longpacket_pcap.png)

建立连接时，连接是由服务器通过 Source Connection ID 字段分配的，这样，后续传输时，双方只需要固定住 Destination Connection ID，就可以在客户端 IP 地址、端口变化后，绕过 UDP 四元组（与 TCP 四元组相同），实现连接迁移功能。下面是 **Short Packet Header** 头部的格式，这里就不再需要传输 Source Connection ID 字段了：

![quic_shortpacket](https://img1.kiosk007.top/static/images/network/QUIC/quic_shortpacket.png)

![quic_longpacket_pcap](https://img1.kiosk007.top/static/images/network/QUIC/quic_shortpacket_pcap.png)


**Packet Number** 是每个报文独一无二的序号，基于它可以实现丢失报文的精准重发。**为了防范各类网络攻击 Packet Number 会被 TLS 层加密保护。自Packet以下基本实现了全加密，Packet层自己也实现了半加密（握手除外）**


### 如何实现多路复用

一个 Packet 报文中可以存放多个 QUIC Frame，所有 Frame 的长度之和不能大于 PMTUD（Path Maximum Transmission Unit Discovery，这是大于 1200 字节的值），你可以把它与 IP 路由中的 MTU 概念对照理解：

![quic_frame](https://img1.kiosk007.top/static/images/network/QUIC/quic_frame.png)

每一个 Frame 都有明确的类型：如PADDING、PING等
![quic_frame](https://img1.kiosk007.top/static/images/network/QUIC/quic_frame_type.png)

其中， **0x08-0x0f 这 8 种 STREAM 类型的 Frame 用于传递 HTTP 消息**，它的格式如下所示：

![quic_frame](https://img1.kiosk007.top/static/images/network/QUIC/quic_frame_stream.png)

可见，Stream Frame 头部的 3 个字段，完成了多路复用、有序字节流以及报文段层面的二进制分隔功能，包括：

- Stream ID 定义了一个有序字节流。当 HTTP Body 非常大，需要跨越多个 Packet 时，只要在每个 Stream Frame 中含有同样的 Stream ID，就可以传输任意长度的消息。多个并发传输的 HTTP 消息，通过不同的 Stream ID 加以区别；

- 消息序列化后的“有序”特性，是通过 Offset 字段完成的，它类似于 TCP 协议中的 Sequence 序号，用于实现 Stream 内多个 Frame 间的累计确认功能；

- Length 指明了 Frame 数据的长度。

0x08-0x0f 这 8 种类型其实是由 3 个二进制位组成，它们实现了以下 3 标志位的组合：

- 第 1 位表示是否含有 Offset，当它为 0 时，表示这是 Stream 中的起始 Frame，这也是上图中 Offset 是可选字段的原因；
- 第 2 位表示是否含有 Length 字段；
- 第 3 位 Fin，表示这是 Stream 中最后 1 个 Frame，与 HTTP/2 协议 Frame 帧中的 FIN 标志位相同。

Stream 数据中并不会直接存放 HTTP 消息，因为 HTTP/3 还需要实现服务器推送、权重优先级设定、流量控制等功能，所以 Stream Data 中首先存放了 **HTTP/3 Frame** (这里就相当于是HTTP2的FRAME了，如 HEADER 帧、DATA 帧 、 SETTINGS 帧等)：

![quic_http3](https://img1.kiosk007.top/static/images/network/QUIC/quic_http3.png)

其中，Length 指明了 HTTP 消息的长度，而 Type 字段（低 2 位在 QPACK 实现上有特殊用途）。QUIC Stream Frame 定义了有序字节流，且多个 Stream 间的传输没有时序性要求。这样，HTTP 消息基于 QUIC Stream 就实现了真正的多路复用，队头阻塞问题自然就被解决掉了。

### 如何阻塞队头阻塞

其实上面的多路复用已经解决了一大半的对头阻塞问题。但实际上在H2时代除了 TCP 、TLS 的队头阻塞还有一个 HPACK 的动态表阻塞。

所谓动态表，就是将未包含在静态表中的 Header 项，在其首次出现时加入动态表，这样后续传输时仅用 1 个数字表示，大大提升了编码效率。因此，动态表是天然具备时序性的，如果首次出现的请求出现了丢包，后续请求解码 HPACK 头部时，一定会被阻塞！

QPACK 将动态表的编码、解码独立在单向 Stream 中传输，仅当单向 Stream 中的动态表编码成功后，接收端才能解码双向 Stream 上 HTTP 消息里的动态表索引。

QUIC Stream Frame 中的 Stream ID 别有玄机，除了标识 Stream 外，它的低 2 位还可以表达以下组合：
![quic_stream](https://img1.kiosk007.top/static/images/network/QUIC/quic_stream_id.png)

当 Stream ID 是 0、4、8、12 时，这就是客户端发起的双向 Stream（HTTP/3 不支持服务器发起双向 Stream），它用于传输 HTTP 请求与响应。单向 Stream 有很多用途，所以它在数据前又多出一个 Stream Type 字段，当 Stream Type 为 
- 0x02：用于编码 QPACK 动态表，比如面对不属于静态表的 HTTP 请求头部，客户端可以通过这个 Stream 发送动态表编码；
- 0x03：用于通知编码端 QPACK 动态表的更新结果。

由于 HTTP/3 的 Stream 之间是乱序传输的，因此，若先发送的编码 Stream 后到达，双向 Stream 中的 QPACK 头部就无法解码，此时传输 HTTP 消息的双向 Stream 就会进入 Block 阻塞状态。 而单向流中的QUIC FRAME 会立马更新动态表内容。

> 另外可以看到，QUIC的每一层之间都实现了良好的隔离性，这就意味着其实QUIC可以作为一个传输协议，上面不仅可以跑 HTTP协议，还可以用来跑raw data。就是剥离 HTTP3 层以上的数据即可。


## QUIC核心特性
### 0RTT握手
![quic_0rtt](https://img1.kiosk007.top/static/images/network/QUIC/quic_0rtt.gif)

在HTTPS over TCP+TLS的时代。HTTPS需要3个RTT，在session 复用的情况下是2个RTT。而QUIC做到了1RTT和会话复用的0RTT。
QUIC的TLS只能使用TLS1.3，这就可以做到PSK的0RTT。
### 改进的拥塞控制
TCP 的拥塞控制实际上包含了四个算法：慢启动，拥塞避免，快速重传，快速恢复。其中TCP中拥塞控制是被编译进内核中的，如果想要更改就需要改变内核参数，但是想要对已有的拥塞控制算法进行更改就需要重新编译内核。QUIC用很多应用层实现的拥塞控制算法，想要修改是十分方便的。

### 无队头阻塞

TCP和TLS由于设计都会存在队头阻塞的情况。
- `TCP队头阻塞` 是由于传输的请求由于中间的报文没有收全，会一直重传，直到报文被重传成功，之后的数据才能接着传。
- `TLS队头阻塞` Record 是 TLS 协议处理的最小单位，最大不能超过 16K，Record由Encrypt和SSL Record Header组成， 一个Record可能是由多个TCP Segment组成，只要是Record中16KB有一个字节没有收到，整个TLS过程都无法完成。

![tls_record](https://img1.kiosk007.top/static/images/network/QUIC/tls_record.png)

- `HPACK队头阻塞` Hpack是H2中提出的一个压缩传输体积的算法，但是HPACK是一个有时间顺序的算法。（上面已解释）

基于UDP实现的QUIC，当丢包发生之后，不必等丢失的报文完全响应，可以将已经收到的响应交给上层处理。QUIC的解决思路如下
- QUIC 最基本的传输单元是 Packet，不会超过 MTU 的大小，整个加密和认证过程都是基于 Packet 的，不会跨越多个 Packet。这样就能避免 TLS 协议存在的队头阻塞。
- Stream 之间相互独立，比如 Stream2 丢了一个 Pakcet，不会影响 Stream3 和 Stream4。不存在 TCP 队头阻塞。这是基于QUIC 的Packet Header 中的Connection Id 实现的。


### 加密认证的报文
TCP 协议头部没有经过任何加密和认证，所以在传输过程中很容易被中间网络设备篡改，注入和窃听。比如修改序列号、滑动窗口。这些行为有可能是出于性能优化，也有可能是主动攻击。

但是 QUIC 的 packet 可以说是武装到了牙齿。除了个别报文比如 PUBLIC_RESET 和 CHLO，所有报文头部都是经过认证的，报文 Body 都是经过加密的。这样只要对 QUIC 报文任何修改，接收端都能够及时发现，有效地降低了安全风险。


### 基于 stream 和 connecton 级别的流量控制

QUIC 的流量控制类似 HTTP2，即在 Connection 和 Stream 级别提供了两种流量控制。
- Stream 可以认为就是一条 HTTP 请求。
- Connection 可以类比一条 TCP 连接。多路复用意味着在一条 Connetion 上会同时存在多条 Stream。既需要对单个 Stream 进行控制，又需要针对所有 Stream 进行总体控制。

QUIC 实现流量控制的原理比较简单：
通过 window_update 帧告诉对端自己可以接收的字节数，这样发送方就不会发送超过这个数量的数据。

通过 BlockFrame 告诉对端由于流量控制被阻塞了，无法发送数据。QUIC 的流量控制和 TCP 有点区别，TCP 为了保证可靠性，窗口左边沿向右滑动时的长度取决于已经确认的字节数。如果中间出现丢包，就算接收到了更大序号的 Segment，窗口也无法超过这个序列号。

但 QUIC 不同，就算此前有些 packet 没有接收到，它的滑动只取决于接收到的最大偏移字节数。

针对 QUIC Stream：
```
可用窗口数 = 最大窗口数 - 接收到的最大偏移数
```

针对 Connection：
```
可用窗口数 = Stream1窗口 + Stream2窗口 + Stream3窗口 + SteamN窗口
```


### 更灵活的拥塞控制算法

QUIC 协议当前默认使用了 TCP 协议的 Cubic 拥塞控制算法，但是可以灵活的支持 CubicBytes, Reno, RenoBytes, BBR, PCC 等拥塞控制算法

#### 单调递增的Package Number

TCP 为了保证可靠性，使用了基于字节序号的 Sequence Number 及 Ack 来确认消息的有序到达。

QUIC 同样是一个可靠的协议，它使用 Packet Number 代替了 TCP 的 sequence number，并且每个 Packet Number 都严格递增，也就是说就算 Packet N 丢失了，重传的 Packet N 的 Packet Number 已经不是 N，而是一个比 N 大的值。而 TCP 呢，重传 segment 的 sequence number 和原始的 segment 的 Sequence Number 保持不变，也正是由于这个特性，引入了 Tcp 重传的歧义问题。

<img src="https://img1.kiosk007.top/static/images/network/QUIC/tcp_rt.jpg" style="width:600px;height:200px">

如上图所示，超时事件 RTO 发生后，客户端发起重传，然后接收到了 Ack 数据。由于序列号一样，这个 Ack 数据到底是原始请求的响应还是重传请求的响应呢？不好判断。

如果算成原始请求的响应，但实际上是重传请求的响应（上图左），会导致采样 RTT 变大。如果算成重传请求的响应，但实际上是原始请求的响应，又很容易导致采样 RTT 过小。

由于 Quic 重传的 Packet 和原始 Packet 的 Pakcet Number 是严格递增的，所以很容易就解决了这个问题。
<img src="https://img1.kiosk007.top/static/images/network/QUIC/quic_rt.jpg" style="width:600px;height:200px">
如上图所示，RTO 发生后，根据重传的 Packet Number 就能确定精确的 RTT 计算。如果 Ack 的 Packet Number 是 N+M，就根据重传请求计算采样 RTT。如果 Ack 的 Pakcet Number 是 N，就根据原始请求的时间计算采样 RTT，没有歧义性。

但是单纯依靠严格递增的 Packet Number 肯定是无法保证数据的顺序性和可靠性。QUIC 又引入了一个 Stream Offset 的概念。

即一个 Stream 可以经过多个 Packet 传输，Packet Number 严格递增，没有依赖。但是 Packet 里的 Payload 如果是 Stream 的话，就需要依靠 Stream 的 Offset 来保证应用数据的顺序。如错误! 未找到引用源。所示，发送端先后发送了 Pakcet N 和 Pakcet N+1，Stream 的 Offset 分别是 x 和 x+y。

假设 Packet N 丢失了，发起重传，重传的 Packet Number 是 N+2，但是它的 Stream 的 Offset 依然是 x，这样就算 Packet N + 2 是后到的，依然可以将 Stream x 和 Stream x+y 按照顺序组织起来，交给应用程序处理。

#### 更多的 Ack 块

TCP 的 Sack 选项能够告诉发送方已经接收到的连续 Segment 的范围，方便发送方进行选择性重传。

由于 TCP 头部最大只有 60 个字节，标准头部占用了 20 字节，所以 Tcp Option 最大长度只有 40 字节，再加上 Tcp Timestamp option 占用了 10 个字节，所以留给 Sack 选项的只有 30 个字节。

每一个 Sack Block 的长度是 8 个，加上 Sack Option 头部 2 个字节，也就意味着 Tcp Sack Option 最大只能提供 3 个 Block。

但是 Quic Ack Frame 可以同时提供 256 个 Ack Block，在丢包率比较高的网络下，更多的 Sack Block 可以提升网络的恢复速度，减少重传量。

#### Ack Delay 时间

Tcp 的 Timestamp 选项存在一个问题，它只是回显了发送方的时间戳，但是没有计算接收端接收到 segment 到发送 Ack 该 segment 的时间。这个时间可以简称为 Ack Delay。这样就会导致RTT计算误差

<img src="https://img1.kiosk007.top/static/images/network/QUIC/ack_delay.jpg" style="width:600px;height:300px">

可以认为 TCP 的 RTT 计算：
```
RTT = timestamp1 - timestamp2 
```
QUIC 的 RTT 计算：
```
RTT = timestamp1 - timestamp2 - Ack Delay
```
当然具体还得基于采样和历史数据综合计算。

 

## 参考: 

[1] [科普：QUIC协议原理分析
](https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651000241&idx=2&sn=2b6e9426d15099c799c1d0a34d3877d4&chksm=bdbef5e28ac97cf4207ebcb8d119eafc96d590e167921bbc29b180210321a9d21d5c38c44f77&scene=27#wechat_redirect)

[2] [腾讯HTTPS优化性能实践](https://www.sohu.com/a/126685728_355140)

[3] [QUIC 协议在腾讯的实践和优化
](https://www.infoq.cn/article/Iiv7uzf6ayb9jsxWqkbe)

[4] [技术扫盲-新一代的基于UDP的低延时网络传输协议](http://www.52im.net/forum.php?mod=viewthread&tid=1309)

[5] [HTTP3原理与实践](http://www.alloyteam.com/2020/05/14385/)
