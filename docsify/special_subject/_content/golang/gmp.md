# GMP


G 是 gorouting、M 是系统级线程、P是调度器。

- `G (Goroutine)`: Go 语言中的轻量级并发执行单元。它是用户态的“线程”，由 Go 运行时环境管理。每个并发执行的函数都对应一个或多个 Goroutine。Goroutine 包含执行所需的指令、栈空间以及其他元数据。

- `M (Machine)`: 代表着一个操作系统线程。Go 运行时会创建并管理一组 M。M 负责执行 Goroutine。通常，M 的数量与 CPU 核心数相关，可以通过 GOMAXPROCS 来设置。

- `P (Processor)`: 逻辑处理器，它是 Goroutine 能够运行的必需的上下文。P 的数量也通常与 CPU 核心数相关，并且也由 GOMAXPROCS 控制。P 持有一个 Goroutine 队列（称为 Run Queue），以及一些其他的调度相关的状态。

**GMP 模型的工作原理**：

- `Goroutine 的创建`： 当你使用 go 关键字启动一个函数时，Go 运行时会创建一个新的 Goroutine (G) 并将其放入某个 P 的全局运行队列 (Global Run Queue)，或者更倾向于放入本地运行队列 (Local Run Queue)。

- `Processor 的调度`： 每个 P 都与一个或多个 M 绑定。P 的主要职责是从其本地运行队列中获取 Goroutine 并交给与其绑定的 M 执行。

- `Machine 的执行`： M 是真正执行 Goroutine 的实体。它会不断地从与其绑定的 P 的本地运行队列中取出 Goroutine (G)，然后执行 G 的代码。

- `本地运行队列 (Local Run Queue)`： 每个 P 都有自己的本地运行队列，用于存放待执行的 Goroutine。这有助于减少全局锁的竞争，提高调度效率。Go 调度器会尽量将新创建的 Goroutine 放入当前 P 的本地运行队列。

- `全局运行队列 (Global Run Queue)`: 当某个 P 的本地运行队列为空，或者为了实现负载均衡，P 会尝试从全局运行队列中获取 Goroutine 来执行。全局运行队列由所有的 P 共享，因此访问需要加锁。

- `工作窃取 (Work Stealing)`: 为了进一步提高调度效率和避免某些 P 上的 Goroutine 过多而其他 P 空闲的情况，当一个 P 的本地运行队列为空时，它会尝试从其他 P 的本地运行队列中“窃取”一部分 Goroutine 来执行。这种机制有助于更均匀地分配 Goroutine 到不同的 M 上。

- `系统监控 Goroutine (Sysmon)`: Go 运行时还有一个特殊的系统监控 Goroutine，它负责执行一些后台任务，例如：

回收长时间阻塞在系统调用中的 M。
周期性地检查和抢占长时间运行的 Goroutine，以防止某个 Goroutine 独占 CPU 资源。
执行垃圾回收的相关工作。



程序刚启动时，P会绑定M0，G被放入P的本地队列中执行，当P的本地队列满了，G可能会被放到一个全局队列中。

线程M想要运行G，就必须先和一个P进行关联
P 获取本地队列的G，如果获取不到就从全局队列或者其他的 MP 中偷取。
G 运行结束后，M会从P中获取下一个 G，如此往复。
大致流程如上，还有一些细节，比如 M 与 P 是如何绑定的？G 阻塞了怎么办？M与P的关系是什么？可以具体再说。

详见：https://kiosk007.top/post/golang-gmp/

调度器的设计策略（基本就是为了复用）：

work stealing 机制：当本线程无可以运行的G时，回去其他线程绑定的P中偷取G。而不是直接销毁线程。
hand off 机制：当本线程上的G运行系统调用阻塞时，线程会释放绑定的P，把 P 转移给其他的空闲的线程。而阻塞的协程会留在线程之上。