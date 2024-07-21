---
title: Golang
subtitle: Build fast, reliable, and efficient software at scale
type: gallery
disallow: true
lightgallery: true
comment: false
featuredImage: https://img1.kiosk007.top/static/images/go/golang.png
date: 2022-06-08 23:41:26
---


{{< typeit tag=h4 >}}
To Be a Gopher !!!
{{< /typeit >}}

{{< image src="https://img1.kiosk007.top/static/images/go/golang1.webp" caption="golang" src_s="https://img1.kiosk007.top/static/images/go/golang1.webp" src_l="https://img1.kiosk007.top/static/images/go/golang1.webp" title="Hello Golang" >}}

不说其他的，来做一个牛逼的 Gopher 吧。:surfer:

<br/>
<br/>

> 官网：{{< link "https://golang.google.cn/" golang >}}

- GoLang Language：
1. [Uber开源的Go语言开发规范](https://github.com/uber-go/guide/blob/master/style.md)
2. [超全总结：Go语言如何操作文件](https://segmentfault.com/a/1190000042353267?utm_source=sf-similar-article)

<br/>

- GoLang 实用小技巧
1. [Go实用小技巧](https://segmentfault.com/a/1190000021275844?utm_source=sf-similar-article)

<br/>

- 开源项目:

1. [推荐几个可以写到简历上的Go方向优质开源项目（需花点心思研究）](https://segmentfault.com/a/1190000041080720)
2. [https://landscape.cncf.io/](https://landscape.cncf.io/)  
3. [Go语言如何在测试发现goroutine泄漏](https://segmentfault.com/a/1190000041853511)


<br/>

- 基础：
1. [详解Go内联优化](https://segmentfault.com/a/1190000039146279)
2. [详解Go语言中的内存逃逸](https://segmentfault.com/a/1190000040450335)
3. [Go语言中结构体打Tag是什么意思？](https://segmentfault.com/a/1190000040999517)
4. [Golang 详解内存对齐](https://segmentfault.com/a/1190000040528007)
5. [Mutex：如何解决资源并发访问问题?](https://time.geekbang.org/column/article/294905)

<br/>

- 基础工具包

并发检测

``` bash
$ go test -race mypkg
$ go run -race mysrc.go
$ go build -race mycmd
$ go install -race mypkg
```