+++
date = '2025-12-23T21:47:09+08:00'
draft = false
title = 'Rockscache Consistency'
+++

# 深入解析 RocksCache：如何优雅地解决缓存与数据库一致性问题

> 本文深入剖析 RocksCache 的设计思想与核心实现，带你理解这个首创的缓存一致性解决方案。

## 前言

在分布式系统中，缓存是提升性能的利器，但也是一致性问题的重灾区。你是否曾经遇到过这样的困扰：

- 明明更新了数据库，为什么缓存里还是旧数据？
- 用了「先更新DB，再删缓存」策略，为什么还是会出现不一致？
- 如何在保证一致性的同时，还能保持高性能？

今天介绍的 **RocksCache**，是一个来自 DTM Labs 的开源项目，它通过一套精巧的设计，**在不引入版本号的前提下**，优雅地解决了缓存与数据库的一致性难题。

---

## 一、经典的缓存一致性问题

### 1.1 常见的缓存策略

最常用的缓存管理策略是 **Cache-Aside**（旁路缓存）：

```
读取：先查缓存 → 缓存命中则返回 → 未命中则查DB → 写入缓存 → 返回
更新：更新DB → 删除缓存
```

这个策略看似简单，却隐藏着一个致命的并发问题。

### 1.2 并发场景下的数据不一致

考虑以下时序：

```
时间  →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→

线程A（读请求）:
        查DB(v1) ─────────────────────────────→ 写缓存(v1)
                        (网络延迟)

线程B（写请求）:
                   更新DB(v2) → 删除缓存
```

**问题**：线程A 查询到 v1 后，发生了网络延迟。此时线程B 完成了更新并删除缓存。但线程A 的写缓存操作在删除之后执行，导致缓存中存储了旧值 v1。

这就是著名的 **"删除后写入"** 问题，常规的「先更新DB再删缓存」策略无法解决。

### 1.3 传统解决方案的局限

| 方案 | 描述 | 缺点 |
|------|------|------|
| 延迟双删 | 删除缓存后，延迟一段时间再删一次 | 延迟时间难以确定，仍有不一致窗口 |
| 版本号 | 每条数据带版本号，写入时比较版本 | 侵入业务，改造成本高 |
| 分布式锁 | 读写都加锁 | 性能差，热点数据成为瓶颈 |
| 订阅 binlog | 通过 Canal 等订阅 DB 变更 | 架构复杂，延迟较高 |

有没有一种方案，既能保证一致性，又不侵入业务，还能保证高性能？

---

## 二、RocksCache 的设计思想

RocksCache 提出了一个创新方案：**标记删除 + 锁持有者验证**。

### 2.1 核心理念

> **不直接删除缓存，而是标记为已删除；写入时验证锁持有者，拒绝过期的写入。**

### 2.2 数据结构设计

RocksCache 将每个缓存 key 存储为 Redis Hash：

```redis
key → {
    value:     "实际缓存值"
    lockUntil: 1703347200        # 锁定到期时间戳
    lockOwner: "uuid-xxxxx"      # 锁持有者ID
}
```

这三个字段协同工作，实现了隐式的版本控制。

### 2.3 两个核心方法

RocksCache 对外只暴露两个核心方法：

```go
// 读取缓存（缓存不存在时自动调用 fn 获取数据）
value, err := rc.Fetch(key, expire, func() (string, error) {
    return queryFromDB()
})

// 标记删除（更新DB后调用）
err := rc.TagAsDeleted(key)
```

极简的 API 背后，是精妙的一致性保证机制。

---

## 三、源码深度剖析

### 3.1 TagAsDeleted：标记删除

当数据更新后，调用 `TagAsDeleted` 标记缓存失效：

```lua
-- deleteScript
redis.call('HSET', KEYS[1], 'lockUntil', 0)    -- 标记锁定时间为0（已失效）
redis.call('HDEL', KEYS[1], 'lockOwner')       -- 清除锁持有者
redis.call('EXPIRE', KEYS[1], ARGV[1])         -- 设置延迟过期（默认10秒）
```

**关键点**：
- 将 `lockUntil` 设为 0，表示缓存已失效，需要刷新
- 数据**不立即删除**，保留一段时间（Delay），这段时间内仍可返回旧值
- 清除 `lockOwner`，使得之前的读请求无法写入新值

### 3.2 Fetch：读取缓存

Fetch 是读取的核心方法：

```go
func (c *Client) Fetch2(ctx context.Context, key string, expire time.Duration, 
    fn func() (string, error)) (string, error) {
    
    // 使用 singleflight 防止进程内并发
    v, err, _ := c.group.Do(key, func() (interface{}, error) {
        if c.Options.DisableCacheRead {
            return fn()  // 降级模式：直接查DB
        } else if c.Options.StrongConsistency {
            return c.strongFetch(ctx, key, ex, fn)  // 强一致模式
        }
        return c.weakFetch(ctx, key, ex, fn)  // 最终一致模式（默认）
    })
    return v.(string), err
}
```

#### 3.2.1 luaGet：获取值并尝试加锁

```lua
-- getScript
local v = redis.call('HGET', KEYS[1], 'value')
local lu = redis.call('HGET', KEYS[1], 'lockUntil')

-- 需要加锁的条件：锁已过期 或 (无锁且无值)
if lu ~= false and tonumber(lu) < tonumber(ARGV[1]) or lu == false and v == false then
    redis.call('HSET', KEYS[1], 'lockUntil', ARGV[2])   -- 设置锁到期时间
    redis.call('HSET', KEYS[1], 'lockOwner', ARGV[3])   -- 设置锁持有者
    return { v, 'LOCKED' }  -- 返回值 + 锁定标记
end
return {v, lu}  -- 返回值 + 锁定时间
```

返回值含义：
- `{nil, "LOCKED"}`：无缓存，获得锁，需要查询DB
- `{value, "LOCKED"}`：有旧值，获得锁，可返回旧值并后台刷新
- `{value, lockUntil}`：有缓存且未过期，直接返回
- `{nil, lockUntil}`：无缓存但被其他线程锁定，需要等待

#### 3.2.2 weakFetch：最终一致模式

```go
func (c *Client) weakFetch(ctx context.Context, key string, expire time.Duration, 
    fn func() (string, error)) (string, error) {
    
    owner := shortuuid.New()  // 生成唯一ID
    r, err := c.luaGet(ctx, key, owner)
    
    // 等待其他线程释放锁（无值且被锁定的情况）
    for err == nil && r[0] == nil && r[1].(string) != locked {
        time.Sleep(c.Options.LockSleep)
        r, err = c.luaGet(ctx, key, owner)
    }
    
    if r[1] != locked {
        return r[0].(string), nil  // 有缓存且未过期，直接返回
    }
    
    // 获得锁
    if r[0] == nil {
        return c.fetchNew(ctx, key, expire, owner, fn)  // 无旧值，同步查询
    }
    
    // ⭐ 关键优化：有旧值时，立即返回旧值，后台异步刷新
    go withRecover(func() {
        _, _ = c.fetchNew(ctx, key, expire, owner, fn)
    })
    return r[0].(string), nil
}
```

**设计亮点**：当缓存失效但有旧值时，**立即返回旧值**，同时**异步刷新缓存**。这保证了：
- 用户请求不会阻塞
- 热点数据删除时不会造成响应延迟
- 最终数据会被刷新为新值

#### 3.2.3 luaSet：写入缓存（带验证）

```lua
-- setScript
local o = redis.call('HGET', KEYS[1], 'lockOwner')
if o ~= ARGV[2] then
    return  -- ⭐ 关键：不是锁持有者，拒绝写入！
end
redis.call('HSET', KEYS[1], 'value', ARGV[1])
redis.call('HDEL', KEYS[1], 'lockUntil')
redis.call('HDEL', KEYS[1], 'lockOwner')
redis.call('EXPIRE', KEYS[1], ARGV[3])
```

**这就是解决"删除后写入"问题的关键**：

1. 线程A 读取时获得锁，记录自己的 `owner`
2. 线程B 更新DB后调用 `TagAsDeleted`，清除了 `lockOwner`
3. 线程A 尝试写入缓存时，发现 `lockOwner` 不匹配，**写入被拒绝**
4. 后续的读请求会重新查询DB，获取最新值

---

## 四、防御机制详解

### 4.1 防缓存击穿

```go
// 进程内使用 singleflight
c.group.Do(key, func() (interface{}, error) {
    // 同一进程内相同key只执行一次
})
```

```lua
-- Redis层使用分布式锁
if lu ~= false and tonumber(lu) < tonumber(ARGV[1]) then
    redis.call('HSET', KEYS[1], 'lockUntil', ARGV[2])
    return { v, 'LOCKED' }  -- 只有一个请求能获得锁
end
```

双重防护确保：无论多少请求，最终只有一个请求会查询DB。

### 4.2 防缓存穿透

```go
func (c *Client) fetchNew(...) (string, error) {
    result, err := fn()
    if result == "" {
        if c.Options.EmptyExpire == 0 {
            return c.rdb.Del(ctx, key).Err()  // 不缓存空值
        }
        expire = c.Options.EmptyExpire  // 缓存空值，使用较短过期时间
    }
    // ...
}
```

空结果也会被缓存，默认过期时间60秒，防止恶意请求反复查询不存在的数据。

### 4.3 防缓存雪崩

```go
// 过期时间随机化
ex := expire - c.Options.Delay - time.Duration(
    rand.Float64() * c.Options.RandomExpireAdjustment * float64(expire)
)
```

默认 `RandomExpireAdjustment = 0.1`，即过期时间会在 90%~100% 之间随机波动，避免大量缓存同时过期。

---

## 五、一致性保证的数学证明

我们来严格证明 RocksCache 如何解决"删除后写入"问题。

**场景重现**：
- T1: 线程A 执行 `luaGet`，获得锁，`lockOwner = A`
- T2: 线程A 查询DB，获得 v1
- T3: 线程B 更新DB为 v2
- T4: 线程B 执行 `TagAsDeleted`，`lockOwner` 被清除
- T5: 线程A 执行 `luaSet`，尝试写入 v1

**关键步骤分析**：

在 T5 时刻，`luaSet` 脚本执行：
```lua
local o = redis.call('HGET', KEYS[1], 'lockOwner')  -- o = nil (被T4清除)
if o ~= ARGV[2] then  -- nil != 'A'
    return  -- 写入被拒绝！
end
```

**结论**：v1 不会被写入缓存，后续请求会重新查询DB获取 v2。✅

---

## 六、最佳实践

### 6.1 基础使用

```go
import "github.com/dtm-labs/rockscache"

// 创建客户端
rc := rockscache.NewClient(redisClient, rockscache.NewDefaultOptions())

// 读取数据
func GetUser(userID int64) (*User, error) {
    key := fmt.Sprintf("user:%d", userID)
    data, err := rc.Fetch(key, 10*time.Minute, func() (string, error) {
        user, err := db.QueryUser(userID)
        if err != nil {
            return "", err
        }
        return json.Marshal(user)
    })
    if err != nil {
        return nil, err
    }
    var user User
    json.Unmarshal([]byte(data), &user)
    return &user, nil
}

// 更新数据
func UpdateUser(user *User) error {
    // 1. 更新数据库
    if err := db.UpdateUser(user); err != nil {
        return err
    }
    // 2. 标记缓存删除
    key := fmt.Sprintf("user:%d", user.ID)
    return rc.TagAsDeleted(key)
}
```

### 6.2 配合分布式事务

对于需要保证「更新DB」和「删除缓存」原子性的场景，推荐配合 DTM 使用：

```go
// 使用 DTM 二阶段消息保证原子性
msg := dtmcli.NewMsg(DtmServer, gid).
    Add(BusiUrl+"/UpdateDB", req).
    Add(BusiUrl+"/DeleteCache", req)
msg.Submit()
```

### 6.3 参数调优建议

```go
opts := rockscache.Options{
    Delay:                  10 * time.Second,   // 根据业务容忍的不一致时间调整
    EmptyExpire:            60 * time.Second,   // 空结果缓存时间
    LockExpire:             3 * time.Second,    // 应 >= 最大DB查询时间
    LockSleep:              100 * time.Millisecond,
    RandomExpireAdjustment: 0.1,                // 10%的随机波动
    StrongConsistency:      false,              // 除非必要，否则用最终一致
}
```

---

## 七、与其他方案对比

| 方案 | 一致性 | 性能 | 复杂度 | 侵入性 |
|------|--------|------|--------|--------|
| 延迟双删 | 弱 | 高 | 低 | 低 |
| 版本号 | 强 | 中 | 高 | 高 |
| Binlog订阅 | 最终 | 中 | 高 | 低 |
| **RocksCache** | **最终/强** | **高** | **低** | **低** |

RocksCache 在各个维度都取得了很好的平衡。

---

## 八、总结

RocksCache 通过以下设计解决了缓存一致性难题：

1. **标记删除而非直接删除**：保留旧值用于快速响应，同时触发刷新
2. **lockOwner 验证机制**：拒绝过期的写入请求，避免"删除后写入"问题
3. **读时加锁+写时验证**：无需应用层版本号，隐式实现版本控制
4. **返回旧值+异步刷新**：在保证最终一致的同时，提供极致的响应速度
5. **三大防护内置**：防击穿、防穿透、防雪崩开箱即用

如果你正在为缓存一致性问题头疼，RocksCache 绝对值得一试。它的设计思想也可以迁移到其他语言实现，核心在于理解「标记删除 + 锁持有者验证」这套机制。

---

## 参考资料

- [RocksCache GitHub](https://github.com/dtm-labs/rockscache)
- [DTM 缓存一致性文档](https://dtm.pub/app/cache.html)
- [携程最终一致和强一致性缓存实践](https://www.infoq.cn/article/hh4iouiijhwb4x46vxeo)
