+++
date = '2026-03-05T15:34:07+08:00'
draft = false
title = 'Id Seq Generation'
+++

# 分布式 ID / 序列号生成方案技术选型

## 1. 方案概述

### 1.1 UUID

128 位随机数（V4）或基于时间+MAC（V1）。无需中心节点，生成即唯一。

- 完全无序、不连续，无法用于排序或范围查询
- 36 字节字符串存储开销大，随机值导致 B+Tree 页分裂，写入性能差

### 1.2 数据库自增 ID

MySQL `AUTO_INCREMENT` 主键，单表内严格递增且连续。

- 每次发号伴随磁盘写（INSERT/UPDATE），单机上限约 1000-3000 TPS
- 强依赖单点数据库，宕机即停；分库后各库独立递增，全局不连续

### 1.3 Snowflake（雪花算法）

64 位 = 1 bit 符号 + 41 bit 时间戳 + 10 bit 机器 ID + 12 bit 序列号。纯内存计算，单机 400 万+/秒。

- 趋势递增但不连续，两个 ID 的差值无业务含义
- 依赖机器时钟，时钟回拨可能导致 ID 重复

### 1.4 Leaf-Segment（美团 Leaf 号段模式）

从数据库批量预取号段（如 1000 个），应用内存中顺序分发。当前号段消耗到阈值时异步预加载下一号段，避免切换时阻塞。

- 号段内连续，但进程重启时未消费的号段被浪费，产生空洞
- 多实例部署时各进程持有不同号段，同一业务维度内 ID 交叉，无法保证连续

### 1.5 Leaf-Snowflake

Snowflake 变体，用 ZooKeeper 管理 workerID 并解决时钟回拨。本质仍是 Snowflake，不连续的缺陷不变。

### 1.6 Redis INCR

对 Redis Key 执行 `INCR` / `INCRBY` 原子递增。严格递增且连续，单机 10 万+ QPS。

- 依赖 Redis 可用性；主从异步复制下，切换后可能 Seq 回退导致重复

### 1.7 Redis + MySQL 号段（混合方案）

MySQL 持久化号段上界，Redis 在号段范围内高速分发。

```
业务服务 ──INCRBY──> Redis（号段内分发）──号段申请──> MySQL（持久化上界）
```

```sql
CREATE TABLE seq_segment (
    conversation_id VARCHAR(128) PRIMARY KEY,
    max_seq BIGINT NOT NULL DEFAULT 0
);
-- 申请号段：UPDATE seq_segment SET max_seq = max_seq + 1000 WHERE conversation_id = ?
-- Redis 在 [old_max+1, old_max+1000] 范围内通过 INCRBY 分发
-- 当前号段消耗到阈值时异步向 MySQL 预加载下一号段
```

- 正常运行时严格连续；Redis 故障切换时未用完的号段被跳过，可能产生空洞，但**绝不重复**
- 99.9% 请求走 Redis，MySQL 调用频率 = QPS / 号段大小

---

## 2. 对比总结

| 方案                 | 唯一 | 有序 | 连续 | 性能 | 故障安全（不重复） | 外部依赖      |
| -------------------- | :--: | :--: | :--: | :--: | :----------------: | ------------- |
| UUID                 |  ✅   |  ❌   |  ❌   |  ✅   |         ✅          | 无            |
| DB 自增              |  ✅   |  ✅   |  ✅   |  ❌   |         ✅          | MySQL         |
| Snowflake            |  ✅   |  ⚠️   |  ❌   |  ✅   |         ⚠️          | 时钟          |
| Leaf-Segment         |  ✅   |  ✅   |  ⚠️   |  ✅   |         ✅          | MySQL         |
| Leaf-Snowflake       |  ✅   |  ⚠️   |  ❌   |  ✅   |         ✅          | MySQL + ZK    |
| Redis INCR           |  ✅   |  ✅   |  ✅   |  ✅   |         ⚠️          | Redis         |
| **Redis+MySQL 号段** |  ✅   |  ✅   |  ✅   |  ✅   |         ✅          | Redis + MySQL |

---

## 3. IM 场景选型：会话级 Redis + MySQL 号段

### 3.1 业务约束

IM 消息标识符需要满足以下需求：

| 需求     | 原因                                                         |
| -------- | ------------------------------------------------------------ |
| 严格连续 | 增量同步依赖 `serverMaxSeq - clientLastSeq = 差量条数`；未读数依赖 `maxSeq - readSeq` |
| 不可重复 | 重复 Seq 导致增量同步和未读计数逻辑错乱（数据库主键为雪花 ID，消息通过 clientMsgID 去重，Seq 仅用于排序与同步） |
| 高性能   | 万级 QPS 消息写入，发号不能成为瓶颈                          |

优先级：**故障安全（不重复） > 连续性（隐含有序性） > 性能**

连续性是比有序性更强的约束——严格连续天然保证有序，反之不成立（有序但不连续无法做差值计算）。

### 3.2 为什么是"会话级"

全局 Seq 所有会话共享一个计数器，单会话内必然产生空洞（被其他会话占用），违反连续性。会话级 Seq 每个 `conversationID` 独立计数，天然隔离。

此外，会话级 Seq 将压力分散到成千上万个独立 Key，消除全局单 Key 热点。在 Redis Cluster 下自动分布到不同 Slot，实现水平扩展。

### 3.3 批量预分配

消息经过 Batcher 聚合后，使用 `INCRBY` 一次从 Redis 申请一批 Seq，Redis 调用从 O(N)降为 O(1)
```go
maxSeq, _ := db.seqConversation.Malloc(ctx, conversationID, int64(len(msgs)))
for i, msg := range msgs {
    msg.Seq = maxSeq + int64(i) + 1
}
```

当 Redis 中的值逼近当前号段上界时，触发异步向 MySQL 申请下一号段。

### 3.4 故障与自愈

**Redis 宕机时**：未用完的号段被跳过（如 [901-1000] 只用到 950，恢复后从 MySQL 读取上界 1000，申请新号段 [1001-2000]），产生空洞但不重复。

**空洞对未读数的影响**：`maxSeq(1003) - readSeq(900) = 103`，实际只有 3 条新消息，未读数临时偏大。但用户点进会话拉取消息后，readSeq 直接更新到实际最新消息的 Seq（1003），空洞被自然跳过。IM 用户的阅读行为本身即是校准机制，无需额外定时任务。

### 3.5 风险缓解

| 风险             | 措施                                                         |
| ---------------- | ------------------------------------------------------------ |
| MySQL 短暂不可用 | 当前号段消耗到阈值（如 80%）时异步预加载下一号段，确保切换时已有备用号段 |
| 号段大小选择     | 过大则故障时跳号多，过小则频繁访问 DB。根据业务消息量取平衡值 |

---

## 参考资料

1. [Leaf——美团点评分布式ID生成系统](https://tech.meituan.com/2017/04/21/mt-leaf.html) — Leaf 核心架构原理，号段模式双 Buffer 机制与雪花模式的底层实现
2. [Leaf：美团分布式ID生成服务开源](https://tech.meituan.com/2019/03/07/open-source-project-leaf.html) — Leaf 开源版本特性、高可用容灾部署与 MySQL 半同步复制方案
3. [Announcing Snowflake](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake) — Twitter 雪花算法原始设计
4. [微信如何生成连续单调递增的消息序号](https://www.infoq.cn/article/wechat-serial-number-generator-architecture) — 微信 IM 场景下消息序号生成架构与连续性保障实践