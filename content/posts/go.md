+++
date = '2026-03-02T17:44:21+08:00'
draft = false
title = 'Go'
+++

> draft
> need to be improved

## Slice
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
// 扩容
func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		return newLen
	}
	const threshold = 256
	if oldCap < threshold {
		return doublecap
	}
	for {
		// 1.25x
		newcap += (newcap + 3*threshold) >> 2
		if uint(newcap) >= uint(newLen) {
			break
		}
	}
	// overflowed
	if newcap <= 0 {
		return newLen
	}
	return newcap
}
```
## Map

```go
type Map struct {
	
	used uint64 // The number of filled slots
	seed uintptr // the hash seed, computed as a unique random number per map.

	// The directory of tables.
	// Normally dirPtr points to an array of table pointers
	// dirPtr *[dirLen]*table
	dirPtr unsafe.Pointer
    
	dirLen int // 1 << globalDepth
	globalDepth uint8 // The number of bits to use in table directory lookups.
	globalShift uint8 // The number of bits to shift out, 64 - globalDepth
	writing uint8 // detect the race.
	tombstonePossible bool // whether a table in this map contains a tombstone.
	clearSeq uint64 // version number, used to detect map clears during iteration.
}

```

### Swiss Table
> refer: [Faster Go maps with Swiss Tables](https://golang.google.cn/blog/swisstable)

![swiss table](/images/swiss_table.png)

插入过程：

1. 计算 `hash(key)` 并将其分成两部分：高 57 位（称为 `h1`）和低 7 位（称为 `h2`）。
2. 高位（`h1`）用于选择要考虑的第一个组：在本例中为 `h1 % 2`，因为只有 2 个组。
3. 在一个组内，所有槽都有可能容纳该键。我们必须首先确定是否有槽已包含此键，在这种情况下，这是一个更新而不是新的插入。（SIMD）
4. 如果没有槽包含该键，那么我们寻找一个空槽来放置该键。
5. 如果没有空槽，则继续探测序列，搜索下一个组。

### 增量式增长
> refer: [Extendible hashing](https://en.wikipedia.org/wiki/Extendible_hashing)

将每个 Map 分成多个 Swiss Tables。每个 Map 由一个或多个独立表组成，这些表覆盖键空间的一部分，而不是由一个 Swiss Table 实现整个 Map。单个表最多存储 1024 个条目。哈希中可变数量的高位用于选择键所属的表。（可扩展哈希）

### 迭代期间的修改

迭代期间可修改

- 如果在到达条目之前删除该条目，则不会生成该条目。
- 如果在到达条目之前更新该条目，则会生成更新后的值。
- 如果添加了新条目，则可能生成也可能不生成。

## Channel

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	timer    *timer // timer feeding this chan
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters
	bubble   *synctestBubble

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	lock mutex
}
type waitq struct {
	first *sudog
	last  *sudog
}
```
### 发送数据
依次尝试 recvq->buf->sendq
向已关闭的channel发送数据会panic
### 接收数据
sendq->buf->recvq
向已关闭且缓冲区为空的channel读取数据会返回零值
## Select
```go
type scase struct {
	c    *hchan         // chan
	elem unsafe.Pointer // data element
}
func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
	...
	// 洗牌算法，保证概率均等不会出现饥饿
	norder := 0
	for i := range scases {
		...
		j := cheaprandn(uint32(norder + 1))
		pollorder[norder] = pollorder[j]
		pollorder[j] = uint16(i)
		norder++
	}
	...
	// 堆排序，死锁避免
	// sort the cases by Hchan address to get the locking order.
	// simple heap sort, to guarantee n log n time and constant stack footprint.
	// 插入建堆，上滤（还有一种自底向上建堆的方式）
	for i := range lockorder {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	// 排序与节点下沉
	for i := len(lockorder) - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}
	...

	// lock all the channels involved in the select
	sellock(scases, lockorder)

	var (
		sg     *sudog
		c      *hchan
		k      *scase
		sglist *sudog
		sgnext *sudog
		qp     unsafe.Pointer
		nextp  **sudog
	)

	// pass 1 - look for something already waiting
	var casi int
	var cas *scase
	var caseSuccess bool
	var caseReleaseTime int64 = -1
	var recvOK bool
	for _, casei := range pollorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c

		if casi >= nsends { // 接收
			sg = c.sendq.dequeue()
			if sg != nil { // 通道满了，发送方挂起等待
				goto recv
			}
			if c.qcount > 0 { // 有缓存
				goto bufrecv
			}
			if c.closed != 0 { // 通道关闭
				goto rclose
			}
		} else { // 发送
			if raceenabled {
				racereadpc(c.raceaddr(), casePC(casi), chansendpc)
			}
			if c.closed != 0 {
				goto sclose
			}
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			if c.qcount < c.dataqsiz {
				goto bufsend
			}
		}
	}

	if !block {
		selunlock(scases, lockorder)
		casi = -1
		goto retc
	}
	
	// 生成当前协程的sudog,放入所有channel的等待队列中,挂起等待唤醒
	// pass 2 - enqueue on all chans
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		// No stack splits between assigning elem and enqueuing
		// sg on gp.waiting where copystack can find it.
		sg.elem.set(cas.elem)
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c.set(c)
		// Construct waiting list in lock order.
		*nextp = sg
		nextp = &sg.waitlink

		if casi < nsends {
			c.sendq.enqueue(sg)
		} else {
			c.recvq.enqueue(sg)
		}

		if c.timer != nil {
			blockTimerChan(c)
		}
	}

	// wait for someone to wake us up
	gp.param = nil
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	gp.parkingOnChan.Store(true)
	gopark(selparkcommit, nil, waitReason, traceBlockSelect, 1)
	gp.activeStackChans = false

	sellock(scases, lockorder)

	gp.selectDone.Store(0)
	sg = (*sudog)(gp.param)
	gp.param = nil

	// pass 3 - dequeue from unsuccessful chans
	// otherwise they stack up on quiet channels
	// record the successful case, if any.
	// We singly-linked up the SudoGs in lock order.
	casi = -1
	cas = nil
	caseSuccess = false
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem.set(nil)
		sg1.c.set(nil)
	}
	gp.waiting = nil

	for _, casei := range lockorder {
		k = &scases[casei]
		if k.c.timer != nil {
			unblockTimerChan(k.c)
		}
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			casi = int(casei)
			cas = k
			caseSuccess = sglist.success
			if sglist.releasetime > 0 {
				caseReleaseTime = sglist.releasetime
			}
		} else {
			c = k.c
			if int(casei) < nsends {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	if cas == nil {
		throw("selectgo: bad wakeup")
	}

	c = cas.c

	if casi < nsends {
		if !caseSuccess {
			goto sclose
		}
	} else {
		recvOK = caseSuccess
	}
	...
	selunlock(scases, lockorder)
	goto retc

bufrecv:
bufsend:
recv:
rclose:
send:
	...
	goto retc

retc:
	if caseReleaseTime > 0 {
		blockevent(caseReleaseTime-t0, 1)
	}
	return casi, recvOK

sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```

## Sync
### Mutex
```go
// 互斥锁的公平性。
//
// 互斥锁有两种工作模式：正常模式（normal）和饥饿模式（starvation）。
// 在正常模式下，等待者（waiters）按 FIFO 顺序排队，但被唤醒的等待者
// 并不会直接拥有互斥锁，而是需要与新到达的 goroutine 竞争锁的所有权。
// 新到达的 goroutine 有优势——它们已经在 CPU 上运行，而且可能有很多个，
// 因此被唤醒的等待者很有可能竞争失败。在这种情况下，它会被重新放回
// 等待队列的队首。如果一个等待者在超过 1ms 的时间内仍未能获得互斥锁，
// 则会将互斥锁切换到饥饿模式。
//
// 在饥饿模式下，互斥锁的所有权会由解锁的 goroutine
// 直接移交给等待队列最前面的等待者。
// 新到达的 goroutine 即使看到锁似乎已经解锁，也不会尝试获取锁，
// 也不会进行自旋；相反，它们会把自己加入到等待队列的尾部。
//
// 如果某个等待者获得了互斥锁，并且发现以下任一情况成立：
// (1) 它是队列中的最后一个等待者；或者
// (2) 它等待的时间少于 1ms，
// 那么它会将互斥锁切换回正常模式。
//
// 正常模式的性能要好得多，因为即使有被阻塞的等待者，
// 某个 goroutine 仍然可能连续多次获取同一个互斥锁。
// 而饥饿模式对于防止某些极端情况下的尾延迟（tail latency）问题非常重要。
const (
    mutexLocked = 1 << iota // bit0: 是否已加锁
    mutexWoken              // bit1: 是否已有被唤醒的等待者
    mutexStarving           // bit2: 是否处于饥饿模式
    mutexWaiterShift = iota // 从 bit3 开始存等待者数量
)
type Mutex struct {
	state int32
	sema  uint32
}

func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}

func (m *Mutex) lockSlow() {
	var waitStartTime int64 // 开始排队等待的时间
	starving := false       // 是否因为等待时间过长而处于“饥饿”状态
	awoke := false          // 是否是刚从沉睡（等待队列）中被唤醒的
	iter := 0               // 尝试自旋（Spin）的次数
	old := m.state          // 记录互斥锁当前的状态

	// 进入一个死循环，不断尝试获取锁或者挂起自己
	for {
		// ==========================================
		// 第一阶段：尝试自旋 (Spinning)，这是一种乐观等待机制
		// 只有在满足以下所有条件时才会自旋：
		// 1. 锁当前被占用 (mutexLocked == 1)
		// 2. 锁没有处于饥饿模式 (mutexStarving == 0)。饥饿模式下绝对不允许自旋插队。
		// 3. 当前的 CPU 状态允许自旋 (runtime_canSpin)。比如多核且有空闲的 P，且自旋次数不足 4 次。
		// ==========================================
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			
			// 在自旋的过程中，如果当前 goroutine 没有被打上"已唤醒"标记，
			// 且锁的"已唤醒"标志位还没有被别人设置，而且等待队列里确实有其他人正在排队，
			// 那么当前 goroutine 就尝试用 CAS 把锁的状态标记为"已唤醒" (mutexWoken)。
			// 为什么要这样做？
			// 这是为了告诉当前正在持有锁的那个 goroutine："嘿，我就在这里盯着呢(在CPU上空转)，
			// 你一会儿释放锁的时候直接走人就行，千万别再去慢吞吞地唤醒队列里的老实人了，把锁留给我就行！"
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			
			runtime_doSpin() // 执行底层汇编 CPU PAUSE 指令，让出几个时钟周期，但不让出线程
			iter++           // 增加自旋次数
			old = m.state    // 刷新锁的旧状态，看看主人在自旋期间是不是把锁释放了
			continue         // 继续这轮抢锁尝试
		}

		// ==========================================
		// 第二阶段：走到这里，说明要么没资格自旋，要么自旋结束了（无论持锁者是否已释放锁）。
		// 我们需要基于目前的旧状态 old，推算出我们期望锁变成的新状态 new。
		// ==========================================
		new := old
		
		// 1. 如果当前不是饥饿模式，说明大家公平竞争，那么新来的或刚醒的 goroutine 就可以尝试设置加锁标志位！
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		
		// 2. 如果锁当前被占用，或者正处于不能插队的"饥饿模式"
		// 那么当前 goroutine 没有任何取巧办法，只能乖乖去排队（将 Waiter 等待者的数量 +1）
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		
		// 当前线程饥饿，锁仍被占用
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		
		// 4. 清除唤醒标志位：如果当前 goroutine 之前被标记为 awoke (不管是自旋设置的还是被唤醒的)
		if awoke {
			// 将 Woken 标志位清零（因为当前 goroutine 已经"醒了"，使命完成了）
			new &^= mutexWoken
		}

		// ==========================================
		// 第三阶段：尝试使用 CAS 原语，把算好的期望新状态 new 替换掉实际的锁状态。
		// ==========================================
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			
			// 如果旧状态既没有被锁死，也不处于饥饿模式 -> CAS抢到了一个空闲的锁
			if old&(mutexLocked|mutexStarving) == 0 {
				break
			}
			
			// 走到这，说明 CAS 虽然成功修改了状态（比如成功给自己排上了队，或者成功把锁打上了饥饿标记），
			// 但是并没有抢到锁的所有权。接下来只能去睡觉（阻塞挂起）了。

			queueLifo := waitStartTime != 0 // 排到最前面
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime() // 新来的，记录一下开始排队的时间点
			}
			
			// 将自己挂起，让出 P，陷入沉睡，等待被别人通过信号量唤醒...
			runtime_SemacquireMutex(&m.sema, queueLifo, 2)

			// 当前 goroutine 被某一个释放锁的人唤醒了！

			// 醒来后，计算等待时间(1ms)，是否饥饿
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state // 醒来时刻锁状态

			// 关键判断：目前锁处于什么模式？
			if old&mutexStarving != 0 {
				// 【逻辑分支 A：锁已经处于饥饿模式！】
				// 锁直接给了当前线程
				
				// 既然我拿到了锁，那就需要修正状态：加上 Locked 标志，减去 Waiter 人数 1
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				
				// 如果我不饿（等待没超过1ms），队列只有我自己，关闭饥饿模式
				if !starving || old>>mutexWaiterShift == 1 {
					delta -= mutexStarving
				}
				
				// 通过原子操作应用这些修正逻辑
				atomic.AddInt32(&m.state, delta)
				break // 拿到锁，跳出死循环
			}
			
			// 【逻辑分支 B：锁处于正常模式】
			// 被唤醒，锁处于正常模式，需要与刚刚来抢锁的那些正在 CPU 上活蹦乱跳的 goroutine 公平竞争
			awoke = true // 标记是刚醒的（这样在上面第二阶段就可以清除锁的 Woken 标记）
			iter = 0     // 经历了一次沉睡，重新开始清算自旋次数
			
			// 回到 for 循环的头部，刚唤醒需要线程调度优劣势，和那些新来的线程抢锁。
		} else {
			// CAS 失败了，说明在计算 new 的过程中，有其他 goroutine 修改了锁的状态。
			// 刷新 old 为最新的状态，回到 for 循环重新再计算一次期望的状态 new。
			old = m.state 
		}
	}
}

```


### RWMutex
```go
// A RWMutex is a reader/writer mutual exclusion lock.
// The lock can be held by an arbitrary number of readers or a single writer.
// The zero value for a RWMutex is an unlocked mutex.
//
// If any goroutine calls [RWMutex.Lock] while the lock is already held by
// one or more readers, concurrent calls to [RWMutex.RLock] will block until
// the writer has acquired (and released) the lock, to ensure that
// the lock eventually becomes available to the writer.
// Note that this prohibits recursive read-locking.
// A [RWMutex.RLock] cannot be upgraded into a [RWMutex.Lock],
// nor can a [RWMutex.Lock] be downgraded into a [RWMutex.RLock].
//
type RWMutex struct {
	w           Mutex        // held if there are pending writers
	writerSem   uint32       // semaphore for writers to wait for completing readers
	readerSem   uint32       // semaphore for readers to wait for completing writers
	readerCount atomic.Int32 // number of pending readers
	readerWait  atomic.Int32 // number of departing readers
}
```
### Once
```go
type Once struct {
	_ noCopy
	done atomic.Bool
	m    Mutex
}
func (o *Once) Do(f func()) {
	if !o.done.Load() {
		o.doSlow(f)
	}
}
func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if !o.done.Load() {
		defer o.done.Store(true)
		f()
	}
}
```
### WaitGroup
```go
type WaitGroup struct {
	noCopy noCopy

	// Bits (high to low):
	//   bits[0:32]  counter
	//   bits[32]    flag: synctest bubble membership
	//   bits[33:64] wait count
	state atomic.Uint64
	sema  uint32
}
```

### Map
```go
type Map struct {
	_ noCopy

	m isync.HashTrieMap[any, any]
}

type HashTrieMap[K comparable, V any] struct {
	inited   atomic.Uint32
	initMu   Mutex
	root     atomic.Pointer[indirect[K, V]]
	keyHash  hashFunc
	valEqual equalFunc
	seed     uintptr
}
```
## 内存模型
> 参考 [The Go Memory Model](https://go.dev/ref/mem)

### Happens-Before
Go内存模型定义了在什么条件下，一个goroutine对变量的写入能被另一个goroutine的读取观察到。

**数据竞争(data race)**：对同一内存位置的写操作与另一个读/写操作并发执行，且至少有一个不是原子操作。无数据竞争的程序表现为所有goroutine在单处理器上顺序执行（DRF-SC）。

**Happens-before** 是 sequenced before（同一goroutine内的程序顺序）和 synchronized before（跨goroutine的同步关系）的传递闭包。对于普通读操作 r，它读取到的值必须是某个对 r **可见**的写操作 w 所写的值——即 w happens before r，且在 w 和 r 之间没有其他写操作。

### 同步保证

**初始化**：包 p 导入包 q，则 q 的 `init` 函数完成 *synchronized before* p 的 `init` 开始。所有 `init` 完成 *synchronized before* `main.main` 开始。

**Goroutine 创建**：`go` 语句 *synchronized before* 新goroutine执行开始。
```go
var a string
func f() { print(a) }
func hello() {
	a = "hello, world"
	go f() // 保证能打印 "hello, world"
}
```

**Goroutine 销毁**：goroutine的退出**不保证** *synchronized before* 程序中的任何事件，必须使用同步机制。

**Channel 通信**：
- 向channel发送 *synchronized before* 对应的接收完成
- channel关闭 *synchronized before* 因关闭而收到零值的接收
- **无缓冲channel**：接收 *synchronized before* 对应的发送完成
- 容量为C的channel上第k次接收 *synchronized before* 第k+C次发送完成（可用于限流信号量）

```go
var c = make(chan int)
var a string
func f() {
	a = "hello, world"
	<-c
}
func main() {
	go f()
	c <- 0     // 无缓冲: 接收 synchronized before 发送完成
	print(a)   // 保证打印 "hello, world"
}
```

**Locks**：对于 `sync.Mutex`/`sync.RWMutex` 变量 l，第n次 `l.Unlock()` *synchronized before* 第m次 `l.Lock()` 返回（n < m）。

**Once**：`once.Do(f)` 中 f() 的完成 *synchronized before* 任何 `once.Do(f)` 调用的返回。

**Atomic**：如果原子操作A的效果被原子操作B观察到，则A *synchronized before* B。所有原子操作表现为某种顺序一致的全序。

### 错误的同步
```go
// ❌ 双重检查锁定 - 观察到done=true不意味着能观察到a的写入
var a string
var done bool
func setup() {
	a = "hello, world"
	done = true
}
func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a) // 可能打印空字符串！
}
```
```go
// ❌ 忙等待 - 不保证能观察到done的写入，循环可能永不结束
var a string
var done bool
func setup() {
	a = "hello, world"
	done = true
}
func main() {
	go setup()
	for !done {
	}
	print(a) // 可能打印空字符串，甚至死循环
}
```
```go
// ❌ 指针观察 - 即使观察到g!=nil，也不保证能观察到g.msg的初始化
type T struct { msg string }
var g *T
func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}
func main() {
	go setup()
	for g == nil {
	}
	print(g.msg) // 可能打印空字符串
}
```

以上错误的根本原因都是缺少显式同步，应使用channel、mutex、atomic等同步原语建立happens-before关系。

## Context
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{})
}
```
## Interface
```go
type eface struct {
   _type *_type
   data  unsafe.Pointer
}

type iface struct {
   tab  *itab
   data unsafe.Pointer
}
type itab struct {
   inter *interfacetype
   _type *_type
   hash  uint32 // copy of _type.hash. Used for type switches.
   _     [4]byte
   fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```
## 反射
通过反射可获取接口的类型和数据信息
## GMP
goroutine, machine, processor
一个machine绑定一个processor, processor调度goroutine，优先处理本地队列的协程。
每个processor有一个本地队列，还有一个全局队列。
## 内存管理
mcache,mcentral, mheap
微小对象，小对象，大对象
## 垃圾回收
三色标记：
白色：未访问
灰色：已访问，未完全处理引用对象
黑色：已访问，并处理了引用对象
广度优先搜索