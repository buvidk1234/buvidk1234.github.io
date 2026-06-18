+++
date = '2026-02-28T16:43:35+08:00'
draft = false
title = 'Linux Troubleshooting'
+++

## L1 日常排错篇

> **适用场景**：服务连不上、启动报错、明显卡顿。L1 不追求命令全，只保留日常最常用、能马上定方向的几组操作；指标解释和深入定位放到 L2。

### 1. 服务是不是还活着

**使用场景**：服务启动后访问不了、发布后怀疑没起来、进程异常退出、systemd 服务状态不对。

```bash
systemctl status <service> --no-pager
journalctl -u <service> -n 100 --no-pager
journalctl -u <service> -f
pgrep -af <name>
```

先看服务状态是不是 `active (running)`，最近 100 行日志里有没有启动失败、配置错误、端口占用、权限错误。`pgrep -af` 用来确认进程是否真的存在，以及启动参数是不是预期的。

---

### 2. 端口有没有开

**使用场景**：服务进程存在，但客户端连不上；启动时报 `Address already in use`；不确定服务监听的是本机、内网还是所有网卡。

```bash
ss -lntp
ss -lntp | grep ':8080'
lsof -i :8080
```

重点看有没有 `LISTEN`，监听地址是 `0.0.0.0`、`127.0.0.1` 还是具体内网 IP。服务只监听 `127.0.0.1` 时，本机能访问，外部机器通常访问不到。

---

### 3. 从外部能不能访问

**使用场景**：本机服务看起来正常，但浏览器、调用方、其他机器访问失败；需要区分是 DNS、网络、端口、HTTP 还是服务端处理慢。

```bash
nc -zv <ip> <port>
curl -Iv http://<host>:<port>/
dig <domain>
ping -c 4 <ip>
```

`nc` 看 TCP 端口能不能连上，`curl -Iv` 看 HTTP 状态码、重定向和握手过程，`dig` 看域名是否解析到预期 IP。`ping` 只作为辅助参考，很多云服务器会禁 ICMP，ping 不通不代表 TCP 一定不通。

HTTP 慢时再拆阶段：

```bash
curl -o /dev/null -s -w "dns: %{time_namelookup}s\ntcp: %{time_connect}s\ntls: %{time_appconnect}s\nttfb: %{time_starttransfer}s\ntotal: %{time_total}s\n" https://example.com
```

`dns` 高查 DNS；`tcp` 高查网络和端口；`ttfb` 高多半是服务端处理慢；`total - ttfb` 高再看响应体传输和带宽。

---

### 4. 机器是不是没资源

**使用场景**：服务没有明显报错，但响应变慢、机器卡顿、CPU/内存/磁盘告警，先快速判断是不是系统资源不够。

```bash
top
uptime
free -h
df -h
```

日常先看这四个就够了：`top` 看谁吃 CPU/内存，`uptime` 看负载趋势，`free -h` 看 `available` 和 Swap，`df -h` 看磁盘是否满。`top` 里常用 `1` 展开 CPU 核，`P` 按 CPU 排序，`M` 按内存排序，`H` 显示线程。

---

### 5. 磁盘满了先找大目录

**使用场景**：报警提示磁盘空间不足，服务写日志失败，数据库/中间件报 no space left on device。

```bash
df -h
du -sh .
du -sh ./*
du -xhd1 / 2>/dev/null | sort -hr | head -20
du -xhd1 /var 2>/dev/null | sort -hr | head -20
find /var/log -type f -size +100M -printf '%s %p\n' 2>/dev/null | sort -nr | head -20
```

先用 `df -h` 找哪个挂载点满了，再用 `du -xhd1` 只看下一层目录大小，逐层往下找。`2>/dev/null` 是把权限不足之类的错误输出丢掉，避免刷屏。

如果 `df -h` 显示满了，但 `du` 找不到对应的大文件，常见原因是文件已经被删除、但还被进程占着：

```bash
lsof +L1
```

---

### 6. 日志里先捞错误

**使用场景**：服务启动失败、接口报错、请求超时、进程退出但原因不明，先从最近日志和内核事件里找直接证据。

```bash
tail -n 200 /var/log/xxx.log
tail -f /var/log/xxx.log
grep -iE 'error|exception|failed|timeout|refused|oom' /var/log/xxx.log | tail -50
dmesg -T | tail -50
```

应用日志先看最近报错；systemd 服务优先用 `journalctl`；怀疑 OOM、磁盘错误、TCP 异常、驱动问题时再看 `dmesg -T`。

---

## L2 高阶诊断篇

> **适用场景**：系统负载飙升、偶发性延迟、无明显报错但疯狂卡顿。

当线上出现诡异性能问题时，按以下顺序执行，可在 **60 秒内**快速框定瓶颈在 CPU、内存、磁盘还是网络。

### 01. uptime — 全局负载定界

```bash
$ uptime
 16:40:00 up 10 days,  4 users,  load average: 12.05, 8.50, 4.20
```

- **当前状态**：`16:40:00` 是当前系统时间，`up 10 days` 表示已经连续运行 10 天，`4 users` 表示当前登录会话数。排障时先看这几项，可以快速判断是否刚重启、是否有人正在登录操作。
- **load average**：`12.05, 8.50, 4.20` 分别是最近 1 / 5 / 15 分钟的平均负载。它表示“正在运行或等待资源的任务数”，在 Linux 上不仅包含等待 CPU 的任务，也包含卡在不可中断 I/O 里的任务，所以高负载不一定等于 CPU 高。
- **和 CPU 核数对比**：先用 `nproc` 看 CPU 核数。若 8 核机器上 1 分钟负载长期在 12 左右，说明任务需求已经超过机器承载；若 32 核机器上负载 12，通常还不算满。
- **看趋势**：`12.05 > 8.50 > 4.20` 说明负载正在上升；如果是 `4.20, 8.50, 12.05`，说明高峰可能已经过去。1 分钟值看现场，15 分钟值看背景压力。
- **下一步判断**：`uptime` 只能告诉你“系统有压力”，不能告诉你压力来自哪里。后面用 `vmstat` 区分 CPU 队列和 I/O 阻塞，用 `pidstat` 找具体进程，用 `iostat` 或 `sar` 继续定位磁盘和网络。

### 02. dmesg | tail — 内核级事件捕获

```bash
$ dmesg -T | tail -n 10
[Tue Mar  3 10:21:34 2026] perl invoked oom-killer: gfp_mask=0x280da, order=0, oom_score_adj=0
[...]
[Tue Mar  3 10:21:34 2026] Out of memory: Kill process 18694 (perl) score 246 or sacrifice child
[Tue Mar  3 10:21:34 2026] Killed process 18694 (perl) total-vm:1972392kB, anon-rss:1953348kB, file-rss:0kB
[Tue Mar  3 10:24:16 2026] TCP: Possible SYN flooding on port 7001. Dropping request.  Check SNMP counters.
```

- **看点**：查看内核环形缓冲区里的最近事件。`-T` 会把默认的“开机后经过多少秒”转换成人类可读时间，方便和应用日志、监控报警时间对齐。
- **OOM / 内存耗尽**：看到 `invoked oom-killer`、`Out of memory`、`Killed process`，说明不是应用自己正常退出，而是内核因为内存不足主动杀掉了进程。重点看被杀进程名、PID、`anon-rss`、`total-vm`，再回到应用和容器内存限制排查。
- **网络异常**：看到 `TCP: Possible SYN flooding`、`time wait bucket table overflow`、`nf_conntrack: table full`，通常说明连接建立、短连接、连接跟踪或防火墙状态表压力过大。下一步看 `ss -s`、`sar -n TCP,ETCP 1` 和连接数分布。
- **磁盘 / 文件系统 / 硬件错误**：看到 `I/O error`、`EXT4-fs error`、`Buffer I/O error`、`Hardware Error`，不要只盯应用日志，先确认磁盘、文件系统或宿主机硬件是否有问题。
- **权限与安全拦截**：看到 `audit`、`apparmor`、`SELinux`、`denied`，可能是系统安全策略拦截了文件、端口或系统调用访问。

### 03. vmstat 1 — 系统状态快照

每秒采样一次系统运行状态，用来同时观察 CPU 队列、内存、Swap、块设备 I/O 和内核调度活动。

```bash
$ vmstat -S M 1
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 0  0      9  11549    233   1256    0    0   391   193 1630    0  0  0 100  0  0  0
```

- **procs / 进程队列**
  - **r (run queue)**：正在运行或正在等待 CPU 的进程数。它只反映 CPU 维度的排队，不包含阻塞在磁盘 I/O 上的进程。若 `r` 长时间大于 CPU 核数，通常说明 CPU 已经不够用了。
  - **b (blocked)**：处于不可中断睡眠的进程数，最常见原因是等待磁盘、网络文件系统等 I/O 返回。若 `b` 持续大于 0，同时 `wa` 或 `bi`/`bo` 偏高，优先怀疑 I/O 瓶颈。
- **memory / 内存**
  - **swpd**：已经使用的 Swap 大小。它是存量指标，不代表此刻正在换页；判断是否正在抖动要看 `si`/`so`。
  - **free**：完全空闲、尚未被使用的物理内存。
  - **buff**：用于块设备元数据、文件系统元数据等缓冲区的内存。
  - **cache**：用于文件页缓存的内存。Linux 会尽量把空闲内存拿来做缓存，所以 `free` 低不一定代表内存不足，要结合 `available`、`si`/`so` 判断。
- **swap / 换页**
  - **si (swap in)**：每秒从 Swap 读回内存的数据量。持续大于 0 说明系统正在把被换出的内存页读回来。
  - **so (swap out)**：每秒写入 Swap 的数据量。持续大于 0 说明物理内存压力较大，系统正在把内存页挪到磁盘上。
- **io / 块设备 I/O**
  - **bi (blocks in)**：每秒从块设备读入的数据量，可近似理解为磁盘读吞吐。
  - **bo (blocks out)**：每秒写到块设备的数据量，可近似理解为磁盘写吞吐。
- **system / 内核活动**
  - **in (interrupts)**：每秒中断次数，包括时钟中断、设备中断等。异常偏高时，可能与网卡、磁盘、中断风暴有关。
  - **cs (context switches)**：每秒上下文切换次数。异常偏高时，可能是线程太多、锁竞争、频繁唤醒或 I/O 等待导致调度开销变大。
- **cpu / CPU 时间占比**
  - **us (user)**：用户态消耗的 CPU 时间，通常是应用代码在跑。
  - **sy (system)**：内核态消耗的 CPU 时间，例如系统调用、网络协议栈、文件系统、驱动处理。持续偏高说明内核开销重，需要结合 `pidstat`、`perf`、网络或磁盘指标继续看。
  - **id (idle)**：CPU 空闲时间。`id` 很低且 `r` 很高，通常是 CPU 饱和。
  - **wa (iowait)**：CPU 因等待 I/O 完成而空闲的时间。`wa` 持续偏高时，说明任务不是没事做，而是在等磁盘或其他块设备返回。
  - **st (steal)**：虚拟机被宿主机或同宿主机其他虚拟机“偷走”的 CPU 时间。云服务器上 `st` 偏高，常见于宿主机超卖或邻居实例抢占严重。
  - **gu (guest)**：CPU 运行虚拟机客户系统所花的时间，通常只有宿主机或虚拟化场景下才需要重点关注。

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

- **看点**：`all` 行是所有 CPU 的平均值，下面的 `0`、`1`、`2`、`3` 是每个逻辑 CPU 的单独统计。`top` 只能告诉你整体忙不忙，`mpstat -P ALL 1` 能看出负载是否均匀分布到每个核心。
- **字段含义**：`%usr` 是用户态 CPU，通常对应应用代码；`%nice` 是被 `nice` / `renice` 调整过优先级的用户态进程消耗的 CPU；`%sys` 是内核态 CPU，通常对应系统调用、协议栈、文件系统等内核开销；`%iowait` 是等待 I/O 完成的时间；`%irq` 是硬中断；`%soft` 是软中断，网络收发包压力大时常会升高；`%steal` 是虚拟机被宿主机抢走的 CPU；`%guest` 是运行虚拟机客户系统消耗的 CPU；`%gnice` 是被 `nice` 调整过优先级的虚拟机客户系统消耗的 CPU；`%idle` 是空闲时间。
- **nice / gnice 怎么看**：`nice` 是 Linux 的进程优先级调节手段，常用来让备份、压缩、批处理这类低优先级任务“有空再跑”。所以 `%nice` 高，说明 CPU 正在被这类被调权的普通进程使用；`%gnice` 是同一个概念在虚拟机 guest 上的统计，普通物理机或不跑虚拟化时通常为 0。日常排障里它们多数不重要，除非你怀疑低优先级任务抢了 CPU，或者宿主机上有虚拟机负载。
- **整体 CPU 打满**：如果 `all` 行里 `%idle` 接近 0，同时 `%usr` 很高，说明 CPU 主要被应用代码吃掉，下一步用 `pidstat 1` 找具体进程。
- **单核打满**：如果只有某一个 CPU 的 `%usr` 接近 100%，其他核心很空，通常是单线程任务、线程绑定 CPU、锁竞争导致并行度上不去，或应用没有把负载分摊到多个 worker。
- **中断不均衡**：如果某一个 CPU 的 `%soft` 或 `%irq` 明显高于其他核心，常见原因是网卡中断、软中断或队列没有均衡分布。下一步看 `/proc/interrupts`、网卡多队列、RSS/RPS/XPS、`irqbalance` 是否正常。
- **不是 CPU 算力问题**：如果 `%iowait` 高，说明 CPU 空着等 I/O，不该只盯应用 CPU；如果 `%steal` 高，说明虚拟化层在抢 CPU，需要关注云主机宿主机争用或实例规格。

### 05. pidstat 1 — 微观进程 CPU 溯源

```bash
$ pidstat 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015    _x86_64_    (32 CPU)

07:41:02 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
07:41:03 PM     0         9    0.00    0.94    0.00    0.00    0.94     1  rcuos/0
07:41:03 PM     0      4214    5.66    5.66    0.00    0.00   11.32    15  mesos-slave
07:41:03 PM     0      6521 1596.23    1.89    0.00    0.01 1598.11    27  java
07:41:03 PM     0      6564 1571.70    7.55    0.00    0.00 1579.25    28  java
07:41:03 PM 60004     60154    0.94    4.72    0.00    0.00    5.66     9  pidstat
```

- **看点**：`pidstat 1` 每秒按进程输出一次 CPU 使用情况，用来回答“到底哪个 PID 在吃 CPU”。它比 `top` 更适合留现场，因为输出是一行行追加的，方便复制、保存和事后对齐时间。
- **字段含义**：`UID` 是进程所属用户，`PID` 是进程 ID；`%usr` 是进程在用户态消耗的 CPU，通常是业务代码；`%system` 是进程在内核态消耗的 CPU，通常是系统调用、网络、文件 I/O 等；`%guest` 是运行虚拟机客户系统消耗的 CPU；`%wait` 是进程等待 CPU 调度的时间比例，高的时候说明进程想跑但没抢到 CPU；`%CPU` 是该进程总 CPU 使用率；`CPU` 是采样时进程正在运行的逻辑 CPU 编号；`Command` 是进程名。
- **UID / 用户名**：`pidstat` 默认显示数字 UID。想看用户名，可以用 `pidstat -U 1`，这时第一列会从 `0`、`33`、`1000` 变成 `root`、`www-data`、`me` 这类用户名；也可以用 `getent passwd <UID>` 查 UID 对应的账户，或用 `ps -o user= -p <PID>` 直接查某个 PID 的运行用户。
- **为什么会超过 100%**：`%CPU` 是按所有 CPU 核心累加的。1 个核心打满约等于 100%，16 个核心打满约等于 1600%。示例里 `java` 的 `%CPU` 是 `1598.11`，说明它大约吃掉了 16 个核心。
- **用户态高还是内核态高**：如果 `%usr` 高，重点看应用计算、死循环、热点函数；如果 `%system` 高，重点看系统调用是否过多、网络包处理、文件 I/O、锁或线程调度开销。
- **继续拆线程**：如果一个多线程进程很高，比如 Java、Nginx、数据库，下一步用 `pidstat -t -p <PID> 1` 看这个进程内部哪个线程在吃 CPU，再结合线程栈、火焰图或 `perf top` 定位代码热点。

### 06. iostat -xz 1 — 块设备 IO 饱和度解剖

```bash
$ iostat -xz 1
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.16    0.00    0.26    0.02    0.00   99.56

Device            r/s     rkB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wkB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dkB/s   drqm/s  %drqm d_await dareq-sz     f/s f_await  aqu-sz  %util
sdd             22.81    372.51     5.93  20.63    0.22    16.33    2.61    169.44     6.87  72.51    3.34    65.02    0.26    397.27     0.03  11.40    0.12  1531.17    0.47    2.35    0.01   0.82
```

- **先看采样方式**：建议用 `iostat -xz 1` 看实时变化。只执行 `iostat -xz` 时，第一屏通常是从开机到现在的累计平均值，容易把短时尖峰抹平。
- **avg-cpu**：上方 `avg-cpu` 是 CPU 总览。`%iowait` 高说明 CPU 经常在等块设备 I/O 返回；如果设备指标很忙但 `%iowait` 不高，可能是 I/O 压力不大，或负载主要被异步 I/O、缓存、虚拟化层吸收了。
- **读 I/O**：`r/s` 是每秒读请求数，`rkB/s` 是每秒读吞吐，`r_await` 是读请求平均耗时，`rareq-sz` 是平均每次读请求大小。读多但 `r_await` 低，通常只是正常吞吐；读多且 `r_await` 高，才更像读瓶颈。
- **写 I/O**：`w/s` 是每秒写请求数，`wkB/s` 是每秒写吞吐，`w_await` 是写请求平均耗时，`wareq-sz` 是平均每次写请求大小。写延迟高时，常见原因是日志刷盘、数据库落盘、文件系统同步写或底层存储慢。
- **合并请求**：`rrqm/s`、`wrqm/s` 是每秒被合并的读写请求数，`%rrqm`、`%wrqm` 是合并比例。合并高不一定是坏事，说明内核把相邻 I/O 合成了更大的请求；但也说明上层可能在发大量小 I/O。
- **discard / trim**：`d/s`、`dkB/s`、`d_await`、`dareq-sz` 是 discard/TRIM 相关指标，常见于 SSD、虚拟磁盘或文件删除、空间回收场景。平时多数为 0，异常升高时要看是否有大规模删除、fstrim 或存储回收。
- **flush**：`f/s` 和 `f_await` 表示 flush 请求及其平均耗时，反映把缓存数据强制刷到稳定存储的成本。数据库、日志系统、`fsync` 密集场景里，`f_await` 高会直接放大写延迟。
- **队列深度**：`aqu-sz` 是平均 I/O 队列长度，旧版可能叫 `avgqu-sz`。持续升高说明请求在排队；如果 `await` 也一起升高，基本可以认为设备或底层存储响应跟不上。
- **设备利用率**：`%util` 是设备忙碌时间比例。接近 100% 通常说明设备一直有 I/O 在处理，但在 SSD、NVMe、RAID、云盘、WSL2 虚拟磁盘这类能并行处理请求的设备上，不能只凭 `%util` 判定饱和，要结合 `await`、`aqu-sz` 和业务延迟一起看。

### 07. free -m — 内存快速确认

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       1.3Gi        12Gi       3.7Mi       1.6Gi        14Gi
Swap:          4.0Gi          0B       4.0Gi
```

- **看点**：快速确认内存是否真的紧张。优先看 `available`，它表示系统在不明显影响当前进程的情况下，大致还能分配给新程序的内存，比单看 `free` 更可靠。
- **`buff/cache`**：这是 Linux 用空闲内存做的文件缓存和块设备缓存，正常情况下可以偏高，内存紧张时也会被回收给进程使用。不要把 `free` 很低直接理解为内存不够，Linux 倾向于把暂时不用的内存拿去做缓存。
- **Swap**：如果 `available` 很低，同时 `Swap` 已经被使用，再结合前面 `vmstat` 的 `si`/`so`（Swap 换入换出）频繁升高，说明系统正在把内存页换到磁盘上，性能很容易明显下降。
- **磁盘 I/O 关联**：当内存不足、页缓存被持续挤压时，应用可能更频繁地访问磁盘；此时再用 `iostat` 看 `await`、`aqu-sz`、`%util`，确认是否已经出现磁盘等待或排队。

### 08. sar -n DEV 1 — 网卡吞吐量定界

```bash
$ sar -n DEV 1
Linux 6.6.87.2-microsoft-standard-WSL2 (LAPTOP-HOAN8M13) 	06/19/26 	_x86_64_	(32 CPU)

14:04:40        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
14:04:41           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
14:04:41         eth0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

- **看点**：按网卡观察流量方向和规模。`rxkB/s` 是每秒接收吞吐，`txkB/s` 是每秒发送吞吐；`rxpck/s`、`txpck/s` 是每秒收发包数量。吞吐高说明大流量多，包速率高但吞吐不高则可能是大量小包。
- **接口选择**：排查业务流量时重点看真实业务网卡，例如 `eth0`、`ens*`、`bond*`。`lo` 是本机回环接口，只反映本机内部通信；容器、虚拟机、WSL2、云主机里还可能看到虚拟网卡，判断时要先确认流量实际走哪块接口。
- **带宽判断**：将 `rxkB/s`、`txkB/s` 换算成 MB/s 后，对比网卡、云主机规格或负载均衡限额。千兆网卡理论上限约 125 MB/s，但实际可用值会低一些；如果吞吐长期逼近上限，同时业务出现超时或下载变慢，才更像带宽瓶颈。
- **`%ifutil`**：表示接口利用率，但依赖系统能否正确识别网卡速率。虚拟网卡、云网络、bond、WSL2 环境里这个值可能不准，不能只凭它判断是否打满。
- **继续定位**：如果吞吐不高但连接仍然慢，继续看 `sar -n TCP,ETCP 1` 的重传、连接创建速率，再结合 `ss`、`tcpdump` 判断是网络质量、连接堆积，还是应用处理慢。

### 09. sar -n TCP,ETCP 1 — TCP 协议栈健康度

```bash
$ sar -n TCP,ETCP 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015    _x86_64_    (32 CPU)

12:17:19 AM  active/s passive/s    iseg/s    oseg/s
12:17:20 AM      1.00      0.00  10233.00  18846.00

12:17:19 AM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
12:17:20 AM      0.00      0.00      0.00      0.00      0.00
```

- **看点**：`TCP` 行看连接建立和 TCP 段收发，`ETCP` 行看异常事件。它适合用来判断“网络慢”到底是新连接多、数据量大、重传高，还是连接失败/被重置。
- **连接建立**：`active/s` 是本机每秒主动发起的连接数，常见于应用访问数据库、缓存、下游 HTTP 服务；`passive/s` 是本机每秒被动接受的连接数，常见于 Web 服务收到客户端请求。两者突然升高，说明连接创建压力变大，可继续用 `ss -s` 看连接状态分布。
- **段收发量**：`iseg/s` 是每秒收到的 TCP 段数，`oseg/s` 是每秒发出的 TCP 段数。它们反映协议栈层面的包量，不等同于业务请求数；如果包量很高但 `sar -n DEV` 吞吐不高，可能是大量小包。
- **重传**：`retrans/s` 是每秒 TCP 重传段数。持续升高通常表示链路丢包、网络拥塞、对端处理慢、队列溢出，或本机/对端缓冲区压力大。它是重要信号，但不能单独判断问题一定在网络侧，需要结合 `tcpdump`、`ss -ti`、业务延迟和对端指标确认。
- **失败和重置**：`atmptf/s` 是主动连接失败次数，`estres/s` 是已建立连接被重置次数，`orsts/s` 是本机发出的 RST 数。它们升高时，重点排查端口未监听、连接被应用主动关闭、负载均衡/防火墙重置、连接池配置或服务过载。

### 10. top — 全局状态与进程热点复核

```bash
$ top
top - 16:32:18 up 18 days,  2 users,  load average: 10.84, 8.21, 5.66
Tasks: 238 total,   6 running, 229 sleeping,   1 stopped,   2 zombie
%Cpu(s): 52.4 us, 10.8 sy,  4.1 ni, 18.6 id,  8.7 wa,  0.6 hi,  2.3 si,  2.5 st
MiB Mem :  32064.0 total,   1240.5 free,  21480.2 used,   9343.3 buff/cache
MiB Swap:   8192.0 total,   6120.0 free,   2072.0 used.   7688.4 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
  28431 app       20   0   12.4g   5.8g   420m R 385.6 18.5   1284:33 java
  28792 app       20   0    4.2g   1.1g   128m S  72.4  3.5     92:18 nginx
  28110 mysql     20   0   18.8g  10.6g   312m S  48.7 33.8    411:02 mysqld
  29041 root      20   0       0      0      0 S  16.5  0.0     18:41 ksoftirqd/3
  30188 backup    30  10    912m   180m    24m R  14.2  0.6      6:12 gzip
  30211 app       20   0    648m    92m    18m Z   0.0  0.0      0:00 worker <defunct>
```

- **看点**：`top` 适合做现场总览，把负载、CPU、内存、任务状态和最耗资源的进程放在同一屏里看。它不一定比前面的命令更精确，但能快速确认“谁最可疑”。
- **负载和任务**：`load average` 是 1 / 5 / 15 分钟平均负载，要和 CPU 核数一起看。若这是 8 核机器，示例里的 `10.84` 已经超过 CPU 核数，说明任务在排队；`Tasks` 里 `running` 多代表正在争 CPU，`zombie` 持续存在则要查父进程是否没有回收子进程。
- **CPU 行**：`us` 是用户态程序占用，`sy` 是内核态占用，`ni` 是调整过 nice 优先级的进程占用，`id` 是空闲，`wa` 是等待 I/O，`hi` 是硬中断，`si` 是软中断，`st` 是虚拟机被宿主机抢走的 CPU 时间。示例中 `us` 高说明应用忙，`wa` 不低说明也有 I/O 等待，`si` 偏高可能和网络包处理有关，`st` 偏高则提示虚拟化层也在影响性能。
- **内存行**：`buff/cache` 是内核用于文件缓存、块设备缓存等用途的内存，内存紧张时可以回收一部分给进程使用；`avail Mem` 是更适合判断“还能不能继续分配内存”的指标。判断内存是否紧张时不要只看 `free`，要结合 `avail Mem`、Swap 使用量和 `vmstat si/so`。
- **进程列**：`PID` 是进程号，`USER` 是运行用户，`PR` 是调度优先级，`NI` 是 nice 值，数值越高越“礼让”CPU；`VIRT` 是进程可访问的虚拟内存总量，`RES` 是实际常驻物理内存，`SHR` 是可共享内存；`S` 是进程状态，常见 `R` 运行、`S` 睡眠、`D` 不可中断 I/O 等待、`Z` 僵尸；`TIME+` 是进程累计消耗的 CPU 时间。
- **热点判断**：`%CPU` 在多核机器上可能超过 100%，例如 8 核机器上一个进程最高可接近 `800%`。示例里的 `java` 占用 `385.6%`，约等于吃掉 4 个 CPU 核；`ksoftirqd/3` 占用较高时要联想到软中断和网络包处理；`NI=10` 的 `gzip` 是被降低优先级的后台任务；`worker <defunct>` 是僵尸进程，真正要查的是它的父进程。
- **下一步**：CPU 热点用 `pidstat -p <PID> 1` 或 `top -H -p <PID>` 看线程；I/O 等待用 `iostat -xz 1`；软中断高用 `sar -n DEV 1`、`cat /proc/softirqs`；`st` 高则需要看宿主机、云主机规格或同宿主机资源争用。



---

## 总结：排障入口

现场排障先别急着把所有命令跑一遍。先回答三个问题：系统是不是有压力，压力落在哪类资源上，具体是谁制造了压力。

| 现象 | 先看 | 关键判断 | 下一步 |
|:---|:---|:---|:---|
| **机器整体变慢、负载高** | `uptime`, `top` | `load average` 是否超过 CPU 核数；`top` 里是 `us/sy` 高，还是 `wa`、`st` 高 | 用 `vmstat 1` 区分 CPU 队列、I/O 等待和 Swap；再用 `pidstat 1` 找进程 |
| **CPU 被打满** | `top`, `mpstat -P ALL 1` | 是所有核心都忙，还是单核热点；`us` 高偏应用，`sy` 高偏内核开销 | 用 `pidstat -p <PID> 1`、`top -H -p <PID>` 查线程；再结合应用栈定位代码 |
| **内存紧张或 OOM** | `free -h`, `dmesg -T \| tail` | `available/avail Mem` 是否很低；Swap 是否使用；是否出现 OOM kill | 用 `vmstat 1` 看 `si/so` 是否持续升高；再查进程 RSS 和应用内存配置 |
| **磁盘响应慢** | `iostat -xz 1`, `vmstat 1` | `await` 是否高，`aqu-sz` 是否排队，`%util` 是否长期偏高；`wa` 是否升高 | 区分读慢还是写慢，看 `r_await/w_await`、请求大小和吞吐；再定位具体进程或存储层 |
| **网络请求慢或超时** | `curl -w`, `ss`, `sar -n DEV 1` | 慢在 DNS、建连、TLS、首包，还是传输；网卡吞吐或包速率是否异常 | 用 `sar -n TCP,ETCP 1` 看重传和重置；必要时用 `tcpdump` 抓包确认 |
| **连接数异常或端口问题** | `ss -lntp`, `ss -ant`, `ss -s` | 服务是否监听；连接是否堆在 `ESTAB`、`TIME-WAIT`、`SYN-SENT`、`CLOSE-WAIT` | 结合应用日志、连接池配置、负载均衡和 `sar -n TCP,ETCP 1` 继续查 |
| **磁盘空间不足** | `df -h`, `du -sh <path>` | 是分区满，还是某个目录/日志文件增长异常；inode 是否耗尽 | 清理或轮转日志前先确认归属；空间释放后再观察 `iostat` 是否仍有压力 |
| **DNS 解析异常** | `dig`, `nslookup` | 是本机解析失败、权威解析异常，还是解析结果不符合预期 | 用 `dig +trace` 追解析链路；同时检查 `/etc/resolv.conf`、DNS 服务和缓存 |

核心思路：先用 `top`、`uptime`、`vmstat` 定界方向，再用 `pidstat`、`iostat`、`sar`、`ss` 把问题缩小到进程、设备或网络连接。不要只看单个指标，至少用两个角度互相印证。

---

## 参考资料

- [Linux Performance Analysis in 60,000 Milliseconds — Netflix Technology Blog](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)
- [Netflix_Linux_Perf_Analysis_60s.pdf](https://www.brendangregg.com/Articles/Netflix_Linux_Perf_Analysis_60s.pdf)
