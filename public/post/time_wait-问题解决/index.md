# TIME_WAIT 问题解决

应用服务器通过发起 TCP 连接其他服务器时，如代理服务器需要请求上游服务器，每个连接会占用一个连接发起方端口，在高并发场景下，可能会导致端口耗尽。同时连接主动关闭方为了TCP正常断开连接，TCP 主动关闭方为了确保最后一个ACK能够达到被动关闭方，所以会等待2MSL。在等待期间该端口就会处于 [TIME_WAIT](https://benohead.com/blog/2013/07/21/tcp-about-fin_wait_2-time_wait-and-close_wait/) 状态。
<!--more-->

**MSL (maximum segment lifetime)**：LINUX 的硬编码字段，名称为 TCP_TIMEWAIT_LEN，值为60s，而一个time_wait 默认等待的时间为 2MSL
## TIME_WAIT 

### 危害

1. 内存资源占用，内存资源的占用不是很严重，可暂且忽略
2. 端口资源占用，一个TCP连接需要消耗一个本地端口，一般可开启的端口为32768 ～ 61000。


### 优化方案
#### net.ipv4.ip_local_port_range
调大主动建联时的端口范围

``` bash
$ sudo sysctl -w net.ipv4.ip_local_port_range="1024 65000"
```
or
``` bash
sudo vim /etc/sysctl.conf

# increase system IP port limits
net.ipv4.ip_local_port_range = 1024 65000
```
More info: [ip_local_port_range](https://mozillazg.com/2019/05/linux-what-net.ipv4.ip_local_port_range-effect-or-mean.html)

 但是这种方案治标不治本

#### net.ipv4.tcp_max_tw_buckets
此值默认是 18000，当系统中处于 TIME_WAIT 的连接大于该值后，系统会将所有的 TIME_WAIT 连接重置，并打印出警告信息。这个方法过于暴力，解决的问题比带来的问题多，不建议使用

``` bash
$ sudo sysctl -w net.ipv4.tcp_max_tw_buckets = 55000
```

More info: [tcp_max_tw_buckets](https://sysctl-explorer.net/net/ipv4/tcp_max_tw_buckets/)

#### 调低 TCP_TIMEWAIT_LEN，重新编译系统
得编译内核，而且TCP发明至今这些固化到内核的参数都是有一定道理的，不要乱改。

#### SO_LINGER 设置
在应用程序中设置套接字选项，调用close 或者 shutdown 关闭连接时候的行为。是用来设置 **延迟关闭** 的选项。

等待套接字发送缓冲区中的数据发送完成。没有设置该选项时，在调用close()后，在发送完FIN后会立即进行一些清理工作并返回。如果设置了SO_LINGER选项，并且等待时间为正值，则在清理之前会等待一段时间。

SO_LINGER 的一个作用就是用来减少TIME_WAIT套接字的数量。在设置SO_LINGER选项时，指定等待时间为0，此时调用主动关闭时不会发送FIN来结束连接，而是直接将连接设置为CLOSE状态，清除套接字中的发送和接收缓冲区，直接对对端发送RST包。
``` c
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);

struct linger {
　int　 l_onoff;　　　　/* 0=off, nonzero=on */
　int　 l_linger;　　　　/* linger time, POSIX specifies units as seconds */
}
```
- onoff: linger 的开关
 - l_onoff 为 0：关闭linger 选项，默认行为，close 或 shutdown 立即返回，如果在套接字发送缓冲区有数据残留，系统会将试着把这些数据都发送出去
 - l_onoff 为 1：打开linger 选项，具体行为看 l_linger
	- l_linger 为0：调用close后，立即发送一个RST标志给对端，该TCP跳过四次挥手，直接关闭，这种方式被称为“强行关闭”，这种情况下，排队的数据不会被发送，被动关闭方也不知道对端已经彻底断开，只有当被动关闭方正阻塞在recv() 调用上，接受到 RST 时，会立刻得到一个 “connect reset by peer”的异常。
    - l_linger 为1：调用close后，调用close的线程将阻塞，直到数据都被发送出去，或者设置 l_linger 的计时时间到。
    
SO_LINGER选项的作用是等待发送缓冲区中的数据发送完成，但是并不保证发送缓冲区中的数据一定被对端接收（对端宕机或线路问题），只是说会等待一段时间让这个过程完成。如果在等待的这段时间里接收到了带数据的包，还是会给对端发送RST包，并且会reset掉套接字，因为此时已经关闭了接收通道。

More info [SO_LINGER](https://www.cnblogs.com/javawebsoa/p/3201324.html)
    
#### net.ipv4.tcp_tw_reuse : 更安全的设置
从协议角度理解如果是安全的话，可以复用处于 TIME_WAIT 的套接字为新的连接所用。

这里的协议角度 的安全是指：
只适用与连接的发起方，即客户端
对应的TIME_WAIT 状态的连接创建时间超过1s才可以被复用。

使用的这个选项的前提，需要打开对TCP时间戳的支持。
即 net.ipv4.tcp_timestamps = 1 （默认即为1），重复的数据包会因为时间戳过期被自然丢弃。

More info [tcp_tw_reuse](https://sysctl-explorer.net/net/ipv4/tcp_tw_reuse/)


 **同时这个也是最推荐的设置**

#### SO_REUSEADDR
这个比较特殊，网上有很多教程都说拿这个解决 TIME_WAIT，其实是对的，但是不是一回事。为什么？

前面解决的问题都是客户端角度没有新端口去建联了，而这个是服务端挂了重启服务后，监听的端口处于TIME_WAIT 的解决方案

这个是解决端口复用问题的，并不是解决 TIME_WAIT ，这个是告诉内核，即使TIME_WAIT 的套接字，也可以作为新的套接字使用，这是为了避免服务端监听端口时，因为被监听的端口处于 TIME_WAIT 导致服务端无法启动。
其本质是解决 服务端 监听端口时的 TIME_WAIT ，而我们上面一直说的是作为客户端建联时没有足够的随机端口导致的无法建联。

More info [SO_REUSEADDR](https://www.jianshu.com/p/141aa1c41f15)

#### 终极解决方案： 长连接
既然代理服务器需要与后端上游服务器通信，最好保持好长连接，连接复用。不然反复的新建连接，握手也是一种消耗。

但是仅限内网这么搞，公网的话，一条连接总是不断，运营商可能会搞些小动作，给你限个速。

