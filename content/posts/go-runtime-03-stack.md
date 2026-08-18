+++
date = '2026-08-18T00:12:00+08:00'
draft = false
title = 'Go Runtime（三）：Goroutine栈管理'
tags = ['Go', 'Runtime', 'Stack']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64**。文中的源码链接固定到 `go1.26.2` tag；其他版本和架构可能具有不同的汇编入口、常量或实现细节。

goroutine 能够以较低成本大量创建，一个重要原因是它不需要预留传统线程栈那样大的连续空间。Go 为普通 G 使用可增长的连续栈：空间不足时分配更大的栈、复制仍在使用的部分、修正指针，再从原调用点继续执行。

本文只讨论 goroutine 栈的运行时管理。对象为什么逃逸到堆由编译器决定，堆对象如何分配放在[下一篇：内存分配器](/posts/go-runtime-04-allocator/)中。

## 三类栈不要混淆

一个 M 至少关联三种执行上下文：

| 上下文 | 作用 | 是否普通调度G |
| --- | --- | --- |
| 用户 G 的栈 | 执行用户 Go 函数，可增长和缩小 | 是 |
| `m.g0` 的系统栈 | 调度、栈扩容、部分 runtime 内部路径 | 否 |
| `m.gsignal` 的信号栈 | Unix 信号处理 | 否 |

`g0` 和 `gsignal` 都是 `g`，但它们不是放进运行队列的普通 goroutine。`newstack` 必须在 g0 上运行，因为当前用户栈已经被判定为不足，不能继续依赖它完成内存分配和现场搬迁。

## g中的栈字段

[`runtime.g`](https://github.com/golang/go/blob/go1.26.2/src/runtime/runtime2.go#L473) 的栈相关字段是理解整条链路的入口：

```go
type g struct {
	stack       stack
	stackguard0 uintptr
	stackguard1 uintptr

	m         *m
	sched     gobuf
	syscallsp uintptr
	syscallpc uintptr
	// ...
}

type stack struct {
	lo uintptr
	hi uintptr
}
```

`stack` 描述实际区间 `[lo, hi)`。Go 栈在 amd64 上向低地址增长，SP 越来越接近 `lo`。`stackguard0` 是普通 Go 函数入口用于比较的保护值；正常情况下约为 `stack.lo + stackGuard`，也可以被设置成特殊值 `stackPreempt`，借用同一套入口检查触发协作式抢占。

`sched` 保存 G 暂停时的 PC、SP、BP 等调度现场。栈复制后，`sched.sp` 也必须迁移到新地址。

## 初始栈从哪里来

新 G 在 [`newproc1`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L5313) 中取得。runtime 先尝试从 P 的空闲 G 列表复用；没有可用 G 时，`malg` 创建结构并按需要分配初始栈。

普通小栈由 [`stackalloc`](https://github.com/golang/go/blob/go1.26.2/src/runtime/stack.go) 管理。固定大小的小栈可以利用每个 P 的 `mcache.stackcache`，缓存不足时再访问全局 `stackpool`；较大的栈通过页级堆分配。这个缓存属于栈分配器，不等同于小对象分配使用的 `mcache.alloc[spanClass]`。

```text
newproc1
  -> gfget：尝试复用已结束的 G
  -> malg：需要时创建 g
  -> stackalloc：分配初始连续栈
  -> 初始化 sched.sp/sched.pc
  -> _Gdead -> _Grunnable
```

## 编译器生成的栈检查

除 `NOSPLIT` 函数外，编译器会根据帧大小在函数入口生成栈检查。amd64 上不能把它理解成永远相同的两条指令：小帧、中帧和大帧会采用不同序列，以正确处理下溢和异步抢占特殊值。

概念条件是：

```text
guard := g.stackguard0
如果当前 SP 无法为目标栈帧留下足够空间
  -> 调用 runtime.morestack
否则
  -> 建立栈帧并执行函数体
```

真正的生成规则位于编译器的 [`cmd/internal/obj/x86/obj6.go`](https://github.com/golang/go/blob/go1.26.2/src/cmd/internal/obj/x86/obj6.go)；保护区常量定义在 [`runtime/stack.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/stack.go)。分析汇编时必须同时看目标架构和函数帧大小。

### 可复现实验

创建 `stackcheck.go`：

```go
package main

//go:noinline
func frame(x int) int {
	var buf [256]byte
	buf[0] = byte(x)
	return int(buf[0])
}

func main() { println(frame(7)) }
```

查看编译器输出：

```bash
go tool compile -S -N -l stackcheck.go
```

在 `main.frame` 开头可以观察 SP 与栈保护值的比较，以及失败后对 `runtime.morestack_noctxt` 的调用。`-N -l` 用于减少优化和内联干扰，不代表生产构建一定生成完全相同的指令。

## morestack如何切到g0

栈检查失败后进入架构相关的 [`runtime.morestack`](https://github.com/golang/go/blob/go1.26.2/src/runtime/asm_amd64.s)。它不能在当前栈上直接调用复杂 Go 代码，主要工作是：

1. 确认当前不是 g0 或 gsignal；这两类栈不能走普通增长路径。
2. 把用户 G 的返回 PC、SP 等保存到 `g.sched`。
3. 把调用者现场保存到 `m.morebuf`，供 `newstack` 判断和恢复。
4. 切换到 `m.g0`，调用 `runtime.newstack`。

```text
用户 G：函数入口检查失败
  -> morestack（汇编，保存现场）
  -> g0 栈
  -> newstack
  -> gogo(&gp.sched)
  -> 回到用户 G 的原调用位置重试
```

这里的“重试”很重要：增长完成后并不是从目标函数中间继续，而是恢复到能够再次完成入口检查和调用的位置。

## newstack同时处理扩容和抢占

[`newstack`](https://github.com/golang/go/blob/go1.26.2/src/runtime/stack.go#L1026) 入口首先读取一次 `gp.stackguard0`。如果它等于 `stackPreempt`，本次进入可能是协作式抢占，而非真实空间不足。

```text
newstack
  -> 验证 morebuf 对应当前用户 G
  -> 读取 stackguard0
  -> stackPreempt：检查 canPreemptM
       -> 可以抢占：进入调度路径
       -> 暂时不安全：恢复 guard 并继续执行
  -> 普通空间不足：计算新栈大小
  -> copystack(gp, newsize)
  -> gogo 恢复执行
```

runtime 持锁、正在分配、关闭抢占等状态下，`canPreemptM` 可能拒绝本次请求。拒绝不是丢失抢占意图；G 的抢占标记仍可使 runtime 在之后的安全位置重试。

扩容通常按倍数增加，以获得摊销后的常数复杂度。如果下一帧异常大，runtime 会继续扩大候选大小，直到容纳目标帧；超过 `maxstacksize` 或发生地址溢出则报告 stack overflow。

## copystack为什么不只是memmove

[`copystack`](https://github.com/golang/go/blob/go1.26.2/src/runtime/stack.go#L900) 先根据 `gp.sched.sp` 计算活动栈大小，再分配新栈和地址差值：

```text
old: [old.lo ........ old.hi-used | active frames | old.hi)
new: [new.lo ........ new.hi-used | active frames | new.hi)

delta = new.hi - old.hi
```

随后它要处理所有可能指向旧栈的引用：

- `sudog.elem` 等 channel 等待结构中的栈指针。
- `g.sched`、defer 链和 panic 链中的栈地址。
- 活动栈帧内被编译器标记为指针的槽位。
- traceback 和上下文恢复依赖的寄存器保存值。

核心顺序是：

```text
计算 delta
  -> 与活跃 channel 操作同步并调整 sudog
  -> memmove 活动栈
  -> adjustctxt / adjustdefers / adjustpanics
  -> 切换 gp.stack、stackguard0、sched.sp
  -> unwinder 遍历新栈，adjustframe 修正帧内指针
  -> 释放旧栈
```

帧内指针修正依赖编译器生成的 stack map。GC 和栈搬迁都必须区分“真实指针”和“数值碰巧像地址的整数”；这也是把 Go 指针转换成 `uintptr` 后跨越栈增长点非常危险的原因。

## Channel为什么让栈复制更复杂

阻塞的 channel 发送或接收可能通过 `sudog.elem` 指向某个 G 的栈槽。对应 G 已经 park，但另一方可能正在直接向该槽位复制数据。

`g.activeStackChans` 和 `g.parkingOnChan` 用于描述这种状态。扩容由当前 G 自己触发，通常不应等待自己持有的 channel 锁；缩栈由其他 GC/扫描路径发起，则必须确认 park 过程完成，并在需要时获取相关 channel 锁。`copystack` 中的 `adjustsudogs`、`syncadjustsudogs` 正是在处理这种并发关系。

所以“G 停止运行”不自动意味着它的栈无人访问。

## 栈缩小

[`shrinkstack`](https://github.com/golang/go/blob/go1.26.2/src/runtime/stack.go#L1257) 复用 `copystack`，但前置条件比增长严格：

- 调用者必须通过 `_Gscan` 状态或 system stack 路径拥有该栈。
- `isShrinkStackSafe` 必须确认 channel park 等状态安全。
- libcall 中可能存在伪装成 `uintptr` 的栈指针，不能移动。
- `gcshrinkstackoff` 调试开关没有禁用缩栈。
- 当前使用量加 `stackNosplit` 必须小于栈容量四分之一。
- 新大小不能低于 `fixedStack`。

满足条件后，新大小为旧大小的一半，再调用 `copystack`。缩栈通常与 GC 扫描协作，而不是每次函数返回都发生，这避免了深浅调用交替时的频繁搬迁。

## 栈扫描与GC根

goroutine 活动栈是 GC 根。扫描器使用 PC 找到函数元数据和安全点，再用 stack map 判断局部变量、参数和保存寄存器中哪些槽位包含指针。

```text
暂停或取得目标 G 的栈所有权
  -> 从 sched/syscall 保存的 PC、SP 开始 unwind
  -> 找到每一帧对应的 stack map
  -> 扫描被标记为指针的槽位
  -> 将新发现的堆对象加入 GC work
```

处于 `_Gsyscall` 的 G 使用 `syscallpc/syscallsp`。在异步抢占入口，`asyncPreempt` 帧保存了被打断函数的寄存器，但被打断点未必有普通同步安全点的精确 stack map；扫描器会保守扫描 `asyncPreempt` 及其直接父帧，随后恢复对更老调用帧的精确扫描。`g.asyncSafePoint` 也会禁止缩栈，因为搬迁要求所有相关帧都具有精确指针图。栈管理、调度状态和 GC 不能彼此独立实现。

## NOSPLIT的真实约束

`//go:nosplit` 表示函数入口不执行普通栈检查。它用于 morestack 本身、信号 trampoline、调度切换等无法再次触发栈增长的路径。

它不是“更快函数”的通用注解。链接器会分析静态 nosplit 调用链是否超出预留空间；函数若包含可能增长栈、分配或进入不受控调用的行为，使用 nosplit 会破坏 runtime 的安全假设。

## 如何验证栈增长

可以在深递归前后读取同一个局部变量的地址，观察包含它的栈帧是否被整体迁移：

```go
package main

import (
	"fmt"
	"runtime"
	"unsafe"
)

//go:noinline
func grow(depth int) {
	var buf [1024]byte
	if depth > 0 {
		grow(depth - 1)
	}
	runtime.KeepAlive(buf)
}

//go:noinline
func observe() {
	var marker byte
	before := uintptr(unsafe.Pointer(&marker))
	grow(10000)
	after := uintptr(unsafe.Pointer(&marker))
	fmt.Printf("before=%#x after=%#x moved=%v\n", before, after, before != after)
}

func main() { observe() }
```

这里把 `uintptr` 只当成数值打印，绝不在可能的栈增长后把 `before` 转回指针。若 `moved=true`，说明 `observe` 的活动栈帧在递归期间被复制；使用 `GODEBUG=gcshrinkstackoff=1` 还能隔离 GC 缩栈的影响。单次未移动不能证明栈不会增长，因为初始容量、编译后的帧大小和调用前已有栈空间都会影响结果。

## 源码阅读顺序

1. [`runtime2.go: g`](https://github.com/golang/go/blob/go1.26.2/src/runtime/runtime2.go#L473)：确认栈边界和保存现场。
2. [`asm_amd64.s: morestack`](https://github.com/golang/go/blob/go1.26.2/src/runtime/asm_amd64.s)：理解用户 G 到 g0 的切换。
3. [`stack.go: newstack`](https://github.com/golang/go/blob/go1.26.2/src/runtime/stack.go#L1026)：区分抢占与扩容。
4. [`stack.go: copystack`](https://github.com/golang/go/blob/go1.26.2/src/runtime/stack.go#L900)：追踪外部引用和帧内指针修正。
5. [`stack.go: shrinkstack`](https://github.com/golang/go/blob/go1.26.2/src/runtime/stack.go#L1257)：理解栈所有权与缩小阈值。
6. 编译器架构后端：验证目标平台的栈检查序列。

## 结论

连续栈把普通函数调用保持为传统 ABI 形式，把复杂度集中到少数增长点。它能够成立依赖三个前提：编译器生成精确 stack map，runtime 能在 g0 上安全搬迁现场，调度器和同步原语能够声明栈是否仍被其他执行者引用。

因此，goroutine 栈“小且可增长”不是一个孤立技巧，而是编译器、调度器、GC 和 channel 共同维护的协议。

## 延伸阅读

- [Go语言的栈内存和逃逸分析](https://draven.co/golang/docs/part3-runtime/ch07-memory/golang-stack-management/)：用于理解设计与历史，具体常量和安全条件以 Go 1.26.2 为准。
- [Go逃逸分析诊断说明](https://pkg.go.dev/cmd/compile)

## 系列导航

- [上一篇：GMP与Goroutine调度](/posts/go-runtime-02-scheduler/)
- 当前：Go Runtime（三）：Goroutine栈管理
- [下一篇：内存分配器](/posts/go-runtime-04-allocator/)
- [原始长文](/posts/go-runtime/)
