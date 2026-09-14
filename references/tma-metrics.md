# TMA 判读与性能度量参考（perf-cpu100 深化篇）

> 火焰图/热点函数只回答"哪里最热"，TMA（Top-Down Microarchitecture Analysis）回答"为什么慢"。
> 适用阶段：SOP 步骤 2/3 已锁定热点函数之后，决定修复方向之前。

## 一、TMA 四分类

CPU 每个周期都要为指令分配 pipeline slot，分配失败的原因只有四种，构成 L1 四桶（合计 100%）：

| 桶                  | 含义                      | 判读阈值                                                                | 修复方向                                  |
| ------------------- | ------------------------- | ----------------------------------------------------------------------- | ----------------------------------------- |
| **Retiring**        | 正常退休的有用工作        | 越高越好；>80% 但性能仍差 = 大量标量指令待向量化                        | 向量化（见 playbook §3）                  |
| **Frontend Bound**  | 取指/译码跟不上           | <10% 正常，>20% 值得投入                                                | 代码布局/PGO/LTO/ITLB 大页（playbook §5） |
| **Bad Speculation** | 分支误预测浪费的 slot     | 通用应用 5–10% 正常，>10% 重点排查（perf stat branch-misses >10% 同判） | 无分支化/cmov/查表（playbook §4）         |
| **Backend Bound**   | 后端（内存/执行单元）过载 | 剩余大头，需下钻 L2：Memory Bound vs Core Bound                         | playbook §2 / §3                          |

L2 细分：Backend → Memory Bound（L1/L2/L3/DRAM/Store Bound）+ Core Bound（Divider/Ports Utilization）；Frontend → Fetch Latency/Fetch Bandwidth；Bad Spec → Branch Mispredict/Machine Clears；Retiring → Base/Microcode Sequencer。注意 L3 起计数域变为 Stalls/Clocks，与 L1 的 Slots 不直接可比。

**陷阱**：Retiring 高 ≠ 性能好——自旋等锁的紧循环 Retiring 很高却无有用工作；TMA 只识别微架构瓶颈，不与程序性能正相关。代码存在高级缺陷（烂算法）时不要用 TMA，先修算法。

## 二、采集命令（按平台）

```bash
# Intel（Sandy Bridge+）：新版 perf 默认输出 TopdownL1 四桶
perf stat -- ./your_app
#   输出 tma_backend_bound / tma_bad_speculation / tma_frontend_bound / tma_retiring

# L2/L3 下钻用 pmu-tools 的 toplev（github.com/andikleen/pmu-tools）
~/pmu-tools/toplev.py --core S0-C0 -l2 -v --no-desc taskset -c 0 ./your_app
# 已知瓶颈后限定节点，减少事件多路复用：
~/pmu-tools/toplev.py --core S0-C0 -l2 --nodes L1_Bound,L2_Bound,L3_Bound,DRAM_Bound,Store_Bound -- ./your_app
# toplev 超阈值的指标自动标 "<=="；--show-sample 直接给出定位用的 perf record 命令行

# AMD Zen4+（内核 6.2+）
perf stat -M PipelineL1,PipelineL2 -- ./your_app
#   输出 backend_bound_cpu / backend_bound_memory / frontend_bound_latency / retiring 等
#   括号内百分比 = 该指标实际监控时长占比（多路复用），负载行为不均时分次采集

# Arm Neoverse N1/V1+（topdown-tool，developer.arm.com；Apple 处理器不支持）
topdown-tool --all-cpus -m Topdown_L1 -- ./your_app
topdown-tool --all-cpus -n BackendBound -- ./your_app   # V1 无 L2，按类别收集下钻指标
```

## 三、TMA 三步工作流（迭代）

1. **识别瓶颈**：先收 L1 四桶 → 逐层下钻（负载平稳时一次跑完，不稳时分多轮各下钻一层；可同时存在多类瓶颈）。
2. **定位代码**：用与瓶颈类型对应的精确事件采样。事件名查 Intel TMA_Metrics.xlsx（github.com/intel/perfmon）的 Locate-with 列，或 `toplev --show-sample` 直接给命令。
3. **修复 → 回到第 1 步**。瓶颈必然转移（见 SKILL.md"瓶颈转移"章节）。

定位示例（DRAM Bound 下钻到指令级）：

```bash
# 验证 DRAM/L3 瓶颈：L3 miss 停顿占总 cycles 比例
perf stat -e cycles,cycle_activity.stalls_l3_miss -- ./your_app
#   例：32.2G cycles 中 19.7G 停顿（~60%）→ 很严重

# 用精确事件把 L3 miss 定位到源码行（:ppp 后缀消除 skid）
perf record -e cpu/event=0xd1,umask=0x20,name=MEM_LOAD_RETIRED.L3_MISS/ppp -- ./your_app
perf report -n --stdio
perf annotate --stdio -M intel foo   # 热点逐条指令带百分比
```

案例：随机下标访问数组 → 在生成下标后插 `__builtin_prefetch(a+idx,0,1)` → 8.5s→6.5s，L3 miss 停顿从 190 亿降到 20 亿（约 10 倍）。

## 四、常用比率指标（判读 perf stat 输出）

原始事件计数没有分母无意义（10 亿次 L3 miss：若只有 20 亿 load 是灾难，1 万亿 load 则可忽略），一律用比率：

| 指标                             | 公式                                           | 判读                                                                                                                                                       |
| -------------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IPC                              | instructions / cycles                          | 内存密集型通常 0–1（属正常，勿惊慌）；计算密集型 4–6。IPC 上限 = 机器发射宽度（Skylake/Zen3=4，Golden Cove/Zen4=6，Apple M1/M2=8），IPC 算出超过宽度应起疑 |
| CPI                              | 1/IPC                                          | 同上反向                                                                                                                                                   |
| L1/L2/L3 MPKI                    | 1000 × MEM_LOAD_RETIRED.Lx_MISS / instructions | 每千条指令 miss 数，越低越好；逐层分解：all_loads = fb_hit + l1_hit + l1_miss                                                                              |
| Branch Mispredict Ratio          | branch-misses / branches                       | >10% 严重；误预测惩罚典型 10–25 周期                                                                                                                       |
| Code/Load/Store STLB MPKI        | 1000 × ITLB/DTLB miss walk / instructions      | TLB 压力，大代码库/大内存随机访问时高                                                                                                                      |
| Load Miss Real Latency           | L1D miss 平均延迟（周期）                      | 高 + DRAM 带宽接近峰值 = bandwidth-bound；高 + 带宽远未饱和 = latency-bound（预取/乱序可解）                                                               |
| DRAM BW Use                      | 64 × CAS_COUNT(RD+WR) / 时间                   | 接近平台峰值 = 带宽打满，代码优化基本无效                                                                                                                  |
| ILP / MLP                        | 见 TMA_metrics.xlsx                            | 乱序并行度实测值                                                                                                                                           |
| IpCall / IpBranch / IpMispredict | instructions / 各类指令数                      | IpCall 小（如 41）= 函数太小太密，该 inline/LTO                                                                                                            |

## 五、延迟与容量参考数值

| 层级       | 典型延迟                                    |
| ---------- | ------------------------------------------- |
| L1 hit     | ~4 cycles（约 1 ns）                        |
| L2         | 10–25 cycles（5–10 ns）                     |
| L3/LLC     | ~40 cycles（约 20 ns）                      |
| 主存       | 200+ cycles（客户端 70–110 ns，服务器更高） |
| 分支误预测 | 10–25 cycles                                |
| cache line | 64B（Apple L2 为 128B 例外）                |

内存带宽：DDR4-3200 单通道 25.6 GB/s，DDR5-6400 单通道 51.2 GB/s；笔记本典型 2 通道，服务器 8–12 通道（整机可达 500 GB/s，单线程约 50 GB/s）。带宽每代升、延迟持平或变差。

TLB：x86 默认 4KB 页，20MB 数据需 5120 个页表项 vs 2MB 大页仅 10 项；Golden Cove L2 STLB 2048 项、4 个并行 page walker。TLB miss 至今仍是许多应用的性能瓶颈。

## 六、度量陷阱（判读指标前必读）

1. **CPU 利用率会说谎**：高利用率只说明"在忙"，不说明在做什么——可能满载 stall 在等内存，也可能线程在锁上自旋空转。有效利用率 = (CPU Time − Spin Time − Overhead Time) / 总时间；多线程程序 CPU 高但吞吐低，先想 Spin Time（VTune 可测；futex 高 → 锁竞争路径的量化版）。
2. **SMT/超线程**：逻辑核 100% ≠ 物理核饱和；两逻辑核竞争 L1/L2 导致驱逐，SMT 兄弟线程扩展典型仅 1.1–1.3x。混合架构（P/E 核）下 sibling 空闲与否也影响同一"利用率"的实际吞吐。
3. **频率缩放**：`cycles` 按实际频率计数（含 turbo），`ref-cycles` 按基频计数不受升降频影响。比较两段代码用时钟周期而非纳秒，避免频率波动干扰；多核并发会 throttling（关 Turbo 后 16 线程扩展率几乎翻倍：Blender 38%→69%）。降频使内存相对变快，可能掩盖内存瓶颈、人为抬高 IPC。
4. **事件多路复用**：事件数超过物理 PMC 数（Intel 8 个可编程/核，AMD Zen4 与 Arm Neoverse 6 个/核）时按时间轮转，结果按 `raw × time_running / time_enabled` 换算——是估算不是实测，有阶段切换的工作负载会有盲点。避免：事件降到 PMC 数内、多跑几次。
5. **采样盲区**：采样把时间段聚合成样本，检测不到短时异常（如网络包驱动型程序的忙等会被归到正常循环）；加大采样率（>1000/s）仍不够时改用 tracing。采样 skid：普通 PMI 中断的事件 IP 可能偏移数百条指令，精确事件加 `:pp`/`:ppp` 后缀（PEBS EventingIP）消除。
6. **开销量级**：perf 计数/采样通常 <2%，事件多路复用后 5–15%；strace 对重 syscall 程序可达 100x；代码插桩可致 2x 减速。生产常驻剖析（Parca 等）聚合开销须 <1%。
7. **统计判读**：汇总表只是平均值（IPC 0.2 可能是 0.1 与 0.3 两相位平均），看 min/max/p95/随时间曲线；标准差与均值同量级时均值无代表性，先降噪再谈加速比；"Performance measurements should be considered guilty until proven innocent"。
8. **测量噪声**：turbo 冷热跑差异、文件系统 cache 冷热、链接顺序/内存布局都能改变结果；微基准最大坑是编译器死代码消除（被测代码根本没跑）——用 DoNotOptimize/Blackhole 消费结果，并对 benchmark 本身做 profile 确认热点确实是目标代码（"Always Measure One Level Deeper"）。
9. **计时器选型**：`clock_gettime` 系统调用约 500 ns，适合 >1μs 事件；TSC（`__rdtsc`）读取约 5 ns，适合 ns 级低开销计时。

## 七、四个真实负载的判读示范（i7-1260P，toplev 采集）

| 指标                | Blender       | Stockfish                                      | Clang 自举                                       | CloverLeaf                 |
| ------------------- | ------------- | ---------------------------------------------- | ------------------------------------------------ | -------------------------- |
| IPC                 | 1.40          | 1.80                                           | 0.64                                             | 0.20                       |
| L1/L2/L3 MPKI       | 3.9/0.15/0.04 | 21.4/1.7/0.14                                  | 6.0/1.1/0.56                                     | 13.4/3.6/3.4               |
| 误预测率            | 0.02          | **0.08**                                       | 0.03                                             | 0.01                       |
| Load Miss 延迟(cyc) | 12.9          | 10.4                                           | 76.7                                             | **253.9**                  |
| DRAM BW (GB/s)      | 1.6           | 1.4                                            | 10.7                                             | **24.6**（近峰值）         |
| 结论                | FP 计算受限   | 整数+**分支误预测**受限（每 120 周期一次惩罚） | 大代码库：D-cache/TLB miss+函数太小（IpCall 41） | **内存带宽打满**，加核无用 |

CloverLeaf 判读要点：IPC 0.20 + L3MPKI 3.4 + Load Miss 延迟 254 周期 + DRAM 带宽逼近上限 → 所有核共享内存总线互相竞争，CPU 数据供给不足；P 核频率高，等内存时浪费更多时钟，IPC 反而更低。

## 八、生产环境补充

- **连续剖析（Continuous Profiling）**：系统级常驻低频采样（Parca 默认 19 样本/秒采全部进程栈），给采样加时间维度，可回溯任意时间点、对比任意两时刻调用栈（差分火焰图）；最佳实现基于 eBPF，无需改代码。局限同普通剖析：CPU 最高的函数未必在关键路径上。
- **性能回归检测**：固定阈值（如 2%）会漏小回归（日均 1.5% 累计 10 天 = 15% 全被滤掉）；Change Point Detection 无需阈值、抗噪但反馈慢。同时告警"意外的性能提升"——无害提交突然快 10% 可能暴露功能测试缺口。
- perf.data 可用 KDAB Hotspot（类 VTune GUI）或 Netflix Flamescope（时间热图 + 圈选时段出火焰图）离线分析。

<!-- 来源: external/perf-book/chapters/6-CPU-Features-For-Performance-Analysis/ -->
<!-- 来源: external/perf-book/chapters/4-Terminology-And-Metrics/ -->
<!-- 来源: external/perf-book/chapters/3-CPU-Microarchitecture/ -->
<!-- 来源: external/perf-book/chapters/2-Measuring-Performance/ -->
<!-- 来源: external/perf-book/chapters/7-Overview-Of-Performance-Analysis-Tools/ -->
