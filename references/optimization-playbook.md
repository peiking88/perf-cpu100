# 热点修复 playbook（按 TMA 瓶颈分类）

> 先用 references/tma-metrics.md 判明瓶颈类型，再按本手册对应章节找修复手法。
> 每条手法附实测收益数字（来源 perf-book / perf-ninja 课程）。
> **反例→正例完整代码对**：§2 看 code-examples-memory.md；§3/§4 看 code-examples-compute.md；§5/§6/§7 看 code-examples-frontend-thread.md。

## 总纲

- 优化顺序：算法/数据结构等大头 → 并行化 → 消除冗余（缓存/查表/预计算/不变量外提/传引用）→ 批处理 → 重排序 → 机器级优化。任何优化须在目标平台实测。
- 编译选项底线三件套：`-O3 -march=native -flto`（跨翻译单元内联/优化）。
- 心法：预测现代平台上的性能几乎不可能，Always measure；懂得在收益递减处停止。

## §1 通用：动态内存分配

- 栈分配仅移动栈指针；malloc 可能触发 minor page fault（首次访页 <1μs–数μs）与分配锁。
- 首先用 heaptrack 定位临时分配（分配后立即释放），消除之。
- 批量分配（arena/内存池）代替多次小分配；每线程独立 arena 消除分配锁竞争；jemalloc/tcmalloc 可 drop-in 替换 glibc malloc。

## §2 Memory Bound（Backend → Memory）

**数据结构（最优先）**

- 数组优先于链表/指针容器：链表遍历跟随指针、元素散落，同为 O(N) 实际远慢；扁平容器 `boost::flat_map`，无序时 `unordered_map` 快于 `map`。
- 行序遍历：`matrix[col][row]` → `matrix[row][col]`，让内层循环按内存布局顺序访问。
- AoS → SoA：循环只访问部分字段时 SoA 全顺序化且是向量化前提；若每次访问全部字段则 AoS 反而带宽利用率高。
- 字段按大小降序重排消 padding：`{bool,int,short}` 12B → `{int,short,bool}` 8B；位域打包：3 个 uint8 → `a:4,b:2,c:2`，存储缩 3 倍。
- hot/cold 拆分：同阶段访问的字段聚到同一 cache line，大结构按访问模式拆两个 struct。
- 大数组二分查找可改 Eytzinger 布局（BFS 式隐式树）。

**循环变换**

- Interchange 内外交换：让最内层连续访问。实测：矩阵幂 1261ms→126ms（**10x**）；高斯模糊需先把标量累加器升维成数组才能重排，290ms→71.4ms（4x，Memory_Bound 49.7%→5.4%）。
- Tiling 分块：多维循环切适配 L1/L2 的块（如 16×16），数据在被复用前不逐出；块大小按机器实测。实测矩阵转置 16.8ms→5.72ms（3x）。
- Fusion 合并（省循环开销+同 cache line 只加载一次）/ Fission 拆分（提高时间局部性、降寄存器压力）。

**预取**（随机访存、硬件预取失效时）

- `__builtin_prefetch(&arr[next_idx]);` 提前若干迭代预取。实测哈希表随机查找每迭代 13.7ms→1.81ms（+87%）。
- 三原则：足够早（满足预取窗口）、尽量晚（别把在用数据挤出 cache）、别预取已在 cache 的数据。step 从 cache line 宽度起步（int 数组 = 16）。
- 验证：除耗时外确认 L3 miss 数确实下降。

**TLB/大页**（随机访存大数据集 dTLB miss 高）

- 2MB 大页使单 TLB 项覆盖 ×512。实测吞吐 +20%（有限元矩阵向量积）；SPEC2006 最高单项 22%/27%，上限约 30%，期望 2x 不现实。
- 启用：THP `echo always > /sys/kernel/mm/transparent_hugepage/enabled`（全局，实验后记得关闭）；按进程 `madvise(ptr, size, MADV_HUGEPAGE)`；显式大页 `echo 128 > /proc/sys/vm/nr_hugepages` + mmap `MAP_HUGETLB`（不可换出、延迟最稳定，HFT 首选）。
- THP 代价：khugepaged 后台合并带来非确定性延迟。

**对齐**

- SIMD 负载跨 cache line 边界产生 split load/store（TMA 下钻：Memory Bound → L1_Bound → Split Loads）。修复：数据起始 `alignas(64)` + 行宽补齐到 cache line 倍数。实测 512 列比 511/513 快 15–20%。
- 对齐标准：AVX2 32B、SSE/Neon 16B、AVX-512 64B；Apple L2 cache line 128B。

**带宽打满时**

- 判别：DRAM BW 接近平台峰值。此时代码优化（含向量化）基本无效——CPU 无数据可用。
- 修复方向：降内存强度（压缩数据、实时重算代替缓存、量化 fp32→fp16/int8 降 2x/4x 流量）；最终手段是加内存通道。多线程下 3–4 线程即可能饱和（实测 CloverLeaf 3 线程饱和 35GB/s）。

## §3 Core Bound（Backend → Core）

**依赖链断裂**

- 规则："在关键路径上的指令看延迟，不在关键路径的看吞吐"。
- 多累加变量/多独立链交织：XorShift RNG 双实例交织，19ms→10ms（近 2x，IPC 4.0→7.1）。长链（万条指令级）必须逐语句**交织**两条链——只展开不交织仅 +5%，寄存器不够会 spill。
- 链表追逐是天然超长依赖链：分批重叠（一次摘 4 个节点存数组再逐个比对），85.5ms→34.7ms（2.5x）。

**内联**

- 消除 CALL/RET + prologue/epilogue，更重要的是扩大编译器分析范围。profile 中函数 prologue/epilogue 占比高（如 ~50%）是强信号。
- `qsort` → `std::sort`+lambda（编译库函数不可内联）：760µs→518µs（+47%）。
- 强制内联：`[[gnu::always_inline]]`；冷函数反用 `noinline`。

**向量化**

- 确认向量化发生：Clang `-Rpass=vectorizer` / `-Rpass-missed=loop-vectorize`；GCC `-fopt-info`。看汇编：向量指令操作打包数据（助记符含 P）+ XMM/YMM/ZMM 寄存器；看到 XMM 不等于向量化（VMULSS 是标量）。
- 触发条件：`-O2` 起自动尝试；浮点加法不满足结合律阻碍向量化 → `-ffast-math`（风险：NaN/带符号零；可 `#pragma clang fp reassociate(on)` 局部开）；指针别名 → `__restrict__`；数据布局暴露 SIMD（转置成一行=一个向量宽）：+60%。
- 进位/前次结果依赖阻断向量化（校验和 `acc += acc < value`）：宽累加器+分块延迟进位，33.4µs→3.61µs（10x）。
- Retiring >80% 但性能仍差 = 大量简单标量指令待向量化。
- AVX-512 有降频/启动开销，向量段需足够热才划算。
- 手写 intrinsics（`<immintrin.h>`）仅在编译器失败处用；跨平台考虑 Highway/std::experimental::simd。

## §4 Bad Speculation（分支误预测 >10%）

- **条件传送 cmov**：两侧计算代价小（≤20 条指令/周期）且分支难预测时用。`__builtin_unpredictable(cond)`（Clang 17+）诱导生成 cmov（ARM 为 csel）。
- **条件存储无分支化**：`output[count] = item; count += (lower <= v && v <= upper);` 无条件写+谓词累加，286µs→65.6µs（4x+，误预测 23.54%→0.03%）。
- **查表**：值域小的多级 if-else → 数组寻址 `bucket[v]` + 越界钳位（`std::min`，-O3 下无分支），5475µs→995µs（5.5x，误预测 11.93%→0.01%）。大范围用分段表/interval map。
- **算术替换**：`if (v < 50) return v/10; return -1;` 类可用乘法+移位等价式；编译器通常不会自动发现，需手工。
- **多比较合并单分支**：SIMD 一次比较 32 字符转掩码，非零才处理（tzcnt 找位），分支指令减 5–6 倍、提速 >4x。
- **虚调用误预测**：随机混排的多态对象数组 → 按派生类型分组连续存放，589µs→186µs（3x，误预测 21.5%→0.12%）。
- 间接手段：循环展开/向量化/位运算减少动态分支数；PGO/BOLT 拉直热路径。

## §5 Frontend Bound（>20% 时投入）

- `[[likely]]/[[unlikely]]`（C++20）或 `__builtin_expect` 让热路径 fall-through 连续，兼顾 I-cache/μop-cache 密度。
- 热循环跨 cache line 强制取指两行：`[[clang::code_align(64)]]` 或 LLVM 默认 16B 对齐。
- 函数拆分 hot/cold split：大块冷代码外移 `.text.cold`，热路径只剩一条 CALL。
- 函数重排序：`-ffunction-sections` + LLD `--symbol-ordering-file`；HFSort 自动生成（大云应用实测 ~2%）。
- PGO：插桩三步（`-fprofile-instr-generate` 跑负载 `-fprofile-instr-use` 重编译；GCC 为 `-fprofile-generate/-fprofile-use`）。前端瓶颈严重负载最高 +30%；插桩运行有 5–10x 减速，训练负载必须有代表性。AutoFDO 用 perf 采样替代插桩，可生产采集，还解锁 branch-to-cmov 转换。
- LTO 之上再叠加 BOLT（后链接优化器，基于 perf 采样重排）：再 +5–10%；`-hugify` 只把热代码放 2MB 大页。
- ITLB 大页（代码段 >1MB 的大代码库：数据库/V8/JVM/node.js）：代码段重链接对齐 2MB（`-Wl,-zmax-page-size=2097152`）或运行时 `LD_PRELOAD=liblppreload.so`；ITLB miss 最多 -50%，部分应用最高 +10%。
- 判据提醒：二进制大 ≠ 前端瓶颈，关键是热代码量与页密度（Clang 60MB 代码 ITLB 开销 ~7% 周期；Blender .text 133MB 但热代码 <1%）。

## §6 多线程与锁（CPU 飙高姐妹场景）

**量化指标**

- Spin Time = 线程在同步 API 上忙等自旋的 CPU 时间——"锁竞争导致 CPU 飙高"的直接量化。多线程 CPU 高但吞吐低 → 先查 Spin/Sync Wait Time（VTune Threading Analysis；eBPF+GAPP 可无插桩追踪 futex 阻塞栈）。
- Amdahl：75% 可并行的程序加速比极限 4x；USL：超过临界点加核反降速（retrograde），根因=竞争（同步开销）+一致性（跨核失效广播）。
- 频率 throttling 是扩展性损失大头：多核并发即降频（16 线程 P 核 3.2GHz）；关 Turbo 后扩展率几乎翻倍。SMT 兄弟线程扩展仅 1.1–1.3x。

**伪共享 false sharing**（多线程 CPU 高但 IPC 极低 → 跑 perf c2c）

- 现象：多线程各写各的变量，但变量同处一条 cache line（64B），MESI 一致性强制核间来回传行——`perf c2c record/report` 显示大量 Shared Cache Lines、HITM 集中；IPC 可低至 0.05。
- 修复：`struct alignas(64) Accumulator { std::atomic<uint32_t> value; };`（Apple M1/M2+ 注意 L2 行 128B）。实测 209ms→36.3ms（>80%）。
- 真共享先排数据竞争（ThreadSanitizer），再改 TLS：`thread_local` 各线程本地累加最后合并，优于 atomic 串行化。

**TLB shootdown**（多线程低延迟最易忽视）

- munmap/madvise 等触发内核 IPI 使所有相关核失效 TLB，随线程数放大。检测：`watch -n5 -d 'grep TLB /proc/interrupts'`（某核比其他核高一个数量级即中招；常见元凶是自动 NUMA 均衡：`sysctl -w numa_balancing=0`）。

**任务调度**

- 别绑核（pinning）：工作不均时阻断 work stealing，E 核长尾拖累整体；别静态均分：动态分区 + 适度 chunk（实测 128 块是甜点，1024 块管理开销反升）。

## §7 低延迟/系统级

- Minor page fault：`top` 加 vMn 列（-H 线程级）；HFT 要求 0，一般系统 100–1000 faults/s 即应调查。预防：启动时逐页预热 + `mlockall(MCL_CURRENT|MCL_FUTURE)` + `mallopt(M_TRIM_THRESHOLD,-1)` 等。
- Cache warming：延迟敏感路径周期性"演练"保温（不执行副作用）。
- 无解释的延迟抖动查 AVX-512 降频：`-mprefer-vector-width=256` 禁之。
- 系统级干扰（一次系统中断可挂起全系统数十至上百毫秒）：低延迟场景禁 C-states、按 Red Hat 低延迟调优指南消除干扰中断；方向性手段还包括 CPU 亲和/核隔离（具体命令参考发行版低延迟指南）。
- LLC 敏感性：带宽/容量敏感型应用（如 omnetpp 32MB LLC 比 0MB 快 2.5x）值得买大 LLC；不敏感应用买小 LLC 省钱。

## §8 perf-ninja 实测收益速查表

| 手法                       | 瓶颈类   | 实测收益                               |
| -------------------------- | -------- | -------------------------------------- |
| 循环交换（连续访问）       | Memory   | 10x                                    |
| 依赖链交织                 | Core     | 2x（IPC 4.0→7.1）                      |
| 分批重叠（链表）           | Core     | 2.5x                                   |
| 校验和分块进位向量化       | Core     | 10x                                    |
| 数据布局转置+自动向量化    | Core     | +60%                                   |
| qsort→std::sort            | Core     | +47%                                   |
| AVX2 字符串搜索 intrinsics | Core     | 4x                                     |
| 条件存储无分支化           | BadSpec  | 4x+（误预测 23.5%→0.03%）              |
| 查表替换 if 链             | BadSpec  | 5.5x（11.9%→0.01%）                    |
| 虚调用对象分组             | BadSpec  | 3x（21.5%→0.12%）                      |
| alignas(64) 消伪共享       | 多线程   | >80%                                   |
| 软件预取（随机哈希查）     | Memory   | +87%                                   |
| 2MB 大页                   | Memory   | +20%                                   |
| 循环分块 16×16             | Memory   | 3x                                     |
| 对齐+行宽补齐              | Memory   | 15–20%                                 |
| PGO（真实负载）            | Frontend | 最高 +15%（书：前端瓶颈负载最高 +30%） |
| LTO / BOLT                 | Frontend | / 再 +5–10%                            |

<!-- 来源: external/perf-book/chapters/8-Optimizing-Memory-Accesses/ -->
<!-- 来源: external/perf-book/chapters/9-Optimizing-Computations/ -->
<!-- 来源: external/perf-book/chapters/10-Optimizing-Branch-Prediction/ -->
<!-- 来源: external/perf-book/chapters/11-Machine-Code-Layout-Optimizations/ -->
<!-- 来源: external/perf-book/chapters/12-Other-Tuning-Areas/ -->
<!-- 来源: external/perf-book/chapters/13-Optimizing-Multithreaded-Applications/ -->
<!-- 来源: external/perf-ninja/labs/ -->
