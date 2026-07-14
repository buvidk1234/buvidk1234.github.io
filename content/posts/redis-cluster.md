+++
date = '2026-07-14T00:32:01+08:00'
draft = false
title = 'Redis Cluster'
tags = ['Redis']
categories = ['数据库']
+++

# Redis Cluster：从 Sentinel 到分片与故障转移

Redis 高可用有两个常见答案：Sentinel 和 Cluster。它们并不是同一套方案的两个名字。

Sentinel 解决的是一组主从节点如何自动切主；Cluster 除了故障转移，还要把数据和请求分散到多个 master。理解 Cluster，只需要先抓住一条主线：

```text
key -> hash slot -> master -> 执行命令
                    |
                    +-> replica 复制并在故障后接管
```

下文先完成选型，再沿着一次请求看 slot 路由，最后解释迁移和故障转移。

---

## 1. Sentinel 和 Cluster 怎么选

Sentinel（哨兵模式）由两部分组成：

```text
Redis 数据节点：1 个 master + 若干 replica，彼此保存同一份完整数据
Sentinel 进程：监控 master，协商故障状态，选择并提升 replica
```

Sentinel 进程不保存业务数据，也不负责分片。切主以后，客户端需要发现新 master 并重新连接。它提高了可用性，但单个复制组的容量和写入吞吐仍受一个 master 限制。

所谓“哨兵集群”，通常是用多个 Sentinel 共同监控主从复制组，常见部署是 3 个。达到配置的 quorum 后，master 才会被判为客观下线；真正发起故障转移还需要获得 Sentinel 多数授权。

Cluster 则把数据拆到多个 master：

```text
master A + replica A：负责一部分 slot
master B + replica B：负责一部分 slot
master C + replica C：负责一部分 slot
```

每个 replica 只复制自己 master 的数据，不是全量集群数据。Cluster 节点通过 cluster bus 协调拓扑和故障转移，不需要再部署 Sentinel。

| 维度 | Sentinel | Redis Cluster |
|-|-|-|
| 数据分布 | 同一主从复制组内保存同一份完整数据 | 16384 个 slot 分散到多个 master |
| 写入入口 | 每个复制组一个 master | 多个 master |
| 容量上限 | 单个复制组受一个 master 限制 | 多个 master 容量之和 |
| 故障协调 | 多个独立 Sentinel 进程 | Cluster 节点，master 形成多数派 |
| 客户端视图 | 每个复制组的当前 master 地址 | `slot -> master` 拓扑 |
| 多 key 操作 | 没有跨 slot 问题 | key 通常必须位于同一 slot |

因此，选择标准很直接：

- 数据和写入压力能由一台 Redis 承担，只需要自动切主，优先 Sentinel。
- 数据量或写入吞吐需要横向扩展，选择 Cluster，并接受分片带来的 key 设计和运维成本。

Sentinel 只解决高可用，Cluster 同时解决分片和高可用。下面只讨论 Cluster。

---

## 2. key 如何找到 master

### 2.1 先算 slot，再找节点

Redis Cluster 没有直接维护 `key -> node` 映射，而是增加一层固定的 hash slot：

```text
slot = CRC16(hash_input) & 0x3FFF
```

源码把 slot ID 固定为 14 bit：

```c
/* src/cluster.h */
#define CLUSTER_SLOT_MASK_BITS 14
#define CLUSTER_SLOTS (1 << CLUSTER_SLOT_MASK_BITS)  // 16384
#define CLUSTER_SLOT_MASK ((unsigned long long)(CLUSTER_SLOTS - 1))
```

所以 slot 范围是 `0 ~ 16383`。每个 master 可以负责若干段 slot，例如：

```text
master A:     0 ~ 5460
master B:  5461 ~ 10922
master C: 10923 ~ 16383
```

slot 是 key 和节点之间的稳定中间层。扩容时只需把一段 slot 及其中的数据迁给新节点，不需要为每个 key 保存独立的节点映射。

为什么固定为 16384？这首先是 Cluster 协议的设计选择，不是根据节点数动态计算出来的。14 bit 提供了足够细的迁移粒度，同时一个节点的 slot 位图只有 `16384 / 8 = 2048` 字节，可以随 cluster bus 消息传播。可以把它理解为迁移粒度和拓扑传播成本之间的固定折中。

### 2.2 hash tag 让相关 key 进入同一 slot

普通 key 使用完整内容计算 CRC16。如果第一个 `{` 与其后的第一个 `}` 之间非空，则只使用这部分内容；否则仍使用完整 key。Redis 不会跳过空的 `{}` 再寻找下一组花括号。

```c
/* src/cluster.h:keyHashSlot，省略循环体的注释 */
static inline unsigned int keyHashSlot(const char *key, int keylen) {
    int s, e;

    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;
    if (s == keylen) return crc16(key,keylen) & 0x3FFF;

    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;
    if (e == keylen || e == s+1)
        return crc16(key,keylen) & 0x3FFF;

    return crc16(key+s+1,e-s-1) & 0x3FFF;
}
```

例如：

| key | CRC16 的输入 | slot |
|-|-|-:|
| `cart:9` | `cart:9` | `1156` |
| `user:{42}:name` | `42` | `8000` |
| `user:{42}:orders` | `42` | `8000` |

hash tag 的主要用途是让必须一起操作的 key 落到同一 slot：

```bash
MGET user:{42}:name user:{42}:orders
```

不要给大量无关 key 使用同一个 tag。它们会集中到同一个 slot，分片还在，负载却没有被分散。

---

## 3. 一次请求如何路由

### 3.1 客户端先按本地拓扑直发

支持 Cluster 的客户端通常会：

1. 通过 `CLUSTER SHARDS` 获取 slot、master 和 replica 拓扑。
2. 在本地计算 key 的 slot，直接把命令发给对应 master。
3. 收到 `MOVED` 后修正或刷新拓扑，再重试命令。

`CLUSTER SLOTS` 仍被大量旧客户端使用，但从 Redis 7.0 起已标记为废弃，由 `CLUSTER SHARDS` 取代。

客户端缓存只是性能优化，服务端不会盲目信任它。每个 Cluster 节点都有一份通过 cluster bus 最终收敛的本地视图，其中记录 `slot -> owner` 以及正在迁入、迁出的 slot。这不是由中心节点提供的强一致路由表。

命令执行前，`processCommand` 会调用 `getNodeByQuery` 做服务端校验。稳定状态下的主干是：

```text
提取命令涉及的 key
  -> 计算每个 key 的 slot
  -> slot 不同：CROSSSLOT
  -> 查本地 slot owner
  -> owner 是自己：执行命令
  -> owner 是其他节点：MOVED
```

### 3.2 MOVED 修正缓存拓扑

假设 `user:{42}:name` 的 slot `8000` 归 master B，客户端却把命令发给 master A：

```text
GET user:{42}:name
  -> keyHashSlot("42") = 8000
  -> A 的本地视图：slot 8000 的 owner 是 B
  -> A 不执行 GET
  -> 返回 -MOVED 8000 10.0.0.2:6379
```

客户端应根据 `MOVED` 更新 `slot 8000 -> B`，然后把命令重试到 B。与下一节只生效一次的 `ASK` 不同，`MOVED` 可以修正长期缓存；未来再次迁移或故障切换时，owner 仍会改变。

### 3.3 CROSSSLOT 是请求本身不成立

下面两个 key 分别位于 slot `1156` 和 `8000`：

```bash
MGET cart:9 user:{42}:name
```

Redis 不会为了普通多 key 命令跨 master 发起分布式事务，因此返回：

```text
-CROSSSLOT Keys in request don't hash to the same slot
```

`MGET`、`MSET`、事务和 Lua 脚本等需要同时访问多个 key 时，应在建模阶段用 hash tag 把相关 key 放进同一 slot。代价是这些 key 无法并行分摊到多个 master；slot 的 owner 仍可能在迁移或故障切换时改变。

`MOVED` 是路由信号，`CROSSSLOT` 是请求约束。迁移期间还会出现 `ASK` 和 `TRYAGAIN`，下面结合迁移过程解释。

---

## 4. slot 迁移为什么需要 ASK

扩容、缩容和 rebalance 最终都要移动 slot。传统的 `CLUSTER SETSLOT` 路径会先分批迁移 key，再切换 slot owner，因此“数据在哪里”和“slot 归谁”会短暂不同。

假设 slot `8000` 从 A 迁到 B：

```text
1. A 标记 slot 8000 为 MIGRATING，目标是 B
2. B 标记 slot 8000 为 IMPORTING，来源是 A
3. 通过 MIGRATE 把 slot 中的 key 从 A 分批搬到 B
4. key 搬完后，把 slot 8000 的 owner 切换为 B
5. 新配置通过 cluster bus 传播
```

迁移期间，A 仍是 owner，但一部分 key 已经在 B：

| 状态 | A 如何响应 |
|-|-|
| A 本地存在请求涉及的 key | A 直接执行 |
| A 本地不存在 key，可能已迁走或原本就不存在 | A 返回 `ASK 8000 B` |
| owner 已切换到 B | A 返回 `MOVED 8000 B` |

`ASK` 的意思是“只对这一次请求去 B”，不是“以后都去 B”。客户端收到它后，要在同一连接上连续发送：

```text
ASKING
GET user:{42}:name
```

`ASKING` 只授权紧随其后的命令。B 此时还处于 `IMPORTING`，没有这个标记就会根据旧 owner 把客户端重定向回 A。客户端也不能因为 `ASK` 更新长期 slot 缓存，因为其他尚未迁移的 key 仍在 A。

如果一个多 key 请求在迁移时只有部分 key 已搬走，任何单个节点都无法完整执行，Redis 会返回 `TRYAGAIN`。这也是在线迁移比稳定路由多出的复杂度。

将常见返回合在一起：

| 返回 | 含义 | 客户端行为 |
|-|-|-|
| `MOVED` | 按当前 slot 配置，owner 是另一个节点 | 更新拓扑并重试 |
| `ASK` | slot 正在迁移，这一次去目标节点尝试 | 先发 `ASKING`，再发原命令，不更新长期拓扑 |
| `CROSSSLOT` | 一个命令涉及不同 slot | 修改 key 设计或拆分命令 |
| `TRYAGAIN` | 迁移期间暂时无法安全处理 | 退避后重试 |
| `CLUSTERDOWN` | 集群状态或 slot 覆盖不满足服务条件 | 等待恢复并检查集群状态 |

### 4.1 Redis 8.4 的原子 slot 迁移

Redis 8.4 还引入了原子 slot 迁移，入口是：

```bash
CLUSTER MIGRATION IMPORT <start-slot> <end-slot> [...]
```

`src/cluster_asm.c` 中的流程可以概括为：目标节点发起任务，源节点发送 slot 快照并继续转发增量写；目标追平后，源节点短暂停止这些 slot 的写入，目标应用最后的增量并接管 slot；新配置通过 cluster bus 发布，源节点最后清理旧数据。

它减少了传统 `SETSLOT + MIGRATE` 暴露给请求路径的中间状态，但没有改变最重要的客户端模型：客户端仍缓存 slot owner，并在拓扑变化时根据服务端反馈刷新。ASM 只影响 slot 迁移过程，不改变普通请求的路由规则。

---

## 5. master 故障后谁接管 slot

Cluster 没有独立 Sentinel。节点之间使用 cluster bus 发送 PING、PONG 和 gossip，既传播拓扑，也传播故障判断。

故障检测分两步：

```text
PFAIL：等待 PONG 或其他 cluster bus 数据超过 cluster-node-timeout，主观怀疑对方故障
FAIL：本地已是 PFAIL，且近期 master 故障报告达到多数
```

从 `PFAIL` 升级为 `FAIL` 的核心判断是：

```c
/* src/cluster_legacy.c:markNodeAsFailingIfNeeded，省略日志和持久化 */
int failures;
int needed_quorum = (server.cluster->size / 2) + 1;

if (!nodeTimedOut(node)) return;
if (nodeFailed(node)) return;

failures = clusterNodeFailureReportsCount(node);
if (clusterNodeIsMaster(myself)) failures++;
if (failures < needed_quorum) return;

node->flags &= ~CLUSTER_NODE_PFAIL;
node->flags |= CLUSTER_NODE_FAIL;
clusterSendFail(node->name);
```

这里的多数派降低了单点网络抖动造成误切换的概率，也意味着少数派分区不能自行提升 replica。系统不能同时保证任意网络分区下的写入可用和单一 owner。

下面讨论自动故障转移。master 进入 `FAIL` 后，它的 replica 才会尝试接管：

```text
replica 检查复制数据是否足够新
  -> 根据复制进度安排竞选时机
  -> 增加 currentEpoch，向其他 master 请求投票
  -> 获得多数票
  -> replica 变为 master
  -> 接管旧 master 的全部 slot
  -> 广播新配置
```

`configEpoch` 相当于这次 slot 配置的版本。新 master 使用更高 epoch 声明所有权，其他节点据此收敛到新拓扑。故障期间，客户端可能先遇到连接失败，需要连接其他可达节点或刷新拓扑；请求发到可达但不是 owner 的节点时，再通过 `MOVED` 修正缓存。手工 `CLUSTER FAILOVER` 还有其他授权路径，不属于这里的自动切换主线。

它与 Sentinel 的 `SDOWN -> ODOWN -> 提升 replica` 只有概念上的相似，参与判断的节点和协议都不同。

故障转移还有几个边界：

- 发生故障的 master 必须有可用且数据足够新的 replica，否则对应 slot 无法自动恢复。
- Redis 复制是异步的，已向客户端确认但尚未复制的写入，在故障切换时仍可能丢失。
- 默认 `cluster-require-full-coverage yes`，任一 slot 无 owner 或 owner 为 `FAIL` 时，集群会进入 `CLUSTER_FAIL`，直到拓扑恢复完整覆盖。
- replica 主要用于故障接管；用 `READONLY` 从 replica 读时，需要接受复制延迟和陈旧数据。

生产环境通常至少部署 3 个 master 形成多数派，并给每个 master 配置 replica。常见起点因此是 3 主 3 从，而不是 3 个节点。

---

## 源码索引

本文只截取了支撑主线的代码。继续追源码时，可以从以下入口进入：

| 问题 | 源码入口 |
|-|-|
| Sentinel 的主观/客观下线 | `src/sentinel.c:sentinelCheckSubjectivelyDown`、`sentinelCheckObjectivelyDown` |
| slot 和 hash tag | `src/cluster.h:keyHashSlot` |
| 命令执行前路由 | `src/server.c:processCommand` |
| owner、MOVED、ASK、CROSSSLOT | `src/cluster.c:getNodeByQuery`、`clusterRedirectClient` |
| 节点与 slot 视图 | `src/cluster_legacy.h:clusterState` |
| 传统 slot 迁移 | `src/cluster_legacy.c:clusterCommandSpecial` |
| Redis 8.4 原子迁移 | `src/cluster_asm.c` |
| gossip 和故障确认 | `src/cluster_legacy.c:clusterCron`、`markNodeAsFailingIfNeeded` |
| replica 竞选与接管 | `src/cluster_legacy.c:clusterHandleSlaveFailover`、`clusterFailoverReplaceYourMaster` |

---

## 总结

Redis Cluster 可以收束为五句话：

1. Sentinel 让一份完整数据自动切主；Cluster 把 16384 个 slot 分给多个 master，并为每个分片提供故障接管。
2. 客户端通过 `CLUSTER SHARDS`（旧客户端也可能使用 `CLUSTER SLOTS`）缓存拓扑，计算 `CRC16(key or hash-tag) & 0x3FFF` 后直达 owner。
3. 服务端在执行命令前再次校验：owner 不在当前节点时返回 `MOVED`，跨 slot 请求返回 `CROSSSLOT`。
4. 传统迁移先搬 key 再切 owner，所以需要只对单次请求生效的 `ASK`；Redis 8.4 的原子迁移缩短了这段中间状态。
5. 节点通过 cluster bus 从 `PFAIL` 达成 `FAIL` 多数判断，replica 获得多数票后接管旧 master 的 slot。

沿着 `key -> slot -> owner` 这条线理解请求，再把迁移和故障看成 owner 发生变化，Cluster 的路由、重试和高可用就能放进同一个模型。
