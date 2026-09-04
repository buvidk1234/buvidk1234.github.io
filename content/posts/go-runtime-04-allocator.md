+++
date = '2026-08-18T00:13:00+08:00'
draft = false
title = 'Go Runtime（四）：内存分配器'
tags = ['Go', 'Runtime', 'Memory']
+++

> 本文源码基线为 **Go 1.26.2、Linux/amd64、默认 GOEXPERIMENT**。对象大小阈值、size class 表和局部快路径可能随版本变化。

本文从两个现象出发：源码里的 `new(T)` 可能产生零次堆分配，对象失去引用后 RSS 也可能维持原值。第一个现象把我们带入 `mallocgc`、mcache、mcentral 和 mheap；第二个现象继续追踪 sweep、页分配器与 scavenger。

Go 内存分配器要同时满足三类目标：小对象分配足够快，多 P 并发时减少共享锁竞争，GC 又能由任意堆地址快速定位对象边界和标记元数据。它采用按对象规格分级、按 P 缓存、按页统一管理的结构。

把这些现象对应到当前源码前，必须先分清实验开关：源码中存在字段或函数，不代表默认构建会执行它。

| 实验（`GOEXPERIMENT`值） | Go 1.26.2默认值 | 本文中的影响 |
| --- | --- | --- |
| `GreenTeaGC`（`greenteagc`） | 启用 | 部分小对象 span 使用页内 mark/scan bits |
| `SizeSpecializedMalloc`（`sizespecializedmalloc`） | 禁用 | 默认仍从通用 `mallocgc` 分派 |
| `RuntimeFreegc`（`runtimefreegc`） | 禁用 | `mcache.reusableNoscan` 默认不参与分配 |

默认值以 [`internal/buildcfg` 的实验 baseline](https://github.com/golang/go/blob/go1.26.2/src/internal/buildcfg/exp.go#L80) 为准；后文提到关闭的实验路径时会再次标明。

## 先确认对象是否真的进入堆

`new(T)` 不等于堆分配，返回局部变量地址也不必机械地等于堆分配；编译器根据值是否可能在当前栈帧结束后继续可达、地址是否暴露给未知调用等条件做逃逸分析，随后内联和标量替换还可能消除分配。

```go
package alloc

type Node struct {
	next *Node
	val  int
}

//go:noinline
func makeNode(v int) *Node { return &Node{val: v} }
```

```bash
go test -gcflags='all=-m=2' ./...
go test -run='^$' -bench=. -benchmem ./...
```

第一条解释编译器决定，第二条测量最终二进制的每次操作分配。只看源码语法或只看逃逸日志都不足以判断生产路径成本。

## 分配器的层级

Go 将小对象归入 size class，再按 spanClass 管理 span。spanClass 同时编码 size class 和对象是否为 noscan。

```text
P.mcache
  每个 P 的本地 span 缓存，常见小对象分配快路径
       |
       v
mcentral[spanClass]
  同一规格的共享 span 集合，负责给 mcache 补货
       |
       v
mheap
  全局页级资源、arena 元数据和大对象分配
       |
       v
OS virtual memory
```

这里缓存的是具有空槽位的 span，而不是为每种对象单独维护一个普通对象列表。

## page、span与object

```text
heapArena
  +-- page page page page ...
          +------ mspan ------+
          | obj | obj | obj   |
          +-------------------+
```

- page 是页分配器管理堆空间的基本粒度。
- mspan 描述一段连续页，以及其中对象的规格、位图和清扫状态。
- 小对象 span 被切成固定大小的槽位。
- 大对象通常独占一个或多个 span。

固定规格降低了查找成本和外部碎片，但会产生向上取整的内部碎片。例如 Go 1.26.2 的 size class 表中 24 字节本身就是一个规格，而请求 25 字节会进入 32 字节规格；不能脱离当前版本的表猜测具体槽位大小。

## mallocgc主流程

默认构建中的堆分配最终进入 [`mallocgc`](https://github.com/golang/go/blob/go1.26.2/src/runtime/malloc.go#L1119)，再按大小和指针布局分派。Go 1.26.2 源码还包含 `SizeSpecializedMalloc` 实验路径，但默认 baseline 没有启用它；只有显式设置 `GOEXPERIMENT=sizespecializedmalloc` 且平台、sanitizer 等条件允许时，编译器选择的特化入口才会生效：

```text
size == 0 -> &zerobase
  |
  +-- size-specialized malloc 已启用且类型布局允许
  |     -> 按 size 和 scan/noscan 进入生成的专用函数表
  |
  +-- 通用路径
        -> 并发标记时 deductAssistCredit(size)
        -> small noscan
             -> tiny（小于 maxTinySize 且上下文允许）
             -> mallocgcSmallNoscan
        -> small scan
             -> heap bits in span：mallocgcSmallScanNoHeader
             -> 否则：mallocgcSmallScanHeader
        -> large：mallocgcLarge
        -> 更新 sanitizer、profile、GC assist 和统计
```

`sizeSpecializedMallocEnabled` 同时受 GOEXPERIMENT、race/asan/msan/Valgrind 和平台条件约束。特别是默认 Go 1.26.2 构建中它为 false，不能看到源码中的函数表就断言生产程序已经使用专用入口。无论分派入口如何，常见小对象最终仍依赖当前 P 的 mcache，避免每次分配争用全局堆锁。

## mcache与mcentral

每个 P 持有一个 mcache，因此 M 绑定 P 后可以访问对应的本地缓存。某个 spanClass 用满后，mcache 把旧 span 归还给 mcentral，并获取一个仍有空闲槽位的 span。

mcentral 按 spanClass 分组。多个 P 可能同时补货，因此这条路径需要共享同步；但补货频率远低于每次对象分配，同步成本被整批 span 的后续分配摊薄。现代实现使用并发 `spanSet` 和必要的内部锁，不能再把 mcentral 描述成由一把粗粒度中心锁保护的普通链表。

Go 1.26.2 的 [`mcache`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mcache.go#L20) 关键字段包括：

```go
type mcache struct {
	nextSample  int64
	memProfRate int
	scanAlloc   uintptr

	tiny       uintptr
	tinyoffset uintptr
	tinyAllocs uintptr

	alloc          [numSpanClasses]*mspan
	reusableNoscan [numSpanClasses]gclinkptr
	stackcache     [_NumStackOrders]stackfreelist
	flushGen       atomic.Uint32
}
```

`alloc` 是按 spanClass 索引的当前 span。结构体中虽然存在 `reusableNoscan`，但 `addReusableNoscan`、`hasReusableNoscan` 和取对象路径都会先检查 `runtimeFreegcEnabled`；默认构建中该实验关闭，所以这个对象复用列表不参与分配。只有显式启用 `GOEXPERIMENT=runtimefreegc` 时，它才成为区别于 allocation bitmap 新槽位查找的另一条路径。`flushGen` 用来判断跨 GC 周期后缓存是否过期，需要在重新获得 P 时刷新。

[`mcentral`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mcentral.go#L22) 不再适合画成一个简单链表：

```go
type mcentral struct {
	spanclass spanClass
	partial  [2]spanSet
	full     [2]spanSet
}
```

两组 `partial/full` 随 `sweepgen` 交换“已清扫”和“未清扫”角色。`cacheSpan` 先扣除 proportional sweep credit，然后优先取已有空槽的 `partialSwept`；没有时再依次尝试并清扫 `partialUnswept` 与 `fullUnswept`，最后才向 mheap 申请新 span。未清扫路径有固定的 `spanBudget`，避免一次补货为了寻找空槽而清扫过多 span。

## mheap与heapArena

`mheap` 负责跨规格的全局页资源、span 元数据、arena 映射和向 OS 申请或归还虚拟内存。大对象绕过 mcache/mcentral，直接进入页级分配路径。

堆地址空间按 heapArena 分区。runtime 可以由对象地址定位对应 arena，再通过页到 span 的映射找到 mspan：

```text
object address
  -> arena index
  -> heapArena
  -> page index
  -> spans[page]
  -> mspan
  -> object slot
```

这条反查链路用于 GC、对象边界检查和堆诊断，不需要遍历整个堆。

[`mheap`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mheap.go#L64) 的 `pages pageAlloc` 管理页占用，`arenas` 保存地址空间到 `heapArena` 元数据的映射，`allspans` 保存所有曾创建的 span 描述符。`mheap.lock` 必须在 system stack 上获取，否则当前 G 在持锁期间增长栈，栈分配再次进入 mheap 会自锁。

页分配器不是按页线性扫描整个堆。[`pageAlloc.alloc`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mpagealloc.go#L879) 使用分层摘要定位足够长的空闲页区间；每个 P 的 `pageCache` 又缓存一小段页位图，降低单页 span 分配访问全局 pageAlloc 的频率。

## 分配与清扫如何交接span

`allocBits` 记录 span 中哪些槽位已经分配，是分配器寻找和发布对象的依据；GC 的 mark bits 则记录本轮仍然存活的对象。清扫必须在独占该 span 的前提下比较两者，回收未标记槽位，再把存活结果发布成下一轮分配状态。

标记位的暂存位置取决于 GC 实现。Go 1.26.2 默认启用 Green Tea GC，部分小对象 span 在页内 `spanInlineMarkBits` 积累 mark/scan 状态，其他对象使用 `gcmarkBits`。清扫前半段仍通过统一的 mark-bit 访问器处理对象；在最终统计存活对象和轮换位图之前，才把需要的 inline marks 合并到 `gcmarkBits`。随后 `gcmarkBits` 成为新的 `allocBits`，runtime 再准备一组清零的标记位供下一轮 GC 使用。

分配器关心的是这次所有权和位图交接；Green Tea 的 mark/scan 队列、并发标记过程及存活对象如何到达这些位图，放在[下一篇：GC、写屏障与Pacer](/posts/go-runtime-05-gc/)中展开。

`sweepgen` 用代际计数表示 span 与当前清扫周期的关系，帮助多个 goroutine 并发或按需清扫时判断一个 span 是否已经处理，避免只用简单布尔值造成周期混淆。

在 [`mspan`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mheap.go#L422) 中，几个字段共同定义槽位状态：

```text
freeindex / allocCache   下一次快速查找空闲槽位的位置和缓存
nelems / elemsize       span 内对象数量与槽位大小
allocBits               哪些槽位已分配
gcmarkBits/inline marks 本轮 GC 标记到哪些对象
allocCount              当前已分配数量
spanclass               size class + scan/noscan
sweepgen                 与 mheap_.sweepgen 的相对关系
```

`freeIndexForScan` 与 allocator 使用的 `freeindex` 分开，保证扫描器只在对象内容和 heap bits 初始化完成后才把对象视为可扫描。这是分配发布顺序的一部分，不是冗余计数。

## Tiny allocator

tiny allocator 会合并某些很小且不含指针的对象，以降低元数据和碎片成本。它不适用于含指针对象，因为 GC 必须准确识别和扫描指针；对象身份、对齐和生命周期条件也会限制是否能够进入 tiny 路径。

多个 tiny 子对象共享一个底层块，所以只要其中任意子对象仍可达，整个块都会存活。这节省分配元数据，但可能延长同块其他对象占用内存的时间。优化小对象时不能只比较 allocation count，还要观察 live heap。

## 分配如何推动GC

分配并非 GC 之外的纯消费动作：

- 分配量接近触发线时，分配路径可能启动新一轮 GC。
- 并发标记期间，新分配对象需要满足标记不变量。
- 分配过快的 goroutine 会承担 mark assist 工作。
- span 在使用前可能触发按需清扫。

下一篇将沿着这条接口继续解释标记、写屏障、assist 和 pacer。

## 从page allocator到操作系统

`mheap` 管理的 page 是 runtime 视角的虚拟地址页，并不等于这些地址已经全部占用物理内存。更底层的 [`sysAlloc`、`sysMap`、`sysUsed`、`sysUnused`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mem.go#L49) 把跨平台内存状态翻译到 OS 实现：

```text
None      地址范围尚未保留
Reserved  地址空间已保留，但不可直接访问
Prepared  OS已准备，可转为Ready
Ready     可安全读写，可能占用物理页/RSS
```

这些是 runtime 的抽象状态，不应直接套用某一个内核 API。Linux 的 [`mem_linux.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mem_linux.go#L21) 主要使用 `mmap`、`madvise` 等机制：保留虚拟地址、让页可用、提示内核回收物理页的动作彼此分开。

因此 `HeapSys`、进程虚拟地址空间和 RSS 不是同一个量：

- `HeapSys` 表示 runtime 从 OS 获得的堆地址空间规模。
- `HeapIdle` 表示已获得但当前未用于 live span 的部分。
- `HeapReleased` 表示已提示 OS 可回收的物理页。
- RSS 由内核驻留策略、页是否实际触碰、其他映射和统计口径共同决定。

对象被 GC 判死，只会先让对应 span/object 变得可复用；这不保证进程 RSS 同步下降。

## Scavenger归还物理页

scavenger 处理的是页分配器中的空闲页，不负责判断对象存活。[`bgscavenge`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcscavenge.go#L649) 是后台 scavenger G；[`pageAlloc.scavenge`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcscavenge.go#L675) 按目标字节数查找候选区间，[`scavengeOne`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcscavenge.go#L732) 在 chunk 中选择连续空闲页并调用 `sysUnused`。

```text
对象不可达
  -> sweep清除allocBits并释放空span
  -> page allocator看到空闲页
  -> scavenger选择足够老/合适的连续范围
  -> sysUnused提示OS回收物理页
```

scavenger 有独立于 GC mark pacer 的 [`gcPaceScavenger`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcscavenge.go#L167)，但输入仍会考虑 heap goal、内存限制、页年龄和物理页回收成本。后台 scavenger 默认按有限 CPU 利用率目标节流，内存压力较强时也可能进入更积极的 synchronous scavenging。强制频繁归还会降低 RSS，却可能在之后重新 fault 页面并增加延迟；只看 RSS 最小化并不是分配器调优目标。

## 源码入口

| 目标 | 入口 |
| --- | --- |
| 对象分配 | [`runtime/malloc.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/malloc.go#L1119) |
| P缓存 | [`runtime/mcache.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mcache.go#L20) |
| 中心缓存 | [`runtime/mcentral.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mcentral.go#L22) |
| mspan、mheap、arena | [`runtime/mheap.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mheap.go#L64) |
| size class | [`internal/runtime/gc/sizeclasses.go`](https://github.com/golang/go/blob/go1.26.2/src/internal/runtime/gc/sizeclasses.go) |
| 页分配 | [`runtime/mpagealloc.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mpagealloc.go#L879) |
| 物理页回收 | [`runtime/mgcscavenge.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcscavenge.go) |
| Green Tea标记位 | [`runtime/mgcmark_greenteagc.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mgcmark_greenteagc.go) |
| OS内存状态 | [`runtime/mem.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mem.go#L49)、[`runtime/mem_linux.go`](https://github.com/golang/go/blob/go1.26.2/src/runtime/mem_linux.go#L21) |

## 观测与验证

```bash
GODEBUG=gctrace=1 go test -run='^$' -bench=. -benchmem ./...
go test -run='^$' -bench=. -memprofile=mem.pprof ./...
go tool pprof -alloc_space mem.pprof
go tool pprof -inuse_space mem.pprof
```

`alloc_space` 回答累计分配压力，`inuse_space` 更接近采样时仍存活的堆。二者都受内存采样率影响；小基准应配合 `-memprofilerate`、多次运行和 benchmark 的 `allocs/op`，不能从单张 heap profile 反推出每个对象精确生命周期。

## 延伸阅读

- [Go语言内存分配器的实现原理](https://draven.co/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)：适合阅读分级分配的设计背景；Go 1.26 的专用分配和复用路径以本文源码为准。
- [runtime.MemStats](https://pkg.go.dev/runtime#MemStats)

## 系列导航

- [系列目录](/posts/go-runtime/)
- [上一篇：Goroutine栈管理](/posts/go-runtime-03-stack/)
- 当前：Go Runtime（四）：内存分配器
- [下一篇：GC、写屏障与Pacer](/posts/go-runtime-05-gc/)
