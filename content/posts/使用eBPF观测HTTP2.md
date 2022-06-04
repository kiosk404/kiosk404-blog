---
title: 使用eBPF观测HTTP2
author: kiosk
draft: true
tags:
  - eBPF
  - golang
categories:
  - Linux
date: 2022-06-04 16:30:00
---







在当今充满微服务的世界中，对服务之间的消息传输的可观察性和理解变得越来越重要。

其中一些微服务的传输协议比较复杂，如HTTP/2 协议为了降低头部内容的重复传输，使用表头压缩算法 HPACK，虽然HPACK有助于提升 HTTP/2 的传输效率，但也使得跟踪变得复杂。不过，通过使用 eBPF uprobes,可以在 Header 被压缩之前对其进行跟踪。



<!--more-->



# 为什么HTTP2 会难以跟踪

我们知道HTTP2是一个取代HTTP1.1的协议，其解决了大量HTTP1.1 所为人诟病的缺点，如大头、管线化等问题。





# eBPF uprobes 如何解决跟踪难题













参考：

- https://blog.px.dev/ebpf-http2-tracing/
- 