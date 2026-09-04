+++
date = '2026-08-18T00:10:00+08:00'
draft = false
title = 'Go Runtime（一）：运行时架构与程序启动'
tags = ['Go', 'Runtime']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64**。源码链接固定到 `go1.26.2` tag；其他操作系统、架构和构建模式的入口汇编会不同。

一个 Go 程序可以只有几行代码，但它的第一条指令并不属于 `main.main`。在用户函数开始前，可执行文件必须先建立线程本地状态、调度资源、栈和堆等运行环境。

本文先从 ELF 入口这个可观察事实出发，再沿 Linux/amd64 启动链追踪到 `main.main`。重点不是背诵初始化函数，而是回答：每一步补上了什么运行前提，编译器、链接器、runtime 和操作系统各自负责哪一段。

## 一个最小Go程序

以下程序表面上只有 goroutine、Channel 和一次函数返回：

```go
package main

func main() {
	ch := make(chan int)
	go func() { ch <- 1 }()
	println(<-ch)
}
```

这段程序的启动和运行依次经过编译、链接与操作系统装载。执行期间，runtime 把操作系统提供的线程、虚拟内存、信号和 I/O 事件转换成 Go 的调度、栈、堆和阻塞语义：

```text
Go源码
  -> 编译器：机器码 + 类型/栈图/init task 等元数据
  -> 链接器：用户包 + 标准库 + runtime -> ELF可执行文件
  -> OS loader：建立初始线程和进程地址空间，跳到ELF入口
  -> runtime启动：rt0 -> schedinit -> runtime.main
       |
       +-- 执行：G / M / P / runq
       +-- 栈：stack check / morestack / copystack
       +-- 内存：mcache / mcentral / mheap / GC
       +-- 等待：gopark / goready / timer / netpoll / syscall
  -> package init -> main.main
```

这里没有一个独立安装、解释字节码的“Go 虚拟机”：CPU 执行的是提前编译并链接好的本地机器码，runtime 也是可执行文件的一部分。标准库提供稳定 API，并在内部接入 runtime，例如 `net.Conn` 通过 `internal/poll` 使用 netpoll，`sync` 通过 runtime 等待设施挂起和唤醒 G。

几组对象把这些层连接起来：G 保存 goroutine 的栈、状态和现场，M 表示 OS 线程及其 runtime 上下文，P 提供执行 Go 代码的资格和本地资源。P 不只持有运行队列，还持有 `mcache`、timer 和 GC work；goroutine 栈同时又是 GC 根。因此调度、内存和系统交互必须作为同一个执行系统理解。

## 先验证：ELF入口不是main.main

先记录实验环境，避免本机源码、在线链接和实际构建版本不一致：

```bash
go version
go env GOROOT GOOS GOARCH GOEXPERIMENT CGO_ENABLED
```

把上面的程序保存为 `hello.go`，编译后查看 ELF 入口和几个关键符号：

```bash
go build -o hello ./hello.go
readelf -h hello | grep 'Entry point'
go tool nm hello | grep -E '(_rt0_amd64_linux|runtime.rt0_go|runtime.main|main.main)'
go tool objdump -s 'runtime.rt0_go' hello
```

`readelf` 给出的是装载器将跳转到的机器地址，`nm` 则能同时找到平台入口、`runtime.main` 和用户 `main.main`。它们不是同一个符号，这正是接下来要解释的现象：用户入口存在于二进制中，却要等 runtime 建立执行环境后才会被调用。

本文的在线链接固定到 `go1.26.2`；本地阅读应确认 `$GOROOT/src/runtime` 来自同一版本。内部链接、外部链接、PIE、c-shared 等模式可能增加其他入口或 trampoline，以下主线只对应默认 Linux/amd64 可执行文件。

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

`rt0_go` 用汇编实现，因为它接手的是操作系统刚创建的初始线程：此时 TLS、当前 G、运行时栈边界和调度器都尚未初始化，它必须先直接操作栈指针、线程本地状态和静态启动对象，建立普通 Go 代码所依赖的运行时前提。它的职责可以分成四组：

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

[`osinit`](https://github.com/golang/go/blob/go1.26.2/src/runtime/os_linux.go#L353) 通过 `getCPUCount` 读取启动时可用 CPU 数（Linux 实现会考虑进程亲和性），并探测透明大页大小和 `vgetrandom` 能力。这里的 `physHugePageSize` 是 huge page 配置，不是普通物理页大小。`osinit` 必须早于调度器根据 CPU 情况确定初始 P 数量。

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

## 源码入口

| 目标 | 入口 |
| --- | --- |
| Linux/amd64入口 | [`runtime/rt0_linux_amd64.s`](https://github.com/golang/go/blob/go1.26.2/src/runtime/rt0_linux_amd64.s) |
| amd64启动汇编 | [`runtime/asm_amd64.s: rt0_go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/asm_amd64.s#L175) |
| 平台早期初始化 | [`runtime/os_linux.go: osinit`](https://github.com/golang/go/blob/go1.26.2/src/runtime/os_linux.go#L353) |
| Runtime与main G初始化 | [`runtime/proc.go: schedinit/runtime.main`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L149) |
| init task执行 | [`runtime/proc.go: doInit`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L8068) |

阅读时应先锁定正在使用的 Go 版本。runtime 的函数名、字段和局部实现会变化，但“状态如何转换、资源归谁持有、阻塞后谁继续执行”这些主线相对稳定。

## 延伸阅读

- [Go程序初始化与执行规范](https://go.dev/ref/spec#Program_initialization_and_execution)
- [Go语言设计与实现](https://draven.co/golang/)：适合建立全局阅读路线；具体启动代码以本文固定版本链接为准。

## 系列导航

- [系列目录](/posts/go-runtime/)
- 当前：Go Runtime（一）：运行时架构与程序启动
- [下一篇：GMP与Goroutine调度](/posts/go-runtime-02-scheduler/)
