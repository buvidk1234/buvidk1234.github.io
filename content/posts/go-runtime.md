+++
date = '2026-07-04T02:44:43+08:00'
draft = false
title = 'Go Runtime'
tags = ['Go']
+++

# Go 运行时

## Part 1 Runtime基础

### 01 Runtime是什么

Go runtime 是 Go 程序运行时依赖的一组基础设施。它不是 JVM 那种执行字节码的虚拟机，也不是一个需要单独安装和启动的外部进程。普通 Go 程序在编译链接之后，最终生成的是机器码可执行文件；runtime 会作为程序的一部分被链接进去，在程序启动、并发调度、内存分配、垃圾回收、栈管理、阻塞唤醒、网络轮询、定时器、panic/recover 等方面提供支持。

可以先把 Go 程序理解成两部分：

```text
用户代码
  main.main
  业务函数
  package init
  普通 goroutine

运行时
  程序启动
  goroutine 调度
  栈增长
  内存分配
  GC
  channel/select
  timer
  netpoll
  syscall/signal
  panic/recover
```

这两部分最终在同一个进程里执行。用户代码看起来像是在直接调用函数、创建 goroutine、分配对象、读写 channel，但很多操作背后都会落到 runtime 的实现上。

例如下面这段代码：

```go
package main

func main() {
	ch := make(chan int)

	go func() {
		ch <- 1
	}()

	println(<-ch)
}
```

表面上只有 `make(chan int)`、`go func()`、发送、接收几个语法动作；底层则至少涉及：

- 创建 channel：分配并初始化 `hchan`。
- 创建 goroutine：创建 `g`，把它放入可运行队列。
- 调度 goroutine：由调度器把 `g` 放到某个线程上执行。
- channel 发送和接收：可能直接交接数据，也可能阻塞和唤醒 goroutine。
- 内存分配：channel、goroutine 栈、闭包对象都可能触发分配。
- GC 协作：分配对象需要记录类型信息，写指针可能触发写屏障。

所以 runtime 不是某个单独功能，而是一组支撑 Go 语言语义落地的机制。Go 语法越是“高级”，背后越可能有 runtime 参与。

#### Runtime解决的问题

如果没有 runtime，Go 语言里的很多能力就无法用简单语法表达：

- `go f()` 需要有人把函数变成可调度的 goroutine。
- goroutine 数量可以远多于 OS 线程，需要有人做 M:N 调度。
- goroutine 栈可以很小地开始并按需增长，需要有人检查和搬迁栈。
- `new(T)`、`&T{}`、逃逸对象需要有人在堆上分配内存。
- 堆对象不再使用后需要自动回收，需要 GC。
- channel、select、mutex、timer、netpoll 都需要阻塞和唤醒机制。
- panic 需要沿调用栈展开，defer 需要按规则执行，recover 需要在特定位置生效。

这些能力有些属于语言语义，有些属于工程优化，有些属于操作系统适配。runtime 的价值在于把它们组织成统一的运行模型，使开发者可以用相对简单的语法写并发程序，而不必手动管理线程、栈和堆对象生命周期。

#### Runtime不是JVM

Go runtime 很容易被误解成“Go 虚拟机”。这个说法不准确。

JVM 的典型模型是：Java 源码编译成字节码，运行时由 JVM 解释执行或 JIT 编译执行。Go 的常规模型是：Go 源码提前编译成本地机器码，程序启动后直接在目标机器上执行。runtime 并不负责解释 Go 字节码，它更像是被链接进可执行文件的一套底层库和调度系统。

可以粗略对比为：

| 项目 | Go runtime | JVM |
| --- | --- | --- |
| 输入形式 | 已编译的机器码 | 字节码 |
| 是否通常需要独立运行环境 | 不需要单独启动 runtime 进程 | 需要 JVM |
| 主要职责 | 调度、内存、GC、栈、阻塞唤醒、系统集成 | 字节码执行、JIT、GC、类加载、运行时优化 |
| 用户程序形态 | 一个包含 runtime 的可执行文件 | 字节码运行在虚拟机上 |

这并不是说 Go runtime 更简单。相反，它把大量复杂性放在编译期、链接期和运行时协作中：编译器负责插入 runtime 调用和生成元数据，链接器负责把 runtime 代码和用户代码组织进二进制，程序启动后 runtime 再接管调度、内存和 GC。

#### 典型源码入口

学习 runtime 时，不需要一开始把源码从头读到尾。先知道几个入口文件即可：

```text
runtime/proc.go       程序启动、调度器、GMP 主线
runtime/runtime2.go   g、m、p、schedt 等核心结构体
runtime/stubs.go      编译器和 runtime 之间的一些声明桥接
runtime/malloc.go     内存分配入口
runtime/mheap.go      heap、span、arena
runtime/mgc.go        GC 主流程
runtime/chan.go       channel 实现
runtime/select.go     select 实现
runtime/sema.go       信号量、阻塞唤醒
runtime/time.go       timer
runtime/netpoll.go    网络轮询抽象
runtime/panic.go      defer、panic、recover
```

这篇第一部分先不展开每个模块的细节，只建立整体地图。后续章节再分别进入调度器、内存、GC、channel、timer、netpoll 等具体实现。

### 02 Runtime整体架构

从职责上看，Go runtime 可以分成几层：

```text
              用户代码
                 |
         编译器插入的 runtime 调用
                 |
+----------------------------------+
|            Go runtime            |
|                                  |
|  启动与初始化                    |
|  Scheduler: G / M / P            |
|  Memory Allocator: mcache/mcentral/mheap
|  GC: mark/sweep/write barrier    |
|  Stack: goroutine stack          |
|  Blocking: park/unpark/sema      |
|  Channel / Select                |
|  Timer / Netpoll / Syscall       |
|  Panic / Defer / Recover         |
+----------------------------------+
                 |
          操作系统与硬件
```

这张图的重点是：runtime 位于用户代码和操作系统之间，但它不是把所有系统调用都包一层那么简单。它真正管理的是 Go 自己的执行模型。

操作系统看到的是线程、内存页、信号、文件描述符、网络事件；Go 程序员看到的是 goroutine、channel、timer、heap object、panic、defer。runtime 的工作就是在这两套模型之间做转换。

#### 调度器：把goroutine映射到线程

Go 的并发单位是 goroutine，操作系统的调度单位是线程。两者不是一一对应关系。runtime 调度器负责把大量 goroutine 放到较少数量的 OS 线程上执行。

核心模型是 GMP：

```text
G: goroutine，表示一个待执行的 Go 任务
M: machine，表示一个 OS 线程
P: processor，表示执行 Go 代码所需的调度资源
```

抽象关系如下：

```text
        runq
      +------+
      |  G   |
      |  G   |
      |  G   |
      +------+
         |
         v
M  <->  P  ->  执行某个 G
```

`M` 必须拿到 `P` 才能执行 Go 代码。`P` 持有本地运行队列、分配缓存、timer 等关键资源。这样设计可以减少全局锁竞争，也让调度器更容易在多核机器上扩展。

调度器要处理的问题包括：

- 新 goroutine 创建后放到哪里。
- 当前 goroutine 阻塞后，线程是否还能继续执行别的 goroutine。
- 本地运行队列空了以后，如何从全局队列或其他 P 偷取任务。
- goroutine 长时间运行时，如何触发抢占。
- 线程进入 syscall 阻塞后，P 如何转移给其他线程继续执行 Go 代码。

这些问题会在 Scheduler 部分展开。第一部分只需要记住：goroutine 不是 OS 线程，`go f()` 创建的是 `G`，真正运行时还需要 `M` 和 `P` 配合。

#### 内存分配器：管理堆对象

Go 中对象可能分配在栈上，也可能分配在堆上。是否逃逸由编译器的逃逸分析决定；一旦对象需要上堆，分配动作就由 runtime 内存分配器完成。

Go 的分配器大致是分层结构：

```text
goroutine
   |
   v
P.mcache        每个 P 的本地缓存，快速分配小对象
   |
   v
mcentral        按 size class 管理 span
   |
   v
mheap           全局堆管理，向操作系统申请或归还内存
   |
   v
OS memory
```

这种结构的核心目标是降低分配成本。小对象分配非常频繁，如果每次都去全局堆上加锁申请，性能会很差。因此 runtime 会把对象按大小分类，让每个 P 持有一部分可直接分配的本地缓存。

内存分配器和 GC 是强耦合的。分配器不仅要给对象找空间，还要记录对象大小、是否包含指针、所在 span、GC 标记状态等信息。GC 后续才能知道哪些对象需要扫描、哪些内存可以复用。

#### GC：自动回收堆对象

Go 使用自动垃圾回收管理堆内存。程序不需要手动 `free`，但这不代表内存没有成本。runtime 需要周期性判断哪些对象仍然可达，哪些对象可以回收。

GC 主流程可以先抽象成：

```text
分配对象
   |
   v
堆增长或达到触发条件
   |
   v
开始 GC
   |
   v
标记仍然可达的对象
   |
   v
清扫未标记对象占用的空间
   |
   v
内存重新进入可分配状态
```

现代 Go GC 的重点不是“是否能回收”，而是“在低暂停时间下完成回收”。因此它会尽量让标记工作和用户 goroutine 并发执行，同时通过写屏障保证并发标记期间对象引用关系的变化不会破坏正确性。

需要先建立几个关键词：

- 根对象：全局变量、栈上的指针、寄存器中的指针等 GC 起点。
- 标记：从根对象出发，找到所有仍然可达的堆对象。
- 清扫：回收未被标记的对象所在空间。
- STW：Stop The World，短暂停止所有用户 goroutine。
- 写屏障：在并发 GC 期间，记录指针写入带来的引用变化。
- GC Pacer：控制 GC 触发和工作节奏，平衡吞吐、内存和延迟。

GC 会在 Memory & GC 部分详细展开。这里先记住：Go 的自动内存管理不是编译器单独完成的，真正的堆对象生命周期管理在 runtime 中完成。

#### 栈管理：让goroutine可以轻量

OS 线程通常有比较大的固定栈，而 goroutine 的初始栈很小，并且可以按需增长。这是 goroutine 能大量创建的重要原因之一。

函数调用时，编译器会在很多函数入口插入栈空间检查。如果当前 goroutine 的栈空间不够，执行会进入 runtime 的栈增长逻辑，分配更大的栈并把旧栈内容搬过去。

抽象过程如下：

```text
函数调用
   |
   v
检查当前 goroutine 栈空间是否足够
   |
   +-- 足够：继续执行函数体
   |
   +-- 不足：进入 runtime.morestack
             分配更大栈
             搬迁旧栈内容
             修正指针
             重新执行函数
```

栈管理和调度器、GC 都有关：

- 调度器切换 goroutine 时，需要保存和恢复 goroutine 的栈状态。
- GC 扫描根对象时，需要扫描 goroutine 栈上的指针。
- 栈增长时，如果栈上保存了指向旧栈位置的指针，需要修正。

所以 goroutine 的“轻量”不是免费得来的，而是 runtime 和编译器一起做了大量工作。

#### 阻塞与唤醒：park/unpark

Go 程序中很多操作都可能阻塞：

- channel 读写无法继续。
- `sync.Mutex` 抢锁失败。
- `time.Sleep` 等待时间到达。
- 网络 IO 暂时没有数据。
- syscall 阻塞在内核中。

阻塞的关键问题是：不能因为一个 goroutine 阻塞，就让整个 OS 线程无意义地闲着。runtime 会尽量把当前 goroutine 挂起，让线程继续执行其他可运行 goroutine。

抽象模型是：

```text
G 执行中
   |
   v
遇到无法继续的条件
   |
   v
gopark：保存状态，挂起 G，切换到调度器
   |
   v
其他 G 继续运行
   |
   v
条件满足
   |
   v
goready：把 G 放回可运行队列
```

channel、select、mutex、timer、netpoll 的底层都会和这套阻塞唤醒机制发生关系。理解 `gopark` 和 `goready`，后面很多模块会更容易串起来。

#### Timer与Netpoll：时间和网络事件

Go 的 `time.Sleep`、`time.Timer`、`time.Ticker` 都不是简单地让线程睡死。runtime 会把定时任务纳入调度系统，到时间后再唤醒对应 goroutine。

网络 IO 也是类似思路。Go 的网络库希望表现成同步阻塞风格：

```go
n, err := conn.Read(buf)
```

但底层不能让每个网络连接都独占一个阻塞线程。runtime 会结合操作系统的 IO 多路复用机制，例如 Linux 上的 epoll、macOS/BSD 上的 kqueue、Windows 上的 IOCP，把“等待网络事件”变成“挂起 goroutine，等待事件到达后唤醒”。

抽象流程：

```text
goroutine 调用 Read
   |
   v
数据未就绪
   |
   v
把 fd 注册到 netpoll
   |
   v
挂起当前 goroutine
   |
   v
OS 通知 fd 可读
   |
   v
runtime 唤醒 goroutine
```

这就是 Go 网络编程看起来简单，但大量连接下仍然能维持较好伸缩性的原因之一。

#### Runtime整体地图

把上面的模块串起来，可以得到一张更完整的心智图：

```text
                           +----------------+
                           |   main.main    |
                           +----------------+
                                    |
                                    v
+------------------------------------------------------------------+
|                            Go runtime                            |
|                                                                  |
|  startup                                                         |
|    rt0 -> rt0_go -> schedinit -> runtime.main -> main.main       |
|                                                                  |
|  scheduler                                                       |
|    G goroutine   M thread   P processor   runq   work stealing   |
|                                                                  |
|  memory                                                          |
|    stack growth   malloc   mcache   mcentral   mheap   arena     |
|                                                                  |
|  GC                                                              |
|    root scan   mark   write barrier   sweep   pacer              |
|                                                                  |
|  blocking                                                        |
|    gopark   goready   sema   channel   select   mutex            |
|                                                                  |
|  system integration                                              |
|    syscall   signal   timer   netpoll   cgo                     |
+------------------------------------------------------------------+
                                    |
                                    v
                    +-------------------------------+
                    | OS: threads / memory / events |
                    +-------------------------------+
```

后续学习不要把这些模块割裂看。比如 channel 阻塞会进入调度器，网络 IO 会进入 netpoll 和调度器，分配对象会影响 GC，GC 又会影响调度和写屏障，syscall 阻塞会触发 P 的转移。runtime 的难点不只是单个模块，而是模块之间的协作。

### 03 Go程序启动过程

很多人以为 Go 程序从 `main.main` 开始执行。严格来说不是。`main.main` 是用户代码入口，但不是整个进程的第一条逻辑入口。

操作系统加载可执行文件后，首先进入 Go runtime 提供的启动入口。runtime 完成栈、线程、调度器、内存、参数、CPU、操作系统相关初始化之后，才会创建并调度执行运行 `runtime.main` 的 goroutine；`runtime.main` 再负责执行包初始化和用户的 `main.main`。

可以先记住一条主线：

```text
OS loader
  -> runtime 的平台入口 rt0
  -> runtime.rt0_go
  -> runtime.args / osinit / schedinit
  -> 创建 main goroutine
  -> mstart 启动调度
  -> runtime.main
  -> 执行 init
  -> main.main
  -> runtime.exit
```

不同操作系统和架构的入口文件不同，例如 amd64、arm64、Linux、Windows、Darwin 都有各自的汇编入口。但主线是一致的：先进入 runtime，再进入用户 `main`。

#### 第一步：操作系统加载程序

当执行一个 Go 二进制时，操作系统会做常规进程加载工作：

- 创建进程地址空间。
- 加载代码段、数据段等可执行文件内容。
- 准备命令行参数和环境变量。
- 创建初始线程。
- 跳转到可执行文件记录的入口地址。

这个入口地址通常不是用户写的 `main.main`，而是 Go runtime 的一段启动汇编代码。它负责从操作系统提供的原始状态过渡到 Go runtime 能理解的状态。

#### 第二步：进入rt0和rt0_go

Go runtime 在不同平台上有不同的 `rt0` 入口。`rt0` 可以理解为 runtime 的最低层启动代码，主要由汇编实现，因为这时 Go 运行环境还没有准备好，不能随便调用普通 Go 函数。

抽象流程如下：

```text
_rt0_$GOARCH_$GOOS
   |
   v
runtime.rt0_go
   |
   v
初始化 g0 / m0 / TLS / 参数 / CPU能力 / OS信息
```

这里会出现两个重要对象：

```text
g0: 每个 M 都有的调度栈，用于执行 runtime 调度逻辑
m0: 程序启动时的第一个 OS 线程
```

普通 goroutine 执行用户代码时用自己的 goroutine 栈；调度器、栈增长、GC 等 runtime 内部逻辑常常需要切到 `g0` 栈执行。`m0` 则是初始线程，后续 runtime 可以根据需要创建更多线程。

#### 第三步：初始化调度器

启动代码会调用 runtime 的初始化逻辑，其中一条关键主线是 `schedinit`。它会准备调度器运行所需的全局状态。

可以抽象为：

```text
schedinit
  -> 初始化栈、内存分配器、GC相关状态
  -> 初始化 P
  -> 初始化全局调度结构
  -> 初始化模块和类型元数据
  -> 准备执行 main goroutine
```

这里的“初始化 P”很关键。`GOMAXPROCS` 决定同时执行 Go 代码的 P 数量上限。runtime 会根据这个值创建对应数量的 P。后续所有 Go 代码都要依附于某个 P 执行。

此时用户的 `main.main` 仍然没有运行。runtime 还在搭舞台。

#### 第四步：创建main goroutine

调度器初始化后，runtime 会创建第一个真正要运行的 goroutine。它不是直接指向用户的 `main.main`，而是指向 `runtime.main`。

抽象关系：

```text
newproc(runtime.main)
   |
   v
创建一个新的 G
   |
   v
把 G 放入可运行队列
   |
   v
mstart 启动调度循环
```

为什么不直接运行 `main.main`？因为在用户 `main` 之前，还需要执行一批 runtime 和包级初始化逻辑，包括：

- runtime 自身后续初始化。
- 启动后台监控线程或相关机制。
- 执行所有依赖包的 `init`。
- 执行 `main` 包的 `init`。
- 处理一些必须发生在主 goroutine 上的初始化顺序。

所以 `runtime.main` 更像是 Go 世界里的主控入口，`main.main` 是它调用的用户入口。

#### 第五步：执行init和main.main

`runtime.main` 会按依赖顺序执行包初始化。初始化顺序可以简单理解为：

```text
先初始化被依赖的包
再初始化依赖它们的包
同一个包内：
  先初始化包级变量
  再执行 init 函数
最后执行 main.main
```

例如：

```go
package main

import "fmt"

var name = initName()

func initName() string {
	fmt.Println("var init")
	return "go"
}

func init() {
	fmt.Println("init")
}

func main() {
	fmt.Println("main")
}
```

抽象执行顺序是：

```text
runtime 启动
  -> 依赖包初始化
  -> main 包变量初始化
  -> main 包 init
  -> main.main
```

这里要注意：`init` 不是由用户手动调用，也不是编译器简单地插到 `main.main` 开头，而是由 runtime 根据链接器准备好的初始化任务列表统一调度执行。

#### 第六步：main返回和程序退出

当 `main.main` 返回后，Go 程序通常会退出。其他 goroutine 不会阻止进程退出。

例如：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	go func() {
		time.Sleep(time.Second)
		fmt.Println("worker")
	}()

	fmt.Println("main return")
}
```

这段程序大概率只打印：

```text
main return
```

原因是 `main.main` 返回后，runtime 会走程序退出流程，进程结束，其他还没执行完的 goroutine 会被直接终止。想等待 goroutine 完成，需要使用 `sync.WaitGroup`、channel 或其他同步机制。

#### 启动流程总图

把 Go 程序启动过程画成一张图：

```text
执行二进制
   |
   v
OS 创建进程和初始线程
   |
   v
runtime rt0 汇编入口
   |
   v
runtime.rt0_go
   |
   v
初始化 g0、m0、TLS、参数、CPU、OS信息
   |
   v
runtime.schedinit
   |
   v
初始化调度器、P、内存分配器、GC等
   |
   v
newproc(runtime.main)
   |
   v
mstart / schedule
   |
   v
runtime.main
   |
   v
执行 package init
   |
   v
main.main
   |
   v
程序退出
```

这条链路里，最关键的是三个入口：

```text
rt0              平台相关启动入口
runtime.main     runtime 层面的主 goroutine 函数
main.main        用户代码入口
```

如果只能记一句话，可以这样说：Go 程序不是直接从 `main.main` 开始，而是先进入 runtime 启动入口，初始化调度器、内存、GC 等基础设施，再创建并调度执行 `runtime.main`，最后由 `runtime.main` 调用用户的 `main.main`。

### 04 Runtime与编译器、标准库、操作系统的关系

runtime 不是孤立存在的。它和编译器、链接器、标准库、操作系统共同组成 Go 程序的执行闭环。

可以用一张图概括：

```text
Go 源码
  |
  v
编译器
  - 类型检查
  - 逃逸分析
  - 插入 runtime 调用
  - 生成类型/GC 元数据
  - 生成机器码
  |
  v
链接器
  - 链接用户代码
  - 链接 runtime
  - 组织 init 任务
  - 生成可执行文件
  |
  v
运行期
  用户代码 <-> 标准库 <-> runtime <-> 操作系统
```

#### Runtime与编译器

编译器负责把 Go 源码变成机器码，但很多语言特性不能只靠普通机器指令完成。编译器会在合适的位置插入 runtime 调用，也会生成 runtime 需要的元数据。

常见协作包括：

| Go代码或语义 | 编译器/Runtime协作 |
| --- | --- |
| `go f()` | 编译器生成创建 goroutine 的调用，runtime 创建 `g` 并加入队列 |
| `make(chan T)` | 调用 runtime 创建并初始化 channel |
| `append(slice, x)` | 容量不足时调用 runtime 扩容 |
| `map[k]` | 调用 runtime 的 map 访问、赋值、删除逻辑 |
| `new(T)` / `&T{}` | 逃逸到堆时调用 runtime 分配 |
| 函数调用 | 插入栈增长检查，必要时进入 `morestack` |
| 写入指针 | GC 并发标记期间可能走写屏障 |
| `defer` / `panic` / `recover` | 编译器改写控制流并调用 runtime 机制 |
| interface 转换 | 依赖类型元数据和 runtime 辅助函数 |

例如：

```go
s = append(s, x)
```

如果 `s` 的容量足够，编译器可能生成直接写入底层数组的代码；如果容量不足，就需要进入 runtime 的 `growslice` 逻辑，分配更大的底层数组并搬迁数据。

再比如：

```go
p := &User{Name: "alice"}
```

这个对象到底在栈上还是堆上，不是由 `&` 这个符号单独决定，而是由编译器逃逸分析决定。如果对象生命周期不会逃出当前函数，可能放在栈上；如果会被外部继续访问，就需要放到堆上，由 runtime 分配并交给 GC 管理。

因此，学习 runtime 不能只看 runtime 目录。很多 runtime 行为的入口其实来自编译器生成的代码。

#### Runtime与链接器

Go 程序编译完成后，还需要链接。链接器会把用户包、依赖包、runtime、类型元数据、符号表、初始化任务等组织成最终可执行文件。

链接器和 runtime 的关系主要体现在：

- 把 runtime 代码链接进最终二进制。
- 确定程序入口地址，让进程先进入 runtime 启动代码。
- 收集和组织各个包的初始化任务。
- 保留类型元数据、接口方法表、GC bitmap 等运行期需要的数据。
- 处理跨包符号引用。

这也是为什么一个很简单的 Go 程序，编译后的二进制里也能看到大量 `runtime.*` 符号。即使用户代码只写了几行，程序仍然需要启动、调度、分配、退出等运行时能力。

可以用下面的命令观察二进制里的 runtime 符号：

```bash
go build -o app .
go tool nm app | rg "runtime.main|runtime.newproc|runtime.mallocgc"
```

如果没有安装 `rg`，也可以用系统自带的文本过滤工具。重点不是记住命令，而是理解：runtime 是最终二进制的一部分。

#### Runtime与标准库

标准库是用户直接导入的包，比如 `fmt`、`net`、`time`、`sync`、`reflect`。runtime 是更底层的支撑。很多标准库功能会通过编译器特权、`//go:linkname`、内部 ABI 或 runtime 暴露的低层函数与 runtime 协作。

典型关系：

| 标准库 | Runtime支撑 |
| --- | --- |
| `sync.Mutex` | 阻塞、唤醒、信号量、自旋策略 |
| `sync.WaitGroup` | 计数状态 + runtime semaphore |
| `time.Timer` / `time.Sleep` | runtime timer 和调度器 |
| `net` | runtime netpoll 和 OS 多路复用 |
| `reflect` | 类型元数据、方法表、接口表示 |
| `runtime` 包 | 暴露部分运行时能力，如 `GOMAXPROCS`、`GC`、`Caller` |

例如 `sync.Mutex` 看起来是标准库类型，但当 goroutine 抢锁失败并需要睡眠时，最终需要 runtime 提供挂起和唤醒能力。再比如 `net.Conn.Read` 看起来是一个普通阻塞调用，但底层会把 goroutine 挂起到 netpoll，等待 fd 可读后再唤醒。

标准库提供的是稳定、面向用户的 API；runtime 提供的是底层机制。两者边界并不总是完全公开，因为 runtime 中有很多细节不是给普通应用直接调用的。

#### Runtime与操作系统

runtime 最终仍然要依赖操作系统。它不能凭空创建线程、申请内存或等待网络事件。操作系统提供的是更底层的原语，runtime 在这些原语之上实现 Go 的执行模型。

常见依赖包括：

| 操作系统能力 | Runtime用途 |
| --- | --- |
| 线程 | 承载 M，执行 Go 代码 |
| 虚拟内存 | 堆、栈、arena、span 分配 |
| 系统调用 | 文件、网络、时间、进程等 OS 能力 |
| 信号 | 抢占、崩溃处理、profiling、用户信号 |
| futex/semaphore 等等待机制 | 阻塞和唤醒线程 |
| epoll/kqueue/IOCP | 网络事件轮询 |
| 时钟 | timer、sleep、调度和 profiling |

runtime 会为不同平台提供不同实现。比如网络轮询在不同系统上的底层机制不同：

```text
Linux      epoll
macOS/BSD  kqueue
Windows    IOCP
```

但用户代码通常只看到统一的 Go API：

```go
conn.Read(buf)
time.Sleep(time.Second)
go handle(conn)
```

这就是 runtime 做系统集成的意义：屏蔽操作系统差异，把不同平台的线程、内存、事件、信号机制统一映射到 Go 的 goroutine、channel、timer、netpoll 模型中。

#### 四者协作示例：go func()

以 `go f()` 为例，可以把编译器、runtime、标准库、操作系统的关系串起来：

```go
go f()
```

大致过程：

```text
编译器
  -> 识别 go 语句
  -> 生成创建 goroutine 所需的调用和参数准备代码

runtime
  -> 创建 g
  -> 准备 goroutine 栈和入口函数
  -> 把 g 放入 P 的运行队列
  -> 调度器选择合适时机运行它

操作系统
  -> 提供承载 M 的线程
  -> 在线程上执行最终机器码
```

如果 `f` 里面使用了 `time.Sleep`：

```go
func f() {
	time.Sleep(time.Second)
}
```

又会多一层标准库协作：

```text
time.Sleep
  -> 调用 runtime timer 相关逻辑
  -> 当前 goroutine 挂起
  -> timer 到期
  -> runtime 把 goroutine 放回运行队列
```

这说明 Go 的一个简单语法动作，背后常常跨过多个层次。

#### 四者协作示例：对象分配和GC

再看对象分配：

```go
func build() *User {
	u := &User{Name: "alice"}
	return u
}
```

大致过程：

```text
编译器
  -> 做逃逸分析
  -> 判断 u 需要在函数返回后继续存活
  -> 生成堆分配相关代码
  -> 生成类型和 GC 扫描元数据

runtime
  -> 在堆上分配 User 对象
  -> 必要时触发 GC 或协助 GC
  -> GC 时根据元数据扫描对象中的指针

操作系统
  -> 当堆空间不足时提供更多虚拟内存
```

这里最重要的点是：GC 不只是 runtime 自己“猜”对象结构。它依赖编译器生成的类型信息和指针位图。没有这些元数据，GC 就不知道对象里哪些字段是指针、哪些只是普通整数或字节。

#### 常见误区

误区一：Go 程序从 `main.main` 第一行开始。

更准确的说法是：进程先进入 runtime 启动入口，完成初始化后由 `runtime.main` 调用用户的 `main.main`。

误区二：goroutine 就是线程。

goroutine 是 Go runtime 管理的轻量执行单元，线程是操作系统调度单位。runtime 负责把多个 goroutine 调度到多个线程上运行。

误区三：GC 只和堆有关，和编译器无关。

GC 的运行在 runtime 中，但它依赖编译器生成的类型元数据、栈图、写屏障插入和逃逸分析结果。

误区四：标准库和 runtime 没什么关系。

`sync`、`time`、`net`、`reflect` 等包都和 runtime 有深度协作。标准库提供 API，runtime 提供底层机制。

误区五：runtime 只是性能优化。

runtime 不只是优化层。没有 runtime，goroutine、channel、自动栈增长、GC、panic/recover 等语言语义都无法完整落地。

#### 问答速览

问题一：Go runtime 是什么？

```text
Go runtime 是被链接进 Go 可执行文件的一组运行时基础设施。
它不是 JVM 式虚拟机，不负责解释字节码，而是在本地机器码执行过程中提供程序启动、goroutine 调度、栈管理、内存分配、GC、channel/select、timer、netpoll、syscall、panic/recover 等能力。
编译器会为一些语言特性插入 runtime 调用并生成元数据，链接器会把 runtime 和用户代码组织到最终二进制中，运行时再通过操作系统线程、内存、信号和 IO 多路复用等能力实现 Go 的执行模型。
```

问题二：Go 程序启动流程是什么？

```text
Go 程序不是直接从 main.main 开始。
操作系统加载可执行文件后先进入 runtime 的平台启动入口 rt0，然后进入 runtime.rt0_go，初始化 g0、m0、TLS、参数、CPU、OS、调度器、P、内存分配器和 GC 等状态。
之后 runtime 创建运行 runtime.main 的 main goroutine，启动调度循环。
runtime.main 会执行包初始化，包括依赖包 init 和 main 包 init，最后调用用户的 main.main。
main.main 返回后，runtime 执行退出流程，进程结束。
```

问题三：runtime、编译器、标准库、操作系统是什么关系？

```text
编译器负责把 Go 代码编译成机器码，同时插入 runtime 调用并生成类型、栈图、GC bitmap 等元数据。
链接器把用户代码、依赖包和 runtime 链接成可执行文件，并组织初始化任务。
标准库提供用户 API，其中 sync、time、net、reflect 等包会依赖 runtime 的阻塞唤醒、timer、netpoll 和类型系统能力。
runtime 则通过操作系统提供的线程、虚拟内存、系统调用、信号、epoll/kqueue/IOCP 等机制实现 Go 的调度、内存和系统集成模型。
```

## Part 2 Scheduler

### 05 GMP模型

Go 调度器要解决的核心问题是：用少量 OS 线程执行大量 goroutine，并且尽量减少锁竞争、线程切换和阻塞带来的浪费。

操作系统只认识线程。线程创建、销毁和上下文切换都比 goroutine 重得多；如果每个并发任务都绑定一个线程，高并发程序会很快被线程数量、栈空间、内核调度和锁竞争拖垮。Go 的做法是在用户态维护自己的调度器，把 goroutine 映射到 OS 线程上执行。

今天的 Go 调度器可以先抽象成三个对象：

```text
G: goroutine，Go 代码的执行任务
M: machine，操作系统线程
P: processor，执行 Go 代码所需的调度资源
```

三者关系可以画成这样：

```text
             全局运行队列
          +----------------+
          | G  G  G  G ... |
          +----------------+
                  ^
                  |
                  v
+---------+    +---------+       +---------+    +---------+
|   M     | -> |   P     |       |   M     | -> |   P     |
| OS线程  |    | runq    |       | OS线程  |    | runq    |
+---------+    +---------+       +---------+    +---------+
                  |                           |
                  v                           v
               当前 G                      当前 G
```

`G` 是真正要执行的任务，里面保存了栈、程序计数器、调度上下文、状态、关联的 `M` 等信息。`M` 是 OS 线程，真正运行机器指令。`P` 是中间层，它持有本地运行队列、内存分配缓存、timer 等执行 Go 代码时需要的资源。

一个 `M` 必须绑定一个 `P` 才能执行普通 Go 代码。没有 `P` 的 `M` 可以存在，例如陷入系统调用的线程，但它不能继续调度新的 Go 代码。`GOMAXPROCS` 控制的是同时处于活跃状态的 `P` 的数量，通常也就是同一时刻最多有多少个线程在执行 Go 代码。

为什么需要 `P` 这一层？早期的 G-M 模型把可运行 goroutine 放在全局队列里，多个线程都要竞争同一把调度锁。并发一高，调度器自己就变成瓶颈。引入 `P` 后，每个 `P` 都有本地运行队列，大多数创建和调度都可以在本地完成，只有本地队列放不下、负载不均衡或者需要公平性时，才访问全局队列或其他 `P`。

可以把 `P` 理解成一个“本地调度器”：

```text
P
  runnext     下一个优先执行的 G
  runq        本地可运行 G 队列
  mcache      当前 P 的小对象分配缓存
  timers      与当前 P 相关的定时器
  status      _Pidle / _Prunning / _Psyscall / _Pgcstop / _Pdead
```

`P` 的本地运行队列是调度器性能的关键。新创建的 goroutine 通常会优先放到当前 `P` 的 `runnext` 或本地队列；本地队列满了，才会把一部分任务转移到全局队列。这样可以减少全局锁竞争，也能提高缓存局部性。

#### GMP各自负责什么

`G` 关注的是“任务本身”：

- 当前 goroutine 的栈范围。
- 保存和恢复执行现场所需的 `gobuf`，里面包含 `sp`、`pc` 等信息。
- 当前状态，例如 `_Grunnable`、`_Grunning`、`_Gwaiting`、`_Gsyscall`。
- 抢占相关标记，例如 `preempt`。
- defer、panic 等与当前 goroutine 绑定的链表。

`M` 关注的是“线程执行环境”：

- `g0`：调度专用 goroutine，使用系统栈执行 runtime 调度逻辑。
- `curg`：当前线程正在运行的用户 goroutine。
- `p`：当前绑定的 `P`。
- `oldp`：进入系统调用前绑定的 `P`。
- 与线程创建、阻塞、锁定、cgo、信号相关的状态。

`P` 关注的是“执行 Go 代码的资源”：

- 本地运行队列。
- 空闲 `G` 缓存。
- 小对象分配缓存 `mcache`。
- timer、GC 辅助、trace 等调度相关资源。

这也是为什么 `M` 和 `P` 要分开。线程可能因为 syscall 阻塞，但 `P` 不应该一直跟着线程一起卡住；否则这个 `P` 本地队列里的其他 goroutine 也会无法运行。把 `P` 从阻塞的 `M` 上摘下来，再交给其他 `M`，是 Go 调度器提高线程利用率的重要手段。

举个例子，假设 `GOMAXPROCS=1`，当前只有一个 `P`。`G1` 在 `M1` 上执行，并调用一个可能长时间阻塞的系统调用，例如读取一个暂时没有数据的 pipe、等待某个设备 IO，或者执行一个绕过 netpoll 的阻塞 syscall。此时 `M1` 可能卡在内核里，短时间回不到用户态。

如果 `P` 也一直挂在 `M1` 身上，那么即使本地队列里还有 `G2`、`G3`，它们也拿不到执行 Go 代码所需的调度资源，整个 Go 程序看起来就像被这个 syscall 拖住了一样。Go runtime 会让 `M1` 进入 syscall 时暂时交出 `P`；如果 syscall 没有很快返回，`sysmon` 可以把这个 `P` 夺回来，绑定到另一个空闲线程 `M2` 上，让 `G2`、`G3` 继续运行。等 `G1` 的 syscall 返回后，它再重新争取 `P`，拿不到就先回到可运行队列。

#### 调度器的演进脉络

Go 调度器大致经历过几个阶段：

```text
单线程 G-M
  -> 能运行，但没有并行能力

多线程 G-M
  -> 可以多个线程执行 goroutine
  -> 全局运行队列和全局锁竞争严重

G-M-P + work stealing
  -> 引入 P 和本地运行队列
  -> 本地队列为空时从其他 P 窃取任务

协作式抢占
  -> 通过函数调用入口检查抢占请求
  -> 长时间不发生函数调用的循环仍可能占用线程

信号式异步抢占
  -> Go 1.14 后引入
  -> runtime 可以通过信号打断运行中的线程并触发抢占
```

调度器的方向一直很明确：减少全局共享状态，尽量把调度动作放在本地完成；当 goroutine 长时间运行、线程陷入系统调用、GC 需要扫描栈或者某个 `P` 没有任务时，再通过抢占、解绑 `P`、全局队列和工作窃取来恢复整体公平性。

### 06 Goroutine生命周期

一个 goroutine 从创建到结束，通常会经历下面几类状态：

| 状态 | 含义 |
| --- | --- |
| `_Gidle` | 刚分配出来，还没有完成初始化 |
| `_Grunnable` | 已经准备好，可以被调度执行 |
| `_Grunning` | 正在某个 `M` 上运行 |
| `_Gwaiting` | 因 channel、锁、timer、netpoll 等原因等待 |
| `_Gsyscall` | 正在执行系统调用 |
| `_Gpreempted` | 因抢占被暂停，等待重新调度 |
| `_Gdead` | 已结束或空闲，可被复用 |
| `_Gcopystack` | 栈正在复制 |
| `_Gscan` | GC 正在扫描栈，可与其他状态组合出现 |

理解生命周期时，不必一开始记住所有内部状态。先记住三类就够了：

```text
等待中   _Gwaiting / _Gsyscall / _Gpreempted
可运行   _Grunnable
运行中   _Grunning
```

#### 创建：go语句不是直接启动线程

当代码里写下：

```go
go f(x)
```

编译器会把它改写成创建 goroutine 的 runtime 调用。runtime 不会为它直接创建一个 OS 线程，而是创建一个新的 `G`，准备好栈、参数和入口函数，然后把这个 `G` 放入运行队列。

抽象流程如下：

```text
go f(x)
   |
   v
runtime.newproc
   |
   v
获取或创建 g
   |
   v
分配初始栈，拷贝参数
   |
   v
设置 sched.sp / sched.pc / startpc
   |
   v
状态切到 _Grunnable
   |
   v
放入当前 P 的 runnext 或 runq
   |
   v
必要时 wakep 唤醒空闲线程
```

这里有几个细节值得注意。

第一，`G` 可以复用。goroutine 结束后不会马上把结构体彻底丢掉，runtime 会把它放回空闲列表，后续创建 goroutine 时可以复用，降低分配成本。

第二，新 goroutine 的参数需要放到它自己的栈上。因为它何时执行不由当前函数决定，如果只引用当前调用方的临时参数区域，就可能在真正执行时读到无效数据。

第三，goroutine 的入口不是简单地“调用 f”。runtime 会预先设置这个 `G` 的调度现场，让它首次被调度执行时从 `f` 开始运行，并且在 `f` 返回后进入 `goexit` 做清理。

可以把新 goroutine 的执行栈想成这样：

```text
新 G 的栈
+---------------------+
| f 的参数             |
+---------------------+
| 返回后进入 goexit    |
+---------------------+

g.sched.pc -> f
g.sched.sp -> 栈顶位置
```

#### 运行：从队列到线程

当某个 `M` 绑定了 `P` 并进入调度循环后，会从运行队列里取出一个 `_Grunnable` 的 `G`。调度器把它切换为 `_Grunning`，建立 `M.curg` 和 `G.m` 的关联，然后通过低层汇编恢复 `G` 的栈指针和程序计数器。

抽象过程如下：

```text
schedule
   |
   v
找到一个 _Grunnable 的 G
   |
   v
execute
   |
   v
G -> _Grunning
M.curg = G
G.m = M
   |
   v
gogo(&G.sched)
   |
   v
继续执行 G 的 Go 代码
```

`gogo` 是调度器真正把执行权交给 goroutine 的地方。它会恢复 `sp`、`pc` 等寄存器，让 CPU 从 goroutine 上次停下的位置继续执行。对刚创建的 goroutine 来说，这个位置就是入口函数；对被阻塞后重新唤醒的 goroutine 来说，这个位置就是上次保存的继续点。

#### 阻塞：让出线程，不让线程白等

很多操作会让 goroutine 无法继续执行：

- channel 读写暂时不能完成。
- `sync.Mutex` 抢锁失败并进入等待。
- `time.Sleep` 等待 timer 到期。
- 网络 IO 需要等待 fd 可读可写。
- GC 或抢占要求当前 goroutine 暂停。

如果让 OS 线程跟着一起阻塞，Go 的 M:N 调度就失去了意义。runtime 通常会把当前 goroutine 挂起，然后让 `M` 去执行其他可运行的 goroutine。

典型路径是 `gopark`：

```text
G 正在运行
   |
   v
遇到不能继续的条件
   |
   v
gopark
   |
   v
切到 g0 栈执行 park_m
   |
   v
G: _Grunning -> _Gwaiting
解除 G 和 M 的绑定
   |
   v
schedule 选择下一个 G
```

等条件满足后，例如 channel 另一端准备好了、timer 到期了、网络事件到了，runtime 会走唤醒路径：

```text
条件满足
   |
   v
goready / ready
   |
   v
G: _Gwaiting -> _Grunnable
   |
   v
放回运行队列
   |
   v
必要时唤醒空闲 M
```

这套 `gopark` / `goready` 机制是很多同步原语的共同底座。channel、select、mutex、timer、netpoll 看起来是不同功能，底层都需要把 goroutine 从“运行中”切到“等待中”，再在合适时机放回“可运行”。

#### 结束：goexit清理现场

当 goroutine 的入口函数返回后，它不会回到创建它的 goroutine。runtime 在它的栈上预先布置了返回位置，使它返回后进入 `goexit`。

结束流程可以抽象成：

```text
f 返回
   |
   v
goexit
   |
   v
切到 g0 栈执行 goexit0
   |
   v
G: _Grunning -> _Gdead
清理 defer、panic、timer、label 等字段
解除 G 和 M 的关系
   |
   v
把 G 放回空闲列表
   |
   v
schedule 继续调度
```

所以 goroutine 的生命周期不是“函数调用完就消失”这么简单。它背后包含结构体复用、栈管理、状态迁移、调度上下文保存和恢复。

### 07 Goroutine调度

调度器的主循环可以理解成一句话：找到一个可以运行的 `G`，把它放到当前 `M` 上执行；当前 `G` 停下后，再回到调度器继续找下一个。

抽象伪代码如下：

```text
for {
  gp = 从合适的位置找一个可运行 G
  execute(gp)
}
```

真实实现要复杂得多，因为它要处理全局队列、本地队列、netpoll、timer、GC、syscall、抢占、空闲线程、工作窃取等情况。但理解主干时，可以先从“找 G 的顺序”入手。

#### 从哪里找可运行的G

一个 `P` 想找可运行 goroutine，通常会关注这些来源：

```text
1. 为了公平性，偶尔检查全局运行队列
2. 检查当前 P 的 runnext
3. 检查当前 P 的本地 runq
4. 检查全局运行队列
5. 检查 netpoll 是否返回了可运行 G
6. 从其他 P 的运行队列里偷一部分 G
7. 实在没有任务时，进入休眠等待唤醒
```

可以画成这样：

```text
schedule / findrunnable
   |
   +--> global runq
   |
   +--> local runnext / runq
   |
   +--> netpoll
   |
   +--> steal from other P
   |
   +--> park M
```

本地队列优先是为了性能。全局队列偶尔检查是为了公平性：如果只看本地队列，全局队列里的 goroutine 可能长期得不到执行。工作窃取是为了负载均衡：如果一个 `P` 空了，而另一个 `P` 的队列很长，空闲 `P` 会随机选择其他 `P` 偷一部分任务。

#### 本地队列、全局队列和runnext

每个 `P` 都有一个本地运行队列。新创建的 goroutine 通常放在当前 `P`，这样创建者和被创建者有更高概率在同一个 CPU 缓存环境下执行。

`runnext` 是一个特殊位置，表示“下一个优先运行的 G”。它常用于让刚刚变为可运行的 goroutine 更快接上执行，减少调度延迟。不过 runtime 不能只偏向 `runnext`，否则会破坏公平性，所以调度器仍然会周期性地查看全局队列。

整体关系如下：

```text
P
  runnext:  G?
  runq:     [G, G, G, ...]

sched
  runq:     [G, G, G, ...]  全局运行队列
```

本地队列满时，runtime 会把一部分本地任务转移到全局队列。这样做有两个好处：当前 `P` 不会无限堆积任务，其他空闲 `P` 也更容易从全局队列拿到工作。

#### 工作窃取

工作窃取解决的是负载不均衡问题。假设有 4 个 `P`：

```text
P0: G G G G G G
P1:
P2:
P3: G
```

如果 `P1` 和 `P2` 只能等全局队列，它们可能会空转。工作窃取允许空闲 `P` 随机选择其他 `P`，从对方本地队列中拿走一部分可运行 goroutine：

```text
P0: G G G
P1: G G G
P2:
P3: G
```

它不是为了保证每个 `P` 的队列长度完全相同，而是用较低成本让空闲执行资源尽快找到任务。

#### g0和mcall

调度器不能直接在普通用户 goroutine 的栈上随便做所有事情。每个 `M` 都有一个特殊的 `g0`，它使用的是线程的调度栈，专门执行 runtime 内部调度逻辑。

当当前 goroutine 要挂起、退出、让出或者进入某些 runtime 调度路径时，会通过 `mcall` 切换到 `g0` 栈：

```text
用户 G 栈
  执行业务代码
  |
  | mcall
  v
g0 栈
  修改 G 状态
  解除 G 和 M 绑定
  调用 schedule
```

为什么需要这样做？因为调度器要保存和修改当前 goroutine 的执行现场，还可能切换到另一个 goroutine。如果继续在当前用户栈上执行复杂调度逻辑，会让栈所有权、GC 扫描和恢复现场变得更难处理。

#### 常见调度触发点

调度不只发生在 goroutine 主动结束时。常见触发点包括：

| 触发点 | 典型路径 | 当前 G 后续状态 |
| --- | --- | --- |
| goroutine 正常返回 | `goexit` | `_Gdead` |
| channel、锁、timer、netpoll 等等待 | `gopark` | `_Gwaiting` |
| 条件满足被唤醒 | `goready` | `_Grunnable` |
| 主动让出 | `runtime.Gosched` | `_Grunnable` |
| 系统调用 | `entersyscall` / `exitsyscall` | `_Gsyscall` 或回到运行队列 |
| 抢占 | `preemptone` / `asyncPreempt` | `_Gpreempted` 或重新入队 |
| GC STW 或栈扫描 | runtime 内部路径 | 视具体阶段而定 |

这张表说明一个事实：调度器不是后台定时器每隔几毫秒简单轮转一次 goroutine。很多调度动作是由 runtime 的具体事件触发的，例如阻塞、唤醒、syscall、GC 和抢占。

#### 一次完整调度的抽象流程

把上面的概念串起来，可以得到一次调度循环：

```text
M 绑定 P
   |
   v
schedule 在 g0 栈上运行
   |
   v
从 runnext / local runq / global runq / netpoll / steal 找到 G
   |
   v
execute
   |
   v
G: _Grunnable -> _Grunning
M.curg = G
G.m = M
   |
   v
gogo 恢复 G 的执行现场
   |
   v
G 执行 Go 代码
   |
   +-- 正常返回 -> goexit -> schedule
   |
   +-- 阻塞等待 -> gopark -> schedule
   |
   +-- 主动让出 -> Gosched -> schedule
   |
   +-- syscall -> entersyscall -> P 可能交给其他 M
   |
   +-- 被抢占 -> 保存现场 -> schedule
```

理解这条主线后，再看 `runtime/proc.go` 里的 `schedule`、`findrunnable`、`execute`、`gopark`、`goready`、`goexit0`，就不会觉得它们是互不相关的函数。

### 08 抢占机制

如果 goroutine 必须主动阻塞或调用 `runtime.Gosched` 才能让出线程，那么下面这种代码就会有问题：

```go
func spin() {
	for {
	}
}
```

这个 goroutine 不读 channel、不睡眠、不系统调用，也没有主动让出。没有抢占机制时，它可能长期占住当前 `M` 和 `P`，导致其他 goroutine 饥饿，GC 也更难在合适时间扫描它的栈。

抢占机制要解决的就是：当一个 goroutine 运行太久，或者 runtime 需要它暂停时，调度器如何让它停下来，把执行权交给别的 goroutine。

#### 协作式抢占

Go 1.2 开始引入协作式抢占。它的基本思路是：runtime 发出抢占请求，goroutine 在函数调用等安全位置检查这个请求，如果发现需要抢占，就进入调度器。

抽象流程如下：

```text
sysmon 发现某个 G 运行过久
或 GC 需要停止/扫描
   |
   v
设置 G 的抢占标记
   |
   v
G 发生函数调用
   |
   v
函数入口的栈检查进入 runtime
   |
   v
发现抢占请求
   |
   v
保存现场，让出 M
```

这种方式实现成本相对低，因为函数调用入口本来就要做栈增长检查，runtime 可以借这个机会检查抢占请求。问题也很明显：如果 goroutine 长时间不发生函数调用，例如紧密循环，就可能迟迟无法响应抢占。

所以协作式抢占的关键不是“runtime 想抢就一定能立刻抢”，而是“goroutine 运行到可检查的位置时配合让出”。

#### 信号式异步抢占

Go 1.14 之后引入了基于信号的异步抢占，弥补了协作式抢占在紧密循环上的不足。它的思路是：runtime 可以向正在运行某个 goroutine 的线程发送信号，操作系统中断该线程，信号处理函数再把执行流导向 runtime 的抢占逻辑。

抽象流程如下：

```text
sysmon / GC 发现需要抢占
   |
   v
runtime 标记目标 G 可抢占
   |
   v
向目标 M 发送抢占信号
   |
   v
OS 中断正在运行的线程
   |
   v
信号处理函数检查当前 PC/SP 是否可安全抢占
   |
   v
注入 asyncPreempt 调用
   |
   v
当前 G 进入抢占路径
   |
   v
状态切换，回到 schedule
```

异步抢占不是在任意机器指令上都粗暴打断。runtime 仍然要考虑安全点：当前寄存器和栈是否能被 GC 正确理解，是否处于不能被打断的 runtime 临界区，是否会破坏写屏障、栈增长或系统调用边界。抢占越“强”，实现越需要小心。

#### sysmon的角色

`sysmon` 是 runtime 的系统监控线程，它不依赖普通 `P` 执行。调度器很多后台维护动作都和它有关，例如：

- 检查是否有长时间运行的 goroutine 需要抢占。
- 检查陷入 syscall 的 `P` 是否需要被夺回。
- 处理 timer 和 netpoll 相关唤醒。
- 协助 GC 和调度器推进一些后台状态。

在调度器视角里，`sysmon` 像一个监督者。普通 `M` 正在执行用户代码时，不一定有机会主动回到调度器；`sysmon` 可以从旁边观察运行时间和系统状态，在必要时发起抢占或唤醒。

#### 抢占和公平性

抢占的目标不是让 goroutine 严格按时间片平均分配 CPU。Go 调度器追求的是实用的公平性：

- 长时间运行的 goroutine 不能永久占住 `P`。
- 全局队列里的 goroutine 不能长期饥饿。
- GC 需要在可接受时间内暂停或扫描 goroutine 栈。
- 网络事件和 timer 到期后，对应 goroutine 应该能及时恢复。

因此，抢占只是调度公平性的一部分。全局队列检查、工作窃取、`runnext`、netpoll 唤醒、syscall 解绑 `P`，都在共同维持整体调度效果。

#### 抢占不等于并行

容易混淆的一点是：抢占解决的是“正在运行的 goroutine 能不能被停下来”，并行解决的是“同一时刻能有多少 goroutine 真正在 CPU 上执行”。后者主要由 `GOMAXPROCS` 和可用 CPU 决定。

例如：

```go
runtime.GOMAXPROCS(1)
```

这时即使有抢占，同一时刻也只有一个 `P` 执行 Go 代码。抢占能让多个 goroutine 轮流获得执行机会，但不会让它们在多个 CPU 上同时运行。

### 09 Syscall与调度器

系统调用是调度器必须特别处理的场景。因为普通 Go 阻塞，例如 channel 等待，runtime 可以直接挂起当前 `G` 并让 `M` 执行别的任务；但系统调用会让当前 OS 线程进入内核，线程可能长时间不能回来。

如果 `M` 带着 `P` 一起卡在系统调用里，这个 `P` 上的本地队列、分配缓存和执行资格都会被浪费。Go 的处理方式是：在可能阻塞的系统调用前，把当前 `G` 标记为 `_Gsyscall`，并让 `M` 和 `P` 暂时分离。

#### 进入系统调用

普通系统调用会经过 runtime 包装，大致流程如下：

```text
G 正在 M 上运行，M 绑定 P
   |
   v
准备进入 syscall
   |
   v
保存 G 的 PC/SP
   |
   v
G: _Grunning -> _Gsyscall
   |
   v
M.oldp = P
M.p = nil
P.status = _Psyscall
   |
   v
M 进入内核执行系统调用
```

关键点在于 `P` 的状态变成 `_Psyscall`，并且从当前 `M` 上解绑。这样当 runtime 发现这个系统调用时间较长时，可以把这个 `P` 交给其他空闲 `M`，继续执行其他 goroutine。

抽象图如下：

```text
进入 syscall 前

M ---- P ---- runq: G G G
|
curg: Gsys

进入 syscall 后

M ----> syscall 阻塞在内核
|
oldp: P(_Psyscall)

P 可被 runtime 夺回并交给其他 M
```

#### sysmon夺回P

如果系统调用很快返回，原来的 `M` 可能还能直接拿回 `P`。但如果它卡住太久，`sysmon` 会发现 `_Psyscall` 状态的 `P` 需要被重新利用，于是把它交给其他线程。

```text
M1 带着 G 进入 syscall
P 变成 _Psyscall
   |
   v
syscall 长时间未返回
   |
   v
sysmon retake P
   |
   v
P 绑定到 M2
   |
   v
M2 继续调度 P 队列里的其他 G
```

这样即使某个 goroutine 正在做阻塞系统调用，其他 goroutine 仍然可以继续运行。代价是：当原来的系统调用返回时，它所在的 `M1` 可能已经没有 `P` 了。

#### 退出系统调用

系统调用返回后，回到用户态的是当前 `M`。但 `M` 只有重新绑定到 `P`，才能继续执行当前 `G` 的 Go 代码。根据 `M` 能不能立刻拿到 `P`，这里有两条常见路径。

第一条是快速路径：原来的 `P` 还在，或者 `M` 能立刻拿到空闲 `P`。这时当前 `G` 可以从 `_Gsyscall` 回到 `_Grunning`，继续在当前线程上执行。

```text
syscall 返回
   |
   v
原 P 仍可用或拿到空闲 P
   |
   v
M 绑定 P
G: _Gsyscall -> _Grunning
   |
   v
继续执行
```

第二条是慢速路径：当前 `M` 没有拿到可用 `P`。这时当前 `G` 会变回 `_Grunnable`，放入全局运行队列，当前 `M` 也可能休眠或去找别的工作。

```text
syscall 返回
   |
   v
没有可用 P
   |
   v
G: _Gsyscall -> _Grunnable
   |
   v
放入全局运行队列
   |
   v
等待之后被某个 P 调度
```

这就是为什么系统调用边界必须和调度器协作。OS 只知道线程阻塞和返回，runtime 还要维护 `G`、`M`、`P` 的关系，保证 Go 层面的并发不会因为一个线程阻塞而整体停住。

#### RawSyscall为什么特殊

并不是所有系统调用都需要完整进入 runtime 调度流程。对于确定很快返回、不会长时间阻塞的系统调用，Go 可以走更轻量的 `RawSyscall` 路径，减少进入和退出调度器的成本。

但只要系统调用可能阻塞较长时间，就需要让 runtime 参与。否则当前 `M` 可能长时间占着 `P`，导致 `GOMAXPROCS` 名义上有多个 `P`，实际可用执行资源却被阻塞线程吞掉。

可以粗略理解成：

```text
很快返回的 syscall
  -> 可以少让 runtime 参与

可能阻塞的 syscall
  -> 必须通知 runtime
  -> 保存 G 状态
  -> 解绑 P
  -> 返回时重新竞争 P
```

#### LockOSThread和调度器

默认情况下，goroutine 不固定在某个 OS 线程上。一次阻塞、抢占或调度之后，同一个 goroutine 可能在另一个 `M` 上继续执行。这是 Go 调度器能灵活利用线程的基础。

但有些场景依赖线程局部状态，例如：

- 调用需要固定线程的 C 库。
- GUI 框架要求某些操作必须在主线程。
- 使用线程本地存储或特定 OS 线程状态。

这时可以使用：

```go
runtime.LockOSThread()
defer runtime.UnlockOSThread()
```

它会把当前 `G` 和当前 `M` 绑定起来。绑定后调度器仍然可以调度其他 goroutine，但这个 goroutine 不能随意迁移到其他线程，当前线程也会受到更多限制。因此它不是普通并发代码的性能优化手段，只应该在确实依赖线程身份时使用。

#### 问答速览

问题一：Go 调度器的 GMP 是什么？

```text
G 是 goroutine，表示待执行的 Go 任务；M 是 OS 线程，真正执行机器指令；P 是处理器，表示执行 Go 代码需要的调度资源。
M 必须绑定 P 才能执行普通 Go 代码，P 持有本地运行队列、mcache、timer 等资源。
引入 P 的核心价值是减少全局运行队列和全局锁竞争，并通过本地队列、全局队列和工作窃取实现更好的扩展性。
GOMAXPROCS 控制 P 的数量，也基本决定同一时刻最多有多少线程在执行 Go 代码。
```

问题二：goroutine 是怎么被调度的？

```text
go 语句会被编译器改写为 runtime 创建 goroutine 的调用。
runtime 创建 G，准备栈、参数和入口函数，把 G 标记为 _Grunnable 后放入当前 P 的 runnext 或本地运行队列。
M 绑定 P 后进入 schedule 循环，优先从本地队列取 G，也会为了公平性检查全局队列；找不到任务时会查 netpoll，或从其他 P 窃取任务。
找到 G 后，runtime 把它切到 _Grunning，建立 G 和 M 的关系，并通过 gogo 恢复它的执行现场。
当 G 阻塞、退出、主动让出、进入 syscall 或被抢占时，会重新回到调度器选择下一个 G。
```

问题三：系统调用时 P 怎么处理？

```text
goroutine 进入可能阻塞的 syscall 前，runtime 会保存它的 PC/SP，把状态从 _Grunning 改成 _Gsyscall，并让当前 M 和 P 暂时分离，P 进入 _Psyscall 状态。
如果 syscall 很快返回，当前 M 可能直接拿回原 P 或获取空闲 P 继续运行。
如果 syscall 阻塞较久，sysmon 可以夺回这个 P，交给其他 M 执行队列里的 goroutine。
syscall 返回后，如果拿不到 P，当前 G 会被改回 _Grunnable 并放入全局运行队列等待后续调度。
```

## Part 3 Memory & GC

调度器解决的是“谁来运行”的问题，Memory & GC 解决的是“运行时数据放在哪里、什么时候回收”的问题。它们不是两个孤立模块：`P` 持有 `mcache`，goroutine 栈是 GC 根，分配路径会触发 GC assist，GC 又需要调度器停止世界、启动后台标记 worker、抢占 goroutine 扫描栈。

先不要急着记源码名。Memory 这一部分最容易被一串术语劝退，先把后文会反复出现的词铺开。下面这张表不是背诵表，而是阅读索引：第一次看到某个词时，先回来确认它在整张图里的位置。

#### 基础对象和分配词

```text
object / 对象
  程序真正想要的那块数据，例如一个 Node、一个 map 桶、一个 slice 底层数组。

pointer / 指针
  指向另一个对象地址的值。GC 最关心的是对象里哪些字段是指针。

allocation / 分配
  给对象找一块可用内存。可能发生在栈上，也可能发生在堆上。

stack allocation / 栈分配
  对象跟随函数调用栈生命周期，函数返回后自然失效，通常不需要 GC 单独回收。

heap allocation / 堆分配
  对象生命周期可能超过当前函数，放到 Go 堆上，由 GC 判断什么时候回收。

escape analysis / 逃逸分析
  编译器判断对象能否放在栈上的过程。逃逸到函数外、被闭包长期持有、太大或情况复杂时，可能进入堆。

runtime metadata / 运行时元数据
  runtime 用来理解内存的账本，例如类型信息、栈图、堆位图、span 信息。
```

#### 栈相关词

```text
goroutine stack / goroutine 栈
  每个 goroutine 自己的调用栈。初始较小，不够时按需增长，也可能在 GC 时机收缩。

stack frame / 栈帧
  一次函数调用在栈上占用的一段空间，保存参数、局部变量、返回地址等。

SP / stack pointer
  当前栈指针。Go 栈通常向低地址增长，函数调用会让 SP 向 stack.lo 靠近。

PC / program counter
  当前或将要执行的指令位置。调度和栈扫描都需要知道 PC。

stack.lo / stack.hi
  一个 goroutine 当前栈内存的低地址和高地址边界。

stackguard
  栈检查哨兵。函数入口会比较 SP 和 stackguard，发现空间不够就走 morestack。

morestack
  编译器在函数入口栈检查失败时跳到的汇编入口，负责切到 runtime 扩栈流程。

newstack
  runtime 执行栈增长的核心逻辑，决定分配更大的栈并迁移旧内容。

copystack
  把旧栈内容复制到新栈，同时修正栈里指向旧栈地址的指针。

shrinkstack
  在合适时机把过大的 goroutine 栈缩小，避免峰值调用深度长期占内存。

g0 stack / system stack
  每个 M 的 runtime 专用栈。调度、扩栈、GC 等底层逻辑常切到 g0 上执行。

stack map / 栈图
  编译器生成的元数据，告诉 GC 某个安全点上栈帧哪些位置是指针。

safe point / 安全点
  runtime 能正确理解当前栈、寄存器和对象指针的位置。栈增长、抢占、GC 栈扫描都依赖安全点。

root / 根
  GC 标记的起点。goroutine 栈、全局变量、寄存器、runtime 堆外引用都可能是根。

g.sched
  某个 goroutine 被挂起时保存的 PC/SP 现场，可以理解成下次恢复执行的书签。
```

#### 分配器相关词

```text
mallocgc
  Go 堆分配的核心入口。很多 new、make、逃逸对象最终会落到这里。

newobject
  分配一个具体类型对象的 runtime 入口，内部通常会调用 mallocgc。

tiny allocator
  给极小且不含指针的对象用的合并分配优化，减少小碎片和分配次数。

small object / 小对象
  不超过 runtime 小对象上限的分配，会按 size class 从 span 槽位里拿。

large object / 大对象
  超过小对象上限的分配，通常直接从 mheap 拿一个或多个专用 span。

size class
  小对象的规格表。不是 Go 类型，而是 runtime 预设的内存格子大小。
  例如 17B 的对象不能随便切 17B，可能被放进 24B 或 32B 的格子。

spanClass
  size class 再加上 scan/noscan 信息。相同大小但是否含指针不同，会进入不同 spanClass。

scan object
  对象里可能含堆指针，GC 标记时要扫描对象内容。

noscan object
  对象里不含堆指针，例如纯字节数据。GC 可以跳过对象内容，降低扫描成本。

mcache
  每个 P 本地的小对象分配缓存。常见小对象分配先从这里拿，快路径通常不用全局锁。

mcentral
  某个 spanClass 的中心仓库。mcache 没有可用 span 时，向对应 mcentral 补货。

mheap
  runtime 的全局堆管理器。它不是单个数组，而是管理 arena、page、span、central、页分配器等结构的总管。

object slot / 对象槽位
  小对象 span 被切成很多同规格格子，每个格子就是一个可放对象的槽位。

freeindex
  mspan 中下次查找空闲槽位的大致起点，用来加快分配。

allocCache
  mspan 中缓存的一小段空闲位信息，用位运算快速找空槽位。
```

#### 堆结构和位图词

```text
page
  runtime 管理堆时使用的页粒度。span 由连续 page 组成。

span / mspan
  一段连续 heap page。小对象 span 会被切成很多同样大小的槽位。
  mspan 是这段内存的管理账本，记录格子大小、哪些格子已分配、哪些对象被 GC 标记。

arena
  堆地址空间中的一大片区域。可以把它理解成一片仓库园区。

heapArena
  arena 的地图和档案。runtime 用它从一个地址快速查到：它属于哪个 span、哪里有位图。

heap bitmap / 堆位图
  标记和扫描用的位图信息，帮助 runtime 判断对象中哪些位置是指针、哪些对象被标记。

allocBits
  mspan 的分配位图。表示每个对象槽位当前是否已经分配出去。

gcmarkBits
  mspan 的标记位图。表示当前 GC 周期里哪些已分配对象被标记为存活。

sweepgen
  span 的清扫代际编号。runtime 用它判断某个 span 是否已经完成本轮 sweep。

pageInUse
  heapArena 里记录页是否正在使用的元数据。

pageMarks
  heapArena 里记录哪些页上有标记对象的元数据，辅助 GC 和清扫。

scavenger
  后台归还物理内存的机制。它处理的是 runtime 空闲页和 OS 物理内存之间的关系，不等同于 GC 标记对象。
```

#### GC相关词

```text
GC cycle / GC 周期
  从准备标记、并发标记、标记终止到清扫的一轮回收过程。

mutator
  用户程序本身。GC 论文和 runtime 里经常把正在分配对象、修改对象指针关系的用户代码叫 mutator。

STW / stop the world
  暂停所有用户 goroutine，让 runtime 做阶段切换或一致性处理。Go GC 仍有 STW，但通常较短。

mark / 标记
  从 roots 出发，沿指针找到所有仍可达对象，并设置 gcmarkBits。

sweep / 清扫
  GC 标记完以后，把“已分配但没被标记”的对象槽位重新变成可分配。

white / grey / black
  三色标记模型。白色是未发现，灰色是已发现但还没扫描完，黑色是已扫描完成。

work queue / GC 工作队列
  保存待扫描 root job 和灰色对象的队列。mark worker 和 assist 都会从这里取活干。

gcDrain
  消耗 GC 工作队列的核心逻辑：取灰色对象，扫描指针字段，发现新对象再入队。

write barrier / 写屏障
  并发标记期间插入到指针写入附近的逻辑，防止用户代码改指针时 GC 漏标对象。

shade
  写屏障里的概念动作：把某个对象标记成灰色或确保它进入标记流程。

mark worker
  后台执行 GC 标记工作的 goroutine。常见有 dedicated、fractional、idle 几类。

mutator assist
  用户 goroutine 分配太快时，runtime 让它先帮 GC 扫一会儿对象，再继续分配。

assist debt / assist credit
  分配 goroutine 在 GC 标记期间欠下或拥有的“扫描工作账”。欠太多就要进入 assist。

pacer
  GC 节奏控制器。它决定什么时候开始下一轮 GC，以及后台 worker 和分配者要做多少标记工作。

GOGC
  以存活堆为基准控制下一轮 heap goal 的比例参数。默认 100，大致表示目标堆约为存活堆两倍。

GOMEMLIMIT
  Go runtime 的软内存限制。它会影响 pacer，使 GC 更积极，但不是 OS 级硬限制。

heapLive
  当前 runtime 认为仍然活着或已分配的堆大小估计。

heapMarked
  上一轮 GC 标记完成后确认存活的堆大小。

heapGoal
  当前 GC 周期希望控制住的目标堆大小。

trigger
  开始下一轮 GC 的触发点。它通常小于 heapGoal，因为并发标记需要时间。

assistWorkPerByte
  pacer 计算出的比例：每分配一定字节，mutator 大概要帮 GC 做多少扫描工作。
```

#### 调度协作和观测词

```text
G / M / P
  G 是 goroutine，M 是 OS 线程，P 是执行 Go 代码所需的处理器资源。
  内存分配里 P 很重要，因为 P 持有 mcache。

runtime.sched
  全局调度器状态，保存全局运行队列、空闲 M/P 等信息。不要和 g.sched 混淆。

sysmon
  runtime 系统监控线程。它会参与抢占、syscall retake、timer/netpoll、GC 辅助推进等后台维护。

HeapAlloc
  当前仍被 Go 堆对象占用的字节数，接近“业务对象还占着多少堆”。

HeapSys
  runtime 从 OS 获得的堆相关虚拟内存总量，不等于当前活对象大小。

HeapIdle
  runtime 堆里空闲但还没完全归还 OS 的内存。

HeapReleased
  runtime 已经归还或建议归还给 OS 的堆内存。

NextGC
  runtime 估计的下一轮 GC 目标附近的堆大小。

NumGC
  已完成的 GC 周期数。
```

脑子里可以先有这张粗图：

```text
用户代码创建对象
  |
  v
逃逸到堆上
  |
  v
按大小选择 size class
  |
  v
从当前 P 的 mcache 找一个 mspan
  |
  v
mspan 里拿一个空格子放对象
  |
  v
GC 标记时通过 heapArena 和 mspan 元数据识别对象
  |
  v
sweep 时把死亡对象的格子重新标为空闲
```

这一部分可以先建立一张地图：

```text
goroutine
  |
  +-- stack
  |     stackguard / morestack / copystack / shrinkstack
  |
  +-- heap allocation
        |
        v
      mallocgc
        |
        +-- tiny allocator
        +-- small object: mcache -> mcentral -> mheap
        +-- large object: mheap
        |
        v
      span / heapArena metadata
        |
        v
      GC mark & sweep
        |
        +-- root scan: stacks, globals, runtime data
        +-- write barrier
        +-- background mark worker
        +-- mutator assist
        +-- concurrent sweep
        +-- pacer
```

理解这部分时要避免两个误区：

- 栈不是固定大小的一块内存。goroutine 栈可以增长，也可以在合适时机缩小，增长时会复制到新位置。
- GC 不是“等内存满了再停下来全量清理”。现代 Go GC 的主要工作和用户 goroutine 并发执行，只在阶段切换时短暂 STW。

### 10 Goroutine栈管理

goroutine 之所以能大量创建，一个重要原因是它不直接使用 OS 线程那种较大的固定栈。每个 goroutine 有自己的用户态栈，初始大小很小，随着调用深度和局部变量需求按需增长。

源码入口主要在：

```text
runtime/runtime2.go   g.stack、g.stackguard0、g.sched
runtime/stack.go      stackalloc、newstack、copystack、shrinkstack
runtime/asm_*.s       morestack、gogo 等汇编入口
runtime/proc.go       newproc1 创建 goroutine 时分配栈
```

`g` 里和栈、恢复现场直接相关的字段可以抽象为：

```text
g
  stack.lo       当前栈内存低地址
  stack.hi       当前栈内存高地址
  stackguard0    普通 Go 代码的栈检查边界，也可被设置为抢占哨兵值
  stackguard1    系统栈相关检查

  sched.sp       goroutine 暂停时保存的栈指针
  sched.pc       goroutine 暂停时保存的程序计数器
  sched.g        指回当前 goroutine
```

`stack.lo`、`stack.hi` 和 `stackguard` 描述的是“这段栈在哪里、还能不能继续用”；`g.sched` 描述的是“这个 goroutine 被挂起后，下次从哪里恢复”。调度器把 `G` 从一个 `M` 切走时，会把现场保存到 `g.sched`；以后再运行这个 `G`，`gogo` 之类的汇编入口会根据这份现场恢复执行。

注意 `g.sched` 和 `runtime.sched` 不是一回事。前者是单个 goroutine 的恢复书签；后者是全局调度器状态，保存空闲 `M`、空闲 `P`、全局运行队列等信息。

Go 栈通常向低地址增长，所以 `stack.hi` 是栈顶方向，`stack.lo` 是栈底方向。函数调用会消耗更多栈空间，使 SP 向 `stack.lo` 靠近。

#### 函数入口的栈检查

编译器会在很多函数入口插入栈检查。不同架构生成的指令不同，但逻辑可以简化为：

```text
进入函数
  |
  v
计算当前函数需要的栈空间
  |
  v
检查 SP 是否越过 g.stackguard0
  |
  +-- 没越过：继续执行函数体
  |
  +-- 越过：调用 runtime.morestack
              |
              v
            保存当前调用现场
              |
              v
            切到当前 M 的 g0 栈
              |
              v
            runtime.newstack
              |
              +-- 如果这是抢占请求：走抢占路径
              |
              +-- 如果确实空间不足：分配更大栈并复制
```

`morestack` 是汇编入口。它不直接在当前 goroutine 栈上执行复杂逻辑，而是保存现场后切到 `g0` 栈，再进入 `newstack`。这样做的原因是：当前 goroutine 的栈已经可能不够用了，继续在这块栈上跑 runtime 代码本身也可能溢出。

这里有一个关键点：`stackguard0` 不只表示“栈空间不够”。runtime 也会把它设置成特殊值，例如 `stackPreempt`，让下一次函数入口检查失败，从而把 goroutine 引到 `morestack/newstack` 这条路径上处理同步抢占。所以看到 `morestack`，不一定意味着栈真的满了，也可能是调度器想让当前 goroutine 到达安全点。

#### 连续栈：扩容和缩小都靠copystack

Go 早期使用过分段栈，后来改为连续栈。今天的主流模型是：每个 goroutine 持有一段连续栈空间；不够用时复制到更大的栈，太空时也可以复制到更小的栈。扩容和缩小的方向不同，但真正搬栈的核心都是 `copystack`。

先看 `newstack` 的关键代码。下面摘自 `runtime/stack.go`，删掉了 debug 打印和部分错误处理：

```go
func newstack() {
	oldsize := gp.stack.hi - gp.stack.lo
	newsize := oldsize * 2

	if f := findfunc(gp.sched.pc); f.valid() {
		max := uintptr(funcMaxSPDelta(f))
		needed := max + stackGuard
		used := gp.stack.hi - gp.sched.sp
		for newsize-used < needed {
			newsize *= 2
		}
	}

	casgstatus(gp, _Grunning, _Gcopystack)
	copystack(gp, newsize)
	casgstatus(gp, _Gcopystack, _Grunning)
	gogo(&gp.sched)
}
```

这段代码说明几件事：

- 扩容默认从 `oldsize * 2` 开始。
- 如果当前函数帧需要更多空间，就继续翻倍。
- 搬栈前把 goroutine 状态切到 `_Gcopystack`，避免并发 GC 同时扫描这段正在移动的栈。
- 真正复制和修正指针的是 `copystack`。
- 完成后通过 `gogo(&gp.sched)` 回到 goroutine 的执行现场。

为什么复制栈是可行的？关键是编译器会为安全点生成栈图。栈图可以粗略理解成“这个函数帧里哪些槽位是指针”的位图：

```text
frame f, safe point pc=...

slot        内容示例          栈图
SP+0        返回地址          -
SP+8        x int             scalar
SP+16       p *int            pointer
SP+24       n *Node           pointer
SP+32       count int         scalar
```

实际 runtime 里的栈图不是这样的人类表格，而是压缩后的位图和 PC 元数据；但读代码时可以把它想成这张表。

栈搬家时，`copystack` 先复制字节，再根据栈图修正“指向旧栈内部”的指针：

```text
例子：p := &x

复制前 old stack        只复制后 new stack        修正后 new stack
0x1000 x = 10           0x3000 x = 10             0x3000 x = 10
0x1008 p = 0x1000  -->  0x3008 p = 0x1000   -->   0x3008 p = 0x3000
        指向旧 x                 错：还指向旧栈           对：指向新 x
```

这就是“栈可以搬家”的关键：runtime 不猜哪些字节像地址，而是靠栈图知道哪些槽位是指针，再只修正落在旧栈范围内的指针。

`copystack` 的关键代码更直接。下面同样是关键摘录，删掉了调试和非主线分支：

```go
func copystack(gp *g, newsize uintptr) {
	if gp.syscallsp != 0 {
		throw("stack growth not allowed in system call")
	}
	old := gp.stack
	used := old.hi - gp.sched.sp
	new := stackalloc(uint32(newsize))

	var adjinfo adjustinfo
	adjinfo.old = old
	adjinfo.delta = new.hi - old.hi

	ncopy := used
	if !gp.activeStackChans {
		adjustsudogs(gp, &adjinfo)
	} else {
		adjinfo.sghi = findsghi(gp, old)
		ncopy -= syncadjustsudogs(gp, used, &adjinfo)
	}

	memmove(unsafe.Pointer(new.hi-ncopy), unsafe.Pointer(old.hi-ncopy), ncopy)

	adjustctxt(gp, &adjinfo)
	adjustdefers(gp, &adjinfo)
	adjustpanics(gp, &adjinfo)

	gp.stack = new
	gp.stackguard0 = new.lo + stackGuard
	gp.sched.sp = new.hi - used

	for u.init(gp, 0); u.valid(); u.next() {
		adjustframe(&u.frame, &adjinfo)
	}
	stackfree(old)
}
```

这段代码比文字流程图更能说明问题：

- `used := old.hi - gp.sched.sp` 算出旧栈真正用到的部分。
- `adjinfo.delta = new.hi - old.hi` 是新旧栈地址差，后面所有指针修正都靠它。
- `adjustsudogs` / `syncadjustsudogs` 处理 channel 等待记录里的栈指针。
- `adjustdefers`、`adjustpanics` 处理 defer/panic 链里的栈指针。
- `adjustframe` 根据栈图遍历每个栈帧，修正栈帧里的指针。
- `gp.stack`、`gp.stackguard0`、`gp.sched.sp` 最后切到新栈。

所以 channel、defer、panic 的问题不是“缩栈专属问题”，而是所有栈搬迁都必须面对的问题。扩容和缩小都应该从 `copystack` 这里统一理解。

#### 栈分配与栈缓存

goroutine 栈不是用普通堆对象的形式直接交给用户代码管理。`stackalloc` 会在系统栈上运行，并根据栈大小选择不同来源：

```text
小栈
  |
  +-- 优先从当前 P 的 mcache.stackcache 获取
  +-- 不足时从全局 stackpool 补充

大栈
  |
  +-- 从 stackLarge 缓存获取
  +-- 不足时向 mheap 申请 manual span
```

这里又能看到调度器和内存管理的关系：`P` 不只持有运行队列，也持有分配相关缓存，包括普通小对象的 `mcache` 和小栈缓存。这样可以避免每次创建 goroutine 都争用全局堆锁。

#### 栈缩小

栈能增长，也能缩小。一个 goroutine 曾经递归很深或调用过大栈帧函数，不代表它之后永远占着那块大栈。GC 扫描栈时可以发现栈使用量，runtime 会在安全时机尝试缩小栈。

缩栈比扩栈多一层安全检查。`isShrinkStackSafe` 的关键代码是：

```go
func isShrinkStackSafe(gp *g) bool {
	if gp.syscallsp != 0 {
		return false
	}
	if gp.asyncSafePoint {
		return false
	}
	if gp.parkingOnChan.Load() {
		return false
	}
	if readgstatus(gp)&^_Gscan == _Gwaiting &&
		gp.waitreason.isWaitingForSuspendG() {
		return false
	}
	return true
}
```

这里的重点不是“channel 一定不能缩栈”，而是缩栈必须在 runtime 能精确掌握指针和同步状态时发生。`syscall`、异步安全点、正在 channel parking 的窗口，也就是 goroutine 正在把自己挂入 channel 等待队列的窗口，都可能让指针信息或同步状态不够可靠。

真正决定缩不缩的是 `shrinkstack`：

```go
func shrinkstack(gp *g) {
	if !isShrinkStackSafe(gp) {
		throw("shrinkstack at bad time")
	}
	if debug.gcshrinkstackoff > 0 {
		return
	}

	oldsize := gp.stack.hi - gp.stack.lo
	newsize := oldsize / 2
	if newsize < fixedStack {
		return
	}

	avail := gp.stack.hi - gp.stack.lo
	if used := gp.stack.hi - gp.sched.sp + stackNosplit; used >= avail/4 {
		return
	}

	copystack(gp, newsize)
}
```

这段代码和 `newstack` 对照看就很清楚：

```text
扩栈 newstack:
  oldsize * 2
  当前帧放不下就继续翻倍
  copystack(gp, bigger)

缩栈 shrinkstack:
  oldsize / 2
  小于 fixedStack 不缩
  已使用 >= 1/4 不缩
  copystack(gp, smaller)
```

所以缩栈和扩栈不是两套搬迁逻辑。它们只是计算 `newsize` 的策略不同，最后都回到 `copystack`。

#### 栈与GC根扫描

GC 判断堆对象是否存活时，需要从根对象出发。goroutine 栈就是最重要的根之一。栈上局部变量如果保存了堆指针，那么这个堆对象必须被认为可达。

栈扫描大致需要：

```text
找到一个 goroutine
  |
  v
让它处在可扫描的稳定状态
  |
  v
沿调用栈遍历栈帧
  |
  v
根据函数栈图找出指针槽位
  |
  v
把槽位里的堆指针标记为灰色对象
```

栈越大、栈上的指针越多，GC 根扫描成本越高。所以 goroutine 虽然轻量，但不是没有成本。大量 goroutine 如果都持有很深的栈或很多指针，仍然会增加 GC 工作量。

#### g0、gsignal和用户栈

不是所有 goroutine 栈都一样。每个 `M` 通常有：

```text
g0       当前 OS 线程的调度栈，执行 runtime 调度、栈增长、系统级逻辑
gsignal  信号处理相关栈
普通 G   用户 goroutine 栈
```

普通 Go 函数运行在用户 goroutine 栈上；调度器、栈分配、栈复制等关键 runtime 路径通常要切到 `g0` 栈上执行。`g0` 不像普通 goroutine 那样参与用户代码调度，也不能随便执行可能增长栈的逻辑。

可以把三者区别记成：

| 栈 | 主要用途 | 是否运行普通用户代码 |
| --- | --- | --- |
| 普通 G 栈 | 执行业务函数、保存调用帧 | 是 |
| g0 栈 | 调度、栈管理、mcall/systemstack 逻辑 | 否 |
| gsignal 栈 | 信号处理 | 否 |

#### 栈管理小结

goroutine 栈管理的核心不是“初始栈很小”这一句话，而是一整套动态机制：

```text
编译器插入栈检查
  -> runtime.morestack 进入栈增长或抢占路径
  -> stackalloc 分配新栈
  -> copystack 复制并修正指针
  -> GC 扫描栈作为根
  -> 合适时机 shrinkstack 缩小栈
```

这套机制让 goroutine 创建成本低于 OS 线程，但代价是编译器、调度器、GC 和栈复制逻辑必须紧密协作。

### 11 内存分配器

Go 代码里写下的分配并不一定都会进入堆分配器。编译器会先做逃逸分析：

```go
func f() *int {
	x := 1
	return &x
}
```

这里 `x` 的地址逃出了函数，通常需要分配到堆上。相反，如果对象只在当前函数内部使用，编译器可能把它放在栈上，甚至完全优化掉。

先看一个具体对象怎么进入堆。假设有一个 24 字节左右的 `Node`：

```go
type Node struct {
	next *Node
	val  int
}

func makeNode(v int) *Node {
	return &Node{val: v}
}
```

如果 `&Node{}` 逃逸到堆上，runtime 大概会经历这条路径：

```text
1. 需要分配一个 Node
   |
   v
2. 计算对象大小，大约 24B
   |
   v
3. 查 size class 表，选择能装下 24B 的规格
   |
   v
4. 因为 Node 里有 next *Node，所以选择 scan 类型的 spanClass
   |
   v
5. 找当前 P 的 mcache.alloc[spanClass]
   |
   v
6. 找到一个 mspan，它里面都是同规格格子
   |
   v
7. 从 mspan 的空闲格子取一个，设置 allocBits
   |
   v
8. 返回对象地址给用户代码
```

这几个词在这个例子里分别是什么：

```text
size class
  “24B 或更大一点的格子规格”

spanClass
  “格子规格 + 这个格子里的对象要不要被 GC 扫描”

mspan
  “一整排同规格格子，以及这排格子的账本”

mcache
  “当前 P 手边的一小批可用格子，分配时先从这里拿”

mcentral
  “某种规格格子的中转仓库，mcache 没货时来这里补”

mheap
  “总仓库，mcentral 也没货时从这里切新的 span”
```

所以 `span` 不是某个对象，也不是 Go 类型。它更像一排货架：这排货架的每个格子大小一样，里面可以放同一规格的小对象。`size class` 则是货架格子的规格表。

一旦对象确实需要进入堆，分配动作就会落到 runtime。常见入口包括：

```text
runtime.newobject       new(T)、&T{} 等对象分配
runtime.makeslice       slice 底层数组分配
runtime.makemap         map 初始化
runtime.makechan        channel 初始化
runtime.mallocgc        堆分配核心入口
```

分配器要同时满足几个目标：

- 常见小对象分配要快，最好不加全局锁。
- 内存碎片要可控。
- 分配元数据要能支持 GC 精确扫描。
- 向操作系统申请内存要批量化，不能每个对象都系统调用。
- GC 并发标记期间，新分配对象要维持标记不变量。

#### 分配器层级

Go 分配器借鉴了 tcmalloc 的分层思想，但 runtime 源码已经演化出自己的实现。整体层级可以画成：

```text
mallocgc
  |
  +-- tiny allocator      极小且不含指针的对象
  |
  +-- small object        小对象，按 size class 分配
  |     |
  |     v
  |   P.mcache            每个 P 的本地 span 缓存，无锁快路径
  |     |
  |     v
  |   mcentral            每个 spanClass 的中心 span 集合
  |     |
  |     v
  |   mheap               全局页级堆
  |
  +-- large object        大对象，直接向 mheap 申请 span
        |
        v
      OS memory           必要时向操作系统申请更多页
```

源码注释里把小对象定义为不超过 32KB 的分配。小对象会被向上取整到某个 size class。每个 size class 对应固定大小的槽位，同一个 span 里的对象大小相同。

例如要分配一个 24 字节对象，runtime 可能把它放进 24 或 32 字节的 size class 中，然后从当前 `P.mcache` 对应 span 里找一个空槽位。

#### mcache：每个P的快速缓存

`mcache` 是每个 `P` 持有的小对象分配缓存。它的关键特点是：当前 `M` 绑定着 `P` 执行 Go 代码时，可以直接访问这个 `P` 的 `mcache`，常见路径不需要全局锁。

可以简化为：

```text
mcache
  alloc[spanClass]    每个 spanClass 当前可分配的 mspan
  tiny                tiny allocator 当前块
  tinyoffset          tiny 块内偏移
  stackcache          小栈缓存
  scanAlloc           本 P 累计的可扫描分配量
  flushGen            与 sweepgen 协作，判断缓存 span 是否过期
```

`spanClass` 不是单纯的 size class，它还包含“对象是否需要扫描”的信息。相同大小的对象，如果一种包含指针、一种不包含指针，runtime 会把它们放进不同的 span class。这样 GC 扫描时可以跳过整块不含指针的 span。

```text
spanClass = sizeClass + scan/noscan
```

可以把 `spanClass` 想成二维坐标，而不是一个单独数字：

```text
                 是否需要 GC 扫描
              +-----------+-------------+
              | scan      | noscan      |
+-------------+-----------+-------------+
| 16B class   | 16B/scan  | 16B/noscan  |
| 24B class   | 24B/scan  | 24B/noscan  |
| 32B class   | 32B/scan  | 32B/noscan  |
+-------------+-----------+-------------+
     大小规格
```

例如同样是 24B 左右：

```text
struct { next *Node; val int }   -> 24B/scan
[3]uint64                        -> 24B/noscan
```

这个设计很重要。`[]byte`、字符串底层字节、纯数字结构体这类不含指针的数据，如果被放进 noscan span，GC 就不需要逐字扫描它们。

#### mcentral：按spanClass管理span

当 `mcache` 里的某个 span 用满了，它会向 `mcentral` 换一个有空位的 span。`mcentral` 按 `spanClass` 划分，每个 class 有自己的 span 集合。

`mcentral` 自身并不保存每个空闲对象，它管理的是一批 `mspan`：

```text
mcentral(spanClass=N)
  |
  +-- partial: 还有空槽位的 span
  |
  +-- full:    当前没有空槽位的 span
```

源码中还把这些集合分成 swept 和 unswept 两组，用来配合并发清扫。GC 结束后，并不是所有 span 都立刻被清扫完；分配器在拿 span 时可能顺手清扫需要的 span，这叫 lazy sweeping。

#### mheap：全局页级堆

如果 `mcentral` 也拿不到合适的 span，就要向 `mheap` 申请页。`mheap` 是全局堆管理器，负责：

- 管理页级分配器。
- 维护所有 size class 的 `mcentral`。
- 维护 heap arena 元数据。
- 向操作系统申请或归还内存。
- 管理 span 的生命周期和 sweep 状态。
- 为大对象直接分配 span。

大对象不经过 `mcache/mcentral`，会直接走 `mheap`：

```text
large object
  |
  v
计算需要多少页
  |
  v
mheap.alloc
  |
  v
得到一个专用 mspan
```

这样做的原因是大对象本身已经足够大，不适合塞进小对象 size class 的槽位里；直接按页分配更简单，也减少内部碎片。

#### mallocgc主流程

`mallocgc` 是堆分配的核心入口。它不是只做“找一块内存”这么简单，还要和 GC、类型元数据、内存清零、profile、race/asan/msan 等机制协作。

这里的几个源码名先简单对齐一下：`zerobase` 是零大小对象共用的哑地址；`gcBlackenEnabled` 表示当前是否处于需要把对象标记变黑的 GC 标记阶段；race/asan/msan 是数据竞争和内存错误检测工具的插桩路径。

简化流程如下：

```text
mallocgc(size, typ, needzero)
  |
  +-- size == 0：返回 zerobase
  |
  +-- 如果 GC 正在标记：扣除当前 G 的 assist credit
  |
  +-- 根据 size 和 typ.Pointers() 选择分配路径
  |     |
  |     +-- tiny: 极小 noscan 对象合并进 tiny block
  |     +-- small noscan: size class + noscan span
  |     +-- small scan: size class + scan span + 类型/bitmap 元数据
  |     +-- large: mheap 直接分配
  |
  +-- 必要时清零
  |
  +-- 发布前保证对象和 heap bits 初始化完成
  |
  +-- 如果写屏障开启：新对象直接标黑
  |
  +-- 记录 profile / sanitizer / 统计信息
  |
  +-- 如果达到触发条件：gcStart
```

`typ` 是否包含指针会影响分配路径。含指针对象需要让 GC 知道哪些字段是指针；不含指针对象则可以进入 noscan 路径，降低后续扫描成本。

#### tiny allocator

tiny allocator 处理极小且不含指针的对象。它会把多个小对象合并到一个小块里，减少分配次数和元数据开销。

可以把它看成一个小拼盘：

```text
tiny block

+------+------+--------+---------+
|  3B  |  5B  |   2B   |  free   |
+------+------+--------+---------+
   a      b       c

tinyoffset 指向下一个可放入的位置
```

典型目标包括：

- 很小的字符串相关对象。
- 逃逸的独立小变量。
- 不含指针的小结构体。

约束也很清楚：tiny 对象必须是 noscan。因为多个对象会共享一个 tiny block，只要其中一个子对象还活着，整个 block 就不能回收。如果对象里有指针，GC 精确性和浪费边界都会变复杂。

#### 分配和GC的耦合

分配器和 GC 的关系非常紧：

```text
分配器需要 GC：
  - 知道什么时候触发下一轮 GC
  - 在并发标记时执行 assist
  - 分配期间维护对象颜色
  - sweep 后复用空槽位和空 span

GC 需要分配器：
  - 通过 span 找到对象大小和边界
  - 通过 bitmap/type 信息知道哪些字段是指针
  - 通过 allocBits/gcmarkBits 区分已分配和已标记对象
  - 通过 mheap/arena 元数据从地址反查对象
```

所以 Go 的内存分配器不是普通库函数。它是 runtime 的中心模块之一，调度器、GC、栈管理、类型系统都会和它发生关系。

### 12 Span、Heap与Arena

理解 Go 堆，最好从三个层次看：

```text
对象 object
  |
  v
span: 一段连续页，切成同大小对象槽位，或专门服务一个大对象
  |
  v
heap: runtime 管理的页级堆，由 mheap 统筹
  |
  v
arena: 按大块虚拟地址划分的堆区域和元数据索引
```

用一个不严谨但好记的类比：

```text
arena
  一片仓库园区，代表一大段连续的堆地址空间。

heapArena
  这片园区的地图。地图本身不是货物，它记录每块地属于哪排货架、哪里有标记位图。

page
  园区里的地砖。runtime 切 span 时按连续地砖来切。

span
  一排连续货架。小对象 span 会把货架切成很多同规格格子。

object
  放进格子里的货物。用户代码真正拿到的是 object 的地址。
```

如果画成内存布局，大概是：

```text
arena address range

+------------------------------------------------------------------+
| page | page | page | page | page | page | page | page | page ... |
+------------------------------------------------------------------+
  \______________/   \____________________/   \__________________/
      span A                span B                    span C
   16B objects           32B objects              large object

heapArena metadata

+------------------------------+
| page -> span mapping         |
| alloc / mark bitmap metadata |
| page in-use information      |
+------------------------------+
```

注意，`heapArena` 不是“堆里的另一块业务内存”。它更像 runtime 放在旁边的索引表。GC 拿到一个地址时，要靠这些索引快速判断这个地址是不是 Go 堆对象、对象边界在哪里、对应哪个 span。

#### page与span

Go runtime 的堆按页管理。源码注释中 `mheap` 以 8192 字节页粒度管理堆。`mspan` 表示一段连续页：

```text
mspan
  startAddr    span 起始地址
  npages       包含多少页
  elemsize     每个对象槽位大小
  nelems       当前 span 可容纳多少对象
  spanclass    size class + scan/noscan
  allocBits    对象槽位是否已分配
  gcmarkBits   当前 GC 周期对象是否被标记
  freeindex    下次查找空槽位的起点
  allocCache   加速查找空槽位的位缓存
  sweepgen     和 mheap.sweepgen 配合判断清扫状态
  state        mSpanInUse / mSpanManual / mSpanDead
```

`state` 可以先这样理解：

```text
mSpanInUse    普通堆对象正在使用的 span
mSpanManual   runtime 手动管理的 span，例如某些栈内存，不按普通堆对象分配
mSpanDead     空闲或未使用的 span
```

小对象 span 会被切成固定大小的槽位：

```text
span for 32-byte objects

+----+----+----+----+----+----+----+----+
| 32 | 32 | 32 | 32 | 32 | 32 | 32 | 32 |
+----+----+----+----+----+----+----+----+
  ^         ^
  allocated free
```

更接近分配器视角时，它不是看“格子画得漂不漂亮”，而是看位图：

```text
slot:       0 1 2 3 4 5 6 7
allocBits:  1 1 0 1 0 0 1 0
freeindex:      ^
                从这里附近继续找空槽

allocCache:
  缓存后续一段槽位的空闲情况，方便用位运算快速找到下一个 0
```

`freeindex` 像书签，避免每次都从 slot 0 开始找空位；`allocCache` 像一小段预读缓存，让分配器少扫内存、多用位运算。

大对象 span 则通常是一整个对象占用一个或多个页：

```text
large object span

+--------------------------------------+
|            one large object           |
+--------------------------------------+
```

#### allocBits与gcmarkBits

`allocBits` 和 `gcmarkBits` 是理解 sweep 的关键。

```text
allocBits:   哪些槽位已经被分配
gcmarkBits:  本轮 GC 标记阶段哪些槽位可达
```

标记结束后：

- `gcmarkBits=1` 的对象仍然存活。
- `allocBits=1` 但 `gcmarkBits=0` 的对象不可达，可以回收。

清扫 span 时，runtime 会根据标记结果更新可分配状态。一个简化视角是：

```text
GC mark phase:
  标记活对象 -> gcmarkBits

sweep phase:
  对比 allocBits 和 gcmarkBits
  未标记对象变成空闲槽位
  为下一轮 GC 准备新的 bitmap
```

这就是为什么 span 必须知道自己的对象大小、对象数量、位图和 sweep generation。GC 不是拿着一个裸地址就能回收内存，它要通过 span 元数据理解这段地址的结构。

#### sweepgen

`mheap` 有一个全局 `sweepgen`，`mspan` 也有自己的 `sweepgen`。它们配合表示 span 是否需要清扫、是否正在清扫、是否已经清扫并可用。

可以不用记住每个数值，只要记住：

```text
span.sweepgen 落后于 mheap.sweepgen
  -> 这个 span 属于上一轮结果，还没清扫

span.sweepgen 等于 mheap.sweepgen
  -> 已清扫，可以安全使用

span 被 mcache 缓存
  -> 会用特殊 generation 表示，避免被并发 sweeper 误处理
```

用表格看更像一个“清扫进度戳”：

```text
mheap.sweepgen = 42

span A  sweepgen=40   未清扫，需要 sweep
span B  sweepgen=41   正在被某个 sweeper 处理
span C  sweepgen=42   已清扫，可以分配
span D  mcache 持有   暂时归本地缓存管理
```

具体数值不是重点，重点是 runtime 不靠“我记得扫过它”这种模糊状态，而是用 generation 判断每个 span 在当前 GC 周期里走到哪一步。

这个设计让清扫可以并发、懒惰地进行。分配器拿 span 前先确认它已经清扫；后台 sweeper 也可以逐个 span 清扫。

#### heapArena

Go 堆的虚拟地址空间被划分为 arena。源码注释中，heap arena 在 64 位平台通常是 64MB，在 32 位平台通常是 4MB。每个 arena 有一个对应的 `heapArena` 元数据对象，存放这块地址范围的索引和位图。

可以抽象为：

```text
virtual address space

+----------+----------+----------+----------+
| arena 0  | arena 1  | arena 2  | arena 3  |
+----------+----------+----------+----------+
     |          |          |
     v          v          v
 heapArena  heapArena  heapArena
```

`heapArena` 里重要的元数据包括：

```text
spans        arena 内每个 page 对应哪个 mspan
pageInUse    哪些 span 处于使用状态
pageMarks    哪些 span 有标记对象
pageSpecials 哪些 span 有 finalizer 等 special 记录
bitmap       堆指针位图相关信息
```

这里的 special 记录指 runtime 挂在对象旁边的额外记录，例如 finalizer、profile 相关信息等；它不是对象本体字段，但 GC 和回收时需要看见它。

堆位图可以粗略想成“对象字段扫描表”。例如：

```text
type Node struct {
	next *Node
	val  int
}

object Node in heap

offset 0   next pointer   bitmap=1   GC 要沿着它继续标记
offset 8   val  int       bitmap=0   普通整数，跳过
```

实际实现会结合类型元数据、span 信息和 bitmap 编码；这里的重点是：GC 扫描堆对象时也不是盲扫字节，而是知道哪些位置可能是指针。

这些元数据本身通常在 Go 堆外管理，因为 GC 需要在扫描堆对象时依赖它们，不能让它们像普通对象一样被移动或回收。

#### 地址如何反查对象

GC 扫描时经常拿到一个地址，需要判断它是不是 Go 堆指针、属于哪个对象、对象边界在哪里。这个过程可以简化为：

```text
address p
  |
  v
计算 arena index
  |
  v
mheap_.arenas 找到 heapArena
  |
  v
根据 page index 找到 mspan
  |
  v
根据 span.elemsize 计算对象起始地址和槽位编号
  |
  v
检查 allocBits / gcmarkBits / 类型信息
```

这也是为什么堆被组织成 arena、page、span 多级结构。它不是为了让概念复杂，而是为了让 runtime 能快速从任意地址定位到对象元数据。

#### spanClass和noscan

同一个 size class 会被拆成 scan 和 noscan 两类 span：

```text
size class 24 bytes
  |
  +-- scan span:   对象可能包含指针，GC 要扫描
  |
  +-- noscan span: 对象不包含指针，GC 可跳过对象内容
```

这对性能影响很大。假设程序分配了一个 100MB 的 `[]byte` 底层数组，它可能占用很多内存，但不含堆指针。GC 不需要把这 100MB 当成指针数组逐字检查。真正影响标记成本的是“可扫描内存”和“指针数量”，不只是堆占用字节数。

#### mheap中的central数组

`mheap` 持有所有 `mcentral`：

```text
mheap
  central[spanClass 0]
  central[spanClass 1]
  central[spanClass 2]
  ...
```

每个 `mcentral` 只服务一个 `spanClass`。当某个 `P.mcache` 需要新的 span 时，会通过 `mheap_.central[spc].mcentral.cacheSpan()` 获取。

关系可以画成：

```text
P0.mcache.alloc[spc] ----\
                         \
P1.mcache.alloc[spc] -----> mheap.central[spc] -> mheap pages -> OS
                         /
P2.mcache.alloc[spc] ----/
```

`mcache` 让常见分配无锁，`mcentral` 让同类 span 集中管理，`mheap` 负责跨 class 的全局页资源。三层结构共同降低锁竞争和碎片。

### 13 GC原理

Go 当前 GC 可以概括为：

```text
并发
精确
三色标记
标记-清扫
非分代
非压缩
依赖写屏障
```

这里的“精确”表示 runtime 知道哪些位置是指针，哪些不是指针；它不会把任意整数都当作指针。精确性的基础来自编译器生成的类型信息、栈图、堆 bitmap 和 runtime 元数据。

#### 为什么需要GC

堆对象的生命周期不一定和函数调用栈一致：

```go
func newNode(v int) *Node {
	return &Node{Value: v}
}
```

`Node` 在函数返回后仍然可能被外部使用，不能随着栈帧一起释放。手动管理这类对象容易出现重复释放、悬垂指针和内存泄漏。Go 选择自动 GC，让 runtime 根据可达性回收不再使用的堆对象。

GC 要回答的问题是：

```text
从程序还能直接访问到哪些对象？
从这些对象继续沿指针能访问到哪些对象？
剩下访问不到的对象占用的内存能否复用？
```

#### 根对象

标记从根对象开始。Go GC 的根大致包括：

- 各 goroutine 栈上的指针。
- 全局变量和包级变量里的指针。
- 寄存器和调用现场中保存的指针。
- runtime 自己维护的一些堆外结构中的堆指针。
- finalizer、special record 等特殊引用。

根对象不是“要回收的对象”，而是“搜索存活对象的起点”。

```text
roots
  |
  +-- goroutine stacks
  +-- globals
  +-- registers
  +-- runtime off-heap references
```

根对象展开后，GC 看到的是一张对象图：

```text
stack slot ----> A ----> B
global var ----> C ----> D
register  -----> E

X ----> Y   没有任何 root 能到达，标记结束后可以被 sweep 回收
```

#### 三色标记

三色标记是一种理解标记过程的模型：

```text
white  尚未发现，标记结束后仍是 white 就会被回收
grey   已发现，但它引用的对象还没扫描完
black  已发现，并且它引用的对象也已经处理过
```

标记过程可以简化为：

```text
初始：所有对象 white
  |
  v
把根引用到的对象标成 grey，放入工作队列
  |
  v
不断取出 grey 对象
  |
  v
扫描它的指针字段，把发现的 white 对象标成 grey
  |
  v
当前对象扫描完，标成 black
  |
  v
工作队列为空，标记结束
```

如果世界完全停止，这个算法很直观。但 Go 的目标是降低暂停时间，因此标记主要和用户 goroutine 并发执行。并发之后，问题变复杂了：用户代码可能在 GC 标记期间修改对象引用关系。

#### 并发标记的问题

假设 GC 已经把对象 `A` 扫描成 black，之后用户 goroutine 执行：

```text
A.ptr = B
```

如果 `B` 原来是 white，而 GC 不知道这次写入，就可能在标记结束时误以为 `B` 不可达，从而回收仍然被 `A` 引用的对象。

危险画面是这样的：

```text
GC 已扫描完成 A

black A  ----new pointer---->  white B
   ^                              ^
   |                              |
不会再扫描 A                 B 还没被发现
```

写屏障要做的事，就是在这条新边出现时把 `B` 放进标记流程，避免它一直保持 white。

并发标记要维持一个核心不变量：不能让黑色对象悄悄指向未被发现的白色对象。写屏障就是为这个问题服务的。它在指针写入时通知 GC，把相关对象标记或放入工作队列。

#### 标记和清扫

Go GC 是 mark-sweep，不是 copying GC。标记阶段只判断对象是否可达，不移动对象；清扫阶段回收未标记对象占用的槽位或 span。

```text
mark:
  roots -> reachable objects
  设置 gcmarkBits

sweep:
  对比 allocBits 和 gcmarkBits
  未标记对象变成空闲
  空 span 归还给 mheap
  部分空闲 span 回到 mcentral
```

把 sweep 放到 span 里看会更清楚。假设一个 span 被切成 8 个对象槽位：

```text
GC 标记前，allocBits 表示哪些格子当前被分配：

slot:      0 1 2 3 4 5 6 7
allocBits: 1 1 1 1 0 1 0 1

GC 标记后，gcmarkBits 表示哪些已分配对象仍然可达：

slot:       0 1 2 3 4 5 6 7
gcmarkBits: 1 0 1 0 0 1 0 0
```

sweep 做的事就是对账：

```text
slot 0: alloc=1 mark=1  活着，保留
slot 1: alloc=1 mark=0  死了，变成空闲
slot 2: alloc=1 mark=1  活着，保留
slot 3: alloc=1 mark=0  死了，变成空闲
slot 5: alloc=1 mark=1  活着，保留
slot 7: alloc=1 mark=0  死了，变成空闲
```

清扫后，这个 span 不是被“删除”，而是多出了一些空格子，可以继续给同一个 size class 的对象复用。如果整个 span 的对象都死了，它才可能整体回到 `mheap`，以后被改造成别的用途。

因为对象不移动，所以 Go 不做堆压缩。优点是对象地址稳定，和 `unsafe`、cgo、系统调用等场景协作更简单；缺点是需要靠 size class、span 管理和页回收控制碎片。

#### STW与并发

Go GC 不是完全无暂停。它仍然需要短暂 STW 来完成阶段切换：

```text
sweep termination:
  STW，结束上一轮残留 sweep，准备进入 mark

mark:
  大部分与用户 goroutine 并发执行

mark termination:
  STW，确认标记完成，关闭标记相关机制，准备 sweep

sweep:
  与用户 goroutine 并发执行
```

GC 的优化重点不是把 STW 变成绝对 0，而是把长时间工作移到并发阶段，让暂停时间主要由阶段切换、根处理准备、统计更新等短任务组成。

#### GC触发

最常见的触发来自堆增长。`GOGC` 控制下一轮 GC 的目标堆大小，默认值是 100。一个简化公式是：

```text
下一轮目标堆大小 ≈ 上一轮存活堆大小 * (1 + GOGC / 100)
```

例如上一轮 GC 后存活堆是 100MB，`GOGC=100`，那么目标堆大小大约是 200MB。为了让并发标记有时间完成，GC 不会等到真正达到目标才开始，而是由 pacer 提前计算 trigger。

触发来源可以概括为：

```text
heap trigger    堆增长达到 pacer 计算的触发点
time trigger    长时间没有 GC 时的周期性触发
cycle trigger   runtime.GC 等强制触发
memory limit    GOMEMLIMIT 软内存限制影响 heap goal
```

#### GC成本看什么

直觉上很多人只看堆大小，但 GC 成本更接近：

```text
GC标记成本 ≈ 根扫描成本 + 可扫描堆对象成本 + 指针边数量成本
```

几个结论：

- 大量 `[]byte` 占内存，但本身不含指针，标记扫描成本相对低。
- 大量小对象如果彼此用指针连接，标记成本可能高。
- 大量 goroutine 栈会增加根扫描成本。
- 对象分配越快，GC 越需要更早开始，或者让分配者 assist。
- 降低临时堆分配，通常比单纯调 `GOGC` 更根本。

### 14 GC实现

GC 的主流程在 `runtime/mgc.go`。源码开头已经给出了高层算法：并发 mark and sweep，使用 write barrier，非分代，非压缩，小对象通过每 P 的分配区域降低锁竞争。

源码入口可以先看这些：

```text
runtime/mgc.go        gcStart、gcMarkDone、gcSweep、GC phase
runtime/mgcmark.go    gcDrain、gcAssistAlloc、mark worker
runtime/mgcpacer.go   gcController、heapGoal、trigger、assist ratio
runtime/mbarrier.go   write barrier 说明和 bulk barrier
runtime/mbitmap.go    heap bitmap、findObject、bulkBarrierPreWrite
runtime/mheap.go      span sweepgen、allocBits、gcmarkBits
```

#### GC状态

Go runtime 用 `gcphase` 表示当前 GC 阶段，核心状态可以简化为：

```text
_GCoff
  GC 标记未运行
  write barrier 关闭
  后台 sweep 可以进行

_GCmark
  并发标记中
  write barrier 开启
  新对象分配为 black
  background mark worker 和 mutator assist 工作

_GCmarktermination
  标记终止阶段
  STW
  write barrier 仍开启
  做收尾、统计、切换到 sweep
```

#### 一轮GC的源码流程

从 `gcStart` 开始，一轮 GC 可以抽象为：

```text
1. 触发检查
   gcTriggerHeap / gcTriggerTime / gcTriggerCycle

2. sweep termination
   stopTheWorld
   finishsweep_m
   清理上一轮未完成 sweep

3. 准备 mark
   gcController.startCycle
   setGCPhase(_GCmark)
   gcBgMarkPrepare
   gcPrepareMarkRoots
   gcMarkTinyAllocs
   开启 gcBlackenEnabled

4. startTheWorld
   用户 goroutine 继续运行
   写屏障已经开启

5. concurrent mark
   background mark workers 扫描对象
   mutator assist 在分配时帮忙标记
   root marking 扫描栈、全局变量、runtime 数据
   gcDrain 消耗灰色对象队列

6. mark done
   检测没有 root job 和 grey object
   进入 mark termination

7. mark termination
   stopTheWorld
   setGCPhase(_GCmarktermination)
   关闭 assist 和 worker
   更新统计信息
   刷新 mcache 等状态

8. 准备 sweep
   setGCPhase(_GCoff)
   gcSweep 设置 sweepgen
   唤醒后台 sweeper

9. startTheWorld
   用户 goroutine 继续运行
   sweep 与分配并发进行
```

这个流程里两次 STW 很关键：一次在进入标记前，保证所有 `P` 都能看到写屏障已经开启；一次在标记完成后，保证不会再有未处理的标记工作，并完成阶段切换。

#### 后台标记worker

并发标记不是一个单独线程从头扫到尾，而是调度器配合运行 GC worker。大致有几类 worker：

```text
dedicated mark worker
  专门执行标记工作，目标是提供稳定 GC CPU

fractional mark worker
  按比例占用某些 P 的时间，补齐目标利用率

idle mark worker
  在 P 没有普通 goroutine 可运行时执行 GC 工作

mutator assist
  用户 goroutine 分配太快时，自己做一部分标记工作
```

可以把标记工作的来源画成：

```text
GC work queue
   ^
   |
   +-- dedicated worker   稳定消耗 GC 工作
   +-- fractional worker  按比例补足 GC CPU
   +-- idle worker        P 空闲时顺手做
   +-- mutator assist     分配者欠债时被拉来做
```

`mgcpacer.go` 里默认后台标记目标利用率是 `GOMAXPROCS` 的一部分，源码常量里可以看到 25% 这个目标。实际执行会结合专用 worker、fractional worker、idle worker 和 assist 共同完成。

#### mutator assist

mutator 指用户程序本身。GC 并发标记时，如果用户 goroutine 不断快速分配，堆增长可能跑在标记前面。为避免 GC 永远追不上，Go 使用 allocation assist。

分配路径中可以看到：

```text
mallocgc
  |
  +-- gcBlackenEnabled != 0
        |
        v
      deductAssistCredit(size)
        |
        v
      如果欠债太多，当前 G 执行 gcAssistAlloc
```

直观理解：

```text
你分配得越多，就越需要帮 GC 做一些扫描工作。
```

这不是惩罚，而是背压机制。它让分配速度和标记速度保持相对平衡，避免到了 heap goal 还没标记完，最终被迫长时间 STW。

用一个时间线看更直接：

```text
G1 正在执行业务代码
  |
  v
make([]*Node, 10000) 触发较多堆分配
  |
  v
当前处于 _GCmark，GC 正在并发标记
  |
  v
mallocgc 发现 G1 的 assist credit 不够
  |
  v
G1 暂停继续分配，进入 gcAssistAlloc
  |
  v
从 GC 工作队列取一些灰色对象，扫描它们的指针字段
  |
  v
还清一部分 assist debt
  |
  v
回到 mallocgc，继续完成这次分配
```

所以 mutator assist 干的活不是“清理自己刚分配的对象”，而是帮当前 GC 周期推进标记工作。它可能扫描的是别的对象，只要这些对象在 GC 工作队列里。用户能感受到的是：某次分配突然变慢，因为这个 goroutine 被拉去帮 GC 还债了。

#### 根标记和栈扫描

标记阶段会准备 root jobs，包括：

- 扫描 goroutine 栈。
- 扫描全局变量。
- 扫描 runtime 堆外结构里的堆指针。

栈扫描需要让目标 goroutine 处在可扫描状态。运行中的 goroutine 可能需要被抢占到安全点；已经阻塞的 goroutine 通常更容易扫描，但也要注意它是否正在 channel 操作等特殊状态。

栈扫描过程依赖编译器元数据：

```text
函数 PC
  |
  v
找到对应函数元数据
  |
  v
找到当前安全点的 stack map
  |
  v
知道哪些栈槽是指针
```

画成栈扫描就是：

```text
goroutine stack

frame main.main      stack map: [ptr, scalar, ptr]
  slot0 -> heap A    扫描，标记 A
  slot1 = 10         跳过
  slot2 -> heap B    扫描，标记 B

frame handle         stack map: [scalar, ptr]
  slot0 = fd         跳过
  slot1 -> heap C    扫描，标记 C
```

GC 不会把栈上的每个整数都当指针试一遍；它按 PC 找到对应栈图，只扫描标成 pointer 的槽位。

这就是 Go GC 能做到精确扫描的基础。

#### gcDrain与工作队列

标记时，灰色对象会被放入 GC work queue。`gcDrain` 的核心工作是：

```text
while 有灰色对象或 root job:
  取一个 job
  如果是 root job：扫描根
  如果是 heap object：扫描对象指针字段
  发现 white 对象：标记并入队
```

为了降低全局竞争，GC work 也有本地缓存和批量转移机制。每个 `P` 可以持有部分 GC work，必要时再和全局队列交换。

#### sweep实现

标记结束后，清扫并不要求一次性完成。Go 的 sweep 可以：

- 后台 sweeper 逐个 span 清扫。
- 分配器需要 span 时顺手清扫相关 span。
- 如果下一轮 GC 要开始而还有未清扫 span，则先完成 sweep termination。

清扫一个 span 时，runtime 会根据 `gcmarkBits` 判断哪些对象仍然存活，把不可达对象的槽位释放出来。结果可能有三种：

```text
span 里还有活对象，并且有空槽
  -> 回到 mcentral partial

span 里还有活对象，但没有空槽
  -> 回到 mcentral full

span 里没有活对象
  -> 页归还 mheap，可被其他 size class 或大对象复用
```

这解释了为什么 Go 内存曲线里经常看到“对象已经不可达，但 RSS 不一定立刻下降”。GC 回收的是 Go 堆可复用空间；是否把物理页归还给 OS，还涉及 scavenger 和操作系统内存管理。

#### runtime.GC做了什么

`runtime.GC()` 会强制触发一次 GC cycle，并等待这轮 GC 和相关 sweep 进展到足够完成的状态。它不是普通业务逻辑里应该频繁调用的性能优化手段。

适合了解的场景包括：

- benchmark 前后人为稳定堆状态。
- 测试 finalizer 或内存释放行为。
- 特殊工具或诊断代码。

常规服务里频繁调用 `runtime.GC()` 往往只是把自动 pacer 的策略打乱，增加 CPU 和暂停成本。

### 15 写屏障与GC Pacer

写屏障和 pacer 是 Go 并发 GC 的两个关键支点：

```text
写屏障：保证并发标记的正确性
pacer：保证 GC 在合适时间开始，并以合适速度完成
```

没有写屏障，并发标记会漏标对象。没有 pacer，GC 可能开始太晚导致内存暴涨，也可能开始太早导致 CPU 浪费。

#### 写屏障解决什么问题

GC 标记和用户 goroutine 并发执行时，用户代码可能随时改指针：

```go
obj.next = other
```

如果这次写入发生在 GC 已经扫描过 `obj` 之后，GC 需要知道 `other` 可能因此变成可达。写屏障就是编译器在指针写入附近插入的一小段 runtime 协作逻辑。

概念上可以理解为：

```text
写入前：
  obj.next(slot) ---> old
  ptr -----------> new

写入时：
  shade(old)     保护被覆盖的旧指针
  shade(new)     保护即将写入的新指针
  obj.next = new

writePointer(slot, ptr):
  shade(*slot)   // 被覆盖掉的旧指针
  shade(ptr)     // 即将写入的新指针
  *slot = ptr
```

Go 源码注释把它描述为混合写屏障：结合 Yuasa deletion barrier 和 Dijkstra insertion barrier。删除屏障关注被覆盖的旧指针，插入屏障关注新写入的指针。二者配合，防止 mutator 在并发标记期间把白色对象藏起来。

#### 编译器何时插入写屏障

写屏障不是每次赋值都有。它主要针对可能把堆指针写入堆对象或全局变量的操作。

例如：

```go
type Node struct {
	next *Node
}

func link(a, b *Node) {
	a.next = b
}
```

如果 `a` 指向堆对象，`a.next = b` 在 GC 标记期间就需要写屏障。编译器通常会生成类似这样的逻辑：

```text
if writeBarrier.enabled {
	runtime.gcWriteBarrier(...)
} else {
	普通指针写入
}
```

而写当前函数栈帧上的局部变量，通常不需要写屏障，因为当前 goroutine 的栈会作为根扫描。对于 `typedmemmove`、`typedmemclr`、slice copy、reflect 等批量移动或清零指针的操作，runtime 还有 bulk barrier 入口。

```text
单个指针写：
  a.next = b
  -> gcWriteBarrier

批量移动含指针内存：
  copy(dstObjects, srcObjects)
  -> bulkBarrier 覆盖整段 dst/src 指针范围
```

#### 为什么新对象分配为黑色

并发标记期间，新分配对象会被直接视为 black。这样做的意义是：新对象不会在本轮 GC 里被误回收，也不会要求 GC 立刻重新扫描所有刚分配出来的对象。

分配路径里可以看到类似逻辑：

```text
if writeBarrier.enabled {
  gcmarknewobject(span, uintptr(x))
}
```

这和写屏障共同维护并发标记不变量。用户 goroutine 可以继续分配和写指针，GC 仍然能得到一个保守且正确的存活集合。

#### 写屏障的成本

写屏障开启后，指针写入会变慢一些。成本来自：

- 每次指针写入前检查 `writeBarrier.enabled`。
- 屏障需要把相关指针放入标记缓冲。
- 缓冲满后要把工作提交给 GC。
- 批量复制含指针对象时需要 bulk barrier。

所以降低指针写入密度、减少堆上指针对象数量，也能间接降低 GC 成本。

这也是为什么“少分配”之外，还要关注“少创建复杂指针图”。两个程序堆大小相近，指针密集程度不同，GC 压力可能完全不同。

#### GC Pacer的目标

Pacer 要解决三个问题：

```text
什么时候开始下一轮 GC？
这轮 GC 的目标堆大小是多少？
标记期间要让后台 worker 和 mutator assist 做多少工作？
```

`mgcpacer.go` 里的 `gcController` 维护了这些关键量：

```text
gcPercent        来自 GOGC
memoryLimit      来自 GOMEMLIMIT
heapLive         当前认为的 live heap
heapMarked       上一轮标记后的存活堆
heapScan         可扫描堆大小
lastStackScan    上轮栈扫描量
globalsScan      全局变量扫描量
heapGoal         当前周期目标堆大小
trigger          开始 GC 的堆大小
assistWorkPerByte 每分配 1 字节需要承担多少扫描工作
```

把它想成 pacer 的仪表盘：

```text
内存进度条：

heapMarked -------- trigger -------- heapLive -------- heapGoal
上轮活堆           开始 GC          当前堆          本轮目标上限

扫描进度条：

已完成扫描 workDone -------------- 剩余扫描 workLeft

pacer 根据两条进度条计算：
  - GC 是否该启动
  - 后台 worker 要跑多积极
  - mutator 每分配 1B 要还多少 assist work
```

最简单的情况是只看 `GOGC`：

```text
heapGoal = liveHeap + liveHeap * GOGC / 100
```

默认 `GOGC=100`，意味着目标堆大约是上一轮存活堆的 2 倍。`GOGC` 越大，GC 越不频繁，CPU 压力通常下降，但内存占用上升；`GOGC` 越小，内存目标更紧，GC 更频繁。

举个简化例子：

```text
上一轮 GC 后仍然活着的对象：100MB
GOGC=100

heapGoal ≈ 200MB
```

这句话的意思不是“到 200MB 才开始 GC”，而是“希望这一轮 GC 标记完成时，堆大致不要冲过 200MB”。因为 GC 标记和用户程序并发执行，标记期间用户还会继续分配，所以必须更早开始。

#### trigger为什么小于goal

如果等到 `heapLive == heapGoal` 才开始 GC，已经太晚了。并发标记需要时间，在标记期间用户 goroutine 还会继续分配。所以 pacer 会计算一个更早的 trigger：

```text
heapMarked ---------------- trigger ---------------- heapGoal
上一轮存活堆             开始本轮 GC              希望标记结束时不要超过
```

trigger 和 heapGoal 之间的距离就是 GC 的 runway。runway 太短，GC 可能来不及完成，需要大量 assist 或更长暂停；runway 太长，GC 过早启动，吞吐下降。

Pacer 会根据历史分配速度、扫描速度、上轮扫描工作量、当前堆增长情况动态调整。

还是用上面的数字：

```text
heapMarked = 100MB
heapGoal   = 200MB

如果预计标记期间还会分配 40MB，
那 trigger 可能会被放在 160MB 附近。

160MB 开始标记
  + 标记期间继续分配约 40MB
  = 标记完成时接近 200MB
```

真实 runtime 的计算更复杂，但直觉就是这个：pacer 要给并发 GC 留出刹车距离。

#### assist ratio

标记期间，pacer 会计算一个 assist ratio：

```text
assistWorkPerByte = 剩余扫描工作 / 距离 heapGoal 还能分配的字节数
```

假设剩余扫描工作很多，而距离 heapGoal 已经很近，那么 assist ratio 会升高。结果是分配 goroutine 更容易欠下 assist debt，被迫执行更多标记工作。

这会牺牲部分分配延迟，但避免堆无限增长或最终进入更糟糕的 STW。

继续用同一个例子：

```text
距离 heapGoal 还能分配 20MB
但 GC 估计还剩很多对象没扫描

结果：
  每分配一点内存，就要附带做更多标记工作
  分配 goroutine 更容易进入 gcAssistAlloc
```

所以 pacer 不是“回收内存的算法”，而是 GC 的速度控制系统。它一边看堆增长速度，一边看标记完成进度，然后调节后台 worker 和 mutator assist 的压力。

#### GOMEMLIMIT

`GOMEMLIMIT` 给 Go runtime 一个软内存上限。它不是操作系统级硬限制，而是 pacer 在计算 heap goal 时会考虑的目标。内存限制会让 GC 更积极，也会推动 scavenger 归还不用的页。

可以把 `GOGC` 和 `GOMEMLIMIT` 的关系理解成：

```text
GOGC       根据存活堆比例控制下一轮堆目标
GOMEMLIMIT 在总内存目标上给 pacer 施加额外约束
```

当二者冲突时，runtime 会尝试在内存限制下运行，代价通常是更频繁的 GC、更高的 assist 压力和更低的吞吐。

#### 写屏障与pacer如何配合

整个并发 GC 能成立，靠的是多处协作：

```text
pacer 提前启动 GC
  |
  v
STW 开启写屏障
  |
  v
用户 goroutine 继续运行
  |
  +-- 指针写入走写屏障，维持标记正确性
  |
  +-- 新分配对象直接标黑
  |
  +-- 分配过快时执行 mutator assist
  |
  v
后台 mark worker 消耗标记队列
  |
  v
标记完成后 STW 收尾
  |
  v
并发 sweep 回收未标记对象
```

如果只看某一个点，会觉得复杂；但从目标看很直接：在用户程序继续运行的同时，尽量正确、及时、低暂停地完成堆回收。

#### 调优时看什么

调 GC 之前，先确认问题是什么：

```text
内存占用太高
  -> 看 live heap、GOGC、GOMEMLIMIT、对象是否长期被引用

GC CPU 太高
  -> 看分配速率、对象数量、指针密度、GOGC 是否过低

延迟尖刺
  -> 看 assist、STW pause、栈扫描、巨大对象扫描、调度阻塞

RSS 不下降
  -> 区分 Go heap 可复用、scavenger 归还、OS 是否真正回收物理页
```

常见优化方向不是一上来改环境变量，而是先减少无意义分配：

- 让短生命周期对象留在栈上。
- 复用临时 buffer。
- 减少指针密集的小对象数量。
- 用值类型或扁平结构替代大量链式对象。
- 避免把大对象长时间挂在全局缓存里。
- 对高频路径做 pprof/trace 验证，而不是凭感觉调参。

#### 问答速览

问题一：goroutine 栈为什么能动态增长？

```text
编译器在函数入口插入栈检查。空间不足时进入 morestack，runtime 切到 g0 栈执行 newstack，分配更大的连续栈，把旧栈已使用部分复制过去，并根据栈图和 runtime 元数据修正指向旧栈的指针。
```

问题二：小对象分配为什么通常很快？

```text
小对象按 size class 分配。当前 P 的 mcache 持有各 spanClass 的可分配 span，常见路径只需要在当前 span 的 bitmap 里找空槽位，不需要全局锁。mcache 用完再向 mcentral 取 span，mcentral 不足再向 mheap 取页。
```

问题三：span、heap、arena分别是什么？

```text
span 是一段连续页，是分配器和 GC 管理对象的基本单位；heap 是 runtime 通过 mheap 管理的全局页级堆；arena 是堆虚拟地址空间的大块划分，每个 heapArena 保存这块地址范围的 span map、bitmap 等元数据。
```

问题四：Go GC为什么需要写屏障？

```text
因为标记和用户 goroutine 并发执行。用户代码可能在 GC 已经扫描过某个对象后继续写入新指针。写屏障在指针写入时把旧指针或新指针通知给 GC，避免可达对象被错误地保持为白色并在 sweep 时回收。
```

问题五：GOGC=100是什么意思？

```text
可以近似理解为：下一轮 GC 的目标堆大小是上一轮存活堆的 2 倍。若上一轮标记后 live heap 是 100MB，GOGC=100 时 heap goal 大约是 200MB。实际 trigger 会更早，由 pacer 根据扫描速度、分配速度和内存限制计算。
```

问题六：为什么 GC 后内存不一定马上还给操作系统？

```text
GC sweep 首先把不可达对象占用的槽位或 span 变成 Go 堆可复用空间。是否把空闲页归还给 OS 由 scavenger 和操作系统共同决定。Go 程序内看到的 HeapAlloc、HeapIdle、HeapReleased、RSS 代表不同层面的内存状态，不能只看一个指标。
```

## Part 4 Synchronization

### 16 Channel实现
### 17 阻塞、唤醒与信号量
### 18 Mutex/RWMutex实现
### 19 WaitGroup/Once/Cond实现
### 20 Timer实现

## Part 5 System Integration

### 21 Netpoll实现
### 22 Syscall处理
### 23 Signal处理
### 24 cgo与线程管理

## Part 6 语言特性底层实现

### 25 Interface实现
### 26 类型元数据与Reflect
### 27 Defer/Panic/Recover
### 28 Map实现
### 29 Slice实现
### 30 String实现

## Part 7 Diagnostics & Optimization

### 31 Runtime性能优化
### 32 pprof
### 33 trace
### 34 GODEBUG
### 35 常见Runtime问题排查
