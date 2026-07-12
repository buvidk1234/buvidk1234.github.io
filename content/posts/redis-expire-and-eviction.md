+++
date = '2026-07-12T12:42:00+08:00'
draft = false
title = 'Redis Expire and Eviction'
tags = ['Redis']
categories = ['数据库']
+++

# Redis 过期和淘汰：从 SET token abc EX 60 开始

先固定一条命令：

```text
SET token abc EX 60
```

这条命令写入了一个 key，并给它挂上 TTL。之后 `token` 会遇到两条删除路径：

| 路径 | 触发点 | 对 `token` 的影响 |
|-|-|-|
| expire 过期 | 到达 `EX 60` 对应的过期时间 | 读路径把它视为已过期，后台扫描也可能删除它 |
| eviction 淘汰 | Redis 内存超过 `maxmemory` | 按 `maxmemory-policy` 判断它是否进入淘汰候选 |

全文围绕这条 `token` 主线展开：TTL 写到哪里、到期后怎样被发现、删除时发生什么、
内存压力下又怎样被淘汰策略处理。

文中的源码摘录保留主线分支，省略 Cluster、module、统计细节和错误处理。

---

## SET token abc EX 60 写入了什么

Redis 的 DB 里有两套和 `token` 相关的索引：

```c
/* server.h，省略其他字段 */
typedef struct redisDb {
    kvstore *keys;      /* 当前 DB 的全部 key */
    kvstore *expires;   /* 设置了 TTL 的 key */
    unsigned long expires_cursor;
} redisDb;
```

执行 `SET token abc EX 60` 后，`token` 会出现在 `db->keys` 中；因为带 TTL，
它也会出现在 `db->expires` 中。两套索引指向同一个 `kvobj`，过期时间跟着
`kvobj` 走：

```text
db->keys
  token ──┐
          ├──> kvobj(key=token, value=abc, expire=<absolute time>)
db->expires
  token ──┘
```

读取 TTL 时，Redis 直接看 `kvobj` 的 metadata：

```c
/* object.c */
long long kvobjGetExpire(const kvobj *kv) {
    if (kv->metabits & KEY_META_MASK_EXPIRE) {
        return (long long)
            (*kvobjMetaRef((kvobj *)kv, KEY_META_ID_EXPIRE));
    }
    return -1;
}
```

设置 TTL 的主线在 `setGenericCommand` 里。`EX 60` 先通过
`getExpireMillisecondsOrReply` 转成绝对过期时间，再写入 `kvobj`，同时把这个 key
加入 `db->expires`：

```c
/* t_string.c:setGenericCommand，省略条件写入分支 */
if (expire &&
    getExpireMillisecondsOrReply(
        c, expire, relative_ttl, unit, &milliseconds) != C_OK)
{
    return;
}

setKeyByLink(c, c->db, key, valref, setkey_flags, &link);

if (expire)
    *valref = setExpireByLink(
        c, c->db, key->ptr, milliseconds, link);
```

`setExpireByLink` 负责把 TTL 写到 `kvobj`，并维护 `db->expires`：

```c
/* db.c:setExpireByLink，保留首次设置 TTL 的主线 */
kvobj *kv = dictGetKV(*keyLink);
long long old_when = kvobjGetExpire(kv);

if (old_when != -1) {
    kvobjSetExpire(kv, when);
} else {
    kvobj *kvnew = kvobjSetExpire(kv, when);

    if (kv != kvnew) {
        kvstoreDictSetAtLink(db->keys, slot, kvnew, &keyLink, 0);
        kv = kvnew;
    }

    kvstoreDictAddRaw(db->expires, slot, kv, NULL);
}
```

到这里，`token` 的状态可以这样理解：

- `db->keys` 负责普通读写查找；
- `db->expires` 负责让后台过期扫描找到它；
- 绝对过期时间保存在 `kvobj` 上。

Redis 传播这条写入时，会把相对 TTL 规范化成绝对过期时间，例如 `PXAT`。这样 AOF
重放和 replica 接收命令时，使用同一个到期点。

---

## GET token 时的懒过期

客户端读取 `token` 时，查找路径会顺手检查 TTL：

```text
GET token
  -> lookupKeyRead
  -> lookupKey
  -> expireIfNeeded
```

核心逻辑在 `lookupKey`：

```c
/* db.c:lookupKey，省略访问统计 */
kvobj *val = dbFindByLink(db, key->ptr, link);

if (val) {
    if (expireIfNeeded(db, key, val, expire_flags) != KEY_VALID) {
        val = NULL;
        if (link) *link = NULL;
    }
}
return val;
```

`expireIfNeeded` 先判断当前时间是否已经越过 `token` 的绝对过期时间：

```c
/* db.c:expireIfNeeded，保留主线 */
if (!keyIsExpired(db, key->ptr, kv))
    return KEY_VALID;

if (server.masterhost != NULL &&
    !(flags & EXPIRE_FORCE_DELETE_EXPIRED))
{
    return KEY_EXPIRED;
}

deleteExpiredKeyAndPropagate(db, key);
return KEY_DELETED;
```

在 master 上，过期的 `token` 会被删除并进入传播流程。在普通 replica 上，读路径会把
`token` 隐藏成缺席状态，物理删除等待 master 传播过来的删除命令。

所以到期后的读行为是明确的：客户端读到的是 key 缺席；物理删除可能发生在这次访问中，
也可能由后台扫描完成。

---

## 冷 key 依靠主动过期清理

如果 `token` 到期后一直无人读取，懒过期失去访问入口。Redis 还会在后台主动扫描
带 TTL 的 key。

主动过期有两个入口：

```c
/* server.c:databasesCron，由 serverCron 周期调用 */
if (server.active_expire_enabled) {
    if (iAmMaster()) {
        activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
    } else {
        expireSlaveKeys();
    }
}

/* server.c:beforeSleep，事件循环阻塞前 */
if (server.active_expire_enabled && iAmMaster())
    activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);
```

后台扫描的核心是 `db->expires_cursor`。每个 DB 保存自己的 cursor，下一轮从上次
位置继续扫：

```c
/* expire.c:activeExpireCycle，省略统计和时间判断 */
unsigned long num = kvstoreSize(db->expires);
if (num > config_keys_per_loop)
    num = config_keys_per_loop;

long max_buckets = ...;
long checked_buckets = 0;

while (data.sampled < num && checked_buckets < max_buckets) {
    db->expires_cursor = kvstoreScan(
        db->expires,
        db->expires_cursor,
        -1,
        expireScanCallback,
        expirySamplingShouldSkipDict,
        &data);

    if (db->expires_cursor == 0) {
        db_done = 1;
        break;
    }
    checked_buckets++;
}
```

扫描到 `token` 对应的 `kvobj` 后，回调会读取过期时间。已经到期就删除：

```c
/* expire.c */
int activeExpireCycleTryExpire(
    redisDb *db, kvobj *kv, long long now)
{
    if (now < kvobjGetExpire(kv))
        return 0;

    enterExecutionUnit(1, 0);
    sds key = kvobjGetKey(kv);
    robj *keyobj = createStringObject(key, sdslen(key));
    deleteExpiredKeyAndPropagate(db, keyobj);
    decrRefCount(keyobj);
    exitExecutionUnit();

    postExecutionUnitOperations();
    return 1;
}
```

主动过期可以压缩成三步：

```text
从 db->expires_cursor 继续扫描 db->expires
  -> 扫到 token 后读取 kvobj 上的绝对过期时间
  -> 已到期则删除并传播
```

当前 DB 中到期 key 命中率高时，`activeExpireCycle` 会继续处理当前 DB；每轮也会受
时间预算约束，把后台工作切成主线程可以承受的小段。

---

## 过期删除会做哪些事

无论 `token` 是被 `GET` 触发删除，还是被主动过期扫描删除，最后都会进入同一条
删除传播路径：

```c
/* db.c，省略延迟统计 */
static void deleteKeyAndPropagate(
    redisDb *db,
    robj *keyobj,
    int notify_type,
    long long *key_mem_freed)
{
    int del_flag = notify_type == NOTIFY_EXPIRED
        ? DB_FLAG_KEY_EXPIRED
        : DB_FLAG_KEY_EVICTED;
    int lazy_flag = notify_type == NOTIFY_EXPIRED
        ? server.lazyfree_lazy_expire
        : server.lazyfree_lazy_eviction;
    char *notify_name = notify_type == NOTIFY_EXPIRED
        ? "expired"
        : "evicted";

    dbGenericDelete(db, keyobj, lazy_flag, del_flag);
    notifyKeyspaceEvent(notify_type, notify_name, keyobj, db->id);
    keyModified(NULL, db, keyobj, NULL, 1);
    propagateDeletion(db, keyobj, lazy_flag);

    if (notify_type == NOTIFY_EXPIRED)
        server.stat_expiredkeys++;
    else
        server.stat_evictedkeys++;
}
```

对 `token` 来说，这一步包含四件事：

- 从 `db->expires` 和 `db->keys` 移除；
- 发送 `expired` keyspace notification；
- 更新相关统计；
- 向 AOF 和 replica 传播删除命令。

传播时使用 `DEL` 还是 `UNLINK`，由对应的 lazy-free 配置决定：

```c
/* db.c:propagateDeletion */
robj *argv[2];
argv[0] = lazy ? shared.unlink : shared.del;
argv[1] = key;

alsoPropagate(
    db->id, argv, 2, PROPAGATE_AOF | PROPAGATE_REPL);
```

到这里，expire 主线完整闭合：`SET token abc EX 60` 写入绝对过期时间；读路径和后台
扫描都通过 `kvobjGetExpire` 判断到期；删除时清理索引、通知、统计和传播。

---

## 内存压力下的淘汰入口

现在换一个场景：`token` 到期前，Redis 实例已经超过 `maxmemory`。这时进入
eviction 主线。

Redis 在命令执行前检查内存状态，并尝试执行淘汰：

```c
/* server.c:processCommand */
if (server.maxmemory && !isInsideYieldingLongCommand()) {
    int out_of_memory =
        (performEvictions() == EVICT_FAIL);

    if (out_of_memory && is_denyoom_command) {
        rejectCommand(c, shared.oomerr);
        return C_OK;
    }

    server.pre_command_oom_state = out_of_memory;
}
```

这段代码给 `token` 带来两个结论：

- 淘汰由内存压力触发；
- TTL 只在某些淘汰策略选择候选时参与。

如果淘汰路径选中了 `token`，它会立刻走淘汰删除，直接复用前面讲过的删除传播流程：

```c
/* evict.c，省略候选选择 */
enterExecutionUnit(1, 0);
deleteEvictedKeyAndPropagate(db, keyobj, &key_mem_freed);
exitExecutionUnit();

postExecutionUnitOperations();
```

---

## maxmemory-policy 怎样影响 token

`maxmemory-policy` 先决定候选集合。源码里候选来源只有两类：

```c
/* evict.c:performEvictions */
if (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) {
    kvs = db->keys;
} else {
    kvs = db->expires;
}
```

套到 `SET token abc EX 60` 上：

| 策略族 | 候选集合 | `token` 的位置 |
|-|-|-|
| `noeviction` | 无淘汰候选 | 内存增长命令可能收到 OOM；`token` 仍由过期路径处理 |
| `allkeys-*` | `db->keys` | `token` 作为普通 key 参与淘汰 |
| `volatile-*` | `db->expires` | `token` 因为带 TTL 参与淘汰 |

候选集合确定后，再看具体选择方式：

| 策略后缀 | 选择方式 |
|-|-|
| `random` | 从候选集合中随机取 key |
| `lru` | 倾向选择最近访问更早的 key |
| `lfu` | 倾向选择访问频率更低的 key |
| `ttl` | 倾向选择更早到期的 key，只存在 `volatile-ttl` |
| `lrm` | 倾向选择最近修改更早的 key |

因此，`token` 在不同策略下的命运很直接：

- `allkeys-lru`：它和所有 key 一起比较访问时间；
- `volatile-lru`：它因为带 TTL 进入候选，再和其他带 TTL 的 key 比较访问时间；
- `volatile-ttl`：它和其他带 TTL 的 key 比较绝对过期时间；
- `noeviction`：淘汰路径无 key 可删，内存增长命令转成 OOM。

---

## 淘汰怎样从候选中选 key

随机策略直接选 key。LRU、LFU、LRM 和 TTL 优先策略会先采样，再把样本放进
eviction pool。

`evictionPoolPopulate` 会按当前策略给样本打分。LRU/LRM 和 TTL 的分数方向在源码里
很直接：

```c
/* evict.c:evictionPoolPopulate，摘出相关分支 */
if (server.maxmemory_policy &
    (MAXMEMORY_FLAG_LRU | MAXMEMORY_FLAG_LRM))
{
    idle = estimateObjectIdleTime(kv);
} else if (server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL) {
    idle = ULLONG_MAX - kvobjGetExpire(kv);
}
```

这里的 `idle` 可以理解成统一的淘汰分：

- LRU / LRM：越久未访问或越久未修改，分数越高；
- LFU：也会转换成同一方向，访问频率越低，分数越高；
- TTL：到期时间越早，分数越高。

样本进入 pool 时，Redis 保留当前见过的高分候选：

```c
/* evict.c:evictionPoolPopulate，保留插入逻辑的形状 */
k = 0;
while (k < EVPOOL_SIZE &&
       pool[k].key &&
       pool[k].idle < idle) k++;

if (k == 0 && pool[EVPOOL_SIZE-1].key != NULL) {
    continue;
}

...

pool[k].key = ...;
pool[k].idle = idle;
pool[k].dbid = db->id;
pool[k].slot = slot;
```

`performEvictions` 删除 key 时，从 pool 的高分端取候选。取出前会回到对应的
dict 查一次，确认这个 key 仍然存在：

```c
/* evict.c:performEvictions，保留 pool 取 key 的主线 */
for (k = EVPOOL_SIZE-1; k >= 0; k--) {
    if (pool[k].key == NULL) continue;

    bestdbid = pool[k].dbid;
    kvs = (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS)
        ? server.db[bestdbid].keys
        : server.db[bestdbid].expires;

    de = kvstoreDictFind(kvs, pool[k].slot, pool[k].key);

    pool[k].key = NULL;
    pool[k].idle = 0;

    if (de) {
        bestkey = kvobjGetKey(dictGetKV(de));
        break;
    }
}
```

对 `token` 来说，完整淘汰链路是：

```text
内存超过 maxmemory
  -> 根据 policy 选择 db->keys 或 db->expires
  -> token 如果属于候选集合，就有机会被采样
  -> 采样命中 token 后，按策略给它打分
  -> 分数足够高时进入 eviction pool
  -> pool 高分端选中 token 后执行淘汰删除
```

这个过程解释了 `token` 的两个常见结果：

- 它带 TTL，所以在 `volatile-*` 策略下会进入候选集合；
- 它到期前也可能被淘汰，淘汰路径只关心内存压力和策略选择。

---

## SET token abc EX 60 的完整模型

把两条主线合起来看：

```text
SET token abc EX 60
  -> 写入 db->keys
  -> 写入过期时间到 kvobj
  -> 加入 db->expires

时间到达过期点
  -> GET token 触发 expireIfNeeded
  -> 或后台 activeExpireCycle 扫到 token
  -> deleteExpiredKeyAndPropagate

时间到达前发生内存压力
  -> processCommand 调用 performEvictions
  -> maxmemory-policy 决定候选集合
  -> token 可能被采样、打分、进入 eviction pool
  -> deleteEvictedKeyAndPropagate
```

最后用一张表收束：

| 问题 | 回答 |
|-|-|
| `EX 60` 存在哪里 | 绝对过期时间存到 `kvobj`；`db->expires` 负责索引带 TTL 的 key |
| 到期后谁发现它 | 读路径的 `expireIfNeeded`，以及后台的 `activeExpireCycle` |
| 过期删除做什么 | 清理索引、发通知、更新统计、传播 `DEL/UNLINK` |
| 内存压力下谁处理它 | `performEvictions` |
| TTL 对淘汰有什么影响 | 在 `volatile-*` 策略里，带 TTL 的 `token` 会进入候选集合 |
| 到期前会被删吗 | 内存压力和淘汰策略选中它时，会走淘汰删除 |

`SET token abc EX 60` 的核心语义可以记成一句话：TTL 决定 `token` 到期后的有效性；
`maxmemory-policy` 决定内存压力下 `token` 是否成为牺牲对象。过期和淘汰最终都会删除
同一个 key，但它们的触发条件和选择路径不同。
