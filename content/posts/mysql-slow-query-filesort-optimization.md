+++
date = '2025-12-28T22:27:08+08:00'
draft = false
title = 'Mysql Slow Query Filesort Optimization'
+++

# IM 消息拉取接口的 SQL 优化记录

### 1. 背景与问题
在开发 IM 系统的“拉取历史消息”功能时，我发现当单张消息表（`messages`）的数据量达到百万级，且某个热点会话（`conversation_id`）拥有超过 10 万条消息时，接口响应出现明显卡顿。

**环境信息：**
*   数据库：MySQL 8.0+ (InnoDB)
*   数据量：单表 100 万行，大会话 10 万条信息
*   场景：用户查看最新的 20 条消息（Top-N 查询）

**慢查询 SQL：**
```sql
SELECT * FROM messages 
WHERE conversation_id = 'chat_hot' 
ORDER BY seq DESC 
LIMIT 20;
```

### 2. 排查过程

#### 2.1 慢日志抓取
开启 MySQL 慢查询日志（`log_output=FILE`, `long_query_time=0.1`），捕获到该语句的执行情况：

```text
# Query_time: 0.148528  Lock_time: 0.000003  Rows_sent: 20  Rows_examined: 151676
```
**异常点**：为了返回 20 条数据，MySQL 实际扫描了 15 万行数据。扫描/返回比极低，说明索引效率存在严重问题。

#### 2.2 执行计划分析 (EXPLAIN)
执行 `EXPLAIN` 查看当前索引使用情况：
![sql_optimization_before_explain](/images/sql_optimization_before_explain.png)

**分析结论**：
当前表只有单列索引 `idx_conversation`。
1.  MySQL 虽然能通过索引快速定位到该会话的所有消息（15万条）。
2.  但由于索引中没有包含排序字段 `seq`，MySQL 必须将这 15 万条数据加载到 Server 层的内存（Sort Buffer）中进行**文件排序（Filesort）**，如果Sort Buffer不够大还会用到磁盘。
3.  排完序后，再截取前 20 条。这一步消耗了大量的 CPU 和内存带宽。

### 3. 优化方案

为了消除 `Using filesort`，需要利用 B+ 树索引的**天然有序性**。

**方案**：建立联合索引 `(conversation_id, seq)`。

**原理**：
根据 B+ 树结构，在联合索引中，数据首先按照 `conversation_id` 排序；在 `conversation_id` 相同的情况下，数据严格按照 `seq` 排序。
这样 MySQL 只需要定位到该会话的最后一条记录，然后利用双向链表向前扫描 20 条即可。

**操作步骤**：

```sql
-- 1. 创建联合唯一索引 (seq 在会话内不重复，故使用 UNIQUE)
CREATE UNIQUE INDEX idx_conv_seq ON messages (conversation_id, seq);

-- 2. 删除原有的冗余单列索引 (遵循最左前缀原则，新索引已覆盖旧索引功能)
DROP INDEX idx_conversation ON messages;
```

### 4. 验证与结果

#### 4.1 性能数据对比
再次执行业务 SQL，耗时变化如下：
*   **优化前**：~150 ms
*   **优化后**：~0.08 ms
*   **提升幅度**：约 2000 倍
![sql_optimization_before_explain](/images/sql_optimization_before_explain.png)
#### 4.2 深入验证 (EXPLAIN ANALYZE)
为了确认 MySQL 确实只扫描了 20 行（而不是 EXPLAIN 估算的 15 万行），使用 `EXPLAIN ANALYZE` 查看实际执行路径：
![sql_optimization_after_explain_analyze](/images/sql_optimization_after_explain_analyze.png)
```text
-> Limit: 20 row(s)  (actual time=0.046..0.079 rows=20 loops=1)
    -> Index scan on messages using idx_conv_seq (backward)  
       (actual time=0.031..0.063 rows=20 loops=1)
```

**验证点**：
1.  **`Index scan ... (backward)`**：使用了索引的反向扫描，对应 SQL 中的 `ORDER BY DESC`。
2.  **`rows=20`**：实际扫描行数严格等于 Limit 行数。
3.  **`actual time`**：底层索引查找耗时仅 0.06ms，完全消除了排序开销。

### 5. 总结与反思

1.  **关于 Filesort**：在“查询+排序+分页”的高频场景下，`Using filesort` 是性能杀手。必须确保 `ORDER BY` 的字段包含在联合索引中。
2.  **关于 EXPLAIN**：普通的 `EXPLAIN` `rows` 字段只是统计估算值，在 Limit 查询中往往不准。如果要验证优化效果，推荐使用 MySQL 8.0 的 `EXPLAIN ANALYZE` 查看真实扫描行数。
3.  **关于生产变更**：虽然本地是直接执行 DDL，但在生产环境面对大表（如过亿行）加索引时，应使用 `gh-ost` 或 `pt-online-schema-change` 等工具，避免锁表引发故障。
4.  **索引治理**：加了新索引后，务必清理被覆盖的旧索引，避免写放大和空间浪费。