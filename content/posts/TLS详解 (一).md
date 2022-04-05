---
title: TLS 详解（一）
author: kiosk
tags:
  - tls
categories:
  - network
mathjax: true
date: 2020-05-02 23:31:00
---
TLS的设计目标是构建一个安全传输层（Transport Layer Security ），在基于连接的传输层（如tcp）之上提供安全加密的通信信道。它是旨在防止窃听，篡改和消息伪造的 [IETF](https://mailarchive.ietf.org/arch/msg/ietf-announce/IhM9JJHVs_ZeK-_1eaVZrqxbnL8/) 标准。常见应用程序包括Web浏览器，即时消息传递，电子邮件和IP语音都在使用TLS。
> Title From [What is Transport Layer Security (TLS)?](https://www.networkworld.com/article/2303073/lan-wan-what-is-transport-layer-security-protocol.html)
<!--more-->

-------------------------------------------------------

<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/encript-for-security.jpg" style="height:450px" >

- [TLS 详解（二）](https://kiosk007.top/blog/2020/05/04/tls-详解二/)
- [TLS 详解（三）](https://kiosk007.top/blog/2021/01/16/tls详解三/)

# TLS 简介


下面主要介绍一下TLS的设计目标和发展历史。

## TLS的设计目标

TLS用于两个应用程序之间提供保密性和数据完整性。一个TLS需要满足(1)、密码学安全 (2)、互操作，通用性 (3)、高效性 (4)、可扩展性 才能称为一个合格的TLS。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/TLS.png"/>

## TLS的发展历史

- 1994: **SSL 1.0**, 由Netscape提出，但未公开。
- 1995: **SSL 2.0**, 由Netscape提出，这个版本由于设计缺陷，并不安全，很快被发现有严重漏洞，已经废弃。
- 1996: **SSL 3.0**. 作为 RFC 6101 发布。2015年后已经不安全，必须禁用。
- 1999: **TLS 1.0**. 互联网标准化组织ISOC接替NetScape公司，发布了SSL的升级版TLS 1.0版.
- 2006: **TLS 1.1**. 作为 RFC 4346 发布。主要fix了CBC模式相关的如BEAST攻击等漏洞
- 2008: **TLS 1.2**. 作为RFC 5246 发布 ，增进安全性。此版本中的一个主要新功能是**身份验证（AEAD）加密**，它消除了对流和分组密码的需要（因而消除了固有的易受攻击的CBC模式）。
- 2011: 正式弃用SSL 2.0，IETF尝试通过发布RFC 6176正式弃用SSL v2 。根据SSL Labs的调查，2011年有54％的HTTPS服务器支持此过时的协议版本。
- 2013-2014: 多个针对TLS漏洞攻击手段相继爆出和同期发生的棱镜门事件也改变了人们对互联网安全的看法
 - AlFardan和Paterson发布了[pLucky 13](http://www.isg.rhul.ac.uk/tls/Lucky13.html)，他们对CBC套件的攻击。在TLS中，块加密旨在对纯文本（而不是密文）进行身份验证，这为攻击者提供了执行填充oracle攻击的机会。
 - 发现了[针对RC4的新攻击](http://www.isg.rhul.ac.uk/tls/)。以前，人们认为RC4弱点对TLS的影响不大，但这被证明是错误的。这项研究标志着RC4的死亡。
 - 爱德华·斯诺登（Edward Snowden）向英国卫报记者发布了数千份美国国家安全局（NSA）分类文件，从而永远改变了公众对互联网的看法。
 - [Dual EC DRBG](http://dualec.org/) (由NIST标准化生成和推广的伪随机数）被认为是潜在的后门。
 - 一项名为["三重握手攻击"](https://mitls.org/pages/attacks/3SHAKE)的新研究已经发布，TLS中的重新协商需要再次修复。
 - ["心脏出血攻击（heartbleed）"](http://heartbleed.com/)发现了OpenSSL（一个使用非常广泛的TLS库）中的一个严重漏洞。
 - [POODLE TLS](https://blog.qualys.com/ssllabs/2014/12/08/poodle-bites-tls)一些在SSL3.0 的填充攻击，即使TLS 1.0确实具有针对填充oracle攻击的内置防护，但某些实现仍无法正确避免。
- 2015：TLS证书的不安全再次让互联网安全遭受打击，各机构再次出击。
 - 从2015-2016，Apple，Chrome和Mozilla 相继因为一些CA机构违反规则等问题选择不再信任 WoSign、StartCom，2016年Chrome又宣布取消对Symantec（赛门铁克）证书的信任。DigiCert收购Symantec后，各厂商恢复信任Symantec
 - 由于多家CA机构存在乱签发CA的行为，经过多年的讨论，发布了[RFC 7469](https://tools.ietf.org/html/rfc7469)，使任何人都可以使用公钥固定来保护自己免受欺诈性颁发的证书的侵害。
 - [RFC 7633](https://tools.ietf.org/html/rfc7633) 中发布了新的X.509扩展，以将具有某些TLS功能的证书耦合在一起。这是一项迫切需要的功能，它在信任证书之前需要有效的OCSP响应。
- 2018: **TLS 1.3**，支持0-rtt，大幅增进安全性，砍掉了AEAD之外的加密方式


From [@SSL/TLS and PKI History][3]

# TLS CipherSuite

加密套间一般在 TLS 握手时协商得出，之后的整个加密过程都遵循套件的约定进行加密通信，每个集合的名称代表组成它的特定算法。

以TLS1.2中最常见的 **`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`** 为例。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/tls_ciphersuite.png" style="height:200px" >
- `密钥交换算法` -规定交换对称密钥的方式；
- `身份验证算法` -指示如何执行服务器身份验证和（如果需要）客户端身份验证。 
- `对称加密算法` -指示将使用哪种对称密钥算法来加密实际数据；包含对称加密算法、强度、工作模式。
- `消息身份验证代码（MAC）算法` -指示连接将用于执行数据完整性检查的方法。

在tls的世界有许多密码学名词，组合在一起就成了密码套件。
1. 块加密算法 block cipher : AES, Serpent, 等
2. 流加密算法 stream cipher: RC4，ChaCha20 等
3. Hash函数 hash funtion:MD5，sha1，sha256，sha512 , ripemd 160，poly1305 等
4. 消息验证码函数 message authentication code: HMAC-sha256，AEAD 等
5. 密钥交换 key exchange: DH，ECDH，RSA，DHE，ECDHE 等
6. 公钥加密 public-key encryption: RSA，rabin-williams 等
7. 数字签名算法 signature algorithm: RSA，DSA，ECDSA 等
8. 密码衍生函数 key derivation function: TLS-12-PRF(SHA-256) , bcrypto，scrypto，pbkdf2 等


## 非对称加密/密钥交换/身份验证

非对称加密是一种使用公钥和私钥加解密的算法，一段明文如果用公钥加密，只能用私钥解密。如果用私钥加密，只能用公钥解密。公钥可以公开，但是私钥只能是一方保存。

### RSA 算法

**一. 随机选择两个不相等的质数 p 和 q , p 与 q 越大，越安全**

比如 P = 67 ，Q = 71。计算他们的乘积 n = P * Q = 4757 ，转化为二进为 1001010010101，该加密算法即为 13 位，实际算法是 1024 位 或 2048 位，位数越长，算法越难被破解。

**二. 计算 p 和 q 的乘积 n, 并计算 n 的欧拉函数 φ(n)**

φ(n) 表示在小于等于 n 的正整数之中，与 n 构成互质关系的数的个数。例如：在 1 到 8 之中，与 8 形成互质关系的是1、3、5、7，所以 φ(n) = 4。 如果 n = P * Q，P 与 Q 均为质数，则 φ(n) = φ(P * Q)= φ(P - 1)φ(Q - 1) = (P - 1)(Q - 1) 。 本例中 φ(n) = 66 * 70 = 4620，这里记为 m， m = φ(n) = 4620

**三. 随机选择一个整数 e，条件是1< e < m，且 e 与 m 互质**

公约数只有 1 的两个整数，叫做互质整数，这里我们随机选择 e = 101 请注意不要选择 4619，如果选这个，则公钥和私钥将变得相同。

**四. 计算模反元素 d，可以使得 e*d 除以 m 的余数为 1。** 

即找一个整数 d，使得  (e * d ) % m = 1。 等价于 e * d - 1 = y * m ( y 为整数） 找到 d ，实质就是对下面二元一次方程求解。 e * x - m * y =1 ，其中 e = 101，m = 4620 101x - 4620y =1  这个方程可以用"扩展欧几里得算法"求解，此处省略具体过程。 总之算出一组整数解（x，y ）= （ 1601，35），即 d = 1601。 到此密钥对生成完毕。不同的 e 生成不同的 d，因此可以生成多个密钥对。本例中公钥为 （n，e) = (4757 , 101)，私钥为 （n，d) = (4757 ，1601) ，仅（n，e) = (4757 , 101) 是公开的，其余数字均不公开。可以想像如果只有 n 和 e，如何推导出 d，目前只能靠暴力破解，位数越长，暴力破解的时间越长。

**RSA 加密解密流程**

- 加密：c=$m^k\pmod n$  
- 解密：m=$c^d\pmod n$   

 m 是明文，c 是密文

> 缺点： RSA 不具备前向保密性，也就是加密的通信所依赖的 Pre Master Key 是基于公钥加密，私钥解密的。如果私钥丢失，前面加密过的会话如果被保存下来，都可以用泄漏的私钥还原的。

#### 基于openssl实战验证RSA

- 待加密的明文

``` bash
➜ echo "hello world" > hello.txt
```

- 生成私钥

```bash
➜ openssl genrsa -out private.pem
```

- 从私钥中提取公钥

``` bash
➜ openssl rsa -in private.pem -pubout -out public.pem
```

- 加密

``` bash
➜ openssl rsautl -encrypt -in hello.txt -inkey public.pem -pubin -out hello.en
```

- 解密

``` bash
➜ openssl rsautl -decrypt -in hello.en  -inkey private.pem -out hello.de
➜ cat hello.de
hello world
```

#### RSA 握手

<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/ssl_handshake_rsa.jpg
" style="height:600px" >


### DH密钥交换

1976年有 Bailey Whitfield Diffe 和 Martin Edward Hellman 首次发表，所以也被称作 Diffie-Hellman key exchange，简称DH。可以让双方在完全没有对方任何预先信息的条件下建立一个密钥。所以其具备向前安全性。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/Diffie-Hellman.png
" style="height:200px" >

- g、p、A、B 公开。A 为 Alice 的公钥，B 为 Bob 的公钥
- a、b 保密。 a 为 Alice 的私钥匙，b 为 Alice 的私钥匙

**DH 密钥交换协议举例**
- 协定使用 p=23 以及 g=5
- Alice选择密钥 a=6, 计算公钥 $A=g^a\pmod p$ ,并发送给Bob (A=$5^6\pmod{23} = 8$)
- Bob 选择密钥 b=15，计算公钥 $B=g^b\pmod p$ ,并发送给Alice (A=$5^{15}\pmod{23} = 19$)
- Alice 计算 $s=B^a\pmod p$ (s=$19^6\pmod{23}=2$)
- Bob 计算 $s=A^b\pmod p$ (s=$8^{15}\pmod{23}=2$)

> 缺点：大数乘法运算，速度很慢。实际使用的大多都是 ECDH 交换算法，也就是ECC椭圆曲线+DH

**ECDH**
ECDH 是椭圆曲线的（DH）笛福赫尔曼算法的变种，它其实不单单是一种加密算法，而是一种密钥协商协议，也就是说 ECDH 定义了（在某种程度上）密钥怎么样在通信双方之间生成和交换，至于使用这些密钥怎么样来进行加密完全取决通信双方。

ECDH比DH计算更快，同等安全程度下，密钥比DH更短。

#### DH 握手

<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/ssl_handshake_diffie_hellman.jpg
" style="height:600px" >




## 对称加密/消息身份验证

对称加密是一种使用单个密钥对数据进行加密（编码）和解密（解码）的加密方法。其明文的加密使用的是相同的一把密钥。

### XOR 与 填充

对称加密的基本原理是XOR (异或运算)。

<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/XOR.png" style="height:200px" >
但是密钥和明文存在长度不一致的情况。为了将密钥和明文对齐，这就引入了分组的概念，对称加密中分为块加密和流式加密。以块加密为例。

Block cipher 加密方式：将明文分为多个等长的Block块，再对每个模块分别加解密。若当最后一个明文block模块长度不够时，需要引入填充算法进行填充。
填充方案
- 位填充：以bit为单位填充
- 字节填充：以字节为单位填充
 - 补零：缺少字节数全部补0
 - ANSI X9.23: 全部填0，但最后一个字节需要补充总共填写0个个数
 - ISO 10126：完全随即字符，但最后一个字节需要补充总共填写随即字符个数 
 - PKCS7（RFC5652）：全部填充缺省的字节个数，如缺4个字节，填4个04
 
### 工作模式


#### ECB(Electronic codebook) 模式

ECB是DES的最简单和最弱的形式。它不使用初始化向量, 直接将明文分解为多个块，对每个块作独立的加密。其最致命的弱点就是无法隐藏数据特征。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/ECB.jpg" style="height:400px" >
图a 是加密前的明文， 图b 是ECB加密后的密文。可以看出，加密根本没有什么用处。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/ECB_encryption.png" style="height:250px" >

以下是zoom在其加密白皮书（2020）上所宣传的，其实很不安全。
> For use cases such as meeting real-time content (video, voice, and content share), where data is transmitted over User
Datagram Protocol (UDP), we use AES-256 in ECB mode to encrypt these compressed data streams. We expect to upgrade
this soon to AES-256 GCM. Additionally, for video, voice, and content share encrypted with AES, once it’s encrypted, it
remains encrypted as it passes through Zoom’s meeting servers until it reaches another Zoom Client or a Zoom Connector,
which helps translate the data to another protocol.

Link: [Zoom的加密算法，到底有什么问题？](https://xie.infoq.cn/article/644f4ed6b8a84286cc9f74ebc)

#### CBC(Cipher-block chaining) 模式

CBC 模式是最常见的传统加密模式。每个明文块先与前一个密文块异或后，再进行加密。由于此XOR处理，相同的明文块将不再导致加密产生相同的密文。

**缺点:** 加密过程串行化，只有第一步完成才能接下来的第二步，多核CPU无法发挥作用。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/CBC_encryption.png
" style="height:250px">
第一组通过初始化向量 IV 将第一组明文和 密钥K 加密得到密文，第二组再用第一组得到的密文和第二组的明文和 密钥K 加密得到第二组的密文，如此往复。

> 1. IV不是密码短语的一部分，因为它不是秘密。它是作为随机数生成的，但是只有在了解此IV的情况下，您才能解密密钥本身。
> 2. 因为漏洞（Lucky 13）问题，被高版本TLS所宣布禁止，另外H2的实现也不允许使用CBC模式的TLS。

#### CTR（Counter）模式

为了解决CBC不能串行加密的缺点。CTR模式支持每组加密采用一个计数向量，这样就可以多个块同时加密。它不具有消息依赖性，因此密文块不依赖于先前的明文块。

**缺点：** 不能提供密文的消息完整性校验。

<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/CTR_encryption.svg
" style="height:250px" >

#### GCM（Galois Counter Mode）模式

GCM 是 CTR + GMAC ，其兼顾CTR的并法加解密的特点，也可以使用MAC算法实现消息的完整性验证。
- 验证完整性 MAC(Message Authentication Code)

通过将加密后的密文消息和密钥经过专门的Hash函数加密，传输时需要将密文和MAC一块传给接收方。接收方通过相同的MAC算法，将密文和密钥再次算出MAC值，判断MAC值是否相等。

可以使用安全伪随机函数（[PRF](http://www.crypto-it.net/eng/theory/prf-and-prp.html)）创建安全MAC算法。

<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/Message_Authentication_Code.png
" style="height:250px">

顾名思义，可以验证消息完整性，防止中间人恶意修改消息内容。
- GCM

GMAC 是用到 Galois域的MAC算法，是众多消息验证算法的一种。
<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/two-parts-of-gcm-l.jpg
" style="height:350px" >

- AEAD

Authenticated Encryption with Associated Data (AEAD) 是一种同时具备保密性，完整性和可认证性的加密形式。

AEAD 产生的原因很简单，单纯的对称加密算法，其解密步骤是无法确认密钥是否正确的。也就是说，加密后的数据可以用任何密钥执行解密运算，得到一组疑似原始数据，而不知道密钥是否是正确的，也不知道解密出来的原始数据是否正确。因此，需要在单纯的加密算法之上，加上一层验证手段，来确认解密步骤是否正确。

`AES-256-GCM` 就是常见的 AEAD 算法。其他的还有诸如 `ChaCha20-Poly1305`，`AES-128-GCM` 等

### AES(Advanced Encryption Standard) 加密算法

AES是当今可能会遇到的流行和被广泛采用的对称加密算法。它由比利时密码学家 Joan Daemen 和 Vincent Rijmen 所设计，常用的填充算法为：PKCS7。常用的分组模式为：GCM。分组长度只能是16字节。

| AES      | 密钥长度（32 bit）  |  分组长度（32 bit）  |  加密轮数    |
| -------- | :-----:           | :----:             | :----:     | 
| AES-128  | 4         		   |   4                |    10      |
| AES-192  | 6                 |   4                |    12      |
| AES-256  | 8                 |   4                |    14      |

**AES加密步骤**
1. 将明文按照128 bit (16 字节)拆分为若干个明文块，每个明文块是 4\*4 矩阵
2. 按照选择的填充方式来填充最后一个明文块。（PKCS7）
3. **每一个明文块利用AES加密器和密钥，加密成密文块。**
4. 拼接所有的密文块，加上上文提到的加密模式GCM。成为最终的密文。

下面是详细的加密步骤。

第三步是最复杂，也是最重要的一步。也是上述加密轮数体现的地方。

#### AES 加密流程

AES中的加密回合数是可变的，并且取决于密钥的长度。AES对128位密钥使用10轮，对192位密钥使用12轮，对于256位密钥使用14轮。这些回合中的每个回合都使用不同的128位回合密钥，该密钥是根据原始AES密钥计算得出的。
大体分为 `初始轮` 、 `普通轮` 、 `最终轮`

**初始轮**
- `AddRoundKey 轮密钥加`  将明文矩阵的16个字节视为128位，并与回合密钥的128位进行XOR运算。

**普通轮**
- `AddRoundKey 轮密钥加`  
- `Subytes 字节替代`   通过查找设计中给定的固定表（S-box）来替换16个输入字节。结果是四行四列的矩阵。
- `ShiftRows 行移位`    矩阵的四行中的每一行都向左移动。第一行不变，第二行左移1个字节，第三行左移2个字节，第四行左移3个字节。
- `MixColumns 列混合`  使用特殊的数学函数对每四个字节的列进行转换。此函数将一列的四个字节作为输入，并输出四个全新的字节，以替换原始列。结果是由16个新字节组成的另一个新矩阵。

**最终轮**
- `SubBytes 字节替代`
- `ShiftRows 行移位`
- `AddRoundKey 轮密钥加`

<img src="https://img1.kiosk007.top/static/images/network/TLSDetailAnalysis/AES_Encrpyt.png
" style="height:350px" >

- 动画演示 `AES加密` : https://coolshell.cn/articles/3161.html

``` c
/**
 * 参数 p: 明文的字符串数组。
 * 参数 plen: 明文的长度。
 * 参数 key: 密钥的字符串数组。
 */
void aes(char *p, int plen, char *key){
 
    int keylen = strlen(key);
    if(plen == 0 || plen % 16 != 0) {
        printf("明文字符长度必须为16的倍数！\n");
        exit(0);
    }
 
    if(!checkKeyLen(keylen)) {
        printf("密钥字符长度错误！长度必须为16、24和32。当前长度为%d\n",keylen);
        exit(0);
    }
 
    extendKey(key);//扩展密钥
    int pArray[4][4];
 
    for(int k = 0; k < plen; k += 16) {
        convertToIntArray(p + k, pArray);
        addRoundKey(pArray, 0);//一开始的轮密钥加
 
        for(int i = 1; i < 10; i++){//前9轮
            subBytes(pArray);//字节代换
            shiftRows(pArray);//行移位
            mixColumns(pArray);//列混合
            addRoundKey(pArray, i);
        }
 
        //第10轮
        subBytes(pArray);//字节代换
        shiftRows(pArray);//行移位
        addRoundKey(pArray, 10);
        convertArrayToStr(pArray, p + k);
    }
}
```

更多有关对称加密的实现可以参考： https://tools.ietf.org/html/rfc5246#section-6.3


# TLS的分层结构

TLS 从实现上主要包含 `Record 记录协议` 和 `Handshake 握手协议`。
- **handshake protocol**，验证通讯双方的身份，交换加解密的安全套间，协商加密参数
- **record protocol**，记录层作为对称加密传输的record协议

另外还有三种辅助协议
- **changecipher spec** ，用来通知对端从handshake切换到record协议(有点冗余，在TLS1.3里面已经被删掉了)
- **alert协议，the alert**, 用来通知各种返回码，
- **application data协议**，就是把http，smtp等的数据流传入record层做处理并传输。

这种 **认证密钥协商** + **对称加密传输** 的结构，是绝大多数加密通信协议的通用结构。

## record protocol

record协议做应用数据的对称加密传输，占据一个TLS连接的绝大多数流量。记录层将信息块分割成携带 2^14 字节 (16KB) 或更小块的数据的 TLSPlaintext 记录。有点像TCP中的segment，所有的其他子协议需要通过 record 协议封装，多个record 数据可以在一个TCP包里记录一次性发出。
<img src="https://img1.kiosk007.top/static/images/network/tls_record.png" style="height:390px" >
Record 协议 – 从应用层接受数据:
- 分片，逆向是重组
- 生成序列号，为每个数据块生成唯一编号，防止被重放或被重排序
- 压缩，可选步骤，使用握手协议协商出的压缩算法做压缩 （大多不压缩）
- 加密，使用握手协议协商出来的key做加密/解密
- 算HMAC，对数据计算HMAC，并且验证收到的数据包的HMAC正确性
- 发给tcp/ip，把数据发送给 TCP/IP 做传输(或其它ipc机制)。

## handshake protocol

handshake 是 TLS里最复杂的子协议，浏览器和服务器在握手过程中协商TLS版本号、随机数、密码套件等信息，并且交换证书、身份验证、交换密钥参数等。最终用于后续的混合加密系统。
handshake在TLS的发展中又产生了多个变种，比如RSA密钥交换DH密钥交换。0RTT，false start 等等。
```bash
      Client                                               Server
      ClientHello
      (empty SessionTicket extension)------->
                                                      ServerHello
                                  (empty SessionTicket extension)
                                                     Certificate*
                                               ServerKeyExchange*
                                              CertificateRequest*
                                   <--------      ServerHelloDone
      Certificate*
      ClientKeyExchange
      CertificateVerify*
      [ChangeCipherSpec]
      Finished                     -------->
                                                 NewSessionTicket
                                               [ChangeCipherSpec]
                                   <--------             Finished
      Application Data             <------->     Application Data
   Figure 1: Message flow for full handshake issuing new session ticket
```

# 参考：

- [0]: https://blog.helong.info/blog/2015/09/06/tls-protocol-analysis-and-crypto-protocol-design/
- [1]: https://blog.wangriyu.wang/2018/03-http-tls.html 
- [3]: https://www.feistyduck.com/ssl-tls-and-pki-history/ （TLS发展史）

**加密套件相关**
- [ECB]: https://www.sciencedirect.com/topics/computer-science/electronic-code-book
- [CBC]: https://www.sciencedirect.com/topics/computer-science/cipher-block-chaining
- [CTR]: https://www.cryptopp.com/wiki/CTR_Mode
- [MAC]: http://www.crypto-it.net/eng/theory/mac.html
- [AES]：https://www.tutorialspoint.com/cryptography/advanced_encryption_standard.htm
- [AEAD]: https://zhuanlan.zhihu.com/p/28566058
- [Asymmetric encryption]: https://www.cloudflare.com/learning/ssl/what-is-asymmetric-encryption/
- [RSA]: https://www.zhihu.com/search?type=content&q=RSA  (知乎 一文搞懂 rsa 算法)