# 探索 Webtransport

常见的长连接协议如 websocket通信协议于2011年被IETF定为标准RFC 6455，并由RFC7936补充规范。WebSocket API也被W3C定为标准。时至今日，互联网的时代已由HTTP2.0 迈入了 HTTP3 的时代，而对长连接的需求日益升温，由于websocket本身的限制，完全不能复用 QUIC 的高性能优势，所以孕育而生了基于 QUIC 的新一代长连接 Webtransport 。

<!--more-->


# 概述

[WebTransport](https://wicg.github.io/web-transport/) 是一个新一代的浏览器API，提供客户端-服务端之间的双向低延迟交互，并在顶部使用常见 API 来实现其下的可插拔协议（<font style="color:#FF7F50">尤其是基于<a style="color:#FF7F50" href=" https://www.chromium.org/quic">QUIC</a></font>）。该 API 与 WebSocket 相似，也是客户端和服务器的双向连接，但允许进一步减少客户端和服务器之间的网络通信延迟，并且还支持多个流、单向流、乱序和不可靠传输。基于QUIC的Webtransport (Quictransport)即支持通过 datagram API 发送不可靠的数据，也支持通过 stream API 实现可靠数据传输。

使用场景包括使用不可靠且乱序的消息向服务器重复发送低延迟的游戏状态、从服务器到客户端的媒体片段的低延迟传输以及大多数逻辑在服务器上运行的云场景。


WebTransport 提案详细介绍：<font style="color:#FF7F50"> https://wicg.github.io/web-transport/ </font>


**重点**
1. Webtransport 支持不可靠传输，通过轻量级、低延迟的UDP协议传输。
2. Webtransport 可基于 QUIC 实现 Client-Server 可靠的流式传输。
3. 可支持多条流的相互独立 + QUIC 多路复用\非队头阻塞特性 完美代替当前的 Websocket。Webtransport 提供了一些当前websocket规范不可能提供的功能。可消除当前多个数据包之间的队头阻塞。

**标准规范**
1. <a style="color:#FF7F50" href="https://tools.ietf.org/html/draft-vvv-webtransport-overview-01"> WebTransport overview </a> : Webtransport 的概述及对传输层的要求。
2. <a style="color:#FF7F50" href="https://tools.ietf.org/html/draft-vvv-webtransport-quic"> WebTransport over QUIC</a> : 定义了基于QUIC的 Webtransport
3. <a style="color:#FF7F50" href="https://tools.ietf.org/html/draft-vvv-webtransport-http3-02"> WebTransport over HTTP/3 </a>: 定义了基于HTTP/3的 Webtransport （实际上 HTTP/3 也是基于QUIC的）


当前 Chrome 团队只实现了基于 QUIC 的 Webtransport 。然而目前也仅仅是实验性的。
``` javascript
const transport = new QuicTransport('quic-transport://localhost:4433/path');
```

Webtransport draft 标明是支持TCP的， 但显然目前大家都在UDP上了投入了大量精力，也主要是以UDP去实现的。

<img src='https://img1.kiosk007.top/static/images/network/WebTransport/common_transport_requirements.png'>

# WebTransport 与其他技术的关系

## WebTransport 可以代替 Websockets 吗？

也许可以，在某些互联网大厂的 IM、直播信令等应用其实已经在探索。

Websocket 通信围绕着 单一、可靠、有序 的消息流建模，对于某些类型的通信需求来说很好。但是 WebTransport 的API 也可以提供相同的功能，相比之下，WebTransport 的数据报 API 可以提供低延迟交付。

通过数据包 API ，多个 并发的 Stream API 实例使用 WebTransport，意味着不必担心[队列阻塞](https://en.wikipedia.org/wiki/Head-of-line_blocking) ，这可能是 WebSocket 的问题，此外，在建立新链接时还有性能优势，因为底层的[QUIC 握手](https://www.fastly.com/blog/quic-handshake-tls-compression-certificates-extension-study) 通过TLS启动TCP更快。

> 举个例子，我一个直播应用有多个 消息类型需要发送，
>
> - 消息1. 表示有客户加入或退出
> - 消息2. 表示有人私信
> - 消息3. 表示有人发弹幕
>
> 很显然，这3类消息并不强相关，也就是3类消息不需要严格保序发送，WebTransport 可以为3类消息各开一个流然后发送，假设一个某消息因丢包无法及时发送到对端，但是并不影响其他类型的消息。  



WebTransport 属于新草案规范，因此围绕客户端和服务器库的 WebSocket 生态系统目前更加强大。如果您需要具有通用服务器设置和广泛的 Web 客户端支持的“开箱即用”工具，WebSockets 是目前更好的选择。



## Websocket Over QUIC 和 WebTransport 对比有什么区别？

WebTransport 和 [Websocket Over QUIC](https://www.rfc-editor.org/rfc/rfc9220.html#name-websockets-upgrade-over-htt)  虽然都基于 QUIC 协议提升传输性能，但它们的设计理念、技术架构和应用场景存在本质差异。

WebSocket over QUIC 是将 WebSocket 协议（基于 TCP 的全双工通信）封装在 QUIC 流中，属于 **应用层协议的传输层优化**。

其底层利用的还是QUIC的一个 流，不是多个流，所以本质上没有利用QUIC的多路复用能力，IEEE 实验数据显示，在相同 QUIC 环境下，WebSocket over QUIC 的延迟和吞吐量与原生 WebSocket（基于 TCP）相比提升不足 10%，且在复杂网络条件下可能因协议叠加导致性能下降。



### 与现有技术的对比

| **技术**         | **传输协议** | **延迟** | **可靠性** | **适用场景**                |
| ---------------- | ------------ | -------- | ---------- | --------------------------- |
| **WebTransport** | QUIC + UDP   | 低       | 可选择     | 实时游戏、4K 视频流、物联网 |
| **WebSocket**    | TCP          | 中       | 可靠       | 聊天、简单通知              |
| **WebRTC**       | SRTP + DTLS  | 极低     | 可配置     | 视频会议、P2P 传输          |
| **HTTP/3**       | QUIC         | 低       | 可靠       | 网页加载、API 调用          |



# 探索

当前的Webtransport 必须基于 QUIC draft-29 或更高版本。客户端主要以 chrome 浏览器为主，版本必须 >= 85 。服务端我们将基于 <a href="github.com/lucas-clemente/quic-go" style="color:#FF7F50"> github.com/lucas-clemente/quic-go </a>  go library 。因为是本地测试，我们还需要签发一个自签名证书。

## 自签名证书

因为当前Webtransport的底层实现是基于 QUIC or HTTP/3 ，所以我们必须要实现自签名证书，确保通信过程的安全性。这里我们使用的是 `openssl` 

首先需要确保你的 `openssl` 安装
``` bash
➜ which openssl
/usr/bin/openssl
➜ openssl version
OpenSSL 1.1.1f  31 Mar 2020

```

创建证书和私钥

``` bash
➜ openssl req -newkey rsa:2048 -nodes -keyout certificate.key \
-x509 -out certificate.pem -subj '/CN=Test Certificate' \
-addext "subjectAltName = DNS:localhost"
```

计算证书的指纹

``` bash
➜ openssl x509 -pubkey -noout -in certificate.pem |
openssl rsa -pubin -outform der |
openssl dgst -sha256 -binary | base64
#      The result should be a base64-encoded blob that looks like this:
#          "Gi/HIwdiMcPZo2KBjnstF5kQdLI5bPrYJ8i3Vi6Ybck="
```

向chrome传入参数指明允许使用自签证书的服务端地址+端口。

``` bash
--origin-to-force-quic-on=localhost:4433
```
使用如下参数以信任证书
``` bash
--ignore-certificate-errors-spki-list=Gi/HIwdiMcPZo2KBjnstF5kQdLI5bPrYJ8i3Vi6Ybck=
```
更多可以参考： [docs on how to run Chrome/Chromium with custom flags.](https://www.chromium.org/developers/how-tos/run-chromium-with-flags)



最后打开 <a style="color:#FF7F50" href="https://googlechrome.github.io/samples/webtransport/client.html">https://googlechrome.github.io/samples/webtransport/client.html</a>

<img src="https://img1.kiosk007.top/static/images/network/WebTransport/webtransport_client.png" style="height:550px" />

## 服务端

我们使用 github.com/lucas-clemente/quic-go 来实现QUIC。

Run 方法来实现接受客户端的连接请求。quic.ListenAddr 创建一个监听器

``` go
// Run server.
func (s *WebTransportServerQuic) Run() error {
   listener, err := quic.ListenAddr(s.config.ListenAddr, s.generateTLSConfig(), s.generateQUICConfig())
   utils.Logging.Info().Err(err)
   if err != nil {
      return err
   }
   utils.Logging.Info().Msgf("WebTransport Engine v0.1 Start ...")
   utils.Logging.Info().Msgf("Listening for %s connections on %s","udp", s.config.ListenAddr)

   for {
      session, err := listener.Accept(context.Background())
      if err != nil {
         return err
      }
      utils.Logging.Info().Msgf("session accepted: %s", session.RemoteAddr().String())

      go func() {
         defer func() {
            _ = session.CloseWithError(0, "bye")
            utils.Logging.Info().Msgf("close session: %s", session.RemoteAddr().String())
         }()
         s.handleSession(session)
      }()
   }
}
```

客户端需要在 ALPN 中携带 alpnQuicTransport = "wq-vvv-01" 服务端读取后就会针对开始Webtransport 传输。
<img src='https://img1.kiosk007.top/static/images/network/WebTransport/wq-vvv-01_alpn.png' style="height:550px">

代码参见：<a href="https://github.com/weijiaxiang007/webtransport/" style="color:#FF7F50"> https://github.com/weijiaxiang007/webtransport/ </a>

## 传输模式

<font style="color:#6495ED">QUIC使用流ID的最低两位指示流标识以下信息</font>
1. 单向 or 双向流
2. 由客户端 or 服务端发起。

``` bash
                +------+----------------------------------+
                | Bits | Stream Type                      |
                +======+==================================+
                | 0x0  | Client-Initiated, Bidirectional  |
                +------+----------------------------------+
                | 0x1  | Server-Initiated, Bidirectional  |
                +------+----------------------------------+
                | 0x2  | Client-Initiated, Unidirectional |
                +------+----------------------------------+
                | 0x3  | Server-Initiated, Unidirectional |
                +------+----------------------------------+
```
- 对于每一个双向流，流的发起方和流的接收方均可以在一条双向流上传输数据。
- 对于每一条单向流，只能是流的发起方向流的接收方发送数据。接收方可以在一条新的单向流上回复数据。
- 对于数据报格式的数据，由于 quic-go 底层不支持，这里不再赘述。不过已经有相关的提交去支持 <a href="https://github.com/lucas-clemente/quic-go/pull/2162" style="color:#FF7F50"> https://github.com/lucas-clemente/quic-go/pull/2162 </a>

更多信息参见：<a href="https://tools.ietf.org/html/draft-ietf-quic-transport-27#section-2.1" style="color:#FF7F50">https://tools.ietf.org/html/draft-ietf-quic-transport-27#section-2.1</a>


<font style="color:#6495ED">客户端请求</font>
客户端请求一般 按照 Key -- Value 的方式携带请求的资源标识。如下是请求的 Origin 和 Path。

``` go
type clientIndicationKey int16

const (
   clientIndicationKeyOrigin clientIndicationKey = 0
   clientIndicationKeyPath                       = 1
)
```



``` bash
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |           Key (16)            |          Length (16)          |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                           Value (*)                         ...
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

这里只实现了双向流，可以看到双向流建立之前会先建立一个单向流用于认证信息。之后的数据会在双向流上传递

``` go
func (s *WebTransportServerQuic) handleSession(sess quic.Session) {
	stream, err := sess.AcceptUniStream(context.Background())
	if err != nil {
		utils.Logging.Error().Err(err)
		return
	}
	utils.Logging.Info().Msgf("unidirectional stream accepted, id: %d", stream.StreamID())
	indication, err := receiveClientIndication(stream)
	if err != nil {
		utils.Logging.Error().Err(err)
		return
	}
	utils.Logging.Info().Msgf("client indication: %+v", indication)
	if err := s.validateClientIndication(indication); err != nil {
		utils.Logging.Error().Err(err)
		return
	}
	err = s.communicate(sess)
	if err != nil {
		utils.Logging.Error().Err(err)
		return
	}
}
```
