+++
date = '2026-08-18T00:11:00+08:00'
draft = false
title = 'Go Runtime（二）：GMP与Goroutine调度'
tags = ['Go', 'Runtime', 'Scheduler']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64**。调度器变化频繁；函数名、队列容量和分支顺序均以固定版本源码为准。

Go 调度器解决的核心问题是：怎样把大量 goroutine 映射到较少的 OS 线程，同时控制竞争、阻塞和调度延迟。

理解调度器不需要记住所有函数。先抓住三件事：执行资格属于 P，等待发生在 G 上，真正运行机器指令的是 M。

## GMP模型

```text
G（goroutine）
  栈、入口函数、调度现场、状态、等待原因

M（machine）
  OS线程、当前G、g0、信号处理上下文

P（processor）
  本地运行队列、runnext、mcache、timer、执行资格
```

抽象关系如下：

```text
          P.local runq
        [G][G][G][G]
             |
             v
M(OS thread) + P  ---->  执行某个 G
             |
          P.runnext
```

M 必须绑定 P 才能执行普通 Go 代码。没有 P 的 M 可以停在线程阻塞、syscall、cgo 或 runtime 的特定路径中，但不能继续执行任意用户 Go 代码。

`GOMAXPROCS` 控制 P 的数量。它限制的是并行执行 Go 代码的能力，不限制 goroutine 或 OS 线程的总数。

[`runtime.p`](https://github.com/golang/go/blob/go1.26.2/src/runtime/runtime2.go#L772) 直接体现了 P 不只是抽象的“逻辑处理器”：

```go
type p struct {
	id          int32
	status      uint32
	m           muintptr
	mcache      *mcache
	pcache      pageCache
	schedtick   uint32
	syscalltick uint32
	runqhead    uint32
	runqtail    uint32
	runq        [256]guintptr
	runnext     guintptr
	gcw         gcWork
	// timers、sudog cache、defer pool、trace state ...
}
```

本地队列容量 256 是 Go 1.26.2 的实现事实，不是 GMP 模型的语言规范。`mcache`、页缓存、GC work 和 timer 同样绑定 P，解释了 syscall 中为什么要转移整个 P，而不是只搬走运行队列。

## Goroutine状态主线

常见状态可以压缩为：

```text
                schedule
 _Grunnable ----------------> _Grunning
      ^                            |
      |                            | gopark / syscall / preempt
      | goready                    v
 _Gwaiting <------------------ 阻塞条件

 _Grunning -- goexit --> _Gdead
 _Grunning -- syscall --> _Gsyscall -- exitsyscall --> runnable/running
```

状态转换比状态名称本身重要：谁执行转换、G 被放到哪里，以及此时 M/P 是否仍然绑定，决定了后续调度行为。

## 创建：go语句做了什么

`go f(x)` 不会立即创建线程，也不保证 `f` 立刻执行。编译器准备入口和参数信息，runtime 创建或复用一个 G：

```text
go f(x)
  -> newproc
       -> gfget：尝试复用空闲 G
       -> malg：必要时创建 G 和初始栈
       -> 设置入口 PC、参数和 goexit 返回点
       -> _Gdead -> _Grunnable
       -> runqput：放入当前 P 的队列
```

新 G 通常优先进入当前 P 的 `runnext` 或本地队列。`runnext` 使刚唤醒或刚创建且相关性较高的 G 有机会较快运行，但调度器也会通过全局队列检查避免它长期破坏公平性。

Go 1.26.2 的真实入口 [`newproc`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L5295) 很短：

```go
func newproc(fn *funcval) {
	gp := getg()
	pc := sys.GetCallerPC()
	systemstack(func() {
		newg := newproc1(fn, gp, pc, false, waitReasonZero)
		pp := getg().m.p.ptr()
		runqput(pp, newg, true)
		if mainStarted {
			wakep()
		}
	})
}
```

`systemstack` 说明创建过程在 g0 上完成；`runqput(..., true)` 表示优先尝试 runnext；`wakep` 只在启动完成后考虑唤醒额外 P。`newproc1` 通过 `gfget` 复用 `_Gdead`，必要时 `malg(stackMin)`，再设置 `sched.pc/sched.sp`、父 G、创建 PC、goid 和 trace 状态，最后发布为 `_Grunnable`。

## 从哪里找可运行的G

调度循环最终需要得到一个 runnable G。来源包括：

1. 当前 P 的 `runnext` 和本地运行队列。
2. 全局运行队列。
3. netpoll 已就绪的网络等待者。
4. 到期 timer 唤醒的 G。
5. 从其他 P 的本地队列窃取的 G。

Go 1.26.2 的 [`findRunnable`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L3389) 包含大量维护分支。保留主干后的实际顺序是：

```text
STW / per-P safe point
  -> 当前 P 的 timer
  -> trace reader
  -> GC mark worker
  -> 每 61 个 schedtick 检查一次全局队列
  -> finalizer / cleanup G
  -> 当前 P 的 runnext 和本地 runq
  -> 全局队列，并批量搬一部分到本地
  -> 非阻塞 netpoll
  -> spinning M 偷取 G 和 timer
  -> idle GC marking
  -> 仍无任务则释放 P，让 M 休眠
```

源码还有 cgo yield、GC stop、调试冻结和最终阻塞 netpoll 等分支。这里不是定义永不变化的优先级，而是强调固定版本中，本地 runq 也不是所有维护任务之前的绝对第一选择。

## 本地队列与工作窃取

每个 P 持有本地队列，常见入队和出队不需要竞争全局锁。队列溢出时，一部分 G 会转移到全局队列。

当某个 P 没有工作时，它可以从其他 P 窃取一部分 runnable G：

```text
P0: [G G G G G G]       P1: []
          |                 |
          +---- steal ----->+

P0: [G G G]             P1: [G G G]
```

窃取一批而不是一个 G，可以摊薄同步成本；保留一部分给原 P，避免把局部性全部破坏。

## g0、mcall与schedule

普通 G 使用可增长栈执行用户代码，调度器和栈管理的关键路径则需要稳定的系统栈。每个 M 都有一个 g0：

```text
用户 G
  -> mcall(fn)
  -> 切换到当前 M 的 g0 栈
  -> fn 保存或改变用户 G 状态
  -> schedule 选择下一个 G
  -> execute/gogo 恢复目标 G
```

阻塞函数不能在把当前 G 挂起后继续依赖它的用户栈执行调度，因此需要通过 `mcall` 切到 g0。

## 阻塞与唤醒

channel、锁、timer 和 netpoll 最终都会使用相似的 park/unpark 模型：

```text
当前 G 无法继续
  -> gopark
  -> 在 g0 上把 G 改为 _Gwaiting
  -> M/P 去执行其他 G

等待条件满足
  -> goready/ready
  -> _Gwaiting -> _Grunnable
  -> 放入运行队列
```

阻塞的是 G，不一定是 M。等待网络数据时尤其如此：G 被挂到 pollDesc，M 可以继续执行别的工作。

[`gopark`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L445) 自己不直接修改为 `_Gwaiting`。它先把解锁回调、等待原因和 trace 信息写到 M，然后通过 `mcall(park_m)` 切到 g0。`park_m` 才执行：

```text
trace GoPark
  -> casgstatus(_Grunning, _Gwaiting)
  -> dropg：解除 G 与 M 的双向关联
  -> 执行 waitunlockf
       -> 返回 false：撤销 park，直接 execute 原 G
       -> 返回 true：schedule 选择其他 G
```

解锁回调解决“登记等待”和“释放条件锁”之间的丢失唤醒窗口。唤醒端的 `goready` 通过 `systemstack` 进入 `ready`，完成 `_Gwaiting -> _Grunnable` 并入队；它不会在调用者栈上直接运行目标 G。

## schedule如何交付执行权

[`schedule`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L4135) 必须运行在 g0。它先拒绝“持有 runtime 锁却试图调度”和“仍在 cgo 调用中调度”等非法状态，再调用 `findRunnable`。找到 G 后仍需处理三种约束：

- spinning M 要在执行前撤销 spinning 状态，并可能补充另一个自旋 M。
- 用户调度若被临时禁用，普通 G 要移入等待重新启用的队列。
- 目标 G 若绑定了 locked M，需要通过 `startlockedm` 移交 P。

普通路径最后调用 [`execute`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go)：

```text
G: _Grunnable -> _Grunning
  -> gp.m = mp，mp.curg = gp
  -> 更新 P.schedtick 和时间片继承状态
  -> gogo(&gp.sched)
  -> 在目标 G 栈上恢复 PC/SP
```

`schedule`、`execute` 和 `gogo` 分别承担选择、状态交接和机器现场恢复，不能把三者统称为一次普通函数调用。

## 抢占

只有协作式调度时，长时间不调用函数的计算循环可能长期占用 P。现代 Go 同时使用协作式与基于信号的异步抢占。

### 协作式抢占

runtime 设置抢占请求，并修改目标 G 的栈保护值。G 到达函数入口栈检查或其他安全点时进入 `morestack` 路径，发现抢占请求后让出执行权。

优点是现场清晰、安全点明确；限制是没有合适检查点的代码可能响应较慢。

### 异步抢占

`sysmon` 发现 G 运行过久后，可以请求向目标 M 发送抢占信号。信号处理器检查当前 PC 是否位于可安全抢占的位置，满足条件时修改执行现场，使线程返回后进入抢占处理路径。

```text
sysmon 发现长时间运行
  -> 标记抢占并向 M 发信号
  -> signal handler 检查安全性
  -> 调整返回现场
  -> G 进入调度器
```

异步不等于任意指令都能停。runtime、写屏障和持锁等危险位置可能拒绝本次抢占，稍后再试。

## sysmon的角色

`sysmon` 不依赖普通 P 执行，负责观察和推动若干后台事件：

- 发现长时间运行的 G并发起抢占。
- 从长期 syscall 中夺回 P。
- 当普通 P 长时间忙碌时执行非阻塞 netpoll，注入已就绪 G。
- 用 `timeSleepUntil` 决定自身深度休眠期限，并在需要时唤醒相关调度路径；到期 Timer 的实际执行仍由持有 P 的 timer 检查路径完成。
- 协助 GC 和调度状态维护。

它不是调度器的唯一入口，也不直接替代普通 schedule 循环；它负责处理正在运行的 M/P 不方便自行发现的问题。

## Syscall只保留调度结论

goroutine 进入可能阻塞的 syscall 时会切到 `_Gsyscall`。如果调用很快返回，M 可能继续使用原 P；如果阻塞变长，`sysmon` 可以 retake P，让其他 M 执行队列中的 G。返回 syscall 的 M 若拿不到 P，就把 G 放回 runnable 队列。

这里理解 G/M/P 的所有权即可。`entersyscall`、`syscalltick`、`exitsyscall` 和 `RawSyscall` 的完整流程放在[第六篇：Netpoll与Syscall](/posts/go-runtime-06-netpoll-syscall/)，避免两篇重复解释同一机制。

## M如何休眠与被唤醒

没有工作时，M 不能带着 P 直接睡眠。它先交出 P，再由 [`stopm`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L2992) 把自己放进 idle M 链表并睡在 note 上；被唤醒后从 `m.nextp` 取得已经移交给它的 P。

[`startm`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L3035) 为一个 P 找 idle M；没有可复用 M 时才创建新线程。P 从调用者临时所有转移到目标 M 的窗口中必须禁用抢占，否则该 P 可能既不归运行 M，也无法响应 STW。

当 G 变为 runnable，[`wakep`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L3212) 只在当前没有 spinning M 时启动一个新的 spinning M。spinning M 暂时没有确定工作，会检查全局队列、netpoll 和其他 P 的队列；限制其数量可避免少量任务引发线程惊群。

[`handoffp`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L3131) 决定失去 M 的 P 应立即交给另一个 M，还是进入 idle P 列表。它不仅检查 run queue，还检查 GC worker、trace、STW、netpoll 和 Timer，所以“队列为空”不是 P 可以休眠的充分条件。

```text
G ready -> wakep -> startm(P, spinning=true)
                        |
                     M找工作
                        |
       找到G -> execute | 找不到 -> 交出P -> stopm
```

## STW如何停止所有P

[`stopTheWorld`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L1532) 先取得 `worldsema` 串行化多个 STW 请求，再切到 system stack 执行 [`stopTheWorldWithSema`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L1628)。核心停止单位是 P，不是遍历所有 G 逐个暂停：

```text
sched.gcwaiting = true，stopwait = gomaxprocs
  -> preemptall请求运行中的P到安全点
  -> 当前P进入_Pgcstop
  -> 夺回或停止syscall相关P
  -> idle P直接进入_Pgcstop
  -> 等待剩余P自行确认停止
  -> 校验所有P状态
```

等待期间 runtime 每 100 微秒重新发起抢占，覆盖请求与状态转换的竞态。所有 P 到达 `_Pgcstop` 后，用户 Go 代码不再执行，runtime 才能安全完成 GC 阶段切换、堆快照等全局操作。

[`startTheWorldWithSema`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L1760) 先做一次非阻塞 netpoll，把积累的事件注入 runnable 队列，再按新的 `GOMAXPROCS` 调整 P，清除 `gcwaiting`，唤醒或创建 M 承载恢复的 P。STW 因此是一次调度器全局状态转换，而不只是一个全局布尔量。

## checkdead判断的不是“所有G都waiting”

[`checkdead`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L6367) 在调度器锁下先计算可运行用户代码的 M 数量，扣除 idle M、locked idle M 和 system M。只要仍有普通运行 M（包括正在承担阻塞 netpoll 的 M），就直接返回；不是先遍历所有 `_Gwaiting` 再下结论。

没有运行 M 时，它再处理 c-shared/c-archive、panic、extra M 和假时间等例外，检查用户 G 不应仍处于 runnable/running/syscall，最后查看各 P 的 Timer heap。真实时间 Timer 还在堆中时不报死锁，因为未来 deadline 可以重新启动执行；确认没有执行者和 Timer 后才报告 `all goroutines are asleep - deadlock!`。

这解释了为什么“主 G 等待、其余 G 在网络读”不是死锁，而所有 G 永久等待 nil Channel 且没有 Timer/IO 才会触发检测。

## 调度器源码入口

| 目标 | 入口 |
| --- | --- |
| 核心结构 | [`runtime2.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/runtime2.go#L473) 中的 `g`、`m`、`p` |
| 创建 G | [`newproc`、`newproc1`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L5295) |
| 本地队列 | [`runqput`、`runqget`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L7478) |
| 调度主线 | [`schedule`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L4135)、[`findRunnable`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L3389)、`execute` |
| 阻塞唤醒 | [`gopark`、`goready`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L445)、`park_m`、`ready` |
| 工作窃取 | [`stealWork`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L3828)、`runqsteal` |
| 抢占 | [`preemptone`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L6866)、`preemptM`、`asyncPreempt` |
| 系统监控 | [`sysmon`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L6486)、`retake` |
| M/P休眠与唤醒 | [`stopm`、`startm`、`handoffp`、`wakep`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L2992) |
| STW与死锁检测 | [`stopTheWorldWithSema`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L1628)、[`checkdead`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L6367) |

## 可复现实验

用下面的程序制造 runnable、timer 和 syscall 混合负载：

```go
package main

import (
	"runtime"
	"syscall"
	"time"
)

func main() {
	runtime.GOMAXPROCS(2)
	for range 20 {
		go func() {
			for {
				runtime.Gosched()
			}
		}()
	}
	var pipe [2]int
	if err := syscall.Pipe(pipe[:]); err != nil {
		panic(err)
	}
	defer syscall.Close(pipe[0])
	defer syscall.Close(pipe[1])

	go func() {
		var b [1]byte
		_, _ = syscall.Read(pipe[0], b[:]) // 阻塞线程，供sysmon观察和retake
	}()
	time.AfterFunc(500*time.Millisecond, func() {
		_, _ = syscall.Write(pipe[1], []byte{1})
	})
	time.Sleep(3 * time.Second)
}
```

```bash
GODEBUG=schedtrace=1000,scheddetail=1 go run main.go
```

该实验只适用于 Unix/Linux：它故意用阻塞 pipe 的原始 syscall 占住一个 M，再由 Timer 解除阻塞。重点观察 `gomaxprocs`、idle P、spinning thread、各 P 的 `runqsize` 和线程数变化。输出是采样快照，不能用单次结果证明严格调度顺序；需要结合 `go tool trace` 查看 G 状态转换和延迟分布。

## 延伸阅读

- [Go语言调度器与Goroutine实现原理](https://draven.co/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)：包含调度器演进背景；当前字段和流程需与 Go 1.26.2 对照。
- [Go语言系统监控的实现原理](https://draven.co/golang/docs/part3-runtime/ch06-concurrency/golang-sysmon/)

## 系列导航

- [上一篇：运行时架构与程序启动](/posts/go-runtime-01-bootstrap/)
- 当前：Go Runtime（二）：GMP与Goroutine调度
- [下一篇：Goroutine栈管理](/posts/go-runtime-03-stack/)
- [原始长文](/posts/go-runtime/)
