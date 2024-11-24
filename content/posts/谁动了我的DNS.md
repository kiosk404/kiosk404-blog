---
title: 网速变慢？你可能需要先设置好 DNS
author: kiosk
tags:
  - web
categories:
  - network
date: 2024-10-24 16:08:00
autoCollapseToc: true
draft: true  
weight: 10
contentCopyright: MIT
mathjax: true
---

有没有觉得 `kiosk007.top` 最近访问变快了？没错，我的主站最近接入了 腾讯云的 [EO(EdegeOne)](https://console.cloud.tencent.com/edgeone)，当然只是14天体验版，体验过后还是会改会直连模式，毕竟是要收费的。不过能使得访问质量得到加速，第一个重要原因就是我们的请求被指向了就近节点，实现了就近访问。其中就依赖一个重要的底层技术 - **DNS**。

<!--more-->

<br/>

# DNS：网络世界的指南针

DNS，英文全称「Domain Name Server」，中文全称「域名服务器」。这种服务器不像网站使用的服务器为用户提供内容，而是将人类易于理解的「域名」转换为机器易于理解的「IP 地址」。

```bash
# dig kiosk007.top +short
kiosk007.top.eo.dnse1.com.
101.72.224.113
119.188.123.202
```

这块的逻辑大家应该有基础网络知识的应该都了解。这里就不多做介绍了。

具体还可以参考: [DNS 原理入门](https://ruanyifeng.com/blog/2016/06/dns.html)

<br/>

# 智能解析



<br/>
