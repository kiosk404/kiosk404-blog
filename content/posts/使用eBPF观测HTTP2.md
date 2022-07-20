---
title: 使用eBPF观测HTTP2
author: kiosk
draft: true
math: true
tags:
  - eBPF
  - golang
categories:
  - Linux
date: 2022-06-04 16:30:00
---

在当今充满微服务的世界中，对服务之间的消息传输的可观察性和理解变得越来越重要。

其中一些微服务的传输协议比较复杂，如HTTP/2 协议为了降低头部内容的重复传输，使用表头压缩算法 HPACK，虽然HPACK有助于提升 HTTP/2 的传输效率，但也使得跟踪变得复杂。不过，通过使用 eBPF uprobes,可以在 Header 被压缩之前对其进行跟踪。

![ebpf_logo](https://img1.kiosk007.top/static/images/blog/ebpf_logo.png)

<!--more-->



# 为什么HTTP2 会难以跟踪

我们知道HTTP2是一个取代HTTP1.1的协议，其解决了大量HTTP1.1 所为人诟病的缺点，如大头、管线化等问题。

但是 HTTP/2 的头部压缩技术是一个靠通信双方维护字典的方式进行压缩通信。所以如果且少从头开始追踪的动态字典索引的话，无法完整的还原压缩前的字典头。具体 Hpack 的原理可以参考我之前的[HTTP/2.0 Header Compression](https://kiosk007.top/post/http-2-0-header-compression/)这篇文章。



# eBPF uprobes 如何解决跟踪难题

那么该如何正确解密 HTTP/2 流量呢，eBPF 技术可以使得我们探究 HTTP/2 通信的细节。而不需要协议本身的运行状态。



具体来说，eBPF uprobes 通过直接跟踪引用程序内存中的明文数据来解决HPACK问题。通过将 uprobes 附加到 HTTP/2 的API库，uprobes 就能在 Hpack 压缩之前从应用程序的内存中读取到Header 头部。由于我们 [之前的文章](https://kiosk007.top/post/http-2-%E7%89%B9%E6%80%A7%E6%A6%82%E8%A7%88/) 已经探索了 Go 官方库对HTTP2 的实现，那么这次肯定还是追踪 Golang 编写的 HTTP2 程序。

<br/>

# Golang 如何进行eBPF追踪

在开始 HTTP2 的追踪之前，我们先看一下 Golang 程序是如何进行eBPF追踪的。这个demo是一个简单的HTTP服务器，computeE 函数使用迭代近似计算欧拉数（$ e $）, 接受单个查询参数（iters），参数 iters 越大表示迭代次数越多，迭代次数越多，近似值约准确。

```go
// computeE computes the approximation of e by running a fixed number of iterations.
func computeE(iterations int64) float64 {
  res := 2.0
  fact := 1.0

  for i := int64(2); i < iterations; i++ {
    fact *= float64(i)
    res += 1 / fact
  }
  return res
}

func main() {
  http.HandleFunc("/e", func(w http.ResponseWriter, r *http.Request) {
    // Parse iters argument from get request, use default if not available.
    // ... removed for brevity ...
    w.Write([]byte(fmt.Sprintf("e = %0.4f\n", computeE(iters))))
  })
  // Start server...
}

```

为了跟踪 uprobes，需要看二进制文件中的符号，Linux 上的 Go 二进制文件使用 ELF 来存储调试信息，除非是已剥离调试数据，否则即可在二进制文件中找到。可以使用 `objdump` 或 `readelf`等工具检查二进制文件中的符号。

``` powershell
$ objdump --syms app|grep "computeE" 
00000000005ebc40 g     F .text	0000000000000043              main.computeE

$ readelf -Ws app |grep "computeE"
  4618: 00000000005ebc40    67 FUNC    GLOBAL DEFAULT    1 main.computeE

$ nm  ./app|grep "computeE" 
00000000005ebc40 T main.computeE

$ sudo bpftrace -l 'uprobe:./app:*' |grep computeE
uprobe:./app:main.computeE 

```

<br>

至此，就可以开始一个简单的 eBPF 程序的追踪过程了。

```go
package main

import (
	"encoding/binary"
	"flag"
	"fmt"
	"github.com/iovisor/gobpf/bcc"
	"os"
	"os/signal"
)

const bpfProgram = `
#include <uapi/linux/ptrace.h>
BPF_PERF_OUTPUT(trace);
// This function will be registered to be called everytime
// main.computeE is called.
inline int computeECalled(struct pt_regs *ctx) {
  // The input argument is stored in ax.
  long val = ctx->ax;
  trace.perf_submit(ctx, &val, sizeof(val));
  return 0;
}
`

var binaryProg string

func init() {
	flag.StringVar(&binaryProg, "binary", "", "The binary to probe")
}

func main() {
	flag.Parse()
	if len(binaryProg) == 0 {
		panic("Argument --binary needs to be specified")
	}

	bccMod := bcc.NewModule(bpfProgram, []string{})
	uprobeFD, err := bccMod.LoadUprobe("computeECalled")
	if err != nil {
		panic(err)
	}

	// Attach the uprobe to be called everytime main.computeE is called.
	// We need to specify the path to the binary so it can be patched.
	err = bccMod.AttachUprobe(binaryProg, "main.computeE", uprobeFD, -1)
	if err != nil {
		panic(err)
	}

	// Create the output table named "trace" that the BPF program writes to.
	table := bcc.NewTable(bccMod.TableId("trace"), bccMod)
	ch := make(chan []byte)

	pm, err := bcc.InitPerfMap(table, ch, nil)
	if err != nil {
		panic(err)
	}

	// Watch Ctrl-C so we can quit this program.
	intCh := make(chan os.Signal, 1)
	signal.Notify(intCh, os.Interrupt)

	pm.Start()
	defer pm.Stop()

	for {
		select {
		case <-intCh:
			fmt.Println("Terminating")
			os.Exit(0)
		case v := <-ch:
			// This is a bit of hack, but we know that iterations is a
			// 8 bytes int64 value.
			d := binary.LittleEndian.Uint64(v)
			fmt.Printf("Value = %v\n", d)
		}
	}
}
```



<br>

当我们启动了 ebpf 程序，并完成一次 curl 127.0.0.1:9090/e 访问时，可以看到 computeE 函数的参数是 Value=100

<br/>

# 基于Uprobe 的HTTP/2 跟踪

我们可以基于 eBPF 技术使我们可以探究 HTTP/2 实现以获取我们需要的信息和需要的状态。

<br/>

具体来说，eBPF uprobes 通过直接跟踪应用程序中的明文数据来解决 HPACK 问题。通过将 uprobes 可以在HPACK压缩前拿到标头的内容。

通过研究 Golang 的 HTTP/2 源码，可以看到 [loopWriter.writeHeader()](https://github.com/grpc/grpc-go/blob/584d9cd11a1da55e3558041c9f88f22ca2255f4e/internal/transport/controlbuf.go#L678)是一个理想的跟踪点。此函数接受明文标题字段并将它们发送到内部缓冲区。

<br/>

> 这里演示通过 GRPC 演示，而 GRPC 底层基于 HTTP2 ，所以看的是 grpc 的golang 官方库。



接下来的事情就是弄清楚数据结构的内存布局，并编写 BPF 代码以在正确的内存地址读取数据了。

先看看函数签名。

```go
func (l *loopyWriter) writeHeader(streamID uint32, endStream bool, \
                                  hf []hpack.HeaderField, onWrite func())

```

第三个参数 hf 是`HeaderField` ，就是 hpack 中的内容，

















参考：

- https://blog.px.dev/ebpf-http2-tracing/
- https://blog.px.dev/ebpf-function-tracing/