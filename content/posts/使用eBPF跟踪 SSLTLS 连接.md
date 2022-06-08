---
title:  使用eBPF跟踪 SSL/TLS 连接
author: kiosk
draft: false
tags:
  - eBPF
categories:
  - Linux
date: 2022-05-15 16:30:00
---



前面的文章介绍了 eBPF 的入门知识，这篇算是一个实战吧，使用 eBPF 来跟踪 TLS加密连接。[TLS](https://kiosk007.top/post/tls%E8%AF%A6%E8%A7%A3-%E4%B8%80/) 是目前互联网上的一个标准工具了，可用于加密的通信。本文尝试使用 eBPF 技术将加密的密文还原为明文。当然这并不意味着TLS不再安全，想干坏事的人可以在中间代理服务器、网关设备上偷听。因为eBPF 所实现的追踪明文一定是要建立在真正去做ssl 明文转密文写入操作的。

<br>

关于eBPF基础知识请查看

- [认识eBPF](https://kiosk007.top/post/%E8%AE%A4%E8%AF%86ebpf/)

- [如何使用eBPF进行追踪](https://kiosk007.top/post/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8ebpf%E8%BF%9B%E8%A1%8C%E8%BF%BD%E8%B8%AA/)



# 简介

BPF 可用于对内核函数、用户态函数进行插桩进而实现观测的需求。当然一个有趣的应用就是跟踪网络流量。虽然TLS 流量通过网卡时是密文的，但是我们可以利用 eBPF在加密前进行跟踪。eBPF 的事件追踪并不局限于 kprobes，uprobes也可以用来追踪应用程序代码到达某个指令时触发 BPF 程序。

{{< image src="https://img1.kiosk007.top/static/images/blog/bpf-tls-tracing.svg" caption="TLS Tracing (`openssl`)" src_s="https://img1.kiosk007.top/static/images/blog/bpf-tls-tracing.svg" src_l="https://img1.kiosk007.top/static/images/blog/bpf-tls-tracing.svg" >}}

通过上图，我们可以知道TLS库的数据读写是 `SSL_write` 和 `SSL_read` 函数。跟踪这些调用就可以在加密之前或者解密之后的流量。



## 查询跟踪函数

其实上面的图已经说明了我们要追踪的函数是 `SSL_write` 和 `SSL_read` ，但是这里也聊一下这两个函数是如何查找的吧。一般来说，我们可以使用 `readelf` 命令。输出的结果，排除一些明显和加密过程相关的函数之外，映入眼帘的便是 `SSL_read` 

```powershell
$ readelf -Ws /usr/lib/x86_64-linux-gnu/libssl.so |grep -i 'ssl_read'
   658: 0000000000032ce0   126 FUNC    GLOBAL DEFAULT   15 SSL_read@@OPENSSL_3.0.0
   762: 0000000000038cf0   414 FUNC    GLOBAL DEFAULT   15 SSL_read_early_data@@OPENSSL_3.0.0
   811: 0000000000032d60    25 FUNC    GLOBAL DEFAULT   15 SSL_read_ex@@OPENSSL_3.0.0

```

也可以使用 bpftrace 或者 nm 命令进行相关追踪。

```powershell
$ sudo bpftrace -l 'uprobe:/usr/lib/x86_64-linux-gnu/libssl.so:*'
$ nm --dynamic /usr/lib/x86_64-linux-gnu/libssl.so|grep read
```



有了追踪点，就可以开始完成一次 openssl  的追踪了。

# eBPF 代码

当设计到选择库和工具来与 eBPF 进行交互时，会有些让人困惑，有基于 [Python 的 BCC 框架](https://github.com/iovisor/bcc) 、基于 C 的 [lbbpf](https://github.com/libbpf/libbpf) 和一系列基于 Go 的 [Dropbox](https://github.com/dropbox/goebpf)、[Cilium](https://github.com/cilium/ebpf)、[Aqua](https://github.com/aquasecurity/tracee/tree/main/libbpfgo) 和 [Calico](https://github.com/projectcalico/calico/tree/master/felix/bpf) 等选择。

我们知道，eBPF库主要协助实现了两个功能。

- 将 eBPF 程序和Map载入内核并执行加载，通过其文件描述符将 eBPF 程序与正确的 Map 进行关联。
- 与 eBPF Map 交互，允许对存储在Map中的键/值对进行标准的 CRUD 操作。

<br>

## 框架选择

我们选择 Golang 作为我们的 eBPF 交互部分的开发语言。每个库都有各自的范围和限制。

- [Calico](https://github.com/projectcalico/calico/tree/master/felix/bpf)：用 bpftool 和 iproute2 实现的 CLI 命令基础上用 Go 封装了一层。
- [Aqua](https://github.com/aquasecurity/tracee/tree/main/libbpfgo)：对 libbpf C 库的一层封装。
- [Dropbox](https://github.com/dropbox/goebpf)：支持了一小部分程序，但有一个方便的用户API
- [iovisor](https://github.com/iovisor/bcc)：是BCC 框架的 Go 语言绑定，更注重于跟踪和性能分析。
- [Cilium](https://github.com/cilium/ebpf)：是 Cilium 和 Cloudflare 公司维护的一个[纯Go编写的库](https://linuxplumbersconf.org/event/4/contributions/449/attachments/239/529/A_pure_Go_eBPF_library.pdf) ，将所有的 eBPF 系统调用抽象在一个本地的 Go 接口后面。

参考 [使用Go语言管理和分发ebpf程序](https://www.ebpf.top/post/ebpf_go/)可以看到 `cilium/ebpf` 更加活跃，本文也选择基于 `cilium/ebpf` 来开发，因为其纯Go语言编写，**从而实现了程序最小依赖**，与此同时其还提供了 `epf2go` 小工具，可以用来将 eBPF 程序编译成为 Go 语言的一部分，使得交互变得更加方便，后续如果配合 CO_RE 功能则更加 :ox: :beers: ​

<br>

# **前期梳理**

我们需要一个可以按照应用过滤的 SSL 解密器。需要实现这个功能就必须按照 PID 进行追踪。在[BCC](https://github.com/iovisor/bcc)项目中有不少程序都是直接使用 bpf_get_current_pid_tgid() 直接与用户空间传入的 pid 进行对比的。[^1] 但是BCC项目中大多是 Python 语言开发，也可以通过加载 BPF 程序时添加编译参数的形式传入用户空间的 PID 数据[^2]，但是我们用的是 Go 语言，且使用 cilium 开发相关功能，所以需要一些脚手架。

这里比较推荐 https://github.com/ehids/ebpfmanager 这个项目，ebpfmanager参照datadog/ebpf/manager包的思想，基于cilium/ebpf实现的ebpf类库封装。

(另外有一个比较重要的点是因为他注入变量方便一些)



<br>

# 功能开发

可以在 [此处](https://github.com/kiosk404/openssl_tracer) 找到完整的 Openssl 跟踪器，运行请参考 [README](https://github.com/kiosk404/openssl_tracer) 。



项目中的主要目录如下：

- **bpf** : 包含 bpf 探针C程序和依赖的头文件
- **uprobe_tracing** : 用户空间追踪器
- **ssl_client_server** : Python 的 openssl 通信 Demo 运行样例，可以无限产生 ssl 通信数据。



## BPF 程序



开发前需要先导入必要的头文件，这个在 [bpf/openssl_trace.bpf.h](https://github.com/kiosk404/openssl_tracer/blob/master/bpf/openssl_trace.bpf.h) , 其中的 vmlinux.h 是生成的，可以使用 bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h 命令生成。其余的头文件是安装 libbpf-dev 获取到的。



BPF uprobes 的主要目的是跟踪 `SSL_write` 和 `SSL_read` 函数并捕获数据，这意味着我们需要跟踪这些函数的入口和出口。目前，用户态探针有两种类型： uprobes 和 uretprobes（也叫 return 探针）。可以在应用程序的虚拟地址空间的任意指令上插入 uprobe 。 当用户函数返回的时候触发 return 探针。

|                        功能                        |         探测操作          |
| :------------------------------------------------: | :-----------------------: |
| int SSL_write(SSL *ssl, const void *buf, int num); |  将写数据记录到BPF映射中  |
|    int SSL_read(SSL *ssl, void *buf, int num);     | 将读的数据记录到BPF映射中 |

</br>

下面介绍详细的 BPF 代码前需要强调的是， libbpf 的程序方式会定义一些宏。在编译时，通过 SEC() 宏定义的数据结构交互。



```c
struct ssl_data_event_t {
    enum ssl_data_event_type type;
    u64 timestamp_ns;
    u32 pid;
    u32 tid;
    char data[MAX_DATA_SIZE_OPENSSL];
    s32 data_len;
    char comm[TASK_COMM_LEN];
    u32 fd;
};

// BPF programs are limited to a 512-byte stack. We store this value per CPU
// and use it as a heap allocated value.
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __type(key, u32);
    __type(value, struct ssl_data_event_t);
    __uint(max_entries, 1);
} data_buffer_heap SEC(".maps");
```

<br>

这里的 BPF_MAP_TYPE_PERCPU_ARRAY 是每个 CPU的另一个 BPF 映射。我们将其用作临时的暂存空间。因为不能将太多的数据复制到BPF堆栈上。我们创建了多个数据结构来存储通信过程产生的数据。



**libbpf_register_prog_handler()** 注册一个自定义 BPF 程序 SEC() 处理程序。

接下来，将展示探针代码 SSL_write 函数的追踪，所有的 BPF 程序提供的功能都需要通过 SEC() (来自 bpf_helpers.h) 宏来定义 section 名称

```c
// Function signature being probed:
// int SSL_write(SSL *ssl, const void *buf, int num);
SEC("uprobe/SSL_write")
int probe_entry_SSL_write(struct pt_regs* ctx) {
    u64 current_pid_tgid = bpf_get_current_pid_tgid();
    u32 pid = current_pid_tgid >> 32;
#ifndef KERNEL_LESS_5_2
    // if target_ppid is 0 then we target all pids
    if (target_pid != 0 && target_pid != pid) {
        return 0;
    }
#endif

    void* ssl = (void*)PT_REGS_PARM1(ctx);
    // https://github.com/openssl/openssl/blob/OpenSSL_1_1_1-stable/crypto/bio/bio_local.h
    struct ssl_st ssl_info;
    bpf_probe_read_user(&ssl_info, sizeof(ssl_info), ssl);

    struct BIO bio_w;
    bpf_probe_read_user(&bio_w, sizeof(bio_w), ssl_info.wbio);

    // get fd ssl->wbio->num
    u32 fd = bio_w.num;
    //    debug_bpf_printk("openssl uprobe SSL_write FD:%d\n", fd);

    const char* buf = (const char*)PT_REGS_PARM2(ctx);
    struct active_ssl_buf active_ssl_buf_t;
    __builtin_memset(&active_ssl_buf_t, 0, sizeof(active_ssl_buf_t));
    active_ssl_buf_t.fd = fd;
    active_ssl_buf_t.buf = buf;
    bpf_map_update_elem(&active_ssl_write_args_map, &current_pid_tgid,
                        &active_ssl_buf_t, BPF_ANY);

    return 0;
}
```

<br>

入口函数中，先获取当前进程的 PID 并对比是不是感兴趣的PID，如果没有 target_pid 没有定义的话，则默认监控所有的 ssl 通信。如果是感兴趣的目标PID，则探测器会读取缓冲区指针并将值存储在全局PID 中 active_ssl_write_args_map 。

通过 PT_REGS_PARM1 获取 int SSL_write(SSL *ssl, const void *buf, int num);  函数中的第一个参数 ssl 。再通过 bpf_probe_read_user 辅助函数将ssl里的信息读入到 ssl结构体里。

第二个参数 buf 是实际的明文数据，是写入加密前的明文数据。再将数据通过辅助函数 [bpf_map_update_elem() ](https://prototype-kernel.readthedocs.io/en/latest/bpf/ebpf_maps.html#interacting-with-maps) 将BPF映射中的数据修改。



```c
SEC("uretprobe/SSL_write")
int probe_ret_SSL_write(struct pt_regs* ctx) {
    u64 current_pid_tgid = bpf_get_current_pid_tgid();
    u32 pid = current_pid_tgid >> 32;

#ifndef KERNEL_LESS_5_2
    // if target_ppid is 0 then we target all pids
    if (target_pid != 0 && target_pid != pid) {
        return 0;
    }
#endif
    //    debug_bpf_printk("openssl uretprobe/SSL_write pid :%d\n", pid);
    struct active_ssl_buf* active_ssl_buf_t =
        bpf_map_lookup_elem(&active_ssl_write_args_map, &current_pid_tgid);
    if (active_ssl_buf_t != NULL) {
        const char* buf;
        u32 fd = active_ssl_buf_t->fd;
        bpf_probe_read(&buf, sizeof(const char*), &active_ssl_buf_t->buf);
        process_SSL_data(ctx, current_pid_tgid, kSSLWrite, buf, fd);
    }
    bpf_map_delete_elem(&active_ssl_write_args_map, &current_pid_tgid);
    return 0;
}
```

<br>

等待函数退出后，通过 process_SSL_data 函数并最终在该函数中通过 bpf_perf_event_output 将 tls_events 内容推送到 perf 缓冲区，用户空间跟踪程序会在被唤醒时打印到屏幕。

## 用户空间追踪器

用户空间程序最大的作用就是从 BPF 映射的缓冲区中读取数据。

用户态程序使用 go 语言开发，基于cilium 的 goebpf ，这里用 https://github.com/ehids/ebpfmanager 封装了一下。ebpfmanager参照datadog/ebpf/manager包的思想，实现了配置化、自动加载，我选择这个的最大原因，其对用户态程序向 BPF 程序传递参数的封装做的比较好。



```go
const binaryPath = "/lib/x86_64-linux-gnu/libssl.so"

var m = &manager.Manager{
	Probes: []*manager.Probe{
		{
			Section:          "uprobe/SSL_write",
			EbpfFuncName:     "probe_entry_SSL_write",
			AttachToFuncName: "SSL_write",
			BinaryPath:       binaryPath,
		},
		{
			Section:          "uretprobe/SSL_write",
			EbpfFuncName:     "probe_ret_SSL_write",
			AttachToFuncName: "SSL_write",
			BinaryPath:       binaryPath,
		},
		{
			Section:          "uprobe/SSL_read",
			EbpfFuncName:     "probe_entry_SSL_read",
			AttachToFuncName: "SSL_read",
			BinaryPath:       binaryPath,
		},
		{
			Section:          "uretprobe/SSL_read",
			EbpfFuncName:     "probe_ret_SSL_read",
			AttachToFuncName: "SSL_read",
			BinaryPath:       binaryPath,
		},
	},
	Maps: []*manager.Map{
		{
			Name: "tls_events",
		},
	},
}


```

<br>

这里我们需要定义探测器，一个函数入口一个函数返回。 manager.Map 定义了要读取的perf 事件，由上面介绍的 BPF 推送而来。

```go
func EBPFManagerOption(pid int) manager.Options {
	return manager.Options{
		DefaultKProbeMaxActive: 512,

		VerifierOptions: ebpf.CollectionOptions{
			Programs: ebpf.ProgramOptions{
				LogSize: 2097152,
			},
		},

		RLimit: &unix.Rlimit{
			Cur: math.MaxUint64,
			Max: math.MaxUint64,
		},
		ConstantEditors: constantEditor(pid),
	}
}

func constantEditor(pid int) []manager.ConstantEditor {
	var editor = []manager.ConstantEditor{
		{
			Name:  "target_pid",
			Value: uint64(pid),
			FailOnMissing: true,
		},
	}

	if pid <= 0 {
		logrus.Printf("target all process. \n")
	} else {
		logrus.Printf("target PID:%d \n", pid)
	}
	return editor
}
```

<br>

这里的 Manager.ConstantEditor 中可以实现对 target_pid 的修改，并且可以传递给 BPF 程序。



封装下的 cilium 已经非常简单了。到此初始化部分基本完成，还剩下 [event.go](https://github.com/kiosk404/openssl_tracer/blob/master/uprobe_tracing/event.go)（tls_events 事件结构的定义）和 [user.go](https://github.com/kiosk404/openssl_tracer/blob/master/uprobe_tracing/user.go) (读取BPF映射数据) 中的东西。



```go
func PerfEventReader(errChan chan error, em *ebpf.Map, ctx context.Context) {
	rd, err := perf.NewReader(em, os.Getpagesize()*64)
	if err != nil {
		errChan <- fmt.Errorf("creating %s reader dns: %s", em.String(), err)
		return
	}

	defer rd.Close()
	for {
		//判断ctx是不是结束
		select {
		case _ = <- ctx.Done():
			logrus.Printf("readEvent received close signal from context.Done.")
			return
		default:
		}

		record, err := rd.Read()
		if err != nil {
			if errors.Is(err, perf.ErrClosed) {
				return
			}
			errChan <- fmt.Errorf("reading from perf event reader: %s", err)
			return
		}

		if record.LostSamples != 0 {
			logrus.Printf("perf event ring buffer full, dropped %d samples", record.LostSamples)
			continue
		}

		var sslEvent = &SSLDataEvent{}
		err = sslEvent.Decode(record.RawSample)
		if err != nil {
			log.Printf("decode error:%v", err)
			continue
		}

		// 打印数据
		str := sslEvent.String()
		fmt.Println(str)
	}
}
```

通过 perf.NewReader 方法将 BPF 映射中的 tls_events 事件读出，再经过 event.go 文件中

## 效果

最后看看效果吧，不加 --pid 参数的情况下可以追踪所有使用了 openssl 动态库的程序传输的加密数据。

![ebpf_openssl_tracer](https://img1.kiosk007.top/static/images/blog/ebpf_openssl_tracer.png)



这里把 curl 的内容也可以打印。但是加密库不止 openssl.so 这一种。还有 boringssl ，不过追踪方法大同小异。





<br>

文章参考：

- [ebpf-openssl-tracing](https://blog.px.dev/ebpf-openssl-tracing/)
- [使用 Go 语言开发 ebpf 程序](https://houmin.cc/posts/adca5ae5/)

[^1]:https://www.ebpf.top/post/ebpf_prog_pid_filter/
[^2]:https://github.com/pixie-io/pixie-demos/blob/main/openssl-tracer/openssl_tracer.cc#L107