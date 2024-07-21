# eBPF


{{< typeit tag=h4 >}}
eBPF 是什么？
{{< /typeit >}}

{{< image src="https://img1.kiosk007.top/static/images/blog/ebpf_logo.png" caption="eBPF" src_s="https://img1.kiosk007.top/static/images/blog/ebpf_logo.png" src_l="https://img1.kiosk007.top/static/images/blog/ebpf_logo.png" >}}

Linux 内核一直是实现监控/可观测性、网络和安全功能的理想地方。 不过很多情况下这并非易事，因为这些工作需要修改内核源码或加载内核模块， 最终实现形式是在已有的层层抽象之上叠加新的抽象。 eBPF 是一项革命性技术，它能在内核中运行沙箱程序（sandbox programs）， 而无需修改内核源码或者加载内核模块。
<br/>

将 Linux 内核变成可编程之后，就能基于现有的（而非增加新的）抽象层来打造更加智能、 功能更加丰富的基础设施软件，而不会增加系统的复杂度，也不会牺牲执行效率和安全性。
<br/>
<br/>

- 官网：{{< link "https://ebpf.io/zh-cn/" eBPF.io >}}
- 项目列表：{{< link "https://ebpf.io/projects/" eBPF projects >}}

<br/>
<br/>
相关文章链接：

性能：
1. {{< link "https://colobu.com/2022/06/05/use-bpf-to-make-the-go-network-program-8x-faster/" 使用BPF将Go网络程序的吞吐提升8倍 >}}

观测：
1. {{< link "https://blog.huoding.com/2021/12/12/970" 如何用eBPF分析Golang应用 >}}
2. {{< link "https://colobu.com/2022/05/22/use-ebpf-to-trace-rpcx-microservices/" 使用ebpf跟踪rpcx微服务 >}} 
3. {{< link "https://blog.0x74696d.com/posts/challenges-of-bpf-tracing-go/" "Challenges of BPF Tracing Go" >}}
4. <a href="https://colobu.com/2017/09/22/golang-bcc-bpf-function-tracing/">[译]使用 bcc/BPF 分析 go 程序</a>
5. {{< link "https://blog.px.dev/ebpf-function-tracing/" "eBPF tracing in Go" >}}

<br/>
<br/>

视频资料：

{{< bilibili id=BV19U4y1N7PM >}}
