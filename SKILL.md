---
name: perf-cpu100
description: 线上 CPU 飙高（如 90%+）/响应时间暴涨的排查技能：strace 定方向 → perf 找热点+火焰图 → ftrace 入内核，热点定位后用 TMA 四分类判读瓶颈（Frontend/Backend/Bad Speculation/Retiring）并按 playbook 修复（内存/向量化/分支/伪共享等），附 D 状态（kill -9 无效）进程排查与整机 CPI 突增的 split lock/总线锁排查。Use when：CPU 使用率高、load 高、服务变慢需区分业务代码/系统调用/内核开销时；热点函数已找到但不知为什么慢、怎么优化时；多线程 CPU 高但吞吐低（自旋/伪共享）时；整机所有核 CPI 突增怀疑总线锁（bus lock/split lock）时。即使用户只说"服务变卡""top 里进程吃 CPU""线程空转""这个函数为什么慢""所有核一起变慢"而未提任何性能工具，也要使用。Do NOT use：数据库锁（MySQL 行锁/分布式锁）、内存泄漏 OOM、网络丢包、容量规划问题。
---

# perf-cpu100：CPU 飙高性能问题排查（strace + perf + ftrace 三板斧）

来源：《线上服务CPU飙到90%，我用perf+ftrace+strace三板斧，20分钟从用户态追到内核态》
典型场景：4 核机器 CPU 使用率飙到 90%+，响应时间从 20ms 飙到 800ms，top 只能看到业务进程吃 CPU，但无法区分是业务代码、系统调用慢、还是内核堵塞。

## 核心方法论：三层工具各看一层

| 工具   | 层级            | 粒度         | 定位                             |
| ------ | --------------- | ------------ | -------------------------------- |
| strace | 用户态/内核边界 | 系统调用     | 快速定方向，5 分钟出结论，0 门槛 |
| perf   | CPU 级          | 采样热点函数 | 找出热点在哪，火焰图可视化       |
| ftrace | 内核内部        | 函数级追踪   | 终极武器，看内核到底在干什么     |

排查顺序：**从粗到细、从外到内**。大部分线上问题 strace + perf 就够了，ftrace 留给需要深入内核的疑难杂症。

## 排障决策树

```
                    线上问题
                       │
           ┌───────────┴───────────┐
           │                       │
      CPU 高/响应慢            进程卡死/D 状态
           │                       │
     ┌─────┴─────┐       cat /proc/<pid>/stack
     │           │        看内核调用栈
  strace -c   perf top           │
  看系统调用   看热点函数    ┌────┴────┐
   分布        分布         │         │
     │           │       I/O 等待   锁等待
     │      perf record     │         │
     │      生成火焰图    iostat    ftrace
     │           │       blktrace  kprobe
     │      ┌────┴────┐
     │      │         │
     │  用户态热点  内核态热点
     │      │         │
     │  优化业务代码  ftrace
     │             function_graph
     │             深入分析
     │
  ┌──┴──┐
  │     │
futex  read/write
锁竞争  I/O 慢
```

strace -c 结果的分支解读：

- **futex 占比高** → 锁竞争问题
- **read/write 占比高** → I/O 慢问题

## 执行 SOP：拿到问题按此顺序走

0. **先定位目标再 attach**：多线程程序必须先找到吃 CPU 的具体线程：
   ```bash
   top -Hp <pid>           # 看进程内哪个线程吃 CPU
   pidstat -t -p <pid> 1 5
   ```
   后续 strace/perf 用该 TID 定向：`strace -f -p <tid>`、`perf record --tid=<tid>`；进程级命令仅用于线程定位后的全景确认。
1. **strace -c 定方向**（timeout 10 采样）：futex 高 → 锁竞争路径；read/write 高 → I/O 路径（执行 `iostat -xz 1 5` 看 await/%util，异常时：有 D 状态进程转"番外：D 状态排查"五步；无 D 状态进程走 blktrace 路径：`trace-cmd record -e block:block_rq_issue -e block:block_rq_complete sleep 10` 定位慢设备）
2. **perf top 看热点**，按符号前缀判定分支：
   - `[k]` 前缀 = 内核态热点 → 升级 ftrace（function_graph）
   - `[.]` 或模块名（如 your_app、libc.so.6）= 用户态热点 → 查业务代码
3. **perf record + 火焰图**确认完整调用链，用户态热点直接改代码
4. **内核态热点用 ftrace 深入**
5. **修复后必须验证**（见"验证"章节），未达标回到步骤 1

每完成一步向用户汇报发现、确认方向后再继续，不要一次挂满所有工具。

步骤 3 的"直接改代码"若不知从何下手：先用 TMA 判读瓶颈类型再查修复手册，见下文"定位热点之后：TMA 判读 + 修复 playbook"章节。

## 第一板斧：strace——先看进程在干什么

遇到线上问题别急着上重武器，先用 strace 摸底，看它在做什么系统调用、卡在哪。

### 快速定位：哪个系统调用最慢

```bash
# -p 附加到运行中的进程，-c 统计汇总，-T 显示每次调用耗时
strace -p $(pidof your_app) -c -T -f 2>&1 | head -60
```

跑 10 秒钟 Ctrl+C 中断，输出示例：

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 78.32    2.847129       14235       200           futex
  9.21    0.334892          41      8127           write
  5.44    0.197888          24      8127           read
  3.11    0.113024          55      2048           epoll_wait
```

判读：futex 吃掉 78% 时间、平均每次 14ms——正常 futex 应该很快，14ms 明显不对。

### 追踪单个系统调用的详细参数

```bash
# -e trace=futex 只跟踪 futex 调用，-tt 打印微秒级时间戳
strace -p $(pidof your_app) -e trace=futex -tt -T -f 2>&1 | head -30
```

输出示例：

```
15:03:22.847291 futex(0x7f3a8c0012d0, FUTEX_WAIT_PRIVATE, 0, {tv_sec=0, tv_nsec=999999000}) = -1 ETIMEDOUT (Connection timed out) <0.014523>
15:03:22.862018 futex(0x7f3a8c0012d0, FUTEX_WAIT_PRIVATE, 0, {tv_sec=0, tv_nsec=999999000}) = -1 ETIMEDOUT (Connection timed out) <0.014891>
```

判读：一堆 futex 做 FUTEX_WAIT_PRIVATE 且全是超时返回 → 有线程反复尝试获取锁但一直拿不到 → 锁竞争问题。

### strace 生产环境注意事项（5 个坑）

```bash
# 1. 一定要加 -f 跟踪子线程，否则多线程程序只能看到主线程
strace -f -p <pid>

# 2. 生产环境用 -c 做统计就好，不要直接 trace 全部调用
#    strace 用的是 ptrace，会让进程变慢 2-10 倍
strace -c -p <pid>

# 3. 如果要详细看某个调用，用 -e 过滤，减少开销
strace -e trace=network -p <pid>  # 只看网络相关
strace -e trace=file -p <pid>     # 只看文件操作

# 4. 输出到文件，别直接打屏幕
strace -o /tmp/strace.log -p <pid>

# 5. 设置超时自动退出，防止忘了关
timeout 10 strace -c -p <pid>
```

strace 优点：简单直接、0 门槛。缺点：性能开销大（ptrace 机制决定），只适合短时间采样看方向。

## 第二板斧：perf——揪出 CPU 热点函数

perf 是 Linux 内核自带的性能分析工具，基于硬件性能计数器（PMU）做采样，开销很小，生产环境可用。

### perf top：实时看 CPU 热点

```bash
# 实时查看系统级 CPU 热点函数
perf top -g

# 只看某个进程
perf top -p $(pidof your_app) -g
```

类似 htop 但显示函数级 CPU 占用。输出示例：

```
  35.21%  your_app       [.] SpinLock::lock()
  18.44%  your_app       [.] ConnectionPool::getConn()
  12.03%  libc.so.6      [.] __pthread_mutex_lock
   8.92%  [kernel]       [k] _raw_spin_lock_irqsave
   6.11%  [kernel]       [k] futex_wait_queue
```

判读：SpinLock::lock() 占 35%（自旋锁疯狂空转）+ 连接池 getConn() 占 18% + 内核态锁等待 → 超过 60% CPU 花在"等锁"上。

### perf record + 火焰图：看完整调用链

```bash
# 采样 30 秒，-g 记录调用栈，--call-graph dwarf 用 DWARF 解析栈帧（推荐）
perf record -p $(pidof your_app) -g --call-graph dwarf -F 99 -- sleep 30

# 如果是 C/C++ 程序，也可以用 fp（帧指针）方式，开销更小
perf record -p $(pidof your_app) -g --call-graph fp -F 99 -- sleep 30
```

`-F 99` 每秒采样 99 次，选 99 而不是 100 是为了避免和定时器产生共振（Brendan Gregg 的建议）。

生成火焰图：

```bash
# 方式一：perf 自带（内核 5.8+）
perf script report flamegraph

# 方式二：Brendan Gregg 的 FlameGraph 工具（更通用）
perf script > /tmp/perf.out
git clone https://github.com/brendangregg/FlameGraph.git
cd FlameGraph
./stackcollapse-perf.pl /tmp/perf.out | ./flamegraph.pl > /tmp/flamegraph.svg
```

**火焰图读法四要点：**

- 越宽的"平顶"代表 CPU 占用越多——那就是热点
- 从下往上看是调用栈，底部是入口函数，顶部是实际执行的函数
- 关注那些宽且在顶部的函数，那才是真正消耗 CPU 的地方
- **颜色是随机的，不代表性能好坏**——新手常见误解，别按颜色下结论

### perf report：文字版热点（先于火焰图看）

不想开浏览器看 SVG，直接命令行看热点排序：

```bash
perf report --stdio --no-children -g none --percent-limit 1
```

输出示例：

```
  Overhead  Command    Symbol
   94.95%  perf_demo  [.] bubble_sort
    3.54%  perf_demo  [.] matrix_multiply
    1.22%  perf_demo  [.] hash_compute
```

`--percent-limit 1` 只显示占比 ≥1% 的函数，过滤噪音。先看文字报告锁定目标，再看火焰图看调用链——两步配合效率最高。

### perf stat：看硬件计数器

```bash
# 看 CPU 缓存命中率、分支预测失败率、IPC 等
perf stat -p $(pidof your_app) -- sleep 10
```

关键输出示例：

```
       45,231.42 msec task-clock                #    3.78 CPUs utilized
           12,847      context-switches          #  284.049 /sec
             234      cpu-migrations             #    5.174 /sec
           3,021      page-faults                #   66.807 /sec
   98,234,567,890      cycles                    #    2.17 GHz
   34,567,890,123      instructions             #    0.35  insn per cycle    <---
    6,789,012,345      branches                  # 150.082 M/sec
      234,567,890      branch-misses             #    3.45% of all branches
```

判读：IPC（Instructions Per Cycle）只有 0.35，正常应在 1.0 以上。这么低说明 CPU 大量时间在等待——要么等内存（cache miss），要么等锁（空转）。

> IPC 细分阈值（补充）：内存密集型应用 IPC 0–1 属正常，计算密集型 4–6；判读其它指标（MPKI、branch-misses%、Load Miss 延迟、DRAM 带宽）及 SMT/频率缩放/多路复用等度量陷阱见 `references/tma-metrics.md` §四/§六。

### perf stat --topdown / toplev：TMA 四分类，回答"为什么慢"

perf stat 只给一个总 IPC，分不清 CPU 时间浪费在取指、误预测还是等内存。TMA（Top-Down Microarchitecture Analysis）把 pipeline slot 浪费原因分成四桶，直接对应修复方向：

| 桶              | 含义              | 阈值                          | 修复                 |
| --------------- | ----------------- | ----------------------------- | -------------------- |
| Retiring        | 有用工作          | >80% 但仍慢 = 待向量化        | 向量化               |
| Frontend Bound  | 取指/译码瓶颈     | >20% 投入                     | 代码布局/PGO         |
| Bad Speculation | 分支误预测        | >10% 排查                     | cmov/查表            |
| Backend Bound   | 内存/执行单元等待 | 剩余大头，下钻 Memory vs Core | 数据结构/预取/依赖链 |

```bash
perf stat -- ./your_app        # Intel 新版 perf 默认输出 tma_* 四桶
perf stat -M PipelineL1,PipelineL2 -- ./your_app   # AMD Zen4+（内核 6.2+）
```

完整下钻流程、定位到源码行的精确事件采样命令、按平台的 toplev 用法：`references/tma-metrics.md`。

<!-- 来源: external/perf-book/chapters/6-CPU-Features-For-Performance-Analysis/ -->

### perf lock：专门分析锁竞争

```bash
# 录制锁事件
perf lock record -p $(pidof your_app) -- sleep 10

# 查看锁竞争统计
perf lock report
```

输出会告诉你哪个锁等待时间最长、争用次数最多、平均等待时长，比翻 dmesg 日志高效。

## 第三板斧：ftrace——深入内核追踪

strace 看系统调用入口/出口，perf 看采样统计。想知道某个内核函数到底干了什么、调用了哪些子函数、每步花多少时间——用 ftrace。

ftrace 是内核内置追踪框架，通过 `/sys/kernel/debug/tracing/`（或 `/sys/kernel/tracing/`）操作，生产环境可用，开销很小。

### 用 trace-cmd 简化操作

直接操作 tracefs 文件系统繁琐，推荐 trace-cmd：

```bash
apt install trace-cmd   # Debian/Ubuntu
yum install trace-cmd   # CentOS/RHEL
```

### function_graph：看内核函数调用树

```bash
# 追踪 futex 相关的内核函数调用，只看 pid=12345 的进程
trace-cmd record -p function_graph -l 'futex_*' -P 12345 sleep 5

# 查看结果
trace-cmd report | head -80
```

输出示例：

```
 your_app-12345  [002] 15032.847291: funcgraph_entry:        |  futex_wait_queue() {
 your_app-12345  [002] 15032.847293: funcgraph_entry:        |    schedule() {
 your_app-12345  [002] 15032.847294: funcgraph_entry:        |      __schedule() {
 your_app-12345  [002] 15032.847295: funcgraph_entry:   0.341 us  |        _raw_spin_lock();
 your_app-12345  [002] 15032.847296: funcgraph_entry:        |        dequeue_task_fair() {
 your_app-12345  [002] 15032.847297: funcgraph_entry:   0.215 us  |          update_curr();
 ...
 your_app-12345  [002] 15032.861814: funcgraph_exit:  + 14.523 ms |  }  /* futex_wait_queue */
```

判读：futex_wait_queue() 一次调用耗时 14.5ms，中间进了 schedule() 做进程切换——这就是内核态完整调用链。

### 追踪高延迟的 I/O 操作

```bash
# 追踪块设备 I/O 事件
trace-cmd record -e block:block_rq_issue -e block:block_rq_complete sleep 10
trace-cmd report | awk '{print $NF}' | sort | uniq -c | sort -rn | head
```

### 追踪调度延迟

怀疑调度器导致延迟高，用 `wakeup` tracer 看进程从被唤醒到真正运行的延迟：

```bash
echo wakeup > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | tail -20
```

判读：调度延迟经常超过几毫秒 → 可能 CPU 绑核不合理，或有高优先级任务抢占。

### 用 kprobe 动态插桩

不需要重启内核，想追踪哪个函数就追踪哪个：

```bash
# 追踪 tcp_sendmsg 函数，打印参数
trace-cmd record -e 'kprobe:tcp_sendmsg' -P 12345 sleep 5

# 或者手动操作 tracefs
echo 'p:my_probe tcp_sendmsg size=%dx' > /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/my_probe/enable
cat /sys/kernel/debug/tracing/trace_pipe
# 看完了记得关闭
echo 0 > /sys/kernel/debug/tracing/events/kprobes/my_probe/enable
echo > /sys/kernel/debug/tracing/kprobe_events
```

## 跨语言：Java 火焰图

perf + 火焰图不挑语言，但 Java 有特殊问题：**JIT 编译后的代码符号在 perf 里显示为地址**（`[perf-38966.map]`），看不到方法名。

```bash
# 用 perf 采样 Java 进程 → 火焰图里 99% 是 [perf-38966.map]，没用
perf record -F 99 -g -p <java_pid> -- sleep 30
```

**解决方案：async-profiler**，专门解决 JIT 符号问题：

```bash
# 下载（https://github.com/async-profiler/async-profiler）
./asprof -d 30 -f java_flamegraph.html <PID>
```

输出火焰图里 `[perf-38966.map]` 变成真正的 Java 方法名，直接定位到代码行。

**选型规则：** C/C++/Go/Rust → perf + FlameGraph；Java → async-profiler（自动解析 JIT 符号）。

## 番外：D 状态进程排查

进程卡在 D 状态（Uninterruptible Sleep），kill -9 都杀不掉。D 状态意味着进程在内核态等待某个不可中断的操作完成（通常是 I/O），信号送不进去。

排查五步：

```bash
# 1. 找到 D 状态进程
ps aux | awk '$8 ~ /D/ {print $0}'

# 2. 看它在等什么内核函数（wchan）
ps -eo pid,stat,wchan:32,comm | grep ' D'
# 输出类似：12345 D  rpc_wait_bit_killable      nfsd
# 说明卡在 NFS 的 RPC 等待上

# 3. 查看内核调用栈——最关键的一步
cat /proc/12345/stack
# 输出：
# [<0>] rpc_wait_bit_killable+0x2c/0x40 [sunrpc]
# [<0>] __rpc_execute+0x15d/0x1b0 [sunrpc]
# [<0>] rpc_run_task+0x64/0x80 [sunrpc]
# [<0>] nfs4_call_sync_sequence+0x4f/0x70 [nfsv4]

# 4. 如果是 I/O 问题，看块设备队列
cat /sys/block/sda/stat
# 字段：reads completed, reads merged, sectors read, ms reading,
#       writes completed, writes merged, sectors written, ms writing,
#       I/Os in progress, ms doing I/O, weighted ms doing I/O

# 5. iostat 看磁盘延迟
iostat -xz 1 5
# 重点看 await（平均等待时间）和 %util（利用率）
```

内核栈速查（看到这些函数就知道卡在哪）：

- `rpc_wait_bit_killable` → 卡在 NFS 的 RPC 等待
- `jbd2_journal_commit_transaction` → ext4 日志刷盘卡住
- `blk_mq_get_tag` → 块设备队列满了

## 番外二：整机所有核 CPI 突增——split lock / 总线锁

症状签名：整机所有业务、所有核的 CPI 从 <1 突增至 3–4（指令数不变 → CPU 利用率同步涨三四倍）；TMA 表现为 Frontend 取指反常（L1I miss 极高、icache 命中大量来自 remote CCD），而 L3/内存访问**反而下降**——与"内存带宽上涨导致 CPI 升"的常见模式相反，见到即应怀疑总线锁。

成因一句话：**跨缓存行的原子操作**（split lock，如对未对齐地址 xchg/CAS）触发总线锁（bus lock），AMD 上会拖慢整机所有核（Intel 微架构已把影响限制在单物理核）。

快速检测：

```bash
# AMD：数 bus lock 事件（须在产生 bus lock 的环境里采——宿主机与 VM 的 PMU 上下文分离，宿主机抓不到 VM 内的）
perf stat -e ls_locks.bus_lock
perf record -e ls_locks.bus_lock && perf script -F pid   # 定位到引发线程

# Intel：原始事件 + 内核日志（VM 内 split lock 宿主机可见 #AC 日志；AMD 5.10 内核无此检测）
perf stat -e r102c
dmesg | grep 'split_lock trap'   # "#AC: ... took a split_lock trap at address: ..."

# 线程 100% 且热点在 __lll_lock_wait_private→futex 时，抓 futex 等待地址看是否跨行/非法
bpftrace -e 'tracepoint:syscalls:sys_enter_futex /pid==12345/ {@cnt[tid,args->uaddr,args->op,args->val,args->uaddr2,ustack]=count();} interval:s:1 {print(@cnt); clear(@cnt);}'
```

锁地址未对齐跨 64B 行或为非法值（案例中 uaddr=0xffffffff）即实锤。定位元凶进程可用 `kill -SIGSTOP` 逐个停进程二分，机器恢复即锁定。

平台差异（ratelimit 参数）、jemalloc/glibc 混用根因案例、原子变量对齐预防规范 6 条：`references/split-lock.md`。编码预防一句话：原子变量 `alignas` 到自然边界（64 位→64B、`__int128`→16B），禁止 packed 结构体里放原子类型。

<!-- 来源: https://mp.weixin.qq.com/s/4DtVUCPSz7UWQ-icIV830g -->

## 火焰图该用还是不该用

火焰图是手术刀，不是瑞士军刀。

**该用：**

- CPU 飙高——top 看到进程吃满 CPU，但不知道哪个函数
- 压测优化——压到一定 QPS 后 RT 飙高，需要找瓶颈
- 版本对比——新版本比旧版本慢，用火焰图 diff 找差异
- 启动慢——服务启动几十秒，不知道卡在哪

**不该用：**

- 日常开发调试——IDE debugger 就够了
- 业务逻辑 Bug——这是 code review 的事
- I/O 瓶颈——CPU 火焰图看不出来，需要 Off-CPU 火焰图或 iostat

### Off-CPU 火焰图（I/O 瓶颈专用）

CPU 火焰图只能看到"在 CPU 上干什么"，看不到"等 I/O 时等了多久"。进程卡在 I/O 等待时 CPU 使用率反而低，CPU 火焰图会漏掉。

```bash
# 用 perf 记录调度事件（Off-CPU 分析）
perf record -e sched:sched_switch -e sched:sched_stat_sleep -p <pid> -- sleep 30
# 或用 bcc 工具的 offcputime-bpfcc
offcputime-bpfcc -p <pid> 30
```

Off-CPU 火焰图越宽 = 阻塞时间越长，直接定位 I/O 等待点。

## 常见坑

### 1. perf record 没符号表

火焰图里全是 `[unknown]` → 缺符号表。C/C++ 编译时加 `-g`，或装 debuginfo 包：

```bash
# CentOS/RHEL
debuginfo-install glibc
# Ubuntu
apt install libc6-dbg
```

### 2. strace 导致服务变慢

strace 基于 ptrace，每个系统调用都会让进程停两次，高并发服务别长时间挂着。用 `-c` 做统计，或切换到 perf trace（strace 的低开销替代品）：

```bash
perf trace -p $(pidof your_app) -s -- sleep 10
```

### 3. ftrace 忘记关

开了追踪忘记关，内核一直记录，ring buffer 满了会丢数据。用完关掉：

```bash
echo nop > /sys/kernel/debug/tracing/current_tracer
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace   # 清空 buffer
```

用 trace-cmd 没这个问题，它会自己管理生命周期。

### 4. 容器里用不了 perf

容器默认没权限访问 perf_event。两种解法：

```bash
# 方式一：在宿主机上直接 perf，通过 -p 指定容器里进程的宿主机 PID
docker inspect --format '{{.State.Pid}}' <container_id>
perf top -p <host_pid>

# 方式二：启动容器时加权限
docker run --privileged --pid=host ...
# 或者只加需要的 cap
docker run --cap-add SYS_ADMIN --cap-add SYS_PTRACE ...
```

## 工具安装备忘

```bash
# perf（跟内核版本绑定，装对应版本）
apt install linux-tools-$(uname -r)    # Debian/Ubuntu
yum install perf                        # CentOS/RHEL

# trace-cmd
apt install trace-cmd
yum install trace-cmd

# pmu-tools（toplev，TMA L2/L3 下钻用，见 references/tma-metrics.md）
git clone https://github.com/andikleen/pmu-tools ~/pmu-tools

# strace
apt install strace
yum install strace

# FlameGraph
git clone https://github.com/brendangregg/FlameGraph.git /opt/FlameGraph

# 一键生成火焰图的别名，加到 .bashrc 里
alias flamegraph='perf script | /opt/FlameGraph/stackcollapse-perf.pl | /opt/FlameGraph/flamegraph.pl > /tmp/flame_$(date +%Y%m%d_%H%M%S).svg && echo "saved to /tmp/flame_*.svg"'
```

### 权限前置：perf_event_paranoid

`perf_event_paranoid` 默认值 4，普通用户无法采样。**生产环境 perf 没数据时先查这个**：

```bash
cat /proc/sys/kernel/perf_event_paranoid   # 查看当前值
echo 1 | sudo tee /proc/sys/kernel/perf_event_paranoid   # 临时放开（重启失效）
# 永久：echo 'kernel.perf_event_paranoid=1' >> /etc/sysctl.conf && sysctl -p
```

值含义：4=仅 root；2=不允许内核分析；1=允许用户态采样（推荐生产值）；0=允许内核态采样；-1=无限制。

<!-- 来源: external/perf-book/chapters/7-Overview-Of-Performance-Analysis-Tools/7-4 Linux perf.md -->

配套权限：内核符号显示为地址串时，再放开 `kptr_restrict`：

```bash
echo 0 | sudo tee /proc/sys/kernel/kptr_restrict   # 允许非特权用户解析内核模块符号
```

perf.data 离线分析可用 KDAB Hotspot（类 VTune GUI）或 Netflix Flamescope（时间热图，圈选时段出该时段火焰图，可发现分阶段异常）。

## 验证：修复后必须复测

修复不等于结束，用数据确认问题解决才算关闭。

复测需在与事故时点相当的 QPS/负载下进行（对比监控面板事故时段流量）——低谷期流量自然回落时 CPU 也低，会造成修复生效的假象；流量不足先压测复现事故负载再测。

```bash
# 1. CPU 回落至事故前基线
pidstat -p <pid> 1 10

# 2. IPC 恢复正常（≥1.0）
perf stat -p <pid> -- sleep 10
```

3. 业务侧指标恢复（如 P99 响应时间回到事故前水平），持续观察 10 分钟无复发。

三项全部达标才能宣布解决；任一未达标回到 SOP 步骤 1 重新排查。

### 瓶颈转移：优化后必须重新出图

解决一个瓶颈后，原来不起眼的函数会暴露为新热点——不是它们变慢了，而是"巨型怪物"消失了。**每次优化后必须重新采样出图**，确认新热点是否可接受，避免盲目优化。

示例：bubble_sort 从 94.95% 消失后，matrix_multiply 从 3.54% 暴露为 71.2% 的新热点——这就是瓶颈转移。

## 定位热点之后：TMA 判读 + 修复 playbook

SOP 步骤 3 说"用户态热点直接改代码"——改成什么？先回答"为什么慢"再动手：

1. **判瓶颈类型**：`perf stat` 看 TMA 四桶 + branch-misses%（速查表见上文"perf stat --topdown"小节，完整方法 `references/tma-metrics.md`）。
2. **按类型查修复手册** `references/optimization-playbook.md`：
   - Memory Bound → 数据结构（数组优先/SoA/字段重排）→ 循环交换/分块 → 软件预取 → 大页（典型收益：循环交换 10x、预取 +87%、大页 +20%）
   - Core Bound → 依赖链交织（2x）→ 内联（+47%）→ 向量化（10x）
   - Bad Speculation → cmov / 条件存储 / 查表 / 虚调用对象分组（3–5x，误预测可降到 0.1% 以下）
   - Frontend Bound → likely 提示 / hot-cold 拆分 / PGO（最高 +30%）/ LTO/BOLT

   每类手法的**反例→正例完整代码对**（含实测收益数字，perf-book + perf-ninja 27 个实验）：
   - Memory Bound → `references/code-examples-memory.md`
   - Core Bound + Bad Speculation → `references/code-examples-compute.md`
   - Frontend Bound + 多线程/锁 + 低延迟 + 方法论 → `references/code-examples-frontend-thread.md`

3. **多线程专项**（与本技能锁竞争场景直连）：CPU 高但吞吐低先想 Spin Time；**多线程各写各的变量但同处一条 cache line（伪共享）**——现象是 IPC 极低（可达 0.05），检测 `perf c2c record -p <pid> -- sleep 10 && perf c2c report --stdio` 看 HITM/共享行，修复 `alignas(64)`（实测 >80% 收益）；`grep TLB /proc/interrupts` 某核异常高 = TLB shootdown。
4. 修完回到"验证"章节复测，循环到瓶颈可接受。

C/C++ 编译选项底线三件套：`-O3 -march=native -flto`；确认向量化是否发生：Clang `-Rpass=vectorizer` / GCC `-fopt-info`。

<!-- 来源: external/perf-book/ + external/perf-ninja/（详见 references/ 内分节标注） -->

## 实战案例参照（锁竞争导致 CPU 90%+）

完整时间线，可作为排查节奏参照：

| 时间  | 操作            | 工具                     | 发现                 |
| ----- | --------------- | ------------------------ | -------------------- |
| 15:02 | 收到 CPU 告警   | 监控系统                 | CPU 90%+             |
| 15:03 | 登机器看进程    | top                      | 业务进程 CPU 高      |
| 15:05 | 看系统调用分布  | strace -c                | futex 占 78%         |
| 15:08 | 看 futex 详情   | strace -e futex          | 锁等待超时           |
| 15:10 | 看 CPU 热点函数 | perf top                 | SpinLock::lock() 35% |
| 15:13 | 生成火焰图      | perf record + FlameGraph | 连接池自旋锁竞争     |
| 15:16 | 确认 IPC 低     | perf stat                | IPC 0.35，锁等待     |
| 15:18 | 看内核调用链    | ftrace function_graph    | futex_wait 14ms      |
| 15:20 | 定位根因        | 综合分析                 | 连接池自旋锁改互斥锁 |

根因：连接池用自旋锁（SpinLock），低并发没问题，并发一上来自旋空转消耗大量 CPU。火焰图最宽平顶：`SpinLock::lock()` → `ConnectionPool::getConn()` → `handleRequest()`——每个请求都从连接池拿连接，连接池用自旋锁保护，高并发疯狂自旋。

修复：自旋锁改互斥锁（mutex）+ 适当扩大连接池大小 → CPU 降到 30% 以下。

## Brendan Gregg 工具体系（知识框架）

火焰图只是 Brendan Gregg 工具体系的一层。完整体系分三层，**绝大多数开发者掌握前两层就够**：

| 层级   | 工具                | 定位                   | 使用者          |
| ------ | ------------------- | ---------------------- | --------------- |
| 基础层 | ftrace、perf        | 内核内置，日常性能采样 | 所有开发者      |
| 可视层 | FlameGraph、HeatMap | 把数据画成图           | 所有开发者      |
| 进阶层 | bcc、bpftrace       | 内核级动态追踪         | SRE、内核工程师 |

日常 CPU 排查用 perf + 火焰图（基础层 + 可视层）就够了。bcc/bpftrace 留给需要内核级动态追踪的疑难杂症。
