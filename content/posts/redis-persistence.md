+++
date = '2026-07-12T21:52:00+08:00'
draft = false
title = 'Redis Persistence'
tags = ['Redis']
categories = ['数据库']
+++

# Redis 持久化：从一条 SET 命令看 RDB 与 AOF

沿着下面这条命令观察数据怎样进入内存、AOF 和 RDB，最后又怎样在重启时恢复：

```redis
SET token abc EX 60
```

它表达了两项状态：

```text
value(token) = "abc"
expire(token) = 当前时间 + 60 秒
```

Redis 有两种方式把这两项状态保存到磁盘：

```text
RDB：保存某个时刻的 value + expire 快照
AOF：保存能够重建 value + expire 的写命令
```

主要源码入口：

| 环节 | 源码 |
|-|-|
| `SET` 执行与过期时间改写 | `src/t_string.c:setGenericCommand` |
| 写命令传播 | `src/server.c:call`、`propagateNow` |
| AOF 编码、写入与刷盘 | `src/aof.c:feedAppendOnlyFile`、`flushAppendOnlyFile` |
| RDB 编码、保存与加载 | `src/rdb.c:rdbSave*`、`rdbLoadRio*` |
| AOF rewrite 与加载 | `src/aof.c:rewriteAppendOnlyFileBackground`、`loadAppendOnlyFiles` |
| 自动持久化调度 | `src/server.c:serverCron` |

下文每个 C 代码块首行都标注 `源码文件:函数`；`/* ... */` 表示省略了与当前链路无关的源码。

---

## 1. `SET token abc EX 60` 先改内存

命令入口在 `src/t_string.c:setCommand`：

```c
/* src/t_string.c:setCommand */
void setCommand(client *c) {
    extendedStringArgs args;

    if (parseExtendedStringArgumentsOrReply(c, 3, &args, COMMAND_SET) != C_OK) {
        return;
    }

    c->argv[2] = tryObjectEncoding(c->argv[2]);
    setGenericCommand(c, args.flags, c->argv[1], &(c->argv[2]),
                      args.expire, args.unit, args.match_value, NULL, NULL);
}
```

`EX 60` 是相对 TTL。Redis 先把它换成绝对的毫秒时间戳：

```c
/* src/t_string.c:getExpireMillisecondsOrReply，省略参数校验 */
static int getExpireMillisecondsOrReply(client *c, robj *expire,
        int relative_ttl, int unit, long long *milliseconds) {
    int ret = getLongLongFromObjectOrReply(c, expire, milliseconds, NULL);
    if (ret != C_OK) return ret;

    /* ... */

    if (unit == UNIT_SECONDS) *milliseconds *= 1000;

    if (relative_ttl) {
        *milliseconds += commandTimeSnapshot();
    }

    /* ... */
    return C_OK;
}
```

假设 `commandTimeSnapshot()` 为 `1720791000000`，计算结果就是 `1720791060000`。

然后写入数据库并设置过期时间：

```c
/* src/t_string.c:setGenericCommand，省略条件写入分支 */
if (expire && getExpireMillisecondsOrReply(
        c, expire, relative_ttl, unit, &milliseconds) != C_OK) {
    return;
}

/* ... */

setKeyByLink(c, c->db, key, valref, setkey_flags, &link);
if (expire)
    *valref = setExpireByLink(c, c->db, key->ptr, milliseconds, link);
incrRefCount(*valref);

server.dirty++;
notifyKeyspaceEvent(NOTIFY_STRING,"set",key,c->db->id);
```

此时内存中的逻辑状态是：

```text
db["token"] = "abc"
expire["token"] = 1720791060000
```

这里保存的是绝对过期点，不是“还剩 60 秒”。否则 Redis 重启或复制有延迟时，`token` 会凭空多活一段时间。

### 传播前改写成 `PXAT`

Redis 还会改写客户端命令向量：

```c
/* src/t_string.c:setGenericCommand */
if (expire) {
    if (!(flags & OBJ_PXAT)) {
        robj *milliseconds_obj = createStringObjectFromLongLong(milliseconds);

        if ((c->cmd->proc == setCommand) && c->argc == 5) {
            rewriteClientCommandArgument(c, 3, shared.pxat);
            rewriteClientCommandArgument(c, 4, milliseconds_obj);
        } else {
            rewriteClientCommandVector(c, 5, shared.set, key, *valref,
                                       shared.pxat, milliseconds_obj);
        }
        decrRefCount(milliseconds_obj);
    }
    notifyKeyspaceEvent(NOTIFY_GENERIC,"expire",key,c->db->id);
}
```

所以客户端发送的是：

```redis
SET token abc EX 60
```

真正交给 AOF 和 replica 的是：

```redis
SET token abc PXAT 1720791060000
```

这个转换由 `setGenericCommand()` 完成，`feedAppendOnlyFile()` 只负责通用 RESP 编码。

---

## 2. `dirty` 决定是否传播

`src/server.c:call` 在执行命令前后各读取一次 `server.dirty`：

```c
/* src/server.c:call，省略统计和客户端标志处理 */
void call(client *c, int flags) {
    long long dirty = server.dirty;

    /* ... */
    c->cmd->proc(c);
    /* ... */

    dirty = server.dirty-dirty;
    if (dirty < 0) dirty = 0;

    /* ... */
    if (flags & CMD_CALL_PROPAGATE &&
        (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP &&
        c->cmd->proc != execCommand &&
        !(c->cmd->flags & CMD_MODULE))
    {
        int propagate_flags = PROPAGATE_NONE;
        if (dirty) propagate_flags |= (PROPAGATE_AOF|PROPAGATE_REPL);

        /* 省略强制传播及阻止 AOF/replication 的标志处理 */

        if (propagate_flags != PROPAGATE_NONE)
            alsoPropagate(c->db->id,c->argv,c->argc,propagate_flags);
    }
}
```

这里的 `alsoPropagate()` 并不会立即写 AOF 或向 replica 发送命令。它会复制参数，并把命令追加到
`server.also_propagate` 待传播队列。`call()` 执行完命令实现并调用 `exitExecutionUnit()` 后，会在末尾进入
`afterCommand()`；当执行嵌套已经回到最外层时，才统一处理队列：

```text
processCommand()
  -> call(c, CMD_CALL_FULL)
      -> enterExecutionUnit()
      -> c->cmd->proc(c)
      -> exitExecutionUnit()
      -> alsoPropagate()                 # 加入 server.also_propagate
      -> afterCommand()
          -> postExecutionUnitOperations()
              -> propagatePendingCommands()
                  -> propagateNow()      # 实际写 AOF、传播给 replica
```

`postExecutionUnitOperations()` 会先检查 `server.execution_nesting`。如果仍处于脚本、事务或模块调用等
嵌套执行单元中，它会直接返回；最外层执行单元结束后，`propagatePendingCommands()` 才遍历
`server.also_propagate`，逐条调用 `propagateNow()`：

```c
/* src/server.c:propagateNow，省略防御性检查和 ASM 迁移分支 */
static void propagateNow(int dbid, robj **argv, int argc, int target) {
    if (!shouldPropagate(target))
        return;

    /* ... */
    if (server.aof_state != AOF_OFF && target & PROPAGATE_AOF)
        feedAppendOnlyFile(dbid, argv, argc);

    if (target & PROPAGATE_REPL)
        replicationFeedSlaves(server.slaves, dbid, argv, argc);
}
```

因此这条命令从这里分成两路：

```text
SET token abc EX 60
  -> 修改内存，dirty + 1
  -> 改写为 SET token abc PXAT 1720791060000
  -> AOF：追加到本机日志
  -> replication：发送给 replica
```

`server.dirty` 还有另一项用途：自动 RDB 会用它判断自上次成功快照后是否积累了足够多的修改。它不是“只有大于 0 才允许 AOF”的全局开关，而是一个持续累积的修改计数器；`call()` 使用的是本次命令带来的差值。

---

## 3. AOF：把改写后的命令写成 RESP

`src/aof.c:feedAppendOnlyFile` 先处理数据库选择，再编码命令。下面假设命令在 DB 0 中执行：

```c
/* src/aof.c:feedAppendOnlyFile，省略时间戳注解 */
void feedAppendOnlyFile(int dictid, robj **argv, int argc) {
    sds buf = sdsempty();

    /* ... */
    if (dictid != -1 && dictid != server.aof_selected_db) {
        char seldb[64];

        snprintf(seldb,sizeof(seldb),"%d",dictid);
        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
            (unsigned long)strlen(seldb),seldb);
        server.aof_selected_db = dictid;
    }

    buf = catAppendOnlyGenericCommand(buf, argc, argv);

    if (server.aof_state == AOF_ON ||
        (server.aof_state == AOF_WAIT_REWRITE &&
         server.child_type == CHILD_TYPE_AOF))
    {
        server.aof_buf = sdscatlen(server.aof_buf, buf, sdslen(buf));
    }

    sdsfree(buf);
}
```

`catAppendOnlyGenericCommand()` 使用 RESP 数组和批量字符串格式：

```c
/* src/aof.c:catAppendOnlyGenericCommand，展示参数编码循环 */
for (j = 0; j < argc; j++) {
    o = getDecodedObject(argv[j]);
    buf[0] = '$';
    len = 1+ll2string(buf+1,sizeof(buf)-1,sdslen(o->ptr));
    buf[len++] = '\r';
    buf[len++] = '\n';
    dst = sdscatlen(dst,buf,len);
    dst = sdscatlen(dst,o->ptr,sdslen(o->ptr));
    dst = sdscatlen(dst, "\r\n", 2);
    decrRefCount(o);
}
```

本例最终进入 `server.aof_buf` 的内容类似：

```text
*5\r\n
$3\r\nSET\r\n
$5\r\ntoken\r\n
$3\r\nabc\r\n
$4\r\nPXAT\r\n
$13\r\n1720791060000\r\n
```

这里有三个容易混淆的阶段：

```text
feedAppendOnlyFile  -> Redis 用户态缓冲区 server.aof_buf
write               -> 操作系统页缓存
fsync/fdatasync     -> 请求内核把数据同步到持久化设备
```

进入 `aof_buf` 不等于已经写文件，`write` 成功也不等于断电后一定还在。

---

## 4. `beforeSleep`：先尝试刷新 AOF，再发送客户端响应

Redis 每轮事件循环重新等待事件之前会调用 `src/server.c:beforeSleep`：

```c
/* src/server.c:beforeSleep，省略 AOF 前后的事件循环任务 */
if (server.aof_state == AOF_ON || server.aof_state == AOF_WAIT_REWRITE)
    flushAppendOnlyFile(0);

/* ... */
handleClientsWithPendingWrites();
sendPendingClientsToIOThreads();
```

在不需要延迟写入时，`flushAppendOnlyFile()` 对 `aof_buf` 做一次 `write`，成功后清空或复用缓冲区：

```c
/* src/aof.c:flushAppendOnlyFile，省略错误处理 */
if (sdslen(server.aof_buf) == 0) {
    if (server.aof_last_incr_fsync_offset == server.aof_last_incr_size) {
        /* 已全部 fsync，省略 WAITAOF offset 更新 */
        /* ... */
    } else {
        if (server.aof_fsync == AOF_FSYNC_EVERYSEC &&
            server.mstime - server.aof_last_fsync >= 1000 &&
            !(sync_in_progress = aofFsyncInProgress()))
            goto try_fsync;

        if (server.aof_fsync == AOF_FSYNC_ALWAYS)
            goto try_fsync;
    }
    return;
}

if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
    sync_in_progress = aofFsyncInProgress();

if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
    if (sync_in_progress) {
        if (server.aof_flush_postponed_start == 0) {
            server.aof_flush_postponed_start = server.mstime;
            return;
        } else if (server.mstime - server.aof_flush_postponed_start < 2000) {
            return;
        }
        server.aof_delayed_fsync++;
    }
}

nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));

/* ... */
server.aof_current_size += nwritten;
server.aof_last_incr_size += nwritten;

if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
    sdsclear(server.aof_buf);
} else {
    sdsfree(server.aof_buf);
    server.aof_buf = sdsempty();
}

try_fsync:
/* 后接 always/everysec 分支 */
```

这里首先能确定的是调用顺序：Redis 先调用 `flushAppendOnlyFile()`，再处理待发送的客户端响应。但调用
`flushAppendOnlyFile()` 不等于本轮一定执行了 `write`。在 `appendfsync everysec` 模式下，如果后台 fsync
仍在进行，函数可以提前返回，将写入最多推迟约 2 秒，随后客户端仍可能收到 `OK`；在
`appendfsync always` 模式下，Redis 才会在发送响应前完成本轮 `write + fsync`。具体持久性承诺取决于
下面的 `appendfsync` 策略。

### `appendfsync always`

```c
/* src/aof.c:flushAppendOnlyFile，AOF_FSYNC_ALWAYS 分支 */
if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
    if (redis_fsync(server.aof_fd) == -1) {
        serverLog(LL_WARNING,"Can't persist AOF for fsync error when the "
          "AOF fsync policy is 'always': %s. Exiting...", strerror(errno));
        exit(1);
    }
    server.aof_last_incr_fsync_offset = server.aof_last_incr_size;
    server.aof_last_fsync = server.mstime;
}
```

每轮 AOF 写入后同步刷盘，确认语义最强，但磁盘延迟会直接进入命令响应路径。

### `appendfsync everysec`

```c
/* src/aof.c:flushAppendOnlyFile，AOF_FSYNC_EVERYSEC 分支 */
else if (server.aof_fsync == AOF_FSYNC_EVERYSEC &&
         server.mstime - server.aof_last_fsync >= 1000)
{
    if (!sync_in_progress) {
        aof_background_fsync(server.aof_fd);
        server.aof_last_incr_fsync_offset = server.aof_last_incr_size;
    }
    server.aof_last_fsync = server.mstime;
}
```

主线程仍执行 `write`，大约每秒安排一次后台 fsync。机器掉电时通常最多损失约一秒的 AOF 数据；磁盘繁忙时 fsync 可能延迟，不能把“一秒”理解成严格的实时上界。

### `appendfsync no`

`flushAppendOnlyFile()` 没有 `AOF_FSYNC_NO` 的 fsync 分支，因此只执行前面的 `write`，刷盘时机交给操作系统。

三种策略的差别可以压缩为：

| 策略 | `write` | `fsync` | 主要代价 |
|-|-|-|-|
| `always` | 主线程 | 每次 AOF flush，同步执行 | 延迟最高，持久性最强 |
| `everysec` | 主线程 | 约每秒由 BIO 后台执行 | 常用折中 |
| `no` | 主线程 | Redis 不主动执行 | 依赖操作系统刷盘 |

---

## 5. RDB：直接保存 `token` 的当前状态

AOF 保存的是：

```redis
SET token abc PXAT 1720791060000
```

RDB 不保存这条命令，而是遍历数据库，把当前对象和元数据序列化：

```text
key    = token
type   = string
value  = abc
expire = 1720791060000
```

`src/rdb.c:rdbSaveDb` 的主循环如下：

```c
/* src/rdb.c:rdbSaveDb，省略 Cluster、统计和错误清理 */
ssize_t rdbSaveDb(rio *rdb, int dbid, int rdbflags,
                  long *key_counter, unsigned long long *skipped) {
    redisDb *db = server.db + dbid;

    if ((res = rdbSaveType(rdb,RDB_OPCODE_SELECTDB)) < 0) goto werr;
    if ((res = rdbSaveLen(rdb, dbid)) < 0) goto werr;

    unsigned long long expires_size = kvstoreSize(db->expires);
    if ((res = rdbSaveType(rdb,RDB_OPCODE_RESIZEDB)) < 0) goto werr;
    if ((res = rdbSaveLen(rdb,db_size)) < 0) goto werr;
    if ((res = rdbSaveLen(rdb,expires_size)) < 0) goto werr;

    /* ... */
    while ((de = kvstoreIteratorNext(&kvs_it)) != NULL) {
        /* ... */
        kvobj *kv = dictGetKV(de);
        initStaticStringObject(key,kvobjGetKey(kv));
        expire = kvobjGetExpire(kv);
        res = rdbSaveKeyValuePair(rdb, &key, kv, expire, dbid);
        if (res < 0) goto werr2;
        /* ... */
    }
}
```

单个键按“过期时间、类型、key、value”的顺序保存：

```c
/* src/rdb.c:rdbSaveKeyValuePair，省略 LRU、LFU 和 module metadata */
int rdbSaveKeyValuePair(rio *rdb, robj *key,
                        robj *val, long long expiretime, int dbid)
{
    if (expiretime != -1) {
        if (rdbSaveType(rdb,RDB_OPCODE_EXPIRETIME_MS) == -1) return -1;
        if (rdbSaveMillisecondTime(rdb,expiretime) == -1) return -1;
    }

    /* ... */
    if (rdbSaveObjectType(rdb,val) == -1) return -1;
    if (rdbSaveStringObject(rdb,key) == -1) return -1;
    if (rdbSaveObject(rdb,val,key,dbid) == -1) return -1;

    return 1;
}
```

Redis 会根据对象类型和内部编码选择 RDB 类型，而不是把内存地址或 C 结构体原样写入磁盘：

```c
/* src/rdb.c:rdbSaveObjectType，只展示 String 与 List */
switch (o->type) {
case OBJ_STRING:
    return rdbSaveType(rdb, RDB_TYPE_STRING);
case OBJ_LIST:
    if (o->encoding == OBJ_ENCODING_QUICKLIST ||
        o->encoding == OBJ_ENCODING_LISTPACK)
        return rdbSaveType(rdb, RDB_TYPE_LIST_QUICKLIST_2);
    else
        serverPanic("Unknown list encoding");
/* 其余对象类型分支省略 */
}
```

完整文件还包含 header、辅助字段、EOF 和校验和：

```c
/* src/rdb.c:rdbSaveRio，省略 module/function AUX 和错误处理 */
int rdbSaveRio(int req, rio *rdb, int *error,
               int rdbflags, rdbSaveInfo *rsi) {
    snprintf(magic, sizeof(magic), "REDIS%04d", RDB_VERSION);
    if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;
    if (rdbSaveInfoAuxFields(rdb,rdbflags,rsi) == -1) goto werr;

    /* ... */
    for (int j = 0; j < server.dbnum; j++)
        if (rdbSaveDb(rdb, j, rdbflags, &key_counter, &skipped) == -1)
            goto werr;

    if (rdbSaveType(rdb,RDB_OPCODE_EOF) == -1) goto werr;
    cksum = rdb->cksum;
    memrev64ifbe(&cksum);
    if (rioWrite(rdb,&cksum,8) == 0) goto werr;
    return C_OK;
}
```

所以 RDB 的逻辑结构是：

```text
REDISxxxx
  AUX ...
  SELECTDB 0
  RESIZEDB ...
  EXPIRETIME_MS 1720791060000
  STRING "token" "abc"
  EOF
  CRC64
```

---

## 6. `SAVE` 与 `BGSAVE` 的区别在谁执行 `rdbSave`

`SAVE` 直接在 Redis 主进程中调用 `rdbSave()`：

```c
/* src/rdb.c:saveCommand */
void saveCommand(client *c) {
    /* ... */
    rdbSaveInfo rsi, *rsiptr;
    rsiptr = rdbPopulateSaveInfo(&rsi);
    if (rdbSave(SLAVE_REQ_NONE,server.rdb_filename,
                rsiptr,RDBFLAGS_NONE) == C_OK) {
        addReply(c,shared.ok);
    } else {
        addReplyErrorObject(c,shared.err);
    }
}
```

遍历和写盘都占用主线程，因此数据量大时会阻塞客户端命令。

`BGSAVE` 则 fork 子进程：

```c
/* src/rdb.c:rdbSaveBackground，省略日志和 fork 失败处理 */
int rdbSaveBackground(int req, char *filename,
                      rdbSaveInfo *rsi, int rdbflags) {
    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);

    if ((childpid = redisFork(CHILD_TYPE_RDB)) == 0) {
        int retval = rdbSave(req, filename,rsi,rdbflags);
        exitFromChild(retval == C_OK ? 0 : 1, 0);
    } else {
        /* 父进程记录子进程状态后返回事件循环 */
        /* ... */
        return C_OK;
    }
}
```

执行 `rdbSave()` 的进程先写临时文件，完成 `fflush + fsync + fclose` 后再原子 rename：

```c
/* src/rdb.c:rdbSave，省略日志和错误清理 */
int rdbSave(int req, char *filename, rdbSaveInfo *rsi, int rdbflags) {
    snprintf(tmpfile, 256, "temp-%d.rdb", getpid());
    if (rdbSaveInternal(req,tmpfile,rsi,rdbflags) != C_OK) {
        stopSaving(0);
        return C_ERR;
    }

    if (rename(tmpfile,filename) == -1) {
        /* 省略错误日志 */
        unlink(tmpfile);
        stopSaving(0);
        return C_ERR;
    }
    if (fsyncFileDir(filename) != 0) {
        stopSaving(0);
        return C_ERR;
    }

    server.dirty = 0;
    server.lastsave = time(NULL);
    server.lastbgsave_status = C_OK;
    stopSaving(1);
    return C_OK;
}
```

### fork 为什么能得到一致快照

```text
BGSAVE fork 时刻 T0

父进程页 ─────────┐
                  ├─ 父子最初共享物理页
子进程页 ─────────┘

T0 之后父进程修改某个内存页
  -> 内核复制该页（Copy On Write）
  -> 父进程写新页
  -> 子进程仍读取 T0 时的旧页
```

因此子进程看到的是 fork 时刻的数据集。代价也来自这里：

```text
fork：页表复制和内核工作可能造成短暂停顿
COW：保存期间写流量越大，被复制的内存页可能越多
I/O：子进程顺序写 RDB，会与 AOF 等磁盘操作竞争
```

快照完成后，父进程只扣除 fork 前已经包含在快照中的修改量：

```c
/* src/rdb.c:backgroundSaveDoneHandlerDisk，成功分支 */
server.dirty = server.dirty - server.dirty_before_bgsave;
server.lastsave = save_end;
```

这样 `BGSAVE` 期间发生的新写入仍保留在 `dirty` 中，供下一次自动快照判断。

---

## 7. 自动 RDB 只是定时触发 `BGSAVE`

自动 RDB 的默认条件是：

```conf
save 3600 1 300 100 60 10000
```

等价于：

```text
3600 秒后累计至少 1 次修改
或 300 秒后累计至少 100 次修改
或 60 秒后累计至少 10000 次修改
```

`src/server.c:serverCron` 周期检查这些条件：

```c
/* src/server.c:serverCron，自动 RDB 检查分支 */
for (j = 0; j < server.saveparamslen; j++) {
    struct saveparam *sp = server.saveparams + j;

    if (server.dirty >= sp->changes &&
        server.unixtime - server.lastsave > sp->seconds &&
        (server.unixtime - server.lastbgsave_try >
         CONFIG_BGSAVE_RETRY_DELAY ||
         server.lastbgsave_status == C_OK))
    {
        rdbSaveInfo rsi, *rsiptr;
        rsiptr = rdbPopulateSaveInfo(&rsi);
        rdbSaveBackground(SLAVE_REQ_NONE,server.rdb_filename,
                          rsiptr,RDBFLAGS_NONE);
        break;
    }
}
```

多条 `save <seconds> <changes>` 规则之间是“或”：`for` 循环逐条检查，任意一条满足就执行
`rdbSaveBackground()` 并 `break`。每条规则的完整条件是：

```text
修改次数达标
AND 距上次成功保存的时间达标
AND（已经超过失败重试间隔 OR 上次 BGSAVE 成功）
```

`SET token abc EX 60` 会让 `dirty` 增加，但是否立刻触发 RDB，还取决于是否有任意一条 `save` 规则
同时达到时间和累计修改次数阈值，并且允许进行本次失败重试。

---

## 8. AOF rewrite：从“历史命令”压缩为“当前状态”

假设 `token` 被反复更新：

```redis
SET token old PXAT 1720791000000
SET token temp PXAT 1720791030000
SET token abc PXAT 1720791060000
```

恢复当前状态只需要最后的有效结果。AOF rewrite 不复制旧 AOF，而是遍历当前内存数据，生成新的 base 文件。

AOF 使用多文件组织：

```text
appendonlydir/
  appendonly.aof.manifest
  appendonly.aof.3.base.rdb
  appendonly.aof.3.incr.aof
```

文件职责是：

```text
base：rewrite 时刻的完整数据集，可以是 RDB 或命令格式 AOF
incr：base 之后发生的写命令
history：已被新 base 覆盖、等待后台删除的旧文件
manifest：记录当前有效 base 与 incr 的文件名、类型和顺序
```

`BGREWRITEAOF` 的关键顺序在 `src/aof.c:rewriteAppendOnlyFileBackground`：

```c
/* src/aof.c:rewriteAppendOnlyFileBackground，省略状态检查和日志 */
int rewriteAppendOnlyFileBackground(void) {
    server.aof_selected_db = -1;
    flushAppendOnlyFile(1);
    if (openNewIncrAofForAppend() != C_OK)
        return C_ERR;

    /* ... */
    if ((childpid = redisFork(CHILD_TYPE_AOF)) == 0) {
        snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof", (int)getpid());
        if (rewriteAppendOnlyFile(tmpfile) == C_OK)
            exitFromChild(0, 0);
        else
            exitFromChild(1, 0);
    }

    /* 父进程记录子进程状态后返回事件循环 */
    /* ... */
    return C_OK;
}
```

假设 fork 后又执行：

```redis
SET user:1 online
```

那么两部分的边界是：

```text
新 base：fork 时刻已有的 token = abc，以及它的绝对过期点
新 incr：fork 之后的 SET user:1 online
```

子进程完成后，父进程切换文件集合：

```c
/* src/aof.c:backgroundRewriteDoneHandler，rewrite 成功分支 */
void backgroundRewriteDoneHandler(int exitcode, int bysignal) {
    /* ... */
    temp_am = aofManifestDup(server.aof_manifest);

    sds new_base_filename =
        getNewBaseFileNameAndMarkPreAsHistory(temp_am);
    new_base_filepath = makePath(server.aof_dirname, new_base_filename);
    if (rename(tmpfile, new_base_filepath) == -1) {
        /* ... */
        goto cleanup;
    }

    /* ... */
    markRewrittenIncrAofAsHistory(temp_am);

    if (persistAofManifest(temp_am) == C_ERR) {
        /* ... */
        goto cleanup;
    }
    aofManifestFreeAndUpdate(temp_am);
    aofDelHistoryFiles();
}
```

多文件设计让父进程可以直接把 rewrite 期间的新命令写入独立 incr 文件，不需要长期维护一个随写流量增长的 rewrite 差异缓冲区。

### RDB preamble

默认配置为：

```conf
aof-use-rdb-preamble yes
```

rewrite 子进程据此选择 base 格式：

```c
/* src/aof.c:rewriteAppendOnlyFile */
if (server.aof_use_rdb_preamble)
{
    int error;
    if (rdbSaveRio(SLAVE_REQ_NONE,&aof,&error,
                   RDBFLAGS_AOF_PREAMBLE,NULL) == C_ERR)
        goto werr;
} else {
    if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;
}
```

所以“开启 AOF”不代表目录中每个文件都是 RESP 命令：

```text
appendonly.aof.N.base.rdb  -> RDB 快照，紧凑、加载快
appendonly.aof.N.incr.aof -> RESP 写命令，记录增量
```

---

## 9. 重启恢复：开启 AOF 就加载 AOF，否则加载 RDB

启动时先读取 AOF manifest，然后进入 `src/server.c:loadDataFromDisk`：

```c
/* src/server.c:loadDataFromDisk，省略日志和 RDB 加载参数准备 */
void loadDataFromDisk(void) {
    if (server.aof_state == AOF_ON) {
        int ret = loadAppendOnlyFiles(server.aof_manifest);
        if (ret == AOF_FAILED || ret == AOF_OPEN_ERR)
            exit(1);
        /* ... */
    } else {
        /* ... */
        int rdb_load_ret = rdbLoad(server.rdb_filename, &rsi, rdb_flags);
        /* ... */
    }
}
```

这里不是比较 `dump.rdb` 和 AOF 谁更新，也不是 AOF 加载失败后自动回退 RDB。选择由 `appendonly` 配置决定：

```text
appendonly yes -> 走 AOF 加载路径
appendonly no  -> 走 dump.rdb 加载路径
```

AOF 加载时严格按 manifest 顺序读取：

```c
/* src/aof.c:loadAppendOnlyFiles，省略进度统计和错误处理 */
int loadAppendOnlyFiles(aofManifest *am) {
    if (am->base_aof_info) {
        aof_name = (char*)am->base_aof_info->file_name;
        ret = loadSingleAppendOnlyFile(aof_name);
        /* ... */
    }

    listRewind(am->incr_aof_list, &li);
    while ((ln = listNext(&li)) != NULL) {
        aofInfo *ai = (aofInfo*)ln->value;
        aof_name = (char*)ai->file_name;
        ret = loadSingleAppendOnlyFile(aof_name);
        /* ... */
    }

    /* ... */
    return ret;
}
```

单文件加载会检查前五个字节：

```c
/* src/aof.c:loadSingleAppendOnlyFile，文件格式判断分支 */
char sig[5]; /* "REDIS" */
if (fread(sig,1,5,fp) != 5 || memcmp(sig,"REDIS",5) != 0) {
    /* 不是 RDB：回到文件开头，随后按 RESP 命令读取 */
    if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
} else {
    rio rdb;
    if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
    rioInitWithFile(&rdb,fp);
    if (rdbLoadRio(&rdb,RDBFLAGS_AOF_PREAMBLE,NULL) != C_OK) {
        ret = AOF_FAILED;
        goto cleanup;
    }
}
```

于是本例的恢复过程是：

```text
RDB base 中有 token
  -> 读出 value = abc
  -> 读出 expire = 1720791060000

或 AOF 中有
  SET token abc PXAT 1720791060000
  -> 使用内部 fake client 重放命令
```

两种格式都保存绝对过期点。若 Redis 恢复时已经晚于 `1720791060000`，`token` 不会重新获得 60 秒寿命。

---

## 10. 配置如何选择

一个常见组合是：

```conf
# AOF 作为更连续的恢复记录
appendonly yes
appendfsync everysec

# rewrite 后的 base 使用 RDB 格式
aof-use-rdb-preamble yes

# 同时保留独立 RDB 快照用于备份
save 3600 1 300 100 60 10000
```

选择时关注允许丢失多少已确认写入，以及能承受多少延迟：

| 需求 | 配置方向 |
|-|-|
| 可接受快照间隔内的数据丢失，重视简单备份 | RDB |
| 通常希望把断电数据损失控制在约 1 秒 | AOF `everysec` |
| 要求更强的单机落盘确认 | AOF `always`，同时评估尾延迟 |
| 需要紧凑备份、冷存储或快速全量文件 | 保留 RDB |
| 兼顾恢复连续性与加载速度 | AOF + RDB preamble |

持久化不是高可用的替代品。RDB 与 AOF 都可能和实例位于同一故障域；生产环境仍需副本、异机备份、恢复演练，以及对 `rdb_last_bgsave_status`、`aof_last_write_status` 等指标的监控。

---

## 总结

回到最初的命令：

```redis
SET token abc EX 60
```

整条持久化链路可以压缩成：

```text
1. 执行
   SET token abc EX 60
   -> 内存写入 abc
   -> EX 60 计算为绝对时间 1720791060000
   -> 传播命令改写为 SET token abc PXAT 1720791060000
   -> dirty + 1

2. AOF
   propagateNow
   -> feedAppendOnlyFile
   -> RESP 进入 aof_buf
   -> beforeSleep 中 write
   -> 按 always / everysec / no 决定 fsync

3. RDB
   SAVE 或 BGSAVE
   -> 遍历当前 db
   -> 保存 STRING token abc + EXPIRETIME_MS
   -> 临时文件完成后 rename 为 dump.rdb

4. AOF rewrite
   -> fork 子进程生成当前状态的 base
   -> 父进程把新写命令追加到新 incr
   -> manifest 原子切换到新文件集合

5. 恢复
   appendonly yes -> 依次加载 manifest 中的 base + incr
   appendonly no  -> 加载 dump.rdb
```

RDB 与 AOF 的核心区别在于持久化记录的组织方式：

```text
RDB：某个时点的完整状态快照
AOF：base 状态快照（默认使用 RDB 格式）+ base 之后的有序写命令
```

Redis 再用 fork + COW 构造后台快照，用 `aof_buf + write + fsync` 分离命令执行与磁盘同步，并用 `base + incr + manifest` 完成 AOF 的无停机重写。
