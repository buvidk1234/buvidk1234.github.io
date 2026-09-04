+++
date = '2026-08-17T02:44:43+08:00'
draft = false
title = 'Go Runtime系列导读'
tags = ['Go', 'Runtime', 'Concurrency']
+++

一次 Go 程序执行会经过编译器、链接器、标准库、runtime 与操作系统。下面从一段程序开始，依次展开各层的职责与协作关系。

## 一个程序的执行路径

下面的程序把本系列的主要公共抽象放进同一条调用链，并由每个语句引出后续的系统问题：

```go
package main

import (
	"context"
	"fmt"
	"net"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	result := make(chan error, 1)
	go func() {
		var d net.Dialer
		conn, err := d.DialContext(ctx, "tcp", "127.0.0.1:8080")
		if err == nil {
			err = conn.Close()
		}
		result <- err
	}()

	select {
	case err := <-result:
		fmt.Println(err)
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```

从这段程序及其运行环境，可以继续追问一组系统问题：

| 程序现象 | 需要解释的机制 |
| --- | --- |
| 可执行文件为什么能直接启动 | 链接入口、`rt0_go`、第一个 M/P/G |
| `go func()` 如何映射到 OS 线程 | GMP、运行队列、park/ready、抢占 |
| goroutine 为什么能从很小的栈开始 | 栈检查、`morestack`、栈复制与 stack map |
| `make` 和闭包需要的内存从哪里来 | 逃逸分析、mcache、mcentral、mheap |
| 失去引用的对象何时回收 | 标记、写屏障、assist、sweep、scavenger |
| `DialContext` 等待网络时如何使用线程 | 非阻塞 fd、netpoll 与阻塞 syscall 的分流 |
| 信号或 C 线程如何进入 Go 世界 | gsignal、异步抢占、cgo、extra M |
| Channel 和 `select` 如何保证可靠唤醒 | `hchan`、`sudog`、等待队列与多路竞争 |
| 标准库的锁如何停下一个 G | 原子快路径、runtime semaphore、调度器 |
| 一秒超时由谁唤醒 | 每 P Timer heap、netpoll deadline |
| 取消为何能传播到网络调用 | Context 取消树、Channel、锁与 Timer |

## 系列目录

现有 11 篇按系统职责分成四组：

| 主题组 | 文章 | 核心问题 |
| --- | --- | --- |
| 执行基础 | [01 架构与启动](/posts/go-runtime-01-bootstrap/)、[02 GMP调度](/posts/go-runtime-02-scheduler/)、[03 Goroutine栈](/posts/go-runtime-03-stack/) | 程序如何开始，G 如何获得 CPU，调用现场如何保存和迁移 |
| 内存 | [04 分配器](/posts/go-runtime-04-allocator/)、[05 GC](/posts/go-runtime-05-gc/) | 堆空间如何取得、描述、追踪、复用并归还给 OS |
| OS 边界 | [06 Netpoll与Syscall](/posts/go-runtime-06-netpoll-syscall/)、[07 Signal与cgo](/posts/go-runtime-07-signal-cgo/) | Go 执行模型如何面对内核阻塞、异步信号和外部线程 |
| 同步与取消 | [08 Channel与Select](/posts/go-runtime-08-channel-select/)、[09 同步原语](/posts/go-runtime-09-sync-primitives/)、[10 Timer](/posts/go-runtime-10-timer/)、[11 Context](/posts/go-runtime-11-context/) | 语言和标准库怎样组合底层机制，形成稳定的并发契约 |

`Context` 是标准库提供的协作式取消抽象。最后一篇通过取消传播，把前文的 Channel、锁、原子操作和 Timer 组合成一个完整的请求生命周期协议。

`Netpoll + Syscall` 与 `Signal + cgo` 各自组成一篇。前一组对比“只阻塞 G”和“可能阻塞 M”；后一组共同处理异步现场、线程局部状态、signal stack 与非 Go 线程。每组都围绕一个端到端问题展开。

## 阅读路线

首次阅读可以直接按 01 到 11 前进。Timer、netpoll 和 Channel 之间的交叉引用会先给出结论，再由后文补齐实现。

也可以按具体问题选择阅读路线：

| 目标 | 建议顺序 |
| --- | --- |
| 建立 goroutine 执行模型 | 01 -> 02 -> 03 |
| 理解堆、GC 与 RSS | 01 -> 03 -> 04 -> 05 |
| 排查调度与 I/O 阻塞 | 01 -> 02 -> 06 -> 10 -> 07 |
| 理解 Go 并发原语 | 01 -> 02 -> 08 -> 09 -> 10 -> 11 |

## 版本与证据

系列实现基线为 **Go 1.26.2**；涉及平台路径时默认 **Linux/amd64**。证据按以下优先级使用：

1. Go 语言规范、内存模型和标准库文档定义公开契约。
2. `go1.26.2` tag 的编译器、标准库和 runtime 源码解释这一版如何实现契约。
3. trace、profile、反汇编和最小程序验证实际构建产物。

以下命令可以直接读取精确版本，并保留当前工作分支：

```bash
git show go1.26.2:src/runtime/proc.go
git show go1.26.2:src/context/context.go
```
