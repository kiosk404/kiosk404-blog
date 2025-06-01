## RSA 握手解析


<style>
    /* 终端风格样式 */
    .terminal-container {
        font-family: 'Consolas', 'Monaco', 'Andale Mono', 'Ubuntu Mono', monospace;
        background-color: #282c34; /* 深色背景 */
        color: #abb2bf; /* 终端文字颜色 */
        border: 1px solid #61afef; /* 蓝色边框 */
        box-shadow: 0 0 10px rgba(0, 0, 0, 0.5); /* 阴影效果 */
        padding: 1.5em;
        overflow-x: auto; /* 允许横向滚动 */
        border-radius: 0.5em;
    }
    .terminal-line {
        margin-bottom: 0.2em;
        line-height: 1.4;
    }
    .terminal-line code {
        color: #98c379; /* 代码块颜色 */
        background-color: transparent;
        padding: 0;
    }
    .terminal-line .comment {
        color: #5c6370; /* 注释颜色 */
    }
    .terminal-line .client-msg {
        color: #e06c75; /* 客户端消息颜色 */
    }
    .terminal-line .server-msg {
        color: #61afef; /* 服务器消息颜色 */
    }
    .terminal-line .arrow-right {
        color: #61afef; /* 右箭头颜色 */
        font-weight: bold;
        display: inline-block;
        width: 100%;
        text-align: center;
        margin: 0.5em 0;
    }
    .terminal-line .arrow-left {
        color: #c678dd; /* 左箭头颜色 */
        font-weight: bold;
        display: inline-block;
        width: 100%;
        text-align: center;
        margin: 0.5em 0;
    }
    .terminal-line .arrow-both {
        color: #e5c07b; /* 双向箭头颜色 */
        font-weight: bold;
        display: inline-block;
        width: 100%;
        text-align: center;
        margin: 0.5em 0;
    }
    .indent-client {
        padding-left: 0;
    }
    .indent-server {
        padding-left: 40%; /* 模拟服务器消息的缩进 */
    }
</style>

<div class="terminal-container">
    <div class="terminal-line comment"># TLS RSA 握手流程</div>
    <div class="terminal-line comment"># ------------------------------------------------------------------</div>
    <div class="terminal-line indent-client client-msg">Client: <code>ClientHello</code></div>
    <div class="terminal-line indent-client comment">  &nbsp;- Client Random: [32-byte random number]</div>
    <div class="terminal-line indent-client comment">  &nbsp;- Supported Cipher Suites: [...]</div>
    <div class="terminal-line indent-client comment">  &nbsp;- Supported Curves: [...]</div>
    <div class="terminal-line indent-client comment">  &nbsp;- TLS Version: [...]</div>
    <div class="terminal-line indent-client comment">  &nbsp;- Other Extensions: (e.g., SessionTicket, Point Formats)</div>
    <div class="terminal-line arrow-right">---&gt;</div>
    <div class="terminal-line indent-server server-msg">Server: <code>ServerHello</code></div>
    <div class="terminal-line indent-server comment">  &nbsp;- TLS Version: [...]</div>
    <div class="terminal-line indent-server comment">  &nbsp;- Server Random: [32-byte random number]</div>
    <div class="terminal-line indent-server comment">  &nbsp;- Cipher Suite: [...]</div>
    <div class="terminal-line indent-server comment">Server: (empty SessionTicket extension)</div>
    <div class="terminal-line indent-server server-msg">Server: <code>Certificate*</code> <span class="comment">(服务器证书)</span></div>
    <div class="terminal-line indent-server server-msg">Server: <code>ServerHelloDone</code></div>
    <div class="terminal-line arrow-left">&lt;---</div>
    <div class="terminal-line indent-client client-msg">Client: <code>ClientKeyExchange</code> <span class="comment">(客户端密钥交换)</span></div>
    <div class="terminal-line indent-client client-msg">Client: <code>CertificateVerify*</code> <span class="comment">(证书验证, 如果提供证书)</span></div>
    <div class="terminal-line indent-client client-msg">Client: <code>[ChangeCipherSpec]</code> <span class="comment">(改变加密规范)</span></div>
    <div class="terminal-line indent-client client-msg">Client: <code>Finished</code> <span class="comment">(握手完成)</span></div>
    <div class="terminal-line arrow-right">---&gt;</div>
    <div class="terminal-line indent-server server-msg">Server: <code>NewSessionTicket</code> <span class="comment">(新会话票据, 可选)</span></div>
    <div class="terminal-line indent-server server-msg">Server: <code>[ChangeCipherSpec]</code> <span class="comment">(改变加密规范)</span></div>
    <div class="terminal-line indent-server server-msg">Server: <code>Finished</code> <span class="comment">(握手完成)</span></div>
    <div class="terminal-line arrow-left">&lt;---</div>
    <div class="terminal-line indent-client client-msg">Client: <code>Application Data</code></div>
    <div class="terminal-line arrow-both">&lt;---&gt;</div>
    <div class="terminal-line indent-server server-msg">Server: <code>Application Data</code></div>
    <div class="terminal-line comment"># ------------------------------------------------------------------</div>
    <div class="terminal-line comment">Figure 1: Message flow for full handshake issuing new session ticket</div>
</div>

**TLS 握手过程：RSA 密钥交换**

RSA 密钥交换是 TLS 1.2 及更早版本中常用的一种密钥交换机制。它的核心思想是客户端使用服务器的证书公钥加密一个“预主密钥”(PreMasterSecret)，然后发送给服务器，服务器用自己的私钥解密得到预主密钥。

## ECDHE 握手解析


<div class="terminal-container">
    <div class="terminal-line comment"># TLS ECDHE 握手流程</div>
    <div class="terminal-line comment"># ------------------------------------------------------------------</div>
    <div class="terminal-line indent-client client-msg">Client: <code>ClientHello</code></div>
    <div class="terminal-line indent-client comment">  - Client Random: [32-byte random number]</div>
    <div class="terminal-line indent-client comment">  - Supported Cipher Suites: [...]</div>
    <div class="terminal-line indent-client comment">  - Supported Curves: [...]</div>
    <div class="terminal-line indent-client comment">  - TLS Version: [...]</div>
    <div class="terminal-line indent-client comment">  - Other Extensions: (e.g., SessionTicket, Point Formats)</div>
    <div class="terminal-line arrow-right">---&gt;</div>
    <div class="terminal-line indent-server server-msg">Server: <code>ServerHello</code></div>
    <div class="terminal-line indent-server comment">Server: (Selected Cipher Suite, Curve, Point Format)</div>
    <div class="terminal-line indent-server server-msg">Server: <code>Certificate*</code> <span class="comment">(服务器证书, 例如 RSA 或 ECDSA)</span></div>
    <div class="terminal-line indent-server server-msg">Server: <code>ServerKeyExchange</code> <span class="comment">(ECDHE 公钥 &amp; 握手消息签名)</span></div>
    <div class="terminal-line indent-server server-msg">Server: <code>CertificateRequest*</code> <span class="comment">(证书请求, 可选)</span></div>
    <div class="terminal-line indent-server server-msg">Server: <code>ServerHelloDone</code></div>
    <div class="terminal-line arrow-left">&lt;---</div>
    <div class="terminal-line indent-client client-msg">Client: <code>Certificate*</code> <span class="comment">(客户端证书, 如果被请求)</span></div>
    <div class="terminal-line indent-client client-msg">Client: <code>ClientKeyExchange</code> <span class="comment">(ECDHE 公钥)</span></div>
    <div class="terminal-line indent-client client-msg">Client: <code>CertificateVerify*</code> <span class="comment">(证书验证, 如果提供证书)</span></div>
    <div class="terminal-line indent-client client-msg">Client: <code>[ChangeCipherSpec]</code> <span class="comment">(改变加密规范)</span></div>
    <div class="terminal-line indent-client client-msg">Client: <code>Finished</code> <span class="comment">(握手完成)</span></div>
    <div class="terminal-line arrow-right">---&gt;</div>
    <div class="terminal-line indent-server server-msg">Server: <code>NewSessionTicket*</code> <span class="comment">(新会话票据, 可选)</span></div>
    <div class="terminal-line indent-server server-msg">Server: <code>[ChangeCipherSpec]</code> <span class="comment">(改变加密规范)</span></div>
    <div class="terminal-line indent-server server-msg">Server: <code>Finished</code> <span class="comment">(握手完成)</span></div>
    <div class="terminal-line arrow-left">&lt;---</div>
    <div class="terminal-line indent-client client-msg">Client: <code>Application Data</code></div>
    <div class="terminal-line arrow-both">&lt;---&gt;</div>
    <div class="terminal-line indent-server server-msg">Server: <code>Application Data</code></div>
    <div class="terminal-line comment"># ------------------------------------------------------------------</div>
    <div class="terminal-line comment">Figure 1: Message flow for full handshake issuing new session ticket</div>
</div>

**TLS 握手过程：ECDHE 密钥交换**


ECDHE (Elliptic Curve Diffie-Hellman Ephemeral) 是一种基于椭圆曲线的迪菲-赫尔曼密钥交换协议，双方各自临时生成密钥，交换临时公钥，共同计算预主密钥。它提供**前向保密性 （Forward Secrecy）**，这是 RSA 密钥交换所不具备的关键特性。

## TLS RSA 与 ECDHE 密钥交换核心区别

| 区别点           | RSA 密钥交换 (传统)                                     | ECDHE 密钥交换 (现代，推荐)                                         |
| :--------------- | :------------------------------------------------------- | :------------------------------------------------------------------ |
| **密钥交换方式** | 客户端使用**服务器的公钥加密** PreMasterSecret，并发送给服务器。 | 双方**各自临时生成密钥对**，交换临时公钥，共同计算出 PreMasterSecret。 |
| **核心安全性** | **无前向保密性** (Forward Secrecy)。如果服务器的长期私钥泄露，历史通信可被解密。 | **具有前向保密性** (Forward Secrecy)。即使长期私钥泄露，历史会话仍安全。 |
| **服务器私钥作用** | 直接用于**解密**客户端发送的加密 PreMasterSecret。       | 仅用于**签名**服务器的临时公钥，以验证身份，不直接用于密钥加密。 |
| **性能考量** | 客户端的加密和服务器的解密操作计算量相对较大。             | 双方的密钥生成和计算量相对较轻，且可并行，性能更优。           |
| **握手消息** | 通常没有独立的 `ServerKeyExchange` 消息。                 | 必须有 `ServerKeyExchange`，用于传输临时公钥和签名。              |
| **发展趋势** | 在 TLS 1.2 及之前版本中普遍使用，但逐渐被弃用。           | 现代 TLS 1.2 和 TLS 1.3 中**主流且推荐**的密钥交换方式。        |

## TLS 如何优化

### TLS1.2 False Start
TLS False Start 是一种允许客户端在握手完成之前就发送加密应用数据的优化，可以减少客户端的感知延迟。

**原理：**

- 客户端在发送完 `ClientKeyExchange` (或者 `CertificateVerify` 等相关握手消息) 和 `ChangeCipherSpec` 后，不等服务器的 `Finished` 消息，就立即发送加密的应用数据。
服务器在收到客户端的 `ClientKeyExchange` 和 `ChangeCipherSpec`，并且自己发送了 `Finished` 消息后，就可以开始解密并处理客户端的应用数据。

**优化效果：**

- 减少客户端等待时间： 客户端无需等待服务器的 `Finished` 消息，可以立即开始发送数据。从客户端角度看，数据发送可以减少 `0.5-RTT` 的延迟。
- 并非减少握手 RTT： 握手本身仍然需要 `2-RTT` 才能完整完成（双方都收到对方的 `Finished` 消息），`False Start` 优化的是数据发送的起始时间。

> 参考：https://segmentfault.com/a/1190000004003319

### TLS1.2 Session Tickets
这是最常见的优化方式之一，用于客户端和服务器之间已经建立过 TLS 连接并协商过加密参数的情况。当它们再次连接时，可以尝试恢复之前的会话，而不是执行完整的握手。

**原理：**

- 第一次握手：
    - 在第一次完整握手成功后，服务器会加密协商好的会话状态信息（包括 `Master Secret`），生成一个“会话票据”(`Session Ticket`)。
    - 这个票据通过 `NewSessionTicket` 消息发送给客户端。
    - 服务器不需要在自己的内存中保存这个会话状态。
- 第二次握手：
    - 客户端在 `ClientHello` 消息中，通过 `SessionTicket` 扩展将这个会话票据发送给服务器。
    - 服务器收到票据后，使用一个只有自己知道的密钥（`Ticket Key`）解密票据。
    - 如果解密成功，并且票据有效（未过期），服务器就能从中恢复出之前的会话状态（包括 Master Secret）。
    - 后续流程与 `Session ID` 会话恢复类似，服务器直接发送 `ServerHello `(不含 `Session ID`，或者包含一个新生成的 `Session ID`，但会话数据来自票据) 和 `ChangeCipherSpec`、`Finished`。
**优化效果**： 同样将 `2-RTT` 的完整握手减少到 `1-RTT`。


### TLS1.3 协议升级

`TLS 1.3` 是传输层安全协议（TLS）的最新主要版本，旨在显著提升性能、增强安全性和简化协议复杂性。相较于其前身 `TLS 1.2`，`TLS 1.3` 移除了许多过时且存在安全风险的特性（如 `RC4`、`SHA-1`、`CBC` 模式的密码套件），并强制使用现代的、具有前向保密性（Forward Secrecy）的加密算法。最引人注目的改进包括将默认握手延迟从 `2-RTT` 缩短到 `1-RTT`，并通过预共享密钥（`PSK`）实现 `0-RTT` 恢复握手，大大加快了连接建立速度。同时，协议本身的简化和加密范围的扩大，也使得 `TLS 1.3` 更易于实现、更不容易出错，并且对未来的密码学发展保持了更好的适应性。

| 升级类别     | TLS 1.2 的问题/特点                               | TLS 1.3 的改进/新特性                                      |
| :----------- | :------------------------------------------------ | :--------------------------------------------------------- |
| **性能优化** | | |
| **握手延迟** | 完整握手通常需要 **2-RTT**。<br>会话恢复需 **1-RTT**。 | **默认 1-RTT 完整握手** (ClientHello <-> ServerHello, Finished)。<br>**0-RTT 握手** (通过预共享密钥 PSK)。 |
| **Early Data** | 不支持在握手完成前发送加密应用数据。             | 支持 **0-RTT Early Data**，客户端在发送 `ClientHello` 后可立即发送加密数据。 |
| **加密算法** | 支持大量加密算法，包括许多不安全或弱算法（如 RC4, SHA1, CBC 模式加密套件）。 | **移除了所有弱密码套件和不安全的特性**。<br>强制使用**前向保密性**的密钥交换（如 ECDHE）。<br>所有握手消息都经过身份验证和加密。 |
| **握手加密** | 握手消息大部分是明文。                           | 握手消息中除了 `ClientHello` 和 `ServerHello` 的部分字段外，**几乎所有握手消息都经过加密**。 |
| **数字签名** | 签名算法与密钥算法分离（例如 RSA 密钥可以搭配 MD5/SHA1 签名）。 | 签名算法与密钥算法紧密绑定，并强制使用更强的哈希算法（如 SHA256 及以上）。 |
| **协议降级** | 存在降级攻击的风险（攻击者强制使用旧版本协议）。   | 增加了 **版本协商的抗降级保护机制**，防止降级到 TLS 1.2 或更早版本。 |
| **密码套件** | 密码套件组合方式复杂，容易配置错误，且包含冗余信息。 | 密码套件定义更精简，将密钥交换、身份验证、对称加密和哈希功能整合为少数几种组合。 |
| **扩展协商** | 许多参数的协商（如曲线、签名算法）放在 `ClientHello` 和 `ServerHello` 之后。 | 大部分扩展协商（如支持的椭圆曲线、签名算法）都前置到 `ClientHello` 和 `ServerHello` 阶段。 |
| **握手流程** | 复杂，包含多个可选消息。                           | **大幅简化握手流程**，减少了消息数量，更易于实现和理解。 |
| **会话恢复** | 依赖 Session ID 或 Session Ticket，服务器可能需要管理状态。 | 统一为基于 **PSK (Pre-Shared Key)** 的会话恢复机制，简化了流程。 |
| **不安全/过时** | RC4 流密码<br>SHA-1 哈希函数<br>任意 DH 参数<br>CBC 模式的密码套件<br>一些不安全的椭圆曲线<br>压缩机制<br>ChangeCipherSpec (作为独立记录)<br>重新协商 (Renegotiation) | **全部移除**，仅保留强密码套件和安全特性。               |



### 证书优化

- **传输优化**
    - 要让证书更便于传输，那必然是减少证书的大小，这样可以节约带宽，也能减少客户端的运算量。所以，**对于服务器的证书应该选择椭圆曲线（ECDSA）证书，而不是 RSA 证书，因为在相同安全强度下， ECC 密钥长度比 RSA 短的多。**

- **验证优化**
    - `OCSP stapling`：现在基本都是使用 OCSP ，名为在线证书状态协议（Online Certificate Status Protocol）来查询证书的有效性，它的工作方式是向 CA 发送查询请求，让 CA 返回证书的有效状态。

> 不过值得注意的是，现在仅 `iOS` 还再走 OCSP stapling 这一套，Android 和 Chrome 浏览器，都不再关心 OCSP stapling
> 参考：https://blog.wirelessmoves.com/2015/03/ocsp-stapling-and-android-that-doesnt-care.html