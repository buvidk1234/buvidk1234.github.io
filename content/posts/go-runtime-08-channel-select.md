+++
date = '2026-08-18T00:17:00+08:00'
draft = false
title = 'Go Runtime（八）：Channel与Select'
tags = ['Go', 'Runtime', 'Channel', 'Select']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64**。Channel 的语言语义跨平台一致；文中的字段、函数和行号固定到该版本，后续版本可能调整实现。

Channel 不是“带锁队列”的同义词。它同时实现值传递、goroutine 阻塞与唤醒、`close` 广播、`select` 多路竞争，并且要和栈复制、GC、race detector、Timer 配合。

理解 Channel 的关键，是区分三种正常数据路径：直接交接、环形缓冲区、等待队列。前两条可以立即完成；进入等待队列才会把当前 G 变为 `_Gwaiting`。nil Channel 的阻塞操作是没有数据路径的永久 park，属于另一个边界情况。

## 从语法到runtime入口

编译器把普通发送和接收分别降低为 `runtime.chansend1`、`runtime.chanrecv1/2`。非阻塞的单 case `select` 会被优化为 `selectnbsend` 或 `selectnbrecv`；多个有效 case 才构造 case 数组并调用 `runtime.selectgo`。

```go
ch <- x          // runtime.chansend1
x = <-ch         // runtime.chanrecv1
x, ok = <-ch     // runtime.chanrecv2
close(ch)        // runtime.closechan
```

这意味着 `select` 不是对每个 case 依次执行一次普通 channel 操作。它必须在所有相关 Channel 锁的保护下完成一次原子决策。

源码入口：[`walk/select.go`](https://github.com/golang/go/blob/go1.26.2/src/cmd/compile/internal/walk/select.go#L56) 与 [`chan.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L159)。

## hchan保存什么

[`hchan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L34) 是 Channel 的运行时对象：

```text
hchan
  qcount/dataqsiz       已缓存元素数/容量
  buf                   环形缓冲区
  elemsize/elemtype     元素布局与类型信息
  sendx/recvx           写入和读取下标
  sendq/recvq           阻塞发送者和接收者
  closed                是否关闭
  timer                 关联的runtime timer
  lock                  保护上述可变状态
```

缓冲区是 FIFO 环，但 `sendx` 和 `recvx` 不要求始终不同。缓冲区满时二者可指向同一槽位，接收方取走旧值后，等待发送者的新值可立即补入这个槽位。

源码维护两个重要不变量：通常 `sendq` 与 `recvq` 至少一个为空；对缓冲 Channel，`qcount > 0` 意味着没有等待接收者，`qcount < dataqsiz` 意味着没有等待发送者。无缓冲 Channel 在同一个 `select` 同时收发自身时存在特例。

## makechan的三种分配方式

[`makechan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L75) 先检查元素大小、对齐、容量乘法溢出和最大分配限制，再根据缓冲区布局选择分配方式：

| 条件 | 分配方式 | GC含义 |
|---|---|---|
| 容量为零或元素大小为零 | 只分配 `hchan` | `buf` 指向 race detector 使用的位置 |
| 元素不含指针 | `hchan` 与缓冲区一次连续分配 | 缓冲区无需逐元素扫描 |
| 元素含指针 | 分配 `hchan`，再按元素类型分配缓冲区 | GC 能按类型扫描指针 |

这里再次体现了分配器与 GC 的契约：是否含指针会改变对象布局，而不仅是扫描速度。

## sudog连接G与等待对象

Channel 等待队列不直接串联 `g`，而是串联 [`sudog`](https://github.com/golang/go/blob/go1.26.2/src/runtime/runtime2.go#L406)。同一个 G 在 `select` 中可以同时等待多个 Channel，因此“G 与同步对象”不是一对一关系。

`sudog` 保存等待的 G、Channel、元素地址、链表指针、是否来自 `select`、操作是否成功以及 profiling 时间。其 `elem` 可能指向等待 G 的栈，所以栈复制必须沿 `g.waiting` 修正这些指针。

```text
G --waiting--> sudog --c--> hchan A
             -> sudog --c--> hchan B
             -> sudog --c--> hchan C
```

`acquireSudog` 和 `releaseSudog` 使用缓存复用描述符，避免每次阻塞都产生普通堆分配。

## 发送的四条路径

[`chansend`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L176) 的主线是：

```text
nil channel?
  非阻塞 -> false
  阻塞   -> 永久park
       |
非阻塞且当前不可发送 -> 无锁快速返回false
       |
lock(hchan)
  closed          -> panic
  recvq有等待者   -> 直接复制给接收者并唤醒
  buffer有空位    -> 写sendx，推进环形下标
  非阻塞          -> false
  否则            -> 构造sudog，入sendq，park
```

直接发送由 [`send`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L318) 完成。值从发送方地址直接复制到接收方的 `sg.elem`，不经过缓冲区；随后先释放 Channel 锁，再 `goready` 接收方。

阻塞发送者在入队前把元素地址写进 `sudog`，把 `sudog` 挂到 `g.waiting`，设置 `parkingOnChan`，再通过 `gopark(chanparkcommit, &c.lock, ...)` 原子提交 park 并解锁。醒来后根据 `sudog.success` 判断操作成功还是因为 `close` 而失败。

## 接收的四条路径

[`chanrecv`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L524) 与发送并非完全镜像：

```text
nil channel?
  非阻塞 -> (false, false)
  阻塞   -> 永久park
       |
非阻塞且当前为空 -> 检查closed后快速返回
       |
lock(hchan)
  closed且空       -> 清零目标，返回(selected=true, received=false)
  sendq有等待者    -> 直接接收，或完成“取头补尾”
  buffer有元素     -> 读recvx、清槽位、推进下标
  非阻塞           -> (false, false)
  否则             -> 构造sudog，入recvq，park
```

对于已满的缓冲 Channel，若已有发送者等待，[`recv`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L702) 先把 `recvx` 指向的旧值交给接收方，再把等待发送者的值写回同一槽位。这样缓冲区仍然是满的，但一个接收和一个发送都完成了。

关闭后仍可读完缓冲区；只有“已关闭且缓冲区为空”才返回元素零值和 `ok == false`。因此 `closed` 与 `qcount` 必须一起判断。

## 为什么快速路径可以不加锁

非阻塞发送先观察 `closed == 0` 和 `full(c)`；非阻塞接收先观察 `empty(c)`，再用原子读检查 `closed` 并在必要时复查空状态。这些读只能证明“曾存在一个操作无法立即完成的时刻”，不能承诺下一条指令看到相同状态。

所以 `len(ch)`、`cap(ch)` 和非阻塞探测只能用于即时决策，不能当成随后操作仍安全的同步证明。内存顺序论证直接写在 [`chansend`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L197) 和 [`chanrecv`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L548) 的注释中。

## park与栈复制的交界

Channel 锁保护 `hchan` 字段以及队列中部分 `sudog` 字段，但源码特别禁止持有该锁时直接改变另一个 G 的状态。原因是栈缩小可能持有 G 相关状态并尝试获取 Channel 锁；反向持锁再 `goready` 会形成死锁。

因此直接交接和 `close` 都采用相同纪律：

```text
持有hchan.lock修改队列和sudog
  -> 释放hchan.lock
  -> goready目标G
```

`parkingOnChan` 与 `activeStackChans` 则关闭“已经转入 waiting，但栈指针尚未处于可安全修正状态”的窗口。这正是栈管理篇中 `copystack` 必须理解 Channel wait list 的原因。

## close是一次批量唤醒

[`closechan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L414) 在锁内完成三件事：设置 `closed`；清零所有等待接收者的目标并标记失败；把所有等待发送者标记为失败。它将待唤醒 G 收集到临时 `gList`，解锁后统一 `goready`。

接收者醒来后得到零值与 `false`；发送者醒来后发现 `success == false`，触发 `send on closed channel`。`close(nil)` 和重复关闭直接 panic。

这也解释了为什么关闭是广播，而向 Channel 发送一个零值不是：零值仍是一条普通数据，关闭则改变对象的永久状态并唤醒全部两类等待者。

## selectgo为什么需要两个顺序

[`selectgo`](https://github.com/golang/go/blob/go1.26.2/src/runtime/select.go#L122) 为有效 case 构造两个顺序：

- `pollorder` 使用 `cheaprandn` 打乱，用来探测就绪 case，避免总偏向源码中靠前的分支。
- `lockorder` 按 Channel 地址排序，用来按一致顺序获取所有 Channel 锁，避免两个 `select` 反向加锁死锁。

相同 Channel 上的多个 case 继承随机顺序；无 Channel 的 case 不进入这两个数组。`default` 不作为普通 `scase` 入队，而通过 `block` 参数决定第一轮无就绪 case 时是立即返回还是阻塞。

“随机”只用于候选 case 的选择公平性；“确定”用于并发加锁安全性，两者不能混为一谈。

## select的三轮协议

持有所有相关 Channel 锁后，`selectgo` 执行三轮协议：

1. 按 `pollorder` 查找已经可完成的收发、缓冲操作或关闭状态。找到后立即执行一个 case。
2. 若没有 case 就绪且没有 `default`，为每个 case 获取一个 `sudog`，按 `lockorder` 加入对应 `sendq/recvq`，然后 `gopark`。
3. 被某个 case 唤醒后重新锁住全部 Channel，识别获胜 `sudog`，从其他队列删除失败项，释放全部 `sudog`。

```text
random poll -> no ready case
     -> enqueue one sudog per case
     -> park once
     -> one case wins
     -> remove all losing sudogs
```

等待队列的 [`dequeue`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L886) 对 `select` 节点执行 `g.selectDone.CompareAndSwap(0, 1)`。CAS 只有一个赢家；其他 Channel 即使同时就绪，也会跳过已经输掉竞争的 `sudog`。这解决的是“多个唤醒者竞争同一个 G”，不是语言层面的 case 随机化。

## nil Channel为何有用

对 nil Channel 的阻塞收发会永久 park，但 `selectgo` 会把 Channel 为 nil 的 case 从 poll 和 lock 顺序中删掉。因此把某个 Channel 变量设为 nil，可以动态禁用对应 case：

```go
for in != nil || out != nil {
    select {
    case v, ok := <-in:
        if !ok {
            in = nil
            continue
        }
        _ = v
    case out <- 1:
        out = nil
    }
}
```

若所有 case 都被禁用且没有 `default`，整个 `select` 永久阻塞。这不是特殊语法规则，而是“没有可入队的有效 Channel + block=true”的自然结果。

## Channel与Timer的连接

Go 1.26 的 `hchan.timer` 允许 Timer Channel 参与 Channel 快速路径和 `select`。读取或构造 `pollorder` 时可能调用 [`maybeRunChan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/time.go#L1442)；阻塞/退出 `select` 时调用 `blockTimerChan`、`unblockTimerChan` 维护真正等待该 Timer Channel 的 G 数量。

这套按需挂入 Timer heap 的机制是 Go 1.23 同步 Timer Channel 语义的一部分，详细流程放在第十篇。

## 内存模型与公平性的边界

Channel 的同步保证来自语言内存模型：一次发送 synchronized before 对应接收完成；关闭 synchronized before 某次接收因为关闭而返回零值。对容量为 C 的缓冲 Channel，第 k 次接收 synchronized before 第 k+C 次发送完成；当 C=0 时，还额外有接收发生在对应发送完成之前的保证。

这里刻意使用规范中的 synchronized before：它与 happens-before 一起建立内存可见性，不代表发送方和接收方按墙上时间立即切换，也不规定调度队列顺序。

这些保证不等于：

- goroutine 被唤醒后立即获得 P。
- 多个发送者严格按墙上时间 FIFO 执行。
- `select` 在有限次数内必然选择每个持续就绪 case。
- `len(ch)` 可以替代同步。

运行时队列、随机轮询和调度器共同提供工程上的公平性，但语言规范没有承诺严格实时调度。

## 源码阅读顺序

1. [`hchan` 与 `makechan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L34)
2. [`chansend` 与 `send`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L176)
3. [`chanrecv` 与 `recv`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L524)
4. [`closechan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L414)
5. [`waitq.enqueue/dequeue`](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go#L872)
6. [`selectgo`](https://github.com/golang/go/blob/go1.26.2/src/runtime/select.go#L122)
7. [`walkSelectCases`](https://github.com/golang/go/blob/go1.26.2/src/cmd/compile/internal/walk/select.go#L56)

## 观测与验证

阻塞 profile 可以定位 Channel 等待时间：

```bash
go test -run '^$' -bench . -blockprofile=block.out ./...
go tool pprof -http=:0 block.out
go test -race ./...
go test -trace=trace.out ./...
go tool trace trace.out
```

实验应分别覆盖无缓冲直接交接、缓冲未满、缓冲已满、关闭后排空、多个就绪 case 和 nil case。基准结果只能说明特定负载下的争用与调度成本，不能反推规范没有承诺的公平性。

## 延伸阅读

- [Go Memory Model](https://go.dev/ref/mem)
- [Go语言设计与实现：Channel](https://draven.co/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)：适合补充演进背景；当前字段与流程以本文固定版本源码为准。
- [runtime/chan.go](https://github.com/golang/go/blob/go1.26.2/src/runtime/chan.go)
- [runtime/select.go](https://github.com/golang/go/blob/go1.26.2/src/runtime/select.go)

## 系列导航

- [上一篇：Signal与cgo线程管理](/posts/go-runtime-07-signal-cgo/)
- 当前：Go Runtime（八）：Channel与Select
- [下一篇：Runtime信号量与同步原语](/posts/go-runtime-09-sync-primitives/)
- [原始长文](/posts/go-runtime/)
