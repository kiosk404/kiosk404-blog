---
title: 如何使用eBPF进行追踪
author: kiosk
draft: false
tags:
  - eBPF
categories:
  - Linux
date: 2022-05-10 16:30:00
---



eBPF 我们知道了程序的工作原理和编程接口以及事件触发机制。那么我们就可以开始 eBPF 程序的开发和执行过程了。

跟踪类 eBPF 程序主要包含 内核插桩 （BPF_PROG_TYPE_KPROBE）、跟踪点（BPF_PROG_TYPE_TRACEPONT）以及性能事件（BPF_TYPE_PERF_EVENT）等类型的程序，而每类的 eBPF 程序又可以挂载到不同的内核函数、内核跟踪点或者性能事件上。当这些内核函数、内核跟踪点或者性能事件被调用的时候，挂载到其上的 eBPF 程序就会自动执行。

<!--more-->

那么问题来了！我怎么知道有哪些插桩、跟踪点、性能事件可以用呢？难道真的需要熟读内核代码吗？



# 内核跟踪点

**内核本身的调试和跟踪一直是内核提供的核心功能之一**。内核符号表 `/proc/kallsyms` 不仅包含了内核函数，还包含了非栈数据变量等等，其中被显式导出的内核函数（7w+）才能被eBPF kprobe 类型程序动态追踪。

以 v.5.15 内核版本为例，约有 7.1w 个内核函数

```
# cat /sys/kernel/debug/tracing/available_filter_functions | wc -l
71961
```



除了内核函数和跟踪点外，性能事件可以使用 [perf](https://man7.org/linux/man-pages/man1/perf.1.html) 命令来查询。

```

sudo perf list [hw|sw|cache|tracepoint|pmu|sdt|metric|metricgroup]
```





通常，我们会以 bpftrace 、 BCC 和libbpf 这三种方式进行开发 eBPF 程序。这三种方式各有优缺点。

- **bpftrace 通常用于快速排查问题和定位问题，支持用单行脚本的方式来快速开发并执行一个 eBPF 程序** ，不过 bpftrace 功能有限，不支持特别复杂的 eBPF 程序，也依赖 BCC 和 LLVM 动态编译执行。
- **BCC 通常用于开发复杂的 eBPF 程序**，但是BCC 依赖于 LLVM 和内核头文件才可以动态编译和加载 eBPF 程序。
- **libbpf 是从内核中抽离出来的标准库，用它开发的 eBPF 程序可以直接分发执行，这样就不需要每台机器都安装 LLVM 和内核头文件**。不过他需要内核开启 BTF 特性，需要较新的发行版才会默认开启（如 RHEL 8.2 + 和 Ubuntu 20.10 +）





## bpftrace 查询内核跟踪点

bpftrace 在 eBPF 和 BCC 之上构建了一个简化的跟踪语言，通过简单的几行脚本就可以实现复杂的跟踪功能，多行跟踪指令也可以放到脚本中执行（脚本后缀通常为 .bt）。

安装 bpftrace 

```

# Ubuntu 22.04
sudo apt-get install -y bpftrace

# RHEL8/CentOS8
sudo dnf install -y bpftrace
```



安装完成之后可以使用 bpftrace 加上 -l 的参数来查询内核插桩和跟踪点。

```bash

# 查询所有内核插桩和跟踪点
sudo bpftrace -l

# 使用通配符查询所有的系统调用跟踪点
sudo bpftrace -l 'tracepoint:syscalls:*'

# 使用通配符查询所有名字包含"execve"的跟踪点
sudo bpftrace -l '*execve*'
```

<br>

另外对于跟踪点来说，可以加上 -v 参数查询函数的入参还有返回值。比如查询系统调用 execve 入口参数（对应系统调用 sys_enter_execve）和返回值（对应系统调用 sys_exit_execve）的示例。



``` bash

# 查询execve入口参数格式
$ sudo bpftrace -lv tracepoint:syscalls:sys_enter_execve
tracepoint:syscalls:sys_enter_execve
    int __syscall_nr
    const char * filename
    const char *const * argv
    const char *const * envp

# 查询execve返回值格式
$ sudo bpftrace -lv tracepoint:syscalls:sys_exit_execve
tracepoint:syscalls:sys_exit_execve
    int __syscall_nr
    long ret
```



## 使用 bpftrace 进行跟踪

bpftrace 工具不仅仅可以用来查询相关跟踪点。还可以执行节点的 eBPF 跟踪功能。



比如我们知道Linux 下创建一个新进程通常需要调用 fork() 和 execve() 这两个标准函数。通过 bpftrace 可以查找包含 execve 关键字的跟踪点：

``` bash
# bpftrace -l "*execve"
kfunc:__ia32_compat_sys_execve
kfunc:__ia32_sys_execve
kfunc:__x64_compat_sys_execve
kfunc:__x64_sys_execve
kfunc:bprm_execve
kfunc:kernel_execve
kprobe:__ia32_compat_sys_execve
kprobe:__ia32_sys_execve
kprobe:__x64_compat_sys_execve
kprobe:__x64_sys_execve
kprobe:bprm_execve
kprobe:kernel_execve
tracepoint:syscalls:sys_enter_execve
tracepoint:syscalls:sys_exit_execve

```



因为内核插桩 kprobe 属于不稳定接口，而跟踪点属于稳定接口，**所以在内核插桩和跟踪点都可用的情况下，应该选择更稳定的跟踪点，以保证 eBPF 程序的可移植性**。所以这里我们选择 syscalls:sys_enter_execve 和 syscalls:sys_exit_execve 。这两个是系统调用的入口执行和出口。

<br>

1. **使用 ebpftrace 命令**：

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%-6d %-8s", pid, comm); join(args->argv);}'
```



这个命令的具体含义如下：

- bpftrace -e 表示加载bpf程序
- tracepoint:syscalls:sys_enter_execve 是跟踪点，如果多个跟踪点的话，需要逗号隔开
- printf()打印函数，语法和 C 语言的 printf() 一致
- pid 和 comm 是 bpftrace 的内置变量，分别表示进程PID和进程名称，完整的内置变量可在 [官方文档](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#1-builtins) 查找

- join(args->argv) 表示把字符串数组格式的参数用空格串起来，再打印到终端，对于跟踪点来说，可以使用  args->参数名  的方式直接读取参数（比如这里的  args->argv  就是读取系统调用中的  argv  参数）。



这下在其他终端执行的命令就会在这里被追踪到。

```
Attaching 1 probe...
68460  bash    ping qq.com
68653  cron    /bin/sh -c command -v debian-sa1 > /dev/null && debian-sa1 1 1
68654  sh      debian-sa1 1 1

```



还可以追踪某进程打开的文件：

```bash
bpftrace -e 'tracepoint:syscalls:sys_enter_openat /comm=="feishu"/ { printf("%-6d %-8s %-10s \n",pid, comm, str(args->filename)); }'
```



更多例子详见 https://www.tsingfun.com/it/os_kernel/bpftrace_tutorial.html

官方 bpftrace 脚本 https://github.com/iovisor/bpftrace/tree/master/tools



2. **还可以条件选择，比如我只追踪 curl 命令的网络丢包情况**。

``` bash
sudo bpftrace -e 'kprobe:kfree_skb /comm=="curl"/ {printf("kstack: %s\n", kstack);}'
```

这里两个 “**/**” 中间是条件过滤。



3. **还可以将简单的追踪写成脚本，用 bpftrace 调用**。这个脚本

```c
#!/usr/bin/env bpftrace
/* Watch tcp drop from curl process by probing kfree_skb */ 

// Add required kernel headers
#include <linux/skbuff.h>
#include <linux/ip.h>
#include <linux/netdevice.h>


kprobe:kfree_skb /comm=="curl"/
{
  // First arg is sk_buff.
  $skb = (struct sk_buff *)arg0;

  // Get network header, src IP and dst IP.
  $iph = (struct iphdr *)($skb->head + $skb->network_header);
  $sip = ntop(AF_INET, $iph->saddr);
  $dip = ntop(AF_INET, $iph->daddr);

  // Print kernel stack only when it is TCP.
  if ($iph->protocol == IPPROTO_TCP)
  {
    printf("SKB dropped: %s->%s, kstack: %s\n", $sip, $dip, kstack);
  }
}
```



<br>

## 使用 BCC 方法进行跟踪

我们知道使用 BCC 开发的 eBPF 程序包含两部分：

- 第一部分是用 C 语言开发的 eBPF 程序。
- 第二部分使用 BCC 工具开发，其中包含 eBPF 程序的加载、挂载到内核函数和跟踪点，以及通过 BPF 映射获取和打印执行结果等部分。



我们从第一步开始，不过这里不会做完整的介绍，我们还是监听 创建新进程这一事件。先给出两个使用 BCC 实现的链接吧。一个是[极客时间的实现](https://github.com/feiskyer/ebpf-apps/blob/main/bcc-apps/python/execsnoop.c) 还有一个是 [iovisor的官方例子 execsnoop](https://github.com/iovisor/gobpf/blob/master/examples/bcc/execsnoop/execsnoop.go) 。两个例子大同小异。



1. 创建一个 BPF 事件结构体，通过 `BPF_PERF_OUTPUT(events)` 把这个结构体向BPF映射抛
2. 定义好映射后，接下来就是定义跟踪点的处理函数。BCC 中可以像例子一中一样使用 `TRACEPOINT_PROBE(gategory, event) ` 来定义一个跟踪点处理函数。BCC 会将所有的参数放入到 args 变量中，使用 args-><参数名> 就可以访问跟踪点的参数值。当然也可以像例子二一样，直接定义一个函数，将event事件载入，在该函数设置event事件函数的变量，就像之前的 [openat2()](https://kiosk007.top/post/%E8%AE%A4%E8%AF%86ebpf/#%E8%BF%9B%E9%98%B6) 例子一样

例子一：

```c
// sys_enter_execve tracepoint.
TRACEPOINT_PROBE(syscalls, sys_enter_execve)
{
	// variables definitions
	unsigned int ret = 0;
	const char **argv = (const char **)(args->argv);

	// get the pid and comm
	struct data_t data = { };
	u32 pid = bpf_get_current_pid_tgid();
	data.pid = pid;
	bpf_get_current_comm(&data.comm, sizeof(data.comm));

	// get the binary name (first argment)
	if (__bpf_read_arg_str(&data, (const char *)argv[0]) < 0) {
		goto out;
	}
	// get other arguments (skip first arg because it has already been read)
#pragma unroll
	for (int i = 1; i < TOTAL_MAX_ARGS; i++) {
		if (__bpf_read_arg_str(&data, (const char *)argv[i]) < 0) {
			goto out;
		}
	}

 out:
	// store the data in hash map
	tasks.update(&pid, &data);
	return 0;
}
```



例子二：

```c
int syscall__execve(struct pt_regs *ctx,
    const char __user *filename,
    const char __user *const __user *__argv,
    const char __user *const __user *__envp)
{
    // create data here and pass to submit_arg to save stack space (#555)
    struct data_t data = {};
    struct task_struct *task;
    data.pid = bpf_get_current_pid_tgid() >> 32;
    task = (struct task_struct *)bpf_get_current_task();
    // Some kernels, like Ubuntu 4.13.0-generic, return 0
    // as the real_parent->tgid.
    // We use the getPpid function as a fallback in those cases.
    // See https://github.com/iovisor/bcc/issues/1883.
    data.ppid = task->real_parent->tgid;
    bpf_get_current_comm(&data.comm, sizeof(data.comm));
    data.type = EVENT_ARG;
    __submit_arg(ctx, (void *)filename, &data);
    // skip first arg, as we submitted filename
    #pragma unroll
    for (int i = 1; i < MAX_ARGS; i++) {
        if (submit_arg(ctx, (void *)&__argv[i], &data) == 0)
             goto out;
    }
    // handle truncated argument list
    char ellipsis[] = "...";
    __submit_arg(ctx, (void *)ellipsis, &data);
out:
    return 0;
}
```



<br>

这里需要注意的是在上面的例子中，argv 是一个用户空间的字符串数组（指针数组），所以就需要调用 bpf_probe_read 系列的辅助函数，去这些指针中读取数据，并且，字符串的数量（即参数的个数）和每个字符串的长度（即每个参数的长度）都是未知的，由于 eBPF 的栈只有 512 字节，所以只能读取有限的字符，这里都定义了  MAX_ARGS 个参数读取。



3. eBPF 程序就介绍这么多，下来是 前端处理程序，可以使用 [python/bcc](https://github.com/feiskyer/ebpf-apps/blob/main/bcc-apps/python/execsnoop.py) 也可以使用 [go/ebpf](https://github.com/iovisor/gobpf/blob/master/examples/bcc/execsnoop/execsnoop.go) 。已经到了最拿手的阶段了，Go 、Python 的代码非常简单，这里就不做过多的介绍了。



<br>



**但是问题是如何将我们写的 eBPF 程序分发到线上生产环境去执行**？

方法一：容器，将BCC和开发工具都安装到容器中，容器本身不对外提供服务，可以降低安全风险。

方法二：参考内核的方法，每个版本开发一个匹配当前内核版本的 [eBPF 程序](https://elixir.bootlin.com/linux/v5.13/source/samples/bpf)，并编译为字节码，分发到生成环境。

<br>

## 使用 libbpf 方法进行跟踪

在eBPF 程序中，由于 libbpf 方法内核已经支持了 BTF，不再需要引入众多的内核头文件来获取内核数据结构的定义，取而代之的是 bpftool 生成的 **vmlinux.h** 头文件，其中包含内核数据结构的定义。



使用 libbpf 开发 eBPF 程序通过以下四个步骤完成：

1. 使用 bpftool 生成内核数据结构定义头文件，BTF 开启后，找到 /sys/kernel/btf/vmlinux 这个文件，这个文件正是内核头文件。
2. 开发 eBPF 程序，为了方便后续统一通过 Makefile 编译， eBPF 程序的源码文件一般命名为 <程序名>.bpf.c 。
3. 编译 eBPF 程序为字节码，使用 bpftool gen skeleton 为 eBPF 字节码生成脚手架头文件（Skeleton Header）。这个头文件包含了 eBPF 字节码以及相关的加载、挂载和卸载函数，可在用户态程序中直接调用。
4. 最后通过用户态程序引入上一步生成的头文件，开发用户态程序。



上面说了，可以创建一个 Makefile 简化执行。

```makefile

APPS = execsnoop

.PHONY: all
all: $(APPS)

$(APPS):
    clang -g -O2 -target bpf -D__TARGET_ARCH_x86_64 -I/usr/include/x86_64-linux-gnu -I. -c $@.bpf.c -o $@.bpf.o
    bpftool gen skeleton $@.bpf.o > $@.skel.h
    clang -g -O2 -Wall -I . -c $@.c -o $@.o
    clang -Wall -O2 -g $@.o -static -lbpf -lelf -lz -o $@

vmlinux:
    $(bpftool) btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

有了这个 Makefile 之后，你执行  make vmlinux  命令就可以生成  vmlinux.h  文件，再执行  make  就可以编译  APPS  里面配置的所有 eBPF 程序（多个程序之间以空格分隔）。

<br>

Golang 的 [cilium](https://github.com/cilium/ebpf/) 框架便是经典的使用 libbpf 的例子。

libbpf 的 eBPF 程序开发包括定义哈希映射、性能事件映射以及跟踪点的处理函数等，都使用 SEC() 宏定义完成，在编译时，**通过 SEC() 宏定义的数据结构和函数会放到特定的 ELF 段中，这样后续加载 BPF 字节码时，就可以从这些段中获取所需的元数据**。



比如

``` c

// 包含头文件
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>

// 定义进程基本信息数据结构
struct event {
    char comm[TASK_COMM_LEN];
    pid_t pid;
    int retval;
    int args_count;
    unsigned int args_size;
    char args[FULL_MAX_ARGS_ARR];
};

// 定义哈希映射
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10240);
    __type(key, pid_t);
    __type(value, struct event);
} execs SEC(".maps");

// 定义性能事件映射
struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));
} events SEC(".maps");

// sys_enter_execve跟踪点
SEC("tracepoint/syscalls/sys_enter_execve")
int tracepoint__syscalls__sys_enter_execve(struct trace_event_raw_sys_enter *ctx)
{
  // 待实现处理逻辑
}

// sys_exit_execve跟踪点
SEC("tracepoint/syscalls/sys_exit_execve")
int tracepoint__syscalls__sys_exit_execve(struct trace_event_raw_sys_exit *ctx)
{
  // 待实现处理逻辑
}

// 定义许可证（前述的BCC默认使用GPL）
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```



至于处理逻辑上 BCC 程序基本上是相同的，不过，详细对比一下，他们还有两个最大的不同点。

- 第一：函数名的定义格式不同，BCC 程序使用的是 TRACEPOINT_PROBE 宏，而 libbpf 程序用的是 SEC 宏。
- 第二：映射方法的访问方法不同，BCC 封装了很多更易用的映射访问函数（如 tasks.lookup()），而 libbpf 程序则需要 BPF 辅助函数（比如查询要使用 bpf_map_lookup_elem()）。

<br>

写完 eBPF 程序后，需要用 clang 和 bpftool 将其编译成 BPF 字节码，然后再生成其脚手架头文件。像 cilium 项目中提供了 bpf2go 的小程序可以帮助我们完成这个步骤。



不过似乎没有特别多的使用 libbpf 的例子，这里就看[这个](https://github.com/iovisor/gobpf/blob/master/examples/bcc/execsnoop/execsnoop.go)吧



# 用户态跟踪

用户态空间对应内核态跟踪使用的 kprobe 和 tracepoint 有 **uprobe** 和 **用户空间定义的静态跟踪点**（User Statically Defined Tracing，简称 USDT）。



和内核一个问题，要想进行跟踪需要先找到跟踪点，一般来说，我们找跟踪点是从二进制文件中查找。在有调试信息的情况下，就可以通过 [readdlf](https://man7.org/linux/man-pages/man1/readelf.1.html) 、[objdump](https://man7.org/linux/man-pages/man1/objdump.1.html) 、[nm](https://man7.org/linux/man-pages/man1/nm.1.html) 等工具查询可用于跟踪的函数、变量等符号列表。比如查询加解密的 [openssl 动态库](https://cppget.org/libssl)

```bash

# 查询符号表（RHEL8系统中请把动态库路径替换为/usr/lib64/libssl.so）
readelf -Ws /usr/lib/x86_64-linux-gnu/libssl.so

# 查询USDT信息（USDT信息位于ELF文件的notes段）
readelf -n /usr/lib/x86_64-linux-gnu/libssl.so
```

或者使用 bpftrace 工具。

```bash

# 查询uprobe（RHEL8系统中请把动态库路径替换为/usr/lib64/libssl.so）
bpftrace -l 'uprobe:/usr/lib/x86_64-linux-gnu/libssl.so:*'

# 查询USDT
bpftrace -l 'usdt:/usr/lib/x86_64-linux-gnu/libssl.so:*'
```



## 编程语言对追踪的影响

常用的编程语言按照运行原理，大致可以分为三类;

- 第一类是 C、C++、Golang 等编译为机器码后再执行的**编译型语言**。这类语言通常会被编译成 ELF 格式的二进制文件，包含了保存在寄存器或栈中的函数参数和返回值，因此可以直接通过二进制文件中的符号进行跟踪。
- 第二类是 Python、Bash、Ruby 等通过解释器语法分析之后的**解释型语言**。这类编程语言开发的程序，无法直接从语言运行时的二进制文件中获取应用程序的调试信息，通常需要跟踪解释器的函数，再从其参数中获取应用程序的运行细节。
- 第三类是 Java、.Net、JavaScript 等先编译为字节码，再有即时编译器（JIT）编译为机器码执行的**即时编译型语言**。同解释型语言类似，这类编程语言无法直接从语言运行的二进制文件中获取应用程序的调试信息，跟踪 JIT 编程语言开发的程序最为困难，因为 JIT 编译的状态只存在于内存中。通常需要一个 [map-agent](https://github.com/jvm-profiling-tools/perf-map-agent) 的东西



## 跟踪编译型语言的应用程序

大部分编译型语言会遵循 [ABI（Application Binary Interface）](https://juejin.cn/post/6894179449996312589)调用规范，函数的参数和返回值都存放在寄存器中。**首先需要区分编程语言的调用规范**，然后在从寄存器或堆栈中读取函数的参数和返回值。

{{< admonition type=info title="Golang" open=true >}}
Go 1.17 之前使用 [Plan9](https://9p.io/sys/doc/asm.html) 调用规范，函数的参数和返回值都存放在堆栈中，1.17之后才切换到 ABI 调用规范。
{{< /admonition >}}

此外，**调试信息并非一定要内置在最终分发的应用程序二进制文件中，它们也可以放到独立的调试文件存储**。为了减少二进制文件的大小，通常会把调试信息从二进制文件中剥离，比如每次编译和发布程序之前，会执行 strip 命令，把调试信息删除。



假设我们要跟踪 Bash 这个C语言程序，就需要在跟踪之前，安装调试信息。安装失败参考 [这篇文章](https://cloud.tencent.com/developer/article/1637887)

```
# Ubuntu
sudo apt install bash-dbgsym

# RHEL
sudo debuginfo-install bash
```

有了 Bash 调试信息之后，再执行下面的步骤，查询Bash的符号表：

```bash
# 第一步，查询 Build ID （用于关联调试信息）
readelf -n /usr/bin/bash | grep "Build ID"
    Build ID: 33a5554034feb2af38e8c75872058883b2988bc5

# 第二步，找到调试信息对应的文件（调试信息位于目录 /usr/lib/debug/.build-id中，ID是上一步查到的）
s /usr/lib/debug/.build-id/33/a5554034feb2af38e8c75872058883b2988bc5.debug

# 第三步，查询调试符号表
readelf -Ws /usr/lib/debug/.build-id/33/a5554034feb2af38e8c75872058883b2988bc5.debug
```



参考 bash 命令的[源代码](https://git.savannah.gnu.org/cgit/bash.git/tree/lib/readline/readline.c?h=bash-5.1#n352)，每条Bash命令执行之前都会调用 `char readline(const char prompt)` 函数读取用户的输入。然后再去解析执行。



bpftrace、BCC 以及 libbpf 等工具均支持 uretprobe，可以用下面的方法追踪。

```
sudo bpftrace -e 'uretprobe:/usr/bin/bash:readline { printf("User %d executed \"%s\" command\n", uid, str(retval)); }'
Attaching 1 probe...
User 0 executed "ls" command
User 0 executed "ping qq.com" command

```

> 注：uid 和 retval 是两个内置变量。



如果想要获取返回值，BCC复杂一些，对于BCC的 uretprobe 来说，其官方文档 [uretprobes](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#5-uretprobes)。根据这里的文档。返回值可以通过宏 `RT_REGE_RC(ctx)` 获取(kprobe 其实也一样)。

```c

// 包含头文件
#include <uapi/linux/ptrace.h>

// 定义数据结构和性能事件映射
struct data_t {
    u32 uid;
    char command[64];
};
BPF_PERF_OUTPUT(events);

// 定义uretprobe处理函数
int bash_readline(struct pt_regs *ctx)
{
    // 查询uid
    struct data_t data = { };
    data.uid = bpf_get_current_uid_gid();

    // 从PT_REGS_RC(ctx)读取返回值
    bpf_probe_read_user(&data.command, sizeof(data.command), (void *)PT_REGS_RC(ctx));

    // 提交性能事件
    events.perf_submit(ctx, &data, sizeof(data));
    return 0;
}
```



## 跟踪解释型语言应用程序

对于各种解释型编程语言的二进制文件（如 Python、PHP等）可以使用类似编译型语言的查询方法。



``` bash
sudo bpftrace -l '*:/usr/bin/python3:*'
```



一般我们只能跟踪函数调用，会使用 `function__entry` 和 `function_return` 两个函数。具体就不细说了。而且需要为Python开启USDT跟踪点（编译选项为 --with-dtrace），才可以查询到具体的 USDT 跟踪点。

毕竟Python用的不多，这里有[一篇文章](https://ish-ar.io/python-ebpf-tracing/)可以参考。



## 跟踪即使编译型语言应用程序

这类语言的代表 Java、.Net 等都是即使编译型语言。以Java为例，Java虚拟机（JVM）除了执行一些常规的JIT即使编译之外，还会在执行过程中对运行流程进行剖析和优化。也加大了跟踪的难度。

并且要找出即时编译器的跟踪点同应用程序的运行关系，需要对编程语言的底层运行原理十分熟悉才可以，这也是跟踪即使编译型语言最难的一块。



我不是 Java 用户，但是原理差不太多，可以参考[这边文章](https://wenfeng-gao.github.io/post/profile-java-program-with-bcc-tool/#setup-perf-agent-container)。



最后一个小提示，Android 等基于 Linux 的操作系统也可以做跟踪，参考

- [eBPF/BCC - A better low-level instrumentation tool on Android](https://blog.senyuuri.info/2021/06/30/ebpf-bcc-android-instrumentation/)

- [Android 使用eBPF扩展内核](https://source.android.google.cn/devices/architecture/kernel/bpf?hl=zh-cn)