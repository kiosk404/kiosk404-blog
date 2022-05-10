---
title:  使用eBPF跟踪 SSL/TLS 连接
author: kiosk
tags:

  - eBPF
categories:
  - Linux
date: 2022-05-05 16:30:00
---



前面的文章介绍了 eBPF 的入门知识，这篇算是一个实战吧，使用 eBPF 来跟踪 TLS加密连接。[TLS](https://kiosk007.top/post/tls%E8%AF%A6%E8%A7%A3-%E4%B8%80/) 是目前互联网上的一个标准工具了，可用于加密的通信。本文尝试使用 eBPF 技术将加密的密文还原为明文。当然这并不意味着TLS不再安全，想干坏事的人可以在中间代理服务器、网关设备上偷听。因为eBPF 所实现的追踪明文一定是要建立在真正去做ssl 明文转密文写入操作的。

<br>

关于eBPF基础知识请查看

- [认识eBPF](https://kiosk007.top/post/%E8%AE%A4%E8%AF%86ebpf/)



# 使用eBPF进行跟踪连接

BPF 可用于对内核函数、用户态函数进行插桩进而实现观测的需求。当然一个有趣的应用就是跟踪网络流量。



<img src="https://img1.kiosk007.top/static/images/blog/bpf-tls-tracing.svg" alt="bpf-tls-tracing" style="zoom:50%;" />