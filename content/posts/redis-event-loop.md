+++
date = '2026-07-11T14:42:00+08:00'
draft = false
title = 'Redis Event Loop'
tags = ['Redis']
categories = ['数据库']
+++

# Redis 事件循环

这篇不按 API 一个个介绍，而是跟着一个具体请求读源码。先固定场景：

```text
系统：Linux，使用 epoll
连接：普通 TCP，不考虑 TLS
线程：io-threads = 1

监听 socket：fd = 6
客户端 socket：fd = 8
客户端发送：SET name alice
```

要追的不是一堆互相独立的函数，而是下面两次唤醒：

```text
第一次 epoll_wait 返回 fd=6 可读
  -> accept 新连接
  -> 得到 fd=8
  -> 给 fd=8 注册读回调

第二次 epoll_wait 返回 fd=8 可读
  -> 读请求
  -> 执行 SET
  -> 生成 +OK
  -> 下一轮 beforeSleep 写回客户端
```

主要源码按这个顺序看：

```text
src/server.c
  main -> initServer -> initListeners -> aeMain

src/ae.c / src/ae_epoll.c
  aeCreateFileEvent -> epoll_ctl
  aeProcessEvents -> epoll_wait -> 调用回调

src/socket.c / src/networking.c
  connSocketAcceptHandler -> acceptCommonHandler -> createClient
  connSocketEventHandler -> readQueryFromClient
  addReply -> handleClientsWithPendingWrites -> writeToClient
```

---

## 先认清 events 和 fired

`aeEventLoop` 里最重要的是两张表：

```c
/* ae.h，省略其他字段 */
typedef struct aeEventLoop {
    aeFileEvent *events;  /* 已注册的事件，以 fd 为下标 */
    aeFiredEvent *fired;  /* 本轮 epoll_wait 返回的就绪事件 */
    aeTimeEvent *timeEventHead;
    int maxfd;
    int stop;
    void *apidata;        /* Linux 下指向 aeApiState */
} aeEventLoop;

typedef struct aeFileEvent {
    int mask;
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;

typedef struct aeFiredEvent {
    int fd;
    int mask;
} aeFiredEvent;
```

二者不要混在一起：

```text
events[fd]：Redis 长期保存的“fd 就绪后调用谁”
fired[i] ：内核本轮返回的“哪个 fd 已经就绪”
```

例如监听 fd 注册完成后：

```text
events[6] = {
    mask       = AE_READABLE,
    rfileProc  = connSocketAcceptHandler,
    clientData = listener
}
```

有客户端连接时，`epoll_wait` 只返回：

```text
fired[0] = {
    fd   = 6,
    mask = AE_READABLE
}
```

Redis 再用 `fd=6` 回查 `events[6]`，才知道应该调用
`connSocketAcceptHandler`。整个事件分发的核心就是这次回查：

```c
int fd = eventLoop->fired[j].fd;
aeFileEvent *fe = &eventLoop->events[fd];
fe->rfileProc(eventLoop, fd, fe->clientData, mask);
```

---

## 启动：先把回调注册好

从 `server.c:main` 往下看，只保留事件循环相关代码：

```c
int main(int argc, char **argv) {
    /* 1. 创建 server.el，注册 serverCron 和睡眠钩子 */
    initServer();

    /* 2. 创建监听 socket，并注册 accept 回调 */
    initListeners();

    /* 3. 初始化完成后进入循环 */
    aeMain(server.el);

    aeDeleteEventLoop(server.el);
    return 0;
}
```

### initServer 创建 eventLoop

`server.el` 是整个主线程共用的事件循环对象：

```c
/* server.c:initServer */
server.el = aeCreateEventLoop(
    server.maxclients + CONFIG_FDSET_INCR
);

/* 1ms 后第一次执行 serverCron */
aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);

/* 每次 epoll_wait 前后执行 */
aeSetBeforeSleepProc(server.el, beforeSleep);
aeSetAfterSleepProc(server.el, afterSleep);
```

Linux 下 `aeCreateEventLoop` 还会进入 `ae_epoll.c:aeApiCreate`：

```c
static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state = zmalloc(sizeof(*state));

    state->epfd = epoll_create(1024);
    state->events = zmalloc(
        sizeof(struct epoll_event) * eventLoop->setsize
    );

    eventLoop->apidata = state;
    return 0;
}
```

这里有两组容易重名的数组：

```text
eventLoop->events       Redis 的注册表，元素是 aeFileEvent
eventLoop->fired        Redis 整理后的本轮就绪表，元素是 aeFiredEvent

state->events           epoll_wait 的原始输出，元素是 struct epoll_event
```

### initListeners 注册监听 fd

`initListeners` 创建监听 socket 后，把它的 fd 注册成可读：

```c
/* server.c:initListeners，省略错误处理 */
connListen(listener);

createSocketAcceptHandler(
    listener,
    connAcceptHandler(listener->ct)
);
```

普通 TCP 的 `connAcceptHandler(listener->ct)` 最终取到：

```c
/* socket.c */
static ConnectionType CT_Socket = {
    .accept_handler = connSocketAcceptHandler,
    .ae_handler = connSocketEventHandler,
    .set_read_handler = connSocketSetReadHandler,
    .set_write_handler = connSocketSetWriteHandler,
    /* 其余成员省略 */
};
```

所以 `createSocketAcceptHandler` 实际注册的是：

```c
/* 假设 sfd->fd[0] == 6 */
aeCreateFileEvent(
    server.el,
    6,
    AE_READABLE,
    connSocketAcceptHandler,
    listener
);
```

`aeCreateFileEvent` 同时更新 Redis 和 epoll 两边的状态：

```c
/* ae.c */
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
                      aeFileProc *proc, void *clientData)
{
    aeFileEvent *fe = &eventLoop->events[fd];

    /* 通知内核监听这个 fd */
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;

    /* Redis 自己记住回调 */
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;

    return AE_OK;
}
```

`aeApiAddEvent` 在 Linux 下就是 `epoll_ctl`：

```c
/* ae_epoll.c */
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0};

    int op = eventLoop->events[fd].mask == AE_NONE
           ? EPOLL_CTL_ADD
           : EPOLL_CTL_MOD;

    mask |= eventLoop->events[fd].mask;
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;

    return epoll_ctl(state->epfd, op, fd, &ee);
}
```

到这里，用户态和内核各保存一份信息：

```text
内核 epoll：
  fd=6，关注 EPOLLIN

Redis events[6]：
  fd=6 可读时，调用 connSocketAcceptHandler(listener)
```

注册结束后才进入 `aeMain`。

---

## aeMain：一轮循环到底怎么走

`aeMain` 本身只有一个 `while`：

```c
/* ae.c */
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;

    while (!eventLoop->stop) {
        aeProcessEvents(
            eventLoop,
            AE_ALL_EVENTS |
            AE_CALL_BEFORE_SLEEP |
            AE_CALL_AFTER_SLEEP
        );
    }
}
```

`aeProcessEvents` 才是一轮循环。下面保留了决定调用顺序的分支：

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    struct timeval tv, *tvp = NULL;

    /* 1. 进入 epoll_wait 前，处理上一轮留下的工作 */
    eventLoop->beforesleep(eventLoop);

    /* 2. 最近的定时器还有多久到期，就最多阻塞多久 */
    if ((flags & AE_DONT_WAIT) ||
        (eventLoop->flags & AE_DONT_WAIT))
    {
        tv.tv_sec = tv.tv_usec = 0;
        tvp = &tv;
    } else if (flags & AE_TIME_EVENTS) {
        int64_t us = usUntilEarliestTimer(eventLoop);
        if (us >= 0) {
            tv.tv_sec = us / 1000000;
            tv.tv_usec = us % 1000000;
            tvp = &tv;
        }
    }

    /* 3. 等 fd 就绪，或者等到定时器超时 */
    int numevents = aeApiPoll(eventLoop, tvp);

    /* 4. 已经醒来，但还没有执行 fd 回调 */
    eventLoop->aftersleep(eventLoop);

    /* 5. 分发本轮就绪的 fd */
    for (int j = 0; j < numevents; j++) {
        int fd = eventLoop->fired[j].fd;
        int mask = eventLoop->fired[j].mask;
        aeFileEvent *fe = &eventLoop->events[fd];
        int fired = 0;
        int invert = fe->mask & AE_BARRIER;

        /* 默认先读后写 */
        if (!invert && fe->mask & mask & AE_READABLE) {
            fe->rfileProc(eventLoop, fd, fe->clientData, mask);
            fired++;
            fe = &eventLoop->events[fd];
        }

        /* 读写是同一个函数时，不重复调用，mask 已同时带上读写位 */
        if (fe->mask & mask & AE_WRITABLE &&
            (!fired || fe->wfileProc != fe->rfileProc))
        {
            fe->wfileProc(eventLoop, fd, fe->clientData, mask);
            fired++;
        }

        /* AE_BARRIER 要求先写后读 */
        if (invert) {
            fe = &eventLoop->events[fd];
            if (fe->mask & mask & AE_READABLE &&
                (!fired || fe->wfileProc != fe->rfileProc))
            {
                fe->rfileProc(eventLoop, fd, fe->clientData, mask);
            }
        }
    }

    /* 6. 文件事件处理完，再执行到期的定时器 */
    processTimeEvents(eventLoop);
}
```

`afterSleep` 不负责分发事件。`server.c:afterSleep` 主要重新获取 Module
GIL、更新缓存时间，并记录本轮事件循环的起始时间，然后控制流才回到
`aeProcessEvents` 分发 `fired[]`。

这段先记住两个结果：

```text
普通 fd：先执行可读回调，再执行可写回调
读写回调是同一函数：只调用一次，传入的 mask 同时包含两个标志
```

普通 TCP 的 `rfileProc/wfileProc` 都是 `connSocketEventHandler`，所以
`ae.c` 只进一次连接适配器，再由适配器调用真正的读写处理器。接着看
`aeApiPoll` 怎样把内核结果交回来：

```c
/* ae_epoll.c */
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;

    int retval = epoll_wait(
        state->epfd,
        state->events,
        eventLoop->setsize,
        timeout
    );

    for (int j = 0; j < retval; j++) {
        struct epoll_event *e = state->events + j;
        int mask = 0;

        if (e->events & EPOLLIN)  mask |= AE_READABLE;
        if (e->events & EPOLLOUT) mask |= AE_WRITABLE;

        eventLoop->fired[j].fd = e->data.fd;
        eventLoop->fired[j].mask = mask;
    }

    return retval;
}
```

所以一次分发的完整关系是：

```text
epoll_ctl 注册 fd=6
  -> 客户端发起连接
  -> epoll_wait 返回 { fd=6, EPOLLIN }
  -> aeApiPoll 写入 fired[0] = { fd=6, AE_READABLE }
  -> aeProcessEvents 查 events[6]
  -> 调用 events[6].rfileProc
```

Redis 没有遍历所有客户端。`epoll_wait` 直接给出已经就绪的 fd，Redis 只遍历 `fired[]`。

---

## 第一次唤醒：accept 出 fd=8

现在客户端连接 Redis，监听 fd=6 变成可读：

```text
fired[0] = { fd=6, mask=AE_READABLE }
events[6].rfileProc = connSocketAcceptHandler
```

于是进入 `socket.c:connSocketAcceptHandler`：

```c
static void connSocketAcceptHandler(aeEventLoop *el, int fd,
                                    void *privdata, int mask)
{
    int max = server.max_new_conns_per_cycle;

    while (max--) {
        /* fd=6 是监听 socket，cfd=8 是新客户端 socket */
        int cfd = anetTcpAccept(server.neterr, fd, ...);
        if (cfd == ANET_ERR) return;

        connection *conn =
            connCreateAcceptedSocket(el, cfd, NULL);

        acceptCommonHandler(conn, 0, cip);
    }
}
```

这里限制每轮最多 accept `max_new_conns_per_cycle` 个连接，避免连接风暴让主线程一直困在 accept 回调里。

`acceptCommonHandler` 给 fd=8 创建 `client`：

```c
/* networking.c，省略 maxclients 和错误处理 */
void acceptCommonHandler(connection *conn, int flags, char *ip) {
    client *c = createClient(conn);
    c->flags |= flags;
    connAccept(conn, clientAcceptHandler);
}
```

`createClient` 最关键的不是字段初始化，而是注册读处理器：

```c
client *createClient(connection *conn) {
    client *c = zmalloc(sizeof(client));

    connSetReadHandler(conn, readQueryFromClient);
    connSetPrivateData(conn, c);

    c->conn = conn;
    c->querybuf = NULL;
    c->bufpos = 0;
    /* ... */
    return c;
}
```

这里仍处在**回调注册阶段**，`readQueryFromClient` 还没有执行。
`networking.c` 先调用连接层的统一接口：

```c
/* connection.h */
static inline int connSetReadHandler(
    connection *conn,
    ConnectionCallbackFunc func
) {
    return conn->type->set_read_handler(conn, func);
}
```

`conn->type` 决定使用哪种传输实现。对于当前的普通 TCP 连接：

```text
conn->type                    -> CT_Socket
conn->type->set_read_handler  -> connSocketSetReadHandler
conn->type->ae_handler        -> connSocketEventHandler
```

所以这次调用实际进入 `connSocketSetReadHandler`：

```c
/* socket.c，省略重复注册和 func == NULL 的注销分支 */
static int connSocketSetReadHandler(
    connection *conn,
    ConnectionCallbackFunc func
) {
    /* 业务回调存在 connection 里 */
    conn->read_handler = func;

    /* ae 层注册的是连接适配器，不是 readQueryFromClient */
    return aeCreateFileEvent(
        conn->el,
        conn->fd,
        AE_READABLE,
        conn->type->ae_handler,
        conn
    );
}
```

这个函数做了两件不同层次的事：

1. 把业务读回调 `readQueryFromClient` 保存到 `conn->read_handler`。
2. 把普通 TCP 的事件适配器 `connSocketEventHandler` 注册到 AE。

因此 accept 完成后，保存的是下面这些关系：

```text
events[8] = {
    mask       = AE_READABLE,
    rfileProc  = connSocketEventHandler,
    clientData = conn
}

conn->read_handler = readQueryFromClient
conn->private_data = client c
c->conn = conn
```

这里有两种间接调用，不要把它们混成同一层：

```text
注册阶段：选择传输类型的实现

networking.c: connSetReadHandler(...)
  -> conn->type->set_read_handler(...)
  -> TCP: connSocketSetReadHandler(...)
     TLS: connTLSSetReadHandler(...)

触发阶段：把底层事件转交给业务回调

AE: events[fd].rfileProc(...)
  -> TCP: connSocketEventHandler(...)
     TLS: tlsEventHandler(...)
  -> conn->read_handler(conn)
     当前保存的值是 readQueryFromClient
```

第一种间接调用让 `networking.c` 只面对统一的 `connection` 接口，不需要
判断当前连接是普通 TCP 还是 TLS。第二种间接调用则给不同传输类型留出处理
底层事件的空间：普通 TCP 可以直接转交读写事件，TLS 还要处理握手以及
`SSL_WANT_READ`、`SSL_WANT_WRITE` 等状态。

到这里仅仅完成了 fd=8 的回调注册。等下一次 `epoll_wait` 报告 fd=8 可读时，
`connSocketEventHandler` 才会调用当前保存在 `conn->read_handler` 中的
`readQueryFromClient`。

---

## 第二次唤醒：fd=8 执行 SET

客户端把下面的 RESP 请求发到 fd=8：

```text
*3\r\n
$3\r\nSET\r\n
$4\r\nname\r\n
$5\r\nalice\r\n
```

内核发现 fd=8 有数据，下一次 `epoll_wait` 返回：

```text
fired[0] = { fd=8, mask=AE_READABLE }
```

`aeProcessEvents` 查到：

```text
events[8].rfileProc = connSocketEventHandler
```

先进入连接适配器：

```c
/* socket.c */
static void connSocketEventHandler(
    aeEventLoop *el,
    int fd,
    void *clientData,
    int mask
) {
    connection *conn = clientData;

    int call_read =
        (mask & AE_READABLE) && conn->read_handler;

    if (call_read) {
        callHandler(conn, conn->read_handler);
    }
}
```

因为：

```text
mask = AE_READABLE
conn->read_handler = readQueryFromClient
```

所以接下来才进入 `networking.c:readQueryFromClient`：

```c
void readQueryFromClient(connection *conn) {
    /* createClient 时保存的 client */
    client *c = connGetPrivateData(conn);
    size_t readlen = PROTO_IOBUF_LEN;

    /* 源码会在这里选择可复用或 client 私有的 querybuf */
    if (!c->querybuf)
        c->querybuf = sdsempty();

    /* 给 SDS 留出可写空间 */
    size_t qblen = sdslen(c->querybuf);
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);

    /* 从 fd=8 读到 querybuf 尾部 */
    int nread = connRead(
        c->conn,
        c->querybuf + qblen,
        readlen
    );

    if (nread == -1) {
        /* EAGAIN：连接仍正常，本轮只是没有读到数据 */
        if (connGetState(conn) == CONN_STATE_CONNECTED)
            goto done;

        freeClientAsync(c);
        goto done;
    } else if (nread == 0) {
        /* read 返回 0：对端关闭连接 */
        freeClientAsync(c);
        goto done;
    }

    sdsIncrLen(c->querybuf, nread);

    /* querybuf 里可能已有完整命令，立即解析并执行 */
    if (processInputBuffer(c) == C_ERR)
        c = NULL;

done:
    beforeNextClient(c);
}
```

上面删掉了大参数零拷贝、可复用 query buffer、I/O 线程和统计代码，读请求的主干就是：

```text
connRead(fd=8)
  -> 字节追加到 c->querybuf
  -> processInputBuffer(c)
```

`processInputBuffer` 不保证每次都执行命令：

```text
只读到半条 RESP  -> 保留 querybuf，等 fd=8 下次可读
读到一条完整命令 -> 解析并执行
读到多条 pipeline -> 循环解析并依次执行
```

对当前完整的 `SET`，只保留 multibulk 主分支后的等价骨架是：

```c
int processInputBuffer(client *c) {
    while ((c->querybuf &&
            c->qb_pos < sdslen(c->querybuf)) ||
           c->pending_cmds.ready_len > 0)
    {
        if (!c->reqtype) {
            c->reqtype =
                c->querybuf[c->qb_pos] == '*'
                ? PROTO_REQ_MULTIBULK
                : PROTO_REQ_INLINE;
        }

        /* RESP 字节 -> pendingCommand(argc, argv, cmd) */
        pendingCommand *pcmd = acquirePendingCommand();
        if (processMultibulkBuffer(c, pcmd) == C_ERR &&
            !pcmd->read_error)
        {
            /* RESP 还不完整，保留 querybuf 等下一次读事件 */
            freePendingCommand(c, pcmd);
            break;
        }

        addPendingCommand(&c->pending_cmds, pcmd);
        preprocessCommand(c, pcmd);

        /* 把当前 pendingCommand 挂到旧 client 字段上 */
        c->argc = pcmd->argc;
        c->argv = pcmd->argv;
        c->lookedcmd = pcmd->cmd;

        /* 主线程真正执行命令 */
        if (processCommandAndResetClient(c) == C_ERR)
            return C_ERR;
    }

    /* 丢掉已经消费的 querybuf 前缀 */
    sdsrange(c->querybuf, c->qb_pos, -1);
    c->qb_pos = 0;
    return C_OK;
}
```

命令执行的细节属于上一篇，这里只接到回复生成：

```text
processCommandAndResetClient
  -> processCommand
  -> call
  -> setCommand
  -> setKeyByLink
  -> addReply(c, shared.ok)
```

此时数据库已经变成：

```text
db["name"] = "alice"
```

但是 `+OK\r\n` 还没有写 socket。`addReply` 只做两件事：

```c
void addReply(client *c, robj *obj) {
    /* 第一次产生回复时，把 client 放进待写队列 */
    if (_prepareClientToWrite(c) != C_OK)
        return;

    /* 回复追加到 c->buf 或 c->reply */
    _addReplyToBufferOrList(
        c,
        obj->ptr,
        sdslen(obj->ptr)
    );
}

static inline int _prepareClientToWrite(client *c) {
    if (!clientHasPendingReplies(c))
        putClientInPendingWriteQueue(c);
    return C_OK;
}
```

执行完读回调后的状态是：

```text
c->buf = "+OK\r\n"
c->flags 包含 CLIENT_PENDING_WRITE
server.clients_pending_write 包含 c

fd=8 目前仍只注册了 AE_READABLE
```

---

## 下一轮 beforeSleep：把 +OK 写出去

`readQueryFromClient` 返回后，`aeProcessEvents` 还会处理本轮剩余的
`fired[]` 和到期定时器，然后返回 `aeMain`。

`while` 开始下一轮，再次进入 `aeProcessEvents`，第一步就是：

```c
eventLoop->beforesleep(eventLoop);
```

所以 `beforeSleep` 不是“每条命令执行完立刻调用”，而是“下一次准备
`epoll_wait` 之前调用”。

`server.c:beforeSleep` 很长，当前 `SET` 只需要先看这个顺序：

```c
void beforeSleep(aeEventLoop *eventLoop) {
    /* 过期、阻塞客户端等少量维护工作 */
    if (server.active_expire_enabled && iAmMaster())
        activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);

    /* appendfsync always 时，要先持久化再回 OK */
    if (server.aof_state == AOF_ON ||
        server.aof_state == AOF_WAIT_REWRITE)
    {
        flushAppendOnlyFile(0);
    }

    /* 尝试发送刚刚生成的回复 */
    handleClientsWithPendingWrites();

    /* 异步释放 client、裁剪复制 backlog 等 */
    freeClientsInAsyncFreeQueue();

    aeSetDontWait(server.el, dont_sleep);
}
```

`handleClientsWithPendingWrites` 有两个分支：

```c
int handleClientsWithPendingWrites(void) {
    while (clients_pending_write 还有 client) {
        client *c = 取出一个 client;

        c->flags &= ~CLIENT_PENDING_WRITE;
        从 pending_write 队列移除 c;

        /* 先直接写，不急着注册可写事件 */
        if (writeToClient(c, 0) == C_ERR)
            continue;

        /* socket 写不下剩余数据时，才监听 AE_WRITABLE */
        if (clientHasPendingReplies(c))
            installClientWriteHandler(c);
    }
}
```

### 分支一：短回复一次写完

`+OK\r\n` 很短，通常一次 `write` 就进入内核发送缓冲区：

```text
writeToClient(c, 0)
  -> connWrite(fd=8, "+OK\r\n", 5)
  -> c->buf 清空
  -> 返回
```

这种情况从头到尾都不需要给 fd=8 注册 `AE_WRITABLE`。这样少一次
`epoll_ctl`，也少一次事件循环唤醒。

### 分支二：socket 暂时写不下

如果回复很大，`writeToClient` 发生短写，或者已经达到
`NET_MAX_WRITES_PER_EVENT` 的单轮写入上限，缓冲区里仍有数据：

```c
void installClientWriteHandler(client *c) {
    connSetWriteHandlerWithBarrier(
        c->conn,
        sendReplyToClient,
        ae_barrier
    );
}
```

这里和注册读处理器一样，也会先经过 `ConnectionType` 的统一接口：

```c
/* connection.h */
static inline int connSetWriteHandlerWithBarrier(
    connection *conn,
    ConnectionCallbackFunc func,
    int barrier
) {
    return conn->type->set_write_handler(conn, func, barrier);
}
```

当前连接的 `conn->type` 是 `CT_Socket`，其中 `set_write_handler` 指向
`connSocketSetWriteHandler`，所以普通 TCP 最终进入：

```c
/* socket.c，省略重复设置检查，只保留 func != NULL 的注册路径 */
static int connSocketSetWriteHandler(
    connection *conn,
    ConnectionCallbackFunc func,
    int barrier
) {
    conn->write_handler = func;

    if (barrier)
        conn->flags |= CONN_FLAG_WRITE_BARRIER;
    else
        conn->flags &= ~CONN_FLAG_WRITE_BARRIER;

    return aeCreateFileEvent(
        conn->el,
        conn->fd,
        AE_WRITABLE,
        conn->type->ae_handler,
        conn
    );
}
```

这次调用中，`func` 是 `sendReplyToClient`，而
`conn->type->ae_handler` 取到 `connSocketEventHandler`。这里只是保存并注册
写回调，尚未执行 `sendReplyToClient`；它要等 fd=8 真正可写时才会被调用。

fd=8 已经注册过读事件，所以 `aeApiAddEvent` 使用
`EPOLL_CTL_MOD`，把关注事件合并成：

```text
内核：fd=8，EPOLLIN | EPOLLOUT

events[8]:
  mask      = AE_READABLE | AE_WRITABLE
  rfileProc = connSocketEventHandler
  wfileProc = connSocketEventHandler

conn:
  read_handler  = readQueryFromClient
  write_handler = sendReplyToClient
```

由于 `events[8].rfileProc == events[8].wfileProc`，如果 fd=8 同时读写
就绪，`aeProcessEvents` 只调用一次 `connSocketEventHandler`，并把同时
包含 `AE_READABLE | AE_WRITABLE` 的 `mask` 传进去。连接适配器内部再按
顺序调用 `readQueryFromClient` 和 `sendReplyToClient`。

等 fd=8 可写时：

```text
epoll_wait
  -> fired[] = { fd=8, AE_WRITABLE }
  -> connSocketEventHandler
  -> conn->write_handler
  -> sendReplyToClient
  -> writeToClient(c, 1)
```

全部写完后，`writeToClient(c, 1)` 会撤销写处理器：

```c
if (!clientHasPendingReplies(c)) {
    connSetWriteHandler(c->conn, NULL);
}
```

底层再用 `epoll_ctl(EPOLL_CTL_MOD)` 移除 `EPOLLOUT`，fd=8 回到只监听
`EPOLLIN`，等待下一条命令。

---

## 没有网络请求时，serverCron 怎么被唤醒

`serverCron` 在启动时只注册了一次：

```c
aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);
```

时间事件保存在链表中：

```c
typedef struct aeTimeEvent {
    long long id;
    monotime when;
    aeTimeProc *timeProc;
    void *clientData;
    struct aeTimeEvent *next;
} aeTimeEvent;
```

每轮进入 `epoll_wait` 前，Redis 找到最近定时器还剩多久：

```c
int64_t us = usUntilEarliestTimer(eventLoop);

tv.tv_sec = us / 1000000;
tv.tv_usec = us % 1000000;

aeApiPoll(eventLoop, &tv);
```

假设最近的 `serverCron` 还有 7ms 到期，又一直没有 fd 就绪：

```text
epoll_wait(..., timeout=7ms)
  -> 7ms 后超时，返回 0 个文件事件
  -> aeProcessEvents 不执行 file callback
  -> processTimeEvents
  -> serverCron
```

定时回调的返回值就是下一次执行间隔：

```c
static int processTimeEvents(aeEventLoop *eventLoop) {
    if (te->when <= now) {
        int retval = te->timeProc(
            eventLoop,
            te->id,
            te->clientData
        );

        if (retval != AE_NOMORE)
            te->when = now + retval * 1000;
    }
}

int serverCron(...) {
    clientsCron();
    databasesCron();
    replicationCron();
    /* RDB、AOF rewrite、Cluster 等周期检查 */

    return 1000 / server.hz;
}
```

默认 `hz=10` 时，`serverCron` 返回约 `100ms`。`processTimeEvents`
据此修改 `te->when`，不需要重新创建定时器。

所以时间事件也没有独立线程：

```text
最近定时器时间
  -> 变成 epoll_wait 的 timeout
  -> epoll_wait 超时
  -> 主线程执行 serverCron
```

---

## appendfsync always 为什么要 write barrier

主链路看完后再看这个分支就容易了。

假设 fd=8 在同一次 `epoll_wait` 中既可读又可写。普通情况可以先读请求、
执行命令，再立即调用写回调。但 `appendfsync always` 要保证：

```text
执行写命令
  -> 下一轮 beforeSleep 调 flushAppendOnlyFile
  -> fsync 完成
  -> 才能把 OK 发给客户端
```

因此安装写处理器时会打开 barrier：

```c
if (server.aof_state == AOF_ON &&
    server.aof_fsync == AOF_FSYNC_ALWAYS)
{
    ae_barrier = 1;
}

connSetWriteHandlerWithBarrier(
    c->conn,
    sendReplyToClient,
    ae_barrier
);
```

当前源码的普通 TCP 连接把它记在：

```c
conn->flags |= CONN_FLAG_WRITE_BARRIER;
```

`connSocketEventHandler` 收到读写同时就绪时，按 barrier 决定逻辑回调顺序：

```c
int invert = conn->flags & CONN_FLAG_WRITE_BARRIER;

if (!invert && call_read)
    callHandler(conn, conn->read_handler);

if (call_write)
    callHandler(conn, conn->write_handler);

if (invert && call_read)
    callHandler(conn, conn->read_handler);
```

barrier 打开后先处理旧回复的写事件，再读取并执行新命令。新命令产生的回复
不会在同一轮读回调之后发出，循环必须经过下一次 `beforeSleep`，于是 AOF
有机会先 `fsync`。

---

## 用断点按这条链路读

不要一开始就在 `serverCron`、I/O 线程和 TLS 之间来回跳。先用普通 TCP
单客户端跑通下面这些断点：

```gdb
b aeMain
b aeProcessEvents
b connSocketAcceptHandler
b acceptCommonHandler
b createClient
b connSocketSetReadHandler
b connSocketEventHandler
b readQueryFromClient
b handleClientsWithPendingWrites
b writeToClient
```

第一次在 `connSocketEventHandler` 停下时看：

```gdb
p fd
p mask
p conn->read_handler
p conn->write_handler
p conn->private_data
```

预期是：

```text
fd = 客户端 fd
mask 包含 AE_READABLE
read_handler = readQueryFromClient
write_handler = NULL
private_data = client *
```

执行 `SET name alice` 后，在 `handleClientsWithPendingWrites` 看：

```gdb
p c->bufpos
p c->flags
p c->conn->fd
```

如果 `+OK\r\n` 一次写完，`writeToClient` 返回后 `bufpos` 会变成 0；
如果没有写完，就继续跟进 `installClientWriteHandler`，观察 fd 如何增加
`AE_WRITABLE`。

---

## 完整调用链

把整篇压成一条连续调用链：

```text
main
  -> initServer
      -> aeCreateEventLoop
      -> aeCreateTimeEvent(serverCron)
      -> 设置 beforeSleep / afterSleep
  -> initListeners
      -> listen fd=6
      -> aeCreateFileEvent(fd=6, connSocketAcceptHandler)
  -> aeMain
      -> aeProcessEvents
          -> beforeSleep
          -> epoll_wait

第一次醒来：fd=6 readable
  -> connSocketAcceptHandler
      -> accept 得到 fd=8
      -> acceptCommonHandler
          -> createClient
              -> connSetReadHandler(readQueryFromClient)
              -> aeCreateFileEvent(
                     fd=8,
                     connSocketEventHandler
                 )

第二次醒来：fd=8 readable
  -> connSocketEventHandler
      -> conn->read_handler
      -> readQueryFromClient
          -> connRead
          -> processInputBuffer
          -> processCommand
          -> call
          -> setCommand
          -> addReply("+OK")
          -> client 进入 clients_pending_write

下一轮
  -> beforeSleep
      -> flushAppendOnlyFile
      -> handleClientsWithPendingWrites
          -> writeToClient
          -> 写完：继续只监听 readable
          -> 没写完：注册 writable，稍后继续写
```

理解这条链以后，再分别展开 TLS、I/O 线程、`AE_BARRIER`、Cluster bus
都只是替换某一层回调，不会再丢失“当前 fd 从哪里来、下一步为什么调用这个
函数”的上下文。
