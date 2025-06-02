# 如何减少TTFB以提升WordPress加载性能

当谈论到你的 [WordPress Site](What Is WordPress? Explained for Beginners
) 加载性能时，大多数我们都是在关注前端的性能优化来提升页面加载速度。然而我们有时候应该从服务端视角来看待这个问题。今天我们来讨论一下 **TTFB (time to first byte)** 是怎么影响你的网站加载速度，另外我们如何去降低他。TTFB是一个经常会被忽略的重要性能因素，但是我们在网站性能优化是必须考虑进来。

<img src="http://img1.kiosk007.top/static/images/network/performance/ttfb.png"  style="height:300px">

-- 译 [How to Reduce TTFB to Improve WordPress Page Load Times](https://kinsta.com/blog/ttfb/)

<!--more-->


- [什么是TTFB?]()
- [TTFB 重要吗？]()
- [如何测量TTFB？]()
- [4种方法来减少Wordpress站点的TTFB]()


# 什么是TTFB？
TTFB 表示"首字节时间", 简单的说，表示从浏览器收到服务端的第一个字节所需要的时间。TTFB时间越长、那么页面加载显示的时间就越长。一个常见的误解是，这个时间主要是DNS查找时间计算的。然而，真正的 [TTFB](https://en.wikipedia.org/wiki/Time_To_First_Byte) 其实包含整个网络期间的时延。并且以下三个步骤之和才是总的TTFB。

<img src="https://img1.kiosk007.top/static/images/network/performance/waiting-ttfb.jpg" style="height:300px" >

## 1. 请求服务

当有用户访问你的网站时。发生的第一件事是客户端（浏览器）会先发送一个HTTP请求。在此步骤中，可能有多种因素导致延迟、**缓慢的DNS** 查找时间可能会增加请求的时间。如果服务器在地理上很远，这可能会导致数据传输距离的延迟。此外，如果您有复杂的防火墙规则，这可能会增加路由时间。并且不要忘记客户端的互联网速度。

## 2. 服务端处理耗时

发送请求后，服务器现在必须处理它并生成响应。这可能会引入许多不同的延迟，例如**缓慢的数据库调用**、过多的第三方脚本、**未缓存**第一个响应、糟糕的代码或 WordPress 主题以及低效的服务器资源，例如**磁盘 I/O** 或**内存**。

## 3. 响应

服务器处理完请求后，必须将其发送回客户端（或者更确切地说是发送回第一个字节）。这受到服务器和客户端网络速度的影响。如果客户端的 Wi-Fi 热点速度较慢，这将反映在 TTFB 中。


# TTFB 重要吗？

重要的是要了解 TTFB（到第一个字节的时间）与网站速度不同。这实际上是对响应能力的衡量。网络上有很多关于 TTFB 是否重要的​​讨论。有人说它毫无意义（[Cloudflare](https://blog.cloudflare.com/ttfb-time-to-first-byte-considered-meaningles/)、[LittleBizzy]()），而另一些人说它很重要（[Ilya Grigorik](https://plus.google.com/+IlyaGrigorik/posts/GTWYbYWP6xP)，Google 的 Web 性能工程师）。双方都提出了一些关于为什么或为什么不重要的有效观点，以及关于它如何实际计算的一些问题。


Moz 甚至对[搜索排名与第一个字节的时间之间的相关性](https://moz.com/blog/improving-search-rank-by-optimizing-your-time-to-first-byte)进行了深入研究。 但是，很难知道搜索排名是否是和TTFB相关的 。或者 TTFB 较低的网站是否也只是总体上更快。

然而，与其花时间讨论是否重要，我们宁愿专注于可以做的优化来改进这个指标。在我们具有更大 TTFB 的测试站点中，只需加载并感觉更慢。


通常，**低于 100 毫秒的TTFB 都很棒**。[Google PageSpeed Insights](https://kinsta.com/blog/google-pagespeed-insights/) 建议服务器响应时间小于 200 毫秒。如果您在 300-500 毫秒范围内，这是非常标准的。如果超过 600 毫秒，您的服务或者链路可能有问题，或者可能是时候升级到更好的 Web 服务了。或者按照我们下面有关如何减少 TTFB 的建议进行操作。SSL/TLS 协商也可能是一个点。


# 如何测量TTFB

可以通过多种不同的方式来测试你站点的 TTFB。下面将探索一些。但请记住，每种工具都会给出略有不同的结果，因此重要的是只使用一种工具并坚持使用它作为基线。

## 使用 Google Chrome DevTools 测量 TTFB

可以通过启动DevTools在 Google Chrome 中测量 TTFB 。但请记住，如果您在计算机上测试 TTFB 会受到网络延迟和 Internet 连接的影响。因此，使用从数据中心进行测试的第 3 方工具（如下所示）可能更有效。

- 从 Chrome 菜单中选择更多工具 > 开发者工具。
- 右键单击页面元素并选择检查
- 使用键盘快捷键Ctrl+ Shift+ I(Windows) 或Cmd+ Opt+ I(Mac)
- 您可以启动网络窗口并查看站点的性能。

<img src="https://img1.kiosk007.top/static/images/network/performance/google-chrome-devtools-ttfb.jpg" style="height:300px">



## 使用 Geekflare 的工具测量 TTFB

Geekflare 拥有一系列很棒的免费工具，您可以使用它们来测试和排除网站上的问题。[Geekflare](https://gf.dev/ttfb-test) 的 TTFB 工具简单、快速，可让您查看从全球三个位置获取第一个字节的时间有多快（低）

<img src="https://img1.kiosk007.top/static/images/network/performance/geekflare.png" style="height:300px">


## 使用 WebPageTest 测量 TTFB

您还可以使用 [WebPageTest](https://www.webpagetest.org/) 测量您的 TTFB 。根据他们的术语表，目标时间是 DNS、套接字和 SSL 协商所需的时间 + 100 毫秒。超出目标每100ms扣除一个字母等级。正如您在下面的测试中所见，该站点的 TTFB 为 0.256 秒或 256 毫秒。

<img src="https://img1.kiosk007.top/static/images/network/performance/webpagetest-ttfb.jpg" style="height:300px">


## 用 Pingdom 测量 TTFB

Chrome 和 WebPageTest 将其称为 TTFB。但是，如果您使用Pingdom，它实际上被称为“等待”时间。请务必查看我们关于如何使用 Pingdom的深入指南。

<img src="https://img1.kiosk007.top/static/images/network/performance/wait-time-pingdom.jpg" style="height:300px" >


## 使用 GTmetrix 测量 TTFB

在 GTmetrix 中，就像 Pingdom 一样，TTFB 被称为等待时间。请务必查看我们关于如何使用 [GTmetrix的深入指南](https://kinsta.com/blog/gtmetrix-speed-test/) 。


<img src="https://img1.kiosk007.top/static/images/network/performance/gtmetrix-waiting.png" style="height:300px">



## 使用 KeyCDN 的工具测量 TTFB

KeyCDN 有一个很棒的 [网络性能测试工具](https://tools.keycdn.com/performance) ，您可以在其中同时从 14 个不同的位置测量您的 TTFB。

<img src="https://img1.kiosk007.top/static/images/network/performance/keycdn-ttfb-test.jpg" style="height:300px">

还有一些其他各种工具可以测量 TTFB，例如[Sucuri Performance Tool](https://performance.sucuri.net/)和[ByteCheck](http://www.bytecheck.com/)。

<img src="https://img1.kiosk007.top/static/images/network/performance/google-analytics-ttfb.jpg" style="height:300px">


# 减少 WordPress 网站上 TTFB 的 4 种方法

> 现在让我们深入探讨如何减少 WordPress 网站上的 TTFB。

## 1. 使用快速的 WordPress 主机

减少 TTFB 的第一种方法是确保您使用的是快速的 WordPress 主机。我们比较了第三方云服务器的 TTFB（位于亚利桑那州凤凰城）和 Kinsta 的 TTFB（位于爱荷华州康瑟尔布拉夫斯）。我们使用了完全相同的设置，运行默认的 27 主题。Kinsta 现在拥有所有 25 个可用的Google Cloud Platform 节点，因此战略性地将您的 WordPress 站点放置在离访问者更近的位置非常重要。


切换到更快的主机可以将您网站的 TTFB 减少多达 200%。

Kinsta 还在 所有托管计划中使用了 Google Cloud Platform 的高级网络。许多其他托管服务提供商使用 Google Cloud 的标准层网络，这会导致速度变慢。


在所有地区，平均 TTFB 为 520 毫秒。在美国和加拿大，平均 TTFB 为 240 毫秒。

因此，仅通过使用更快的主机，可以显著降低 TTFB 

## 2. CDN or DSA

另一种减少 TTFB 的简单方法是利用 **[内容交付网络](https://kinsta.com/blog/wordpress-cdn/)** (CDN)。如果您的网站为该国不同地区或全球的访问者提供服务，这会大大降低您的 TTFB。正如我们在上面看到的，位置非常重要。我们进行了一个小测试，以显示 KeyCDN 作为我们的 CDN 提供商的不同之处。每个测试运行 5 次并取平均值。

- 没有 CDN 的 TTFB

我们首先在禁用 CDN 的情况下运行测试，我们的总加载时间为 1.45 秒，资产的平均 TTFB 约为 136 毫秒。

- 带 CDN 的 TTFB
我们的总加载时间下降到 788 毫秒，我们的平均 TTFB 现在是 37 毫秒！CDN 可以带来多大的不同

注意：如果您使用 Cloudflare，您的TTFB可能略高。这很可能是由于运行完全代理服务的额外开销和复杂性。Cloudflare 具有某些 CDN 提供商没有的其他防火墙和其他功能。

## 3. WordPress 缓存

减少 TTFB 的第三种方法，也可能是最简单的方法之一，就是在 WordPress 网站上使用缓存。许多人只认为缓存可以帮助减少加载时间，但实际上，它也有助于减少 TTFB，因为它有助于减少服务器处理时间。


## 4. 使用高级 DNS 提供商

DNS 也在 TTFB 中发挥作用。很难准确计算它受到的影响有多大，但您仍然可以看到总体 DNS 查找时间，并看到那里有越来越快的提供商。

[SolveDNS 速度测试工具](http://www.solvedns.com/dnsspeedtest/) 可以帮我们测试DNS解析速度。 大的云厂商，如 AWS 会比一些免费的DNS提供商的质量要好很多。


更多内容可以查看 [Why Premium DNS is No Longer Optional][https://kinsta.com/blog/premium-dns/)
