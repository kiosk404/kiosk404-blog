---
title: TLS 详解（二）
author: kiosk
tags:
  - tls
categories:
  - network
date: 2020-05-04 14:03:00
---
承接 TLS详解（一），本文主要以源码的视角介绍一下TLS的整体过程，这里以 [GoTLS](http://golang.org/pkg/crypto/tls/) (golang 自己按照 RFC 实现的一套 tls)为切入点还原整个TLS过程。主要介绍 TLS 的 handshake protocol 和 record protocol。

<!--more-->

-------------------------------------------------------
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/CPU.webp" />

- [TLS 详解（一）](https://kiosk007.top/blog/2020/05/02/tls-详解一/)
- [TLS 详解（三）](https://kiosk007.top/blog/2021/01/16/tls详解三/)


# TLS 过程分析

我们按TLS的分record类来分析。共有以下4类。最主要的是 Handshake 和 Application。

``` go
const (
	recordTypeChangeCipherSpec recordType = 20
	recordTypeAlert            recordType = 21
	recordTypeHandshake        recordType = 22
	recordTypeApplicationData  recordType = 23
)
```

## handshake protocol

> **以下按照 TLS1.2 介绍**

这里以LTS握手展开，同样以最常见的 **`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`** 为例。握手采用`ECDHE`，身份认证采用`RSA`。在handshake中，最主要的工作也是客户端和服务器端协商TLS协议版本号和一个CipherSuite，认证对端的身份（可选，一般如https是客户端认证服务器端的身份），并且使用密钥协商算法生成共享的master secret。

大致步骤如下：
- 交换Hello消息，协商出算法，交换random值，检查session resumption.
- 交换必要的密码学参数，来支持client和server协商出premaster secret。
- 交换证书和密码学参数，让client和server做认证，证明自己的身份。
- 从premaster secret和交换的random值 ，生成出master secret。
- 允许client和server确认对端得出了相同的Security Parameters，并且握手过程的数据没有被攻击者篡改。。

为了在握手协议解决降级攻击的问题，TLS协议规定：client发送ClientHello消息，server必须回复ServerHello消息，否则就是fatal error，当成连接失败处理。ClientHello和ServerHello消息用于建立client和server之间的安全增强能力，ClientHello和ServerHello消息建立如下属性：

- Protocol Version
- Session ID
- Cipher Suite
- Compression Method.
另外，产生并交换两个random值 ClientHello.random 和 ServerHello.random
完整流程：
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/ssl_handshake_diffie_hellman.jpg" />

### Client Hello

golang 在 `./tls/handshake_client.go` 中的 `makeClientHello()` 方法中初始化了一个 clienthello。
``` golang
...
   hello := &clientHelloMsg{
		vers:                         clientHelloVersion,
		compressionMethods:           []uint8{compressionNone},
		random:                       make([]byte, 32),
		sessionId:                    make([]byte, 32),
		ocspStapling:                 true,
		scts:                         true,
		serverName:                   hostnameInSNI(config.ServerName),
		supportedCurves:              config.curvePreferences(),
		supportedPoints:              []uint8{pointFormatUncompressed},
		nextProtoNeg:                 len(config.NextProtos) > 0,
		secureRenegotiationSupported: true,
		alpnProtocols:                config.NextProtos,
		supportedVersions:            supportedVersions,
	}
// 插入支持的签名算法
    if hello.vers >= VersionTLS12 {
		hello.supportedSignatureAlgorithms = supportedSignatureAlgorithms
	}
// 客户端如果支持TLS1.3 ，需要带好相应的ECC椭圆参数
    var params ecdheParameters
	if hello.supportedVersions[0] == VersionTLS13 {
		hello.cipherSuites = append(hello.cipherSuites, defaultCipherSuitesTLS13()...)
		curveID := config.curvePreferences()[0]
		if _, ok := curveForCurveID(curveID); curveID != X25519 && !ok { ... }
		params, err = generateECDHEParameters(config.rand(), curveID)
		if err != nil { ... }
		hello.keyShares = []keyShare{{group: curveID, data: params.PublicKey()}}
	}
```
- `vers`: 协议版本（protocol version）指示客户端支持的最佳协议版本。一般默认是TLS1.2，如果要使用TLS1.3的话需要协议升级。
- `compressionMethods`: 压缩算法（compression method），一般被禁用
- `random`: 客户端随机数
- `sessionId`: 用来session恢复使用的
- `ocspStapling`: 支持ocsp装订，允许服务端请求CA的ocsp程序来证明自己的真实性。
- `scts`： scts(signed certificate timestamp support)scts主要用于certificate transparency。主要是为了避免某些ca滥发证书， 通过建立第三方审计服务， 让ca/域名拥有者/用户能够检查到该域名是否有被恶意签发。 （http://www.certificate-transparency.org/what-is-ct 、https://imququ.com/post/certificate-transparency.html）
- `supportedCurves` 和 `supportedPoints`: 支持的椭圆参数曲线。
- `nextProtoNeg` 和 `alpnProtocols`: 协议升级使用
- `supportedVersions`: 如果客户端支持TLS1.3则在此填写，服务端也支持TLS1.3的话后续会用TLS1.3 通信
- `supportedSignatrueAlgorithms`: 支持的签名算法，服务器提供的所有证书必须用 “signature_algoritms"中提供的 hash/signature 算法对之一签署，主要为RSA或ECDSA。常见的有 
 - rsa_pss_rsae_sha256 (0x0804)
 - ecdsa_secp256r1_sha256 (0x0403)
 
除此之外，client hello 阶段，客户端还携带了自己所支持的 ciphersuite 列表。服务端会挑选一个。
下图是wireshark抓包得到的golang 发出的client hello。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/client_hello.png" style="height:300px;border-radius: 20px;" >

-------------------------------------------------------

### 服务端处理

Server 收到 Client Hello 后的处理流程。

Server 会检查是否Session 复用，再做完整的握手过程。
代码在 `./tls/handshake_server.go` 的 `serverHandshake()` 方法中
服务端收到Client Hello后检查是否TLS 复用。

``` go
// 检查是否 TLS 会话复用
	if hs.checkForResumption() {
		// 如果客户端的握手信息包含Session Ticket，则进行简短的握手。
		c.didResume = true
	} else {
		// The client didn't include a session ticket, or it wasn't
		// valid so we do a full handshake.
		if err := hs.pickCipherSuite(); err != nil { return err }
		if err := hs.doFullHandshake(); err != nil { return err }
		if err := hs.establishKeys(); err != nil { return err }
		if err := hs.readFinished(c.clientFinished[:]); err != nil { return err}
		c.clientFinishedIsFirst = true
		c.buffering = true
		if err := hs.sendSessionTicket(); err != nil { return err }
		if err := hs.sendFinished(nil); err != nil { return err }
		if _, err := c.flush(); err != nil { return err }
	}
```
服务端处理后会将Server Hello ,Certificate，Server Key Exchange，Server Hello Down 全部发出。以下代码在 `./tls/handshake_server.go` 的 `doFullHandshake()` 方法中

#### Server Hello

上面的插播看到，Server会先从Client Hello 中的CipherSuite列表中挑选合适的。如果没问题的话会做剩下的握手动作。让我们看看Server端是如何处理的。
``` go
// 设置ocsp标志位
    if hs.clientHello.ocspStapling && len(hs.cert.OCSPStaple) > 0 { hs.hello.ocspStapling = true }
// 设置sessionticket 是否支持
	hs.hello.ticketSupported = hs.clientHello.ticketSupported && !c.config.SessionTicketsDisabled
	hs.hello.cipherSuite = hs.suite.id
// 将TLS版本和加密套件做一个hash，最后的Finished报文用
	hs.finishedHash = newFinishedHash(hs.c.vers, hs.suite)
	...
// 简单的处理就，生成 Server Hello 报文了（TLS1.3的流程不再这里）
    hs.finishedHash.Write(hs.hello.marshal())
	if _, err := c.writeRecord(recordTypeHandshake, hs.hello.marshal()); err != nil { return err }
    ... 
```
很多具体的动作都在`hs.hello.marshal()` 中进行。全部是位操作。
``` go
func (m *serverHelloMsg) marshal() []byte {
	if m.raw != nil {
		return m.raw
	}

	var b cryptobyte.Builder
	b.AddUint8(typeServerHello)
	b.AddUint24LengthPrefixed(func(b *cryptobyte.Builder) {
		b.AddUint16(m.vers)
		addBytesWithLength(b, m.random, 32)  // 设置服务端 Random ，这个也是最重要的一步
		b.AddUint8LengthPrefixed(func(b *cryptobyte.Builder) {
			b.AddBytes(m.sessionId)    // 添加session id
		})
		b.AddUint16(m.cipherSuite)     //  确定Server端选择的加密套件
		b.AddUint8(m.compressionMethod)  // 压缩方法（其实就是空）
    ...
    b.AddUint16(extensionNextProtoNeg)  
    b.AddUint16(extensionStatusRequest) // 如果选择要发起OCSP的话，扩展会带上
    b.AddUint16(extensionRenegotiationInfo)  // 重协商
    b.AddUint16(extensionSupportedVersions)  // 支持的版本
    b.AddUint16(extensionKeyShare)
 	b.AddUint16(extensionPreSharedKey)
	b.AddUint16(extensionCookie)
    b.AddUint16(extensionKeyShare)
    ...
```

go 实现的server hello 十分简短。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/server_hello.png" style="height:250px;border-radius: 20px;" />

#### Certificate

接下来就是生成Certificate报文。携带X.509证书链，证书链以ASN.1 DSR编码的一系列证书。主证书必须第一个发送，中间证书按照正确的顺序跟在后面，根证书需省略。
``` go
// 生成证书
	certMsg := new(certificateMsg)
	certMsg.certificates = hs.cert.Certificate
	hs.finishedHash.Write(certMsg.marshal())
	if _, err := c.writeRecord(recordTypeHandshake, certMsg.marshal()); err != nil { return err }
// 如果是支持ocspStapling的需要服务端发起 OCSP 请求,并生成
	if hs.hello.ocspStapling {
		certStatus := new(certificateStatusMsg)
		certStatus.response = hs.cert.OCSPStaple
		hs.finishedHash.Write(certStatus.marshal())
		if _, err := c.writeRecord(recordTypeHandshake, certStatus.marshal()); err != nil { return err }
     }
```
**Certficate 报文**
<img src="/images/network/TLSDetailAnalysis/Certificate.png" style="height:250px;border-radius: 20px;"/>
**OCSP 响应**（这个不是每次都会出现）
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/ocsp.png" style="height:200px;border-radius: 20px;"/>

#### Server Key Excahnge

这里由于加密套件选择的是 `TLS_ECDHE_RSA_WIRH_AES_128_GCM_SHA256` ，也就是要通过ECDHE进行密钥交换，ECDHE是DH的变形，加入ECC椭圆曲线后安全性和计算性能上得到大幅提升。
``` go
    keyAgreement := hs.suite.ka(c.vers)
	skx, err := keyAgreement.generateServerKeyExchange(c.config, hs.cert, hs.clientHello, hs.hello)
	if err != nil { ... }
	if skx != nil {
		hs.finishedHash.Write(skx.marshal())
		if _, err := c.writeRecord(recordTypeHandshake, skx.marshal()); err != nil { return err }
	}
```
这里可以看到生成交换的Key，generateServerKeyExchange() 传入的参数有handshake信息、证书 client和server的hello和client的hello。

来大致分析以下生成 DH 公钥的过程。代码在 `/tls/key_agreement.go` 的 `generateServerKeyExchange` 方法中，大约147行。

``` go
// 获取最佳的曲线列表
    preferredCurves := config.curvePreferences()
// 从最佳曲线和支持的曲线中匹配出一个曲线（其实做了一大堆都没啥用，go 只支持 x25519 曲线.....）
	var curveID CurveID
NextCandidate:
	for _, candidate := range preferredCurves {
		for _, c := range clientHello.supportedCurves { ... }
    }
// 使用随机数和曲线计算出 DH交换所需要的公钥 
    params, err := generateECDHEParameters(config.rand(), curveID)
	if err != nil { return nil, err }
	ka.params = params
	// See RFC 4492, Section 5.4.
	ecdhePublic := params.PublicKey()
	... 
	copy(serverECDHParams[4:], ecdhePublic)    
// 获取到签名内容 
signed := hashForServerKeyExchange(sigType, hashFunc, ka.version, clientHello.random, hello.random, serverECDHParams)

	signOpts := crypto.SignerOpts(hashFunc)
	if sigType == signatureRSAPSS {
		signOpts = &rsa.PSSOptions{SaltLength: rsa.PSSSaltLengthEqualsHash, Hash: hashFunc}
	}
	sig, err := priv.Sign(config.rand(), signed, signOpts)
	if err != nil { ... }
	skx := new(serverKeyExchangeMsg)
	sigAndHashLen := 0
	if ka.version >= VersionTLS12 { sigAndHashLen = 2 }
	skx.key = make([]byte, len(serverECDHParams)+sigAndHashLen+2+len(sig))
	copy(skx.key, serverECDHParams)
	k := skx.key[len(serverECDHParams):]
```

<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/server_key_exchange.png" style="height:300px;border-radius: 20px;" >

Server Key Exchange 主要是生成 DH 密钥交换的公钥，通过 `generateServerKeyExchange（）`方法拿到生成DH交换的公钥。通过 `随机数`和`curveID` 调用`generateECDHEParameters（）` 获取到ECDHE的公钥。

和RSA做密钥交换不同，证书里的公钥不做交换密钥使用，这也就是说服务端无法证明这个证书是属于自己的。因为RSA密钥交换，客户端拿证书中的公钥加密 pre master。服务端能用私钥解开就证明服务端拥有该证书。而DH 由于证书只用作身份验证，所以需要额外的签名认证。也就是 Server Key Exchange 报文下面的 Signature Algorithm，同样也为了证明签名的有效性，签名的函数`hashForServerKeyExchange()` 传入的参数为签名类型、hash函数、客户端和服务端的Random随机数和 DH 参数。来证明这个签名是本次加密通信使用的。

> ps: 证书是真的，但是属于不属于你就是另一回事了，因为任意一个站点的证书都是可以随便下载到的，所以服务端得证明证书属于自己。

- EC Diffie-Hellman Server Params
 - `Curve Type`: named_curve 
 - `Named Curve`: x25519  (曲线名字， go 只支持这一种.也是当前性能最好的椭圆曲线)
 - `Pubkey Length` 和 `Pubkey`: Diffie-Hellman 交换的公钥 (通过Server random 和 椭圆曲线 获得)
 - `Signature Algorithm`: Hash SHA256 、Signature Hash Algorithm Signature RSA ...
 
ECDHE_RSA 密钥交换算法的 SignatureAlgorithm 是 rsa 。 ECDHE_ECDSA 密钥交换算法的 SignatureAlgorithm 是 ecdsa。RSA 是 RSA证书，而ECDSA是ECC证书。

#### Server Hello Down

没啥好说的，最简单的一个结束符。ServerHelloDone消息表示，服务器已经发送完了密钥协商需要的消息，并且客户端可以开始客户端的密钥协商处理了。


``` go
// 封装Server Hello Down
	helloDone := new(serverHelloDoneMsg)
	hs.finishedHash.Write(helloDone.marshal())
	if _, err := c.writeRecord(recordTypeHandshake, helloDone.marshal()); err != nil { ... }
// 发送上面封装的所有Server 相关的报文一起发送
    if _, err := c.flush(); err != nil {
		return err
	}

```
Server 相关的记录全部发出。舞台交给客户端。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/server_hello_down.png" style="height:250px;border-radius: 20px;" />

### 客户端处理
服务端将服务端信息全部发送过来之后，会处理服务端发送来的信息，如本次握手如果客户端发送了Session Id 等信息，服务端是否成功复用了上次的会话，如证书信任问题处理，Client端DH公私钥生成等，

代码在 `./tls/handshake_client.go` 的 `handshake()` 方法中。代码先是客户端判断是否session 复用，读取server hello 内容，判断选择的TLS版本等，这块的内容忽略。

判断如果复用的话.
处理流程为1. 生成相关的Key 2. 读Session Ticket 3. 读Finished 报文 4. 发送缓冲区内容

如果没有复用的话.
处理流程则需要现做完整的握手

``` go
// 判断是否复用
	if isResume {
		if err := hs.establishKeys(); err != nil { return err }
		if err := hs.readSessionTicket(); err != nil { return err }
		if err := hs.readFinished(c.serverFinished[:]); err != nil { return err }
		c.clientFinishedIsFirst = false
		if err := hs.sendFinished(c.clientFinished[:]); err != nil { return err }
		if _, err := c.flush(); err != nil { return err }
	} else {
		if err := hs.doFullHandshake(); err != nil { return err }
		if err := hs.establishKeys(); err != nil { return err }
		if err := hs.sendFinished(c.clientFinished[:]); err != nil { return err }
		if _, err := c.flush(); err != nil { return err }
		c.clientFinishedIsFirst = true
		if err := hs.readSessionTicket(); err != nil { return err }
		if err := hs.readFinished(c.serverFinished[:]); err != nil { return err }
	}
```
此后则需要经历 Client Key Exchange，Change Cipher Speace， Encrpyted Handshake Record 等过程。代码在 `./tls/handshake_client.go` 中的 `doFullHandshake()` 中。
 
------------------------------------------

插播一段内容，在Client 做DH密钥交换之前，需先验证服务端证书。主要的函数为 `verifyServerCertificate()`
``` go
func (c *Conn) verifyServerCertificate(certificates [][]byte) error {
	certs := make([]*x509.Certificate, len(certificates))
// 解析证书内容
    ...
// InsecureSkipVerify 用来控制客户端是否证书和服务器主机名。如果设置为true,则不会校验证书以及证书中的主机名和服务器主机名是否一致。
	if !c.config.InsecureSkipVerify {
		opts := x509.VerifyOptions{
			Roots:         c.config.RootCAs,
			CurrentTime:   c.config.time(),
			DNSName:       c.config.ServerName,
			Intermediates: x509.NewCertPool(),
		}
		for _, cert := range certs[1:] {
			opts.Intermediates.AddCert(cert)
		}
		var err error
		c.verifiedChains, err = certs[0].Verify(opts)
		if err != nil { ... }
	}
// 校验对方证书
	if c.config.VerifyPeerCertificate != nil {
		if err := c.config.VerifyPeerCertificate(certificates, c.verifiedChains); err != nil { ... }
	}
  	...
}

```
除此之外在 `doFullHandshake()` 函数中处理 OCSP 内容，并且在` keyAgreement.processServerKeyExchange(c.config, hs.hello, hs.serverHello, c.peerCertificates[0], skx)` 读取Server Hello 的内容并处理。


 
#### Client Key Exchange
拿到Server 信息，加上Client的信息，可以生成Pre Master Key。
``` go
preMasterSecret, ckx, err := keyAgreement.generateClientKeyExchange(c.config, hs.hello, c.peerCertificates[0])
	if err != nil { ... }
	if ckx != nil {
		hs.finishedHash.Write(ckx.marshal())
		if _, err := c.writeRecord(recordTypeHandshake, ckx.marshal()); err != nil { ... }
	}
```
相比Server Key Exchange，Client 的很简单，就只有 DH 的公钥。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/client_key_exchange.png" style="height:200px;border-radius: 20px;" />

#### Change Cipher Speace

ChangeCipherSpec用来通知对端，开始启用协商好的Connection State做对称加密，内容只有1个字节。 这个协议是冗余的，在TLS 1.3里面直接被删除了。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/change_cipher_space.png" style="height:300px;border-radius: 20px;" />


不过在此发送此报文之前，所有的Key 都是建立好的，下面分析Key是如何计算得出的。

此时客户端已经可以计算出通信使用的 预主对称密钥（通过ECDH交换得到）。还需要将预主密钥加工生成主密钥，就可以完成通信了。主密钥的计算在 ` masterFromPreMasterSecret(c.vers, hs.suite, preMasterSecret, hs.hello.random, hs.serverHello.random)` 函数中，可以看到，主密钥依赖TLS版本、加密套件、预主密钥、客户端和服务端随机数。其中TLS版本和随机数生成一个种子。通过`prf伪随机函数`生成最终的Master Key。
```go
// masterFromPreMasterSecret generates the master secret from the pre-master
// secret. See RFC 5246, Section 8.1.
func masterFromPreMasterSecret(version uint16, suite *cipherSuite, preMasterSecret, clientRandom, serverRandom []byte) []byte {
	seed := make([]byte, 0, len(clientRandom)+len(serverRandom))
	seed = append(seed, clientRandom...)
	seed = append(seed, serverRandom...)

	masterSecret := make([]byte, masterSecretLength)
	prfForVersion(version, suite)(masterSecret, preMasterSecret, masterSecretLabel, seed)
	return masterSecret
}
```
再通过master secret导出密钥
``` go
func (hs *clientHandshakeState) establishKeys() error {
	c := hs.c

	clientMAC, serverMAC, clientKey, serverKey, clientIV, serverIV :=
		keysFromMasterSecret(c.vers, hs.suite, hs.masterSecret, hs.hello.random, hs.serverHello.random, hs.suite.macLen, hs.suite.keyLen, hs.suite.ivLen)
	var clientCipher, serverCipher interface{}
	var clientHash, serverHash macFunction
	if hs.suite.cipher != nil {
		clientCipher = hs.suite.cipher(clientKey, clientIV, false /* not for reading */)
		clientHash = hs.suite.mac(c.vers, clientMAC)
		serverCipher = hs.suite.cipher(serverKey, serverIV, true /* for reading */)
		serverHash = hs.suite.mac(c.vers, serverMAC)
	} else {
		clientCipher = hs.suite.aead(clientKey, clientIV)
		serverCipher = hs.suite.aead(serverKey, serverIV)
	}

	c.in.prepareCipherSpec(c.vers, serverCipher, serverHash)
	c.out.prepareCipherSpec(c.vers, clientCipher, clientHash)
	return nil
}
```

为什么要通过Master Key 导出 clientMAC, serverMAC, clientKey, serverKey, clientIV, serverIV 。比如streamcipher, 如果攻击者知道TLS数据流一个方向的部分明文，那么对2个方向的密文做一下xor，就能得到另一个方向对应部分的明文了。
还有AEAD也规定了不能使用相同的key+nonce来加密不同的明文，故如果TLS双方使用相同的key，又从相同的数字开始给nonce递增，那就不符合规定，会直接导致aes-gcm 被攻破。

例如CBC算法使用先HMAC后加密的方式， HMAC中使用MAC KEY来做认证。
serverKey和clientKey是对称密钥做记录层的对称加密。
如果使用CBC算法， 在TLS1.1之前，使用clientIV, serverIV作为IV。
AEAD算法先做加密后摘要的方式，更加安全， 不使用mac key。

More Info: [Distinction between the TLS PRF for the Master Key and for the Keys](https://cryptologie.net/article/341/distinction-between-the-tls-prf-for-the-master-key-and-for-the-keys/)

> 这里提一下，看到了吗？有clientMAC, serverMAC, clientKey, serverKey, clientIV, serverIV 六个值。虽然clientMAC和serverMAC 没啥用，因为有AEAD了。但是Key 和 IV 还是有用的，也就是如果内容是服务端发送的，则需要用 serverKey 和 serverIV 加密。客户端解密时也用的server Key 和 ServerIV。

#### Encrpyted Handshake Record
加密第一段内容。如果开了 false start ，不必等服务端回 ChangeCipherSpace 则可以直接发送内容。

### 完成握手
接下来服务端也会发送 ChangeCipherSpace和EncrpytedHandRecord来结束。服务端如果支持Session Tikect的话也可以发送，以支持下一次的快速握手。



## record protocol

一个TLS record Size 的最大为16k，也就是服务端（openssl）一般会开辟一个16K的 buffer存tls record size

![](https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/tcp_tls.png)

- 分片，逆向是重组
- 生成序列号，为每个数据块生成唯一编号，防止被重放或被重排序
- 压缩，可选步骤，使用握手协议协商出的压缩算法做压缩
- 加密，使用握手协议协商出来的key做加密/解密
- 算HMAC，对数据计算HMAC，并且验证收到的数据包的HMAC正确性（AES-GCM 有AEAD了就不需要HMAC了）
- 发给tcp/ip，把数据发送给 TCP/IP 做传输(或其它ipc机制)。


这里就只介绍解密过程吧，加密是解密的逆过程。其中seq是递增的，从加密第一块内容起递增（第一次加密发生在 Finshed 报文）

``` go
func TLSDecrypt(record []byte,version,cipherSuite uint16,clientRandom,serverRandom []byte,seq [8]byte) {
	suite := cipherSuiteByID(cipherSuite)
	_,master := GetMaster(hex.EncodeToString(clientRandom))
	byteMaster,_ := hex.DecodeString(master)
	cCipher,_,cHash,_,err := establishKeys(version,suite,byteMaster,clientRandom,serverRandom)
	...
	var clientConn = halfConn{
		err:nil,
		version:VersionTLS12,
		cipher:cCipher,
		mac:cHash,
		seq:seq,
	}

	plaintext,_,err := clientConn.decrypt(record)
	...
}

```


> 小赠，nginx的tls优化可以参考 https://blog.helong.info/blog/2015/05/08/https-config-optimize-in-nginx/




# 参考
- [0]: https://golang.org/pkg/crypto/tls/
- [1]:https://blog.helong.info/blog/2015/09/06/tls-protocol-analysis-and-crypto-protocol-design/#5-handshake-%E5%8D%8F%E8%AE%AE
- [2]: https://blog.wangriyu.wang/2018/03-http-tls.html
- [3]: http://www.cnhalo.net/2017/03/15/dive-into-tls-with-golang/