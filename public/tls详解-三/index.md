# TLS详解（三）

TLS 1.3现已于2018年8月发布。相比与TLS1.2，TLS1.3 在速度和安全性上做出了更大的性能提升，其最大的特点是支持了 Zero Round Trip Time (0-RTT). 在安全性方面，TLS1.3 摒弃了绝大多数不安全的加密套件，只支持几个AEAD的加密认证方式。


<!--more-->
- [TLS 详解（一）](https://kiosk007.top/post/tls详解-一/)
- [TLS 详解（二）](https://kiosk007.top/post/tls详解-二/)

# TLS1.3 Feature

- <font color="#7B68EE">Speed Benefits of TLS 1.3</font>

TLS1.3 可以使用1RTT建立握手，比1.2版本能节约一个网络来回。

<img style="height:300px" src="/images/network/TLSDetailAnalysis/tls-1.3-handshake-performance.png" />

- <font color="#7B68EE">Improved Security With TLS 1.3</font>

TLS1.3 移除了RC4、DES、MD5 等诸多脆弱不安全的算法，目前仅保持了支持AEAD的ECDH类等算法。


- <font color="#7B68EE"> 1.3 Browser Support</font>

从 Chrome65 开始，Google公司就可以支持 [draft version of TLS 1.3 ](http://www.chromium.org/Home/tls13) , 2018年10月的 Chrome70 就完全支持了TLS1.3。同样Firefox63也在同年10月支持了TLS1.3。Microsoft Edge version 76 及 Safari 12.1 on macOS 10.14.4. 也都支持了TLS1.3 。


# The TLS handshake

总体握手流程如下
``` go
func (hs *serverHandshakeStateTLS13) handshake() error {
	c := hs.c

	// For an overview of the TLS 1.3 handshake, see RFC 8446, Section 2.
	if err := hs.processClientHello(); err != nil { return err }
	if err := hs.checkForResumption(); err != nil { return err }
	if err := hs.pickCertificate(); err != nil { return err }
	c.buffering = true
	if err := hs.sendServerParameters(); err != nil { return err }
	if err := hs.sendServerCertificate(); err != nil { return err }
	if err := hs.sendServerFinished(); err != nil { return err }
	// Note that at this point we could start sending application data without
	// waiting for the client's second flight, but the application might not
	// expect the lack of replay protection of the ClientHello parameters.
	if _, err := c.flush(); err != nil { return err  }
	if err := hs.readClientCertificate(); err != nil {  return err  }
	if err := hs.readClientFinished(); err != nil { return err }

	atomic.StoreUint32(&c.handshakeStatus, 1)
	return nil
}
```

<img src="/images/network/TLSDetailAnalysis/tls-1.3.png" />

## <font style="color:#4682B4">Client Hello</font>

由于TLS1.2已经在互联网上存在了10年。网络中大量的网络中间设备都十分老旧，这些网络设备会识别中间的TLS握手头部，所以TLS1.3的出现如果引入了未知的TLS Version 必然会存在大量的握手失败，为了解决这一点，TLS1.3 的握手头部默认是TLS1.2。

<img style="height:400px" src="/images/network/TLSDetailAnalysis/tls-1.3-handshake-clienthello.png" />


如果客户端支持TLS1.3 则在 **<font style="color:#483D8B">Client Hello</font>** 发出时在Extensions中携带 supported_versions 并标明客户端是支持TLS1.3的，同样为了1RTT快速握手，会将客户端Key_share 发送给服务端。Key_Share是客户端提前生成好的公钥信息。其密钥派生过程依赖于密码套件的 HKDF Extract 和 HKDF Expand 函数以及 Hash函数。

在密钥交换之前，客户端和服务端使用HKDF生成密钥。（它取代了基于HMAC的伪随机密钥生成函数PRF。

下面用代码过一遍客户端的Client Hello流程。

``` go
{
...
var params ecdheParameters
if hello.supportedVersions[0] == VersionTLS13 {
	hello.cipherSuites = append(hello.cipherSuites, defaultCipherSuitesTLS13()...)

	curveID := config.curvePreferences()[0]
	if _, ok := curveForCurveID(curveID); curveID != X25519 && !ok {
		return nil, nil, errors.New("tls: CurvePreferences includes unsupported curve")
	}
	params, err = generateECDHEParameters(config.rand(), curveID)
	if err != nil {
		return nil, nil, err
	}
	hello.keyShares = []keyShare{{group: curveID, data: params.PublicKey()}}
}
}
```

这里，如果客户端是支持  <font style="color:#483D8B">VersionTLS13</font>, 则在创建 <font style="color:#483D8B"> Client Hello </font> 时,添加TLS1.3支持的秘钥套件，并使用 x25519 曲线和随机数生成 <font style="color:#483D8B"> PublickKey </font>放入 <font style="color:#483D8B"> Client Hello Extension</font> 中的 <font style="color:#483D8B"> KeyShares </font> 中。

## <font style="color:#4682B4">Server Hello</font>

服务端的TLS Version仍为TLS1.2（实际上后续的TLS版本均为1.2），如果服务端支持TLS1.3，则会在  <font style="color:#483D8B">supported_versions</font> 中的携带TLS1.3，这样后续的会话便均在TLS1.3下通信。

<img style="height:400px" src="/images/network/TLSDetailAnalysis/tls-1.3-handshake-serverhello.png" />

服务端会在<font style="color:#483D8B"> Server Hello </font> 中的 <font style="color:#483D8B"> key_share </font> 中携带公钥信息。

下面是完整的握手过程，BTW <font style="color:red">虽然0RTT是各大博客都吹嘘的TLS1.3亮点，但是0RTT 当前大多数的官方库都还没有实现（Nginx似乎是支持了）</font> ，比如看这里
``` go
	if hs.clientHello.earlyData {
		// See RFC 8446, Section 4.2.10 for the complicated behavior required
		// here. The scenario is that a different server at our address offered
		// to accept early data in the past, which we can't handle. For now, all
		// 0-RTT enabled session tickets need to expire before a Go server can
		// replace a server or join a pool. That's the same requirement that
		// applies to mixing or replacing with any TLS 1.2 server.
		c.sendAlert(alertUnsupportedExtension)
		return errors.New("tls: client sent unexpected early data")
	}

```

服务端选择和客户端同样支持的 <font style="color:#483D8B"> CurveID </font>(代码中的 selectedGroup，并且是Client支持的Key Share)。
``` go
	params, err := generateECDHEParameters(c.config.rand(), selectedGroup)
	if err != nil {
		c.sendAlert(alertInternalError)
		return err
	}
	hs.hello.serverShare = keyShare{group: selectedGroup, data: params.PublicKey()}
	hs.sharedKey = params.SharedKey(clientKeyShare.data)
	if hs.sharedKey == nil {
		c.sendAlert(alertIllegalParameter)
		return errors.New("tls: invalid client key share")
	}
```
这样以来，客户端和服务端便直接完成了 <font style="color:#483D8B">ECDHE</font> 密钥交换

- 客户端生成随机数x，确定了曲线类型如Golang TLS SDK只支持的 <font style="color:#483D8B"> x25519曲线</font> 即可得方程系数a、b，再调用`generateECDHEParameters` 获得 <font style="color:#483D8B"> PublicKey Q<sub>1</sub> </font>。客户端将 Q<sub>1</sub> 、a、 b、 P 传给服务端。
- 服务端生成随机数y，解析客户端传来的曲线和 Key_Share 对，得到曲线类型 <font style="color:#483D8B"> x25519 </font>既得方程系数a、b，再使用 selectedGroup 和 y 调用`generateECDHEParameters` 生成 <font style="color:#483D8B"> PublicKey Q<sub>2</sub> </font> ,传给客户端
- 这时客户端和服务端可以计算出一个公共的值 <font style="color:#483D8B"> **K** </font>

如下图

<img src="/images/network/TLSDetailAnalysis/ECDHE.png"/>

### <font style="color:#87CEEB"> PSK (Pre-Shared Key)</font>

这里在接着解析代码之前，先插播一个 TLS1.3 的feature 0RTT是如何实现的。这里介绍一下实现的原理 -- <font style="color:#483D8B"> **PSK** </font>

一旦一次握手完成，server 就能给 client 发送一个与一个独特密钥对应的 PSK 密钥，这个密钥来自初次握手。然后 client 能够使用这个 PSK 密钥在将来的握手中协商相关 PSK 的使用。如果 server 接受它，新连接的安全上下文在密码学上就与初始连接关联在一起，从初次握手中得到的密钥就会用于装载密码状态来替代完整的握手。在 TLS 1.2 以及更低的版本中，这个功能由 "session IDs" 和 "session tickets" [RFC5077]来提供。这两个机制在 TLS 1.3 中都被废除了。

PSK 可以与 (EC)DHE 密钥交换算法一同使用以便使共享密钥具备前向安全，或者 PSK 可以被单独使用，这样是以丢失了应用数据的前向安全为代价。

下图显示了两次握手，第一次建立了一个 PSK，第二次时使用它：

``` bash
          Client                                               Server

   Initial Handshake:
          ClientHello
          + key_share               -------->
                                                          ServerHello
                                                          + key_share
                                                {EncryptedExtensions}
                                                {CertificateRequest*}
                                                       {Certificate*}
                                                 {CertificateVerify*}
                                                           {Finished}
                                    <--------     [Application Data*]
          {Certificate*}
          {CertificateVerify*}
          {Finished}                -------->
                                    <--------      [NewSessionTicket]
          [Application Data]        <------->      [Application Data]


   Subsequent Handshake:
          ClientHello
          + key_share*
          + pre_shared_key          -------->
                                                          ServerHello
                                                     + pre_shared_key
                                                         + key_share*
                                                {EncryptedExtensions}
                                                           {Finished}
                                    <--------     [Application Data*]
          {Finished}                -------->
          [Application Data]        <------->      [Application Data]

```

当 server 通过一个 PSK 进行认证时，它不会发送一个 Certificate 或一个 CertificateVerify 消息。当一个 client 通过 PSK 想恢复会话的时候，它也应当提供一个 "key_share" 给 server，以允许 server 拒绝恢复会话的时候降级到重新回答一个完整的握手流程中。Server 响应 "pre_shared_key" 扩展，使用 PSK 密钥协商建立连接，同时响应 "key_share" 扩展来进行 (EC)DHE 密钥建立，由此提供前向安全。

当 PKS 在带外提供时，PSK 密钥和与 PSK 一起使用的 KDF hash 算法也必须被提供。

### <font style="color:#87CEEB">0 RTT</font>

当 client 和 server 共享一个 PSK（从外部获得或通过一个以前的握手获得）时，TLS 1.3 允许 client 在第一个发送出去的消息中携带数据（"application data"）。Client 使用这个 PSK 来认证 server 并加密 early data 信息，最终实现Application数据的0RTT发送。

如下图所示，0-RTT 数据在第一个发送的消息中被加入到 1-RTT 握手过程中。握手的其余消息与带 PSK 会话恢复的 1-RTT 握手消息相同。

``` bash
         Client                                               Server

         ClientHello
         + early_data
         + key_share*
         + psk_key_exchange_modes
         + pre_shared_key
         (Application Data*)     -------->
                                                         ServerHello
                                                    + pre_shared_key
                                                        + key_share*
                                               {EncryptedExtensions}
                                                       + early_data*
                                                          {Finished}
                                 <--------       [Application Data*]
         (EndOfEarlyData)
         {Finished}              -------->
         [Application Data]      <------->        [Application Data]
```
上图是 0-RTT 的信息流

0-RTT 数组安全性比其他类型的 TLS 数据要弱一些，特别是：

1. 0-RTT 的数据是没有前向安全性的，它使用的是被提供的 PSK 中导出的密钥进行加密的。
2. 在多个连接之间不能保证不存在重放攻击。普通的 TLS 1.3 1-RTT 数据为了防止重放攻击的保护方法是使用 server 下发的随机数，现在 0-RTT 不依赖于 ServerHello 消息，因此保护措施更差。如果数据与 TLS client 认证或与应用协议里一起验证，这一点安全性的考虑尤其重要。这个警告适用于任何使用 early_exporter_master_secret 的情况。


参考 **[TLS 1.3 Introduction](https://halfrost.com/tls_1-3_introduction/)** -- Halfrost's Field | 冰霜之地


### <font style="color:#87CEEB"> checkForResumption -- Go</font>

上面2个小节其实就是在介绍 [`checkForResumption()`](https://golang.org/src/crypto/tls/handshake_server_tls13.go#226) 这个函数的作用。

在Client Hello 包的扩展里如果有 **psk_key_exchange_modes** 和  **pre_shared_key** 就表示客户端想要会话复用，即类似TLS1.2的 **Session Ticket** or **Session Id** 的概念。

如下所示：

<img src="/images/network/TLSDetailAnalysis/tls-1.3-psk_key_exchange_modes.png"/>

<font style="color:#483D8B"> **psk_key_exchange_modes**</font>是 psk 密钥交互模式选择. 此处的PSK模式为(EC)DHE下的PSK，客户端和服务器必须提供KeyShare, 如果是仅PSK模式，则服务器不需要提供KeyShare。


<img src="/images/network/TLSDetailAnalysis/tls-1.3-pre_shared_key.png" />

<font style="color:#483D8B"> **pre_shared_key**</font> 是预共享密钥认证机制，相当于session ticket再加一些检验的东西.
Identity中包含的是客户端愿意进行协商的服务器身份列表。PSK binder表示已经构建当前PSK与当前握手之间的绑定。


下面函数中，服务端会将 <font style="color:#483D8B">identity</font> 解析成 <font style="color:#483D8B">plaintext</font>，<font style="color:#483D8B">plaintext</font>中包含TLS版本、证书、复用秘钥、超时时间 等多个信息，如果unmarshal成功，即可以会话复用，继续向下。


``` go
	for i, identity := range hs.clientHello.pskIdentities {
		if i >= maxClientPSKIdentities { break }
		plaintext, _ := c.decryptTicket(identity.label)
		if plaintext == nil { continue }
		sessionState := new(sessionStateTLS13)
        
		if ok := sessionState.unmarshal(plaintext); !ok { continue }
        
        createdAt := time.Unix(int64(sessionState.createdAt), 0)
		if c.config.time().Sub(createdAt) > maxSessionTicketLifetime { continue }

		// We don't check the obfuscated ticket age because it's affected by
		// clock skew and it's only a freshness signal useful for shrinking the
		// window for replay attacks, which don't affect us as we don't do 0-RTT.

		pskSuite := cipherSuiteTLS13ByID(sessionState.cipherSuite)
		if pskSuite == nil || pskSuite.hash != hs.suite.hash { continue }
		...
		psk := hs.suite.expandLabel(sessionState.resumptionSecret, "resumption",
			nil, hs.suite.hash.Size())
		hs.earlySecret = hs.suite.extract(psk, nil)
		binderKey := hs.suite.deriveSecret(hs.earlySecret, resumptionBinderLabel, nil)
		...
	}
    
```



## <font style="color:#4682B4"> Change Cipher Space</font>

发送一个 <font style="color:#483D8B">**ChangeCipherSpec record**</font> 报文，之后的加密方式将会改变。详见 See RFC 8446, Appendix D.4.

## <font style="color:#4682B4"> EncryptedExtensions </font>

随后 Server 会发来建立 EncryptedExtensions Server 参数: 对 ClientHello 扩展的响应，不需要确定加密参数，而不是特定于各个证书的加密参数。一般ALPN会在这里添加。


## <font style="color:#4682B4"> Certificate && Certificate Verify && Finished </font>

最后，Client 和 Server 交换认证消息。TLS 在每次基于证书的认证时使用相同的消息集，(基于 PSK 的认证是密钥交换中的一个副作用)特别是：

- <font style="color:#483D8B">Certificate</font>: 终端的证书和每个证书的扩展。 服务器如果不通过证书进行身份验证，并且如果服务器没有发送CertificateRequest（由此指示客户端不应该使用证书进行身份验证），客户端将忽略此消息。 请注意，如果使用原始公钥 [RFC7250] 或缓存信息扩展 [RFC7924]，则此消息将不包含证书，而是包含与服务器长期密钥相对应的其他值。

- <font style="color:#483D8B">CertificateVerify</font>: 使用与证书消息中的公钥配对的私钥对整个握手消息进行签名。如果终端没有使用证书进行验证则此消息会被忽略。

- <font style="color:#483D8B">Finished</font>: 对整个握手消息的 MAC(消息认证码)。这个消息提供了密钥确认，将终端身份与交换的密钥绑定在一起，这样在 PSK 模式下也能认证握手。

接收到 Server 的消息之后，Client 会响应它的认证消息，即 Certificate，CertificateVerify (如果需要), 和 Finished。

这时握手已经完成，client 和 server 会提取出密钥用于记录层交换应用层数据，这些数据需要通过认证的加密来保护。应用层数据不能在 Finished 消息之前发送数据，必须等到记录层开始使用加密密钥之后才可以发送。需要注意的是 server 可以在收到 client 的认证消息之前发送应用数据，任何在这个时间点发送的数据，当然都是在发送给一个未被认证的对端。

## <font style="color:#4682B4"> New Session Ticket </font>

实际等同于发送 PSK 数据。
``` go
func (hs *serverHandshakeStateTLS13) shouldSendSessionTickets() bool {
	if hs.c.config.SessionTicketsDisabled {
		return false
	}

	// Don't send tickets the client wouldn't use. See RFC 8446, Section 4.2.9.
	for _, pskMode := range hs.clientHello.pskModes {
		if pskMode == pskModeDHE {
			return true
		}
	}
	return false
}
```
