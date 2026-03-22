+++
date = '2026-03-22T14:14:08+08:00'
draft = false
title = 'LSM-Tree 原理与消息存储选型'
tags = ['IM', 'DDIA', 'LSM-Tree', '存储引擎']
+++

## 1. 概述

LSM-Tree（Log-Structured Merge-Tree）是一种典型的**写优化**存储结构。它的核心思路是：不追求每次写入都直接落到最终有序位置，而是将写入拆成**"前台快速落地 + 后台异步整理"**两个阶段。

设计目标可以归纳为：

- **顺序化写入**：前台写入以追加为主，避免随机写。
- **后台整理**：异步 Compaction 控制查询路径复杂度。
- **可调节的权衡**：在写吞吐、读延迟和空间占用之间提供调优空间。

---

## 2. 整体架构与核心组件

LSM-Tree 的运行依赖以下组件协同工作：

| 组件 | 位置 | 职责 |
|------|------|------|
| WAL（Write-Ahead Log） | 磁盘 | 写前日志，保证崩溃恢复 |
| MemTable | 内存 | 有序表，接收实时写入 |
| Immutable MemTable | 内存 | 写满后冻结，等待刷盘 |
| SSTable | 磁盘 | 有序、不可变的持久化文件 |
| Compaction | 后台任务 | 合并 SSTable，版本收敛与空间回收 |

数据在组件间的流转路径如下：

```mermaid
flowchart LR
	W[Write Request] --> L[Append WAL]
	L --> M[Insert MemTable]
	M -->|Threshold reached| IM[Immutable MemTable]
	IM --> F[Flush to SSTable]
	F --> C[Compaction]
	C --> S[(Merged SSTables)]
```

---

## 3. 写路径

写入的关键原则是 **"先确认可恢复，再确认可查询"**。具体步骤：

1. **追加 WAL**——保证持久化，崩溃后可重放。
2. **更新 MemTable**——对外可查。
3. **冻结 MemTable**——达到阈值后转为 Immutable。
4. **Flush 为 SSTable**——后台将 Immutable MemTable 写入磁盘。

这条路径带来的关键特性：

- 前台不做磁盘就地更新，写入代价可预期。
- 磁盘 I/O 模式以追加和顺序刷盘为主。
- 崩溃恢复依赖 WAL 重放，而非 undo/redo 日志。

---

## 4. 读路径

点查按 **"由新到旧"** 的顺序逐层查找：

1. MemTable
2. Immutable MemTable
3. SSTable 层级（从新到旧）

由于可能涉及多层查找，读路径通常依赖以下加速结构来减少 I/O：

| 加速结构 | 作用 |
|----------|------|
| Bloom Filter | 快速排除不存在 key 的 SSTable |
| Block Index | 缩小需要读取的数据块范围 |
| Block Cache | 缓存热点数据块，减少磁盘访问 |

**Bloom Filter 的特性**：

- **可能 false positive**：过滤器判断"可能存在"，但实际 key 不在该 SSTable 中——此时需要继续读取确认。
- **不会 false negative**：过滤器一旦判断"不存在"，则该 key 一定不在该 SSTable 中——可安全跳过。

对实际查询的意义：不存在的 key 可以被快速排除，大幅减少无效 I/O；存在的 key 仍需结合索引和数据块完成最终读取。

---

## 5. Compaction

Compaction 是 LSM-Tree 的核心后台机制。虽然名称中带有"压缩"的含义，但它并非像 gzip 那样对数据做编码压缩，而是通过**合并文件、收敛版本、清理过期数据**来维护读性能和空间可控性。

**主要任务**：

- 合并 SSTable，降低文件数量，控制读放大。
- 对同一 key 进行版本收敛，只保留最新有效版本。
- 清理已过期的 tombstone 标记，回收磁盘空间。

**对应代价**：

- 增加写放大（Write Amplification）。
- 占用后台 I/O 带宽，可能引起尾延迟抖动。

**常见策略**：

| 策略 | 写放大 | 读放大 | 空间放大 | 典型使用 |
|------|--------|--------|----------|----------|
| Size-Tiered | 低 | 高 | 高 | Cassandra 默认 |
| Leveled | 高 | 低 | 低 | LevelDB / RocksDB |

**Size-Tiered Compaction（STCS）**：当同一层积累了若干个大小相近的 SSTable 后，将它们合并为一个更大的 SSTable 推入下一层。写入时只需攒够数量即触发，因此写放大低；但同一个 key 可能同时存在于多个 SSTable 中，点查需要遍历多个文件，读放大高；合并前后新旧文件共存，空间放大也高。

**Leveled Compaction**：将 SSTable 组织为多个层级（L0、L1、L2…），每层有容量上限。当某层超限时，选取部分 SSTable 与下一层中 key 范围重叠的文件合并。这保证了 L1+ 每层内 key 范围不重叠——点查在每层最多访问一个文件，读放大低；但每次合并都可能涉及多个文件重写，写放大高。

实际系统往往混合使用两种策略，例如 RocksDB 在 L0 采用类 Size-Tiered 模式（允许 key 重叠），L1+ 采用 Leveled 模式（保证层内无重叠）。

---

## 6. 三类放大与调优

LSM-Tree 的调优本质是在三类放大之间寻求平衡：

- **WA（Write Amplification）**：同一份数据在生命周期内被写入磁盘的总次数。
- **RA（Read Amplification）**：一次逻辑读取需要访问的物理 I/O 次数。
- **SA（Space Amplification）**：实际占用空间与有效数据量的比值。

建议长期观测以下指标，而不是只看单点 QPS：

| 维度 | 关键指标 |
|------|----------|
| 写路径 | P99 写延迟、写入吞吐、写放大比 |
| 读路径 | P99 点查延迟、缓存命中率、Bloom 误判率 |
| 后台任务 | Compaction backlog、后台 I/O 占比、尾延迟抖动 |
| 空间 | 压缩率、tombstone 清理周期、磁盘增长率 |

---

## 7. LSM-Tree 与 B+Tree 对比

两者的本质差异在于：是否接受"后台异步整理换取前台写入稳定"。

| 维度 | LSM-Tree | B+Tree |
|------|----------|--------|
| 写入模型 | 追加写 + 后台合并 | 就地更新 + 页分裂/重平衡 |
| 随机写压力 | 低 | 高 |
| 点查稳定性 | 受层数/缓存命中率影响 | 路径固定，波动较小 |
| 范围查询 | 跨文件归并 | 叶子链顺序扫描 |
| 后台维护 | Compaction 调度 | 页管理与分裂管理 |

没有绝对优劣——写密集场景（日志、时序、消息）偏向 LSM；读密集且要求点查延迟稳定的场景（OLTP 事务）偏向 B+Tree。

---

## 8. 消息存储选型

前面讨论了 LSM-Tree 的通用原理，这一节聚焦一个具体问题：**消息系统的在线存储应该选什么引擎？**

### 8.1 业务约束分析

消息系统的存储层面临以下典型约束：

- **写入模式**：每条消息写入一次，几乎不更新。峰值写入量可能是均值的 5-10 倍（如早高峰、活动推送）。
- **读取模式**：绝大多数读取集中在最近消息（打开会话加载最新 N 条）。历史消息翻页频率低，属于冷路径。
- **顺序性要求**：同一会话内的消息必须按序返回，跨会话无全局排序需求。
- **延迟要求**：发送端写入延迟需稳定在低毫秒级；接收端拉取最近消息延迟要求 P99 可控。
- **数据生命周期**：消息存在明显的冷热分界——最近 7 天的数据承载 90%+ 的读取量。

### 8.2 为什么 LSM 适合这个场景

将上述约束逐条映射到 LSM 的特性：

| 业务约束 | LSM 的匹配点 |
|----------|-------------|
| 写入一次不更新 | 追加写模型，无就地更新开销 |
| 峰值写入高 | 前台只写 MemTable，不受磁盘随机 I/O 限制 |
| 最近消息热读 | Block Cache 可覆盖热数据，减少磁盘访问 |
| 会话内顺序读 | RowKey 按会话分区 + 时间排序，SSTable 内数据天然有序 |
| 冷热分界明显 | 配合 TTL 和分层 Compaction，冷数据自动下沉 |

相比之下，B+Tree 型关系存储在高写吞吐场景下面临随机写放大和页分裂压力，水平扩展也需要额外的分库分表方案。

### 8.3 RowKey 设计

RowKey 的设计直接决定数据在 SSTable 中的物理布局，进而影响读取效率：

```
RowKey = {conversation_id}#{msg_sequence}
```

- `conversation_id` 作为前缀保证同会话数据在 SSTable 中物理相邻，范围扫描高效。
- `msg_sequence` 使用单调递增序列（如 Snowflake ID），保证会话内消息有序。
- 避免使用时间戳作为唯一排序键——高并发下时间戳可能重复，且纯时间前缀会导致写入热点集中在少数 SSTable。

### 8.4 冷热分离

在线层不应承载全生命周期数据，否则 Compaction 的空间放大会持续增长：

- **热数据层**（最近 N 天）：LSM 引擎（如 RocksDB / Cassandra），配合 Block Cache 保证低延迟读取。
- **冷数据层**（历史归档）：异步归档到对象存储（如 S3 / MinIO），按会话打包，仅在用户主动翻页时按需加载。
- **过渡策略**：通过 TTL 自动过期 + 后台归档任务，而非手动迁移。

---

## 9. 总结

LSM-Tree 适用于写密集场景，其核心取舍是用后台 Compaction 成本换取前台写入的高吞吐与顺序性。使用 LSM 的前提是：系统能够持续承担 Compaction 带来的写放大和 I/O 开销，并具备对三类放大（WA / RA / SA）进行长期观测与调优的能力。

---

## 参考资料

1. Martin Kleppmann. *Designing Data-Intensive Applications*. O'Reilly Media, 2017. Chapter 3: Storage and Retrieval, section "SSTables and LSM-Trees".
2. Fay Chang, Jeffrey Dean, Sanjay Ghemawat, et al. *Bigtable: A Distributed Storage System for Structured Data*. OSDI, 2006.
