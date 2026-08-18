+++
date = '2026-08-18T00:18:00+08:00'
draft = false
title = 'Go Runtime（九）：Runtime信号量与同步原语'
tags = ['Go', 'Runtime', 'sync', 'Mutex']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64**。`sync` 的公开语义由标准库保证；本文讨论的状态位、队列和 runtime 接口属于版本相关实现，不应被业务代码依赖。

`sync.Mutex` 的竞争慢路径、`RWMutex` 的读写协调和 `WaitGroup` 的等待，最终都需要把 G 停下来并在条件满足时重新放回调度器。它们共享 runtime semaphore，但并不共享完全相同的高层状态机。

先建立边界：runtime semaphore 是“按地址等待与唤醒”的底层设施，不是公开的计数信号量类型。存放在 `uint32` 地址中的 token 只服务于防丢唤醒和交接协议。

## 从sync包到runtime

标准库通过无函数体声明和 `go:linkname` 连接 runtime：

```text
internal/sync
  Mutex slow path
      -> runtime_SemacquireMutex
      -> runtime_Semrelease

sync
  RWMutex  -> readerSem / writerSem
  WaitGroup -> sema
  Cond      -> notifyList
```

公开类型保留自己的原子状态；runtime 只负责等待队列、park/ready 和 contention profile。把所有语义都归因于 semaphore，会看不懂 Mutex 的 starvation、RWMutex 的 writer preference 和 WaitGroup 的 misuse 检测。

## semaRoot为何按地址组织

[`semaRoot`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go#L40) 包含锁、treap 根和原子等待者计数。全局 `semtable` 按 semaphore 地址哈希分片，减少无关同步对象之间的争用。

一个 root 可能承载许多不同地址的等待者：

```text
semtable[hash(addr)]
  semaRoot
    treap: key = semaphore address
      addr A -> sudog -> same-address waiters
      addr B -> sudog -> same-address waiters
      addr C -> sudog -> same-address waiters
```

treap 按地址维持二叉搜索树顺序，按随机 `ticket` 维持堆优先级，期望查询复杂度为 O(log n)。每个不同地址只占一个树节点；同地址的其他 `sudog` 通过 `waitlink/waittail` 串联。因此唤醒无需扫描整个分片。

[`queue`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go#L304) 支持 FIFO 或 LIFO 插入。LIFO 会用新 `sudog` 替换 treap 中的同地址头节点，同时保留旧头等待时间，供 Mutex starvation 与 profile 统计使用。

## semacquire如何避免丢失唤醒

[`semacquire1`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go#L146) 先用 CAS 尝试把 `*addr` 从正数减一。失败后不能直接入队，否则 release 可能恰好发生在“检查失败”和“声明正在等待”之间。

慢路径顺序是：

```text
try acquire token
  -> lock semaRoot
  -> nwait++
  -> retry acquire token
       成功: nwait--, unlock, return
       失败: queue sudog, goparkunlock
  -> 被唤醒后取得handoff ticket或再次竞争token
```

`nwait++` 先于第二次尝试，使之后的 release 必然看见潜在等待者；第二次尝试则覆盖 release 已经增加 token、但尚未观察到等待者的情况。这是一个经典的防丢唤醒握手。

等待描述符仍是 `sudog`，但它的 `elem` 记录 semaphore 地址。`goparkunlock` 在提交 G 的等待状态时释放 root 锁，唤醒端不会看到“已经入队但还未真正可 park”的不一致窗口。

## semrelease为何先增加token

[`semrelease1`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go#L207) 先原子增加 `*addr`，再检查分片 `nwait`。若没有等待者即可返回；否则锁 root，按地址 [`dequeue`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go#L405)，解锁后 `readyWithTime`。

顺序不能反过来。先看 `nwait == 0` 再增加 token，可能与 acquire 的初次失败交错，让双方都错过对方。release 先发布 token，与 acquire 的“先登记、再尝试”共同闭合竞态窗口。

handoff 模式下，唤醒端可把特殊 ticket 交给等待者，使它不必和新到达的 goroutine 再抢 token；必要时当前 G `goyield`，让被唤醒者获得执行机会。这是 Mutex 饥饿模式的底座，不代表普通 release 都立即切换 G。

## Mutex状态字

Go 1.26 的 [`internal/sync.Mutex`](https://github.com/golang/go/blob/go1.26.2/src/internal/sync/mutex.go#L20) 包含：

```text
state int32
  bit 0: mutexLocked
  bit 1: mutexWoken
  bit 2: mutexStarving
  bit 3..: waiter count

sema uint32
```

无竞争 `Lock` 只需 CAS `0 -> mutexLocked`，无竞争 `Unlock` 只需原子减去 locked 位。这两个可内联的路径不进入 runtime。

[`lockSlow`](https://github.com/golang/go/blob/go1.26.2/src/internal/sync/mutex.go#L95) 自旋、更新状态字并调用 `runtime_SemacquireMutex`。是否自旋由 `runtime_canSpin` 与 `runtime_doSpin` 决定，受 CPU 数、P 数、队列和累计自旋次数限制；它不是固定忙等。

## 正常模式与饥饿模式

正常模式中，被唤醒 waiter 与新到达 goroutine 竞争锁。新到达者已经运行在 CPU 上，常常更容易取胜，吞吐较高；反复失败的 waiter 在等待超过阈值后请求进入 starvation 模式。

Go 1.26.2 的固定阈值 `starvationThresholdNs` 为 1 ms。这是当前实现的调度启发式，不是 `sync.Mutex` API 对最大等待时间的承诺。

饥饿模式中，Unlock 使用 semaphore handoff 把所有权交给队首 waiter，新到达者不争锁而加入队尾。获得锁的 waiter 在自己是最后一个等待者，或等待时间较短时退出 starvation。

这是吞吐与尾延迟的动态折中。`Mutex` 没有 goroutine owner 字段，所以允许一个 G 加锁、另一个 G 解锁；错误解锁通过状态机检测，而不是线程身份检查。

[`unlockSlow`](https://github.com/golang/go/blob/go1.26.2/src/internal/sync/mutex.go#L202) 在正常模式设置 `mutexWoken` 并唤醒一个 waiter；在饥饿模式调用 handoff release。`mutexWoken` 避免多个 Unlock 无意义地同时唤醒多个竞争者。

## RWMutex的两个semaphore

[`RWMutex`](https://github.com/golang/go/blob/go1.26.2/src/sync/rwmutex.go#L39) 包含一个串行化 writer 的 `Mutex`、`writerSem`、`readerSem`、`readerCount` 和 `readerWait`。

读锁快路径对 `readerCount` 加一；若结果为负，说明 writer 已宣布到来，reader 在 `readerSem` 等待。writer 先获取 `rw.w`，再从 `readerCount` 减去 `rwmutexMaxReaders`，由减法前的值知道已有多少 active reader，并在 `writerSem` 等到最后一个离开。

```text
writer到来
  -> rw.w.Lock，排斥其他writer
  -> readerCount -= MaxReaders，阻止新reader穿过
  -> 等待已有readerCount归零
  -> 临界区
  -> readerCount += MaxReaders
  -> 逐个释放被阻塞reader
  -> rw.w.Unlock
```

最后一个离开的 reader 通过 [`rUnlockSlow`](https://github.com/golang/go/blob/go1.26.2/src/sync/rwmutex.go#L129) 释放 `writerSem`。这种设计偏向已经等待的 writer，因此递归 `RLock` 可能在 writer 到来后自锁，API 明确禁止依赖递归读锁。

## WaitGroup的一个原子状态

Go 1.26 的 [`WaitGroup`](https://github.com/golang/go/blob/go1.26.2/src/sync/waitgroup.go#L48) 用一个 `atomic.Uint64` 打包 task counter、synctest 标志和 waiter count，另有 `sema` 供真正阻塞：

```text
high 32 bits          task counter
bit 31 of low half    synctest bubble flag
remaining low 31      waiter count
```

[`Add`](https://github.com/golang/go/blob/go1.26.2/src/sync/waitgroup.go#L77) 原子修改高位。counter 变成零且 waiter 非零时，它先把状态重置为零，再按 waiter 数调用 `Semrelease`。counter 变负会 panic；从零开始的正 `Add` 与 `Wait` 并发，以及上一轮 Wait 尚未返回就复用，也会被一致性检查尽量捕获。

[`Wait`](https://github.com/golang/go/blob/go1.26.2/src/sync/waitgroup.go#L160) 若 counter 已为零直接返回，否则 CAS 增加 waiter count，再在 `sema` 上等待。醒来后状态必须已经为零，否则说明 WaitGroup 在前一轮结束前被复用。

Go 1.25 加入、Go 1.26 保留的 [`WaitGroup.Go`](https://github.com/golang/go/blob/go1.26.2/src/sync/waitgroup.go#L236) 仍基于 `Add(1)`、新 goroutine 和完成时的 `Done()`，没有创造另一套 runtime 等待机制。但当前实现会在 `f` panic 时重新 panic，且刻意不先调用 `Done`，避免 `Wait` 被释放后主 goroutine 抢先正常退出；这就是文档明确要求“`f` must not panic”的原因。`runtime.Goexit` 不属于 panic，defer 路径仍会调用 `Done`。

## Cond使用ticket通知队列

[`Cond.Wait`](https://github.com/golang/go/blob/go1.26.2/src/sync/cond.go#L67) 的顺序是：从 `notifyList` 领取递增 ticket，解开用户 Locker，等待通知，再重新加锁。先领 ticket 再解锁，避免 Signal 落入“条件已改变但 waiter 尚未登记”的窗口。

runtime 的 [`notifyListAdd`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go#L578) 原子递增 `wait`；`notifyListWait` 在锁下判断 ticket 是否已经被通知，未通知才挂入 `sudog` 链表。`NotifyOne` 推进 `notify` 序号并寻找对应 ticket，`NotifyAll` 将 `notify` 直接推进到当前 `wait` 并唤醒整条链。

`wait` 和 `notify` 都允许 `uint32` 回绕；比较通过 `int32(a-b)` 完成，只要未展开的差值小于 `2^31` 就保持正确。这个条件意味着同一个 Cond 同时阻塞超过约 21 亿个 waiter 才会越界，当前 runtime 的 goroutine 数量限制使其不可达。

`notifyList` 不通过 `semaRoot` 查地址，而是自己持有 ticket 与 waiter 链表。Signal 也不保证被唤醒的 G 先于争抢 `c.L` 的其他 G 运行，所以条件必须始终放在循环中复查。

## Once为何不能只用一次CAS

[`Once.Do`](https://github.com/golang/go/blob/go1.26.2/src/sync/once.go#L52) 快路径读取 `done`，慢路径持有 Mutex，并在 `f` 返回后才发布 `done=true`：

```go
o.m.Lock()
defer o.m.Unlock()
if !o.done.Load() {
    defer o.done.Store(true)
    f()
}
```

若一开始 CAS 把 done 设为 true，其他调用会在 `f` 尚未完成时返回，破坏“所有 `Do` 返回都观察到初始化完成”的保证。`Store` 使用 defer，因此 `f` panic 时也视为已经执行过，后续 `Do` 不会重试。递归调用同一个 Once 会等待自己持有的 Mutex，因而死锁。

## 原子操作、semaphore与调度器各管一层

```text
高层状态机
  Mutex/RWMutex/WaitGroup/Cond/Once
        |
原子快路径与竞争判定
        |
runtime sema / notifyList
        |
sudog + goparkunlock / goready
        |
GMP scheduler
```

原子操作避免无竞争路径进入内核或调度器；runtime 队列解决竞争时的睡眠与唤醒；调度器决定被 ready 的 G 何时真正运行。Linux futex 不是这些普通路径的直接一一对应物：runtime 自己调度 G，只有 OS 线程级 runtime 锁和休眠等更底层场景才需要平台同步原语。

## 内存模型不等于实现细节

`Mutex.Unlock` 与之后的 `Lock`、`Once` 中 `f` 的完成与所有 `Do` 返回、`WaitGroup.Done` 与它解除的 `Wait` 都建立文档规定的 happens-before 关系。正确性应依赖这些公开保证，而不是当前源码中的 CAS 顺序、starvation 阈值或 treap 形状。

同理，`TryLock` 失败不建立同步关系，`RWMutex` 不支持升级/降级，复制首次使用后的同步对象属于错误用法。源码分析应解释契约如何实现，而不是把私有机制升级为新契约。

## 源码阅读顺序

1. [`semacquire1` 与 `semrelease1`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go#L146)
2. [`semaRoot.queue/dequeue`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go#L304)
3. [`internal/sync.Mutex`](https://github.com/golang/go/blob/go1.26.2/src/internal/sync/mutex.go#L20)
4. [`sync.RWMutex`](https://github.com/golang/go/blob/go1.26.2/src/sync/rwmutex.go#L39)
5. [`sync.WaitGroup`](https://github.com/golang/go/blob/go1.26.2/src/sync/waitgroup.go#L48)
6. [`notifyList`](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go#L549)
7. [`sync.Once`](https://github.com/golang/go/blob/go1.26.2/src/sync/once.go#L20)

## 观测与验证

```bash
go test -race ./...
go test -run '^$' -bench . -mutexprofile=mutex.out -blockprofile=block.out ./...
go tool pprof -http=:0 mutex.out
go tool pprof -http=:0 block.out
go test -trace=trace.out ./...
go tool trace trace.out
```

Mutex profile 归因的是 contention delay，不只是某次 `Lock` 自身的墙上时间；block profile 则覆盖 Channel、Cond、WaitGroup 等阻塞点。采样率和生产开销应通过 `runtime.SetMutexProfileFraction`、`runtime.SetBlockProfileRate` 明确控制。

## 延伸阅读

- [Go Memory Model](https://go.dev/ref/mem)
- [Go语言设计与实现：同步原语](https://draven.co/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/)：适合补充设计脉络；具体状态位以本文固定版本源码为准。
- [runtime/sema.go](https://github.com/golang/go/blob/go1.26.2/src/runtime/sema.go)
- [sync包文档](https://pkg.go.dev/sync)

## 系列导航

- [上一篇：Channel与Select](/posts/go-runtime-08-channel-select/)
- 当前：Go Runtime（九）：Runtime信号量与同步原语
- [下一篇：Timer实现](/posts/go-runtime-10-timer/)
- [原始长文](/posts/go-runtime/)
