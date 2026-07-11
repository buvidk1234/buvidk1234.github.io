+++
date = '2026-07-11T14:41:42+08:00'
draft = false
title = 'Redis Command Execution'
tags = ['Redis']
categories = ['数据库']
+++

# Redis 命令执行

这篇看一条命令从客户端发到 Redis 后，源码里大概怎么走。例子还是：

```bash
SET name alice
```

客户端一般发的是 RESP：

```text
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$5\r\nalice\r\n
```

主线可以压成这一条：

```text
readQueryFromClient
  -> processInputBuffer
  -> processMultibulkBuffer / processInlineBuffer
  -> preprocessCommand
  -> processCommandAndResetClient
  -> processCommand
  -> call
  -> setCommand
  -> addReply
  -> handleClientsWithPendingWrites / writeToClient
```

主要源码：

| 环节 | 源码 |
|-|-|
| 读请求、协议解析、回写 | `src/networking.c` |
| 查命令、校验、调度 | `src/server.c` / `src/server.h` |
| 命令声明 | `src/commands/*.json` / `src/commands.def` |
| `SET` 实现 | `src/t_string.c` |
| 写入数据库 | `src/db.c` |

---

## client：一条连接的执行现场

先看 `server.h` 里的 `client`，删掉大量无关字段后是这样：

```c
typedef struct client {
    connection *conn;       /* socket/连接抽象 */
    redisDb *db;            /* 当前 SELECT 到的 DB */

    sds querybuf;           /* socket 读进来的原始字节 */
    size_t qb_pos;          /* querybuf 解析到哪里 */

    int argc;               /* 当前命令参数个数 */
    robj **argv;            /* 当前命令参数对象 */

    pendingCommandList pending_cmds;  /* 已解析、待执行的命令 */
    pendingCommand *current_pending_cmd;

    struct redisCommand *cmd;       /* 当前命令 */
    struct redisCommand *lastcmd;   /* 上一条命令，用于复用查表结果 */
    struct redisCommand *lookedcmd; /* lookahead 阶段提前查到的命令 */
    struct redisCommand *realcmd;   /* 真正用于统计/错误的命令 */

    int reqtype;            /* inline 还是 multibulk */
    int multibulklen;       /* RESP 数组还剩几个 bulk */
    long bulklen;           /* 当前 bulk 的长度 */

    list *reply;            /* 大回复链表 */
    size_t sentlen;         /* 当前回复已经写出去多少 */
} client;
```

所以 Redis 不是每来一条命令就创建一个执行对象，而是复用连接上的 `client`：

```text
querybuf 保存字节
pending_cmds 保存解析好的命令
argc/argv 指向当前正在执行的那条命令
cmd 指向命令表里的 redisCommand
reply 保存待写回的 RESP 回复
```

---

## 读请求：connRead 追加到 querybuf

客户端可读时，事件循环会调用 `networking.c:readQueryFromClient`。核心逻辑删减后是：

```c
void readQueryFromClient(connection *conn) {
    client *c = connGetPrivateData(conn);

    /* 1. 给 querybuf 预留空间 */
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);

    /* 2. 从 socket 读字节，追加到 querybuf 尾部 */
    qblen = sdslen(c->querybuf);
    nread = connRead(c->conn, c->querybuf + qblen, readlen);
    if (nread <= 0) {
        freeClientAsync(c);
        goto done;
    }
    sdsIncrLen(c->querybuf, nread);

    /* 3. querybuf 里可能已经有完整命令，继续解析/执行 */
    if (processInputBuffer(c) == C_ERR)
        c = NULL;

done:
    beforeNextClient(c);
}
```

这里的关键是：`connRead` 只保证读到“一段字节”，不保证刚好是一条完整命令。

比如第一次只读到：

```text
*3\r\n$3\r\nSET\r\n$4\r\nna
```

这时 `processMultibulkBuffer` 发现 bulk 还没收齐，会把状态留在 `client` 里，下次继续读到：

```text
me\r\n$5\r\nalice\r\n
```

才能凑成完整的 `argv`。

---

## 解析请求：querybuf 变成 pendingCommand

新版本 Redis 不是解析完马上执行，而是先把命令放到 `pending_cmds`。`processInputBuffer` 的骨架是：

```c
int processInputBuffer(client *c) {
    const int lookahead = authRequired(c) ? 1 : server.lookahead;

    while (querybuf 还有数据 || pending_cmds 里有 ready 命令) {
        /* 1. 先解析，最多向前看 lookahead 条命令 */
        while (c->pending_cmds.ready_len < lookahead &&
               c->querybuf && c->qb_pos < sdslen(c->querybuf))
        {
            if (!c->reqtype) {
                c->reqtype = c->querybuf[c->qb_pos] == '*'
                           ? PROTO_REQ_MULTIBULK
                           : PROTO_REQ_INLINE;
            }

            pendingCommand *pcmd = acquirePendingCommand();

            if (c->reqtype == PROTO_REQ_INLINE)
                processInlineBuffer(c, pcmd);
            else
                processMultibulkBuffer(c, pcmd);

            addPendingCommand(&c->pending_cmds, pcmd);

            /* 这里会提前查命令、检查参数数量、算 key slot */
            preprocessCommand(c, pcmd);
            resetClientQbufState(c);
        }

        /* 2. 取出队首 ready command，塞回老的 c->argc/c->argv 字段 */
        pendingCommand *curcmd = c->pending_cmds.head;
        c->argc = curcmd->argc;
        c->argv = curcmd->argv;
        c->slot = curcmd->slot;
        c->lookedcmd = curcmd->cmd;
        c->current_pending_cmd = curcmd;

        /* 3. IO 线程只负责读/解析，命令执行要回主线程 */
        if (c->running_tid != IOTHREAD_MAIN_THREAD_ID) {
            c->io_flags |= CLIENT_IO_PENDING_COMMAND;
            enqueuePendingClientsToMainThread(c, 0);
            break;
        }

        processCommandAndResetClient(c);
    }
}
```

对于 `SET name alice`，RESP 解析后得到的不是字符串数组，而是 Redis 对象数组：

```c
c->argc = 3;
c->argv[0] = createStringObject("SET", 3);
c->argv[1] = createStringObject("name", 4);
c->argv[2] = createStringObject("alice", 5);
```

这一步之后，命令实现函数就不用再关心 socket、`\r\n`、bulk 长度这些协议细节了。

---

## preprocessCommand：提前查命令和 key

`server.c:preprocessCommand` 会在真正执行前做一轮轻量预处理：

```c
void preprocessCommand(client *c, pendingCommand *pcmd) {
    struct redisCommand *last_cmd =
        pcmd->prev ? pcmd->prev->cmd : c->lastcmd;

    /* pipeline 里连续多条同名命令，可以复用上一次查表结果 */
    if (isCommandReusable(last_cmd, pcmd->argv[0]))
        pcmd->cmd = last_cmd;
    else
        pcmd->cmd = lookupCommand(pcmd->argv, pcmd->argc);

    if (!pcmd->cmd) {
        pcmd->read_error = CLIENT_READ_COMMAND_NOT_FOUND;
        return;
    }

    /* arity > 0 表示必须等于；arity < 0 表示至少 -arity 个参数 */
    if ((pcmd->cmd->arity > 0 && pcmd->cmd->arity != pcmd->argc) ||
        (pcmd->argc < -pcmd->cmd->arity))
    {
        pcmd->read_error = CLIENT_READ_BAD_ARITY;
        return;
    }

    /* 找出 key 参数，并顺手算 cluster slot */
    int num_keys = extractKeysAndSlot(pcmd->cmd, pcmd->argv, pcmd->argc,
                                      &pcmd->keys_result, &pcmd->slot);
    if (num_keys > 0 && pcmd->slot == CLUSTER_CROSSSLOT) {
        pcmd->read_error = CLIENT_READ_CROSS_SLOT;
        pcmd->slot = INVALID_CLUSTER_SLOT;
    }
}
```

所以很多错误并不是到了 `setCommand` 才发现的，比如命令不存在、参数数量不对、Cluster 多 key 跨 slot，前面就已经标记好了。

---

## 命令表：SET 怎么变成 setCommand

`SET` 的声明在 `src/commands/set.json` 里，删减后是：

```json
{
  "SET": {
    "arity": -3,
    "function": "setCommand",
    "command_flags": ["WRITE", "DENYOOM"],
    "key_specs": [{
      "begin_search": { "index": { "pos": 1 } },
      "find_keys": { "range": { "lastkey": 0, "step": 1 } }
    }]
  }
}
```

生成到 `commands.def` 后，大概就是：

```c
MAKE_CMD("set", ..., setCommand, -3,
         CMD_WRITE | CMD_DENYOOM,
         ACL_CATEGORY_STRING,
         SET_Keyspecs, 1, setGetKeys, ...)
```

Redis 启动时把这些声明灌进 `server.commands` 字典。查表函数很直接：

```c
struct redisCommand *lookupCommandLogic(dict *commands,
                                        robj **argv, int argc, int strict)
{
    struct redisCommand *base_cmd =
        dictFetchValue(commands, argv[0]->ptr);

    if (argc == 1 || !base_cmd->subcommands_dict)
        return base_cmd;

    return lookupSubcommand(base_cmd, argv[1]->ptr);
}

struct redisCommand *lookupCommand(robj **argv, int argc) {
    return lookupCommandLogic(server.commands, argv, argc, 0);
}
```

也就是说：

```text
argv[0] = "SET"
dictFetchValue(server.commands, "SET")
  -> redisCommand {
       proc  = setCommand,
       arity = -3,
       flags = CMD_WRITE | CMD_DENYOOM,
       key_specs = 第 1 个参数是 key
     }
```

命令分发不是 `if (cmd == "SET")`，而是查字典拿函数指针。

---

## processCommand：统一拦截层

`processCommand` 很长，但主干可以删成这样：

```c
int processCommand(client *c) {
    /* 1. 拿到命令。通常 preprocessCommand 已经查好了 */
    struct redisCommand *cmd = c->lookedcmd;
    if (!cmd)
        cmd = lookupCommand(c->argv, c->argc);

    c->cmd = c->lastcmd = c->realcmd = cmd;

    /* 2. 最基础的错误：命令不存在、参数数量不对 */
    if (!commandCheckExistence(c, &err)) {
        rejectCommandSds(c, err);
        return C_OK;
    }
    if (!commandCheckArity(c->cmd, c->argc, &err)) {
        rejectCommandSds(c, err);
        return C_OK;
    }

    /* 3. 根据命令 flags 得到读/写/OOM/loading 等属性 */
    const uint64_t cmd_flags = getCommandFlags(c);
    int is_write_command = cmd_flags & CMD_WRITE;
    int is_denyoom_command = cmd_flags & CMD_DENYOOM;

    /* 4. 通用校验：这些和 SET 自己的业务逻辑无关 */
    if (authRequired(c)) rejectCommand(c, shared.noautherr);
    if (ACLCheckAllPerm(c, &acl_errpos) != ACL_OK)
        rejectCommandFormat(c, "-NOPERM ...");
    if (server.cluster_enabled && cluster_redirection_needed)
        clusterRedirectClient(c, n, c->slot, error_code);
    if (server.maxmemory && performEvictions() == EVICT_FAIL &&
        is_denyoom_command) rejectCommand(c, shared.oomerr);
    if (writeCommandsDeniedByDiskError() && is_write_command)
        rejectCommandSds(c, err);
    if (server.masterhost && server.repl_slave_ro && is_write_command)
        rejectCommand(c, shared.roslaveerr);
    if (server.loading && is_denyloading_command)
        rejectCommand(c, shared.loadingerr);

    /* 5. MULTI 里先排队，否则真正执行 */
    if ((c->flags & CLIENT_MULTI) && should_queue_for_exec) {
        queueMultiCommand(c, cmd_flags);
        addReply(c, shared.queued);
    } else {
        call(c, CMD_CALL_FULL);
    }

    return C_OK;
}
```

所以 `SET name alice` 真正到 `setCommand` 之前，已经过了这些门：

```text
命令存在？
参数至少 3 个？
当前连接已认证？
ACL 允许执行 SET、访问 key=name？
Cluster slot 属于本节点？
写命令在当前内存、磁盘、主从状态下允许执行？
如果在 MULTI 中，是排队还是执行？
```

这就是为什么具体命令函数通常很薄：大量通用规则都在 `processCommand` 里统一处理了。

---

## call：真正调用函数指针

`server.c:call` 里最关键的一行就是 `c->cmd->proc(c)`：

```c
void call(client *c, int flags) {
    long long dirty = server.dirty;
    struct redisCommand *real_cmd = c->realcmd;

    c->flags &= ~(CLIENT_FORCE_AOF |
                  CLIENT_FORCE_REPL |
                  CLIENT_PREVENT_PROP);

    enterExecutionUnit(1, call_timer);
    c->flags |= CLIENT_EXECUTING_COMMAND;

    /* 真正的命令函数调用 */
    c->cmd->proc(c);

    exitExecutionUnit();

    /* 命令是否改了数据集 */
    dirty = server.dirty - dirty;
    if (dirty < 0) dirty = 0;

    slowlogPushCurrentCommand(c, real_cmd, c->duration);
    replicationFeedMonitors(c, server.monitors, c->db->id, c->argv, c->argc);

    real_cmd->calls++;
    real_cmd->microseconds += c->duration;

    /* 写命令真的改了数据，才传播到 AOF / replica */
    if (flags & CMD_CALL_PROPAGATE) {
        int propagate_flags = PROPAGATE_NONE;
        if (dirty) propagate_flags |= PROPAGATE_AOF | PROPAGATE_REPL;
        if (propagate_flags != PROPAGATE_NONE)
            alsoPropagate(c->db->id, c->argv, c->argc, propagate_flags);
    }
}
```

`call` 负责的是“执行框架”：计时、慢日志、MONITOR、命令统计、AOF/复制传播。具体 `SET` 怎么写 key，不在这里。

---

## SET：真正写数据的地方

`SET` 的入口在 `t_string.c:setCommand`：

```c
void setCommand(client *c) {
    extendedStringArgs args;

    /* 解析 NX / XX / GET / EX / PX / KEEPTTL 等参数 */
    if (parseExtendedStringArgumentsOrReply(c, 3, &args, COMMAND_SET) != C_OK)
        return;

    /* 尝试把 value 编码成 int/embstr/raw 等更合适的对象 */
    c->argv[2] = tryObjectEncoding(c->argv[2]);

    setGenericCommand(c, args.flags,
                      c->argv[1],       /* key */
                      &(c->argv[2]),    /* value */
                      args.expire,
                      args.unit,
                      args.match_value,
                      NULL, NULL);
}
```

`SET name alice` 进入这里时：

```text
c->argv[0] = "SET"
c->argv[1] = "name"
c->argv[2] = "alice"
args.flags = 0
args.expire = NULL
```

如果是：

```bash
SET name tom NX EX 60
```

进入 `setGenericCommand` 时就会变成：

```text
key = "name"
value = "tom"
flags 包含 OBJ_SET_NX | OBJ_EX
expire = "60"
unit = UNIT_SECONDS
```

---

## setGenericCommand：条件判断 + 写入 DB

`setGenericCommand` 才是 `SET` 的核心，删减后：

```c
void setGenericCommand(client *c, int flags,
                       robj *key, robj **valref,
                       robj *expire, int unit,
                       robj *match_value,
                       robj *ok_reply, robj *abort_reply)
{
    long long milliseconds = 0;
    int relative_ttl = (flags & (OBJ_EX | OBJ_PX)) != 0;

    /* 1. 过期时间先转成毫秒时间戳 */
    if (expire &&
        getExpireMillisecondsOrReply(c, expire, relative_ttl,
                                     unit, &milliseconds) != C_OK)
        return;

    /* 2. 先查 key 是否存在，同时拿到 dictEntryLink，后面写入少查一次 */
    dictEntryLink link = NULL;
    int found = lookupKeyWriteWithLink(c->db, key, &link) != NULL;

    /* 3. NX / XX 这种条件不满足，直接回复 null，不写 DB */
    if ((flags & OBJ_SET_NX && found) ||
        (flags & OBJ_SET_XX && !found))
    {
        addReply(c, abort_reply ? abort_reply : shared.null[c->resp]);
        return;
    }

    /* 4. 真正写入 */
    int setkey_flags = 0;
    setkey_flags |= expire ? SETKEY_KEEPTTL : 0;
    setkey_flags |= found ? SETKEY_ALREADY_EXIST : SETKEY_DOESNT_EXIST;

    setKeyByLink(c, c->db, key, valref, setkey_flags, &link);

    /* 5. 如果带 EX/PX，再维护 expires */
    if (expire)
        *valref = setExpireByLink(c, c->db, key->ptr, milliseconds, link);

    incrRefCount(*valref);
    server.dirty++;
    notifyKeyspaceEvent(NOTIFY_STRING, "set", key, c->db->id);

    addReply(c, ok_reply ? ok_reply : shared.ok);
}
```

第一次执行：

```bash
SET name alice
```

大致路径是：

```text
found = false
setkey_flags = SETKEY_DOESNT_EXIST
setKeyByLink -> dbAddByLink
server.dirty++
addReply("+OK\r\n")
```

如果再执行：

```bash
SET name bob
```

路径变成：

```text
found = true
setkey_flags = SETKEY_ALREADY_EXIST
setKeyByLink -> dbSetValue
server.dirty++
addReply("+OK\r\n")
```

如果执行：

```bash
SET name tom NX
```

因为 `name` 已存在：

```text
found = true
flags contains OBJ_SET_NX
直接 addReply(null)
不调用 setKeyByLink
server.dirty 不增加
```

---

## setKeyByLink：新增还是覆盖

最终写 DB 的逻辑在 `db.c:setKeyByLink`：

```c
void setKeyByLink(client *c, redisDb *db,
                  robj *key, robj **valref,
                  int flags, dictEntryLink *plink)
{
    int exists;
    kvobj *oldval = NULL;

    if (flags & SETKEY_ALREADY_EXIST) {
        oldval = dictGetKV(**plink);
        exists = 1;
    } else if (flags & SETKEY_DOESNT_EXIST) {
        exists = 0;
    } else {
        oldval = lookupKeyWriteWithLink(db, key, link);
        exists = oldval != NULL;
    }

    if (exists) {
        /* 覆盖旧 value */
        dbSetValue(db, key, valref, *link, 1, 1, flags & SETKEY_KEEPTTL);
        notifyKeyspaceEvent(NOTIFY_OVERWRITTEN, "overwritten", key, db->id);
    } else {
        /* 新增 key */
        dbAddByLink(db, key, valref, link);
    }

    keyModified(c, db, key, *valref, !(flags & SETKEY_NO_SIGNAL));
}
```

可以把 Redis DB 简化成两张核心表：

```text
db->keys:
  "name" -> redisObject("alice")

db->expires:
  "name" -> expire timestamp   // 只有带过期时间的 key 才有
```

`SET name alice` 默认只写 `db->keys`。`SET name alice EX 60` 会先写 `db->keys`，再通过 `setExpireByLink` 写 `db->expires`。

---

## addReply：命令只生成回复，不直接 write

`setGenericCommand` 最后调用的是：

```c
addReply(c, shared.ok);
```

`networking.c:addReply` 的骨架：

```c
void addReply(client *c, robj *obj) {
    if (_prepareClientToWrite(c) != C_OK)
        return;

    if (sdsEncodedObject(obj)) {
        _addReplyToBufferOrList(c, obj->ptr, sdslen(obj->ptr));
    } else if (obj->encoding == OBJ_ENCODING_INT) {
        char buf[32];
        size_t len = ll2string(buf, sizeof(buf), (long)obj->ptr);
        _addReplyToBufferOrList(c, buf, len);
    }
}
```

再往下一层：

```c
void _addReplyToBufferOrList(client *c, const char *s, size_t len) {
    /* 小回复先进固定 buffer */
    size_t reply_len = _addReplyPayloadToBuffer(c, s, len, PLAIN_REPLY);

    /* 固定 buffer 放不下的部分，进 c->reply 链表 */
    if (len > reply_len)
        _addReplyPayloadToList(c, c->reply,
                               s + reply_len, len - reply_len,
                               PLAIN_REPLY);
}
```

所以命令函数只是把 `+OK\r\n` 放进输出缓冲区，并不直接写 socket。

真正写回在事件循环后半段：

```c
int handleClientsWithPendingWrites(void) {
    while ((ln = listNext(&li))) {
        client *c = listNodeValue(ln);

        if (writeToClient(c, 0) == C_ERR)
            continue;

        if (clientHasPendingReplies(c))
            installClientWriteHandler(c);
    }
}
```

`writeToClient` 会循环把 `client.buf` / `client.reply` 写到连接里：

```c
int writeToClient(client *c, int handler_installed) {
    while (_clientHasPendingRepliesNonSlave(c)) {
        int ret = _writeToClientNonSlave(c, &nwritten);
        if (ret == C_ERR) break;

        /* 单次写太多就让出事件循环，避免大回复拖死其他客户端 */
        if (totwritten > NET_MAX_WRITES_PER_EVENT &&
            (server.maxmemory == 0 ||
             zmalloc_used_memory() < server.maxmemory) &&
            is_normal_client)
            break;
    }

    if (!clientHasPendingReplies(c) && handler_installed)
        connSetWriteHandler(c->conn, NULL);
}
```

这就是 Redis 回复链路的分工：

```text
命令函数：addReply，生成 RESP 回复
网络层：writeToClient，把回复刷到 socket
事件循环：一次写不完就注册写事件，下次可写继续发
```

---

## Pipeline 怎么落到源码上

Pipeline 只是一次发多条命令：

```text
SET a 1
INCR a
GET a
```

源码上对应的是：

```c
while (c->pending_cmds.ready_len < lookahead &&
       c->querybuf && c->qb_pos < sdslen(c->querybuf))
{
    processMultibulkBuffer(c, pcmd);
    addPendingCommand(&c->pending_cmds, pcmd);
    preprocessCommand(c, pcmd);
}
```

也就是一次 `read` 后，Redis 可以连续解析出多条 `pendingCommand`：

```text
pending_cmds:
  [SET a 1] [INCR a] [GET a]
```

但执行时还是队首一条一条来：

```text
curcmd = pending_cmds.head
c->argc/c->argv = curcmd->argc/curcmd->argv
processCommandAndResetClient(c)
```

Pipeline 提升的是网络往返和系统调用效率，不是把三条命令并行执行。

---

## I/O 线程到底做什么

`processInputBuffer` 里有一段很能说明问题：

```c
if (c->running_tid != IOTHREAD_MAIN_THREAD_ID) {
    c->io_flags |= CLIENT_IO_PENDING_COMMAND;
    enqueuePendingClientsToMainThread(c, 0);
    break;
}

processCommandAndResetClient(c);
```

意思是：I/O 线程可以读 socket、解析协议、准备 pending command；但真正修改数据库的 `processCommand -> call -> setCommand` 还是回主线程做。

所以 Redis 的“单线程”更准确地说是：

```text
网络读写：可以由 I/O 线程分担
命令执行、数据结构修改、过期淘汰、传播：主线程串行处理
```

---

## 总结

把 `SET name alice` 串起来就是：

```text
readQueryFromClient
  connRead 读到 RESP 字节，追加到 c->querybuf

processInputBuffer
  processMultibulkBuffer 把 RESP 拆成 pendingCommand(argc=3, argv=[SET,name,alice])
  preprocessCommand 提前查到 redisCommand(setCommand)，检查 arity，提取 key/slot
  把 pendingCommand 填回 c->argc/c->argv/c->lookedcmd

processCommand
  统一做认证、ACL、Cluster、OOM、磁盘错误、主从状态、loading、事务等检查

call
  记录 dirty/耗时，调用 c->cmd->proc(c)，执行后做慢日志、统计、AOF/复制传播

setCommand
  解析 SET 参数，调用 setGenericCommand

setGenericCommand
  lookupKeyWriteWithLink 判断 key 是否存在
  根据 NX/XX/EX/PX 等条件决定是否写入
  setKeyByLink 新增或覆盖 db->keys
  必要时写 db->expires
  server.dirty++
  addReply(shared.ok)

handleClientsWithPendingWrites / writeToClient
  把 client 输出缓冲区里的 +OK\r\n 写回 socket
```

这条链路里，`SET` 自己只负责字符串命令的业务规则；协议解析、命令查表、通用校验、慢日志、统计、AOF/复制、网络回写，都在 Redis 的命令执行框架里统一完成。
