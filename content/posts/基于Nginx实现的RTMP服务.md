---
title: 基于Nginx的实时音频直播服务
author: kiosk
tags:
  - 点播
categories:
  - Nginx
date: 2020-11-16 23:02:00
---

最近在百度云上下了一些学习视频（别想歪..），苦于一台笔记本不方便同时跟着操作变播放，那么既然都有笔记本了，何不搭建一个视频播放服务呢？笔记本使用Nginx搭建一个视频服务器，iPad上使用播放器（Aplayer），不仅是播放还可以直播，这不正契合了当前的高热话题“实时音视频直播技术”么，最近也找了一部分资料，趁着直播的热度在这里总结一下。

<!--more-->

  随着互联网用户消费内容和交互方式的升级，支撑这些内容和交互方式的基础设施也正在悄悄发生变革。手机设备拍摄视频能力和网络的升级催生了大家对视频直播领域的关注，吸引了很多互联网创业者或者成熟企业进入该领域。

# 直播中的各个环节

## 完整的直播流程
![](https://img1.kiosk007.top/static/images/live/live.jpg)

- <font color='red'>`音视频采集`</font>: 采集是播放环节中的第一环，iOS 系统因为软硬件种类不多，硬件适配性较好，所以比较简单。Android 则不同，市面上硬件机型非常多，难以做到一个库适配所有硬件。PC 端的采集也跟各种摄像头驱动有关，推荐使用目前市面上最好用的 PC 端开源免费软件 OBS。
- `音视频处理`: 美颜、水印等也都是在这个环节做。目前 iOS 端比较知名的是 GPUImage 这个库，提供了丰富端预处理效果，还可以基于这个库自己写算法实现更丰富端效果。Android 也有 GPUImage 这个库的移植，叫做 android-gpuimage。
- `音视频编码`: iOS 端硬件兼容性较好，可以直接采用硬编。而 Android 的硬编的支持则难得多，需要支持各种硬件机型，推荐使用软编。
- `推流和传输`: 涉及 1、从主播端到服务端 2、从收流服务端到边缘节点 3、从边缘节点到观众端 。推流端和分发端理论上需要支持的并发用户数应该都是亿级的，不过毕竟产生内容的推流端在少数，和消费内容端播放端不是一个量级。对推流稳定性和速度的要求比播放端高很多，这涉及到所有播放端能否看到直播，以及直播端质量如何。
- `实时音视频转码`: 为了让主播推上来的流适配各个平台端各种不同协议，需要在服务端做一些流处理工作，比如转码成不同格式支持不同协议如 RTMP、HLS 和 FLV，一路转多路流来适配各种不同的网络状况和不同分辨率的终端设备。
- `解码和渲染`: 解码和渲染，也即音视频的播放，目前 iOS 端的播放兼容性较好，在延迟可接受的情况下使用 HLS 协议是最好的选择，市面上也提供了能够播放 RTMP 和 HLS 的播放器 SDK。

参考：[《移动端实时音视频直播技术详解》](http://www.52im.net/thread-853-1-1.html)

编码器的介绍可以参考 [编码与封装](http://www.52im.net/thread-965-1-1.html)，下面主要说一说“推流和拉流”

## 推流和拉流

### **推送协议**
推流，指的是把采集阶段封包好的内容传输到服务器的过程。其实就是将现场的视频信号传到网络的过程。

“推流”对网络要求比较高，如果网络不稳定，直播效果就会很差，观众观看直播时就会发生卡顿等现象，观看体验很是糟糕。

要想用于推流还必须把音视频数据使用传输协议进行封装，变成流数据。常用的流传输协议有RTSP、RTMP、WebRTC、HLS等，使用RTMP传输的延时通常在1–3秒，对于手机直播这种实时性要求非常高的场景，RTMP也成为手机直播中最常用的流传输协议。

下面介绍一下以下协议以及他们在直播领域的现状和优缺点：**1、RTMP 2、WebRTC**

#### **RTMP**

RTMP 是 Real Time Messaging Protocol（实时消息传输协议）的首字母缩写。该协议基于 TCP，是一个协议族，包括 RTMP 基本协议及 RTMPT/RTMPS/RTMPE 等多种变种。RTMP 是一种设计用来进行实时数据通信的网络协议，主要用来在 Flash/AIR 平台和支持 RTMP 协议的流媒体/交互服务器之间进行音视频和数据通信。支持该协议的软件包括 Adobe Media Server/Ultrant Media Server/red5 等。
RTMP 是目前主流的流媒体传输协议，广泛用于直播领域，可以说市面上绝大多数的直播产品都采用了这个协议。
另外，RTMPT封装在HTTP请求之上，可穿透防火墙；

**优点:**
> CDN 支持良好，主流的 CDN 厂商都支持；
> 协议简单，在各平台上实现容易。

**缺点:**
> 基于 TCP ，传输成本高，在弱网环境丢包率高的情况下问题显著；
> 不支持浏览器推送；
> Adobe 私有协议，Adobe 已经不再更新。

#### **WebRTC**

WebRTC，名称源自网页即时通信（英语：Web Real-Time Communication）的缩写，是一个支持网页浏览器进行实时语音对话或视频对话的 API。它于 2011 年 6 月 1 日开源并在 Google、Mozilla、Opera 支持下被纳入万维网联盟的 W3C 推荐标准
[WebRTC](https://webrtc.org.cn/) 

**目前主要应用于视频会议和连麦中**

**优点：**
> W3C 标准，主流浏览器支持程度高，Google 在背后支撑，并在各平台有参考实现；
> 底层基于 SRTP 和 UDP，弱网情况优化空间大；
> 可以实现点对点通信，通信双方延时低。

**缺点：**
> ICE、STUN、TURN 传统 CDN 没有类似的服务提供

### **拉流协议**

拉流是指服务器已有直播内容，根据协议类型（如RTMP、RTP、RTSP、HTTP等），与服务器建立连接并接收数据，进行拉取的过程。

拉流端的核心处理在播放器端的解码和渲染，在互动直播中还需集成聊天室、点赞和礼物系统等功能。下面介绍几个常见的拉流协议。

#### **RTMP**

没错，推流和拉流都是RTMP所擅长的。现在 PC 市场巨大，PC 主要是 Windows，Windows 的浏览器基本上都支持 Flash。另外RTMP适合长时间播放，曾经有过测试，联系 100 万秒，即 10 天多连续播放没有出现问题。最后 RTMP 的延迟相对较低，一般延时在 1-3s 之间，一般的视频会议，互动式直播，完全是够用的。

不过，在移动端上，Flash Player 已经被杀绝了，那为啥还会出现这个呢？
因为它主要是针对 PC 端的。现在推流协议各大云厂商基本都是直接支持 rtmp 。

**拉流用 rtmp 的话就不太现实了，现在对 flash 支持都不友好了。**

#### **HLS**

HLS 是苹果提出的基于HTTP的流媒体传输协议，优点是跨平台性比较好，HTML5可以直接打开播放，移动端兼容性良好，但是缺点是延迟比较高。

HLS 主要的两块内容是 .m3u8 文件和 .ts 播放文件。接受服务器会将接受到的视频流进行缓存，然后缓存到一定程度后，会将这些视频流进行编码格式化，同时会生成一份 .m3u8 文件和其它很多的 .ts 文件。

HLS 支持的功能，并不只是分片播放（专门适用于直播），它还包括其他应有的功能。

> 使用 HTTPS 加密 ts 文件
> 快/倒放
> 广告插入
> 不同分辨率视频切换

**HLS 之所以能这么流行，关键在于它的支持度是真的广，所以，对于一般 H5 直播来说，应该是非常友好的。**

#### **HDL(HTTP-FLV)**

HTTP-FLV 和 RTMPT 类似，都是针对于 FLV 视频格式做的直播分发流。
但，两者有着很大的区别：

**相同点：**
> 两者都是针对 FLV 格式
> 两者延时都很低
> 两者都走的 HTTP 通道

**不同点：**
> HTTP-FLV直接发起长连接，下载对应的 FLV 文件
> 头部信息简单

RTMPT握手协议过于复杂。分包，组包过程耗费精力大，因为 RTMP 发的包很容易处理，通常 RTMP 协议会作为视频上传端来处理，然后经由服务器转换为 FLV 文件，通过 HTTP-FLV 下发给用户。


参考：[直播协议RTMP、HLS、HTTP FLV](https://driverzhang.github.io/post/%E7%9B%B4%E6%92%AD%E5%8D%8F%E8%AE%AErtmphlshttp-flv/)


# 基于Nginx搭建视频服务

> 系统版本: Ubuntu 20.04 focal
> Nginx版本:1.17.3
> Nginx Rtmp模块: nginx-rtmp-module

## **Nginx**

我是基于 XPS 上的Ubuntu搭建的，搭建过程之前在 [自建 Nginx 部署
](https://kiosk007.top/2020/04/18/%E8%87%AA%E5%BB%BA-Nginx-%E7%AB%99%E7%82%B9%E9%83%A8%E7%BD%B2/) 文章中已经提过，这里需要注意的是需要安装 `nginx-rtmp-module` 模块。

``` bash
# nginx -m
KServer version: KServer/2.3.2
nginx version: nginx/1.17.3
nginx: loaded modules:
nginx:     ngx_core_module (static)
...
nginx:     ngx_rtmp_module (static)
nginx:     ngx_rtmp_core_module (static)
nginx:     ngx_rtmp_cmd_module (static)
nginx:     ngx_rtmp_codec_module (static)
nginx:     ngx_rtmp_access_module (static)
nginx:     ngx_rtmp_record_module (static)
nginx:     ngx_rtmp_live_module (static)
nginx:     ngx_rtmp_play_module (static)
nginx:     ngx_rtmp_flv_module (static)
nginx:     ngx_rtmp_mp4_module (static)
nginx:     ngx_rtmp_netcall_module (static)
nginx:     ngx_rtmp_relay_module (static)
nginx:     ngx_rtmp_exec_module (static)
nginx:     ngx_rtmp_auto_push_module (static)
nginx:     ngx_rtmp_auto_push_index_module (static)
nginx:     ngx_rtmp_notify_module (static)
nginx:     ngx_rtmp_log_module (static)
nginx:     ngx_rtmp_limit_module (static)
nginx:     ngx_rtmp_hls_module (static)
nginx:     ngx_rtmp_dash_module (static)
nginx:     ngx_openssl_module (static)
...

```

## 视频直播服务

Nginx本身是一个非常出色的HTTP服务器,FFMPEG是非常好的音视频解决方案.这两个东西通过一个Nginx的模块nginx-rtmp-module,组合在一起即可以搭建一个功能相对比较完善的流媒体服务器. 这个流媒体服务器可以支持RTMP和HLS(Live Http Stream)。

配置时需要在Nginx的主配置加上一句`include /home/work/nginx/rtmp/*.conf;`，因为 rtmp 是单独一个块，等价于 http 块。再创建目录 `/home/work/nginx/rtmp/`，之后 rtmp 的配置会放在该目录下。



### **安装FFmpeg**

FFmpeg 是视频处理最常用的开源软件。名称来自MPEG视频编码标准，前面的“FF”代表“Fast Forward”，FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。可以轻易地实现多种视频格式之间的相互转换，FFmpeg的用户有Google，Facebook，Youtube，优酷，爱奇艺，土豆等。可参考阮一峰老师的 [FFmpeg 视频处理](http://www.ruanyifeng.com/blog/2020/01/ffmpeg.html) 这篇文章。



安装方式：
``` bash
sudo add-apt-repository ppa:kirillshkrogalev/ffmpeg-next
sudo apt-get update
sudo apt-get install ffmpeg
```

### RTMP 直播

在 `/home/work/nginx/rtmp/live.conf` 里创建如下内容。

``` nginx
rtmp_auto_push on;
rtmp {
    server {
        listen 1935;
        notify_method get;
        access_log  /home/work/log/nginx/http_rtmp.log;
        chunk_size 4096;
        application rtmplive {
            live on;                # 开启直播模式
            allow publish all;      # 允许从任何源push流
            allow play all;         # 允许从任何地方来播放流
            drop_idle_publisher 20s; # 20秒内没有push，就断开链接
        }
    }
}
```

检查nginx配置后reload nginx,使之生效。

``` bash
# /home/work/nginx/sbin/nginx -t
# /home/work/nginx/sbin/nginx -s reload
```

**试验**

使用 ffmpeg 推流
```
ffmpeg -re -i ./Golang从入门到癫狂.mp4 -vcodec libx264 -acodec aac -strict -2 -f flv 'rtmp://localhost:1935/live/room'
```

ipad 上去App Store下载 **Aplyer**
<img src="https://img1.kiosk007.top/static/images/live/aplyer.jpeg" style='height:350px' />

打开Aplayer，选择最下方的“网络”，在输入框中输入`rtmp://192.168.1.10/live/room` 即可观看直播了。


### HLS直播

配置hls需要有两个步骤
1. 配置rtmp流产生hls文件
2. 设置nginx来访问hls文件

**配置rtmp流产生hls文件**
创建存放hls文件的目录, 确保 Nginx worker 用户可以读写该目录。

```
mkdir -p /home/work/data/hls
chown work:work /home/work/data/hls

```
修改配置，增加hls相关配置
``` nginx
rtmp_auto_push on;
rtmp {
    server {
        listen 1935; # Listen on standard RTMP port
        notify_method get;
        chunk_size 4096;

        application hlslive {
            live on;   # Turn on HLS
            hls on;
            hls_path /home/work/data/hls;
            
            allow publish all;          # 允许从任何源push流
            allow play all;             # 允许从任何地方来播放流
            drop_idle_publisher 20s;    # 20秒内没有push，就断开链接。
            hls_fragment 3;
            hls_playlist_length 60;
        }
    }
}


```
添加了如下两个配置:

- `hls on` : 开启HLS
- `hls_path /home/work/data/hls` : 设置hls目录

此时使用ffmpeg进行推流后，在/home/work/data/hls目录下，就会有HLS要用到的m3u8和ts文件。

```
# ls
room1-0.ts  room1-1.ts  room1.m3u8
```
设置nginx来访问hls文件
配置nginx通过http访问hls文件, 在server中添加如下location。
``` nginx
server {
    listen 8081;

    location /hls {
        add_header 'Cache-Control' 'no-cache';   # Disable cache

        # CORS setup
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Expose-Headers' 'Content-Length';

        # allow CORS preflight requests
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        types {
            application/dash+xml mpd;
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }

        root /home/work/data;
    }
}

```
然后使用支持 HLS 拉流的播放器输入 `http://192.168.1.8:8081/hls/room1.m3u8`


参考：[Setup Nginx-RTMP on CentOS 7](
https://www.vultr.com/docs/setup-nginx-rtmp-on-centos-7#Security_Note)

## 视频播放服务

毕竟是已经下载好的视频，这里是不存在直播推流的，那么可以直接播放已经存在的视频。视频的格式最好是 mp4 格式，Nginx 配置文件如下即可：
### mp4
``` nginx
server {
   listen 80;
   listen 8088;

   #错误日志和访问日志的路径配置
   access_log  /home/work/log/nginx/http_video.log  jxjson;
   error_log   /home/work/log/nginx/video-error.log notice;
    
    
   #项目的路径 
   root /home/work/data/mp4;

   #所有的mp4文件的自动解析
   location ~ \.mp4$ {
      mp4;
      mp4_buffer_size     1m;
      mp4_max_buffer_size 5m;
   }
}
```

浏览器输入 `192.168.1.8:8088/2-1.mp4`