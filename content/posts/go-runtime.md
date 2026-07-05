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

### 10 Goroutine栈管理
### 11 内存分配器
### 12 Span、Heap与Arena
### 13 GC原理
### 14 GC实现
### 15 写屏障与GC Pacer

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
