---
title: 从Linux内存管理去理解游戏是如何作弊
author: kiosk
tags:
  - golang
categories:
  - programming
date: 2025-10-19 00:17:00
math: true
---

最近的 [战地6](https://www.ea.com/zh-hans/games/battlefield/battlefield-6) 发布了，作为一款FPS游戏，是出了名的外挂多， 这才发布几天又变成了天神乱斗，这不怪EA菜，毕竟 TX 的 ACE 保护下的三角洲也遭不住挂B满天飞。这是 FPS 游戏技术架构的 “先天缺陷”。

![battlefield-6](https://img1.kiosk007.top/static/images/blog/20251017233211-battlefield-6.png)

为了提供流畅、无延迟的游戏体验，FPS 游戏通常采用 “**客户端预测（Client Prediction）**” 和 “**客户端信任（Client-Side Trust）**” 的架构。这意味着：

- 可见性数据： 客户端（玩家的电脑）必须提前知道和渲染地图上所有可见对象的信息，包括你的队友、你的敌人、各种道具的精确坐标、血量、装备等。如果客户端不知道敌人的位置，游戏就无法在敌人出现的一瞬间立即渲染出来，会造成延迟。

  > ➡️ 结果： 这一数据漏洞直接催生了 透视（Wallhack/ESP - Extra Sensory Perception） 外挂。外挂程序只需从本地内存中读取这些合法的“敌人坐标”数据，然后绘制到屏幕上，作弊者就能透视障碍物看到敌人。

- 物理与射击计算： 在一些 FPS 游戏中，虽然最终的伤害计算会由服务器验证，但瞄准、弹道落点、后坐力等输入细节，最初都源于客户端的输入和本地计算。

  > ➡️ 结果： 这为 自瞄（Aimbot） 提供了可乘之机。外挂程序通过读取内存中的敌人位置，然后直接劫持鼠标输入，向瞄准目标发送精确的、看起来合法的鼠标移动指令。


<!--more-->

在之前的 [什么？DMA竟然可以作弊
](https://kiosk007.top/post/dma%E7%AB%9F%E7%84%B6%E5%8F%AF%E4%BB%A5%E8%BF%99%E4%B9%88%E7%8E%A9/)文章中, 介绍过一种基于外部PCIe的 DMA 设备做外部内存直接访问进而实现作弊的一个方式。但是总有点觉得这个方式对于我来说不直观，毕竟手上没有 PCIE 设备，想了解一下不是那么先进的游戏作弊方案。

OK，写这篇博客就是为了从 Linux 内存的角度探索一下普通的游戏作弊方法。



# 一. 从一个简易游戏开始

如下是一个简易版的 game，和所有的 FPS 一样，这里定义了 血条（`player_health`），假设一直在随机掉血，当血量小于0则游戏结束。

代码在  https://github.com/kiosk404/linux-mem-hack/blob/master/game_demo.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <time.h>

// 这是一个全局变量，在 Data/BSS 段存储
// 作弊者会寻找并修改它
volatile int player_health = 100;
volatile int damage_counter = 0; // 用于增加血量变化的复杂度

void take_damage() {
    // 模拟受到伤害
    if (player_health > 0) {
        player_health -= (rand() % 5 + 1); // 随机减少 1 到 5 点血
        damage_counter++;
    }
}

void heal() {
    // 偶尔进行少量恢复，防止游戏进程立刻结束
    if (player_health < 100) {
        player_health += (rand() % 3);
        if (player_health > 100) {
            player_health = 100;
        }
    }
}

int main() {
    // 设置随机数种子
    srand(time(NULL));

    // 获取并打印当前进程ID (PID)，这是作弊工具附加进程的关键
    pid_t pid = getpid();
    printf("--- 简易游戏Demo (PID: %d) ---\n", pid);

    // 打印血量变量的内存地址，这是作弊者定位的起点
    printf("玩家血量 (player_health) 地址: %p\n", &player_health);
    printf("--------------------------------\n");

    // 主循环：模拟游戏运行
    while (player_health > 0) {
        
        take_damage();
        
        // 每 10 次伤害，稍微恢复一些
        if (damage_counter % 10 == 0) {
            heal();
        }
        
        printf("当前血量: %d\n", player_health);
        
        // 睡眠一秒，模拟游戏运行速率
        sleep(1);
    }

    printf("\n游戏结束 (Game Over)! 玩家血量降为 0。\n");
    return 0;
}
```

运行效果如下：

```bash
# gcc -g game_demo.c -o game

# ./game 
--- 简易游戏Demo (PID: 34914) ---
玩家血量 (player_health) 地址: 0x56ddbe332010
--------------------------------
当前血量: 97
当前血量: 96
当前血量: 93
...
当前血量: 4
当前血量: 1
当前血量: -1

游戏结束 (Game Over)! 玩家血量降为 0。
```



上述游戏为了简单，将 “血条” 放到了全局变量，在一款真正的、现代的 3A 级或大型网络游戏中，角色的血量等核心参数，**极少会直接放在全局变量（即数据段/BSS段）里。** 它们绝大多数都在堆上，作为动态分配的对象的一部分。



**实例化对象：** 角色、怪物、道具等都是在程序运行时动态创建的“实体”（或对象）。这些对象需要动态分配内存，因此它们被放在堆上。

**运行时创建/销毁：** 怪物生成、角色加入/退出，都需要灵活地分配和释放内存，堆是为此设计的。

**数据封装：** 血量通常是 `Player` 对象或 `Entity` 结构体中的一个成员变量，与角色的位置、ID、状态等其他数据紧密相连。

<br/>

# 二. Linux内存管理基础

我们先来看第一个知识点 - 内存布局

<p align="center">
  <img src="https://img1.kiosk007.top/static/images/blog/20251019131912-mem-space.webp" alt="mem-space" alt="mem-space" width="60%" />
</p>




## 用户态内存布局

这里我们可以看到 `0x588eb18f6010` 这个地址，这是一个**虚拟地址 (Virtual Address)**。真正的物理地址也就是 RAM 芯片上的地址，只有 **内核** 和 **MMU** 知道。

- **进程的视角：** 任何用户空间程序（包括 `game_demo`）在运行时，它看到和操作的所有内存地址都是**虚拟地址**。这是操作系统的**虚拟内存管理系统**提供给它的抽象层。

- **操作系统的保护：** 操作系统通过虚拟内存机制，为每个进程提供了一个私有的、独立的地址空间。进程只能访问它自己的虚拟地址空间。它**无法直接知道或访问**物理内存地址。

在现代 Linux 系统中，默认启用了 **ASLR (Address Space Layout Randomization)**，即地址空间布局随机化。也就是说，每当游戏运行的时候，其内存地址都是不一样的，ASLR 的主要目的是提高系统安全性，使利用内存漏洞变得困难。

下面是查看的是一个进程的 `/proc/<pid>/maps` 文件内容，这是 Linux 进程分析和作弊（内存黑客）的**起点**。

```bash
$ root@instance-vm00:/proc/34914# cat maps
56ddbe32e000-56ddbe32f000 r--p 00000000 fc:00 2359308                    /home/work/game/game
56ddbe32f000-56ddbe330000 r-xp 00001000 fc:00 2359308                    /home/work/game/game
56ddbe330000-56ddbe331000 r--p 00002000 fc:00 2359308                    /home/work/game/game
56ddbe331000-56ddbe332000 r--p 00002000 fc:00 2359308                    /home/work/game/game
56ddbe332000-56ddbe333000 rw-p 00003000 fc:00 2359308                    /home/work/game/game
56ddd00f6000-56ddd0117000 rw-p 00000000 00:00 0                          [heap]
77f690800000-77f690828000 r--p 00000000 fc:00 2505739                    /usr/lib/x86_64-linux-gnu/libc.so.6
77f690828000-77f6909b0000 r-xp 00028000 fc:00 2505739                    /usr/lib/x86_64-linux-gnu/libc.so.6
77f6909b0000-77f6909ff000 r--p 001b0000 fc:00 2505739                    /usr/lib/x86_64-linux-gnu/libc.so.6
77f6909ff000-77f690a03000 r--p 001fe000 fc:00 2505739                    /usr/lib/x86_64-linux-gnu/libc.so.6
77f690a03000-77f690a05000 rw-p 00202000 fc:00 2505739                    /usr/lib/x86_64-linux-gnu/libc.so.6
77f690a05000-77f690a12000 rw-p 00000000 00:00 0 
77f690c03000-77f690c06000 rw-p 00000000 00:00 0 
77f690c0b000-77f690c0d000 rw-p 00000000 00:00 0 
77f690c0d000-77f690c0e000 r--p 00000000 fc:00 2505736                    /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
77f690c0e000-77f690c39000 r-xp 00001000 fc:00 2505736                    /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
77f690c39000-77f690c43000 r--p 0002c000 fc:00 2505736                    /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
77f690c43000-77f690c45000 r--p 00036000 fc:00 2505736                    /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
77f690c45000-77f690c47000 rw-p 00038000 fc:00 2505736                    /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
7ffcaf714000-7ffcaf735000 rw-p 00000000 00:00 0                          [stack]
7ffcaf7db000-7ffcaf7df000 r--p 00000000 00:00 0                          [vvar]
7ffcaf7df000-7ffcaf7e1000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]

```

这个文件列出了进程 **34914** 的**虚拟内存区域映射（VMA - Virtual Memory Area）**。

### 2.1 可执行文件本身（game）



这是 `game_demo` 程序自身的代码和全局数据。

| **地址范围**                    | **权限**      | **含义**                                                     |
| ------------------------------- | ------------- | ------------------------------------------------------------ |
| `56ddbe32e000-56ddbe330000`     | `r--p` `r-xp` | **代码段（Text Segment）**：存储程序的机器指令。`r-xp` 表示可读、可执行、不可写入（防止代码被修改）。 |
| `56ddbe330000-56ddbe331000`     | `r--p`        | **只读数据段（Read-Only Data）**：存储常量字符串等只读数据。 |
| **`56ddbe332000-56ddbe333000`** | **`rw-p`**    | **数据段（Data/BSS Segment）**：存储**可读写**的全局变量和静态变量。 |

全局变量 `player_health` 必须存储在最后一个区域：**`56ddbe332000-56ddbe333000` (rw-p)**。任何可以被修改的游戏数值（如血量、分数）都必须位于 `rw-p`（读写）权限的区域。





### 2.2 运行时动态内存



这些区域是程序运行时动态分配的内存。

| **地址范围**                | **权限** | **区域名**    | **含义**                                                     |
| --------------------------- | -------- | ------------- | ------------------------------------------------------------ |
| `56ddd00f6000-56ddd0117000` | `rw-p`   | **`[heap]`**  | **堆**。通过 `malloc()` 或 `new` 动态分配的内存都位于此。大型游戏对象（如敌人、道具实例）通常在这里。 |
| `7ffcaf714000-7ffcaf735000` | `rw-p`   | **`[stack]`** | **栈**。存储函数调用、局部变量等。                           |





### 2.3 共享库（依赖项）

这是程序运行所需的外部代码和数据。

| **地址范围**       | **权限** | **映射文件**                        | **含义**                                                     |
| ------------------ | -------- | ----------------------------------- | ------------------------------------------------------------ |
| `77f690800000-...` | 各种权限 | `/usr/lib/.../libc.so.6`            | **C 标准库**。包含 `printf()`, `sleep()`, `rand()` 等函数和数据。 |
| `77f690c0d000-...` | 各种权限 | `/usr/lib/.../ld-linux-x86-64.so.2` | **动态链接器**。负责加载和链接所有共享库。                   |



**作弊意义：** 很多高级外挂会通过**注入代码**到这些共享库中，或者通过 Hook **共享库中的函数**（例如 Hook `libc.so.6` 中的 `sleep()` 或网络函数）来实现作弊和隐藏。



### 2.4 内核/系统接口区域

这些区域是操作系统内核提供的特殊机制。

| **地址范围**           | **权限** | **区域名**   | **含义**                                                     |
| ---------------------- | -------- | ------------ | ------------------------------------------------------------ |
| `7ffcaf7db000-...`     | `r--p`   | `[vvar]`     | **虚拟动态共享对象变量**。内核提供的一些只读变量。           |
| `7ffcaf7df000-...`     | `r-xp`   | `[vdso]`     | **虚拟动态共享对象**。内核提供的一些系统调用（如 `time()`）的快速执行代码，在用户态执行，无需陷入内核。 |
| `ffffffffff600000-...` | `--xp`   | `[vsyscall]` | **虚拟系统调用**。旧版内核机制，现在主要由 `[vdso]` 取代。   |



## 程序编译

在我写好这个 `game_demo` 后执行 `gcc -g game_demo.c -o game`  , 其本质上生成的文件 `game` **是一个 ELF 格式的可执行文件** 。

在 Linux 系统上，`gcc`（GNU Compiler Collection）作为标准的 C/C++ 编译器，其最终输出的可执行文件、目标文件（`.o`）和共享库（`.so`）**全部采用 ELF 格式**。而本例中，为了简单，是直接编译成最终的可执行文件。

- **`-g` 选项的含义：** `game` 是 ELF 文件，而 `-g` 选项的作用是告诉编译器。“请将调试信息（Debugging Information）嵌入到 ELF 文件中。” 调试信息通常以一个或多个特殊的 ELF **段（Section）** 的形式（例如 `.debug_info`, `.debug_line`, `.debug_str` 等）存储在文件中。
- 这些信息（如源代码行号与机器指令的对应关系、变量名称与内存地址的对应关系）是供 GDB 这样的**调试器**使用的。



程序编译后的二进制文件（ELF 格式）被映射到进程的虚拟内存空间，这个过程主要由 **操作系统内核（Kernel）** 中的 **程序加载器（Program Loader）** 负责，它遵循 ELF 文件的 **执行视图（Execution View）** 来完成映射。



# 三. 进程间内存访问技术

## 3.1 ptrace系统调用

经常搞开发的时候肯定需要经常 实现断点（breakpoint debugging）、单步执行，这其实本质上是 一个进程侵入了另一个进程的执行过程。

`ptrace` (process trace) 是 Linux (及类 Unix 系统) 中一个核心的系统调用，它允许一个进程（**Tracer/跟踪者**，例如外挂程序）观察和控制另一个进程（**Tracee/被跟踪者**，例如 `game_demo`），并检查和修改其内部状态。

它最初是为了实现 GDB 这样的**调试器**而设计的，但也是进行进程内存注入和作弊的主要技术之一。

```c
# `ptrace` 函数原型
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
```



| **Request 参数**      | **作用**                                           | **对应操作**                                                 | **目标**               |
| --------------------- | -------------------------------------------------- | ------------------------------------------------------------ | ---------------------- |
| **`PTRACE_ATTACH`**   | 附加到目标进程。                                   | 建立跟踪关系，目标进程暂停。                                 | 调试的起点。           |
| **`PTRACE_DETACH`**   | 从目标进程分离。                                   | 解除跟踪关系，目标进程继续运行。                             | 调试的终点。           |
| **`PTRACE_CONT`**     | 让目标进程继续运行。                               | 进程继续执行，直到下一个信号或系统调用被捕获。               | 恢复进程。             |
| **`PTRACE_PEEKDATA`** | **读取**目标进程内存。                             | 从目标进程的虚拟地址 `addr` 读取一个 **word** (通常是 4 或 8 字节) 的数据。 | 内存查看。             |
| **`PTRACE_POKEDATA`** | **写入**目标进程内存。                             | 将 `data` 的值写入到目标进程的虚拟地址 `addr`。              | **实现锁血挂的关键。** |
| `PTRACE_SYSCALL`      | 继续运行，但在下一个系统调用**进入或退出**时停止。 | 用于实现 `strace`（系统调用跟踪）和更精细的作弊控制。        | 系统调用监控。         |

<br/>

#### ptrace 实现进程间内存访问的原理


`PTRACE_PEEKDATA` 和 `PTRACE_POKEDATA` 是实现进程间内存访问（跨进程读写）的核心：

1. **权限与上下文切换：** 当跟踪者调用 `ptrace()` 时，内核会验证其权限（例如，通常需要是父进程或 Root 权限）。一旦验证通过，内核就会在**目标进程的上下文**中执行内存操作。
2. **虚拟地址翻译：** 传递给 `addr` 的地址是目标进程的**虚拟地址**。内核知道目标进程的页表信息，它会利用这些页表将你提供的虚拟地址翻译成**物理地址**。
3. **直接读写：** 内核直接操作物理内存，将数据从物理内存复制到跟踪者内存（`PEEKDATA`）或将跟踪者提供的数据写入到物理内存（`POKEDATA`）。
4. **原子性保证：** **关键在于同步。** 如我们讨论的，在进行 `POKEDATA` 操作时，目标进程必须处于**停止状态**（通过 `ATTACH` 或 `waitpid` 确认）。这保证了数据写入时的原子性，防止被写入的数据在操作过程中被目标进程自己修改。



下面是一个基于 ptrace 的修改上述简单游戏血条的代码。

代码在：https://github.com/kiosk404/linux-mem-hack/blob/master/hacker_trainer_ptrace.c

```go
    ...
	// 附加进程
    if (ptrace(PTRACE_ATTACH, target_pid, NULL, NULL) == -1) {
        perror("附加失败");
        return 1;
    }

    waitpid(target_pid, NULL, 0);

    // 读取原数据（保留其他字节）
    long original_data = ptrace(PTRACE_PEEKDATA, target_pid, (void*)target_addr, NULL);

    printf("当前血量: %ld\n", original_data);

    // 只修改低4字节为100
    long data_to_write = LOCKED_HEALTH;

    // 写入数据
    if (ptrace(PTRACE_POKEDATA, target_pid, (void*)target_addr, (void*)data_to_write) == -1) {
        perror("写入失败");
        ptrace(PTRACE_DETACH, target_pid, NULL, NULL);
        return 1;
    }
	printf("血量已修改为 100\n");

    // 分离进程
    ptrace(PTRACE_DETACH, target_pid, NULL, NULL);
	...
```



因为我们知道游戏的 pid ，当前游戏也直接打印了 “血条“ 的虚拟内存地址，所以我们可以直接去实现锁血。

执行效果如下：

```bash
$ gcc -g hacker_trainer-ptrace.c -o hacker_trainer
    
$ hacker_trainer 34914 0x56ddbe332010
    
# 观察游戏的血条打印
当前血量: 32
当前血量: 29
当前血量: 28
当前血量: 26
当前血量: 25
当前血量: 98          --- 又突然变成从100递减了
当前血量: 97
当前血量: 96
```



## 3.2 mem 内存映射实现进程内存访问的原理

`/proc/<pid>/mem` 是 Linux **`/proc` 虚拟文件系统**中最具代表性和强大功能的接口之一

####  概念：虚拟内存的文件表示

`/proc` 是一个**虚拟文件系统**（Pseudo Filesystem），它并不存储在硬盘上，而是由 Linux 内核在运行时动态生成的。它提供了一种通过文件系统接口来访问内核数据结构和进程信息的方式。

- **`/proc/<pid>/mem`**：在这个虚拟文件系统中，`/proc/<pid>/mem` 文件代表了进程 ID 为 `<pid>` 的整个**虚拟地址空间**。
- **文件操作 = 内存操作：** 当对这个文件进行标准的文件 I/O 操作（`read()`, `write()`, `lseek()`）时，内核并没有去读写硬盘上的文件，而是将这些操作**重定向**到对目标进程内存的实际读写上。

#### 实现机制：文件 I/O 到内存访问的转换

`/proc/<pid>/mem` 的工作流程是一个**文件操作到内核内存操作的转换过程**：

- **A. 定位（`lseek`）**

当使用 `lseek()` 函数来移动 `/proc/<pid>/mem` 的文件指针时，这个操作被内核拦截。

内核操作： 内核将文件指针的位置（一个巨大的偏移量）直接解释为目标进程虚拟地址空间中的一个**虚拟地址**。

意义： 如果 `lseek` 到偏移量 `0x403048`，内核就知道接下来要操作目标进程虚拟内存中的 `0x403048` 地址。



- **B. 读取（`read`）**

当使用 `read()` 函数来读取文件时：

内核检查： 内核首先检查目标虚拟地址（由 `lseek` 设定）是否位于目标进程的有效内存区域（通过 `/proc/<pid>/maps` 映射确定）。它还会检查该区域是否有**读取权限**。

虚拟地址转换： 如果权限允许，内核会使用目标进程的**页表**，将当前的虚拟地址翻译成对应的**物理地址**。

数据拷贝： 内核直接从对应的物理内存位置读取数据，并将其**复制**到调用进程（外挂程序）提供的缓冲区中。



- **C. 写入（`write`）**

当使用 `write()` 函数来写入文件时（实现锁血的关键）：

内核检查： 内核首先检查目标虚拟地址是否有**写入权限**（即该内存区域的权限必须是 `rw-p`）。

虚拟地址转换： 转换为物理地址。

数据写入： 内核将调用进程（外挂程序）提供的新数据，直接写入到目标进程对应的物理内存位置。

<br/>

下图就是对 `game_demo` 进行锁血的一个小案例demo。

代码在：https://github.com/kiosk404/linux-mem-hack/blob/master/hacker_trainer_mem.c

```c
 	int fd = open(mem_path, O_RDWR);
    if (fd < 0)
    {
        printf("进程已退出\n");
        return 1;
    }

    // 读取血量
    int health;
    lseek(fd, addr, SEEK_SET);
    read(fd, &health, sizeof(int));
    printf("当前血量: %d\n", health);

    int new_health = RESTORE_HEALTH;
    lseek(fd, addr, SEEK_SET);
    write(fd, &new_health, sizeof(int));

```





## 3.3 进程间内存访问的技术汇总

| **技术名称** | **机制/系统调用**        | **核心原理**                                                 | **适用场景**                                                 |
| ------------ | ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **文件映射** | `mmap()` / `/dev/mem`    | 允许进程将文件（或特殊设备）映射到自己的虚拟地址空间。`/dev/mem` 甚至可以用于映射物理内存。 | 硬件级 DMA 外挂；驱动程序操作。                              |
| **内存文件** | **`/proc/<pid>/mem`**    | Linux 提供的虚拟文件，代表进程的整个虚拟内存。               | **外挂常用：** 外挂工具（如 `scanmem`）可以直接使用 `open()` 和 `read()` / `write()` 文件操作符来读写目标进程内存，无需复杂的 `ptrace` 状态管理。**（但需要 Root 权限或特定配置）** |
| **共享内存** | `shmget()`, `shmat()`    | 两个或多个进程显式地将同一块物理内存映射到各自的虚拟地址空间。 | **合法的跨进程通信**；在游戏作弊中可用于外挂和游戏程序之间的隐蔽通信（如果能找到方法将外挂代码注入游戏进程）。 |
| **DMA 攻击** | PCIe 总线 + 外部设备     | 利用外部设备（如特制硬件卡）通过 DMA 机制，**绕过 CPU 和操作系统**，直接访问和修改物理内存。 | **最高级、最难检测的硬件外挂。**                             |
| **代码注入** | `dlopen()`, `LD_PRELOAD` | 运行时将恶意 `.so` 动态链接库加载到目标进程的内存空间，让恶意代码在目标进程内部运行。 | 绕过权限检查，在游戏进程内部进行内存读写和函数 Hook。        |



# 四. 定位血条所在的内存地址

上面的例子，`game_demo` 直接将血条的内存地址直接打印了出来，这样外挂程序可以非常方便的对其进行修改，但实际上，游戏开发者可不会如此好心，这就是需要hacker 自己去查找定位游戏角色的血条到底在哪里。



当前我在编译阶段加上了 debug 调试信息。我可以使用 gdb 等命令去打印 `player_health` 的值。

```bash
$ file game
game: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=44b6b4d70b69c7a7063eb5347b06a6767beb3cba, for GNU/Linux 3.2.0, with debug_info, not stripped
```

这里可以看到输出有 `with debug_info, not stripped` 。

<br/>

- **gdb 调试**

现在就可以使用 gdb 指定 pid ，然后打印 player_health 的内存地址。

```bash
$ sudo gdb -q -p 1857
Attaching to process 1857
Reading symbols from /home/work/game/game...
Reading symbols from /lib/x86_64-linux-gnu/libc.so.6...
Reading symbols from /usr/lib/debug/.build-id/27/4eec488d230825a136fa9c4d85370fed7a0a5e.debug...
Reading symbols from /lib64/ld-linux-x86-64.so.2...
Reading symbols from /usr/lib/debug/.build-id/52/0e05878220fb2fc6d28ff46b63b3fd5d48e763.debug...
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
0x0000711284aeca7a in __GI___clock_nanosleep (clock_id=clock_id@entry=0, flags=flags@entry=0, req=req@entry=0x7ffea0fb7c70, rem=rem@entry=0x7ffea0fb7c70)
    at ../sysdeps/unix/sysv/linux/clock_nanosleep.c:78

warning: 78	../sysdeps/unix/sysv/linux/clock_nanosleep.c: No such file or directory
(gdb) p &player_health
$1 = (volatile int *) 0x5ce1ad628010 <player_health>
(gdb) p player_health
$2 = 93

```

<br/>

****

- **根据静态偏移量+基址**

我们使用 `readelf -s` 的输出，这是 ELF 文件的**符号表**信息确定静态偏移量，再加上从 /proc 下确定程序运行时加载基址（Base Address）

```bash
$ readelf -s ./game | grep player_health
    41: 0000000000004010     4 OBJECT  GLOBAL DEFAULT   25 player_health
$ sudo cat /proc/1857/maps |grep '/home/work/game/game'
5ce1ad624000-5ce1ad625000 r--p 00000000 fc:00 2359308                    /home/work/game/game
5ce1ad625000-5ce1ad626000 r-xp 00001000 fc:00 2359308                    /home/work/game/game
5ce1ad626000-5ce1ad627000 r--p 00002000 fc:00 2359308                    /home/work/game/game
5ce1ad627000-5ce1ad628000 r--p 00002000 fc:00 2359308                    /home/work/game/game
5ce1ad628000-5ce1ad629000 rw-p 00003000 fc:00 2359308                    /home/work/game/game
```

$$\mathbf{player\\_health 虚拟地址} = \mathbf{0x5ce1ad624000} + \mathbf{0x4010}$$


因此，在进程 ID 为 `1857` 的这次运行中，`player_health` 的具体虚拟内存地址是：

$$\mathbf{0x5ce1ad628010}$$



然而现实很残酷，没有哪个开发者会犯低级错误会把 debug_info 打包放到二进制里。这是一个典型的**真实世界逆向工程场景**！

在没有调试信息（无 `-g`）且程序不自曝地址的情况下，定位全局变量 `player_health` 的虚拟内存地址，您需要综合运用我们前面讨论的**内存扫描**和**逆向分析**技术。

<br/>

## 运行时定位（内存扫描）

PINCE（**P**INCE **I**s **N**ot **C**heat **E**ngine）是一个面向 Linux 游戏的**逆向工程前端工具**，旨在为 Linux 平台提供类似于 Windows 上 **Cheat Engine** 的功能。

它的核心理念是提供一个易于使用的图形界面，来简化对 **GDB（GNU Project Debugger）** 这个强大但复杂的命令行调试器的操作。

这次我们使用 [https://github.com/korcankaraokcu/PINCE](https://github.com/korcankaraokcu/PINCE) 来持续扫描

本次将游戏代码切换成 https://github.com/kiosk404/linux-mem-hack/blob/master/game_demo_sig.c ，不同的是这款游戏不会让血量持续减少，而是通过信号来控制减少。

运行代码：

```bash
./game                       
--- 简易游戏Demo (PID: 1669072) ---
玩家血量 (player_health) 地址: 0x58b5120eb010
--------------------------------
发送信号以造成伤害: kill -SIGUSR1 1669072
--------------------------------
当前血量: 100
当前血量: 100
当前血量: 100
...
```

因为当前的血量是100，attach 到进程之后，点击首次扫描。然后搜索值100.

<p align="center">
  <img src="https://img1.kiosk007.top/static/images/blog/20251019225404-mem-pc-game-hack-pince.png" alt="mem-space" alt="mem-space" width="60%" />
</p>

通过信号模拟修改用户角色的血量。`kill -SIGUSR1 1669072` 

```
>>> 受到攻击! 减少 7 点血量 <<<
当前血量: 93
当前血量: 93
当前血量: 93
当前血量: 93
```

此次可以观察到有2个地址的值变成了93，



<p align="center">
  <img src="https://img1.kiosk007.top/static/images/blog/20251019225717-mem-pc-game-hack-2.png" alt="mem-space" alt="mem-space" width="60%" />
</p>

实际上，外挂在开发之初也一般也是通过这样的方法找到到底血量是在哪里定义的。



<br/>



## 指针链

**真正的、稳定的外挂绝不能每次都依赖用户手动或程序自动进行耗时且不稳定的内存扫描。**



ASLR 是操作系统安全机制，每次进程启动时，内存布局都会随机化，防止固定地址攻击。游戏引擎（如 Unity 或 Unreal）也常用动态分配，地址不固定。 这导致每次重启游戏，血量地址都不同。如果你每次都手动扫描，就太麻烦了。所以即便每次都能通过修改血量推敲出当前进程的血量运行的虚拟内存地址。但仍然不是一款合格的 “外挂” 。



内存扫描（例如使用 PINCE）只是逆向工程的**第一步**，目的是获取当前这次运行的**内存快照**。外挂程序真正的核心能力在于找到数据的 **“不变性”** ——即 **指针链（Pointer Chain）**。



<br/>



**  真正的外挂如何实现持久化定位？**

外挂开发者不会依赖值扫描，而是通过以下高级技术自动计算地址。这些方法基于游戏的内部结构，一旦逆向出来，就可以硬编码到外挂程序中，实现“一次逆向，多次使用”：

<br/>

虽然血量本身的地址是变化的，但游戏程序内部访问血量的**代码逻辑**和**数据结构**是固定的。

- **固定偏移量 (Offset)：** 血量值作为 `Player` 结构体（Struct）的一个成员时，它距离该结构体起始地址的**偏移量（Offset）** 是固定不变的。
- **固定静态基址 (Static Base):** 无论游戏运行多少次，游戏引擎中总有一个**静态指针**（存储在 Data/BSS 段）指向某个关键的**对象管理器**或**主玩家结构体**。

真正的外挂定位血量地址，依赖于构建一个**指针链**，这个链条的起点是固定的，终点是变化的血量地址。


$$
\mathbf{\text{血量地址}} = \text{读取}(\text{基址} + \text{Offset}_{\text{A}}) + \text{Offset}_B + \text{Offset}_C
$$


**静态根源：** 首先通过内存扫描（如 PINCE）和反汇编分析，找到指针链的**起点**——一个位于 **Data/BSS 段的固定静态指针**。

**固定偏移：** 逆向工程师确定从这个静态指针到玩家对象、再到血量值的所有**固定偏移量** 。

**运行时计算：** 外挂程序在每次启动时，只做两件事：

- **获取 ASLR 基址：** 读取 `/proc/<pid>/maps` 找到游戏的当前加载基址。

- **计算静态指针地址：** $\text{Static Pointer Address} = \text{ASLR Base} + \text{Static Offset}$。

- **追踪指针：** 外挂程序使用 `ptrace_peekdata` 或 `/proc/mem` **读取内存**，一级一级地顺着链条读取指针值，最终计算出血量变量的**当前**虚拟地址。



在上面的例子中，我们需要读到 基址， 还需要偏移量。

基址的获取方法，可以从 /proc/{pid}/mem 中获取，或者在 PINCE 的 gdb 调试中获取。比如本次就获取到了 `0x00006343a6ad9000` ，而偏移，我们通过 反复修改血量确定是 `0x6343a6adc010` ，最终确定偏移是 `0x3010`

![pince-mem](https://img1.kiosk007.top/static/images/blog/20251020001155-pince-mem.png)





## 模拟 hack 一款游戏

这是一个模拟 真正游戏引擎那一层的实现方式，级成**更接近真实游戏**的结构—— 包括「世界（World）」「玩家（Player）」「组件（Component）」的内存关系。

https://github.com/kiosk404/linux-mem-hack/blob/master/game_demo_pointer.c



```c
// 假设的 game_demo_pointer.c 结构
typedef struct {
    int id;        // 0x00
    int max_mana;  // 0x04
    int health;    // 0x08  <-- 这就是我们要找的！
    // ... 
} Player;

Player *player; // 这是一个全局静态指针，指向堆上的Player结构体
```

由于 `Player *player` 是一个**全局静态指针**，它位于程序的 `.data` 段（静态区），因此它的**地址是固定不变的（相对基址）**。而 `health` 位于堆上的 `Player` 结构体内，所以它的**地址是变化的**。



我们的目标是找到这条**指针链**：

$$\mathbf{\text{血量地址}} = \text{读取}(\mathbf{\text{静态基址}} + \mathbf{\text{静态偏移A}}) + \mathbf{\text{固定偏移B}}$$

其中：

- **静态基址** + **静态偏移A** = 静态指针 `player` 的地址。
- **读取 $(\dots)$** = 静态指针 `player` 的值（即 `Player` 结构体的**堆地址**）。
- **固定偏移B** = `health` 在 `Player` 结构体中的**偏移量**（预估为 $0\text{x}8$）。

<br/>

运行效果如下：

虽然还是打印了内存地址，但是我们通过上述方法定位一下血条的偏移。

```bash
./game 
=== Single-player game (signal-driven) ===
PID: 2031220
g_world (global ptr) addr         : 0x63b4f67cd028
g_world value (GameWorld *)       : 0x63b4f9b192a0
g_world->player (Player *) addr   : 0x63b4f9b192a0
g_world->player value             : 0x63b4f9b192c0
player->health_comp (HealthComp*) : 0x63b4f9b192e0
player->health_comp value         : 0x63b4f9b192f0
player->health (int *)            : 0x63b4f9b192f0
------------------------------------------
Send SIGUSR1 to cause damage:  kill -USR1 2031220
Press Ctrl-C to exit

Player1    [########################################] 100/100   

```





#### 使用 PINCE 确定指针链的 4 个关键步骤

由于我们不能直接看到源码，必须通过动态调试来确定 `静态偏移A` 和 `固定偏移B`。

通过 ` kill -USR1 2031220` 模拟掉血

打开 PINCE 定位可以发现有2个地址出现了掉血。其实根据打印，我们已经知道了 就是 0x63b4f9b192f0 这个变量地址。 

<p align="center">
  <img src="https://img1.kiosk007.top/static/images/blog/20251020232139-linux-mem-hack.png" alt="mem-space" alt="mem-space" width="60%" />
</p>
将目标地址 `0x63b4f9b192f0` 移到下方，右键 点击 “Pointer scan for this address”.

<p align="center">
  <img src="https://img1.kiosk007.top/static/images/blog/20251022004959-pince-mem-scan.png" alt="mem-space" alt="mem-space" width="60%" />
</p>

点击scan后，等待扫描完成，这个对话框会消失，并且后面会产生一个文件 -- `game.scandata` ,

点击左上角的 文件[I], 打开刚才的文件。得到一个类似这样的文件，可以点击一下 `sort` 排序。game 的这些都是可疑的，这是最佳的基址模块，因为它直接指向程序内部的静态变量或全局指针。

<p align="center">
  <img src="https://img1.kiosk007.top/static/images/blog/20251022010306-pince-mem-pointer-scan.png" alt="mem-space" alt="mem-space" width="60%" />
</p>



那么接下来的事情就是反复试验了，看看哪个基址偏移是正确的。

所有 game[4] 候选指针：

```shell
层级  指针链                          评分  推荐度
====  ============================  ====  ======
4层   game[4]+10->F8->1298->2F0     ⭐⭐   备用
4层   game[4]+28->0->20->0          ⭐⭐⭐  良好
3层   game[4]+28->0->30             ⭐⭐⭐⭐⭐ 最佳
3层   game[4]+28->40->0             ⭐⭐⭐⭐⭐ 最佳
2层   game[4]+28->50                ⭐⭐⭐⭐  次优
4层   game[4]+8->0->20->50          ⭐⭐⭐  良好
4层   game[4]+8->20->0->30          ⭐⭐⭐  良好
4层   game[4]+8->20->40->0          ⭐⭐⭐  良好
3层   game[4]+8->20->50             ⭐⭐⭐⭐ 次优
```

偏移极小（0x28, 0x0, 0x30），很可能是结构体的直接成员。偏移太大都不像是结构体成员





不过我们需要最后再验证一下。

用这个 python 脚本 https://github.com/kiosk404/linux-mem-hack/blob/master/verify_pointer.py  验证指针链

```bash
[+] 游戏 PID: 2269458

[+] 找到 1 个可读写段:
  [1] 0x58c6f5f3b000-0x58c6f5f3c000 (rw-p)

============================================================
开始测试所有可能的基址...
============================================================

============================================================
测试段 [1]: 基址 0x58c6f5f3b000
============================================================

--- 测试指针组合 1: 0x28 -> 0x0 -> 0x30 ---

[*] 验证指针链: base=0x58c6f5f3b000, offsets=['0x28', '0x0', '0x30']
  [1] 读取地址: 0x58c6f5f3b028 (base+0x28)
      -> 指针值: 0x58c703b032a0
  [2] 读取地址: 0x58c703b032a0 (base+0x0)
      -> 指针值: 0x58c703b032c0
  [3] 最终地址: 0x58c703b032f0 (+0x30)
      ✅ 血量值: 99

🎉 找到有效指针！
   基址: 0x58c6f5f3b000 (段 1)
   偏移: 0x28 -> 0x0 -> 0x30
   血量地址: 0x58c703b032f0

```



<br/>

# 自动锁血外挂

好了，既然已经都可以根据指针链每次游戏启动都找到血条，然后进行锁血了。

完整的代码在 https://github.com/kiosk404/linux-mem-hack/blob/master/game_health_lock.py

核心代码如下，

```python
def resolve_pointer_chain(pid, base_address, offsets):
    """
    解析指针链
    base_address: 基址
    offsets: 偏移量列表，例如 [0x28, 0x0, 0x30]
    返回: 最终地址，如果失败返回 None
    """
    current_address = base_address
    
    # 遍历除最后一个之外的所有偏移（这些是指针）
    for i, offset in enumerate(offsets[:-1]):
        # 计算当前指针地址
        pointer_address = current_address + offset
        
        # 读取指针指向的地址
        next_address = read_memory_qword(pid, pointer_address)
        
        if next_address is None or next_address == 0:
            # print(f"[-] 指针链断裂在第 {i+1} 层 @ 0x{pointer_address:x}")
            return None
        
        # print(f"[DEBUG] 层{i+1}: 0x{pointer_address:x} -> 0x{next_address:x}")
        current_address = next_address
    
    # 最后一层是实际值的偏移
    final_address = current_address + offsets[-1]
    return final_address

def read_memory_dword(pid, address):
    """读取 4 字节 (int32)"""
    mem_path = f"/proc/{pid}/mem"
    try:
        with open(mem_path, 'rb') as f:
            f.seek(address)
            data = f.read(4)
            if len(data) == 4:
                return struct.unpack('I', data)[0]
    except Exception as e:
        # print(f"[-] 读取值失败 @ 0x{address:x}: {e}")
        pass
    return None
```



运行效果如下：

```shell
sudo python3 game_health_lock.py 
[sudo] kiosk 的密码： 
============================================================
      游戏血量锁定外挂 v1.0
============================================================
目标进程: game
指针链: game[4]+0x28->0x0->0x30
============================================================

[*] 正在搜索游戏进程...
[+] 找到进程: PID = 2387565

[*] 从 /proc/2387565/maps 读取内存布局...
[+] game[4] 基址: 0x5af95e194000

[*] 验证指针链...
[+] 指针链有效!
[+] 血量地址: 0x5af983ab32f0
[+] 当前血量: 92

[*] 开始监控血量...
[*] 目标血量: 100
[*] 检查间隔: 100ms
[*] 按 Ctrl+C 停止

------------------------------------------------------------
[11:17:37] 当前血量:   92 | 地址: 0x5af983ab32f0 -> 血量过低! 已锁定为 100 ✓ (第1次)
[11:17:37] 当前血量:  100 | 地址: 0x5af983ab32f0
[11:17:45] 当前血量:   97 | 地址: 0x5af983ab32f0 -> 血量过低! 已锁定为 100 ✓ (第2次)
[11:17:45] 当前血量:  100 | 地址: 0x5af983ab32f0
^C

```



至此，游戏锁血外挂就做好了。
