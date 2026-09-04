+++
date = '2026-08-18T00:19:00+08:00'
draft = false
title = 'Go Runtime（十）：Timer实现'
tags = ['Go', 'Runtime', 'Timer', 'Scheduler']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64**。Go 1.23 重写了 Timer Channel 的可回收性与 Stop/Reset 保证；新语义是否默认生效还取决于主模块的 `go` 版本和 `godebug`/`GODEBUG` 设置，不能只看工具链版本。

Timer 同时服务 `time.Sleep`、`time.Timer`、`time.Ticker`、`AfterFunc` 和网络 Deadline。到期、Stop、Reset 和 Channel 的语义还会受到模块版本与调试开关影响。

## 先锁定可观察语义

[`time.NewTimer`](https://github.com/golang/go/blob/go1.26.2/src/time/sleep.go#L143) 的源码仍写 `make(chan Time, 1)`。新模式（`asynctimerchan=0`）下 runtime 借助 `hchan.timer` 对外呈现同步、容量为零的语义；`asynctimerchan=1` 恢复 Go 1.23 以前的异步 Timer Channel 行为。因此不能只看 `make` 参数或 `go version` 推断 `cap(t.C)` 和 Stop/Reset 语义。

三种有效模式对程序呈现的差异是：

| `asynctimerchan` | Channel语义 | Stop/Reset返回后的旧值 | 未到期且不可达的Timer |
| --- | --- | --- | --- |
| `0` | 同步，`cap(C) == 0` | 保证不会收到 | 可以被GC回收 |
| `1` | Go 1.23以前的异步路径，`cap(C) == 1` | 可能收到 | 不能在到期或Stop前被GC回收 |
| `2` | runtime新实现，但呈现异步语义，`cap(C) == 1` | 可能收到 | 可以被GC回收 |

这里的“从 Go 1.23 起”是模块语义版本，不只是所用 toolchain 的版本。`asynctimerchan` 在 [`internal/godebugs` 表](https://github.com/golang/go/blob/go1.26.2/src/internal/godebugs/table.go#L30)中标记为 Go 1.23 变更项，旧于 `go 1.23` 的主模块默认取旧值 `1`；`go 1.23` 及更新主模块默认取新值 `0`。`go.mod`/`go.work` 的 `godebug` 指令、源码中的 `//go:debug` 或进程环境 `GODEBUG` 都可以显式覆盖它。调试 Timer 时必须记录最终构建配置。

`GODEBUG=asynctimerchan=1` 恢复旧式缓冲 Channel、不可回收的未到期 Timer 和旧 Stop/Reset 规则。它是迁移开关，不应作为新代码的正常依赖。Go 1.26 还保留 `asynctimerchan=2` 作为 runtime 调试模式；它用于区分问题来自新 Timer 实现还是同步 Channel 语义。

开始读实现前，还应固定以下边界：

- Timer 到期只表示任务可以执行，不提供硬实时保证；实际回调还受调度、STW 和系统负载影响。
- 每个 Timer 不对应一个 goroutine；`AfterFunc` 到期时才会另外启动执行用户函数的 G。
- `Ticker` 不补发所有错过的 tick；周期计算会跳过历史区间。
- 新模式下无需为了帮助 GC 而专门 Stop，但业务取消仍应 Stop。
- `AfterFunc.Stop` 返回 false 不表示回调已经结束。
- Stop 后 drain 的旧写法只适用于异步语义，不能脱离最终生效的模式照搬。

## time包如何进入runtime

`time` 包通过 runtime 提供的私有入口创建和修改 Timer：

```text
time.Sleep       -> runtime.timeSleep
time.NewTimer    -> runtime.newTimer(sendTime, channel)
Timer.Stop       -> runtime.stopTimer
Timer.Reset      -> runtime.resetTimer
time.AfterFunc   -> runtime.newTimer(goFunc, nil)
net deadline     -> runtime timer callback
```

Timer 不是一个独立后台 goroutine 加一张全局表。Go 1.26 把真实时间 Timer 组织在每个 P 的四叉最小堆中，由调度器和 netpoll 共同决定睡到何时、何时执行到期回调。实现难点不只是“按时间排序”，还包括并发 Stop/Reset、跨 P 修改、周期跳期、Channel 旧值以及 GC 可达性。

## timer对象

[`timer`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L55) 的核心字段可以分成四组：

```text
并发状态
  mu, astate, state

时间与回调
  when, period, f, arg, seq

堆归属
  ts *timers

Channel协议
  isChan, blocked, sendLock, isSending
```

`when` 是下一次触发的单调时间，零表示 inactive；`period > 0` 表示周期 Timer。回调统一为 `f(arg, seq, delay)`，其中 `delay` 是实际执行相对计划时刻的延迟。

`state` 受 Timer 锁保护，`astate` 是供无锁观察的副本。主要标志包括 `timerHeaped`、`timerModified`、`timerZombie`。`ts` 指向拥有它的 Timer 集合；对象可能被其他 P 修改，所以“每 P 所有”不代表无需锁。

## 每P的timers

[`timers`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L131) 位于 P 中，主要保存：

```text
mu                 heap锁
heap               四叉最小堆
len                原子可见长度
zombies            已停止但仍留在堆中的数量
minWhenHeap        堆中已同步条目的最早时间
minWhenModified    尚未调整条目的最早时间
```

每 P 一组堆分散了插入和触发竞争。Timer 可能由别的 P Stop/Reset，调度器也会查看非当前 P 的 timer，因此 `timers.mu` 与每个 `timer.mu` 都不可省略。

堆是四叉而不是二叉：父节点索引为 `(i-1)/4`，四个孩子从 `4*i+1` 开始。[`siftUp`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1355) 和 [`siftDown`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1379) 维持最小 `when` 位于根部。

## 创建与入堆

[`newTimer`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L389) 初始化 `timeTimer`、回调和关联 Channel，再调用 `modify`。需要入堆时，[`maybeAdd`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L681) 通过 `acquirem` 固定当前 M/P，避免在读取当前 P 与实际加锁插入之间被抢占到另一个 P。

```text
timer.modify
  -> needsAdd?
  -> acquirem，确定当前P
  -> lock P.timers
  -> cleanHead
  -> lock timer并复查needsAdd
  -> addHeap
  -> 必要时wakeNetPoller
```

[`addHeap`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L450) 会确保 netpoll 已初始化。原因不是 Timer 依赖网络事件，而是 runtime 用同一个 poller 进行带 deadline 的线程睡眠与提前唤醒；新 Timer 若早于当前休眠期限，必须唤醒 poller 重算等待时间。

## 调度器如何发现到期Timer

调度器在 `findRunnable` 等路径检查本地 P 的 Timer，并把最早唤醒时间合并进 `pollUntil`。没有 runnable G 时，M 可以在 netpoll 中睡到：

```text
min(网络事件到达, 最早Timer到期, 被其他M显式唤醒)
```

[`timers.check`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1009) 先通过原子维护的 `wakeTime` 做无锁快速判断。若到期或 zombie 比例过高，才获取堆锁、调整 modified 项并循环运行根节点。

[`timers.run`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1074) 只检查堆顶：先同步 modified/zombie 状态；若未到期返回下一时间；若已到期进入 `unlockAndRun`。最小堆保证无需扫描整个集合。

`sysmon` 和死锁检查使用 [`timeSleepUntil`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1322) 汇总所有 P 的最早时间。Timer 的存在意味着“当前没有 runnable G”不一定是死锁。

## 到期回调如何执行

[`unlockAndRun`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1119) 在 system stack 上准备下一状态，临时释放 Timer 集合锁后调用回调，再恢复锁约束。不能持有全堆锁执行任意回调，否则一个慢回调会阻塞同一 P 上所有 Timer 操作。

已经位于堆中的一次性 Timer 到期后，`unlockAndRun` 将 `when` 设为 0、短暂标记为 zombie，随即调用 `updateHeap` 把当前堆顶从所属堆中删除。这里的 zombie 是同一条到期路径中的过渡状态；`Stop` 则统一采用下面的延迟删除协议。周期 Timer 会计算下一次时刻：

```text
delay = now - when
next  = when + period * (1 + delay/period)
```

这个公式基于原计划时刻推进，并直接跳过已经错过的周期，不会为了“补发”而连续执行所有历史 tick。`time.Ticker` 的接收方仍可能观察到丢 tick 或合并后的节奏。

## Stop为何采用延迟删除

[`timer.stop`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L497) 可能操作属于另一个 P 的 Timer。为兼容这种跨 P 修改，Stop 对所有已经入堆的 Timer 使用统一协议：标记 `timerModified|timerZombie`，增加 `zombies`，并令 `when=0`，而不根据调用者是否恰好运行在所属 P 上尝试立即删除。

拥有该堆的 P 之后在 `adjust`、`cleanHead` 或处理根节点时删除 zombie。当本地堆 zombie 数超过长度四分之一，[`check`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1028) 会强制清理，防止大量 Stop 让堆持续膨胀。

延迟删除降低跨 P 修改成本，但需要 `astate`、`minWhenModified` 和 zombie 计数让调度器不会睡过新的最早期限。

## Reset如何跨P修改

[`timer.modify`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L572) 更新 `when/period/f/arg/seq`。若 Timer 已在堆中，它只标记 modified，并用原子 `minWhenModified` 发布一个可能更早的 deadline；堆顺序由拥有 P 稍后调整。

若新 deadline 早于 poller 当前等待时间，`wakeNetPoller(when)` 会唤醒休眠 M。若 Timer 此前不在堆且当前需要被调度，`maybeAdd` 将它放入当前 P 的堆。

`Reset` 的返回值表示调用前是否 active，而不是回调是否已经执行完。对 `AfterFunc`，旧回调可能已在另一个 goroutine 中运行，Reset 返回 false 后安排的新回调可能与它并发；需要完成性时必须由应用额外同步。

## 同步Channel语义如何实现

在 `asynctimerchan=0` 模式中，`Timer.Stop` 返回后，后续接收不会得到停止前遗留值；`Timer.Reset` 返回后，后续接收不会得到旧配置的值。runtime 用以下机制实现该保证：

- `hchan.timer` 把 Channel 操作连接到 Timer。
- `sendLock` 串行 Channel 发送与 Stop/Reset。
- `seq` 标识配置代次，过时回调在真正发送前重新校验。
- `isSending` 标识一次性 Timer 正在准备发送，帮助计算 Stop/Reset 返回值。
- `timerchandrain` 在锁序允许时清除旧的缓冲值。

在 `unlockAndRun` 中，回调准备发送前记录 `seq` 并增加 `isSending`；真正调用 `sendTime` 前获取 `sendLock` 再比较代次。并发 Stop/Reset 会先改变 `seq`，使旧发送变成空操作。

这就是“底层容量 1、公开同步语义”并存的原因。`chanlen/chancap` 对默认 Timer Channel 也做特殊处理，对用户呈现容量 0。

## Timer Channel按需入堆

默认同步模式下，没有 goroutine 阻塞接收的 Channel Timer 不一定需要留在堆中。[`blockTimerChan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1474) 增加 `blocked` 并在需要时入堆；[`unblockTimerChan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1516) 减少计数，最后一个 waiter 离开时可把 Timer 标为 zombie。

[`maybeRunChan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1442) 允许 Channel receive 或 `select` 在发现 Timer 已到期时就地推动回调。这既支持同步语义，也让不再可达、无人等待的 Channel Timer 能被 GC 回收。

## Ticker如何处理慢接收者

[`NewTicker`](https://github.com/golang/go/blob/go1.26.2/src/time/tick.go#L36) 与 Timer 共享 `timeTimer` 布局，把 `period` 设置为间隔并复用 `sendTime` 的非阻塞发送。runtime 的周期公式会直接跳到下一个尚未错过的计划时刻，不逐次补跑历史周期；新同步模式还会在接收方实际等待时才把 Channel Timer 放回堆，并用 `delay` 还原应观察到的 tick 时间。旧异步模式下若容量 1 的底层 Channel 已满，非阻塞发送会直接丢弃本次 tick。两种模式都不会累积无限补发队列，但具体 Channel 状态机不同。

`Ticker.Stop` 只令 Timer inactive，不关闭 `C`，避免并发接收者把关闭产生的零值误认为一次 tick。`Ticker.Reset` 要求正 duration，并从 Reset 时刻重新计算下一次到期。

## Sleep为何复用G上的Timer

[`timeSleep`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L330) 为当前 G 延迟初始化并复用一个 `timer`，回调为 `goroutineReady`。真实时间路径不是“先 reset 再 park”，而是把 [`resetForSleep`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L366) 作为 park commit 回调：只有 G 已经提交等待后才激活 Timer，避免极短 Sleep 在 G 尚未 park 时就先触发唤醒。`internal/synctest` 的 fake-time Timer 是例外，因为虚拟时间在 G park 前不会前进，可以先 reset 再 park。

```text
time.Sleep
  -> 取得/初始化 gp.timer
  -> 记录 gp.sleepWhen
  -> gopark(resetForSleep)
       -> G提交为waiting
       -> reset timer到sleepWhen
  -> timer到期调用goroutineReady(gp)
  -> G回到runnable队列
```

因此 `Sleep` 不占用一个 OS 线程，也不需要为每次调用创建业务 goroutine。复用 G 上的 Timer 还减少重复分配。

## AfterFunc为何不同

[`AfterFunc`](https://github.com/golang/go/blob/go1.26.2/src/time/sleep.go#L210) 不创建 Channel，runtime 到期回调最终执行 `goFunc`，由它启动新的 goroutine 调用用户函数。Timer 调度路径不会直接在 system stack 上运行任意用户函数。

`Stop` 只阻止尚未开始的回调，不等待已经启动的函数；Reset 安排的后续执行也可能与先前已经启动的函数并发，需要调用方自己协调。这与 Channel Timer 的“无旧值”保证是两种不同问题。

## 网络Deadline复用同一机制

`internal/poll` 为读写 deadline 安装 runtime Timer，回调最终调用 `netpollDeadline`，再由 `netpollunblock` 将等待 G 置为 ready。deadline 序号用于识别已被更新或取消的旧回调，思想上与 Timer Channel 的 `seq` 相同：异步执行前必须验证自己仍代表当前配置。

所以一次带 deadline 的网络等待会同时经过：pollDesc 状态机、runtime Timer、平台 netpoll 和 GMP 调度器。第六篇解释 fd 就绪路径，本文补上“时间到期”这一半。

## 源码阅读顺序

1. [`time.NewTimer/Stop/Reset/AfterFunc`](https://github.com/golang/go/blob/go1.26.2/src/time/sleep.go#L113)
2. [`timer` 与 `timers`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L55)
3. [`newTimer` 与 `addHeap`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L389)
4. [`stop` 与 `modify`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L497)
5. [`check`、`run` 与 `unlockAndRun`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1009)
6. [`maybeRunChan/blockTimerChan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1442)
7. [`findRunnable`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L3463)

## 观测与验证

先用一个最小测试确认当前构建实际呈现哪种 Timer Channel 模式：

```go
package timerlab

import (
	"testing"
	"time"
)

func TestTimerChannelMode(t *testing.T) {
	timer := time.NewTimer(time.Hour)
	t.Cleanup(func() { timer.Stop() })
	t.Logf("cap(timer.C)=%d", cap(timer.C))
}
```

```bash
GODEBUG=schedtrace=1000,scheddetail=1 go test ./...
GODEBUG=asynctimerchan=0 go test -run TestTimerChannelMode -v
GODEBUG=asynctimerchan=1 go test -run TestTimerChannelMode -v
GODEBUG=asynctimerchan=2 go test -run TestTimerChannelMode -v
go test -trace=trace.out ./...
go tool trace trace.out
go test -run '^$' -bench Timer -benchmem ./...
```

三次测试应依次观察到容量 0、1、1。容量能确认当前模式，却不能靠一次未收到旧值的运行证明 Stop/Reset 保证；这仍应由生效模式的公开契约决定。进一步对比时，应记录主模块 `go`/`godebug` 指令、未引用 Timer 的内存回收和大量并发 Reset 的 contention。不要用 `time.Sleep` 测量亚毫秒级硬实时精度；它验证的是“不早于 deadline”与调度延迟分布。

## 延伸阅读

- [time包文档](https://pkg.go.dev/time)
- [Go语言设计与实现：Timer](https://draven.co/golang/docs/part3-runtime/ch06-concurrency/golang-timer/)：适合了解旧实现与演进，Go 1.23 之后的 Channel 语义以当前源码为准。
- [Go 1.23 Timer Channel Changes](https://go.dev/wiki/Go123Timer)
- [runtime/time.go](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go)

## 系列导航

- [系列目录](/posts/go-runtime/)
- [上一篇：Runtime信号量与同步原语](/posts/go-runtime-09-sync-primitives/)
- 当前：Go Runtime（十）：Timer实现
- [下一篇：Context取消传播、Deadline与Cause](/posts/go-runtime-11-context/)
