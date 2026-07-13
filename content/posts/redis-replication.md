+++
date = '2026-07-11T14:42:00+08:00'
draft = false
title = 'Redis Replication'
tags = ['Redis']
categories = ['数据库']
+++

# Redis 复制

用这条命令看 Redis 复制的数据流：

```bash
SET token abc EX 60
```

它在 master 上执行一次，随后进入复制流。replica 重放的命令会被改写为：

```text
SET token abc PXAT <absolute-millisecond-timestamp>
```

主线：

```text
REPLICAOF -> connectWithMaster -> syncWithMaster -> PSYNC
  -> CONTINUE：补 backlog
  -> FULLRESYNC：加载 RDB 基线

SET token abc EX 60
  -> 改写 PXAT
  -> 写入复制流
  -> backlog + replica 输出缓冲
  -> replica 以 CLIENT_MASTER 身份重放
```

以下代码均为源码截取，只保留影响这条路径的判断和赋值。

---

## 复制历史

部分重同步依赖 `replid + offset + backlog`。

```c
/* src/server.h:redisServer / replBacklog，省略无关字段 */
char replid[CONFIG_RUN_ID_SIZE+1];     // 当前实例对下游暴露的复制历史 ID
char replid2[CONFIG_RUN_ID_SIZE+1];    // 当前实例保留的上一个复制历史 ID
long long master_repl_offset;          // 当前实例对下游输出的复制流末尾 offset
long long second_replid_offset;        // replid2 可接受的最大 offset

replBacklog *repl_backlog;             // 当前实例保存的最近一段复制流
long long repl_backlog_size;           // backlog 目标大小
list *repl_buffer_blocks;              // 保存复制流字节的 buffer blocks

typedef struct replBacklog {
    listNode *ref_repl_buf_node;        // backlog 起点所在 block
    size_t unindexed_count;
    rax *blocks_index;
    long long histlen;                  // backlog 当前保存的字节数
    long long offset;                   // backlog 第一个字节的 offset
} replBacklog;
```

`SET token abc EX 60` 改写并编码成 RESP 后，写入多少字节，`master_repl_offset` 就推进多少。

---

## REPLICAOF 发起连接

replica 执行：

```bash
REPLICAOF 127.0.0.1 6379
```

`strcasecmp` 忽略大小写比较，返回 `0` 表示相等，所以 `!strcasecmp(c->argv[1]->ptr,"no")` 表示参数等于 `no`。

```c
/* src/replication.c:replicaofCommand，省略校验、日志和重复 master 检查 */
void replicaofCommand(client *c) {
    if (!strcasecmp(c->argv[1]->ptr,"no") &&
        !strcasecmp(c->argv[2]->ptr,"one")) {
        if (server.masterhost) replicationUnsetMaster();
    } else {
        long port;

        if (getRangeLongFromObjectOrReply(c, c->argv[2], 0, 65535, &port,
                                          "Invalid master port") != C_OK)
            return;

        // REPLICAOF host port：记录 master 并发起连接
        replicationSetMaster(c->argv[1]->ptr, port);
    }
    addReply(c,shared.ok);
}
```

`replicationSetMaster` 记录 master 地址并进入连接状态。

```c
/* src/replication.c:replicationSetMaster / connectWithMaster，省略清理和事件通知 */
void replicationSetMaster(char *ip, int port) {
    server.masterhost = sdsnew(ip);
    server.masterport = port;

    server.repl_state = REPL_STATE_CONNECT;
    connectWithMaster();
}

int connectWithMaster(void) {
    server.repl_transfer_s = connCreate(server.el, connTypeOfReplication());
    if (connConnect(server.repl_transfer_s, server.masterhost, server.masterport,
                server.bind_source_addr, syncWithMaster) == C_ERR) {
        return C_ERR;
    }

    server.repl_state = REPL_STATE_CONNECTING;
    return C_OK;
}
```

连接层创建非阻塞 socket，拿到 fd 后注册可写事件；连接结果由事件回调继续处理。

```c
/* src/socket.c:connSocketConnect，省略错误记录 */
static int connSocketConnect(connection *conn, const char *addr, int port,
        const char *src_addr, ConnectionCallbackFunc connect_handler) {
    // 返回 fd 表示连接动作已发起
    int fd = anetTcpNonBlockBestEffortBindConnect(NULL,addr,port,src_addr);
    if (fd == -1) return C_ERR;

    conn->fd = fd;
    conn->state = CONN_STATE_CONNECTING;
    conn->conn_handler = connect_handler;

    // socket 可写时进入 syncWithMaster
    aeCreateFileEvent(conn->el, conn->fd, AE_WRITABLE,
            conn->type->ae_handler, conn);

    return C_OK;
}
```

`fd == -1` 表示没有拿到可用 socket。正常连接等待会进入可写事件。

---

## syncWithMaster 推进握手

建连完成后，`syncWithMaster` 按状态推进：

```text
CONNECTING
  -> RECEIVE_PING_REPLY
  -> SEND_HANDSHAKE
  -> RECEIVE_AUTH_REPLY
  -> RECEIVE_PORT_REPLY
  -> RECEIVE_IP_REPLY
  -> RECEIVE_REQ_REPLY
  -> RECEIVE_CAPA_REPLY
  -> SEND_PSYNC
```

```c
/* src/replication.c:syncWithMaster，截取 PING、握手和 PSYNC */
if (server.repl_state == REPL_STATE_CONNECTING) {
    connSetReadHandler(conn, syncWithMaster);
    connSetWriteHandler(conn, NULL);
    // 发 PING，下一次事件读取回复
    server.repl_state = REPL_STATE_RECEIVE_PING_REPLY;
    err = sendCommand(conn,"PING",NULL);
    if (err) goto write_error;
    return;
}

if (server.repl_state == REPL_STATE_SEND_HANDSHAKE) {
    if (server.masterauth) {
        err = sendCommandArgv(conn, argc, args, lens);
        if (err) goto write_error;
    }

    err = sendCommand(conn,"REPLCONF", "listening-port",buf, NULL);
    if (err) goto write_error;

    err = sendCommand(conn,"REPLCONF",
                      "capa","eof","capa","psync2",
                      server.repl_rdb_channel ? "capa" : NULL, "rdb-channel-repl", NULL);
    if (err) goto write_error;

    // 后续 RECEIVE_* 状态消费这些回复
    server.repl_state = REPL_STATE_RECEIVE_AUTH_REPLY;
    return;
}

if (server.repl_state == REPL_STATE_SEND_PSYNC) {
    // 握手完成后请求同步
    if (slaveTryPartialResynchronization(conn,0) == PSYNC_WRITE_ERROR) {
        goto write_error;
    }
    server.repl_state = REPL_STATE_RECEIVE_PSYNC_REPLY;
    return;
}
```

`return` 表示当前事件处理结束。后续 socket 可读或可写时，事件循环再次进入 `syncWithMaster`，根据新的 `server.repl_state` 继续执行。

---

## PSYNC 分流

第一次复制没有历史位置：

```text
PSYNC ? -1
```

断线重连时，replica 使用 cached master 的 `replid`，并请求上次已应用 offset 的下一个字节。

```c
/* src/replication.c:slaveTryPartialResynchronization，截取 PSYNC 写入 */
if (!read_reply) {
    if (server.cached_master) {
        psync_replid = server.cached_master->replid;
        // 请求上次已应用 offset 的下一个字节
        snprintf(psync_offset,sizeof(psync_offset),"%lld", server.cached_master->reploff+1);
    } else {
        psync_replid = "?";
        memcpy(psync_offset,"-1",3);
    }

    reply = sendCommand(conn,"PSYNC",psync_replid,psync_offset,NULL);
    if (reply != NULL) return PSYNC_WRITE_ERROR;
    return PSYNC_WAIT_REPLY;
}
```

master 在 `syncCommand` 中先尝试 partial resync；成功直接返回，失败进入 full resync。

```c
/* src/replication.c:syncCommand，截取 PSYNC 分流和 full resync 入口 */
if (!strcasecmp(c->argv[0]->ptr,"psync")) {
    long long psync_offset;
    if (getLongLongFromObjectOrReply(c, c->argv[2], &psync_offset, NULL) != C_OK) {
        return;
    }

    // 成功时已经写入 +CONTINUE 和缺失 backlog
    if (masterTryPartialResynchronization(c, psync_offset) == C_OK) {
        server.stat_sync_partial_ok++;
        return;
    } else {
        char *master_replid = c->argv[1]->ptr;
        if (master_replid[0] != '?') server.stat_sync_partial_err++;
    }
}

server.stat_sync_full++;

// partial resync 失败后，准备 full resync
c->replstate = SLAVE_STATE_WAIT_BGSAVE_START;
c->repldbfd = -1;
c->flags |= CLIENT_SLAVE;
listAddNodeTail(server.slaves,c);

createReplicationBacklogIfNeeded();
```

partial resync 的判断：复制历史 ID 能接上，并且 offset 还在 backlog 范围内。

```c
/* src/replication.c:masterTryPartialResynchronization，省略日志 */
if (strcasecmp(master_replid, server.replid) &&
    (strcasecmp(master_replid, server.replid2) ||
     psync_offset > server.second_replid_offset))
{
    // 复制历史 ID 接不上
    goto need_full_resync;
}

if (!server.repl_backlog ||
    psync_offset < server.repl_backlog->offset ||
    psync_offset > (server.repl_backlog->offset + server.repl_backlog->histlen))
{
    // 请求的字节不在 backlog 中
    goto need_full_resync;
}

c->flags |= CLIENT_SLAVE;
c->replstate = SLAVE_STATE_ONLINE;
listAddNodeTail(server.slaves,c);

if (c->slave_capa & SLAVE_CAPA_PSYNC2) {
    buflen = snprintf(buf,sizeof(buf),"+CONTINUE %s\r\n", server.replid);
} else {
    buflen = snprintf(buf,sizeof(buf),"+CONTINUE\r\n");
}
if (connWrite(c->conn,buf,buflen) != buflen) {
    freeClientAsync(c);
    return C_OK;
}

// 发送缺失的复制流
psync_len = addReplyReplicationBacklog(c,psync_offset);
return C_OK;

need_full_resync:
// 返回给 syncCommand，继续走 full resync
return C_ERR;
```

`replid2` 是上一个复制历史 ID；`second_replid_offset` 是旧 ID 可接受的最大 offset。ID 检查通过后，还要检查 backlog。请求位置早于 `repl_backlog->offset`，说明对应字节已经被裁掉。

---

## FULLRESYNC 建立数据基线

full resync 需要给 replica 一个完整数据基线。master 复用正在生成的磁盘 RDB，或者启动新的 RDB。

```c
/* src/replication.c:syncCommand，截取 full resync 后续分支 */
if (server.child_type == CHILD_TYPE_RDB &&
    server.rdb_child_type == RDB_CHILD_TYPE_DISK)
{
    if (ln && ((c->slave_capa & slave->slave_capa) == slave->slave_capa) &&
        c->slave_req == slave->slave_req)
    {
        // 复用已有 BGSAVE
        replicationSetupSlaveForFullResync(c,slave->psync_initial_offset);
    }
} else {
    if (!hasActiveChildProcess()) {
        // 启动新的 BGSAVE
        startBgsaveForReplication(c->slave_capa, c->slave_req);
    }
}
```

新 RDB 有两种 target：socket target 直接流式写给 replica；磁盘 target 先生成 `server.rdb_filename`，完成后再发送。这里看磁盘路径。

```c
/* src/replication.c:startBgsaveForReplication，截取 RDB target 和磁盘路径 */
socket_target = (server.repl_diskless_sync || req & SLAVE_REQ_RDB_MASK) && (mincapa & SLAVE_CAPA_EOF);

if (rsiptr) {
    if (socket_target)
        retval = rdbSaveToSlavesSockets(req,rsiptr);
    else {
        // 磁盘路径：子进程生成 server.rdb_filename
        retval = rdbSaveBackground(req, server.rdb_filename, rsiptr,
                RDBFLAGS_REPLICATION | RDBFLAGS_KEEP_CACHE);
    }
}

if (!socket_target) {
    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        client *slave = ln->value;

        if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_START) {
            if (slave->slave_req != req)
                continue;
            // 这个 replica 复用本次磁盘 BGSAVE，先发送 FULLRESYNC 头
            replicationSetupSlaveForFullResync(slave, getPsyncInitialOffset());
        }
    }
}
```

`replicationSetupSlaveForFullResync` 设置初始 offset，并发送 `+FULLRESYNC <replid> <offset>`。

```c
/* src/replication.c:replicationSetupSlaveForFullResync，截取 FULLRESYNC 回复 */
int replicationSetupSlaveForFullResync(client *slave, long long offset) {
    slave->psync_initial_offset = offset;
    slave->replstate = SLAVE_STATE_WAIT_BGSAVE_END;
    server.slaveseldb = -1;

    // 让 replica 记录新的 replid / offset
    buflen = snprintf(buf,sizeof(buf),"+FULLRESYNC %s %lld\r\n",
                      server.replid,offset);
    if (connWrite(slave->conn,buf,buflen) != buflen) {
        freeClientAsync(slave);
        return C_ERR;
    }
    return C_OK;
}
```

磁盘 RDB 生成完成后，`rdb.c` 调用 `updateSlavesWaitingBgsave`。这里的 `slave` 是 master 进程里的 replica client，不是 replica 进程本身。master 打开 RDB 文件，把这个 client 的写 handler 改成 `sendBulkToSlave`；后续 socket 可写时，事件循环调用 `sendBulkToSlave`，从文件读数据并写到 replica socket。

```c
/* src/replication.c:updateSlavesWaitingBgsave / sendBulkToSlave，截取磁盘 RDB 发送 */
if (type != RDB_CHILD_TYPE_SOCKET) {
    // master 侧打开刚生成的 RDB 文件
    slave->repldbfd = open(server.rdb_filename,O_RDONLY);
    slave->repldboff = 0;
    slave->repldbsize = buf.st_size;
    slave->replstate = SLAVE_STATE_SEND_BULK;
    slave->replpreamble = sdscatprintf(sdsempty(),"$%lld\r\n",
        (unsigned long long) slave->repldbsize);

    // 注册写事件；socket 可写时调用 sendBulkToSlave
    connSetWriteHandler(slave->conn,sendBulkToSlave);
}

// sendBulkToSlave 中执行
buflen = read(slave->repldbfd,buf,PROTO_IOBUF_LEN);
nwritten = connWrite(conn,buf,buflen);
slave->repldboff += nwritten;
```

replica 加载 RDB 后，继续消费 RDB 之后的复制流。

---

## SET 改写成 PXAT

客户端写入 master：

```bash
SET token abc EX 60
```

入口解析参数后进入通用写入逻辑。

```c
/* src/t_string.c:setCommand */
void setCommand(client *c) {
    extendedStringArgs args;

    if (parseExtendedStringArgumentsOrReply(c, 3, &args, COMMAND_SET) != C_OK) {
        return;
    }

    c->argv[2] = tryObjectEncoding(c->argv[2]);
    setGenericCommand(c, args.flags, c->argv[1], &(c->argv[2]), args.expire, args.unit, args.match_value, NULL, NULL);
}
```

`EX 60` 是相对 TTL，先换算成绝对毫秒时间戳。

```c
/* src/t_string.c:getExpireMillisecondsOrReply，截取相对 TTL 换算 */
if (unit == UNIT_SECONDS) *milliseconds *= 1000;

if (relative_ttl) {
    // 相对 TTL 转成绝对过期时间
    *milliseconds += commandTimeSnapshot();
}
```

写入数据库后，传播参数从 `EX 60` 改成 `PXAT <absolute-millisecond-timestamp>`。

```c
/* src/t_string.c:setGenericCommand，截取写入和命令改写 */
setKeyByLink(c, c->db, key, valref, setkey_flags, &link);
if (expire) *valref = setExpireByLink(c, c->db, key->ptr, milliseconds, link);

server.dirty++;
notifyKeyspaceEvent(NOTIFY_STRING,"set",key,c->db->id);

if (expire) {
    if (!(flags & OBJ_PXAT)) {
        // 复制/AOF 使用 PXAT，避免 replica 重新计算相对 TTL
        robj *milliseconds_obj = createStringObjectFromLongLong(milliseconds);
        if ((c->cmd->proc == setCommand) && c->argc == 5) {
            rewriteClientCommandArgument(c, 3, shared.pxat);
            rewriteClientCommandArgument(c, 4, milliseconds_obj);
        } else {
            rewriteClientCommandVector(c, 5, shared.set, key, *valref, shared.pxat, milliseconds_obj);
        }
        decrRefCount(milliseconds_obj);
    }
}
```

replica 使用 master 计算出的同一个过期时间点。

---

## 写命令进入复制流

`SET` 修改数据集，`server.dirty` 增加。`call` 根据 dirty 把命令加入传播队列，此时 `argv` 已经是 `PXAT` 版本。

```c
/* src/server.c:call，截取 dirty 判断和传播入队 */
dirty = server.dirty;
long long old_master_repl_offset = server.master_repl_offset;

c->cmd->proc(c);

dirty = server.dirty-dirty;
if (dirty < 0) dirty = 0;

if (flags & CMD_CALL_PROPAGATE &&
    (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP &&
    c->cmd->proc != execCommand &&
    !(c->cmd->flags & CMD_MODULE))
{
    int propagate_flags = PROPAGATE_NONE;

    // SET 修改了数据，传播到 AOF 和 replica
    if (dirty) propagate_flags |= (PROPAGATE_AOF|PROPAGATE_REPL);

    if (propagate_flags != PROPAGATE_NONE)
        alsoPropagate(c->db->id,c->argv,c->argc,propagate_flags);
}
```

传播队列刷出时，复制方向调用 `replicationFeedSlaves`。

```c
/* src/server.c:propagateNow，截取复制方向 */
if (target & PROPAGATE_REPL) {
    // 写入复制流
    replicationFeedSlaves(server.slaves,dbid,argv,argc);
    asmFeedMigrationClient(argv, argc);
}
```

`replicationFeedSlaves` 把命令编码成 RESP。

```c
/* src/replication.c:replicationFeedSlaves，截取命令编码 */
void replicationFeedSlaves(list *slaves, int dictid, robj **argv, int argc) {
    int j, len;
    char aux[LONG_STR_SIZE+3];
    replBufWriter wr;
    replBufWriterBegin(&wr);

    // RESP 数组头：*<argc>\r\n
    replBufWriterAppendBulkLen(&wr, '*', argc);

    for (j = 0; j < argc; j++) {
        long objlen = stringObjectLen(argv[j]);
        replBufWriterAppendBulkLen(&wr, '$', objlen);

        if (argv[j]->encoding == OBJ_ENCODING_INT) {
            len = ll2string(aux, sizeof(aux), (long)argv[j]->ptr);
            replBufWriterAppend(&wr, aux, len);
        } else {
            replBufWriterAppend(&wr, argv[j]->ptr, objlen);
        }
        replBufWriterAppend(&wr, "\r\n", 2);
    }

    replBufWriterEnd(&wr);
}
```

改写后的命令类似：

```text
SET token abc PXAT 1783920000123
```

对应 RESP：

```text
*5\r\n
$3\r\nSET\r\n
$5\r\ntoken\r\n
$3\r\nabc\r\n
$4\r\nPXAT\r\n
$13\r\n1783920000123\r\n
```

`replBufWriterEnd` 推进 master offset，并让在线 replica 和 backlog 引用同一批复制缓冲。

```c
/* src/replication.c:replBufWriterEnd，截取 offset、replica 引用和 backlog 引用 */
static void replBufWriterEnd(replBufWriter *wr) {
    if (wr->total_len == 0) return;

    // offset 按复制流字节数推进
    server.master_repl_offset += wr->total_len;
    server.repl_backlog->histlen += wr->total_len;

    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        client *slave = ln->value;
        if (!canFeedReplicaReplBuffer(slave)) continue;

        if (slave->ref_repl_buf_node == NULL) {
            // 在线 replica 引用这批复制缓冲
            slave->ref_repl_buf_node = wr->start_node;
            slave->ref_block_pos = wr->start_pos;
            ((replBufBlock *)listNodeValue(wr->start_node))->refcount++;
        }
    }

    if (server.repl_backlog->ref_repl_buf_node == NULL) {
        // backlog 也引用这批复制缓冲
        server.repl_backlog->ref_repl_buf_node = wr->start_node;
        ((replBufBlock *)listNodeValue(wr->start_node))->refcount++;
    }
    if (wr->new_blocks) {
        // 按 backlog 大小裁剪旧复制流
        incrementalTrimReplicationBacklog(REPL_BACKLOG_TRIM_BLOCKS_PER_CALL);
    }
}
```

同一批 RESP 字节同时用于在线复制和断线续传。

---

## replica 重放命令

full resync 结束后，replica 把 master 连接变成特殊 client。读路径仍是普通命令读取路径，身份是 `CLIENT_MASTER`。

```c
/* src/replication.c:replicationCreateMasterClient，省略无关字段 */
void replicationCreateMasterClient(connection *conn, int dbid) {
    server.master = createClient(conn);
    if (conn)
        connSetReadHandler(server.master->conn, readQueryFromClient);

    // 这个 client 代表上游 master
    server.master->flags |= CLIENT_MASTER;

    server.master->authenticated = 1;
    // 初始 offset 来自 FULLRESYNC/CONTINUE
    server.master->reploff = server.master_initial_offset;
    server.master->read_reploff = server.master->reploff;
    memcpy(server.master->replid, server.master_replid,
        sizeof(server.master_replid));
}
```

只读 replica 拒绝普通客户端写入；来自 master 的命令带 `CLIENT_MASTER`，可以执行。

```c
/* src/server.c:mustObeyClient / processCommand，截取只读 replica 写入校验 */
int mustObeyClient(client *c) {
    // 来自 master 的复制命令绕过 replica-read-only 限制
    return c->id == CLIENT_ID_AOF || c->flags & CLIENT_MASTER;
}

int obey_client = mustObeyClient(c);

if (server.masterhost && server.repl_slave_ro &&
    !obey_client && is_write_command)
{
    rejectCommand(c,shared.roslaveerr);
    return C_OK;
}
```

命令执行后，replica 更新已应用 offset。链式复制时，这些已执行字节继续写入本节点 backlog 和下游 replica。

```c
/* src/networking.c:commandProcessed，截取 master client offset 更新 */
void commandProcessed(client *c) {
    if (c->flags & CLIENT_BLOCKED) return;

    prepareForNextCommand(c, 1);

    // 当前命令执行前，replica 已应用到的位置
    long long prev_offset = c->reploff;
    if (c->flags & CLIENT_MASTER && !(c->flags & CLIENT_MULTI)) {
        serverAssert(c->reploff_next > 0);
        // 当前命令执行完成后推进 offset
        c->reploff = c->reploff_next;
    }

    if (c->flags & CLIENT_MASTER) {
        long long applied = c->reploff - prev_offset;
        if (applied) {
            // 链式复制：继续转发已执行字节
            replicationFeedStreamFromMasterStream(c->querybuf+c->repl_applied,applied);
            c->repl_applied += applied;
        }
    }
}
```

replica 重放的命令是：

```text
SET token abc PXAT <absolute-millisecond-timestamp>
```

---

## 断线后续传

master 连接断开后，replica 保留本地数据，回到连接状态，并重新走 `PSYNC`。

```c
/* src/replication.c:replicationHandleMasterDisconnection，省略事件通知和 rdb-channel 清理 */
void replicationHandleMasterDisconnection(void) {
    server.master = NULL;
    if (server.repl_state == REPL_STATE_CONNECTED)
        server.repl_current_sync_attempts = 0;
    // 保留本地数据，回到连接状态
    server.repl_state = REPL_STATE_CONNECT;
    server.repl_down_since = server.unixtime;
    server.repl_up_since = 0;

    if (server.masterhost) {
        // 重连后继续走 PSYNC
        connectWithMaster();
    }
}
```

```text
PSYNC <old-replid> <old-offset+1>

replid 可接受，offset 在 backlog 中：
  +CONTINUE
  补发缺失字节

其他情况：
  +FULLRESYNC <replid> <offset>
  重新加载 RDB
```

backlog 覆盖窗口由复制流写入速度决定：

```text
可续传时间 ≈ repl-backlog-size / 复制流写入速度
```

---

## 总结

`SET token abc EX 60` 的复制路径：

```text
REPLICAOF
  -> connectWithMaster
  -> syncWithMaster
  -> PSYNC
      -> CONTINUE：补 backlog
      -> FULLRESYNC：加载 RDB 基线

SET token abc EX 60
  -> setGenericCommand 改写成 PXAT <绝对毫秒时间戳>
  -> call 根据 dirty 入队传播
  -> replicationFeedSlaves 编码 RESP
  -> replBufWriterEnd 推进 master_repl_offset，写入 replica/backlog
  -> replica 以 CLIENT_MASTER 身份重放命令
```

RDB 提供数据基线；复制流提供基线之后的有序写入；`replid + offset + backlog` 提供断线后的续传条件。普通写命令返回客户端时不等待所有 replica 执行完成，Redis 复制默认是异步复制。
