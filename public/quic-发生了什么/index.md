# QUIC 发生了什么

从 HTTP/1.1（1999 年发布）到 HTTP/2 发布（2015 年发布）之间的发展差距很大，随着 2019 年 HTTP/3 的发布，HTTP/3 即将成为互联网上超文本传输协议的下一代协议，HTTP/3 是Google QUIC协议的演变。它与传统的 HTTP 有很大的不同。QUIC 是一种新的可靠传输协议，可以被视为一种下一代TCP。


<!--more-->


QUIC 连接通过 UDP 端口和 IP 地址建立的，一旦建立，该连接就通过其“连接 ID”相关联（每个连接都拥有一组连接标识符或连接 ID，每个连接标识符都可用于识别连接）。

QUIC 提供 0-RTT 和 1-RTT 连接设置，所以建立新连接时，会比传统的 TCP + TLS 的组合要快许多。


# HTTP 的演变

超文本传输协议 (HTTP) 是运行在 TCP/IP 之上的应用层协议，
HTTP 有 4 个稳定版本——HTTP/0.9、HTTP/1.0、HTTP/1.1 和 HTTP/2。在 2021年七月，市面上已经有 三分之二的浏览器支持了 HTTP/3 。

同年5月底，IETF：QUIC Version 1 ([RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html#name-version-negotiation)) 作为标准化版本现已发布，QUIC部署将从使用临时草案版本转向新创建的Version 1。


<img src="https://img1.kiosk007.top/static/images/network/QUIC/http_protocol_versions.png">


# QUIC 从哪来

IETF 并不是从头开始研究 QUIC。2012 年，[谷歌设计了自己的 QUIC 版本](https://blog.chromium.org/2013/06/experimenting-with-quic.html)， 随后 Chrome 浏览器及其Google的大部分服务相继支持QUIC，包括 Youtube 和 Google 搜索。

IETF QUIC 与 早期的 gQUIC 相比也已经有了比较大的变化，这里不妨列举几个。
- 加密协商：谷歌 QUIC 是定制加密握手，而 IETF 是采用了 TLS1.3
- 分离：IETF QUIC 的层级是分明的，其上可以承载的应用层协议不止 HTTP 协议
- 头部压缩：gQUIC 沿用了HTTP2的 HPack，而IETF QUIC 使用的是QPACK。


# QUIC 的版本

目前有[15种实验型QUIC](https://github.com/quicwg/base-drafts/wiki/Implementations)。【RFC9000 确定后，所有的QUIC版本会趋于统一】。但是这些实验性QUIC互操性比较差，因为各家的QUIC的实现标准不统一，对具体的一些功能实现存在diff。

<p class='div-border yellow'> 数据截取于 2021.07</p>

<h2> <a href="https://github.com/aiortc/aioquic" aria-hidden="true" class="header-anchor" >aioquic</a></h2>

使用Python和asyncio的QUIC实现

> - Language: Python 
> - Version: draft-29 through version 1(RCF 9000)
> - Handshake: TLS 1.3
> - Protocol IDs: `0xff00001d`, `0xff00001e`, `0xff00001f`, `0xff000020`, `0x1`


<h2> <a href="https://github.com/litespeedtech/lsquic" aria-hidden="true" class="header-anchor" >Chromium</a></h2>

Chromium的QUIC实现（chrome85.0.4171.0及更高版本支持draft-29）

> - **Languages**: C, C++
> - **Versions**: Q043, Q046, Q050, T050, T051, draft-27, draft-29.
> - **Handshakes**: QUIC Crypto, TLS
> - **Protocol IDs**: `Q043`, `Q046`, `Q050`, `T050`, `T051`, `0xff00001b`, `0xff00001d`
> - **ALPNs**: h3-Q043, h3-Q046, h3-Q050, h3-T050, h3-T051, h3-27, h3-29
> - Google将实现命名为“quiche”，但它与Cloudflare的Rust实现完全无关。


<h2> <a href="https://github.com/litespeedtech/lsquic" aria-hidden="true" class="header-anchor" >lsquic</a></h2>

LiteSpeed QUIC和HTTP/3库。适用于Linux、FreeBSD、MacOS、Android和Windows。


> - **Language**: C
> - **Version**: v1, Draft-34, Draft-29, Draft-27, Q043, Q046, and Q050.
> - **Roles**: Client, Server, Library
> - **Handshake**: QUIC Crypto, RFC 8446
> - **Protocol** IDs: `0x00000001`, `0xFF000022`, `0xFF00001D`, `0xFF00001B`


当前互联网上的[QUIC流量](https://quic.netray.io/stats.html)，draft-29 和 Version 1 版本 的量级在逐渐上升。国内的一些大厂也在使用QUIC来作为当前主要的流量接入，比如微信的视频号就使用 gQUIC 43 、快手也宣布了线上的[千万级QUIC计划](https://www.infoq.cn/article/41HJkeZM7hEgIYlJRFK0)。

<img src="https://img1.kiosk007.top/static/images/network/QUIC/quic_version.png">

相信今年的 RFC9000 推出 QUICv1 版本后，QUIC的互操性能得到更广泛的运用。当前各大版本实现的[QUIC互操性](https://interop.seemann.io/)能得到进一步的提升。

<img src="https://img1.kiosk007.top/static/images/network/QUIC/quic_interop.png" >


# QUIC 实验

当前的QUIC 版本很多，支持的比较完整的是 Chromium 和 lsquic 。

下面将以 [lsquic](https://github.com/litespeedtech/lsquic) 为例，实际剖析一下QUIC。

目前支持的QUIC版本有v1、Internet draft-27 和 draft-29, 包括老一些的Google QUIC 版本 `Q043` 、`Q046` 和 `Q050` 。


相关文档: https://lsquic.readthedocs.io/en/latest/

要构建LSQUIC，需要CMake、zlib和BoringSSL。


## 编译安装

- Building BoringSSL

``` bash
# 国内可用 git clone https://github.com/google/boringssl.git
git clone https://boringssl.googlesource.com/boringssl
cd boringssl

# 切换到特殊版本
git checkout a2278d4d2cabe73f6663e3299ea7808edfa306b9

# 编译 
cmake . &&  make

# 记录当前位置
BORINGSSL=$PWD

```

- Building LSQUIC Library

``` bash
# 注意：测试程序由 libevent 驱动，测试程序需要先装 libevent-dev
sudo apt-get install libevent-dev

git clone https://github.com/litespeedtech/lsquic.git
cd lsquic
git submodule init
git submodule update

# Statically:
# $BORINGSSL is the top-level BoringSSL directory from the previous step
cmake -DBORINGSSL_DIR=$BORINGSSL .
make

# As a dynamic library:
cmake -DLSQUIC_SHARED_LIB=1 -DBORINGSSL_DIR=$BORINGSSL .
make

```


- demo

quic 的 demo 在 $lsquic_pwd/bin 目录下, 我们实验用就用 `http_client` 吧。


## 实验

先看看 `http_client` 支持的几个命令行参数.

``` bash
./http_client -help 
Usage: http_client [opts]

Options:
   -p PATH     Path to request.  May be specified more than once.  If no
                 path is specified, the connection is closed as soon as
                 handshake succeeds.
   -n CONNS    Number of concurrent connections.  Defaults to 1.
   -r NREQS    Total number of requests to send.  Defaults to 1.
   -R MAXREQS  Maximum number of requests per single connection.  Some
                 connections will have fewer requests than this.
   -w CONCUR   Number of concurrent requests per single connection.
                 Defaults to 1.
   -M METHOD   Method.  Defaults to GET.
   -P PAYLOAD  Name of the file that contains payload to be used in the
                 request.  This adds two more headers to the request:
                 content-type: application/octet-stream and
                 content-length
   -K          Discard server response
   -I          Abort on incomplete reponse from server
   -4          Prefer IPv4 when resolving hostname
   -6          Prefer IPv6 when resolving hostname
   -C DIR      Certificate store.  If specified, server certificate will
                 be verified.
   -a          Display server certificate chain after successful handshake.
   -b N_BYTES  Send RESET_STREAM frame after the client has read n bytes.
   -t          Print stats to stdout.
   -T FILE     Print stats to FILE.  If FILE is -, print stats to stdout.
   -q FILE     QIF mode: issue requests from the QIF file and validate
                 server responses.
   -e TOKEN    Hexadecimal string representing resume token.
   -3 MAX      Close stream after reading at most MAX bytes.  The actual
                 number of bytes read is randominzed.
   -9 SPEC     Priority specification.  May be specified several times.
                 SPEC takes the form stream_id:nread:UI, where U is
                 urgency and I is incremental.  Matched \d+:\d+:[0-7][01]
   -7 DIR      Save fetched resources into this directory.
   -Q ALPN     Use hq ALPN.  Specify, for example, "h3-29".
   -0 FILE     Provide session resumption file (reading or writing)
   -s SVCPORT  Service port.  Takes on the form of host:port, host,
                 or port.  If host is not an IPv4 or IPv6 address, it is
                 resolved.  If host is not set, the value of SNI is
                 used (see the -H flag).  If port is not set, the default
                 is 443.
                 Examples:
                     127.0.0.1:12345
                     ::1:443
                     example.com
                     example.com:8443
                     8443
                 If no -s option is given, 0.0.0.0:12345 address
                 is used.
   -D          Do not set 'do not fragment' flag on outgoing UDP packets
   -z BYTES    Maximum size of outgoing UDP packets (client only).
                 Overrides -o base_plpmtu.
   -L LEVEL    Log level for all modules.  Possible values are `debug',
                 `info', `notice', `warn', `error', `alert', `emerg',
                 and `crit'.
   -l LEVELS   Log levels for modules, e.g.
                 -l event=info,engine=debug
               Can be specified more than once.
   -m MAX      Maximum number of outgoing packet buffers that can be
                 assigned at any one time.  By default, there is no max.
   -y style    Timestamp style used in log messages.  The following styles
                 are supported:
                   0   No timestamp
                   1   Millisecond time (this is the default).
                         Example: 11:04:05.196
                   2   Full date and millisecond time.
                         Example: 2017-03-21 13:43:46.671
                   3   Chrome-like timestamp: date/time.microseconds.
                         Example: 1223/104613.946956
                   4   Microsecond time.
                         Example: 11:04:05.196308
                   5   Full date and microsecond time.
                         Example: 2017-03-21 13:43:46.671345
   -S opt=val  Socket options.  Supported options:
                   sndbuf=12345    # Sets SO_SNDBUF
                   rcvbuf=12345    # Sets SO_RCVBUF
   -W          Use stock PMI (malloc & free)
   -g          Use sendmmsg() to send packets.
   -j          Use recvmmsg() to receive packets.
   -H host     Value of `host' HTTP header.  This is also used as SNI
                 in Client Hello.  This option is used to override the
                 `host' part of the address specified using -s flag.
   -G dir      SSL keys will be logged to files in this directory.
   -k          Connect UDP socket.  Only meant to be used with clients
                 to pick up ICMP errors.
   -i USECS    Clock granularity in microseconds.  Defaults to 1000.
   -h          Print this help screen and exit

```

# QUIC 发生了什么

lsquic 提供了测试网站 `www.litespeedtech.com`，我们用下面的命令行工具访问这个网站, 并使用Wireshark 抓包和 [qlog 去解析QUIC](https://zhuanlan.zhihu.com/p/151858528)，探索一些QUIC究竟发生了什么。

<p class='div-border green'>Wireshark （3.4.2）已经支持了解析 IETF QUIC，加载秘钥的方法参考 <a href="https://unit42.paloaltonetworks.com/wireshark-tutorial-decrypting-https-traffic/">Wireshark Tutorial: Decrypting HTTPS Traffic
</a></p>

``` bash
# 不指定版本。默认是使用 Version 1 版本 
$ http_client -H www.litespeedtech.com:443 -p /  -G . -o version=h3-29

```

当前目录下会生成一个 sslkeylog `F9CE5E3D598B8D80.keys`，wireshark 导入，注意[Wireshark 要装最新版本（当前是 3.4.2）](https://github.com/quicwg/base-drafts/wiki/Tools#wireshark)。

> 注意： Wireshark 本身无法解密 GQUIC 数据包，即使已配置NSS Keylogging。

<p class="div-border green">更多QUIC debug or 可视化工具参考: https://github.com/quicwg/base-drafts/wiki/Tools </p>

抓包内容如下：
<img src="https://img1.kiosk007.top/static/images/network/QUIC/quic_capture_handshake.png" >


**这里再介绍一下HTTP3的分层结构。**

一个 HTTP3 包会按照以下方式分层。实时上 QUIC 已经完全可以理解为一个传输层协议了，QUIC的实现也是和HTTP协议分离的，正如 RFC 9000 所描述的。“[QUIC 是一种面向连接的协议，它在客户端和服务器之间创建有状态的交互。](https://www.rfc-editor.org/rfc/rfc9000.html#section-1-2)”

<p class='div-border blue'> <a href="https://github.com/lucas-clemente/quic-go/blob/master/example/echo/echo.go">  quic-go 的 echo 示例 </a> 也在其示例代码中提供了QUIC层的demo `session.OpenStreamSync(context.Background())` </p>

``` bash
UDP Header
Packet Header     -----
QUIC Frame Header    --  TLS1.3
HTTP3 Frame Header   --   
HTTP Message      -----
```

## Packet Header

Packet Header实现了可靠的连接。当UDP报文丢失后，通过Packet Header中的Packet Number实现报文重传。连接也是通过其中的Connection ID字段定义的

这里会有两种 Header 类型，**Long Header** 和 **Short Header**。Long Header 用于初始化交换 直到 可以1RTT可以被激动和版本协商完成。Short Header 用于承载数据。

- **Long Header**

``` bash
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|S|Typ|  Next   |              Magic "uic"/"UIC"                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                         Connection ID                         +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            Version                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Packet Number                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       [Header Extensions]                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Payload                           ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

建立连接时，连接是由服务器通过Source Connection ID字段分配的，这样，后续传输时，双方只需要固定住Destination Connection ID，就可以在客户端IP地址、端口变化后，绕过UDP四元组（与TCP四元组相同），实现连接迁移功能。如下图，后续的DCID和SCID都会统一为 `752566b9c5e2b77a`


<img src="https://img1.kiosk007.top/static/images/network/QUIC/quic_connection_id.png" >

- **Short Header**

``` bash
0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|S|K|                Packet Number (30)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                         Connection ID                         +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Packet Number是每个报文独一无二的序号，基于它可以实现丢失报文的精准重发。

参考: [martinthomson/quic_header.md](https://gist.github.com/martinthomson/744d04cbcec9be554f2f8e7bae2715b8)


## QUIC Frame Header

在Packet Header之上的QUIC Frame Header, QUIC中的流为应用程序提供了轻量级的有序字节流抽象。流可以是单向的或双向的，而且Stream之间可以实现真正的并发。

所有的帧被包含在单独的QUIC Packet 中，且没有帧可以跨越QUIC Packet 边界。一个Packet报文中可以存放多个QUIC Frame，当然所有Frame的长度之和不能大于PMTUD（Path Maximum Transmission Unit Discovery，这是大于1200字节的值）, 一个QUIC Frame 也不允许跨Packet。

<img src="https://img1.kiosk007.top/static/images/network/QUIC/quic_frame1.png">


每一个QUIC Frame 都有明确的帧类型。


参考： [帧类型和格式](https://quic.readthedocs.io/zh/latest/Frame%20Types%20and%20Formats.html)

<p class='div-border red'> 上述文档比较老，很多数据有变动，下面是结合 <a href="https://www.rfc-editor.org/rfc/rfc9000.html#name-frames-and-frame-types"> RFC9000 </a> 的版本 </p>




当前定义的**普通帧类型**如下：

``` bash
+-----------------------+-----------------------------+------------+
| Type-field value      |     Control Frame-type      |    Spec    |
+-----------------------+-----------------------------+------------+
| 00000000B (0x00)      |  PADDING                    |     NP     |
| 00000001B (0x01)      |  PING                       |            |
| 00000010B (0x02-0x03) |  ACK                        |     NC     |
| 00000100B (0x04)      |  RESET_STREAM               |            |
| 00000101B (0x05)      |  STOP_SENDING               |            |
| 00000110B (0x06)      |  CRYPTO                     |            |
| 00000111B (0x07)      |  NEW_TOKEN                  |            |
| 00001000B (0x08-0x0F) |  STREAM                     |     F      |
| 00010000B (0x10)      |  MAX_DATA                   |            |
| 00010001B (0x11)      |  MAX_STREAM_DATA            |            |
| 00001100B (0x12-0x13) |  MAX_STREAMS                |            |
| 00001110B (0x14)      |  DATA_BLOCKED               |            |
| 00001111B (0x15)      |  STREAM_DATA_BLOCKED        |            |
| 00010000B (0x16-0x17) |  STREAMS_BLOCKED            |            |
| 00010010B (0x18)      |  NEW_CONNECTION_ID	      |     P      |
| 00010011B (0x19)      |  RETIRE_CONNECTION_ID       |            |
| 00010100B (0x1a)      |  PATH_CHALLENGE             |     P      |
| 00010101B (0x1b)      |  PATH_RESPONSE              |     P      |
| 00010110B (0x1c-0x1d) |  CONNECTION_CLOSE           |     N      |
| 00011000B (0x1e)      |  HANDSHAKE_DONE             |            |
+-----------------------+-----------------------------+------------+
```

> 第三列 SPEC 规定了该帧的一些特殊用法
> N : 该类帧不会被ACK
> C : 出于拥塞控制目的,该类型帧不会被计入飞行中的字节数
> P : 该类型帧的数据包可用于在连接迁移期间探测新的网络路径
> F : 该类型帧用于流控

- **PADDING 帧**：PADDING帧使用0x00字节填充一个包。当遇到该帧时，包的剩余部分需要被填充字节。 该帧包含0x00字节并扩展至QUIC包的末端。
- **PING 帧**：PING帧用来验证对端是否仍然存活。PING帧不包含载荷。 PING帧的接收方只需要应答（ACK）包含该帧的包。 PING帧应该被用于当一条流被打开时，保持连接存活。 默认是30s
- **ACK 帧**: 发送ACK帧以通知对端已经接收了哪些分组，以及接收方仍然认为丢失了哪些分组（可能需要重新发送丢失分组的内容）。
    - 如果帧类型为0x02：普通的ACK应答作用
	- 如果帧类型为0x03：则ACK帧还包含在此点之前在连接上接收到的具有相关ECN标记的QUIC数据包的累积计数。如果发送的数据包启用了ECN，则应使用ECN部分中的信息来管理拥塞状态
- **RST_STREAM 帧**：RST_STREAM帧允许异常终止一条流。当这个帧是流的创建者发出的，表示创建者希望取消这条流。 当接收端发送这个帧，表示有错误或者当前接收端不希望接收这个流，因此这个流应该被关闭。
- **STOP_SENDING 帧**：请求对等方停止流上的传输.
- **CRYPTO 帧**：用于传输加密握手消息，加密帧在功能上与STREAM帧相同,但是不受流控。加密帧上传输 TLS 握手细信息。
- **NEW_TOKEN 帧**：为客户端提供一个令牌，以便用于之后的链接0-RTT。
- **STREAM 帧** : STREAM帧同时被用于隐式地创建流和在流上发送数据。STREAM帧中的类型字段的格式为0B00001xxx（或0x08到0x0f的一组值）,帧类型的三个低阶位确定帧中存在的字段。流可以是单向的，也可以是双向的，对于连接上的所有流都是唯一的。流低2位可标识是客户端还是服务端发起的单向流或双向流。
```
Bits	Stream Type
0x00	Client-Initiated, Bidirectional
0x01	Server-Initiated, Bidirectional
0x02	Client-Initiated, Unidirectional
0x03	Server-Initiated, Unidirectional
```
- **MAX_DATA 帧**：用于流控,用于通知对端整个连接上可发送的最大数据量。
- **MAX_STREAM_DATA 帧**：用于流控，用于通知对端流上可发送的最大数据量。
- **MAX_STREAMS 帧**：通知对等端允许打开的给定类型的流的最大数。
	- 0x12 的MAX_STREAM适用于双向流
    - 0x13 的MAX_STREAM适用于单向流
- **DATA_BLOCKED 帧**：BLOCKED帧用于向远端指明本端点已经准备好发送数据了（且有数据要发送）， 但是当前被流量控制阻塞了。这是一个纯粹的信息帧，它对于调试极其有用。 BLOCKED帧的接收者应该简单的丢弃它（可能在打印了一条有帮助的log消息之后）
- **STREAM_DATA_BLOCKED 帧**：当发送方希望发送数据但由于流级流控制而无法发送数据时，发送方应发送 STREAM_DATA_BLOCKED。
- **STREAMS_BLOCKED 帧**：当发送方希望打开流，但由于其对等方设置的最大流限制而无法打开流时，发送方应发送STREAMS_BLOCKED,
     - 0x16 的STREAMS_BLOCKED适用于双向流
     - 0x17 的STREAMS_BLOCKED适用于单向流
- **NEW_CONNECTION_ID 帧**：发送一个新的 NEW_CONNECTION_ID 帧, 为其对端提供可用于在迁移连接时中断可链接性的替代连接ID
- **RETIRE_CONNECTION_ID 帧**: 一端发送 RETIRE_CONNECTION_ID 表示不再使用对端发送的CONNECTION ID ，包含握手时确定的 连接id，后续可以发送新的 CONNECTION ID 再。
- **PATH_CHALLENGE 帧**：来检查对等方的可达性，并在连接迁移期间进行路径验证
- **PATH_RESPONSE 帧**: PATH_CHALLENGE 的响应帧
- **CONNECTION_CLOSE 帧**：CONNECTION_CLOSE帧用来通知连接将被关闭。如果流仍然有数据在发送，那么在连接关闭时， 这些流将被隐式关闭。
- **HANDSHAKE_DONE 帧**：向客户端发送握手确认信号。



## HTTP3 Frame Header

QUIC 的 STREAM 帧是实际承载流量的帧。自然也是 [HTTP3 协议](https://quicwg.org/base-drafts/draft-ietf-quic-http.html#name-reserved-frame-types)的承载。

<img src="https://img1.kiosk007.top/static/images/network/QUIC/quic_stream_frame.png"> 

Stream Frame头部的3个字段，完成了多路复用、有序字节流以及报文段层面的二进制分隔功能

- **Stream ID** 标识了一个有序字节流。当HTTP Body非常大，需要跨越多个Packet时，只要在每个Stream Frame中含有同样的Stream ID，就可以传输任意长度的消息。多个并发传输的HTTP消息，通过不同的Stream ID加以区别
- 消息序列化后的“有序”特性，是通过Offset字段完成的，它类似于TCP协议中的Sequence序号，用于实现Stream内多个Frame间的累计确认功能；
- Length指明了Frame数据的长度。

HTTP3 的Frame 继承了 HTTP2 的Frame 设计。

``` bash
Frame         Control Stream	Request Stream	Push Stream	
DATA          No                Yes               Yes	
HEADERS       No                Yes               Yes	
CANCEL_PUSH   Yes               No                No	
SETTINGS      Yes              (1)                No	
PUSH_PROMISE  No                Yes               No	
GOAWAY        Yes               No                No	
MAX_PUSH_ID   Yes               No                No	
Reserved      Yes               Yes               Yes	
```
QUIC Stream Frame定义了有序字节流，且多个Stream间的传输没有时序性要求，这样，HTTP消息基于QUIC Stream就实现了真正的多路复用，队头阻塞问题自然就被解决掉了。

**HTTP Header头部的编码方式，它需要面对另一种队头阻塞问题**

与HTTP2中的HPACK编码方式相似，HTTP3中的QPACK也采用了静态表、动态表及Huffman编码：

在HTTP2中，共有61个静态表项,而在QPACK中，则上升为98个静态表项, 比如 qpack 的golang 实现中的 `staticTableEntries` 所示。可以从[这里](https://github.com/marten-seemann/qpack/blob/master/static_table.go)找到 HTTP3 的完整静态表

<img src="https://img1.kiosk007.top/static/images/network/QUIC/qpack.png">


相较于HTTP3的静态表，和HTTP2协议其实并没有特别大的变化。但动态表编解码方式差距很大。

所谓动态表，就是将未包含在静态表中的Header项，在其首次出现时加入动态表，这样后续传输时仅用1个数字表示，大大提升了编码效率。因此，动态表是天然具备时序性的，如果首次出现的请求出现了丢包，后续请求解码HPACK头部时，一定会被阻塞！


事实上，QPACK将动态表的编码、解码独立在单向Stream中传输，仅当单向Stream中的动态表编码成功后，接收端才能解码双向Stream上HTTP消息里的动态表索引。

因此，当Stream ID是0、4、8、12时，这就是客户端发起的双向Stream（HTTP3不支持服务器发起双向Stream），它用于传输HTTP请求与响应。单向Stream有很多用途，所以它在数据前又多出一个Stream Type字段：

<img src="https://img1.kiosk007.top/static/images/network/QUIC/quic_stream_type.png">

Stream Type有以下取值：
-  0x00：控制Stream，传递各类Stream控制消息；
-  0x01：服务器推送消息；
-  0x02：用于编码QPACK动态表，比如面对不属于静态表的HTTP请求头部，客户端可以通过这个Stream发送动态表编码；
-  0x03：用于通知编码端QPACK动态表的更新结果。

由于HTTP3的STREAM之间是乱序传输的，因此，若先发送的编码Stream后到达，双向Stream中的QPACK头部就无法解码，此时传输HTTP消息的双向Stream就会进入Block阻塞状态（两端可以通过控制帧定义阻塞Stream的处理方式）。

QPack 在编解码原理上和 HPack 没有本质区别。从 [QPACK](https://github.com/marten-seemann/qpack) 的代码中可以看出，其内部完全是套了一层 HAPCK 的实现

``` go
// WriteField encodes f into a single Write to e's underlying Writer.
// This function may also produce bytes for the Header Block Prefix
// if necessary. If produced, it is done before encoding f.
func (e *Encoder) WriteField(f HeaderField) error {
	// write the Header Block Prefix
	if !e.wrotePrefix {
		e.buf = appendVarInt(e.buf, 8, 0)
		e.buf = appendVarInt(e.buf, 7, 0)
		e.wrotePrefix = true
	}

	idxAndVals, nameFound := encoderMap[f.Name]
	if nameFound {
		if idxAndVals.values == nil {
			if len(f.Value) == 0 {
				e.writeIndexedField(idxAndVals.idx)
			} else {
				e.writeLiteralFieldWithNameReference(&f, idxAndVals.idx)
			}
		} else {
			valIdx, valueFound := idxAndVals.values[f.Value]
			if valueFound {
				e.writeIndexedField(valIdx)
			} else {
				e.writeLiteralFieldWithNameReference(&f, idxAndVals.idx)
			}
		}
	} else {
		e.writeLiteralFieldWithoutNameReference(f)
	}

	e.w.Write(e.buf)
	e.buf = e.buf[:0]
	return nil
}
```

# QUIC Feature

## 0 RTT

使用 0-RTT 取决于客户端和服务器使用从先前连接协商的协议参数。为了启用 0-RTT, 客户端将服务器传输参数与它在连接上收到的 session tickets 一起存储, 另外还存储应用程序或加密握手所需的信息。


和握手相关的报头都是长报头(`Long Packet`) 。分别是 **`Initial (0x00)`**、**`0-RTT	(0x01)`**、**`Handshake (0x02)`**、**`Retry (0x03)`**。

- `CRYPTO` 帧可以在不同的数据包编号空间（packet number spaces）中发送。CRYPTO 帧使用偏移量（offsets）来确保加密握手数据的有序传递的在每个包编号空间（packet number spaces）都是从零开始。

**握手流程示例**

1. QUIC 在握手前会先进行地址验证（Address Validation），确保请求包里面的源地址不是伪造的。
2. 一旦地址验证交换完成，就可以使用加密握手来获取加密密钥。加密握手通过初始（Initial）和握手（Handshake）包进行传输。
3. 下图展示了 1-RTT 握手的示例。每行显示一个 QUIC 包（packet），首先显示包类型（type）和包编号（number），然后是帧（frames）。例如，第一个包是 Initial 类型，包编号为 0，并且包含一个携带 ClientHello（缩写：CH） 的 CRYPTO 帧。
4. 多个 QUIC 数据包（即便是不同的类型）也可以合并成一个单独的 UDP 数据报（datagram）。因此，下图所示的 1-RTT 握手可以由 4 个UDP数据报（datagrams）组成。如果受协议固有的限制（如拥塞控制（congestion control）和反放大（anti-amplification））也可以使用更多的数据报。

- 1 - RTT

``` 
Client                                                  Server

Initial[0]: CRYPTO[CH] ->

                                 Initial[0]: CRYPTO[SH] ACK[0]
                       Handshake[0]: CRYPTO[EE, CERT, CV, FIN]
                                 <- 1-RTT[0]: STREAM[1, "..."]

Initial[1]: ACK[0]
Handshake[0]: CRYPTO[FIN], ACK[0]
1-RTT[0]: STREAM[0, "..."], ACK[0] ->

                                          Handshake[1]: ACK[0]
         <- 1-RTT[1]: HANDSHAKE_DONE, STREAM[3, "..."], ACK[0]
```

- 0 - RTT

```
Client                                                  Server

Initial[0]: CRYPTO[CH]
0-RTT[0]: STREAM[0, "..."] ->

                                 Initial[0]: CRYPTO[SH] ACK[0]
                                  Handshake[0] CRYPTO[EE, FIN]
                          <- 1-RTT[0]: STREAM[1, "..."] ACK[0]

Initial[1]: ACK[0]
Handshake[0]: CRYPTO[FIN], ACK[0]
1-RTT[1]: STREAM[0, "..."] ACK[0] ->

                                          Handshake[1]: ACK[0]
         <- 1-RTT[1]: HANDSHAKE_DONE, STREAM[3, "..."], ACK[1]
```


如下是取自于 qvis的 1-RTT 握手请求。


<img src="https://img1.kiosk007.top/static/images/network/QUIC/qvis_quictools_info.png" >


## 地址验证（Address Validation）

地址验证主要是用于确保端点（endpoint）不能被用于流量放大攻击（traffic amplification attack）。攻击者如果伪造数据包的源地址为受害者的地址，发送大量的数据包给服务端，如果服务端没有进行地址验证，直接响应大量数据包给源地址（受害者），就会被攻击者利用、进行流量放大攻击。

QUIC 针对放大攻击的主要防御措施是验证端点（endpoint）是否能够在其声明的传输地址接收数据包。地址验证在**连接建立（connection establishment）期间**和 **连接迁移（connection migration）期间**进行。

- 连接建立时，为了验证客户端的地址是否是攻击者伪造的，服务端会生成一个令牌（token）并通过重试包（Retry packet）响应给客户端。客户端需要在后续的初始包（Initial packet）带上这个令牌，以便服务端进行地址验证。
- 服务端可以在当前连接中通过 NEW_TOKEN 帧预先发布令牌，以便客户端在后续的新连接使用，这是 QUIC 实现 0-RTT 很重要的一个功能。
- 当我们的网络路径变化时（比如从蜂窝网络切换到 WIFI），QUIC 提供了连接迁移（connection migration）的功能来避免连接中断。QUIC 通过路径验证（Path Validation）验证网络新地址的可达性（reachability），防止在连接迁移中的地址是攻击者伪造的。

**使用重试数据包（Retry Packets）验证地址**

在接收到客户端的初始数据包（Initial packet）后，服务端可以通过发送包含令牌（token）的重试数据包（Retry packet）来请求地址验证。客户端在接收到这个重试数据包（Retry packet）的令牌（token）之后，必须在该连接中后续所有初始数据包（Initial packet）中附上该令牌（token）。

```
Client                                                  Server

Initial[0]: CRYPTO[CH] ->

                                                <- Retry+Token

Initial+Token[1]: CRYPTO[CH] ->

                                 Initial[0]: CRYPTO[SH] ACK[1]
                       Handshake[0]: CRYPTO[EE, CERT, CV, FIN]
                                 <- 1-RTT[0]: STREAM[1, "..."]
```

**后续的连接使用 NEW_TOKEN 帧的令牌（token）**

服务端可以在一次连接中向客户端提供地址验证令牌（token），该令牌可用于后续的连接。这对于 0-RTT 尤其重要，后续的新连接可以直接使用该令牌进行地址验证，而无需额外的 1-RTT。

服务端使用 NEW_TOKEN 帧向客户端提供地址验证令牌，该令牌可用于验证后续的连接。在后续的连接中，客户端在初始数据包（Initial packets）中包含该令牌，以提供地址验证。

重试数据包（Retry packet）中提供的令牌只能立即使用，不能用于后续连接的地址验证。而 NEW_TOKEN 帧生成的令牌可以在一个时间范围内使用，这个令牌应该有一个过期时间，可以是显式的过期时间，也可以是可用于动态计算过期时间的时间戳（timestamp）。服务端可以存储过期时间，也可以在令牌中以加密的形式包含它。



参考： [跟坚哥学QUIC系列：地址验证（Address Validation)](https://zhuanlan.zhihu.com/p/290694322)




## 链接迁移

QUIC 使用连接ID（而不是 ip + port）来确保数据包的路由一致性。如果用户的 IP 发生变化时，比如从移动蜂窝 4G 网络切换到 WiFi，IP 地址会改变。而使用一个唯一的连接ID 可以确保用户的 IP 变化时业务请求依然能够被继续处理，不用重新建连，可以继续使用当前连接ID 路由数据包，因此 QUIC 可以通过这个特性支持连接迁移

QUIC 的数据包（packets）长报头（long header）包含两个连接ID：目标连接ID（Destination Connection ID）由数据包的接收者选择并用于提供一致的路由，源连接ID（Source Connection ID）用于对端（peer）响应时使用的目标连接ID（Destination Connection ID）。

在握手过程中，带有长报头的包用于建立两端（both endpoints）使用的连接ID。在处理第一个初始数据包（Initial packet）之后，每个端点使用其接收到的源连接ID（Source Connection ID）字段的值设置为后续数据包中的目标连接ID（Destination Connection ID）字段。

当客户端发送了一个初始包（Initial packet），而该客户端之前没有从服务端接收过初始数据包（Initial packet）或重试包（Retry packet），则客户端将用一个不可预测的值（长度至少为8字节）填充到目标连接ID字段。在从服务端接收到数据包之前，客户端必须对该连接中的所有数据包使用相同的目标连接ID值。

当第一次从服务端接收到初始（Initial）或重试（Retry）数据包时，客户端使用服务端提供的源连接ID作为后续数据包（包括任何 0-RTT 数据包）的目标连接ID。这意味着在建立连接的过程中，客户端可能需要两次更改它的目标连接ID字段：一次用于响应重试（Retry），一次用于响应来自服务端的初始数据包（Initial）。一旦客户端从服务端接收到有效的初始数据包，客户端必须丢弃它后续接收到的具有不同源连接ID的数据包。

服务端必须根据第一个接收到的初始数据包（Initial packet）的源连接ID，设置为用于发送数据包的目标连接ID。后续只有当接收到 NEW_CONNECTION_ID 帧时，才允许对目标连接ID进行更改。如果后续初始数据包包含不同的源连接ID，则必须将其丢弃。这样可以避免由于无状态（stateless）处理具有不同源连接ID的多个初始数据包而导致的不可预测结果。



参考文章： 
- [What's Happening with QUIC](https://www.ietf.org/blog/whats-happening-quic/)
- [Road To QUIC](https://blogs.keysight.com/blogs/tech/nwvs.entry.html/2021/07/16/road_to_quic-DGa5.html)
- [深入剖析HTTP3协议](https://www.nginx.org.cn/article/detail/422)
- [跟坚哥学QUIC系列：地址验证（Address Validation)](https://zhuanlan.zhihu.com/p/290694322)

