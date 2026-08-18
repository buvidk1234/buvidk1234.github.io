+++
date = '2026-08-18T00:15:00+08:00'
draft = false
title = 'Go Runtime（六）：Netpoll与Syscall'
tags = ['Go', 'Runtime', 'Netpoll', 'Syscall']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64**。尤其注意：Go 1.26.2 已不再使用历史上的 `_Psyscall` 状态；旧文章基于 `_Psyscall` 描述的状态机不能直接套用到当前源码。

网络等待和阻塞系统调用都会让当前 goroutine 暂时无法继续，但 runtime 的处理方式不同：

```text
非阻塞 fd + netpoll
  G 等在 runtime 中，M/P 可以继续调度

阻塞 syscall
  M 等在内核中，runtime 设法把 P 交给其他 M
```

理解这一区别，就能解释 Go 为什么能用同步风格编写高并发网络服务，以及为什么某些 syscall 仍会增加线程数量。

## 从net.Conn.Read到runtime

以 Unix 上 TCP 读取为例，调用链可简化为：

```text
net.(*TCPConn).Read
  -> netFD.Read
  -> internal/poll.(*FD).Read
       -> syscall.Read
       -> 数据就绪：直接返回
       -> EAGAIN：pollDesc.waitRead
            -> runtime_pollWait
            -> runtime.netpollblock
            -> gopark
```

socket 被设置为非阻塞。第一次 `read` 仍然是真实系统调用；只有内核返回 `EAGAIN`/`EWOULDBLOCK` 时，当前 G 才进入 netpoll 等待。

```text
G 调用 Read
  -> 非阻塞 read
  -> EAGAIN
  -> 登记读等待并 park G
  -> M/P 运行其他 G
  -> OS 报告 fd readable
  -> netpollready 将 G 变为 runnable
  -> G 再次 read
```

fd 就绪只表示再次尝试可能有进展，并不保证应用需要的全部数据已经到达，因此标准库仍需处理短读和再次等待。

## pollDesc是什么

标准库的 `internal/poll.FD` 与 runtime 的 [`pollDesc`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll.go#L75) 共同维护 fd 的等待状态。关键字段包括：

```text
fd          本次 pollDesc 生命周期对应的文件描述符
fdseq       防止 fd 复用后的陈旧事件
atomicInfo  closing、事件错误和 deadline 摘要
rg/wg       读写方向的原子等待槽
lock        保护 closing、timer 和 deadline 字段
rseq/wseq   防止旧 deadline timer 生效
rt/wt       读写 deadline timer
rd/wd       读写 deadline
```

`rg`、`wg` 不是普通 goroutine 列表。每个方向同时只允许一个等待者，并通过原子状态协调两种并发顺序：G 先准备 park，或 OS ready 事件先到达。

```text
pdNil    没有等待者，也没有待消费 ready
pdWait   G 正在提交等待
pdReady  ready 已发生，等待者尚未消费
G pointer 方向上已有实际等待的 G
```

这套握手防止“检查完未就绪、但还没真正睡下时事件到达”导致丢失唤醒。

[`netpollblock`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll.go#L548) 的 CAS 协议是关键：

```text
先尝试 pdReady -> pdNil：事件先到，不需要 park
  -> 否则 pdNil -> pdWait：声明准备等待
  -> 重新检查 closing/deadline/error
  -> gopark(netpollblockcommit)
       -> commit 把 pdWait CAS 为 G 指针
  -> 唤醒后 Swap(pdNil)，检查是否消费 pdReady
```

ready 方的 `netpollunblock` 可能看到 `pdWait`、G 指针或 `pdNil`。看到 `pdWait` 时留下 `pdReady`，让尚未完成 park 的一方自行消费；看到 G 指针时取出 G；这就是事件先到和 park 先完成两种时序能够汇合的原因。

## 打开、等待与关闭

### 打开

标准库创建网络 fd 后，通过 runtime poll API 初始化 pollDesc，并让平台后端关注这个 fd。epoll/kqueue 通常采用适合 runtime 唤醒模型的事件模式；Windows 则走 IOCP 的完成通知模型。

### 等待

等待路径先检查错误、关闭和 deadline，再尝试把当前 G 登记到读或写槽，最后通过 `gopark` 让出执行权。若 ready 在 park 前到达，状态机会让当前 G 消费 `pdReady` 而不真正睡眠。

### 关闭

关闭 fd 时必须：

- 标记 pollDesc 已关闭。
- 让现有读写等待者返回关闭错误。
- 从 OS poller 注销 fd。
- 防止旧事件和旧 deadline 误伤复用后的 fd。

Unix fd 数字会被快速复用，因此仅比较 fd 不够；序列号或等价代际信息是正确性的一部分。

## 平台后端

平台无关层定义等待语义，平台后端负责把 OS 事件转换为读写 ready：

| 平台 | 常见机制 | Runtime入口文件 |
| --- | --- | --- |
| Linux | epoll | `runtime/netpoll_epoll.go` |
| macOS/BSD | kqueue | `runtime/netpoll_kqueue.go` |
| Windows | IOCP | `runtime/netpoll_windows.go` |

后端返回的不是“直接恢复执行的线程”，而是一批已变为 runnable 的 G。它们还需要进入 Go 调度器，绑定 M/P 后才能继续执行。

## Linux epoll后端的具体协议

[`netpollinit`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll_epoll.go#L21) 创建 epoll fd 和非阻塞 eventfd，并把 eventfd 注册到 epoll。`netpollBreak` 向 eventfd 写入 1，用来打断正在 `epoll_wait` 的 M；`netpollWakeSig` 合并重复唤醒，避免每次 Timer/调度变化都累积一次 write。

[`netpollopen`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll_epoll.go#L49) 以 `EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET` 注册 fd，并把 `pollDesc` 指针与 `fdseq` tag 一起编码进 `epoll_event.data`。边缘触发要求上层 fd 使用非阻塞 IO 并持续读/写到 `EAGAIN`，否则不会因为数据仍未排空就重复通知。

[`netpoll`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll_epoll.go#L99) 把纳秒 delay 转成 epoll 毫秒 timeout，批量读取最多 128 个事件，再翻译为读写 mode：

```text
EPOLLIN/RDHUP/HUP/ERR  -> read ready
EPOLLOUT/HUP/ERR       -> write ready
eventfd EPOLLIN        -> poller被runtime显式唤醒
```

普通 fd 事件只有在 event 中的 tag 与 `pd.fdseq` 仍一致时才调用 `netpollready`。这一步挡住 fd/pollDesc 已关闭复用后仍滞留在旧 `epoll_wait` 结果中的事件。

## netpollready如何唤醒G

OS 返回 fd 事件后，runtime 根据读写标志找到 pollDesc，并对相应方向执行 ready 状态转换：

```text
OS event
  -> netpoll
  -> netpollready(pd, mode)
  -> 从 rg/wg 取出等待 G，或留下 pdReady
  -> 形成 runnable G 列表
  -> injectglist / ready
  -> 调度器选择执行
```

如果事件早于 G 完成 park，runtime 会留下 `pdReady`；如果 G 已经挂起，就取出 G。两种时序最终都不会漏通知。

## 调度器何时调用netpoll

netpoll 与调度器不是两个独立循环。常见接入点包括：

- `findRunnable` 没有本地工作时进行非阻塞检查。
- 所有 P 看起来都空闲时，某个 M 可以阻塞等待 OS 事件。
- `sysmon` 在特定条件下轮询，避免事件长期无人处理。
- timer、STW 恢复等路径唤醒或注入 runnable G。

runtime 会避免大量 M 同时阻塞在 poller 中。一般由少量线程等待 OS 事件，其余无工作线程休眠。

## Deadline为什么通常不是socket超时

`SetReadDeadline`/`SetWriteDeadline` 通常通过 runtime timer 实现：

```text
SetDeadline(t)
  -> 更新 pollDesc 的 deadline 和序列
  -> 安排 runtime timer
  -> timer 到期
  -> 标记对应方向超时
  -> 唤醒等待 G
  -> pollWait 返回 timeout
```

序列信息用于让旧 timer 失效。例如先设置 deadline，随后又更新或取消；旧 timer 即使晚到，也不能唤醒新一代等待。

## close、ready与deadline的三方竞态

`pollDesc` 用三层代次/摘要保护不同来源的旧事件：

- `fdseq` 随 pollDesc 回收到 cache 而递增，过滤 OS poller 返回的旧 fd 事件。
- `rseq/wseq` 在 deadline 更新、停止或复用时递增，过滤旧 Timer 回调。
- `atomicInfo` 汇总 closing、deadline expired、event error 和低位 fdseq，让等待方无需获取 `pd.lock` 就能复查错误。

关闭和 deadline 路径都遵循同一个发布顺序：持有 `pd.lock` 修改 `closing/rd/wd`，调用 [`publishInfo`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll.go#L153)，再 [`netpollunblock`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll.go#L591)。等待方则先把 `rg/wg` 从 `pdNil` 改为 `pdWait`，再读 `atomicInfo`。这一对相反顺序保证：若 close/deadline 已发布，新的 waiter 不会睡下；若 waiter 已登记，unblock 一定能看见它。

[`netpolldeadlineimpl`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll.go#L621) 首先比较 Timer 携带的 `seq` 与当前 `rseq/wseq`。过期回调直接丢弃；有效回调把 deadline 设为 `-1`、发布摘要、取出等待 G，解锁后再 `goready`。

close 与 IO ready 对 `rg/wg` 的目标值不同：IO ready 在无人等待时留下 `pdReady`，供未来一次 Wait 消费；close/deadline 在无人等待时保持 `pdNil`，因为未来 Wait 会从 `atomicInfo` 得到持久错误。这样不会把一次旧超时伪装成新连接的一次 IO ready。

## 普通Syscall路径

普通系统调用的调度协作可以概括为：

```text
Go code
  -> runtime.entersyscall
  -> trap/syscall instruction
  -> kernel
  -> runtime.exitsyscall
  -> Go code
```

`entersyscall`/`exitsyscall` 不是替内核执行调用，而是维护 G/M/P 状态，让调度器和 GC 知道当前线程发生了什么。以下描述针对 Go 1.26.2；Go 1.25 及更早版本使用的 `_Psyscall` 状态图属于上一代实现，不能与本节混用。

## entersyscall做什么

[`entersyscall`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L4737) 取得调用者 PC/SP/BP 后进入 `reentersyscall`：

```text
增加 m.locks，禁止抢占
  -> stackguard0 = stackPreempt，throwsplit = true
  -> 保存 m.syscalltick 和 m.oldp
  -> 保存 gp.syscallpc / syscallsp / syscallbp
  -> trace GoSysCall
  -> 若正在 STW，立即交出 P 到 _Pgcstop
  -> G: _Grunning -> _Gsyscall
  -> 必要时唤醒 sysmon
  -> 减少 m.locks
```

保存 PC/SP 不仅用于调度，也用于 GC 和 traceback。系统调用参数可能通过 `uintptr` 指向 Go 栈；内核仍在使用旧地址时移动栈会破坏指针，因此这条路径对栈增长有特殊限制。

## Go 1.26如何表示“P在syscall中”

Go 1.26.2 的 [`runtime2.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/runtime2.go#L126) 明确将 `_Psyscall_unused` 标为废弃状态。普通 `entersyscall` 把 G 改为 `_Gsyscall`，但 P 仍为 `_Prunning`，`M.p` 和 `P.m` 暂时保持连接。runtime 通过“P 关联 M，而 M 的当前 G 处于 `_Gsyscall`”识别这一情况。

这样短 syscall 返回时，M 无需重新连接 P：

```text
短 syscall
  M 很快返回
  -> 原 P 仍可恢复
  -> G 直接继续运行

长 syscall
  sysmon 从 P.m -> M.curg 观察到 _Gsyscall
  -> setBlockOnExitSyscall 获取 G 的 scan bit并复核 G/M/P 链
  -> takeP：断开 M.p 和 P.m，P -> _Pidle
  -> handoffp 给其他 M
  -> 原 M 继续阻塞在内核，m.oldp 保留历史引用
```

[`setBlockOnExitSyscall`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L6750) 与 `exitsyscall` 并发。它先观察 G/M/P，再通过 `_Gscan` bit 暂停 G 的状态转换，最后复核三者仍是同一组，才允许 `takeP`。因此 retake 不是对几个裸指针做无同步拆线。

对于 runtime 明确知道会阻塞的路径，[`entersyscallblock`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L4788) 会在切换 `_Gsyscall` 前主动 `handoffp(releasep())`。源码特意标注了短暂的“`_Grunning` 但没有 P”窗口，并用不可抢占和受控顺序保证安全。

`sysmon` 的 retake 不会打断内核调用。它夺回的是 P 的 Go 执行资格，让其他 M 使用；原来的 M 仍要等待内核返回。

## exitsyscall如何回来

[`exitsyscall`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L4883) 先乐观地把 G 改回 `_Grunning`。此时可能短暂出现 `_Grunning` 但没有 P，后续必须取得 P 或进入 `exitsyscallNoP`：

```text
exitsyscall
  -> 当前 M 仍有 P：直接走快速路径
  -> P 已被 retake：尝试 oldp，再尝试 idle P
  -> 拿到 P：G 回到 _Grunning
  -> 没有 P：mcall(exitsyscallNoP)
       -> G: _Grunning -> _Grunnable
       -> 放入全局队列，M 停止或继续调度
```

慢路径中，返回 syscall 的线程可能停下，G 以后由另一个 M/P 执行。这体现了 G 与 OS 线程默认没有固定绑定关系。

## syscalltick与sysmon

P 的 `syscalltick` 和 `sysmontick.syscallwhen` 帮助 [`retake`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L6630) 判断 syscall 是否持续未变化。它先快速筛选 `_Prunning` P，再用 `setBlockOnExitSyscall` 确认关联 G 确实处于 syscall。若本地队列为空且还有 idle/spinning P，runtime 会给短 syscall 留出时间；超过条件后才 take 并 handoff。

这一机制的目标不是让每个 syscall 都零成本，而是在快速调用的恢复成本和长阻塞时的并行能力之间折中。

## RawSyscall为什么特殊

`RawSyscall` 不走完整的 `entersyscall`/`exitsyscall` 调度通知。它适用于底层且确定很快返回的场景。如果用它执行可能长期阻塞的调用：

- M 会阻塞在内核。
- P 不能被 runtime 及时交给其他 M。
- goroutine 调度、GC 和 trace 协作可能被延迟。

业务代码通常应该使用标准库或 `x/sys` 提供的适当封装，而不是根据“少两次 runtime 调用”自行改用 `RawSyscall`。

## Netpoll与Syscall如何选择路径

```text
网络 fd 可设置非阻塞且受 poller 支持
  -> read/write 得到 EAGAIN
  -> park G，保留 M/P 处理其他工作

文件 I/O、某些设备或不可轮询调用
  -> 可能阻塞 OS 线程
  -> syscall 状态让 P 可以被其他 M 接管
```

所以 Go 并非让所有 I/O 都变成异步。它针对可轮询网络 fd 使用 netpoll，其余阻塞通过 GMP 的线程补偿机制保持程序进展。

## 源码入口

| 目标 | 入口 |
| --- | --- |
| 标准库 fd 循环 | [`internal/poll/fd_unix.go`](https://github.com/golang/go/blob/go1.26.2/src/internal/poll/fd_unix.go) |
| 标准库/runtime 桥接 | [`internal/poll/fd_poll_runtime.go`](https://github.com/golang/go/blob/go1.26.2/src/internal/poll/fd_poll_runtime.go) |
| 平台无关 netpoll | [`runtime/netpoll.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll.go#L75) |
| Linux epoll后端 | [`runtime/netpoll_epoll.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/netpoll_epoll.go) |
| 调度接入 | [`findRunnable`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L3389) 中的 netpoll 调用点 |
| Syscall状态转换 | [`entersyscall`、`exitsyscall`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L4737) |
| Syscall P抢回 | [`setBlockOnExitSyscall`、`retake`](https://github.com/golang/go/blob/go1.26.2/src/runtime/proc.go#L6630) |

## 可复现实验

```bash
go test -run TestServer -trace=trace.out ./...
go tool trace trace.out
strace -f -e trace=epoll_wait,epoll_ctl,read,write,futex ./server
```

trace 用于区分 G 的 `Network wait`、`Syscall` 和 runnable 延迟；`strace -f` 用于确认哪些 fd 进入 epoll、哪些调用真实阻塞线程。两者同时使用会增加扰动，性能结论应分开采样。文件 I/O 是否由额外机制异步化还取决于标准库、平台和具体 fd，不能从“Go 有 netpoll”推断所有 `Read` 都不阻塞线程。

## 延伸阅读

- [Go语言网络轮询器的实现原理](https://draven.co/golang/docs/part3-runtime/ch06-concurrency/golang-netpoller/)：用于理解 poller 设计；其旧版本 runtime 细节不能替代本文的 Go 1.26 状态机。
- [Go语言系统监控的实现原理](https://draven.co/golang/docs/part3-runtime/ch06-concurrency/golang-sysmon/)

## 系列导航

- [上一篇：GC、写屏障与Pacer](/posts/go-runtime-05-gc/)
- 当前：Go Runtime（六）：Netpoll与Syscall
- [下一篇：Signal与cgo线程管理](/posts/go-runtime-07-signal-cgo/)
- [原始长文](/posts/go-runtime/)
