# Frontend Bound / 多线程 / 系统级 / 方法论 代码样例（反例 → 正例）

> 配合 references/optimization-playbook.md §5/§6/§7 使用。每条含实测收益（若有）。

## A. 机器码布局（Frontend Bound）

### [[likely]]/[[unlikely]] 与 __builtin_expect

**反例（慢）**：

```cpp
// hot path
if (cond)
  coldFunc();
// hot path again
```

**正例（快）**：

```cpp
// hot path
if (cond) [[unlikely]]
  coldFunc();
// hot path again
```

C++20 之前的写法与 switch 用法：

```cpp
#define LIKELY(EXPR)   __builtin_expect((bool)(EXPR), true)
#define UNLIKELY(EXPR) __builtin_expect((bool)(EXPR), false)
if (UNLIKELY(cond)) coldFunc();

for (;;) {
  switch (instruction) {
               case NOP: handleNOP(); break;
    [[likely]] case ADD: handleADD(); break;
               case RET: handleRET(); break;
  }
}
```

**为什么**：提示编译器反转条件、让热代码顺序执行（fallthrough）：热代码连续摆放消除 I-cache/μop-cache 碎片化；taken 分支比 not-taken 更贵（Skylake 每两周期只能执行 1 条 taken 分支，但每周期 2 条 not-taken）。`[[unlikely]]` 还会阻止冷函数被内联。
**来源**：perf-book/chapters/11-Machine-Code-Layout-Optimizations/11-3 Basic Block Placement.md

### [[clang::code_align()]] 循环对齐

**反例（慢）**（循环体 0x4046b0 起，横跨 0x80–0xBF 与 0xC0–0xFF 两条 cache line，取指读两行）：

```cpp
void benchmark_func(int* a) {
  for (int i = 0; i < 32; ++i)
    a[i] += 1;
}
```

**正例（快）**（一条 NOP 把整个循环塞进单条 64B cache line）：

```cpp
void benchmark_func(int* a) {
  [[clang::code_align(64)]]
  for (int i = 0; i < 32; ++i)
    a[i] += 1;
}
```

**为什么**：热循环跨 cache line 强制取指两行；code_align 属性细粒度对齐。替代方案：`-mllvm -align-all-blocks=5`（全 TU 生效，不推荐）、内联汇编 `asm(".align 64;")`；LLVM 默认按 16B 对齐循环。
**来源**：perf-book/chapters/11-Machine-Code-Layout-Optimizations/11-4 Basic Block Alignment.md

### hot-cold 拆分（Function Splitting 到 .text.cold）

**反例（慢）**：

```cpp
void foo(bool cond1, bool cond2) {
  // hot path
  if (cond1) {
    /* cold code (1) */
  }
  // hot path
  if (cond2) {
    /* cold code (2) */
  }
}
```

**正例（快）**：

```cpp
void foo(bool cond1, bool cond2) {
  // hot path
  if (cond1) {
    cold1();
  }
  // hot path
  if (cond2) {
    cold2();
  }
}
void cold1() __attribute__((noinline))
{ /* cold code (1) */ }
void cold2() __attribute__((noinline))
{ /* cold code (2) */ }
```

**为什么**：热路径只留一条 CALL，后续热指令大概率与前一指令同 cache line；`noinline` 防止编译器把冷代码内联回去（或对分支加 `[[unlikely]]`）；冷函数放 `.text.cold` 段——不被调用就不加载进内存。
**来源**：perf-book/chapters/11-Machine-Code-Layout-Optimizations/11-5 Function Splitting.md

### 函数重排序（链接器选项）

**命令**：

```bash
# 先编译 -ffunction-sections（每个函数独立 section），再用链接器指定顺序：
clang++ -ffunction-sections ...
# Gold:  --section-ordering-file=order.txt
# LLD:   --symbol-ordering-file order.txt
```

**为什么**：把热函数按调用顺序（foo, zoo, bar）相邻摆放，共享 cache line（书中例 4 行→3 行）。HFSort/HFSort+/CDSort 基于剖析数据自动生成顺序文件（Meta 实测大云应用 +2%）。
**来源**：perf-book/chapters/11-Machine-Code-Layout-Optimizations/11-6 Function Reordering.md

### PGO 全流程（前端瓶颈负载最高 +30%；lab Lua 解释器至少 +5%）

**正例（快）**（Clang 插桩三步）：

```bash
# step1: 编译并插桩
$ clang++ -O2 -fprofile-instr-generate main.cpp -o prog_instr
# step2: 运行插桩二进制（跑典型负载），生成 default.profraw
$ LLVM_PROFILE_FILE=main.profraw ./prog_instr <典型负载>
# step3: 合并并用 profile 重编译
$ llvm-profdata merge -output=main.profdata main.profraw
$ clang++ -O2 -fprofile-instr-use=main.profdata main.cpp -o prog

# GCC 对应:
$ g++ -O2 -fprofile-generate main.cpp -o prog
$ ./prog                     # 生成 *.gcda
$ g++ -O2 -fprofile-use main.cpp -o prog
```

lab 的 CMake 版（Lua 解释器）：

```bash
# 1) 插桩构建
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_C_FLAGS="-fprofile-instr-generate" \
      -DCMAKE_CXX_FLAGS="-fprofile-instr-generate" ..
cmake --build . && ./lab     # 跑 bench lua 脚本，生成 default.profraw
# 2) 合并（可 llvm-profdata show --all-functions --counts 检查训练数据）
llvm-profdata merge -output=lua.profdata default.profraw
# 3) 重编译
cmake -DCMAKE_C_FLAGS="-fprofile-instr-use=lua.profdata" \
      -DCMAKE_CXX_FLAGS="-fprofile-instr-use=lua.profdata" ..
```

**为什么**：profile 告诉编译器真实热路径，改进内联决策、代码布局（热块聚拢、冷块外移）与寄存器分配。注意：插桩运行有 5–10x 减速；训练负载必须代表真实使用场景（只对训练集类似负载提速，换负载可能变慢）。采样式替代：AutoFDO（perf 采样转 profile，可生产采集，还解锁 branch-to-cmov 转换）、BOLT（后链接优化，再 +5–10%，`-hugify` 只把热代码放 2MB 页）、Propeller。
**来源**：perf-book/chapters/11/11-7 PGO.md；perf-ninja/labs/misc/pgo/

### LTO 链接时优化（跨编译单元内联）

**命令**：

```bash
# perf-ninja lab（aobench 多翻译单元）：默认构建无跨 TU 优化
cmake -DCMAKE_BUILD_TYPE=Release ..
# 启用 LTO：链接期做跨单元过程间优化（IPO）
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_C_FLAGS="-flto" \
      -DCMAKE_CXX_FLAGS="-flto" \
      -DCMAKE_EXE_LINKER_FLAGS="-flto" ..
```

**为什么**：热函数调用跨越编译单元边界时常规编译无法内联；LTO 在链接期把调用开销与常量传播打通，替代手工合并源文件。
**来源**：perf-ninja/labs/misc/lto/

### ITLB 2MB 对齐（ITLB miss 最多 -50%，部分应用最高 +10%）

**命令**：

```bash
# 方法一：重链接对齐代码段到 2MB 边界（代价：二进制膨胀，Clang 111MB→114MB）
$ clang++ -Wl,-zcommon-page-size=2097152 -Wl,-zmax-page-size=2097152 ...
# 再用 libhugetlbfs 设置 ELF 头标志位，使代码段默认用 huge page 加载:
$ hugeedit --text /path/to/clang++     # 永久设 ELF 头位
$ hugectl --text /path/to/clang++ a.cpp  # 运行时覆盖

# 方法二：运行时重映射（免重编译，Intel iodlr 库，兼容显式/透明大页）
$ LD_PRELOAD=/usr/lib64/liblppreload.so clang++ a.cpp
```

**为什么**：代码段 2MB 对齐后可映射到 huge page，ITLB 条目覆盖范围扩大。受益应用：代码段 >1MB 的大代码库（数据库/V8/JVM/node.js；Clang 60MB 代码 ITLB 开销约 7% 周期）。BOLT `-hugify` 基于剖析只把热代码放 2MB 页，减少碎片。
**来源**：perf-book/chapters/11/11-8 Reducing ITLB misses.md

## B. 多线程与锁

### 伪共享 false sharing（lab 209ms→36.3ms 约 5.8x）

**反例（慢）**（各线程写自己的 atomic，但 4 字节的累加器在 64B 行上挤 16 个）：

```cpp
struct Accumulator {
    std::atomic<uint32_t> value = 0;
};
// sizeof(Accumulator) == 4：lock add 触发缓存行在核间弹跳
#pragma omp parallel num_threads(thread_count) default(none) shared(accumulators, data)
{
  auto &target = accumulators[omp_get_thread_num()];
  #pragma omp for
  for (int i = 0; i < data.size(); i++) {
    auto item = data[i];
    item += 1000;
    item ^= 0xADEDAE;
    item |= (item >> 24);
    target.value += item % 13;
  }
}
```

**正例（快）**：

```cpp
constexpr int CACHE_LINE_SIZE = 64;

struct alignas(CACHE_LINE_SIZE) Accumulator {
    std::atomic<uint32_t> value = 0;
};
// 其余代码不变：每个线程的累加器独占一条缓存行
```

书的极简对照（`{int sumA; int sumB;}` → alignas 分隔）：

```cpp
constexpr int CacheLineAlign = 64;
struct S {
  int sumA;
  alignas(CacheLineAlign) int sumB;
};
```

**为什么**：各线程写不同变量却共享同一 cache line，MESI 一致性强制核间来回传行（`perf c2c` 显示大量 HITM/87K+ 次锁定访问，IPC 可低至 0.05）；alignas(64) 让每个累加器独占一行（Apple M1/M2+ 的 L2 行是 128B）。TMA 定位：Memory Bound → L3 Bound → Contested Accesses。
**来源**：perf-ninja/labs/memory_bound/false_sharing_1/；perf-book/chapters/13/13-6

### thread_local TLS 累加（真共享的更优解）

**反例（慢）**（共享变量 sum：先有数据竞争，加锁/atomic 后变串行）：

```cpp
unsigned int sum; // shared between all threads
{ // thread A              |  { // thread B
  for (int i = 0; i < N; i++)  |    for (int i = 0; i < N; i++)
    sum += a[i];           |      sum += b[i];
}                         |  }
```

**正例（快）**：

```cpp
thread_local unsigned int sum;  // C++11
// 各线程改本地副本，主线程最后合并各副本结果
```

**为什么**：TLS 让每个线程在自己的缓存行（M/E 状态）上累加，无竞争、无原子序列化；比 `std::atomic` 方案更好——atomic 只解决正确性，访问仍被串行化。真共享先用 ThreadSanitizer/helgrind 排数据竞争。
**来源**：perf-book/chapters/13-Optimizing-Multithreaded-Applications/13-6 Cache Coherence Issues.md

### OpenMP 调度策略（i7-1260P 实测：864ms → 517ms）

**反例（慢）**（两种坏策略）：

```cpp
// 最差 864ms：静态切分 + 绑核（E 核线程无法迁移到已空闲的 P 核）
#pragma omp for schedule(static)    // 且 OMP_PROC_BIND=true
// 次差 567ms：静态切分、不绑核（P 核先完成后 E 核拖尾）
#pragma omp for schedule(static)
```

**正例（快）**（动态切分，最优 128 chunks）：

```cpp
#pragma omp for schedule(dynamic, N/128)
// 实验汇总（10 次平均延迟 ms）:
// Affinity 864 | Static 567 | Dynamic 4: 570 | Dynamic 16: 541
// | Dynamic 128: 517 | Dynamic 1024: 560
```

**为什么**：P 核处理 SIMD 快一倍，静态等分必然拖尾；绑核禁止 work stealing 使 E 核线程在 barrier 干等。动态小粒度 chunk 让运行时持续均衡分发；chunk 过小（1024 份）管理开销反超收益。两条建议：避免绑核；异构核系统避免静态切分。
**来源**：perf-book/chapters/13-Optimizing-Multithreaded-Applications/13-4 Task Scheduling.md

### Spin Time 与有效 CPU 利用率（方法论公式）

```
Effective CPU Time = CPU Time − (Overhead Time + Spin Time)
Effective CPU Utilization = Σ(Effective CPU Time(T,i)) / (T × ThreadCount)
```

**为什么**：Spin Time 是"CPU 忙着但线程在等"的等待时间（内核同步原语先自旋一段时间才让出）——线程可能显示高 CPU 利用率，实际只是在锁上空转（正是"锁竞争导致 CPU 飙高"的指标化表达）；评估并行效率必须用扣除 Overhead+Spin 的 Effective 指标（VTune 可直接给出）。Sync Wait Time 大 → 高竞争同步对象；Preemption Wait Time 大 → 线程过多。
**来源**：perf-book/chapters/13/13-1 Parallel Efficiency Metrics.md

## C. 低延迟 / 系统级

### Minor page fault 预触：mallopt + mlockall 组合

**反例（慢）**（运行期首次访问才触发缺页，只做简单预触也不彻底）：

```cpp
char *mem = malloc(size);
int pageSize = sysconf(_SC_PAGESIZE)
for (int i = 0; i < size; i += pageSize)
  mem[i] = 0;
```

**正例（快）**（glibc 调优 + 锁页，防止内存被回收给 OS 后再次缺页）：

```cpp
#include <malloc.h>
#include <sys/mman.h>

mallopt(M_MMAP_MAX, 0);
mallopt(M_TRIM_THRESHOLD, -1);
mallopt(M_ARENA_MAX, 1);

mlockall(MCL_CURRENT | MCL_FUTURE);

char *mem = malloc(size);
for (int i = 0; i < size; i += sysconf(_SC_PAGESIZE))
    mem[i] = 0;
// ...
free(mem);
```

**为什么**：`M_MMAP_MAX=0` 禁止 malloc 用 mmap（munmap 会抵消 mlockall）；`M_TRIM_THRESHOLD=-1` 禁止 free 后归还内存；`M_ARENA_MAX=1` 禁多 arena（牺牲 glibc 多核扩展性）；`mlockall(MCL_CURRENT|MCL_FUTURE)` 把当前及将来映射页全部锁在 RAM，新线程栈也自动预触+锁定。检测：`top` 加 vMn 列（-H 线程级）、`perf stat -e page-faults`；HFT 要求 0，一般系统 100–1000 faults/s 即应调查（fault 延迟 <1μs–数μs）。
**来源**：perf-book/chapters/12-Other-Tuning-Areas/12-4 Low-Latency-Tuning-Techniques.md

### AVX-512 降频禁用

**命令**：

```bash
$ clang++ -mprefer-vector-width=256 ...   # 或 128，钉死最大向量宽度
```

**为什么**：老一代 CPU 执行 heavy AVX512 指令会触发降频，出现无解释的延迟异常；把指令最大宽度钉在 128/256 避免不知情的 AVX512 生成。最新芯片上该问题已基本可忽略。
**来源**：perf-book/chapters/12-Other-Tuning-Areas/12-4

## D. 方法论

### Clang -Rpass 三类报告：发现未向量化 + 换行修复

**反例（慢）**（loop-carry 依赖导致未向量化）：

```cpp
void foo(float* __restrict__ a,
         float* __restrict__ b,
         float* __restrict__ c,
         unsigned N) {
  for (unsigned i = 1; i < N; i++) {
    a[i] = c[i-1]; // value is carried over from previous iteration
    c[i] = b[i];
  }
}
```

```bash
$ clang -O3 -Rpass-analysis=.* -Rpass=.* -Rpass-missed=.* a.c -c
a.c:5:3: remark: loop not vectorized [-Rpass-missed=loop-vectorize]
```

**正例（快）**（交换两行语义不变，打破依赖）：

```cpp
  for (unsigned i = 1; i < N; i++) {
    c[i] = b[i];
    a[i] = c[i-1];
  }
```

```bash
a.cpp:5:3: remark: vectorized loop (vectorization width: 8, interleaved count: 4) [-Rpass=loop-vectorize]
```

**为什么**：`-Rpass=.*`（做了什么）、`-Rpass-missed=.*`（错过了什么）、`-Rpass-analysis=.*`（为什么错过）三类 remark 直接暴露编译器决策。原代码 `a[i]=c[i-1]` 依赖上一轮对 `c[i-1]` 的写入，向量化会写错值。
**来源**：perf-book/chapters/5-Performance-Analysis-Approaches/5-8 Compiler Opt Reports.md

### GCC -fopt-info 报告（别名检测与循环多版本）

**命令**：

```bash
$ gcc -O3 -march=core-avx2 -fopt-info
a.cpp:2:26: optimized: loop vectorized using 32-byte vectors
a.cpp:2:26: optimized:  loop versioned for vectorization because of possible aliasing
```

**为什么**：GCC 无法证明指针不重叠时，生成多版本循环 + 运行时别名检查再分发；确知不重叠时用 `#pragma GCC ivdep` 或 `__restrict__` 消除运行时检查。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-4 Vectorization.md

### DoNotOptimize 防死代码消除（微基准最大坑）

**反例（无效基准）**：

```cpp
// foo DOES NOT benchmark string creation（整个循环被编译器删除）
void foo() {
  for (int i = 0; i < 1000; i++)
    std::string s("hi");
}
```

**正例（有效基准）**：

```cpp
// foo benchmarks string creation
void foo() {
  for (int i = 0; i < 1000; i++) {
    std::string s("hi");
    DoNotOptimize(s);     // JMH 中对应 Blackhole.consume()
  }
}
```

**为什么**：`DoNotOptimize`（Google benchmark 内联汇编魔法，向编译器假装值被"使用"）阻止编译器消除被测代码。配套原则："Always Measure One Level Deeper"——对 benchmark 本身做 profile 确认被测代码确实是热点，并用真实输入（合成输入会误导结论）。
**来源**：perf-book/chapters/2-Measuring-Performance/2-6 Microbenchmarks.md

<!-- 来源: external/perf-book/chapters/11-Machine-Code-Layout-Optimizations/ -->
<!-- 来源: external/perf-book/chapters/12-Other-Tuning-Areas/12-4 -->
<!-- 来源: external/perf-book/chapters/13-Optimizing-Multithreaded-Applications/ -->
<!-- 来源: external/perf-book/chapters/5-Performance-Analysis-Approaches/5-8 -->
<!-- 来源: external/perf-book/chapters/2-Measuring-Performance/2-6 -->
<!-- 来源: external/perf-book/chapters/9-Optimizing-Computations/9-4 -->
<!-- 来源: external/perf-ninja/labs/misc/ -->
<!-- 来源: external/perf-ninja/labs/memory_bound/false_sharing_1/ -->
