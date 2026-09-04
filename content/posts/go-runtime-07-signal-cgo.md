+++
date = '2026-08-18T00:16:00+08:00'
draft = false
title = 'Go Runtime（七）：Signal与cgo线程管理'
tags = ['Go', 'Runtime', 'Signal', 'cgo']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64**。Signal 的编号、trampoline 和上下文结构具有平台差异；本文只把 Unix/Linux 主线作为具体实现，Windows 异常和 IOCP 回调需要单独对照平台文件。

CPU profile 或崩溃栈可能呈现一个反直觉结果：信号打断了任意指令，C 创建的 pthread 也出现在 Go 调用栈里，但 runtime 仍必须识别当前线程、当前 G 和可扫描的栈。信号和 cgo 都会打破普通 Go 调用链的假设：信号可以异步到达，C 代码可能阻塞、修改线程状态，甚至从一个非 Go 创建的线程回调 Go。

本文先解释异步信号入口，再沿 Go 调 C、C 回调 Go 的调用链前进，最后回答外部线程进入 Go 时 M、g0、gsignal 和 TLS 从哪里来。runtime 必须在这些边界上重新建立可识别的线程、栈和执行现场，同时避免在不安全上下文中执行分配、加锁或调度操作。

## 异步信号边界

### Signal承担哪些职责

Go runtime 使用或处理信号的目的并不相同：

- 把 `SIGINT`、`SIGTERM` 等转交给 `os/signal`。
- 将某些同步 fault 转换为 Go panic。
- 用异步信号抢占长时间运行的 goroutine。
- 采样 `SIGPROF` 生成 CPU profile。
- 在 cgo 场景与 C 安装的 handler 协作。
- 对致命信号打印 traceback 并退出。

因此不能笼统地说“Go 捕获了信号”。同一个 handler 入口会根据 signal 类型、发生线程和当前 G/M 状态走不同分支。

### sigtable描述策略

runtime 为平台信号维护一张策略表，描述信号的默认行为和处理属性。概念上包括：

```text
是否通知用户层
是否默认忽略
是否导致 panic
是否打印 stack
是否导致进程退出
是否属于同步 fault
```

策略还会受程序是否导入 `os/signal`、是否使用 cgo、signal disposition 是否被外部代码修改等因素影响。

### 初始化信号处理

runtime 启动时会检查已有 signal handler，保存需要转发的处理器，并为当前及新建线程配置信号掩码和备用信号栈。

```text
runtime 初始化
  -> 读取已有 signal disposition
  -> 保存需要兼容的 handler
  -> 安装 Go handler
  -> 配置 alt signal stack
  -> 设置线程 signal mask
```

备用信号栈很重要：如果普通 goroutine 栈接近边界，或 fault 与当前栈本身有关，handler 仍需要一块可靠空间运行。

signal mask 是线程属性。runtime 创建新 M 时必须继承或重设正确掩码；cgo 创建的线程回调 Go 时也要处理对应线程状态。

### Handler中的限制

signal handler 可能打断：

- 用户代码中的普通指令。
- runtime 正在持锁或更新内部结构的代码。
- 栈增长、写屏障或 GC 关键路径。
- C 代码。
- g0、gsignal 或未知线程上的执行。

因此 handler 不能像普通 Go 函数一样任意分配、阻塞或获取常规锁。它主要读取保存的寄存器现场，执行受控的原子状态更新，或修改返回 PC 进入预先设计的 runtime 路径。

每个 M 的 `gsignal` 为信号处理提供专用 G 和栈。它不是普通可调度 goroutine，也不承载业务代码。

### sighandler分发

抽象流程如下：

```text
OS signal
  -> 平台汇编 trampoline
  -> sigtramp / sigtrampgo
  -> sighandler
       |
       +-- 异步抢占：尝试改写现场到 asyncPreempt
       +-- SIGPROF：记录当前 PC/SP/G
       +-- 可转 panic 的同步 fault：进入 sigpanic
       +-- os/signal 已订阅：发送到 runtime 信号队列
       +-- 需要转发：调用先前/C handler
       +-- 致命信号：traceback 并退出
```

顺序和条件很关键。同一个 `SIGSEGV` 在用户 Go 代码中的 nil 解引用可能转成 panic；发生在 runtime、g0、C 代码或不可恢复上下文时则通常只能崩溃。

Go 1.26.2 的 [`sighandler`](https://github.com/golang/go/blob/go1.26.2/src/runtime/signal_unix.go#L646) 实际先处理特殊内部信号，再读取 `sigtable`：

```text
SIGPROF -> validSIGPROF -> sigprof -> return
  -> 测试专用 SIGTRAP/SIGUSR1
  -> Linux per-thread syscall signal
  -> sigPreempt -> doSigPreempt（之后仍可能继续分发）
  -> 读取 sigtable flags
  -> 同步 fault 且可安全转换 -> preparePanic -> sigpanic
  -> 用户信号或 _SigNotify -> sigsend
  -> _SigKill -> dieFromSignal
  -> _SigThrow/_SigPanic 未处理 -> traceback/crash
```

异步抢占信号可能与应用观察到的同类信号合并，因此 `doSigPreempt` 后不能无条件返回。同步 fault 转 panic 也有严格门槛：信号不能来自用户发送，目标必须是当前用户 G，且不能处于 `throwsplit` 等无法安全建立 panic 栈帧的状态。

### os/signal如何收到通知

handler 不会直接执行用户注册的 channel 发送。runtime 使用一套受控的信号队列和唤醒机制，把异步上下文中的事件交给普通 goroutine：

```text
signal handler
  -> 在 bitmask 中记录 signal
  -> 唤醒接收循环
  -> 普通 goroutine 读取 pending signal
  -> os/signal 分发到用户 channel
```

相同普通 Unix signal 可能合并，`os/signal.Notify` 不应被当成可靠计数队列。如果业务要求每个事件都不可丢失，应使用具有队列语义的 IPC 或协议。

### 异步抢占信号

调度器发现某个 G 长时间运行后，通过目标 M 请求抢占信号：

```text
sysmon / GC 请求抢占
  -> preemptM(target M)
  -> OS 向线程发送内部信号
  -> handler 检查 PC、SP、G 状态
  -> 安全：改写返回现场，进入 asyncPreempt
  -> 不安全：放弃本次，稍后重试
```

signal handler 不直接运行完整 schedule。它只把被中断现场导向 runtime 可控入口，后续再保存 G 状态并让出 P。

[`doSigPreempt`](https://github.com/golang/go/blob/go1.26.2/src/runtime/signal_unix.go#L342) 会检查目标确实是当前 M 的 curg、G 正在运行、PC 是异步安全点、SP 有足够空间等条件，再通过 `ctxt.pushCall(abi.FuncPCABI0(asyncPreempt), newpc)` 改写 signal context。信号返回后 CPU 才从 `asyncPreempt` 开始执行。

如果当前位置不可安全抢占，handler 只保留抢占请求并唤醒相关路径，等待同步安全点或下一次信号。所谓“信号式抢占”仍然依赖编译器的 unsafe-point 元数据。

### SIGPROF与CPU Profiling

CPU profiler 周期性向正在运行的线程采样。handler 从 ucontext 等平台现场中获取 PC/SP，并尝试展开调用栈，把样本写入无阻塞、受约束的缓冲路径。

采样发生在任意位置，因此栈可能落在 Go、runtime、系统调用或 C 代码中。结果是统计近似值，而不是每次调用的完整追踪；采样频率过高也会增加信号和栈展开成本。

### cgo如何改变Signal边界

进程里可能同时存在 Go 与 C 安装的 handler。runtime 初始化时需要保存原 handler，并按 signal 类型和发生位置判断由 Go 处理、转给 `os/signal`、转为 panic，还是转发给 C。

如果 C 在 Go runtime 之后再次替换关键 handler，或没有遵守 `SA_ONSTACK` 等兼容要求，可能破坏抢占、panic 转换和 profiling。混合程序需要遵守 Go 文档对 signal handler 的集成约束。这也是 Signal 与 cgo 不能被当成完全独立机制的原因。

## Go-C调用边界

### Go调用C

典型路径为：

```text
Go G
  -> generated cgo wrapper
  -> runtime.cgocall
  -> entersyscall
  -> 切换到系统栈/准备 C ABI
  -> C function
  -> 回到 runtime
  -> exitsyscall
  -> Go G
```

C 函数可能长时间运行，因此 runtime 按类似阻塞 syscall 的方式处理 P：当前 M 可以停在 C 中，P 则能够被其他 M 接管。

cgo 又不完全等同于 syscall，因为它还涉及两套 ABI、栈和线程局部状态，C 可能回调 Go，并且双方 GC/指针规则不同。

[`cgocall`](https://github.com/golang/go/blob/go1.26.2/src/runtime/cgocall.go#L134) 的关键顺序是：

```text
统计 ncgocall，清空 cgo traceback buffer
  -> entersyscall
  -> osPreemptExtEnter
  -> m.incgo = true，m.ncgo++
  -> asmcgocall(fn, arg)：在 m.g0/系统栈上遵循 C ABI
  -> m.incgo = false，m.ncgo--
  -> osPreemptExtExit
  -> exitsyscall
  -> 恢复平台线程状态，race acquire
  -> KeepAlive(fn/arg/mp)
```

`incgo` 阻止调度器在仍使用该 M 的 g0 作为 C 调用栈时错误切换；`ncgo` 帮助 traceback 判断栈上是否有 C 帧。`KeepAlive` 处理 C 回调带来的“GC 观察时间倒退”：回调退出重新进入 C 时，保存的 syscall PC/SP 会回到较早位置，参数必须在整个窗口内保持活跃。

### C回调Go

当 C 在同一调用链中回调导出的 Go 函数时，runtime 需要从 C ABI 回到 Go ABI，并恢复原 G 的可执行上下文：

```text
Go -> C
       |
       +-> exported Go callback
             -> cgocallback
             -> 恢复/建立 Go 执行上下文
             -> exitsyscall 类协作，获取 P
             -> 执行 Go callback
             -> 回到 C
       |
       +-> 回到原 Go 调用
```

嵌套回调使状态保存比单向调用更复杂。runtime 要维护 traceback、panic 边界和线程绑定关系，不能只做一次普通函数跳转。

[`cgocallbackg`](https://github.com/golang/go/blob/go1.26.2/src/runtime/cgocall.go#L313) 先根据 C SP 更新 extra M 的 g0 边界，再执行 `lockOSThread`。锁定必须发生在 `exitsyscall` 之前，否则重新获取 P 时 G 可能迁移到另一个 M，而 C 调用仍使用原 M 的 g0 栈。

它把 `syscallsp/syscallbp` 保存为 `unsafe.Pointer` 局部变量，而不是 `uintptr`。这样回调中的 Go 代码导致栈搬迁时，stack map 能识别并修正它们；返回 C 前 `reentersyscall` 再恢复原 syscall 现场。这是 Go 指针类型信息直接影响 runtime 正确性的实例。

### cgo指针规则

Go GC 需要知道哪些 Go 指针仍然可达，而 C 内存对 GC 不透明。因此边界规则的核心是：不能让 C 在 runtime 不知情的情况下长期保存指向含 Go 指针内存的引用。

实践中应遵循：

- 不让 C 在调用返回后保留未经允许的 Go 指针。
- 不把包含 Go 指针的 Go 内存暴露给 C 长期持有。
- 长期句柄使用 `runtime/cgo.Handle` 等显式机制。
- C 缓冲区和 Go 缓冲区的所有权、生命周期要明确。
- 使用当前 Go 版本的 cgo 检查工具验证边界。

这些限制不是因为当前 GC 一定移动堆对象，而是为了保证 GC 可达性、栈增长和未来实现都保持正确。

规则还要区分“传递期间临时可见”和“调用返回后由 C 保留”。`runtime.Pinner` 可以在明确生命周期内固定特定 Go 对象，但不能让包含未固定 Go 指针的对象图整体变得合法；字符串、slice、interface 等值本身包含 Go 指针，也不能只看其数据区地址。

长期由 C 保存的逻辑引用优先使用 [`runtime/cgo.Handle`](https://pkg.go.dev/runtime/cgo#Handle)，把整数句柄传给 C，回调时再在 Go 侧解析。调试时使用当前版本支持的 `GOEXPERIMENT=cgocheck2` 构建进行更完整检查；`GODEBUG=cgocheck` 的可用值和开销应以对应版本文档为准。

### 性能影响

cgo 的成本来自边界切换和系统协作，而不只是一次 call 指令：

- Go/C ABI 与栈切换。
- entersyscall/exitsyscall 状态维护。
- 指针检查和生命周期约束。
- 线程增加及调度局部性下降。
- 回调、signal 转发和 profiling 复杂度。

优化方向通常是减少跨边界次数、批量传输数据、避免在高频小操作上往返，并用 profile 验证瓶颈，而不是假定所有 cgo 调用都一定昂贵。

## 外部线程与Signal协作

### 非Go线程回调Go

C 自己创建的线程没有 Go runtime 所需的 M、g0、gsignal 和 TLS。进入 Go 前，runtime 通过 `needm` 等路径为当前线程临时取得或建立上下文：

```text
C-created thread
  -> callback trampoline
  -> needm
       -> 从 extra M 池取得 M
       -> 设置 TLS
       -> 准备 g0/gsignal 和 signal 状态
  -> 获取 P 并执行 Go callback
  -> 返回 C
       -> pthread 平台：保留线程绑定，在线程退出时由 TLS destructor 调用 dropm
       -> 非 pthread 平台：本次回调结束时调用 dropm
  -> dropm 清理线程关联，并把 M 放回 extra M 池
```

runtime 预留 extra M，是因为回调到达时可能处于无法安全执行普通 Go 分配的边界；必须先有一个可用 M，才能进入常规 runtime 世界。

Go 1.26.2 的 [`needm`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L2385) 顺序强调异步安全：

```text
保存并阻塞当前 C 线程的 signal mask
  -> getExtraM（此时不能临时分配 M）
  -> 设置 needextram，推迟补充池
  -> osSetupTLS
  -> setg(m.g0)，估算/更新 system stack 边界
  -> asminit + minit：建立 signal stack 等线程状态
  -> extra G: _Gdeadextra -> _Gsyscall
  -> 进入 cgocallbackg 后 exitsyscall 获取 P
```

extra M 池必须预先至少有一个元素，因为尚未建立 M/TLS 时无法安全进入普通分配器。若取走最后一个，`needm` 只设置 `needextram`；等回调进入有 P 的 Go 环境后，`cgocallbackg1` 才调用 `newextram` 补货。

[`dropm`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L2583) 把 extra G 从 `_Gsyscall` 退回 `_Gdeadextra`，清理 trace 和线程关联。pthread 平台通常通过线程局部析构在外部线程真正退出时 drop，避免每次回调重复安装 signal stack，也避免外部线程退出后永久泄漏 M。

### Runtime如何创建线程

调度器需要更多 M 时会经过 `newm` 和平台相关 `newosproc` 创建线程。在 Linux/amd64 上，clone trampoline 先安装 TLS，把当前 g 设置为 `m.g0`，然后调用 `mstart`。`mstart0 -> mstart1` 再执行 `asminit` 和 `minit`；备用 signal stack 与线程 signal mask 正是在 `minit` 中建立，之后线程取得 P 并进入 `schedule`。

```text
clone child
  -> 安装 TLS，set g = m.g0
  -> mstart -> mstart0 -> mstart1
  -> asminit + minit：安装 signal stack/mask
  -> acquire P
  -> schedule
```

线程数量不等于 P 数量。阻塞 syscall、cgo 和锁定线程都可能让 M 增多；`GOMAXPROCS` 只控制并行执行普通 Go 代码的 P 数量。

### LockOSThread

默认情况下 G 可以在不同 M 上继续执行。如果 C 库、GUI 框架或线程局部状态要求线程亲和性，可以使用：

```go
runtime.LockOSThread()
defer runtime.UnlockOSThread()
```

锁定后，当前 G 只能在对应 M 上运行。它不等于“禁止调度”，也不是性能优化：G 阻塞时 runtime 可能需要保留或补充线程，降低线程复用能力。

若在 `init` 或 main goroutine 的早期路径锁定线程，还需要仔细验证框架对“进程主线程”与“当前 OS 线程”的具体要求。

## 源码入口

| 目标 | 入口 |
| --- | --- |
| Unix信号策略与分发 | [`runtime/signal_unix.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/signal_unix.go#L646) |
| Linux/amd64 trampoline | [`runtime/sys_linux_amd64.s`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sys_linux_amd64.s) |
| os/signal队列 | [`runtime/sigqueue.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sigqueue.go#L71) |
| 异步抢占 | [`runtime/preempt.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/preempt.go)、`doSigPreempt` |
| Go调C / C回调Go | [`runtime/cgocall.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/cgocall.go#L134) |
| extra M | [`needm`、`dropm`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L2385) |
| cgo指针检查 | [`runtime/cgocheck.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/cgocheck.go) |

## 可复现实验

实验应先回答两个可观察问题：长时间 Go->C 调用期间其他 G 是否仍有 P 可用；C 创建的 pthread 第一次和后续回调 Go 时，线程与 extra M 的关系如何变化。再单独发送用户 signal 或采集 CPU profile，确认 handler 记录的现场可能来自 Go、runtime、syscall 或 C：

```bash
go test -race ./...
GOEXPERIMENT=cgocheck2 go test ./...
go test -run TestCallback -trace=trace.out ./...
go tool trace trace.out
go tool pprof -http=:0 cpu.pprof
```

回调实验应至少覆盖 Go->C、C->Go、C 创建 pthread 后回调 Go、嵌套回调四种路径，并记录 `runtime.NumCgoCall`、线程 profile 和 trace。CPU profile 中 C 栈的完整度依赖外部代码的 unwind 信息、链接方式和平台，不应把缺少 C frame 直接解释成 C 没有消耗 CPU。

## 延伸阅读

- [os/signal包文档](https://pkg.go.dev/os/signal)
- [cgo命令与指针传递规则](https://pkg.go.dev/cmd/cgo)
- [runtime/cgo.Handle](https://pkg.go.dev/runtime/cgo#Handle)

## 系列导航

- [系列目录](/posts/go-runtime/)
- [上一篇：Netpoll与Syscall](/posts/go-runtime-06-netpoll-syscall/)
- 当前：Go Runtime（七）：Signal与cgo线程管理
- [下一篇：Channel与Select](/posts/go-runtime-08-channel-select/)
