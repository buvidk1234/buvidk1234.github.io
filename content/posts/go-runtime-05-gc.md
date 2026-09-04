+++
date = '2026-08-18T00:14:00+08:00'
draft = false
title = 'Go Runtime（五）：GC、写屏障与Pacer'
tags = ['Go', 'Runtime', 'GC']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64、默认 GOEXPERIMENT**。该版本的[实验 baseline](https://github.com/golang/go/blob/go1.26.2/src/internal/buildcfg/exp.go#L80) 已默认启用 Green Tea GC，即 `goexperiment.GreenTeaGC == true`；可用 `GOEXPERIMENT=nogreenteagc` 构建旧标记路径。两条路径共享 GC 阶段、根扫描、写屏障、assist、pacer 和 sweep，本篇会明确指出默认 Green Tea 特有的 span 批处理。

`gctrace` 中的 STW 暂停可能很短，请求尾延迟却仍会在分配突增时升高。原因是 GC 成本不只来自暂停，还包括并发标记占用的 CPU、分配 goroutine 承担的 assist，以及为目标堆大小预留的内存。

Go GC 的核心不是“三色”这个名词，而是在用户 goroutine 持续修改对象图时，怎样并发找出仍然可达的对象，并把这些暂停、CPU 与内存成本控制在目标范围内。本文从“什么条件触发一轮 GC”开始，沿同一轮周期解释工作由谁完成，以及这些成本最终在哪里被程序观察到。

一轮 GC 可以理解为四套机制协作：标记算法判断对象生死，写屏障维护并发正确性，分配 goroutine 通过 assist 偿还标记债务，pacer 控制何时开始以及投入多少资源。

## 模型与触发

### 为什么需要GC

函数返回只能自然回收栈帧，无法决定堆对象是否仍被其他对象引用：

```go
type Node struct {
	next *Node
}

func newNode() *Node {
	return &Node{}
}
```

对象可能通过全局变量、其他堆对象、goroutine 栈或 runtime 结构继续可达。GC 必须从一组根出发遍历引用图，未被发现的堆对象才可以回收。

### 根对象

典型根包括：

- 全局变量和包级数据中的指针。
- 各 goroutine 活动栈帧中的指针。
- 寄存器或保存执行现场中的指针。
- runtime 持有的特定对象引用。

编译器生成的类型位图和栈图告诉 GC 哪些位置是指针。Go 的标记器不会把任意看起来像地址的整数都当作指针，这提高了准确性，也要求 syscall、汇编和 cgo 遵守指针与栈管理规则。

### 三色标记模型

颜色是解释标记进度的模型，并不意味着每个对象真的保存一个三值颜色字段：

```text
白色  尚未被标记器发现
灰色  已发现，但它指向的对象尚未全部扫描
黑色  已发现并完成扫描
```

开始时把根引用的对象置灰，不断取出灰对象扫描其指针，把新发现的白对象置灰；扫描完成后当前对象变黑。灰队列为空时，仍为白色的对象不可达。

```text
roots -> A(gray)
          |
          +-> B(white)

scan A
  A -> black
  B -> gray
```

### 并发标记的问题

标记器工作时，用户 goroutine 也在修改指针。危险场景是：黑对象新指向白对象，同时原来通向该白对象的灰色路径被删除。标记器可能再也发现不了这个仍然存活的白对象。

```text
GC看到： black A       gray C -> white B
用户修改：A -> B       C -/-> B
```

如果没有额外机制，B 会被错误回收。因此并发标记必须监控某些指针写入，维持标记不变量。

### GC阶段

主流程可以概括为：

```text
触发 GC
  -> mark setup：短暂 STW，开启写屏障，准备根任务
  -> concurrent mark：恢复用户代码，并发扫描根和堆
  -> mark termination：再次 STW，完成剩余标记
  -> concurrent sweep：并发或按需清扫 span
```

Go GC 不是全程 STW。暂停主要用于建立一致的阶段边界；堆标记和清扫的大部分工作与用户代码并发执行。暂停时间、并发 CPU 成本和内存峰值是三种不同指标。

Go 1.26.2 在 [`runtime/mgc.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgc.go#L251) 中只维护三个全局 phase：

```go
const (
	_GCoff = iota
	_GCmark
	_GCmarktermination
)
```

`_GCoff` 同时覆盖“没有标记”和后台/按需清扫期。`setGCPhase` 在 `_GCmark` 与 `_GCmarktermination` 打开写屏障，其余阶段关闭。`gcBlackenEnabled` 与 phase 相关但不是同一个变量：它专门控制后台 mark worker 和 mutator assist 是否可以工作。

### GC如何触发

[`gcTrigger.test`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgc.go#L677) 定义三类触发条件：

| 类型 | 条件 | 常见来源 |
| --- | --- | --- |
| `gcTriggerHeap` | `heapLive >= trigger` | 分配路径 |
| `gcTriggerTime` | 距上轮超过 force GC 周期且 GOGC 未关闭 | `forcegchelper` |
| `gcTriggerCycle` | 尚未开始指定 cycle | `runtime.GC` |

三类 trigger 都要求 GC 已启用、当前不在 panic、phase 为 `_GCoff`。时间触发是防止低分配程序长期不回收，不代表固定周期必然发生 GC。

### runtime.GC如何强制触发

`runtime.GC()` 等待一个完整的强制 GC cycle，而不只是等到并发标记结束。若调用时上一轮仍在标记，它先等待其进入 sweep；随后触发下一轮，等待该轮 mark termination，并协助完成该轮 sweep，最后发布与已完成周期一致的 heap profile。

这个完成条件仍不包括“把进程 RSS 立即降到最低”。对象死亡、span 变为空闲、页被 scavenger 处理、操作系统更新 RSS 是不同阶段，因此手动频繁调用 GC 通常不是常规优化。排查内存时应先确认问题是对象仍可达、堆内复用、scavenge 延迟，还是容器统计口径。

## 标记引擎

### 一轮GC的实现主线

从源码职责看：

```text
gcStart
  -> 判断触发条件并进入 mark 阶段
  -> STW 和清扫准备
  -> 启用写屏障
  -> 生成 root jobs
  -> 启动 mark workers
  -> start the world

mark work
  -> background workers：gcDrain
  -> mutator assists：gcDrainN
  -> 扫描根、栈和灰对象
  -> 从对象 workbuf 与 Green Tea span queue 取得工作

mark completion
  -> 检测工作耗尽
  -> STW mark termination
  -> 关闭写屏障并准备 sweep
```

源码细节会随版本变化，阅读时应验证当前版本的阶段切换和 worker 调度条件。

[`gcStart`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgc.go#L733) 可能因为运行在 g0、持有 runtime 锁或禁止抢占而直接返回，因此“分配路径检测到 trigger”不等于当前调用必然启动成功。真正获得 `work.startSema` 后还要再次检查 trigger，防止多个 goroutine 同时观察到条件成立而重复启动。

Go 1.26.2 的开始阶段按源码顺序是：

```text
完成残余 sweep
  -> startSema 下复查 trigger
  -> 获取 gcsema 和 worldsema
  -> 检查每个 P 的 mcache.flushGen
  -> 启动后台 mark worker（worker 先 park）
  -> gcResetMarkState
  -> stopTheWorldWithSema(stwGCSweepTerm)
  -> finishsweep_m
  -> clearpools
  -> gcController.startCycle
  -> setGCPhase(_GCmark)：开启写屏障
  -> gcPrepareMarkRoots：生成根任务
  -> gcBlackenEnabled = 1
  -> startTheWorldWithSema
```

顺序不可随意交换：必须先完成上一轮 sweep 并清理 cache/pool，再建立本轮 mark 状态；写屏障开启后，mutator 才能在恢复执行时与并发标记保持不变量。

### 后台标记Worker

GC 使用不同模式的后台 worker 平衡吞吐与延迟。常见概念包括：

- Dedicated：在一段时间内专门执行标记。
- Fractional：按目标比例投入部分 CPU。
- Idle：只利用调度器暂时没有普通工作时的 CPU。

worker 本质上也是由调度器运行的 G。GC 控制器决定需要多少标记算力，调度器负责让这些 worker 在 P 上执行。

### 根扫描与栈扫描

根工作会被拆成任务供多个 worker 处理。扫描 goroutine 栈时，GC 依赖每个安全点的栈图识别活跃指针。

目标 G 若正在运行，runtime 需要先让它进入可扫描状态；这也是异步抢占能改善 GC 尾延迟的原因之一。已经停在等待、syscall 等状态的 G，则使用相应状态保存的 PC/SP 扫描。

栈缩小也可能与扫描阶段结合，但二者职责不同：扫描发现引用，缩小回收未使用的栈空间。

### 根任务如何切分

[`gcPrepareMarkRoots`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcmark.go#L109) 在 STW 中快照并编号根任务，包括 data/BSS 全局区、带 finalizer special 的 span、各 goroutine 栈以及少量固定根。大范围全局区按 `rootBlockBytes` 切块，span 根按 arena/chunk 切分，避免一个 worker 独占巨型根。

`work.markrootNext` 是全局原子任务游标；worker 领取编号后由 [`markroot`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcmark.go#L221) 映射到相应根类型。新 G 在快照后创建时初始没有用户根，之后产生的堆指针由并发标记期写屏障覆盖，所以无需无限扩展根快照。

扫描 G 栈前必须取得 `_Gscan` 状态，避免它同时移动栈或修改未受堆写屏障保护的栈引用。`scanstack` 使用编译器生成的 stack map 遍历活跃 frame，并可能在安全条件满足时顺便缩栈。

### gcWork、span queue与gcDrain

标记工作不会全部堆在一个全局队列中。每个 P 持有一个 `gcWork`：两组对象 `workbuf` 保存需要逐对象扫描的工作；默认 Green Tea 还增加本地 `spanq`，把适合批处理的小对象按 span 延迟聚合。两类本地工作都会在必要时与全局工作池交换或被其他 worker 窃取。

`gcDrain` 的核心循环是：

```text
先领取根任务
  -> 优先取本地对象 workbuf
  -> 再取本地/全局对象或 span 工作
  -> 按类型信息扫描对象或批量扫描 span
  -> 标记新发现的对象并加入相应工作队列
  -> 根据 worker 模式检查退出条件或工作耗尽
```

“没有拿到工作”不一定等于标记结束，因为其他 worker 的本地缓存、正在扫描的任务或写屏障缓冲还可能产生新工作。终止检测需要全局协调。

共享入口 [`gcDrain`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcmark.go#L1239) 先消费 root jobs；随后按固定优先级尝试本地对象、本地 span、全局对象、全局 span 和 span stealing，分别调用 `scanObject` 或 `scanSpan`，退出前刷新本地扫描工作计数。它的 flags 控制是否在抢占请求、普通调度工作出现或 fractional worker 达到目标时退出，以及是否把后台扫描 credit 刷给控制器。关闭 Green Tea 时 span queue 的方法由 `mgcmark_nogreenteagc.go` 实现为空操作，因此同一个循环自然退化为逐对象路径。

Mutator assist 使用的不是这些 flags，而是单独的 [`gcDrainN`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcmark.go#L1392)。它在 system stack 上按 `scanWork` 做近似有界的扫描，并允许因抢占、CPU limiter 或暂时取不到工作而提前返回。

默认 Green Tea 的核心优化是：对满足条件的小对象，首次发现指针时设置 span 内的 mark bit 并把 span 排队；稍后取得 span 扫描所有已积累对象，提高对象数据和元数据的局部性。inline bitmap 同时保存 `marks` 与 `scans`，通过差集确定仍需扫描的对象。大对象、oblet、根产生的部分工作以及不满足条件的小对象仍可进入普通 workbuf，所以它不是“完全不用对象队列”的另一套 GC。

### workbuf如何在worker间交换

每个 P 的 `gcWork` 通常持有本地 work buffer，减少每发现一个灰对象就操作全局锁。runtime 的 workbuf 通过无锁 `work.full/work.empty` 栈交换：[`getempty`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcwork.go#L424) 取空 buffer，[`putfull`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcwork.go#L490) 发布工作。

当其他 worker 缺工作而本地 buffer 较满时，[`handoff`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcwork.go#L509) 把一半对象移到新 buffer，并把原 buffer 放入全局 full 栈。这是一种批量 work sharing，而不是每个对象都进入单一全局队列。Green Tea 的 span queue 另有 FIFO、本地 refill 和 stealing 协议；对象 workbuf 使用 LIFO，二者的顺序选择服务于不同的局部性目标。

workbuf 自身从专用 manual span 分配，不作为普通 GC 对象参与本轮追踪。结束阶段只有确认所有 workbuf 都已回到 empty 集合，才能回收其 backing span。

## 并发正确性与终止

### 写屏障

写屏障是编译器在特定指针写入附近插入的一小段逻辑。概念上，它会记录被覆盖的旧指针和/或将要写入的新指针，使并发标记不会漏掉仍可能存活的对象。

```go
func link(a, b *Node) {
	a.next = b // 并发标记期间可能进入写屏障路径
}
```

并非所有赋值都需要屏障：非指针写入不涉及对象图；栈上的某些写入可以由栈扫描覆盖；编译器还能证明安全的场景可能省略。具体条件应以当前编译器生成结果为准。

Go 使用 Yuasa 删除屏障与 Dijkstra 插入屏障组合成混合写屏障。对一次 `*slot = ptr`，源码给出的抽象语义是：

```text
shade(*slot)                 // 保住将被覆盖的旧引用
if current stack is grey {
    shade(ptr)               // 灰栈向堆发布新引用时保住新对象
}
*slot = ptr                  // 屏障先于真正发布写
```

当前 G 的栈完成扫描后已经是黑色，插入侧 `shade(ptr)` 可以省略；删除侧仍阻止 mutator 把唯一引用从堆移到尚未被 GC 重新观察的位置。具体实现把待 shade 指针先写入每 P 的 write barrier buffer，批量 flush 以降低成本。这里的伪代码用于解释不变量，不对应业务代码可以调用的函数。

编译器先在 SSA 阶段判断写入是否需要屏障，并生成对 `runtime.writeBarrier.enabled` 的快速检查。需要慢路径时，架构汇编把指针记录到当前 P 的 write barrier buffer；缓冲耗尽再进入 flush 路径执行实际 shade。源码分析需要同时看：

- 编译器 [`cmd/compile/internal/ssa/writebarrier.go`](https://github.com/golang/go/blob/go1.26.2/src/cmd/compile/internal/ssa/writebarrier.go)：哪些写入被改写。
- [`runtime/mbarrier.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mbarrier.go)：typed memory 操作和 bulk barrier。
- [`runtime/mwbbuf.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mwbbuf.go)：每 P 缓冲及 flush。
- 架构 [`runtime/asm_amd64.s`](https://github.com/golang/go/blob/go1.26.2/src/runtime/asm_amd64.s)：写屏障调用约定。

“编译器插入写屏障”不等于每次指针写都会执行完整标记函数；关闭阶段通常只付一个分支成本，开启阶段也先走 P-local buffer。

### 为什么新对象视为已标记

并发标记开始后新分配的对象通常按“黑色”处理。否则它们刚创建就是白色，而本轮根扫描可能已经经过创建位置，容易成为漏标来源。

这不代表对象永远存活。它只是保证新对象至少存活到本轮 GC 结束；若之后不可达，下一轮仍会被回收。这是在正确性和实现成本之间做出的选择。

在 Go 1.26.2 中，新取得堆槽或 tiny backing block 的分配路径会在完成 heap bits 初始化后判断 `writeBarrier.enabled`，再调用 [`gcmarknewobject`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcmark.go#L1748)。并发标记开始时，`gcStart` 还会通过 [`gcMarkTinyAllocs`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgc.go#L921) 标记所有仍活跃的 tiny block；之后复用已有 block 的子对象不需要逐个重复标记。这比笼统说“mallocgc 最后统一涂黑”更准确：不同对象布局和专用分配函数在各自的发布点处理标记。

### Mutator Assist

mutator 是修改对象图的用户 goroutine。如果分配速度高于后台标记速度，只靠 worker 可能在堆迅速增长后仍未完成标记。

assist 将分配量换算为标记债务：

```text
goroutine 分配内存
  -> 累积 assist debt
  -> 债务超过阈值
  -> 暂停继续分配，在 system stack 上执行 gcDrainN
  -> 完成对应标记工作后恢复用户代码
```

这形成负反馈：制造更多 GC 压力的 goroutine 承担更多标记工作。`gcDrainN` 以需要偿还的扫描工作为近似预算，完成整对象扫描时可能略超出目标，也可能因抢占或暂时无工作提前返回。assist 会直接增加分配路径延迟，这正是“STW 很短但请求仍抖动”时需要同时检查 trace 的原因。

### 标记终止的ragged barrier

并发 worker 各自看到“本地无工作”仍不足以结束标记。[`gcMarkDone`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgc.go#L1015) 用 `markDoneSema` 串行化终止尝试，然后执行跨 P 的 ragged barrier：

```text
确认全局工作看似为空
  -> 取得worldsema
  -> forEachP flush写屏障buffer
  -> dispose每P本地gcWork
  -> 若产生新工作，释放worldsema并恢复并发标记
  -> 否则STW，最后再次flush并检查
  -> 仍有工作则重新启动world
  -> 真正固定点后进入mark termination
```

“ragged”表示各 P 不是在同一条机器指令同时到达屏障；`forEachP` 逐个让它们发布本地状态。写屏障直到所有可达对象确认标记完成前仍保持开启，终止 STW 中发现新灰对象也允许退回并发标记，而不是假定第一次检查必然成功。

[`gcMarkTermination`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgc.go#L1344) 完成剩余标记工作与一致性检查，计算 live heap，关闭写屏障，推进 GC cycle，并准备 sweep。并发标记和标记终止是同一固定点协议的两个阶段。

## 回收与控制

### Sweep为何既后台执行又由分配偿债

[`gcSweep`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgc.go#L2049) 将 `mheap_.sweepgen` 增加 2，重置 sweep arena 与计数。强制 GC 模式可以同步扫完；普通周期唤醒 [`bgsweep`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcsweep.go#L272) 并发处理 span。

仅靠后台 G 无法保证分配速度很快时及时扫完。[`gcPaceSweeper`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcsweep.go#L982) 根据待扫页数和距离下一触发点还可分配的字节数计算 `sweepPagesPerByte`；分配新 span 前，[`deductSweepCredit`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcsweep.go#L913) 检查按已分配字节应完成多少页，欠债就同步 `sweepone`。

所以 sweep 有三条推进路径：后台 sweeper、分配路径的 proportional sweep、分配器在取用未清扫 span 时的按需 sweep。mark assist 偿还的是标记债，sweep credit 偿还的是清扫债，二者不能混为一个 pacer 比例。

### Pacer控制什么

Pacer 需要同时解决：

1. 这一轮应该在堆多大时开始。
2. 应投入多少后台 worker。
3. 每分配多少字节需要完成多少标记工作。
4. 如何在目标堆大小前完成标记。

触发点通常早于目标堆大小。因为 GC 启动后，用户代码仍会继续分配；如果等到 goal 才开始，完成前必然超出目标。

```text
上一轮 live heap
      |
      +---- trigger ---- 并发标记开始
      |                    用户继续分配
      +---------------- goal ---- 期望此前完成
```

控制器根据上一轮存活量、扫描工作、分配速度和本轮反馈修正下一轮参数，因此不应把 trigger 理解为固定百分比公式。

Go 1.26.2 的 [`gcControllerState`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcpacer.go#L92) 还显式维护：

- `runway`：GC 开始后允许 mutator 继续分配的目标字节距离。
- `consMark`：按 CPU 时间归一化后的分配速度与扫描速度比值。
- `heapScan`、`lastHeapScan`、`lastStackScan`、`globalsScan`：估计本轮扫描工作。
- `assistWorkPerByte`、`assistBytesPerWork`：分配债务与扫描工作换算。
- `memoryLimit`：软内存限制输入。

`revise` 在堆和扫描估计变化时更新 assist 比例，`commit` 在周期边界确定下一轮目标。trigger 需要同时满足 GOGC、最小堆、sweep 距离、内存限制和 runway 等约束，不是简单的 `live * (1 + GOGC/100)`。

### GOGC与GOMEMLIMIT

`GOGC` 主要根据上一轮存活堆控制相对增长空间。值更高通常减少 GC 频率、增加内存占用；值更低则相反。

`GOMEMLIMIT` 给 runtime 提供软内存上限信号，范围不只等同于 Go heap。接近限制时 runtime 会更积极回收，但它不是操作系统意义上的硬配额；设置得低于程序稳定工作集可能导致持续 GC 和吞吐下降。

调优时应联合观察：

- live heap、heap goal 和实际内存峰值。
- GC CPU、assist 时间和暂停分布。
- 分配速率及热点调用栈。
- goroutine 延迟是否与 assist 或 STW 对齐。
- 容器限制与 `GOMEMLIMIT` 是否留有非 Go 内存余量。

减少不必要分配通常比单独调整 GC 参数更直接。

## 源码入口

| 目标 | 入口 |
| --- | --- |
| GC 启动和阶段 | [`runtime/mgc.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgc.go#L733) |
| 共享标记循环 | [`runtime/mgcmark.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcmark.go#L1239) |
| 默认Green Tea实现 | [`runtime/mgcmark_greenteagc.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcmark_greenteagc.go) |
| 关闭Green Tea的兼容实现 | [`runtime/mgcmark_nogreenteagc.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcmark_nogreenteagc.go) |
| 写屏障 | [`runtime/mbarrier.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mbarrier.go)、编译器 `wb` pass |
| Pacer | [`runtime/mgcpacer.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcpacer.go#L92) |
| 清扫 | [`runtime/mgcsweep.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcsweep.go) |
| Scavenger | [`runtime/mgcscavenge.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcscavenge.go) |

## 可复现实验

使用一个“每次操作突发分配、只保留少量对象”的 benchmark，另外记录每次操作的延迟分布。先固定负载观察 `gctrace`，再只改变 `GOGC`、`GOMEMLIMIT` 或 Green Tea 开关：STW、GC CPU、heap goal、assist 时间和请求尾延迟应作为同一组结果解释，不能只比较暂停列。

```bash
GODEBUG=gctrace=1 go test -run='^$' -bench=. -benchtime=10s ./...
GODEBUG=gcpacertrace=1 go test -run='^$' -bench=. -benchtime=3s ./...
GOEXPERIMENT=nogreenteagc go test -run='^$' -bench=. -benchmem ./...
go test -run='^$' -bench=. -trace=trace.out ./...
go tool trace trace.out
```

`gctrace` 适合观察 cycle、堆目标和 CPU 概况；`gcpacertrace` 是内部调试输出，字段可能跨版本变化；`nogreenteagc` 命令用于和默认 Green Tea 做同负载对照；trace 用于把 STW、mark assist 和 goroutine 延迟放到同一时间轴。实验必须记录 `go version`、`GOEXPERIMENT`、`GOGC`、`GOMEMLIMIT` 和容器限制，否则数据不可复现。

## 延伸阅读

- [Go语言垃圾收集器的实现原理](https://draven.co/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/)：包含并发 GC 和写屏障演进背景。
- [Go GC Guide](https://go.dev/doc/gc-guide)

## 系列导航

- [系列目录](/posts/go-runtime/)
- [上一篇：内存分配器](/posts/go-runtime-04-allocator/)
- 当前：Go Runtime（五）：GC、写屏障与Pacer
- [下一篇：Netpoll与Syscall](/posts/go-runtime-06-netpoll-syscall/)
