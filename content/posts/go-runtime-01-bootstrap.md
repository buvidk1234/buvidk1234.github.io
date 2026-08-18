+++
date = '2026-08-18T00:10:00+08:00'
draft = false
title = 'Go Runtime（一）：运行时架构与程序启动'
tags = ['Go', 'Runtime']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64**。源码链接固定到 `go1.26.2` tag；其他操作系统、架构和构建模式的入口汇编会不同。

Go runtime 是链接进 Go 可执行文件的一组基础设施。它不解释字节码，也不需要作为独立进程启动；它和编译器、标准库及操作系统共同实现 Go 的语言语义和执行模型。

本文先建立 runtime 的模块边界，再从 Linux/amd64 可执行文件入口追踪到 `main.main`。后续文章分别沿调度、栈、分配、GC、Channel、同步、Timer 和系统集成深入。

## Runtime解决什么问题

一段普通 Go 程序表面上只包含函数调用、对象分配和 goroutine：

```go
package main

func main() {
	ch := make(chan int)
	go func() { ch <- 1 }()
	println(<-ch)
}
```

但这些语法背后至少需要 runtime 完成以下工作：

| 语法或行为 | Runtime职责 |
| --- | --- |
| `go f()` | 创建 G、准备初始栈、加入可运行队列 |
| `new(T)`、`&T{}` | 按对象大小和指针信息分配内存 |
| channel 收发 | 匹配发送方与接收方，必要时阻塞或唤醒 G |
| 栈空间不足 | 分配更大连续栈，复制现场并修正栈内指针 |
| 堆对象失去引用 | 并发标记、清扫和复用内存 |
| 网络读写等待 | 将 G 挂起到 netpoll，fd 就绪后恢复运行 |
| panic/defer/recover | 展开调用栈并按语言规则执行延迟调用 |

因此，runtime 不是一个单独功能，而是支撑程序启动、调度、内存、GC、同步和系统交互的一组机制。

## Runtime不是JVM

Go 通常提前编译成本地机器码。程序运行时，CPU 直接执行这些机器指令；runtime 作为可执行文件的一部分提供底层服务。

| 项目 | Go runtime | JVM |
| --- | --- | --- |
| 常见输入 | 本地机器码 | 字节码 |
| 是否解释字节码 | 否 | 可以解释或 JIT 编译 |
| 是否单独安装运行环境 | 通常不需要 | 通常需要 JVM |
| 主要职责 | 调度、内存、GC、栈、系统集成 | 字节码执行、JIT、GC、类加载 |

二者都包含 GC 等运行时能力，但执行模型不同。把 Go runtime 简单叫作“Go 虚拟机”会掩盖编译器、链接器与 runtime 之间的分工。

## 整体架构

```text
                        用户代码
                           |
                 编译器生成的调用和元数据
                           |
+----------------------------------------------------------+
|                        Go runtime                        |
|                                                          |
|  启动       rt0 -> schedinit -> runtime.main             |
|  调度       G / M / P / runq / work stealing            |
|  栈         stack check / morestack / copystack          |
|  分配       mcache / mcentral / mheap / arena            |
|  GC         root scan / mark / barrier / sweep / pacer   |
|  阻塞       gopark / goready / sema                      |
|  系统集成   timer / netpoll / syscall / signal / cgo     |
+----------------------------------------------------------+
                           |
              OS：线程、虚拟内存、信号、I/O事件
```

模块不能割裂理解。`P` 不仅持有运行队列，也持有 `mcache` 和 timer；goroutine 栈既由调度器使用，也是 GC 根；网络 I/O 会经过 netpoll 挂起 G，再回到调度器；阻塞 syscall 则要求 M 暂时交出 P。

## 三组核心对象

### 执行对象

```text
G  goroutine 的运行状态、栈和调度现场
M  OS 线程及其 runtime 上下文
P  执行 Go 代码所需的资源和调度资格
```

M 必须绑定 P 才能执行普通 Go 代码。P 的数量由 `GOMAXPROCS` 控制，也基本决定同一时刻最多有多少线程执行 Go 代码。

### 内存对象

```text
mcache    每个 P 的小对象分配缓存
mcentral  某一 spanClass 的共享 span 集合
mheap     全局页级堆和 arena 元数据
mspan     一段连续页及其对象位图、清扫状态
```

小对象的常见分配路径是 `mcache -> mcentral -> mheap`，大对象通常直接从 `mheap` 获取页。

### GC对象

```text
gcphase       当前 GC 阶段
gcController  触发点、目标和 assist 比例
gcWork        标记工作缓存
write barrier 并发标记期间维护可达性
```

GC 与分配器形成闭环：分配速度推动 GC，GC 回收的 span 又回到分配器复用。

## Runtime与其他组件的边界

### 编译器

编译器负责把语言结构降级成 runtime 能执行的形式，并生成类型、栈图等元数据。例如：

```text
go f()          -> 创建 goroutine 的 runtime 路径
make(chan T)    -> channel 创建路径
map/slice 操作  -> 内联代码或 runtime helper
指针写入       -> 必要时插入写屏障
函数入口       -> 栈空间检查
```

### 链接器

链接器把用户代码、标准库和 runtime 组织为一个可执行文件，解析符号和重定位，并确定程序入口。程序不是先进入 `main.main`，而是先进入平台相关的 runtime 启动代码。

### 标准库

标准库提供稳定、可移植的 API，并在内部使用 runtime。例如 `net.Conn.Read` 经 `internal/poll` 接入 netpoll，`sync` 包通过 runtime 信号量完成阻塞和唤醒。

### 操作系统

操作系统只认识线程、虚拟内存、文件描述符、定时器和信号。runtime 把这些能力转换为 goroutine、Go 堆、网络轮询和异步抢占。

## 可执行文件为什么不从main.main开始

编译器把每个包编译为目标代码和元数据，链接器再完成三件与启动直接相关的工作：

1. 把用户包、依赖包和 runtime 合并为可执行文件。
2. 汇总包初始化任务，并按依赖顺序组织到 moduledata。
3. 把平台启动符号写入 ELF 入口，而不是把 `main.main` 直接写成进程入口。

Linux 内核装载 ELF 后创建初始线程，准备初始栈上的 `argc/argv/envp`，再跳转到入口地址。此时 Go 堆、P、用户 G 和 GC 都还没有完成初始化，因此不能直接执行普通 Go 函数。

Linux/amd64 的第一跳位于 [`runtime/rt0_linux_amd64.s`](https://github.com/golang/go/blob/go1.26.2/src/runtime/rt0_linux_amd64.s)：

```asm
TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
	JMP _rt0_amd64(SB)
```

通用 amd64 入口 `_rt0_amd64` 把初始栈中的参数交给 [`runtime.rt0_go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/asm_amd64.s#L175)。从这里开始，runtime 才逐步建立自己的执行环境。

## rt0_go建立第一个M和g0

`rt0_go` 是汇编函数，因为它运行在普通 Go ABI 和调度环境建立之前。它的职责可以分成四组：

```text
保存 argc/argv
  -> 对齐初始线程栈
  -> 为 m0.g0 设置 stack.lo/stack.hi 和 guard

建立线程本地状态
  -> tls 指向 m0/g0
  -> 验证 getg 能取回 g0

平台和 runtime 初始化
  -> check
  -> args
  -> osinit
  -> schedinit

进入调度系统
  -> newproc(runtime.main)
  -> mstart
```

`m0` 和 `g0` 是静态准备的启动对象，不需要先通过普通分配器创建。TLS 建立后，runtime 汇编和 Go 代码才能通过 `getg()` 找到当前执行上下文。

这里的 `m0.g0` 使用操作系统为初始线程提供的栈空间，`rt0_go` 只为它建立 runtime 可识别的边界和 guard。后续由 runtime 创建的 M 则会获得各自分配的 g0 栈。g0 用于调度、栈增长等 runtime 内部路径；它不是 main goroutine，也不会作为普通 G 放入运行队列。

在 Go 1.26.2 的 amd64 主线中，关键汇编顺序是：

```asm
CALL runtime·args(SB)
CALL runtime·osinit(SB)
CALL runtime·schedinit(SB)

MOVQ $runtime·mainPC(SB), AX
PUSHQ AX
CALL runtime·newproc(SB)
POPQ AX

CALL runtime·mstart(SB)
```

这里证明了两个容易误传的结论：`runtime.main` 是通过 `newproc` 创建的普通 G 入口；启动线程随后进入 `mstart`，而不是直接 `CALL main.main`。

## osinit与schedinit的边界

[`osinit`](https://github.com/golang/go/blob/go1.26.2/src/runtime/os_linux.go#L353) 读取 CPU 数、物理页大小、进程亲和性或平台能力等 OS 信息。它必须早于调度器根据 CPU 情况确定初始 P 数量。

[`schedinit`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L831) 名字叫“调度初始化”，实际建立的是大部分 runtime 基础设施。Go 1.26.2 中可以按依赖关系归纳为：

```text
初始化全局锁和调度结构
  -> 读取早期 GODEBUG、初始化 tick
  -> 验证 moduledata
  -> stackinit
  -> randinit
  -> mallocinit
  -> cpuinit / alginit
  -> mcommoninit(m0)
  -> modulesinit / typelinksinit / itabsinit / stkobjinit
  -> 保存信号掩码，读取参数和环境
  -> gcinit
  -> 初始化崩溃备用栈
  -> 计算初始 GOMAXPROCS
  -> procresize 创建并初始化 P
```

顺序体现真实依赖。例如 `randinit` 必须早于依赖随机种子的哈希初始化；`mallocinit` 之后才能安全执行需要堆的路径；`stkobjinit` 必须在 GC 启动前准备栈对象元数据。

不要把 `schedinit` 简化成“创建 P”。它是从最小汇编环境过渡到可运行 Go runtime 的总初始化点。

## main goroutine如何创建

汇编中的 `runtime.mainPC` 是指向 `runtime.main` ABIInternal 入口的函数值。`newproc` 在系统栈上调用 `newproc1`，为它取得 G、准备初始栈和调度现场，并把状态从 `_Gdead` 转为 `_Grunnable`。

```text
rt0_go on m0.g0
  -> newproc(runtime.main)
       -> gfget / malg
       -> 初始化 sched.pc、sched.sp、startpc
       -> casgstatus(_Gdead, _Grunnable)
       -> runqput 当前 P
  -> mstart
       -> mstart1
       -> schedule
       -> execute(main G)
```

`mstart` 应当永不返回；它最终在 g0 上进入调度循环。调度器选中 main G 后通过上下文恢复执行 [`runtime.main`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L149)。

## runtime.main的初始化阶段

`runtime.main` 已经运行在可增长的普通 G 栈上，但用户包仍未开始。它主要完成：

1. 设置最大 goroutine 栈限制。
2. 允许 `newproc` 根据需要创建新 M。
3. 在独立 M 上启动 `sysmon`。
4. 暂时把 main G 锁在初始 OS 线程上。
5. 执行 runtime 自身的 init task。
6. 启用 GC，并完成 cgo 模式所需初始化。
7. 按 moduledata 顺序执行所有用户包 init task。
8. 间接调用链接器解析的 `main.main`。

锁定初始线程解释了一个细节：包初始化期间若调用 `runtime.LockOSThread`，可以让 `main.main` 继续保留在初始线程；否则完成 init 后 runtime 会解除这次内部锁定。

## init任务由谁排序

编译器为包生成初始化任务，链接器根据导入依赖把它们组织进 moduledata。runtime 的 [`doInit`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L8068) 负责执行已经排好依赖关系的任务，而不是在启动时重新分析 import 图。

```text
runtime_inittasks
  -> 各 module 的 inittasks（依赖顺序）
       -> 包级变量初始化
       -> init 函数
  -> main.main
```

插件和不同 buildmode 会改变 moduledata 的来源，所以源码中遍历的是模块链表，而不是假设只有一个静态任务数组。

## main.main返回后的进程语义

`runtime.main` 通过间接函数值调用用户入口：

```go
fn := main_main
fn()
```

链接器负责把 `main_main` 关联到用户包的 `main.main`。调用返回后，runtime 会处理退出 hook、race/ASAN 等收尾，最终调用 `exit(0)`。

因此其他 goroutine 是否仍然 runnable 不影响进程生命周期：main 返回后进程退出。需要完成的后台任务必须在 main 返回前显式同步。

```go
func main() {
	go flush() // main 返回时，flush 没有完成保证
}
```

这不是调度器“忘记等待”，而是 Go 定义的 main goroutine 进程语义。

## 完整启动时序

```text
Linux ELF loader
  -> _rt0_amd64_linux
  -> _rt0_amd64
  -> runtime.rt0_go on m0.g0
       -> TLS、m0、g0
       -> args -> osinit -> schedinit
            -> stack/malloc/type/GC/P 初始化
       -> newproc(runtime.main)
       -> mstart
            -> schedule
            -> execute main G
                 -> runtime.main
                      -> sysmon
                      -> runtime init tasks
                      -> gcenable
                      -> package init tasks
                      -> main.main
                      -> exit(0)
```

这条主线同时解释了编译器、链接器、runtime 和操作系统的分工：OS 只创建初始线程并跳进入口；链接器准备入口和 init 元数据；runtime 建立 Go 执行世界；用户 `main` 最后才获得控制权。

## 启动源码验证方法

先记录分析环境，避免本机源码、在线链接和实际构建版本不一致：

```bash
go version
go env GOROOT GOOS GOARCH GOEXPERIMENT CGO_ENABLED
```

本文的在线链接固定到 `go1.26.2`；本地阅读应确认 `$GOROOT/src/runtime` 来自同一版本。仅设置 `GOOS/GOARCH` 不会自动让正在运行的操作系统行为变成目标平台，交叉编译结果和本机动态观测也要分开解释。

可以编译最小程序并查看 ELF 入口与符号：

```bash
go build -o hello ./hello.go
readelf -h hello | grep 'Entry point'
go tool nm hello | grep -E '(_rt0_amd64_linux|runtime.rt0_go|runtime.main|main.main)'
```

再用 `go tool objdump -s 'runtime.rt0_go' hello` 查看链接后的启动汇编。启用内部链接、外部链接、PIE 或 c-shared 等模式时，入口和中间 trampoline 可能变化，文章主线只对应默认 Linux/amd64 可执行文件。

## 源码入口

| 主题 | 主要文件 |
| --- | --- |
| 启动和调度 | `runtime/proc.go`、`runtime/runtime2.go` |
| 栈 | `runtime/stack.go`、`runtime/asm_*.s` |
| 分配器 | `runtime/malloc.go`、`runtime/mheap.go` |
| GC | `runtime/mgc.go`、`runtime/mgcmark.go` |
| 阻塞唤醒 | `runtime/proc.go`、`runtime/sema.go` |
| Channel与Select | `runtime/chan.go`、`runtime/select.go` |
| 同步原语 | `runtime/sema.go`、`internal/sync/mutex.go`、`sync/*.go` |
| Timer | `runtime/time.go`、`time/sleep.go` |
| 网络轮询 | `runtime/netpoll.go`、`runtime/netpoll_*.go` |
| 信号 | `runtime/signal_*.go`、`runtime/sigqueue.go` |
| cgo | `runtime/cgocall.go`、`runtime/asm_*.s` |

阅读时应先锁定正在使用的 Go 版本。runtime 的函数名、字段和局部实现会变化，但“状态如何转换、资源归谁持有、阻塞后谁继续执行”这些主线相对稳定。

## 推荐阅读路线

文章编号保留了最初拆分和后续补篇的发布顺序，因此编号不完全等于源码依赖顺序。按依赖首次阅读，推荐路线是：

```text
01 架构与启动
  -> 02 调度器
  -> 03 栈
  -> 04 分配器
  -> 05 GC
  -> 08 Channel与Select
  -> 09 同步原语
  -> 10 Timer
  -> 06 Netpoll与Syscall
  -> 07 Signal与cgo
```

各篇的阅读目标如下：

1. [架构与启动](/posts/go-runtime-01-bootstrap/)：理解第一个 M、P、G 如何出现。
2. [调度器](/posts/go-runtime-02-scheduler/)：理解 goroutine 如何创建、阻塞、恢复和抢占。
3. [栈管理](/posts/go-runtime-03-stack/)：理解函数入口检查和连续栈搬迁。
4. [分配器](/posts/go-runtime-04-allocator/)：理解堆对象、span、arena 和页分配。
5. [GC](/posts/go-runtime-05-gc/)：理解对象如何被追踪和回收。
6. [Netpoll 与 Syscall](/posts/go-runtime-06-netpoll-syscall/)：理解两类阻塞如何接入调度器。
7. [Signal 与 cgo](/posts/go-runtime-07-signal-cgo/)：理解异步事件和外部线程如何进入 runtime。
8. [Channel 与 Select](/posts/go-runtime-08-channel-select/)：理解值传递、等待队列和多路竞争。
9. [Runtime 信号量与同步原语](/posts/go-runtime-09-sync-primitives/)：理解 Mutex、RWMutex、WaitGroup、Cond 和 Once 的阻塞底座。
10. [Timer](/posts/go-runtime-10-timer/)：理解每 P 时间堆、Stop/Reset 竞态及其与 netpoll 的协作。

## 延伸阅读

- [Go程序初始化与执行规范](https://go.dev/ref/spec#Program_initialization_and_execution)
- [Go语言设计与实现](https://draven.co/golang/)：适合建立全局阅读路线；具体启动代码以本文固定版本链接为准。

## 系列导航

- 当前：Go Runtime（一）：运行时架构与程序启动
- [下一篇：GMP与Goroutine调度](/posts/go-runtime-02-scheduler/)
- [原始长文](/posts/go-runtime/)
