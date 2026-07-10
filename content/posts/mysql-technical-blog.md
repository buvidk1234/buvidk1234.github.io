+++
date = '2026-01-15T02:27:08+08:00'
draft = false
title = 'Mysql Technical Blog'
tags = ['MySQL']
+++

# MySQL 技术要点

## 一、MySQL 架构

MySQL 采用分层架构设计，主要分为 **Server 层** 和 **存储引擎层**。

### 1.1 Server 层

| 组件 | 职责 |
|------|------|
| **连接器** | 管理客户端连接、身份认证、权限校验 |
| **分析器** | 词法分析（识别关键字、表名、列名）、语法分析（构建语法树） |
| **优化器** | 生成执行计划、选择最优索引、决定 JOIN 顺序 |
| **执行器** | 调用存储引擎 API 执行查询，返回结果集 |

### 1.2 存储引擎层

MySQL 支持插件式存储引擎，常用引擎对比：

| 引擎 | 事务支持 | 锁粒度 | 适用场景 |
|------|----------|--------|----------|
| **InnoDB** | ✓ | 行锁 | OLTP、高并发读写 |
| **MyISAM** | ✗ | 表锁 | 只读或读多写少 |
| **Memory** | ✗ | 表锁 | 临时表、缓存 |

**InnoDB architecture**

![Innodb-architecture](/images/innodb-architecture-8-0.png)

---

## 二、索引机制

### 2.1 索引数据结构

| 类型 | 特点 | 适用场景 |
|------|------|----------|
| **B+ Tree** | 有序、支持范围查询、树高稳定 | 主键索引、普通索引 |
| **Hash** | O(1) 查找、不支持范围查询 | 等值查询（Memory 引擎） |
| **Full-Text** | 倒排索引、支持自然语言搜索 | 全文检索 |

### 2.2 索引分类

**按物理存储：**
- **聚簇索引（Clustered Index）**：叶子节点存储完整行数据，InnoDB 主键索引即聚簇索引
- **二级索引（Secondary Index）**：叶子节点存储主键值，查询需回表

**按字段特性：**
- **主键索引**：唯一且非空，一张表只能有一个
- **唯一索引**：字段值唯一，允许 NULL
- **普通索引**：无唯一性约束
- **前缀索引**：对字符串前 N 个字符建立索引，节省空间

**按字段数量：**
- **单列索引**：单个字段
- **联合索引**：多个字段组合，遵循最左前缀原则

### 2.3 普通索引 vs 唯一索引

- 普通索引可利用 **Change Buffer** 优化写入，数据不在内存时直接写入 Change Buffer
- 唯一索引必须读取数据页判断唯一性，无法使用 Change Buffer
- 写入 Change buffer, 避免加载冷数据（按页加载，即使修改一个数据，也至少加载16kb）挤出热点数据，造成**Buffer Pool Pollution**

### 2.4 索引进阶概念

**最左前缀原则：**

联合索引按字段顺序构建，例如 `(a, b, c)` 会先按 `a` 排序，再在 `a` 相同的情况下按 `b` 排序，最后按 `c` 排序。因此能命中索引的典型条件包括：

- `WHERE a = ?`
- `WHERE a = ? AND b = ?`
- `WHERE a = ? AND b = ? AND c = ?`

如果跳过最左列，例如 `WHERE b = ?`，通常无法完整利用 `(a, b, c)` 联合索引。范围查询会影响后续字段继续用于有序定位，例如 `WHERE a = ? AND b > ? AND c = ?` 中，`c` 通常不能继续用于索引定位。

**回表：**

InnoDB 的二级索引叶子节点存储的是主键值，不保存完整行数据。通过二级索引找到主键后，再回到聚簇索引查询完整行数据，这个过程叫回表。

```sql
-- idx_name(name)
SELECT * FROM user WHERE name = 'Tom';
```

上面语句先通过 `idx_name` 找到主键，再根据主键回到聚簇索引取整行数据。

**覆盖索引：**

如果查询所需字段都在同一个索引中，存储引擎可以直接从索引返回结果，不需要回表，这就是覆盖索引。

```sql
-- idx_name_age(name, age)
SELECT name, age FROM user WHERE name = 'Tom';
```

`EXPLAIN` 的 `Extra` 出现 `Using index`，通常表示使用了覆盖索引。

**索引下推（Index Condition Pushdown，ICP）：**

索引下推会把部分 `WHERE` 条件下推到存储引擎层，在扫描二级索引时先判断能否过滤，再决定是否回表，减少无效回表次数。

```sql
-- idx_name_age(name, age)
SELECT * FROM user WHERE name LIKE 'Tom%' AND age = 18;
```

没有索引下推时，可能先根据 `name` 找到一批主键再逐行回表判断 `age`；有索引下推时，可以在二级索引中先判断 `age`，只对满足条件的记录回表。`EXPLAIN` 的 `Extra` 出现 `Using index condition`，通常表示使用了索引下推。

**常见索引失效场景：**

- 对索引列使用函数或表达式：`WHERE YEAR(create_time) = 2026`
- 字符串字段发生隐式类型转换：`WHERE phone = 13800138000`
- `LIKE` 以通配符开头：`WHERE name LIKE '%Tom'`
- 联合索引不满足最左前缀：`idx(a, b)` 下只查询 `WHERE b = ?`
- `OR` 两边不是都能使用索引，可能导致全表扫描

### 2.5 优化器选错索引的处理

```sql
-- 重新统计索引信息
ANALYZE TABLE t;

-- 强制使用指定索引
SELECT * FROM t FORCE INDEX(idx_name) WHERE name = 'test';
```

---

## 三、事务

### 3.1 ACID 特性

| 特性 | 含义 | 实现机制 |
|------|------|----------|
| **Atomicity（原子性）** | 事务要么全部成功，要么全部回滚 | Undo Log |
| **Consistency（一致性）** | 事务前后数据库状态一致 | 由其他三个特性共同保证 |
| **Isolation（隔离性）** | 并发事务互不干扰 | MVCC + 锁 |
| **Durability（持久性）** | 已提交事务永久生效 | Redo Log |

### 3.2 隔离级别

| 隔离级别 | 存在问题 | 实现原理 |
|----------|----------|----------|
| Read Uncommitted（读未提交） | 脏读 | 直接读最新版本，可能读到未提交数据 |
| Read Committed（读已提交） | 不可重复读 | 每次一致性读生成新的 Read View |
| Repeatable Read（可重复读，默认） | SQL 标准下可能幻读 | 普通一致性读复用事务内首次 Read View；当前读通过锁处理范围并发 |
| Serializable（串行化） | 并发度最低 | `autocommit=0` 时普通 `SELECT` 隐式转为 `SELECT ... FOR SHARE` |

> InnoDB 的 RR 需要区分普通一致性读和当前读：普通 `SELECT` 读事务快照；`FOR UPDATE`、`FOR SHARE`、`UPDATE`、`DELETE` 读取最新可见版本并按索引范围加锁。

### 3.3 MVCC（多版本并发控制）

MVCC 通过为每行数据维护多个版本，实现非阻塞的一致性读：

- **隐藏列**：`DB_TRX_ID`、`DB_ROLL_PTR`、`DB_ROW_ID`
- **Read View**：记录活跃事务列表，决定一致性读的数据可见性（RC 每次一致性读创建，RR 事务内首次一致性读创建；当前读不走 Read View）
- **版本链**：通过 Undo Log 构建历史版本链表

### 3.4 当前读与快照读

```sql
-- 快照读：读取 MVCC 快照版本
SELECT * FROM t WHERE id = 1;

-- 当前读：读取最新版本并加锁
SELECT * FROM t WHERE id = 1 FOR UPDATE;        -- 排他锁
SELECT * FROM t WHERE id = 1 LOCK IN SHARE MODE; -- 共享锁
```

---

## 四、锁机制

锁的重点不在分类，而在生产中什么时候会把库、表、热点行卡住。

### 4.1 全局锁：FTWRL 与一致性备份

`FLUSH TABLES WITH READ LOCK` 会给整个实例加全局读锁，阻塞数据更新和表结构变更，通常只应该短时间持有：

```sql
FLUSH TABLES WITH READ LOCK;
-- 执行备份或获取一致性位点
UNLOCK TABLES;
```

如果表全部使用 InnoDB，逻辑备份优先使用事务快照：

```bash
mysqldump --single-transaction ...
```

`--single-transaction` 会在事务中创建一致性 Read View，备份期间可以继续写入；如果库里有 MyISAM 这类非事务表，Read View 无法保证一致性，就需要 FTWRL 这类全局读锁兜底。

**FTWRL 与 `SET GLOBAL read_only = true` 的区别：**

- FTWRL 是会话级锁，客户端异常断开后会自动释放；`read_only` 是全局变量，异常后不会自动恢复。
- `read_only` 常被用于主从切换、只读实例标识，不适合临时备份流程随意改动。
- 具有高权限的账号可能绕过 `read_only` 写入，真正限制还要配合 `super_read_only`；FTWRL 更适合短时间冻结实例写入。

### 4.2 表锁与 MDL

显式表锁语法：

```sql
LOCK TABLES t READ;
LOCK TABLES t WRITE;
UNLOCK TABLES;
```

InnoDB 业务表一般不主动使用显式表锁，更常见的问题是 **MDL（Metadata Lock）**。查询、更新会自动持有 MDL 读锁，DDL 需要 MDL 写锁；如果一个长事务迟迟不提交，`ALTER TABLE` 拿不到 MDL 写锁，而后续访问这张表的请求又可能排在 DDL 后面，最终表现为“小表加字段把整张表卡住”。

典型阻塞链路：

```sql
-- session A
BEGIN;
SELECT * FROM small_t WHERE id = 1;

-- session B
ALTER TABLE small_t ADD COLUMN c INT;

-- session C
SELECT * FROM small_t WHERE id = 2;
```

**安全地给小表加字段：**

- 先确认没有长事务持有目标表的 MDL，重点看未提交事务和正在执行的慢查询。
- 给 DDL 设置较短等待时间，拿不到锁就失败，避免排队阻塞业务请求。
- 明确指定在线 DDL 能力，避免语句在不支持时静默退化为重建表。

```sql
SET SESSION lock_wait_timeout = 3;
ALTER TABLE small_t ADD COLUMN c INT DEFAULT 0, ALGORITHM=INSTANT, LOCK=NONE;
```

如果超时失败，说明当前不是安全变更窗口，应重试或先处理阻塞事务。

### 4.3 行级锁：死锁检测与热点行

InnoDB 开启死锁检测后，每个被锁阻塞的事务都会参与 wait-for graph 检查。热点行上大量事务同时更新时，即使没有真正形成死锁，也会频繁触发检测；检测复杂度接近 `O(n)`，等待事务越多，CPU 消耗越明显。

典型热点更新：

```sql
UPDATE account SET balance = balance + 1 WHERE id = 1;
UPDATE counter SET value = value + 1 WHERE name = 'pv';
```

处理思路不是简单调大连接数，而是减少同一行上的并发等待：

- 在进入 MySQL 前按业务 key 限制并发，把同一个热点资源的更新排队或合并。
- 如果客户端很多，单个客户端限流不够，需要在服务端入口、队列或中间层做统一限流。
- 结合业务做分片，把一行热点拆成多行热点，例如计数器拆成多个 bucket，写入时随机选择 bucket，读取时汇总。
- 对库存、账户这类强约束数据，不能机械拆分，需要按业务语义设计库存分桶、流水追加或异步汇总。
- `innodb_deadlock_detect=off` 可以减少检测开销，但必须依赖锁等待超时和重试兜底，通常不是首选方案。

### 4.4 行锁加锁规则

以下默认讨论 InnoDB 的 Repeatable Read 隔离级别；Read Committed 下 gap lock 基本禁用，只在外键检查、唯一键冲突检查等场景保留。

InnoDB 行锁的加锁规则：

1. 加锁基本单位是 Next-Key Lock（前开后闭区间）
2. 只对访问到的对象加锁
3. 唯一索引等值查询命中时，Next-Key Lock 退化为行锁
4. 等值查询向右遍历到不满足条件的记录时，Next-Key Lock 退化为间隙锁

> (5,10,15) \
> `select * from t where id>=10 and id<11 for update;` \
> 这个例子用来说明范围查询会扫描到右侧第一条不满足条件的记录，不能只看 SQL 里的 `id < 11`。

加锁流程：

1. 假设 `id` 是主键或唯一索引，`FOR UPDATE` 是当前读，会沿 `id` 索引扫描并加排他锁。
2. 根据 `id >= 10` 定位到第一条满足下界的记录 `id = 10`，这条记录需要返回，因此给 `id = 10` 加记录锁。
3. 继续向右扫描判断上界 `id < 11`，下一条记录是 `id = 15`，发现不满足条件，扫描结束。
4. 为防止事务期间在 `10` 和 `15` 之间插入新的记录，InnoDB 还需要锁住 `(10,15)` 这个间隙。间隙锁的粒度是两个相邻索引记录之间的 gap，所以不能只锁 `(10,11)`。
5. 核心锁范围是 `id = 10` 的记录锁 + `(10,15)` 的间隙锁，即防住对 `10` 这条记录的修改，以及在 `10` 和 `15` 之间插入新记录。
6. 右侧第一条不满足条件的记录是否被记录锁覆盖，受 MySQL 版本、执行计划、索引类型和优化器行为影响，不能脱离具体环境写死为 `[10,15)` 或 `[10,15]`。

### 4.5 幻读与间隙锁

**幻读定义**：同一事务中，后一次查询看到了前一次查询没看到的行（专指新插入的行，不包括更新和删除）。

**解决方案**：间隙锁 + 行锁 = Next-Key Lock

例如，对表 `t`（主键值为 0, 5, 10, 15, 20, 25）执行 `SELECT * FROM t FOR UPDATE`，会形成以下 Next-Key Lock：
- `(-∞, 0]`, `(0, 5]`, `(5, 10]`, `(10, 15]`, `(15, 20]`, `(20, 25]`, `(25, +supremum]`

> **注意**：`LOCK IN SHARE MODE` / `FOR SHARE` 如果使用覆盖索引且不需要回表，可能只锁扫描到的二级索引记录；如果需要回表，仍会访问并锁相关的聚簇索引记录。`FOR UPDATE` 通常会同时锁二级索引记录和对应的聚簇索引记录。

---

## 五、日志系统

### 5.1 日志类型

| 日志 | 作用 | 所属层 |
|------|------|--------|
| **Redo Log** | 保证持久性，崩溃恢复 | InnoDB 引擎层 |
| **Undo Log** | 保证原子性，支持 MVCC | InnoDB 引擎层 |
| **Binlog** | 主从复制、数据备份 | Server 层 |
| **Relay Log** | 从库接收主库 Binlog 的中继日志 | Server 层 |

### 5.2 Redo Log：WAL 与崩溃恢复

Redo Log 是 InnoDB 的物理日志，记录的是数据页上的局部修改，不是完整数据页镜像。它解决的问题是：事务提交时不必立刻把所有脏页刷到数据文件，只要先保证 redo 持久化。

这就是 WAL（Write-Ahead Logging）：先写日志，再写数据页。

核心概念：

- **Redo Log Buffer**：内存中的 redo 缓冲区，事务执行过程中先把 redo record 写到这里。
- **Redo Log File**：磁盘上的 redo 文件，顺序写，成本低于随机刷脏页。
- **LSN（Log Sequence Number）**：全局递增的日志位置，数据页也会记录 page LSN，用来判断页是否需要恢复。
- **Checkpoint**：表示 checkpoint LSN 之前的脏页修改已经刷入数据文件，这部分 redo 可以被覆盖。

崩溃恢复时，redo 不会重新执行 SQL，也不会自己生成完整数据页。InnoDB 会读取磁盘上的数据页，查看页头里的 page LSN；再扫描 redo log，把 LSN 大于 page LSN、且属于这个页的 redo record 应用到内存页上。恢复后的页后续仍然通过刷脏流程写回数据文件。

Redo Log 是循环写的。如果写入速度长期大于刷脏页速度，checkpoint 推进不及时，redo 空间接近写满时，InnoDB 会被迫加速刷脏页，严重时会影响前台写入。

### 5.3 Redo Log Buffer 刷盘时机

事务执行时，redo 先写入 Redo Log Buffer，不一定等事务提交才刷盘。常见触发时机：

- 事务提交时
- 后台线程周期性刷盘
- Redo Log Buffer 空间不足
- Checkpoint 推进需要

`innodb_flush_log_at_trx_commit` 决定提交时 redo 的持久化强度：

| 值 | 行为 | 风险 |
|----|------|------|
| `1` | 每次提交都 write + fsync redo | 最安全，性能成本最高 |
| `2` | 每次提交 write 到 OS cache，约每秒 fsync | OS 崩溃可能丢最近约 1 秒事务 |
| `0` | 约每秒 write + fsync | MySQL 崩溃也可能丢最近约 1 秒事务 |

Redo 在事务提交前被刷盘不代表事务已经提交。崩溃恢复时还要结合事务状态、undo 和两阶段提交结果判断是提交还是回滚。

### 5.4 Binlog：复制与归档

Binlog 属于 Server 层，记录逻辑变更，主要用于主从复制和按时间点恢复。

| 格式 | 特点 |
|------|------|
| **Statement** | 记录 SQL 语句，日志量小，但依赖执行上下文，可能导致主从不一致 |
| **Row** | 记录行变更，日志量大，但复制结果更稳定 |
| **Mixed** | 通常优先记录 Statement，遇到可能不安全的语句自动切换为 Row |

事务提交时，Server 层会先把 binlog 写入每个线程自己的 binlog cache，提交阶段再写入 binlog 文件。`sync_binlog` 控制 binlog 的 fsync 策略：

- `sync_binlog = 1`：每次事务提交都 fsync binlog，崩溃安全性最好。
- `sync_binlog > 1` 或 `0`：减少 fsync 次数，提升吞吐，但 OS 崩溃时可能丢失最近的 binlog。

### 5.5 Redo Log 和 Binlog 长什么样

同一条更新语句：

```sql
CREATE TABLE account (
  id BIGINT PRIMARY KEY,
  name VARCHAR(32),
  balance INT,
  KEY idx_balance(balance)
) ENGINE=InnoDB;

UPDATE account SET balance = balance - 100 WHERE id = 1;
```

**Binlog 是逻辑日志**，记录“业务上发生了什么变更”，不关心 InnoDB 数据页怎么改。

如果是 Statement 格式，类似：

```text
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

如果是 Row 格式，类似：

```text
Table_map: account -> table_id = 108
Update_rows:
  before: id = 1, name = 'Tom', balance = 1000
  after:  id = 1, name = 'Tom', balance = 900
Xid: commit
```

Row 格式也还是逻辑日志：它描述哪张表的哪一行从什么值变成什么值，而不是描述哪个数据页、哪个页内偏移被改了。

**Redo Log 是页级物理日志**，记录“InnoDB 某个数据页内部怎么改”。真实 redo 是二进制格式，下面只是示意：

```text
LSN=1200 REC_UPDATE
  space_id=32, page_no=100, index=PRIMARY, record_offset=0x2f10
  payload=把页内这条记录对应的若干字节从旧值改成新值

LSN=1230 REC_DELETE_MARK
  space_id=32, page_no=205, index=idx_balance
  payload=在页内标记旧二级索引记录为删除

LSN=1260 REC_INSERT
  space_id=32, page_no=206, index=idx_balance
  payload=在页内插入一条新的二级索引记录
```

所以两者不能互相替代：

- **Binlog** 适合复制和归档，因为它描述表级/行级逻辑变更，可以给从库重放，也可以做按时间点恢复。
- **Redo Log** 适合 crash recovery，因为它描述页级修改。崩溃后 InnoDB 看每个数据页的 page LSN，把 page LSN 之后、属于这个页的 redo record 应用到内存页。
- 只靠 binlog 做崩溃恢复会有问题：如果聚簇索引页刷盘了，二级索引页没刷盘，重新执行整条 `UPDATE` 可能重复扣减，也不知道应该只补哪个页。
- 只靠 redo 做复制和归档也不合适：redo 是 InnoDB 私有页修改日志，循环覆盖，而且不是“某张表某行改成什么”的通用逻辑变更流。

### 5.6 两阶段提交

为了保证 Redo Log 和 Binlog 的一致性，事务提交采用两阶段提交：

1. **Prepare**：InnoDB 写入 redo，并把事务标记为 prepare 状态。
2. **Write Binlog**：Server 层写入 binlog，binlog 中带有事务 XID。
3. **Commit**：InnoDB 写入 redo commit 标记，事务完成。

崩溃恢复时按状态判断：

- redo 没有 prepare：事务未完成，回滚。
- redo 是 prepare，但没有对应 binlog：回滚。
- redo 是 prepare，并且存在完整 binlog：提交。
- redo 已经 commit：提交。

因此 binlog 和 redo 的顺序不能随意交换。先写 binlog 后写 redo，可能出现从库已经复制但主库崩溃后事务不存在；先 redo commit 再写 binlog，可能出现主库有数据但从库永远收不到这条变更。

**组提交优化**：重点是把多个事务的 fsync 合并，不是把多个事务混成一条日志。

大致流程：

1. 多个事务都完成 InnoDB prepare，进入提交队列。
2. 队列里的 leader 线程把这一组事务的 binlog cache 按顺序写入 binlog 文件。
3. 如果 `sync_binlog=1`，leader 执行一次 fsync，把这一组 binlog 一起刷盘；如果 `sync_binlog=N` 或 `0`，是否立即 fsync 取决于对应策略。
4. 按 binlog 写入顺序调用 InnoDB commit，写入 redo commit 标记。

所以“一起”主要指在需要刷盘时 **共用一次 fsync**；写 binlog、提交 InnoDB 仍然保持事务顺序。Redo 本身也可以把多个事务的 redo record 在 Log Buffer 中合并刷盘，但两阶段提交里最关键的是 binlog group commit 保证 binlog 顺序和 InnoDB commit 顺序一致。

### 5.7 Doublewrite

InnoDB 使用 Doublewrite Buffer 解决部分页写入（Partial Page Write）问题：
1. 先将脏页写入 Doublewrite Buffer（顺序写）
2. 再将脏页写入数据文件（随机写）
3. 崩溃恢复时，若数据页损坏，可从 Doublewrite Buffer 恢复

> redo log：记录的是 InnoDB 页级别的物理修改，例如哪个表空间、哪个数据页、页内哪个位置发生了什么变化；它不是记录 SQL，也不是只记录“哪些页改了”。
>
> 内存与磁盘刷盘关系：事务先修改 Buffer Pool 中的数据页，数据页变成脏页；同时生成 redo record 写入 Redo Log Buffer。事务提交时先保证 redo 按配置写入或刷到磁盘 redo 文件，脏页可以之后异步刷盘。正常情况下，数据最终落盘是 Buffer Pool 把完整脏页刷到数据文件，不是 redo log 直接更新磁盘数据页。
>
> doublewrite：解决的是数据页刷盘时只写了一半的问题；redo log 解决的是脏页还没刷盘时如何恢复的问题。

---

## 六、主从复制

### 6.1 复制流程

```
主库：事务提交 → 写 Binlog
   ↓
从库 I/O 线程：读取主库 Binlog → 写入 Relay Log
   ↓
从库 SQL 线程：重放 Relay Log → 更新数据
```

### 6.2 主备延迟原因

- 从库机器性能较差
- 从库承担过多查询压力
- 大事务执行时间长(delete大量数据,大表DDL(gh-ost解决))
- 网络延迟

### 6.3 处理过期读

| 方案 | 实现复杂度 | 一致性 |
|------|------------|--------|
| 强制走主库 | 低 | 强 |
| Sleep 延迟 | 低 | 弱 |
| 判断主备无延迟 | 中 | 较强 |
| Semi-Sync 半同步 | 中 | 防丢增强，不保证读一致 |
| 等主库位点 | 高 | 强 |
| 等 GTID | 高 | 强 |

> Semi-Sync 只保证主库提交返回前，至少有一个从库接收到事务日志；不保证读请求命中的从库已经执行完这条事务。要解决过期读，仍要等目标从库追到指定 binlog 位点或 GTID。

---

## 七、查询执行分析

### 7.1 执行计划分析（EXPLAIN）

![Explain](/images/explain.png)

**Explain extra 字段解释**

- Using index: 使用了覆盖索引，避免回表查询

- Using filesort: 无法直接利用索引顺序满足 `ORDER BY`，需要额外排序，分为全字段排序和 rowid 排序

- Using index condition: 使用了索引下推（Index Condition Pushdown），利用二级索引提前过滤，减少回表次数。

  ```sql
  (age,city)
  select * from tb where age>5 and city="aaa" and name = "abc";
  -- age可以用到索引,city用到索引下推，回表之前就过滤到部分数据。
  ```

- Using join buffer: 使用 Join Buffer。老版本可能是 Block Nested Loop，开启 BKA 时可能是 Batched Key Access；MySQL 8.0.20 起无索引连接更多由 Hash Join 处理。

- Using MRR: 用到 Multi-Range Read 优化，批量回表查询，以id递增顺序回表以实现顺序读。

  ```sql
  (a)
  select * from tb where a>5 and a<10;
  -- 根据索引a得到一些结果，根据id排序，按id递增的方式回表查询
  ```

- Using temporary: 使用临时表，常见于 `UNION` 去重、`GROUP BY`、`DISTINCT`、部分排序和派生表场景

### 7.2 慢查询诊断

```sql
-- 查看当前执行的 SQL 状态
SHOW PROCESSLIST;

-- 查看行锁等待（MySQL 8.0+）
SELECT * FROM sys.innodb_lock_waits;
```

**查询一行数据也很慢：**

即使 `WHERE id = ?` 命中主键，慢查询也不一定是索引问题，可能卡在执行前或读取版本链上。

| 现象 | 典型状态 | 原因 |
|------|----------|------|
| 等 MDL | `Waiting for table metadata lock` | 表上有未提交事务持有 MDL 读锁，DDL 等 MDL 写锁，后续查询排在 DDL 后面 |
| 等 Flush | `Waiting for table flush` | 有 `FLUSH TABLES t` 等待关闭表对象，新的查询需要等待 flush 完成；如果 flush 本身被长查询阻塞，后续查询也会被连带阻塞 |
| 等行锁 | `sys.innodb_lock_waits` 可见等待关系 | 当前读或更新读到被其他事务锁住的记录；普通快照读通常不等行锁 |
| Undo 链过长 | 扫描行数少但耗时高 | 快照读需要沿 undo 版本链查找对当前 Read View 可见的历史版本，长事务会拉长版本链 |

处理思路：

- `SHOW PROCESSLIST` 先看线程状态，区分 MDL、Flush、行锁还是执行慢。
- MDL 问题优先找到未提交长事务，不要让 DDL 长时间排队挡住后续请求。
- Flush 问题要看是谁阻塞了 `FLUSH TABLES`，不是只盯着被卡住的查询。
- 行锁等待用 `sys.innodb_lock_waits` 找 blocking transaction，再决定等待、kill 或业务重试。
- Undo 链过长要处理长事务，避免长时间不提交的快照读拖住 purge。

**其他查询慢的原因：**

- 扫描行数过多
- 索引选择不当
- 客户端接收慢（MySQL 边读边发，导致 `net_buffer` 阻塞）

### 7.3 JOIN 算法

| 算法 | 条件 | 特点 |
|------|------|------|
| **Index Nested-Loop Join** | 被驱动表连接字段有索引 | 性能好，复杂度 O(NlogM) |
| **Block Nested-Loop Join** | MySQL 8.0.20 前无索引连接可能使用 | 大表 BNL 容易导致热数据淘汰 |
| **Hash Join** | MySQL 8.0.18 引入，8.0.20 起替代 BNL 处理无索引连接 | 通常比 BNL 更适合大数据量连接 |

- **MRR**：将随机 I/O 转为顺序 I/O，先读取索引排序后再回表
- **BKA**：用到 join buffer，`NLJ -> BKA`，基于 MRR 批量把驱动表 join key 传给被驱动表，再按主键顺序回表，前提是被驱动表访问路径能用上索引

### 7.4 ORDER BY 优化

**最优情况**：利用索引有序性，EXPLAIN 不出现 `Using filesort`。

**filesort 算法**（无索引时）：

| 算法 | 特点 |
|------|------|
| **全字段排序** | 所有字段放入 Sort Buffer，排序后直接返回 |
| **Rowid 排序** | 仅排序字段+主键，排序后回表，适用于宽表 |

> MySQL 8.0.20 之前会参考 `max_length_for_sort_data` 选择排序方式；8.0.20 起该变量已废弃，不再影响优化器决策。

### 7.5 外部排序

当 Sort Buffer 不足时，MySQL 使用外部排序：
- **归并排序**：分批排序写入临时文件，最后多路归并
- **优先队列**：`ORDER BY ... LIMIT N` 场景，维护 N 个元素的堆，避免全量排序

### 7.6 内存数据结构与临时表选择

- 如果语句执行过程可以一边读数据，一边直接得到结果，不需要额外内存来保存中间结果；

- **join_buffer** 保存驱动表的连接数据，**sort_buffer** 保存排序 tuple（如排序字段、rowid 或返回字段），**临时表**是二维表结构
- 如果执行逻辑需要用到二维表特性，就会优先考虑使用临时表。

---

## 八、Buffer Pool 与刷脏页

### 8.1 性能抖动原因

- Redo Log 写满，触发 Checkpoint 刷脏页
- Buffer Pool 满，淘汰脏页需先刷盘
- 后台线程定期刷脏页

### 8.2 Buffer Pool LRU 优化

InnoDB 采用改进的 LRU 算法（Young 区:Old 区 = 5:3）防止缓存污染：
1. 新数据先放入 Old 区域
2. 在 Old 区域停留超过 `innodb_old_blocks_time` 后再次访问，才移入 Young 区域
3. 防止全表扫描等操作将热点数据挤出缓存

---

## 九、常见问题与解决方案

### 9.1 删除数据不释放空间

```sql
-- DELETE 仅标记删除，空间不会立即释放
DELETE FROM t WHERE create_time < '2024-01-01';

-- 重建表释放空间
ALTER TABLE t ENGINE = InnoDB;
```

### 9.2 COUNT 性能

`COUNT(*)` 统计结果集行数，`COUNT(字段)` 只统计字段值非 `NULL` 的行。InnoDB 会优化 `COUNT(*)` 和 `COUNT(1)`，两者没有本质性能差异。

> 推荐写 `COUNT(*)`：语义最清晰，优化器也会选择可用的最小索引来减少扫描成本。

### 9.3 临时表

常见触发场景：`UNION` 去重、无合适索引的 `GROUP BY` / `DISTINCT`、复杂子查询、部分排序和派生表。

### 9.4 分区表

分区表在引擎层分区，查询语义对业务基本透明，但表设计不完全透明，例如唯一键通常需要包含分区字段。优势：`ALTER TABLE t DROP PARTITION p_2023` 比 `DELETE` 更快。

适用场景：按时间分区的日志表、历史数据表。

### 9.5 数据迁移

| 方式 | 特点 |
|------|------|
| **物理拷贝** | 速度快，仅限同引擎、同版本 |
| **mysqldump** | 通用，不支持复杂 WHERE |
| **SELECT INTO OUTFILE** | 导出 `SELECT` 结果集，可来自过滤、表达式或 JOIN；受文件权限和 `secure_file_priv` 限制 |

---

*参考资料：[MySQL 实战 45 讲](https://time.geekbang.org/column/intro/139)、《高性能 MySQL》、[从一条慢SQL说起：交易订单表如何做索引优化](https://mp.weixin.qq.com/s/sCBOvzUkX7O4fqGTVM68Uw)*
