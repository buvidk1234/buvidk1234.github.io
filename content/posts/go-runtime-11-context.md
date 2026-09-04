+++
date = '2026-09-01T00:10:00+08:00'
draft = false
title = 'Go Runtime（十一）：Context取消传播、Deadline与Cause'
tags = ['Go', 'Context', 'Concurrency']
+++

> 本文源码基线为 **Go 1.26.2**。这是系列从 runtime 核心延伸到标准库并发机制的收束篇；`context` 是标准库中的协作式取消协议，借助 Channel、锁、原子操作和 Timer 实现取消传播。源码链接固定到 `go1.26.2` tag。

`context.Context` 在请求调用链中传递三类信息：取消信号、截止时间和请求范围内的值。任务函数通过观察 `Done`、返回错误和清理资源来响应取消。

本文先建立公开契约和资源所有权，再沿最常见的标准取消树追踪 `WithCancel -> propagateCancel -> cancel`。自定义 Context 的兼容路径作为边界补充，最后再解释 Deadline、Cause、AfterFunc、WithoutCancel 和 Value。

## Context接口表达什么

[`Context`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L67) 接口由四个方法组成：

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

| 方法 | 回答的问题 | 关键语义 |
| --- | --- | --- |
| `Deadline` | 截止时间及其设置状态 | 返回时间和 `ok` 标记 |
| `Done` | 何时收到取消通知 | 返回只读 Channel；永不取消时可以为 `nil` |
| `Err` | 当前取消分类 | 返回 `Canceled`、`DeadlineExceeded` 或 `nil` |
| `Value` | 请求链上携带了什么值 | 沿父链查找最近的同 key 值 |

`Done` 通过关闭 Channel 完成一对多广播，`Err` 和 `Cause` 提供取消原因。Context 支持多个 goroutine 并发调用；`Value` 中对象的并发安全性由对象自身的同步机制保证。

典型消费方式是在可能阻塞的边界同时等待业务结果和取消信号：

```go
func send(ctx context.Context, out chan<- Result, r Result) error {
	select {
	case out <- r:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

取消采用协作式通知。函数在阻塞点检查 `ctx.Done()`，并把 ctx 继续传给支持取消的下层调用，从而及时结束计算并释放资源。

## Context是一条派生链

创建派生 Context 会包装父节点，并按需增加取消、截止时间或值：

```text
Background
  |
  +-- valueCtx(requestID)
        |
        +-- cancelCtx
              |
              +-- timerCtx(deadline)
                    |
                    +-- valueCtx(user)
```

Go 1.26.2 的主要实现可以归纳为：

| 实现 | 来源 | 增加的能力 |
| --- | --- | --- |
| `backgroundCtx` / `todoCtx` | `Background` / `TODO` | 永久有效、无 deadline、无值的根节点 |
| `cancelCtx` | `WithCancel` / `WithCancelCause` | `Done`、取消状态、子节点集合 |
| `timerCtx` | `WithDeadline` / `WithTimeout` | 在 `cancelCtx` 上增加 deadline 和 Timer |
| `valueCtx` | `WithValue` | 保存一组 key/value，其他能力委托给父节点 |
| `withoutCancelCtx` | `WithoutCancel` | 保留值链，切断取消、deadline 和 cause |

`Background` 与 `TODO` 的运行行为相同，分别表达“程序或请求的真实根”和“调用方仍在选择合适的父 Context”。两者都由 [`emptyCtx`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L179) 提供空实现。

## WithCancel建立节点与资源所有权

[`WithCancel`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L233) 创建一个 `cancelCtx`，调用 `propagateCancel` 把它连接到父节点，再返回一个闭包：

```text
WithCancel(parent)
  -> withCancel(parent)
       -> new cancelCtx
       -> propagateCancel(parent, child)
  -> cancel closure
       -> child.cancel(removeFromParent=true, Canceled, nil)
```

`removeFromParent=true` 表示主动取消子节点时同步解除 `parent.children` 对它的引用，避免父节点较长的生命周期无谓延长 child 的可达性。其他变量或 goroutine 仍可能持有 child；创建派生 Context 的一方应在任务完成时调用返回的 `CancelFunc`。

惯用写法是创建后立即安排清理：

```go
ctx, cancel := context.WithTimeout(parent, 500*time.Millisecond)
defer cancel()

return query(ctx)
```

`defer cancel()` 负责处理任务先完成的情况：

```text
任务先完成   -> cancel 停止 Timer、解除父子关系
deadline先到 -> Timer 自动取消，defer cancel 成为空操作
```

正常返回时调用 cancel，可以立即释放相关 Timer 和父子引用，并通知仍在等待取消的工作。`CancelFunc` 支持多个 goroutine 并发调用，第一次取消生效，之后保持已有状态；它发布通知后立即返回，任务完成状态由额外的等待机制确认。

## cancelCtx保存什么

[`cancelCtx`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L429) 嵌入父 Context，并维护取消节点自己的可变状态：

```go
type cancelCtx struct {
	Context

	mu       sync.Mutex
	done     atomic.Value          // chan struct{}
	children map[canceler]struct{}
	err      atomic.Value          // error
	cause    error
}
```

- `done` 在第一次调用 `Done()` 时创建，让实际读取取消信号的节点承担 Channel 分配。
- `children` 记录可以被包内直接取消的子节点。
- `err` 由第一次取消写入，之后可以用原子读取走高频快路径。
- `cause` 与 `children` 由 `mu` 保护。

[`Done`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L448) 先原子读取 Channel，首次创建时加锁并二次检查。取消早于首次 `Done()` 调用发生时，`cancel` 会把 `done` 直接设成包级共享的 `closedchan`，复用一个已经关闭的 Channel。

`Err` 先读取 `err`，读到非 nil 后再执行 `<-c.Done()`。`cancel` 会先发布 `err`，随后关闭或设置已关闭的 `done`；这个接收保证 `Err` 返回非 nil 时，调用方已经能够观察到 `Done` 关闭。

## 标准取消树如何连接

[`propagateCancel`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L473) 负责让父节点取消时继续取消 child。标准库的常见派生链包含两个主要状态，自定义 Context 走后面的兼容路径：

```text
Background / TODO / WithoutCancel
  -> Done() == nil
  -> 作为新取消树的根，向下管理后代

cancelCtx / timerCtx，外面可以包着 valueCtx
  -> 找到父链中最近的 cancelCtx
  -> child 登记进 parent.children
```

例如下面这条标准派生链：

```go
root, cancelRoot := context.WithCancel(context.Background())
defer cancelRoot()

withValue := context.WithValue(root, requestIDKey{}, "req-1")
child, cancelChild := context.WithTimeout(withValue, time.Second)
defer cancelChild()

return query(child)
```

`root` 以永久有效的 `Background` 为父节点，从这里建立一棵新的取消树；`withValue` 透传取消语义，所以 `child` 会越过它，直接进入 `root` 内部的 `cancelCtx.children`。标准派生链通过 map 建立取消传播关系，不需要为父子连接创建桥接 goroutine；但本例的 deadline 到期时，`time.AfterFunc` 仍会启动一个 G 执行取消回调。

[`parentCancelCtx`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L376) 通过包内私有 key 找到最近的 `*cancelCtx`，并验证它的 Done Channel 与 `parent.Done()` 相同。`valueCtx` 之类透传取消语义的包装器由此被越过，自定义包装器提供的 Done 语义则保持优先级。

登记 child 时还要处理父节点恰好正在取消的竞态。源码先快速检查 `parent.Done()`，随后在父节点锁内完成状态复查和 children 更新：

```text
lock(parent)
  -> parent 已取消：child 立即继承 err 和 cause
  -> parent 仍活跃：parent.children[child] = struct{}{}
unlock(parent)
```

因此 child 的状态只有两个完整结果：进入父节点的取消树，或者直接继承已经发生的取消。

### 自定义Context的兼容路径

自定义父 Context 通过两种方式接入传播：提供 `AfterFunc(func()) func() bool` 时，标准库注册一个可注销的取消回调；提供 Done Channel 时，标准库启动一个 goroutine，在 `parent.Done()` 与 `child.Done()` 之间桥接。child 提前取消会注销回调或结束桥接 goroutine。

这两条路径是自定义 Context 的兼容机制。阅读普通的 `Background`、`WithValue`、`WithCancel`、`WithTimeout` 组合时，知道它们存在即可；核心仍是 `cancelCtx.children` 组成的标准取消树。

## cancel如何向下传播

[`cancelCtx.cancel`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L546) 在第一次调用时完成以下顺序：

```text
lock(current)
  -> 已取消：直接返回
  -> 写入 err 和 cause
  -> 关闭 done，或发布共享 closedchan
  -> 遍历 children，递归 child.cancel(false, err, cause)
  -> children = nil
unlock(current)

removeFromParent == true
  -> 从 parent.children 删除自己，或停止 parent AfterFunc
```

传播时调用 `child.cancel(false, ...)`，由持锁的父节点统一遍历并清空 children。主动调用 child 的 CancelFunc 使用 `true`，在完成自身取消后解除父引用。

取消范围由调用位置决定：子节点覆盖自身及其后代，父节点覆盖整棵子树。

```text
             parent
            /      \
       child A    child B
          |
      grandchild

cancel(A)       -> A、grandchild 取消；parent、B 保持原状态
cancel(parent)  -> parent、A、B 及全部后代取消
```

取消树负责发布通知，`errgroup`、`WaitGroup` 或结果 Channel 负责确认任务完成。两套机制组合后形成“发出取消并等待退出”的完整流程。

## Err与Cause的职责

`Err` 是稳定的控制流分类，调用方可以只判断：

```go
switch ctx.Err() {
case context.Canceled:
	// 主动取消
case context.DeadlineExceeded:
	// 截止时间到达
}
```

[`Cause`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L282) 则保存业务层原因：

```go
var ErrQuotaChanged = errors.New("quota changed")

ctx, cancel := context.WithCancelCause(parent)
cancel(ErrQuotaChanged)

fmt.Println(ctx.Err())            // context canceled
fmt.Println(context.Cause(ctx))   // quota changed
```

第一次取消决定节点的 `err` 和 `cause`，后续调用保留这组结果。父、子分别取消时，谁先到达某个节点，谁就决定该节点的 cause：

```text
父先 cancel(causeA)  -> child 继承并持续保留 causeA
子先 cancel(causeB)  -> child 保留 causeB，父稍后仍可拥有自己的 causeA
```

当普通 `WithCancel` 返回的 CancelFunc 首先取消节点时，传入的 nil cause 会被替换为 `Canceled`，因此 `Cause(ctx) == ctx.Err()`。若父 Context 先以自定义 cause 取消，子节点会继承该 cause，此时二者不必相等。

`Cause` 也借助 `Value(&cancelCtxKey)` 找到最近的取消节点。`valueCtx` 会透传这个内部查询，而 `withoutCancelCtx` 明确截断它，保证 `Cause(context.WithoutCancel(parent)) == nil`。

## Deadline和Timeout如何使用Timer

[`WithDeadlineCause`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L629) 是 Deadline 系列的主要实现，`WithDeadline`、`WithTimeout` 和 `WithTimeoutCause` 都在它之上组合。

```text
检查 parent deadline
  -> parent 更早：返回 WithCancel(parent)，复用父节点计时
  -> 当前 deadline 已过：立即以 DeadlineExceeded 取消
  -> 本次 deadline 生效：创建 timerCtx，并用 time.AfterFunc 安排取消
```

[`timerCtx`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L659) 嵌入 `cancelCtx`，额外保存 `deadline` 和 `*time.Timer`。无论是手动 cancel、父节点取消还是 deadline 到达，最终都进入 `timerCtx.cancel`；它先传播取消，再停止并清空自己的 Timer。

父节点具有更早的 deadline 时，新节点通过 `WithCancel(parent)` 复用父节点的计时安排，同时保持独立取消能力，因此调用方可以比父 deadline 更早结束自己的工作。

`WithDeadlineCause(parent, d, cause)` 中的 cause 用于 deadline 自己到期。若调用方手动执行返回的 `CancelFunc` 并赢得第一次取消，`Err` 和 `Cause` 都取值 `Canceled`；若父取消或 deadline 已经先发生，后续 CancelFunc 不会覆盖已有结果。预先传入的 cause 专属于 deadline 触发路径。

```go
ctx, cancel := context.WithTimeoutCause(
	parent,
	200*time.Millisecond,
	errors.New("upstream response timeout"),
)
defer cancel()

<-ctx.Done()
log.Printf("class=%v cause=%v", ctx.Err(), context.Cause(ctx))
```

Deadline 是工作允许持续到的最晚时间边界，也是 deadline 取消的计划触发时刻；父节点取消或手动调用 CancelFunc 都可能让 Context 更早结束。Timer 执行、调度延迟和 STW 又可能让调用方在 deadline 时刻之后才观察到取消。

## AfterFunc的竞争语义

[`context.AfterFunc`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L308) 注册在 ctx 取消后执行的函数。每次注册彼此独立，f 总是在自己的 goroutine 中启动，ctx 已取消时也一样异步启动。

返回的 `stop` 与取消操作竞争：

```text
stop() == true
  -> stop 赢得竞争

stop() == false
  -> f 已开始，或者此前已经 stop
```

false 表示 f 已开始，或者此前已有一次 stop 调用。f 的完成状态通过关闭 Channel 或调用 `WaitGroup.Done` 来确认。下面写法适用于 stop 只有一个调用者的场景；多个调用者可以再增加一次“谁已成功停止”的协调：

```go
finished := make(chan struct{})
stop := context.AfterFunc(ctx, func() {
	defer close(finished)
	cleanup()
})

if !stop() {
	<-finished // 等待 cleanup 完成
}
```

内部的 `afterFuncCtx` 用一个 `sync.Once` 决定究竟由 stop 获胜，还是由取消路径启动 f。这保证两者并发时 f 最多启动一次。

## WithoutCancel建立独立生命周期

[`WithoutCancel`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L581) 继承父节点的 Value 链，并建立一条独立生命周期：

| 调用 | 返回 |
| --- | --- |
| `Deadline()` | 零值、`false` |
| `Done()` | `nil` |
| `Err()` | `nil` |
| `Cause()` | `nil` |
| `Value(key)` | 继续查询父链 |

它适合继承追踪信息并拥有独立生命周期的后续任务。任务通过新建 deadline 或其他完成信号建立结束条件，常见做法是再包一层超时：

```go
detached := context.WithoutCancel(requestCtx)
ctx, cancel := context.WithTimeout(detached, 5*time.Second)
defer cancel()

return flushAuditLog(ctx)
```

直接执行 `<-detached.Done()` 会永久阻塞，因为 Done 是 nil。放在 `select` 中时对应 case 会被禁用，这与 nil Channel 的语言语义一致。

## Value是一条线性查找链

[`WithValue`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L714) 检查 key 非 nil 且可比较，然后创建一个保存单组 key/value 的 `valueCtx`。查询从当前节点开始，匹配时返回对应值，其余情况继续向父节点移动。

```text
valueCtx(k3, v3)
  -> valueCtx(k2, v2)
       -> valueCtx(k1, v1)
            -> Background
```

因此查找成本与命中位置和链深有关。Context Value 用于 request ID、trace/span、认证主体等跨 API 边界的请求元数据；函数参数和配置结构用于可选参数及可变业务状态。

key 使用包内自定义、可比较的类型，为每个包建立独立的 key 空间：

```go
package requestid

import "context"

type key struct{}

func With(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, key{}, id)
}

func From(ctx context.Context) (string, bool) {
	id, ok := ctx.Value(key{}).(string)
	return id, ok
}
```

`Value` 返回 nil 包含“key 未命中”和“对应值为 nil”两种情况。包级类型安全存取函数可以同时封装 key，并通过第二个返回值表达命中状态。

## 使用约定

### 始终调用cancel

父 `cancelCtx` 会持有 child，`timerCtx` 还可能持有待触发的 Timer。创建后立即 `defer cancel()`，可以在正常返回时及时释放这些资源；`go vet` 可以检查 CancelFunc 在各条控制流路径上的使用情况。

### 延续调用链

库函数继续传入原 ctx 或它的派生节点，从而保留调用方的取消、deadline、trace 和请求值。新的独立生命周期从明确选择的 `context.Background()` 或 `WithoutCancel` 开始。

### 把Context作为第一个参数

Context 通常作为第一个参数显式传递，让生命周期、调用边界和静态检查保持清晰。调用方始终传入有效 Context，待确定具体来源时使用 `context.TODO()`。

### 组合取消与等待

`cancel()` 发布取消状态，`errgroup`、`WaitGroup` 或结果 Channel 确认业务 goroutine 已退出。工作函数在阻塞点观察 ctx，两类机制共同完成“取消并等待”。

### Value承载请求元数据

Context Value 承载 request ID、trace/span 和认证主体等请求元数据。分页大小、重试次数、开关等函数配置使用显式参数或配置结构，使依赖和类型边界直接呈现在 API 中。

## 可复现实验

下面的测试覆盖 cause 的先到者、向下传播、兄弟隔离和 WithoutCancel：

```go
package contextdemo

import (
	"context"
	"errors"
	"testing"
)

func TestCancellationTree(t *testing.T) {
	root, cancelRoot := context.WithCancelCause(context.Background())
	left, cancelLeft := context.WithCancelCause(root)
	right, cancelRight := context.WithCancel(root)
	detached := context.WithoutCancel(left)
	defer cancelRight()

	leftCause := errors.New("left stopped")
	cancelLeft(leftCause)

	if !errors.Is(context.Cause(left), leftCause) {
		t.Fatalf("left cause = %v", context.Cause(left))
	}
	if right.Err() != nil {
		t.Fatalf("canceling left affected sibling: %v", right.Err())
	}
	if detached.Err() != nil || context.Cause(detached) != nil {
		t.Fatalf("WithoutCancel inherited cancellation")
	}

	rootCause := errors.New("request closed")
	cancelRoot(rootCause)
	<-right.Done()

	if !errors.Is(context.Cause(right), rootCause) {
		t.Fatalf("right cause = %v", context.Cause(right))
	}
	if !errors.Is(context.Cause(left), leftCause) {
		t.Fatalf("first cancellation did not win")
	}
}
```

运行并检查取消函数：

```bash
go test -race ./...
go vet ./...
```

分析自定义 Context 时，可以对照 [`context` 包测试](https://github.com/golang/go/blob/go1.26.2/src/context/x_test.go)：标准派生链通过 children 直接传播；覆盖 Done 的自定义包装触发桥接 goroutine；实现 AfterFunc 方法的父节点支持注销回调。生产正确性由明确的同步信号验证，`runtime.NumGoroutine` 的瞬时值用于辅助观察。

## 源码阅读顺序

1. [`Context` 与 emptyCtx](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L67)：先确认公开协议和根节点。
2. [`WithCancel` 与 WithCancelCause](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L233)：查看取消节点如何创建。
3. [`cancelCtx`、Done 与 Err](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L429)：理解延迟 Channel 和并发状态。
4. [`propagateCancel`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L473)：先看标准链如何把 child 登记进 `children`，再把自定义 Context 的兼容路径作为边界知识。
5. [`cancel` 与 removeChild](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L398)：观察向下广播与引用解除。
6. [`timerCtx`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L629)：理解 Deadline 和 Timer 清理。
7. [`AfterFunc`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L308)：分析 stop 与 callback 的竞争。
8. [`value`](https://github.com/golang/go/blob/go1.26.2/src/context/context.go#L775)：查看值链和内部 cancelCtx 查询。

## 结论

Context 的取消传播是一棵由父节点管理的通知树。根节点以永久有效的父 Context 为起点，标准库中的后续可取消节点通常进入最近的 `cancelCtx.children`。自定义 Context 通过 AfterFunc 回调或桥接 goroutine 接入传播。

Context 以协作通知驱动任务退出；CancelFunc 释放传播关系和 Timer；等待机制确认任务完成；Value 承载请求元数据。`propagateCancel` 的各条分支共同保证取消可靠到达、关联及时释放，并保留自定义 Context 的语义。

## 延伸阅读

- [context包文档](https://pkg.go.dev/context)
- [Go Concurrency Patterns: Context](https://go.dev/blog/context)
- [Go Concurrency Patterns: Pipelines and cancellation](https://go.dev/blog/pipelines)
- [Go语言设计与实现：上下文Context](https://draven.co/golang/docs/part3-runtime/ch06-concurrency/golang-context/)：适合补充设计脉络；实现细节以本文固定版本源码为准。
- [context.go](https://github.com/golang/go/blob/go1.26.2/src/context/context.go)

## 系列导航

- [系列目录](/posts/go-runtime/)
- [上一篇：Timer实现](/posts/go-runtime-10-timer/)
- 当前：Go Runtime（十一）：Context取消传播、Deadline与Cause
- [回到第一篇：运行时架构与程序启动](/posts/go-runtime-01-bootstrap/)
