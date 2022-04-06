---
title: Express
type: express
disallow: true
comments: false
date: 2021-03-05 23:41:26
---

There are all kinds of mysterious portals here

这里有各式各样的神秘传送门

> 百宝箱

## Blog
- https://colobu.com/ --鸟窝
- https://geektutu.com/ --极客兔兔
- https://eddycjy.com/  --煎鱼

## GoLang 
- 标准库
    - [Go标准库](https://colobu.com/2016/10/12/go-file-operations/)
- 高性能
    - [百万 Go TCP 连接的思考: epoll方式减少资源占用](https://colobu.com/2019/02/23/1m-go-tcp-connection/)
    - [Go语言中实现基于 event-loop 网络处理](https://colobu.com/2017/11/29/event-loop-networking-in-Go/)
- io、bufio、ioutil(Go1.16废弃)    
    - [从 io.Reader 中读数据](https://colobu.com/2019/02/18/read-data-from-net-Conn/)
    - [GO语言基础进阶教程：bufio包](https://zhuanlan.zhihu.com/p/73690883)
    - [bufio — 缓存IO](https://books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter01/01.4.html)
    - [Go文件操作大全](https://colobu.com/2016/10/12/go-file-operations/)
- Other 
    - [优秀开源日志包使用教程](https://github.com/marmotedu/geekbang-go/blob/master/%E4%BC%98%E7%A7%80%E5%BC%80%E6%BA%90%E6%97%A5%E5%BF%97%E5%8C%85%E4%BD%BF%E7%94%A8%E6%95%99%E7%A8%8B.md)

## Tools
- Linux
    - [使用 tc netem 模拟网络异常](https://cizixs.com/2017/10/23/tc-netem-for-terrible-network/)
    - [Create-TLS-Certificates-Using-CFSSL](https://support.pingcap.com/hc/en-us/articles/360050038113-Create-TLS-Certificates-Using-CFSSL)
- Network
    - [BGP Tool](https://bgp.he.net/)
- Golang Format
    - [Golang json Tools](https://mholt.github.io/json-to-go/)
    - [Golang yaml Tools](https://zhwt.github.io/yaml-to-go/)
- OS 
    - [zorin OS - Make your computer better.](https://zorin.com/os/)
    - [cutefish OS - 你的另一个选择](https://cn.cutefishos.com/)

## K8S
- Minikube
    - [https://minikube.sigs.k8s.io/](https://minikube.sigs.k8s.io/)
    

---- 

## Framework / Library
1. **Go-metrics -- https://github.com/rcrowley/go-metrics**
 - 指标统计实现
 - [使用Grafana监控Go应用](https://lihaoquan.me/2017/2/2/monitor-go-with-influxdb-and-grafana.html)
 - [搭建你自己的Go Runtime metrics环境](https://tonybai.com/2017/07/04/setup-go-runtime-metrics-for-yourself/)

2. **Cobra -- https://github.com/spf13/cobra**
 - Cobra 既是一个用于创建强大的现代 CLI 应用程序的库，也是一个用于生成应用程序和命令文件的程序。
 - [golang命令行库--Cobra](https://cobra.dev/)
 - [Golang : cobra 包简介](https://www.cnblogs.com/sparkdev/p/10856077.html)

3. **Viper -- https://github.com/spf13/viper**
 - Viper 是 Go 应用程序的完整配置解决方案。
 - [spf13/viper——Go应用程序的完整配置解决方案](https://www.jianshu.com/p/7bb4f7f69280)

4. **go-mirco -- https://github.com/micro/micro/v2**
 - Micro 是一个用于构建和管理分布式系统的系统.
 - [Micro](http://m2.topgoer.com/%E5%BC%80%E5%A7%8B/%E6%A6%82%E8%BF%B0.html)
 - [Go Micro 入门](http://www.topgoer.com/%E5%BE%AE%E6%9C%8D%E5%8A%A1/GoMicro%E5%85%A5%E9%97%A8.html)

5. **easyjson -- https://github.com/mailru/easyjson**
 - easyjson 是用来快速进行json序列化与反序列化的工具包，通过给我们要进行序列化的struct生成方法来实现不通过反射进行json序列化，对比golang原有json工具包，性能能够提高3倍以上。
 - [Golang高性能json包：easyjson](https://my.oschina.net/qiangmzsx/blog/1503018)
 

6. **gorm -- https://gorm.io/gorm**
 - orm 全称 object relation mapping 对象映射关系，目的是解决面向对象和关系数据库之间存在的互不匹配的现象。orm 也较好的处理了 sql注入的拼接sql行为，而 gorm 就是 golang 的实现
 - [GORM 指南](https://gorm.io/zh_CN/docs/index.html) 

7. **Ginkgo -- github.com/onsi/ginkgo/ginkgo**
 - Ginkgo是一个BDD风格的Go测试框架，旨在帮助你有效地编写富有表现力的全方位测试。它最好与Gomega匹配器库配对使用，但它的设计是与匹配器无关的。
 - [Ginkgo](https://www.ginkgo.wiki/chapter1.html)

----
 ## Funny
1. [red alert](https://game.chronodivide.com/)