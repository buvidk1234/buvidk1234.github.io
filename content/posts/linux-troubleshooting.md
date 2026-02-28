+++
date = '2026-02-28T16:43:35+08:00'
draft = false
title = 'Linux Troubleshooting'
+++

## L1 日常排错篇

> **适用场景**：服务连不上、启动报错、明显卡顿。

### 1. 查死活与资源占用

#### ps -ef | grep \<name\>

- **用途**：最基础的进程探测。看进程是否存在、看启动参数是否正确、看运行态的 PID。

#### top / htop

- **用途**：看系统的整体大盘。
- **极客技巧**：在 top 界面按 `1` 展开所有 CPU 核，按 `P` 找 CPU 消耗大户，按 `M` 找内存泄露元凶。htop 是其彩色增强版，支持鼠标和树状图（需安装）。

#### free -h

- **用途**：看内存还剩多少。
- **避坑**：不要看 `free` 列，直接看 `available`（系统真实可分配内存）。

---

### 2. 查网络与连通性

#### ss -lntp

- **用途**：查本地监听端口。比老旧的 `netstat` 快百倍。排查"服务是否真起来了"、"谁占了 8080 端口"。
- `-l` 仅显示监听状态，`-n` 不做域名反解（提速），`-t` 只看 TCP，`-p` 显示进程名和 PID。

#### ss -ant

- **用途**：查看**所有** TCP 连接的实时状态分布（ESTABLISHED / TIME_WAIT / CLOSE_WAIT 等）。
- **诊断技巧**：
  - `TIME_WAIT` 堆积过多 → 短连接并发太高，考虑开启 `tcp_tw_reuse` 或改用长连接。
  - `CLOSE_WAIT` 堆积 → 应用代码没有正确关闭连接（典型 Bug），需排查代码。
  - 快速统计各状态数量：

```bash
ss -ant | awk '{print $1}' | sort | uniq -c | sort -rn
```

#### ss -s

- **用途**：一行命令输出 TCP/UDP 连接数汇总统计，快速掌握全局连接规模。

```bash
$ ss -s
Total: 1024
TCP:   862 (estab 521, closed 12, orphaned 0, timewait 310)
```

#### lsof -i :\<port\>

- **用途**：报 `Address already in use` 时，一秒揪出占用该端口的凶手 PID。

#### nc -zv \<ip\> \<port\>

- **用途**：纯 TCP 层的端口探活。不发数据，只测三次握手通不通（排查防火墙/安全组拦截的最快手段）。

#### curl -Iv \<url\>

- **用途**：HTTP 层探活。`-I` 只拉取 Header 提速，`-v` 打印详细的 DNS 解析与握手过程。

#### curl -w（请求耗时拆分）

- **用途**：精确测量一次 HTTP 请求各阶段的耗时，**一行命令定界"慢在哪"**。

```bash
$ curl -o /dev/null -s -w "dns: %{time_namelookup}s\ntcp: %{time_connect}s\ntls: %{time_appconnect}s\nttfb: %{time_starttransfer}s\ntotal: %{time_total}s\n" https://example.com

dns:   0.028s      # DNS 解析耗时
tcp:   0.045s      # TCP 三次握手完成
tls:   0.097s      # TLS 握手完成
ttfb:  0.312s      # 首字节到达（服务端处理耗时）
total: 0.340s      # 整个请求完成
```

- **定界思路**：
  - `dns` 高 → DNS 解析慢，用 `dig` 排查是否解析链路有问题
  - `tcp - dns` 高 → 网络延迟大或连接建立慢，用 `ping`/`nc` 验证
  - `ttfb - tls` 高 → **服务端处理慢**（最常见），查服务端日志、CPU、慢 SQL
  - `total - ttfb` 高 → 响应体传输慢，查带宽是否打满（`sar -n DEV`）

#### ping -c 4 \<ip\>

- **用途**：最基础的 ICMP 连通性检测与延迟测量。
- **注意**：很多云服务器默认禁 ping（安全组/iptables 屏蔽 ICMP），ping 不通 ≠ 网络不通，需配合 `nc`/`curl` 交叉验证。

#### dig / nslookup \<domain\>

- **用途**：DNS 解析排查。服务连不上时，第一步确认域名是否解析到了正确的 IP。
- **极客技巧**：`dig +trace <domain>` 从根域开始逐级追踪解析链路，排查 DNS 劫持或缓存污染。

#### ip addr / ip route

- **用途**：查看网卡 IP 地址、子网掩码、路由表。替代已废弃的 `ifconfig` 和 `route`。
- **排查场景**：多网卡环境确认流量走了哪张网卡、默认网关是否正确。

#### tcpdump -i eth0 port 80 -w dump.pcap

- **用途**：终极网络定界工具。当业务层查不出为何丢包或超时，直接到底层网卡抓包，导出后用 Wireshark 分析时序。

---

### 3. 查日志与磁盘

#### tail -f /var/log/xxx.log

- **用途**：实时追踪日志输出，排查启动报错或运行时异常。
- **进阶**：`tail -f log | grep --line-buffered 'ERROR'` 实时过滤错误行；多日志同时追踪用 `tail -f a.log b.log`。

#### journalctl -u \<service\> -f

- **用途**：查看 systemd 管理的服务日志。比翻日志文件更方便，支持时间过滤。
- **常用组合**：
  - `-f`：实时追踪
  - `--since "10 min ago"`：只看最近 10 分钟
  - `-p err`：只看错误级别以上

#### df -h / du -sh \<path\>

- **用途**：`df -h` 查看各挂载点磁盘使用率，`du -sh *` 找出哪个目录最占空间。
- **避坑**：`df` 显示磁盘满但 `du` 加起来不够？可能是已删除文件仍被进程持有（`lsof +L1` 排查）。

#### iostat

- **用途**：最基础的磁盘读写查看，评估是否在疯狂写日志或落盘慢。

---

## L2 高阶诊断篇

> **适用场景**：系统负载飙升、偶发性延迟、无明显报错但疯狂卡顿。

当线上出现诡异性能问题时，按以下顺序执行，可在 **60 秒内**快速框定瓶颈在 CPU、内存、磁盘还是网络。

### 01. uptime — 全局负载定界

```bash
$ uptime
 16:40:00 up 10 days,  load average: 12.05, 8.50, 4.20
```

- **看点**：`load average` 代表想要运行的任务（进程）数量。在 Linux 上，这不仅包括想要使用 CPU 的进程，还包括阻塞在不可中断 I/O（通常是磁盘 IO）中的进程。它给出的是资源负载（需求）的高层概览，但仅凭它无法准确判断瓶颈，需配合后续工具。
- **阈值**：三个数分别是 1/5/15 分钟的指数衰减移动平均值。如果 1 分钟值远高于 15 分钟值，说明负载在近期飙升；反之则可能已经错过了问题现场。

### 02. dmesg | tail — 内核级报错捕获

```bash
$ dmesg | tail -n 10
[1880957.563150] perl invoked oom-killer: gfp_mask=0x280da, order=0, oom_score_adj=0
[...]
[1880957.563400] Out of memory: Kill process 18694 (perl) score 246 or sacrifice child
[1880957.563408] Killed process 18694 (perl) total-vm:1972392kB, anon-rss:1953348kB, file-rss:0kB
[2320864.954447] TCP: Possible SYN flooding on port 7001. Dropping request.  Check SNMP counters.
```

- **看点**：查看内核环形缓冲区。
- **关键特征**：`Out of memory: Kill process`（OOM）、`TCP: time wait bucket table overflow`（网络并发爆满）、`Hardware Error`。不要在应用日志里死磕，先看内核有没有把它干掉。

### 03. vmstat 1 — 虚拟内存与全局大盘

每秒打印一次系统宏观状态。

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 4  0      0   512M   102M  1.2G     0    0    12    34  102  204 80 10 10  0  0
```

- **r (Run Queue)**：正在 CPU 上运行以及等待 CPU 时间片的进程数。比 `load average` 更能反映 CPU 饱和度，因为它不包含 I/O 等待。若 `r` > CPU 核数，证明 **CPU 饱和**。
- **b (Blocked)**：处于不可中断睡眠（通常在等磁盘 IO）的进程数。若 `b` 持续 > 0，说明存在 **IO 瓶颈**。
- **si, so (Swap)**：换入和换出。如果这两个值非零，说明物理内存已经耗尽。
- **us, sy, id, wa, st**：CPU 时间拆分——用户态、内核态、空闲、I/O 等待、被虚拟化偷走的时间。`us` + `sy` 可确认 CPU 是否繁忙；`wa` 持续偏高则指向磁盘瓶颈（CPU 空闲是因为任务在等待挂起的磁盘 I/O）；`sy` 超过 20% 值得深挖，可能是内核在低效地处理 I/O。

### 04. mpstat -P ALL 1 — CPU 核心失衡检查

```bash
$ mpstat -P ALL 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015  _x86_64_ (32 CPU)

07:38:49 PM  CPU   %usr  %nice   %sys %iowait   %irq  %soft  %steal  %guest  %gnice  %idle
07:38:50 PM  all  98.47   0.00   0.75    0.00   0.00   0.00    0.00    0.00    0.00   0.78
07:38:50 PM    0  96.04   0.00   2.97    0.00   0.00   0.00    0.00    0.00    0.00   0.99
07:38:50 PM    1  97.00   0.00   1.00    0.00   0.00   0.00    0.00    0.00    0.00   2.00
07:38:50 PM    2  98.00   0.00   1.00    0.00   0.00   0.00    0.00    0.00    0.00   1.00
07:38:50 PM    3  96.97   0.00   0.00    0.00   0.00   0.00    0.00    0.00    0.00   3.03
[...]
```

- **看点**：拆解每个 CPU 核心的使用率。
- **诊断**：如果发现某一个核 100%，其他核 0%，通常说明应用是单线程的，或者是网卡中断没有均衡分布（查软中断 `%soft`）。

### 05. pidstat 1 — 微观进程 CPU 溯源

```bash
$ pidstat 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015    _x86_64_    (32 CPU)

07:41:02 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
07:41:03 PM     0         9    0.00    0.94    0.00    0.94     1  rcuos/0
07:41:03 PM     0      4214    5.66    5.66    0.00   11.32    15  mesos-slave
07:41:03 PM     0      6521 1596.23    1.89    0.00 1598.11    27  java
07:41:03 PM     0      6564 1571.70    7.55    0.00 1579.25    28  java
07:41:03 PM 60004     60154    0.94    4.72    0.00    5.66     9  pidstat
```

- **看点**：当你通过 `vmstat` 发现系统 CPU 极高时，用此命令直接抓出现场正在狂吃 CPU 的具体 PID。比 `top` 滚动查看更清晰，方便事后追溯。
- **示例解读**：`%CPU` 列是所有 CPU 核心的总和——上面 java 进程显示 1598%，意味着它吃掉了约 16 个核。

### 06. iostat -xz 1 — 块设备 IO 饱和度解剖

```bash
$ iostat -xz 1
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00 1200.00 3400.00 12000.00 34000.00    20.00    15.00   12.50    5.00   15.14   0.21  98.5%
```

- **r/s, w/s, rkB/s, wkB/s**：每秒读写次数和读写吞吐量，用于评估工作负载特征——性能问题可能仅仅是因为施加了过大的负载。
- **await**：I/O 平均响应时间（毫秒），包含排队时间和服务时间。**这是应用层真正感知到的延迟。** 超出预期的平均时间可能是设备饱和或设备故障的信号。
- **avgqu-sz**：平均请求队列长度。大于 1 可能是饱和的证据（不过设备通常能并行处理请求，尤其是前端代理多块后端磁盘的虚拟设备）。
- **%util**：设备利用率（繁忙百分比）。**超过 60%** 通常就会导致性能下降（应能从 `await` 中观察到），接近 100% 通常意味着饱和。但如果存储设备是前端代理多块后端磁盘的逻辑设备，100% 可能只是说明一直有 I/O 在处理，后端磁盘实际远未饱和。

### 07. free -m — 内存快速确认

```bash
$ free -m
             total       used       free     shared    buffers     cached
Mem:        245998      24545     221453         83         59        541
-/+ buffers/cache:      23944     222053
Swap:            0          0          0
```

- **看点**：以 MB 为单位再次确认物理内存分配状态。重点关注 `buffers` 和 `cached` 是否接近零——若接近零则磁盘 IO 会飙升（可用 `iostat` 确认）。此时结合前面 `vmstat` 的 `si`/`so`（Swap 换入换出）指标，如果 `si`/`so` 频繁飙高，说明物理内存已耗尽，系统正在用低速磁盘当内存用，性能会呈现断崖式下跌。

### 08. sar -n DEV 1 — 网卡吞吐量定界

```bash
$ sar -n DEV 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015     _x86_64_    (32 CPU)

12:16:48 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
12:16:49 AM      eth0  18763.00   5032.00  20686.42    478.30      0.00      0.00      0.00      0.00
12:16:49 AM        lo     14.00     14.00      1.36      1.36      0.00      0.00      0.00      0.00
12:16:49 AM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

- **看点**：观察 `rxkB/s`（接收）和 `txkB/s`（发送）。
- **诊断**：对比你的服务器网卡硬件上限（如千兆网卡理论极值 125MB/s）。如果数值逼近物理极限，说明业务卡顿是因为网卡被打满了（下载大文件、遭受泛洪攻击等）。

### 09. sar -n TCP,ETCP 1 — TCP 协议栈健康度

```bash
$ sar -n TCP,ETCP 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015    _x86_64_    (32 CPU)

12:17:19 AM  active/s passive/s    iseg/s    oseg/s
12:17:20 AM      1.00      0.00  10233.00  18846.00

12:17:19 AM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
12:17:20 AM      0.00      0.00      0.00      0.00      0.00
```

- **看点 1 (active/s & passive/s)**：每秒本地主动发起（`connect()`）的连接和被动接受（`accept()`）的连接数。可粗略衡量服务器负载：`passive/s` 是入站新连接数，`active/s` 是出站下游连接数。
- **看点 2 (retrans/s)**：**每秒 TCP 重传数，这是网络排障的黄金指标！** 重传意味着网络或服务端出了问题——可能是不可靠的网络链路（如公网）导致丢包，也可能是服务端过载主动丢弃数据包。

### 10. top — 最后的宏观复核

```bash
$ top
top - 00:15:40 up 21:56,  1 user,  load average: 31.09, 29.87, 29.92
Tasks: 871 total,   1 running, 868 sleeping,   0 stopped,   2 zombie
%Cpu(s): 96.8 us,  0.4 sy,  0.0 ni,  2.7 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:  25190241+total, 24921688 used, 22698073+free,    60448 buffers
KiB Swap:        0 total,        0 used,        0 free.   554208 cached Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 20248 root      20   0  0.227t 0.012t  18748 S  3090  5.2  29812:58 java
  4213 root      20   0 2722544  64640  44232 S  23.5  0.0 233:35.37 mesos-slave
 66128 titancl+  20   0   24344   2332   1172 R   1.0  0.0   0:00.07 top
  5235 root      20   0 38.227g 547004  49996 S   0.7  0.2   2:02.74 java
  4299 root      20   0 20.015g 2.682g  16836 S   0.3  1.1  33:14.42 java
```

- 经历了前面的抽丝剥茧，最后敲下 `top`。`top` 涵盖了前面大多数命令的关键指标，此时用它做最终的宏观复核——如果数值与前几步差异很大，说明负载在持续变化，需要留意动态趋势。

---

## 总结：排障导图速查表

| 故障现象表现 | L1 定界指令 | L2 深度剖析指令 |
|:---|:---|:---|
| **CPU 报警 / 负载高** | `top`, `ps` | `uptime` ➔ `vmstat` ➔ `mpstat` ➔ `pidstat` |
| **内存飙升 / OOM** | `free -h` | `dmesg \| tail` ➔ `vmstat`（查 Swap） |
| **磁盘打满 / 响应极慢** | `iostat` | `vmstat`（查 wa, b）➔ `iostat -xz 1`（查 await） |
| **连接超时 / 报 502/504** | `ss`, `nc`, `curl`, `dig` | `sar -n DEV` ➔ `sar -n ETCP` ➔ `tcpdump` |
| **连接堆积 / TIME_WAIT 爆满** | `ss -ant`, `ss -s` | `sar -n TCP,ETCP`（查 retrans） |
| **磁盘空间不足** | `df -h`, `du -sh` | `iostat -xz 1`（查 await） |
| **DNS 解析异常** | `dig`, `nslookup` | `dig +trace`（逐级追踪解析链路） |

---

## 参考资料

- [Linux Performance Analysis in 60,000 Milliseconds — Netflix Technology Blog](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)

