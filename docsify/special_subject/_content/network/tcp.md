# TCP

## 什么是 TCP
TCP 是面向连接的、可靠的、基于字节流的传输层通信协议。

- **面相链接**：一定是「一对一」才能连接，不能像 UDP 协议可以一个主机同时向多个主机发送消息，也就是一对多是无法做到的；
- **可靠的**：无论的网络链路中出现了怎样的链路变化，TCP 都可以保证一个报文一定能够到达接收端；
- **基于字节流**: 用户消息通过 TCP 协议传输时，消息可能会被操作系统「分组」成多个的 TCP 报文，如果接收方的程序如果不知道「消息的边界」，是无法读出一个有效的用户消息的。并且 TCP 报文是「有序的」，当「前一个」TCP 报文没有收到的时候，即使它先收到了后面的 TCP 报文，那么也不能扔给应用层去处理，同时对「重复」的 TCP 报文会自动丢弃。

## TCP 三次握手

<style>
/* CSS for the TCP 4-Way Handshake Diagram */
.tcp-handshake-diagram {
    position: relative;
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen, Ubuntu, Cantarell, "Fira Sans", "Droid Sans", "Helvetica Neue", Arial, sans-serif;
    font-size: 14px;
    color: #333;
    padding: 20px;
    background-color: #fff;
    border-radius: 12px;
    max-width: 720px; /* Consistent width */
    margin: 2em auto;
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15);
    overflow: hidden;
    min-height: 580px; /* Increased height for more states */
}

.tcp-handshake-column {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 180px;
    margin: 0;
    height: 100%;
    justify-content: flex-start;
}

.tcp-handshake-column-title {
    font-weight: bold;
    font-size: 1.2em;
    margin-bottom: 20px;
    color: #2c3e50;
    text-align: center;
}

.tcp-state-box {
    width: 100%;
    padding: 10px 5px;
    margin-bottom: 10px;
    border: 1px solid #ccc;
    text-align: center;
    font-weight: bold;
    box-sizing: border-box;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
    display: flex;
    justify-content: center;
    align-items: center;
}

/* SVG Overlay for Lines */
.tcp-svg-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 0;
}

.tcp-svg-line {
    stroke: #42b983; /* Green line color */
    stroke-width: 2.5;
    fill: none;
    pointer-events: none;
}

/* Line Labels (HTML elements positioned over SVG lines) */
.tcp-line-label-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 1;
}

.tcp-line-text {
    position: absolute;
    background-color: #fff;
    padding: 3px 8px;
    border-radius: 5px;
    font-size: 0.9em;
    font-weight: 500;
    white-space: nowrap;
    z-index: 2;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    pointer-events: auto;
    transform: translate(-50%, -50%);
}

.tcp-state-close { background-color: #e0e0e0; height: 60px; }
.tcp-state-listen { background-color: #cce7ff; height: 60px; }
.tcp-state-syn-sent { background-color: #ffe6b3; height: 120px; }
.tcp-state-syn-rcvd { background-color: #ffe6b3; height: 120px; }
.tcp-state-established { background-color: #d6f7d6; height: 180px; }

/* Line label specific positions (calculated for alignment) */
.syn-text { top: 90px; left: calc(50% - 20px); }
.synack-text { top: 225px; left: calc(50% - 20px); } /* Keep position relative to new line path */
.ack-text { top: 355px; left: calc(50% - 20px); }

/* Specific state colors and heights */
.tcp-state-established { background-color: #d6f7d6; height: 180px; }
.tcp-state-fin-wait-1 { background-color: #ffe6b3; height: 60px; } /* Orange-like */
.tcp-state-fin-wait-2 { background-color: #ffe6b3; height: 60px; } /* Orange-like */
.tcp-state-time-wait { background-color: #cce7ff; height: 120px; } /* Blue-like */
.tcp-state-close { background-color: #e0e0e0; height: 60px; } /* Grey */

.tcp-state-close-wait { background-color: #ffe6b3; height: 60px; } /* Orange-like */
.tcp-state-last-ack { background-color: #ffe6b3; height: 120px; } /* Orange-like */

/* Line label specific positions (calculated for alignment) */
.fin1-text { top: 200px; left: calc(50% - 20px); }
.ack1-text { top: 280px; left: calc(50% - 20px); }
.fin2-text { top: 360px; left: calc(50% - 20px); }
.ack2-text { top: 460px; left: calc(50% - 20px); }

</style>

<div class="tcp-handshake-diagram">
    <div class="tcp-handshake-column">
        <div class="tcp-handshake-column-title">客户端</div>
        <div id="client-close" class="tcp-state-box tcp-state-close">CLOSE</div>
        <div id="client-syn-sent" class="tcp-state-box tcp-state-syn-sent">SYN_SENT</div>
        <div id="client-established" class="tcp-state-box tcp-state-established">ESTABLISHED</div>
    </div>
    <div class="tcp-handshake-column">
        <div class="tcp-handshake-column-title">服务端</div>
        <div id="server-close" class="tcp-state-box tcp-state-close">CLOSE</div>
        <div id="server-listen" class="tcp-state-box tcp-state-listen">LISTEN</div>
        <div id="server-syn-rcvd" class="tcp-state-box tcp-state-syn-rcvd">SYN_RCVD</div>
        <div id="server-established" class="tcp-state-box tcp-state-established">ESTABLISHED</div>
    </div>
    <svg class="tcp-svg-overlay" viewBox="0 0 720 480">
        <defs>
            <marker id="arrowhead-right" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
                <polygon points="0 0, 10 3.5, 0 7" fill="#42b983" />
            </marker>
            <marker id="arrowhead-left" markerWidth="10" markerHeight="7" refX="0" refY="3.5" orient="auto">
                <polygon points="10 0, 0 3.5, 10 7" fill="#42b983" />
            </marker>
        </defs>
        <line class="tcp-svg-line" x1="210" y1="80" x2="510" y2="100" marker-end="url(#arrowhead-right)" />
        <line class="tcp-svg-line" x1="210" y1="200" x2="510" y2="150" marker-start="url(#arrowhead-left)" />
        <line class="tcp-svg-line" x1="210" y1="220" x2="510" y2="280" marker-end="url(#arrowhead-right)" />
    </svg>
    <div class="tcp-line-label-container">
        <div class="tcp-line-text syn-text">SYN<br>Seq Num = client_isn</div>
        <div class="tcp-line-text synack-text">SYN + ACK<br>Ack Num = client_isn + 1<br>Seq Num = server_isn</div>
        <div class="tcp-line-text ack-text">ACK<br>Ack Num = server_isn + 1</div>
    </div>
</div>

- 一开始，客户端和服务端都处于 `CLOSE` 状态。先是服务端主动监听某个端口，处于 `LISTEN` 状态
- 客户端会随机初始化序号（`client_isn`），将此序号置于 TCP 首部的「序号」字段中，同时把 `SYN`  标志位置为 1, 表示 `SYN` 报文。接着把第一个 SYN 报文发送给服务端，表示向服务端发起连接，该报文不包含应用层数据，之后客户端处于 `SYN_SENT` 状态
- 服务端收到客户端的 `SYN`, 报文后，首先服务端也随机初始化自己的序号（`server_isn`），将此序号填入 TCP 首部的「序号」字段中，其次把 TCP 首部的「确认应答号」字段填入 `client_isn + 1`, 接着把 `SYN` 和 `ACK`  标志位置为 1。最后把该报文发给客户端，该报文也不包含应用层数据，之后服务端处于 `SYN_RECV` 状态。

## TCP 四次挥手

<div class="tcp-handshake-diagram">
    <div class="tcp-handshake-column">
        <div class="tcp-handshake-column-title">客户端</div>
        <div class="tcp-state-box tcp-state-established">ESTABLISHED</div>
        <div class="tcp-state-box tcp-state-fin-wait-1">FIN_WAIT_1</div>
        <div class="tcp-state-box tcp-state-fin-wait-2">FIN_WAIT_2</div>
        <div class="tcp-state-box tcp-state-time-wait">TIME_WAIT<br/>(2 MSL)</div>
        <div class="tcp-state-box tcp-state-close">CLOSE</div>
    </div>
    <div class="tcp-handshake-column">
        <div class="tcp-handshake-column-title">服务端</div>
        <div class="tcp-state-box tcp-state-established">ESTABLISHED</div>
        <div class="tcp-state-box tcp-state-close-wait">CLOSE_WAIT</div>
        <div class="tcp-state-box tcp-state-last-ack">LAST_ACK</div>
        <div class="tcp-state-box tcp-state-close">CLOSE</div>
    </div>
    <svg class="tcp-svg-overlay" viewBox="0 0 720 580"> <defs>
            <marker id="arrowhead-right" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
                <polygon points="0 0, 10 3.5, 0 7" fill="#42b983" />
            </marker>
            <marker id="arrowhead-left" markerWidth="10" markerHeight="7" refX="0" refY="3.5" orient="auto">
                <polygon points="10 0, 0 3.5, 10 7" fill="#42b983" />
            </marker>
        </defs>
        <line class="tcp-svg-line" x1="210" y1="220" x2="510" y2="230" marker-end="url(#arrowhead-right)" />
        <line class="tcp-svg-line" x1="210" y1="310" x2="510" y2="240" marker-start="url(#arrowhead-left)" />
        <line class="tcp-svg-line" x1="210" y1="370" x2="510" y2="300" marker-start="url(#arrowhead-left)" />
        <line class="tcp-svg-line" x1="210" y1="380" x2="510" y2="440" marker-end="url(#arrowhead-right)" />
    </svg>
    <div class="tcp-line-label-container">
        <div class="tcp-line-text fin1-text">FIN<br>Seq Num = u</div>
        <div class="tcp-line-text ack1-text">ACK<br>Ack Num = u + 1</div>
        <div class="tcp-line-text fin2-text">FIN<br>Seq Num = w</div>
        <div class="tcp-line-text ack2-text">ACK<br>Ack Num = w + 1</div>
    </div>
</div>

- 客户端打算关闭连接，此时会发送一个 `TCP` 首部 `FIN` 标志位被置为 1 的报文，也即 `FIN` 报文，之后客户端进入 `FIN_WAIT_1` 状态。
- 服务端收到该报文后，就向客户端发送 `ACK` 应答报文，接着服务端进入 `CLOSE_WAIT` 状态。
- 客户端收到服务端的 `ACK` 应答报文后，之后进入 `FIN_WAIT_2` 状态。
- 等待服务端处理完数据后，也向客户端发送 `FIN` 报文，之后服务端进入 `LAST_ACK` 状态。
- 客户端收到服务端的 `FIN` 报文后，回一个 `ACK` 应答报文，之后进入 `TIME_WAIT` 状态
- 服务端收到了 `ACK` 应答报文后，就进入了 `CLOSE` 状态，至此服务端已经完成连接的关闭。
- 客户端在经过 `2MSL` 一段时间后，自动进入 `CLOSE` 状态，至此客户端也完成连接的关闭。

## TCP 握手的问题

### 为什么是三次握手？不是两次、四次？
**为什么不是两次握手？**
如果只有两次握手，即：

1. **客户端 -> 服务端**：SYN (客户端：我想连接，我的初始序列号是 X)
2. **服务端 -> 客户端**：SYN + ACK (服务端：好的，我同意连接，你的序列号 X 我收到了，我的初始序列号是 Y)

**存在的问题**：

- **无法确认客户端的初始序列号 (ISN) 是否已成功传达给服务端并得到服务端的确认**。
    - 设想一种情况：客户端发送了一个 SYN 包，但这个包因为网络延迟或重复发送等原因滞留在网络中。
    - 客户端超时后重新发送了一个 SYN 包，并成功与服务端建立了连接，完成数据传输后释放了连接。
    - 此时，那个滞留的旧 SYN 包（称为“历史连接请求”）终于到达了服务端。服务端收到后，以为是新的连接请求，便回复一个 SYN+ACK。
    - 如果只有两次握手，服务端在发送 SYN+ACK 后就会认为连接建立成功，并开始等待客户端发送数据。但此时客户端已经关闭了连接，根本不会理会这个旧的 SYN+ACK。
    - 结果是，服务端会一直处于等待状态，浪费资源，甚至可能向一个不存在的连接发送数据，造成数据混乱。
- **无法有效地防止重复连接**： 两次握手无法区分“旧的重复连接请求”和“新的连接请求”，可能导致重复连接的建立，从而引发数据错乱。
- **无法进行双向的序列号同步**： 序列号是 TCP 保证可靠传输的基础。客户端需要知道服务端的 ISN，服务端也需要知道客户端的 ISN。两次握手只能保证服务端收到客户端的 ISN 并发送自己的 ISN，但客户端无法确认服务端是否真的收到了它的 ISN。

**为什么不是四次握手？**
四次握手意味着：

- **客户端 -> 服务端**：SYN (客户端：我想连接，我的 ISN 是 X)
- **服务端 -> 客户端**：ACK (服务端：收到你的 SYN 和 ISN X)
- **服务端 -> 客户端**：SYN (服务端：我也想连接，我的 ISN 是 Y)
- **客户端 -> 服务端**：ACK (客户端：收到你的 SYN 和 ISN Y)

**存在的问题**：

- **效率低下**： 两次握手（第二次和第三次）本可以合并为一次。在三次握手中，服务端在确认收到客户端的 SYN 的同时，也会将自己的 SYN 发送给客户端，实现了同时确认和同步。
- **资源浪费**： 每次握手都需要消耗网络资源（带宽、路由器处理能力）和主机资源（CPU、内存）。在确保可靠性的前提下，减少握手次数可以提高效率。

### 既然 IP 层会分片，为什么 TCP 层还需要 MSS 呢？

1. 当如果一个 IP 分片丢失，整个 IP 报文的所有分片都得重传
2. 分片后存在中间数据包上没有 TCP 头部信息，遇到一些负载均衡的网关时可能会出现奇怪的丢包问题
3. 经过 TCP 层分片后，如果一个 TCP 分片丢失后，进行重发时也是以 MSS 为单位，而不用重传所有的分片，大大增加了重传的效率。

### 什么是 SYN 攻击？如何避免 SYN 攻击？

都知道 TCP 连接建立是需要三次握手，假设攻击者短时间伪造不同 IP 地址的 SYN 报文，服务端每接收到一个 SYN 报文，就进入SYN_RCVD 状态，但服务端发送出去的 ACK + SYN 报文，无法得到未知 IP 主机的 ACK 应答，久而久之就会占满服务端的半连接队列，使得服务端不能为正常用户服务。

SYN 攻击方式最直接的表现就会把 TCP 半连接队列打满，这样当 TCP 半连接队列满了，后续再在收到 SYN 报文就会丢弃，导致客户端无法和服务端建立连接。

避免 SYN 攻击方式，可以有以下四种方法：

- 调大 netdev_max_backlog；
- 增大 TCP 半连接队列；
- 开启 tcp_syncookies；
- 减少 SYN+ACK 重传次数

## TCP 挥手的问题

### 为什么需要 TIME_WAIT 状态？
主动发起关闭连接的一方，才会有 TIME-WAIT 状态。

需要 TIME-WAIT 状态，主要是两个原因：

- 防止历史连接中的数据，被后面相同四元组的连接错误的接收；
- 保证「被动关闭连接」的一方，能被正确的关闭；

### TIME_WAIT 过多有什么危害？
过多的 TIME-WAIT 状态主要的危害有两种：

- 第一是占用系统资源，比如文件描述符、内存资源、CPU 资源、线程资源等；
- 第二是占用端口资源，端口资源也是有限的，一般可以开启的端口为 32768～61000，也可以通过 `net.ipv4.ip_local_port_range` 参数指定范围。

### 如何优化 TIME_WAIT？

####  **方法一：net.ipv4.tcp_tw_reuse 和 tcp_timestamps**

客户端（主动关闭连接方）都是和「目的 IP+ 目的 PORT 」都一样的服务器建立连接的话，当客户端的 TIME_WAIT 状态连接过多的话，就会受端口资源限制，如果占满了所有端口资源，那么就无法再跟「目的 IP+ 目的 PORT」都一样的服务器建立连接了。

不过，即使是在这种场景下，只要连接的是不同的服务器，端口是可以重复使用的，所以客户端还是可以向其他服务器发起连接的，这是因为内核在定位一个连接的时候，是通过四元组（源IP、源端口、目的IP、目的端口）信息来定位的，并不会因为客户端的端口一样，而导致连接冲突。

好在 Linux 操作系统提供了两个可以系统参数来快速回收处于 `TIME_WAIT` 状态的连接，这两个参数都是默认关闭的：

1. `net.ipv4.tcp_tw_reuse`，如果开启该选项的话，客户端（连接发起方） 在调用 `connect()` 函数时，如果内核选择到的端口，已经被相同四元组的连接占用的时候，就会判断该连接是否处于 `TIME_WAIT` 状态，如果该连接处于 `TIME_WAIT` 状态并且 `TIME_WAIT` 状态持续的时间超过了 1 秒，那么就会重用这个连接，然后就可以正常使用该端口了。所以该选项只适用于连接发起方。
2. `net.ipv4.tcp_tw_recycle`，如果开启该选项的话，允许处于 `TIME_WAIT` 状态的连接被快速回收，该参数在 `NAT` 的网络下是不安全的！ 

> - 与 `NAT (Network Address Translation)` 设备冲突导致连接中断。<br/>
> 这是最主要也是最常见的问题。当多个客户端通过同一个 NAT 设备访问服务器时，对于服务器来说，这些客户端都表现为同一个源 IP 地址。
`tcp_tw_recycle` 的工作原理是，它会根据每个源 IP 地址记录上次接收到的 TCP 时间戳。当一个新连接请求到达时，如果时间戳比上次记录的旧，系统会认为这是一个旧的、重复的包，并将其丢弃。
然而，对于通过 NAT 设备的不同客户端，即使它们的实际时间戳是递增的，由于 NAT 将它们映射到同一个公共 IP，服务器会收到来自同一个 IP 地址但时间戳不单调递增的连接请求。这就导致服务器错误地丢弃了这些“看似旧”的连接请求，从而导致客户端连接失败或间歇性中断。


#### 方式二：net.ipv4.tcp_max_tw_buckets

这个值默认为 18000，当系统中处于 `TIME_WAIT` 的连接一旦超过这个值时，系统就会将后面的 `TIME_WAIT` 连接状态重置，这个方法比较暴力。

#### 方式三：程序中使用 SO_LINGER

我们可以通过设置 `socket` 选项，来设置调用 `close` 关闭连接行为。
```c
struct linger so_linger;
so_linger.l_onoff = 1;
so_linger.l_linger = 0;
setsockopt(s, SOL_SOCKET, SO_LINGER, &so_linger,sizeof(so_linger));
```
如果`l_onoff`为非 0， 且`l_linger`值为 0，那么调用`close`后，会立该发送一个RST标志给对端，该 `TCP` 连接将跳过四次挥手，也就跳过了`TIME_WAIT`状态，直接关闭。

### 服务器出现大量 TIME_WAIT 状态的原因有哪些？

`TIME_WAIT` 状态是主动关闭连接方才会出现的状态，所以如果服务器出现大量的 TIME_WAIT 状态的 TCP 连接，就是说明服务器主动断开了很多 TCP 连接。

问题来了，什么场景下服务端会主动断开连接呢？

- 第一个场景：HTTP 没有使用长连接
- 第二个场景：HTTP 长连接超时
- 第三个场景：HTTP 长连接的请求数量达到上限

### 服务器出现大量 CLOSE_WAIT 状态的原因有哪些？

`CLOSE_WAIT` 状态是「被动关闭方」才会有的状态，而且如果「被动关闭方」没有调用 close 函数关闭连接，那么就无法发出 FIN 报文，从而无法使得 `CLOSE_WAIT` 状态的连接转变为 `LAST_ACK` 状态。

所以，当服务端出现大量 `CLOSE_WAIT` 状态的连接的时候，说明服务端的程序没有调用 close 函数关闭连接。

当服务端出现大量 CLOSE_WAIT 状态的连接的时候，通常都是代码的问题，这时候我们需要针对具体的代码一步一步的进行排查和定位，主要分析的方向就是服务端为什么没有调用 close。




