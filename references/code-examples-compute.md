# Core Bound / Bad Speculation 修复代码样例（反例 → 正例）

> 配合 references/optimization-playbook.md §3/§4 使用。每条含实测收益（若有）。
> 注：perf-ninja 本地仓库 solution.cpp 为 baseline（即反例），优化正例出自各 lab README 的 Worked Solution / 上游 golden 分支。

## A. 依赖链断裂（Core Bound）

### 双 RNG 交织（书 19ms→10ms 近 2x，IPC 4.0→7.1）

**反例（慢）**（XorShift32 有状态，全部迭代串成一条链）：

```cpp
class XorShift32 {
  uint32_t val;
public:
  XorShift32 (uint32_t seed) : val(seed) {}
  uint32_t gen() {
    val ^= (val << 13);
    val ^= (val >> 17);
    val ^= (val << 5);
    return val;
  }
};

void particleMotion(vector<Particle> &particles, uint32_t seed) {
  XorShift32 rng(seed);
  for (int i = 0; i < STEPS; i++)
    for (auto &p : particles) {
      uint32_t angle = rng.gen();
      float angle_rad = angle * DEGREE_TO_RADIAN;
      p.x += cosine(angle_rad) * p.velocity;
      p.y += sine(angle_rad) * p.velocity;
    }
}
```

**正例（快）**（两个 RNG 分别喂奇偶迭代，两条并行链）：

```cpp
void particleMotion(vector<Particle> &particles,
                    uint32_t seed1, uint32_t seed2) {
  XorShift32 rng1(seed1);
  XorShift32 rng2(seed2);
  for (int i = 0; i < STEPS; i++) {
    for (int j = 0; j + 1 < particles.size(); j += 2) {
      uint32_t angle1 = rng1.gen();
      float angle_rad1 = angle1 * DEGREE_TO_RADIAN;
      particles[j].x += cosine(angle_rad1) * particles[j].velocity;
      particles[j].y += sine(angle_rad1)   * particles[j].velocity;
      uint32_t angle2 = rng2.gen();
      float angle_rad2 = angle2 * DEGREE_TO_RADIAN;
      particles[j+1].x += cosine(angle_rad2) * particles[j+1].velocity;
      particles[j+1].y += sine(angle_rad2)   * particles[j+1].velocity;
    }
    // remainder (not shown)
  }
}
```

**为什么**：`gen()` 的 val 存在跨迭代递归依赖（3 对 eor+shift 串行，6 周期/迭代即软件下限），其余 fmul/fmadd 不在关键路径；两条独立链可并行，逼近硬件吞吐上限（Apple M1 上 2.25 周期/迭代）。lab 版（dep_chains_2）是同案例的手工 2 展开版：单 RNG 但每 2 个粒子调用一次 gen；README Bonus 给出更通用的解法——把 gen() 改造成一次产出两个交错（interleave）独立链的随机数。**长链（万条指令级）必须逐语句交织两条链**，只分块执行两条链仅 +5%（RS 容量看不见第二条链）。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-1 Data Dependencies.md；perf-ninja/labs/core_bound/dep_chains_2/

### 链表分块重叠多条依赖链（85.5ms→34.7ms 约 2.5x）

**反例（慢）**（O(N²) 链表查找，指针追逐）：

```cpp
unsigned solution(List *l1, List *l2) {
  unsigned retVal = 0;
  List *head2 = l2;
  while (l1) {
    unsigned v = l1->value;
    l2 = head2;
    while (l2) {
      if (l2->value == v) {
        retVal += getSumOfDigits(v);
        break;
      }
      l2 = l2->next;
    }
    l1 = l1->next;
  }
  return retVal;
}
```

**正例（快）**（一次摘 4 个节点，遍历 B 时对 4 值同时比较）：

```cpp
template<int N> // N = 4：节点 16 字节，4 个节点 = 1 条 cache line
unsigned solution(List *l1, List *l2) {
  // ...
  for (int i = 0; i < length1 / N; i++) {
    std::array<unsigned, N> arr{};
    for (int j = 0; j < N; j++) {
      arr[j] = l1->value;
      l1 = l1->next;
    }
    int seen = 0;
    l2 = head2;
    while (l2) {
      int v = l2->value;
      for (int j = 0; j < N; j++) {
        if (arr[j] == v) {
          retVal += getSumOfDigits(v);
          if (++seen == N) {
            break;
          }
        }
      }
      l2 = l2->next;
    }
  }
  // ...
}
```

**为什么**：取链表节点 N+1 必须先拿到节点 N，指针追逐是扯不断的依赖链，ILP 极低；一次从 A 链表摘下 4 个节点（arena 分配保证相邻、同一条 cache line），遍历 B 时对 4 个值同时比较——4 条依赖链在 B 的每次跳转间重叠执行。
**来源**：perf-ninja/labs/core_bound/dep_chains_1/

### FMA 融合伤性能（4 cycles/iter → 2 cycles/iter，nanoBench 实测）

**反例（慢）**（编译器自动融合乘加，乘法被卷入累加依赖链）：

```cpp
float sqSum(float *a, int N) {          // .loop:
  float sum = 0;                        //  vmovss xmm1, dword ptr [rcx + 4*rdx]
  for (int i = 0; i < N; i++ )          //  vfmadd231ss xmm0, xmm1, xmm1
    sum += a[i] * a[i];                 //  inc rdx
  return sum;                           //  cmp rax, rdx
}                                       //  jne .loop
```

**正例（快）**（拆开乘与加，乘法脱离关键路径）：

```cpp
// 汇编形态（vmulss 与 vaddss 分离后乘法可并行）：
// Instructions retired: 3.00, Core cycles: 2.00
// Clang 18+ 可用 #pragma clang fp contract(off) 在作用域内禁止融合
```

**为什么**：`vfmadd231ss` 把乘加融成一条后，乘法也进入 xmm0 的循环携带依赖链（FMA 延迟 4 周期）；拆开则只剩 FADD 的 2 周期依赖，乘法乱序并行。
**来源**：perf-book/chapters/12-Other-Tuning-Areas/12-1 CPU-Specific Optimizations.md

## B. 内联（Core Bound）

### qsort → std::sort（760µs→518µs 约 1.5x）

**反例（慢）**：

```cpp
static int compare(const void *lhs, const void *rhs) {
  auto &a = *reinterpret_cast<const S *>(lhs);
  auto &b = *reinterpret_cast<const S *>(rhs);
  if (a.key1 < b.key1) return -1;
  if (a.key1 > b.key1) return 1;
  if (a.key2 < b.key2) return -1;
  if (a.key2 > b.key2) return 1;
  return 0;
}

void solution(std::array<S, N> &arr) {
  qsort(arr.data(), arr.size(), sizeof(S), compare);
}
```

**正例（快）**：

```cpp
void solution(std::array<S, N> &arr) {
  std::sort(arr.begin(), arr.end(), [](const S& a, const S& b) {
    return a.key1 < b.key1 || (a.key1 == b.key1 && a.key2 < b.key2);
  });
  // C++20 也可用 std::ranges::sort
}
```

**为什么**：qsort 是已编译的 C 库函数，比较器经 `void*` 间接函数指针调用，编译器无法内联，每次比较都付调用序言/尾声；std::sort 的模板代码 + lambda 可完全内联，还解锁后续优化。
**来源**：perf-ninja/labs/core_bound/function_inlining_1/

### 强制内联属性

**正例（快）**（profile 显示函数 prologue/epilogue 占 ~50% 时间时的强信号）：

```cpp
[[gnu::always_inline]] int foo() {
    // foo body
}
// 旧标准：__attribute__((always_inline))；MSVC：__forceinline
```

**为什么**：内联消除 CALL/RET + prologue/epilogue 并扩大编译器分析范围；强制内联可能反而变慢（代码膨胀），务必测量。`[[unlikely]]` 反向可阻止冷函数被内联。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-2 Inlining Functions.md

### 尾调用优化（尾递归 → 迭代）

**反例（慢）**（-O0 下生成递归调用，栈帧堆积）：

```cpp
int sum(int n, int acc) {
  if (n == 0) {
    return acc;
  } else {
    return sum(n - 1, acc + n);
  }
}
```

**正例（快）**（-O2 编译器自动转换的等价迭代）：

```cpp
int sum(int n, int acc) {
  for (int i = n; i > 0; --i) {
    acc += i;
  }
  return acc;
}
```

**为什么**：-O2 下编译器识别尾递归，复用当前栈帧（call 换 jmp）并改写成迭代（GCC 13.2 对两版生成相同机器码）；Clang 可用 `__attribute__((musttail))` 强制保证。存疑时直接写迭代版。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-2

## C. 循环优化（编译器通常自动做，看懂用途即可）

### 不变量外提 LICM

**反例（慢）**：

```cpp
for (int i = 0; i < N; ++i)
  for (int j = 0; j < N; ++j)
    a[j] = b[j] * c[i];
```

**正例（快）**：

```cpp
for (int i = 0; i < N; ++i) {
  auto temp = c[i];
  for (int j = 0; j < N; ++j)
    a[j] = b[j] * temp;
}
```

**为什么**：c[i] 在内层不变，外提后每轮 i 只读一次；编译器仅在能证明 a、c 不别名时才自动做。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-3 Loop Optimizations.md

### 强度削减 LSR（乘法变加法）

**反例（慢）**：

```cpp
for (int i = 0; i < N; ++i)
  a[i] = b[i * 10] * c[i];
```

**正例（快）**：

```cpp
int j = 0;
for (int i = 0; i < N; ++i) {
  a[i] = b[j] * c[i];
  j += 10;
}
```

**为什么**：归纳变量的线性函数（i*10）可用递增计数器替代，乘法换加法。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-3

### 循环外提开关 Unswitching

**反例（慢）**：

```cpp
for (i = 0; i < N; i++) {
  a[i] += b[i];
  if (c)
    b[i] = 0;
}
```

**正例（快）**：

```cpp
if (c)
  for (i = 0; i < N; i++) {
    a[i] += b[i];
    b[i] = 0;
  }
else
  for (i = 0; i < N; i++) {
    a[i] += b[i];
  }
```

**为什么**：循环不变条件搬出循环，消除每次迭代的分支；代价是代码翻倍，但两个循环可各自独立优化。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-3

### Unroll and Jam（外层展开 + 内层融合，双累加器）

**反例（慢）**：

```cpp
for (int i = 0; i < N; i++)
  for (int j = 0; j < M; j++)
    diffs += a[i][j] - b[i][j];
```

**正例（快）**：

```cpp
for (int i = 0; i+1 < N; i+=2)
  for (int j = 0; j < M; j++) {
    diffs1 += a[i][j]   - b[i][j];
    diffs2 += a[i+1][j] - b[i+1][j];
  }
diffs = diffs1 + diffs2;
// remainder (not shown)
```

**为什么**：外层展开再融合，两个独立累加器打破原 diffs 上的依赖链，提升内层 ILP；对内层 trip count 很小（<4）的循环尤其有效，也用于促成外层向量化（SLP）。适用条件：外层无跨迭代依赖、内层访存在外层索引上有 stride。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-3

## D. 向量化（Core Bound）

### 数据布局转置暴露 SIMD（1252µs→497µs 约 2.5x）

**反例（慢）**（标量累加器，一次算一条序列比对）：

```cpp
using score_t = int16_t;
using column_t = std::array<score_t, sequence_size_v + 1>;
// ... 逐 cell 递推（细节略）：
for (unsigned row = 1; row <= sequence1.size(); ++row) {
    score_t best_cell_score =
        last_diagonal_score +
        (sequence1[row - 1] == sequence2[col - 1] ? match : mismatch);
    best_cell_score = std::max(best_cell_score, last_vertical_gap);
    best_cell_score = std::max(best_cell_score, horizontal_gap_column[row]);
    // ...
}
```

**正例（快）**（转置后 16 条序列同位置得分连续 = 256bit YMM，累加器全部升为 16-lane）：

```cpp
using simd_score_t = std::array<int16_t, sequence_count_v>;  // 16 lanes
using simd_sequence_t = std::array<simd_score_t, sequence_size_v>;

simd_sequence_t transpose(const std::vector<sequence_t>& vec) {
    simd_sequence_t transposed{};
    for (size_t i = 0; i < transposed.size(); ++i)
        for (size_t j = 0; j < vec.size(); ++j)
            transposed[i][j] = vec[j][i];
    return transposed;
}
// 计算循环中 score_t 全部换成 simd_score_t，逐 lane 操作
// 注意：match/mismatch 填充循环与 max 更新循环必须分开写，
// 合并成逐列填充则失去向量机会
```

**为什么**：转置使 16 条序列同一位置的得分连续存放，正好填满 YMM 寄存器；编译器即可自动向量化（perf 可见 vmovdqa ymm）。
**来源**：perf-ninja/labs/core_bound/vectorization_1/

### 加宽累加器消除进位依赖（33.4µs→3.61µs 约 9x）

**反例（慢）**（每次迭代依赖上次加法是否溢出，add+adc 串行链）：

```cpp
uint16_t checksum(const Blob &blob) {
  uint16_t acc = 0;
  for (auto value : blob) {
    acc += value;
    acc += acc < value; // add carry
  }
  return acc;
}
```

**正例（快）**（32 位累加器，低/高 16 位当两个并行通道，每 2^16 个元素补一次进位）：

```cpp
uint16_t checksum(const Blob& blob) {
  constexpr std::size_t two_pow_16 = 1 << 16;
  uint32_t acc = 0, prev = 0;
  for (std::size_t i = 0; i < N; i += two_pow_16) {
    for (std::size_t j = i; j < i + two_pow_16 && j < N; j++) {
      acc += blob[j];
    }
    if (acc < prev) {
      acc++;
    }
    prev = acc;
  }
  uint16_t top = acc >> 16;
  uint16_t bottom = acc & (two_pow_16 - 1);
  uint16_t ans = top + bottom;
  ans += ans < top;
  return ans;
}
```

**为什么**：`acc += acc < value` 形成串行依赖链无法向量化；32 位累加器让纯加法内循环可被向量化，进位折叠进高位（RFC 1071 技巧）。
**来源**：perf-ninja/labs/core_bound/vectorization_2/

### 浮点重排许可：-ffast-math / fp pragma

**反例（慢）**：

```cpp
float calcSum(float* a, unsigned N) {
  float sum = 0.0f;
  for (unsigned i = 0; i < N; i++) {
    sum += a[i];
  }
  return sum;
}
// 报告：remark: loop not vectorized: cannot prove it is safe to
//       reorder floating-point operations ...
```

**正例（快）**：

```bash
$ clang++ -c a.cpp -O3 -march=core-avx2 -ffast-math -Rpass=.*
a.cpp:4:3: remark: vectorized loop (vectorization width: 4, interleaved count: 2) [-Rpass=loop-vectorize]
```

**为什么**：浮点加法不满足结合律（舍入时机不同），编译器默认不许重排；`-ffast-math`（`-Ofast` = `-O3` + `-ffast-math`）声明可容忍微小差异后即可向量化。副作用涉及 NaN/带符号零/无穷/次正规数；Clang 18+ 可用 `#pragma clang fp reassociate(on)` 局部开启。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-4 Vectorization.md

### 消除别名假设：ivdep / **restrict**

**反例（慢）**（GCC 报 `loop versioned for vectorization because of possible aliasing`——生成运行时重叠检查+双版本分派）：

```cpp
void foo(float* a, float* b, float* c, unsigned N) {
  for (unsigned i = 1; i < N; i++) {
    c[i] = b[i];
    a[i] = c[i-1];
  }
}
```

**正例（快）**：

```cpp
#pragma GCC ivdep          // GCC 专用
// 或参数声明为 float* __restrict__ a, ...
```

**为什么**：编译器无法证明 a/b/c 不重叠时只能保守做多版本+运行时检查；开发者确知不重叠时给出提示即可消除该开销。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-4

### 强制向量化 pragma

**反例（慢）**（代价模型判定不划算：B[i*3] 是 gather 访问）：

```cpp
void stridedLoads(int *A, int *B, int n) {
  for (int i = 0; i < n; i++)
    A[i] += B[i * 3];
}
```

**正例（快）**：

```cpp
void stridedLoads(int *A, int *B, int n) {
#pragma clang loop vectorize(enable)
  for (int i = 0; i < n; i++)
    A[i] += B[i * 3];
}
// 相关：vectorize_width(N) 控制宽度；vectorize(disable) 禁用
```

**为什么**：scatter/gather 昂贵，编译器不知运行时 trip count 而保守放弃；pragma 强制（适合实验），是否真有收益取决于实际迭代次数与数据。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-4

### 读写依赖：不可向量化的硬限制（单向示例）

**反例（慢）**（无正例——这是硬限制，需改算法）：

```cpp
void vectorDependence(int *A, int n) {
  for (int i = 1; i < n; i++)
    A[i] = A[i-1] * 2;   // A[i] 依赖上一迭代写入的 A[i-1]
}
```

**为什么**：read-after-write 跨迭代依赖，展开两次迭代即可看出，属于无法向量化的硬限制。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-4

### Intrinsics 基本用法与水平归约（单向示例）

**正例（快）**：

```cpp
#include <immintrin.h>

float calcSum(float* a, unsigned N) {
  __m128 sum = _mm_setzero_ps();      // init sum with zeros
  unsigned i = 0;
  for (; i + 3 < N; i += 4) {
    __m128 vec = _mm_loadu_ps(a + i); // load 4 floats
    sum = _mm_add_ps(sum, vec);       // accumulate
  }

  // Horizontal sum of the 128-bit vector
  __m128 shuf = _mm_movehdup_ps(sum); // broadcast elements 3,1 to 2,0
  sum = _mm_add_ps(sum, shuf);        // partial sums [0+1] and [2+3]
  shuf = _mm_movehl_ps(shuf, sum);    // high half -> low half
  sum = _mm_add_ss(sum, shuf);        // result in the lower element
  float result = _mm_cvtss_f32(sum);

  for (; i < N; i++)                  // remainder
      result += a[i];
  return result;
}
```

**为什么**：intrinsics 与汇编指令近 1:1（有类型检查、寄存器分配归编译器），优于内联汇编；末尾三步做 128 位向量内水平求和。仅当编译器实在生成不了目标指令时用，需自管余量与边界。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-5 Compiler Intrinsics.md

### ISPC 与 Highway（可移植向量化，单向示例）

```cpp
// ISPC：类 C 的 SPMD 语言，一份代码编译到 SSE4/AVX2/NEON
export uniform float calcSum(const uniform float array[],
                             uniform ptrdiff_t count)
{
    varying float sum = 0;
    foreach (i = 0 ... count)
        sum += array[i];
    return reduce_add(sum);
}
```

```cpp
// Highway：C++11 可移植 intrinsics 包装（AVX2/AVX-512/NEON 自适应）
#include <hwy/highway.h>

float calcSum(const float* HWY_RESTRICT array, size_t count) {
  const ScalableTag<float> d;  // type descriptor; no actual data
  auto sum = Zero(d);
  size_t i = 0;
  for (; i + Lanes(d) <= count; i += Lanes(d)) {
    sum = Add(sum, LoadU(d, array + i));
  }
  sum = Add(sum, MaskedLoad(FirstN(d, count - i), d, array + i));
  return ReduceSum(d, sum);
}
```

**为什么**：ISPC 默认一切操作皆 SIMD（`foreach` 自动分摊到各 program instance）；Highway 用类型描述符 + 运行时分簇派发。性能可匹敌手写 intrinsics（Unreal Engine Chaos 物理用 ISPC 重写有加速）。同类还有 std::experimental::simd。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-4/9-5

## E. 手写 Intrinsics 案例（lab）

### SSE 向量前缀和（25.4µs→10.4µs 约 2.4x）

**反例（慢）**：

```cpp
// 滑动窗口的和：标量前缀和，累加结果每次写回内存
limit = size - radius;
for (; pos < limit; ++pos) {
  currentSum -= input[pos - radius - 1];
  currentSum += input[pos + radius];
  output[pos] = currentSum;
}
```

**正例（快）**（对数复杂度 SIMD 前缀和：1/2/4/8 字节逐级 shift-add）：

```cpp
using v8i = __m128i;
v8i current = _mm_set1_epi16(currentSum);

int i = 0;
for (; i + 7 + pos < limit; i += 8) {
  v8i sub_pckd_8 = _mm_loadu_si64(subtract_ptr + i);
  v8i add_pckd_8 = _mm_loadu_si64(add_ptr + i);
  v8i sub_ext_16 = _mm_cvtepu8_epi16(sub_pckd_8);   // u8 -> u16
  v8i add_ext_16 = _mm_cvtepu8_epi16(add_pckd_8);

  v8i diff = _mm_sub_epi16(add_ext_16, sub_ext_16);

  v8i delta = _mm_add_epi16(diff, _mm_slli_si128(diff, 2));
  delta = _mm_add_epi16(delta, _mm_slli_si128(delta, 4));
  delta = _mm_add_epi16(delta, _mm_slli_si128(delta, 8));

  v8i result = _mm_add_epi16(delta, current);
  _mm_storeu_si128((v8i*)(output_ptr + i), result);

  currentSum = static_cast<uint16_t>(_mm_extract_epi16(result, 7));
  current = _mm_set1_epi16(currentSum);
}
pos += i;
```

**为什么**：滑动窗口差分（add−sub）向量化后，前缀和用 log 复杂度逐级左移累加（小端序所以是左移 `_mm_slli_si128`），末元素广播接续下一轮。
**来源**：perf-ninja/labs/core_bound/compiler_intrinsics_1/

### AoS 跨界加载解交错求和（golden 分支 RISC-V 实测 70µs→53µs）

**反例（慢）**：

```cpp
Position<std::uint32_t> solution(std::vector<Position<std::uint32_t>> const &input) {
  std::uint64_t x = 0, y = 0, z = 0;
  for (auto pos: input) {
    x += pos.x;
    y += pos.y;
    z += pos.z;
  }
  // ...
}
```

**正例（快）**（x86 无 LD3，不 shuffle——4 个 Position = 48B = 3 次 128-bit 加载，lane 天然呈 XYZX/YZXY/ZXYZ）：

```cpp
int i = 0;
__m256i acc_XYZX = _mm256_setzero_si256();
__m256i acc_YZXY = _mm256_setzero_si256();
__m256i acc_ZXYZ = _mm256_setzero_si256();
constexpr int UNROLL = 4;
auto input_ptr = reinterpret_cast<const __m128i*>(&input[0].x);
for (; i + UNROLL - 1 < input.size(); i += UNROLL) {
  __m128i XMM_XYZX = _mm_load_si128(input_ptr + 0); // load 128 bits
  __m128i XMM_YZXY = _mm_load_si128(input_ptr + 1);
  __m128i XMM_ZXYZ = _mm_load_si128(input_ptr + 2);
  input_ptr += 3;
  __m256i YMM_XYZX = _mm256_cvtepu32_epi64(XMM_XYZX); // 32 -> 64 bit
  __m256i YMM_YZXY = _mm256_cvtepu32_epi64(XMM_YZXY);
  __m256i YMM_ZXYZ = _mm256_cvtepu32_epi64(XMM_ZXYZ);
  acc_XYZX = _mm256_add_epi64(acc_XYZX, YMM_XYZX);
  acc_YZXY = _mm256_add_epi64(acc_YZXY, YMM_YZXY);
  acc_ZXYZ = _mm256_add_epi64(acc_ZXYZ, YMM_ZXYZ);
}
// 归约：3 个累加器按 lane 顺序归属到 x/y/z
x += _mm256_extract_epi64(acc_XYZX, 0);
y += _mm256_extract_epi64(acc_XYZX, 1);
z += _mm256_extract_epi64(acc_XYZX, 2);
x += _mm256_extract_epi64(acc_XYZX, 3);
// acc_YZXY / acc_ZXYZ 同理（lane 错位归属）
```

**为什么**：x86 编译器对交错存放的 {x,y,z} 无法自动去交错（ARM 有 LD3）；按 128-bit 整块加载后 3 个独立累加器并行跑满加法吞吐。
**来源**：perf-ninja/labs/core_bound/compiler_intrinsics_3/（正例取自上游 golden 分支）

### Mandelbrot 掩码批处理 + 软件流水（正例取自 golden 分支）

**反例（慢）**（每像素迭代次数不同，break 时刻不一，无法向量化）：

```cpp
for (auto py = 0; py < data_height; ++py) {
  for (auto px = 0; px < data_width; ++px) {
    const auto c_x = min_x + (max_x - min_x) * px / data_width;
    const auto c_y = min_y + (max_y - min_y) * py / data_height;
    auto z_x = 0.0, z_y = 0.0;
    auto iter_cnt = 0;
    for (; iter_cnt < kMaxIterations; ++iter_cnt) {
      const auto z_xx = z_x * z_x;
      const auto z_yy = z_y * z_y;
      if (z_xx + z_yy > kSquareBound) {
        break;
      }
      const auto z_xy = z_x * z_y;
      z_x = z_xx - z_yy + c_x;
      z_y = z_xy + z_xy + c_y;
    }
    result[result_idx++] = iter_cnt;
  }
}
```

**正例（快）**（核心思路：不 break——向量迭代照跑，cmpgt+movemask 找逃逸 lane，逐 lane 记录 iter_cnt 后在 active_mask 关掉该 lane）：

```cpp
// vec_* 为按架构选择的 SIMD 包装（AVX512/AVX2/SSE/NEON）
constexpr auto kVecSize = sizeof(Vec) / sizeof(double);
constexpr int kUnrollSz = 4;                      // 软件流水，提高 ILP
const auto kBatchSize = kVecSize * kUnrollSz;
const auto squared_bound = vec_set1(kSquareBound);

for (auto data_idx = 0; data_idx < data_size; data_idx += kBatchSize) {
  alignas(sizeof(Vec)) std::array<Vec, kUnrollSz> c_x, c_y, z_x, z_y;
  std::array<std::array<short, kVecSize>, kUnrollSz> res;
  std::array<uint32_t, kUnrollSz> active_mask;

  for (auto u = 0; u < kUnrollSz; ++u) {
    // 逐像素填 c_x/c_y 后 vec_load；z 清零
    res[u].fill(kMaxIterations);
    active_mask[u] = (1 << kVecSize) - 1;         // 所有 lane 激活
  }

  auto active_vec_cnt = kUnrollSz;
  for (auto iter_cnt = 0; iter_cnt < kMaxIterations && active_vec_cnt != 0; ++iter_cnt) {
    for (auto u = 0; u < kUnrollSz; ++u) {
      if (active_mask[u] == 0) continue;
      const auto z_xx = vec_mul(z_x[u], z_x[u]);
      const auto z_yy = vec_mul(z_y[u], z_y[u]);
      // 溢出判定向量化；逐位处理逃逸像素
      for (uint32_t mask = vec_movemask(vec_cmpgt(vec_add(z_xx, z_yy), squared_bound)) & active_mask[u];
           mask; mask &= mask - 1) {
        const auto res_idx = std::countr_zero(mask);
        active_mask[u] &= ~((uint32_t)1 << res_idx);
        res[u][res_idx] = iter_cnt;
        active_vec_cnt -= active_mask[u] == 0;
      }
      const auto z_xy = vec_mul(z_x[u], z_y[u]);  // z = z^2 + c 向量化迭代
      z_x[u] = vec_add(vec_sub(z_xx, z_yy), c_x[u]);
      z_y[u] = vec_add(vec_add(z_xy, z_xy), c_y[u]);
    }
  }
  // res 拷回输出数组
}
```

**为什么**：发散迭代控制的 SIMD 化核心是"不 break、用掩码关闭已逃逸 lane"；kUnrollSz=4 份向量软件流水弥补乘法延时的 ILP 缺口。注意该 lab 禁用 `-ffast-math`（验证需要），必须手写 intrinsic。
**来源**：perf-ninja/labs/core_bound/compiler_intrinsics_4/（正例取自上游 golden 分支）

## F. 分支预测优化（Bad Speculation）

### 查表替换比较链（lab 5.5x：5475µs→995µs，误预测 11.93%→0.01%，IPC 1.03→2.67）

**反例（慢）**：

```cpp
static std::size_t mapToBucket(std::size_t v) {
  if      (v < 13)  return 0; //   13
  else if (v < 29)  return 1; //   16
  else if (v < 41)  return 2; //   12
  else if (v < 53)  return 3; //   12
  else if (v < 71)  return 4; //   18
  else if (v < 83)  return 5; //   12
  else if (v < 100) return 6; //   17
  return DEFAULT_BUCKET;
}
```

**正例（快）**：

```cpp
constexpr int TABLE_SIZE = 101;
uint8_t lookup_table[TABLE_SIZE] = {
  0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
  1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
  2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
  3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
  4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
  5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
  6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6, 6,
  DEFAULT_BUCKET
};
static std::size_t mapToBucket(std::size_t v) {
  return lookup_table[std::min(v, TABLE_SIZE - 1)];
}
```

书的通用模式（剩一条边界保护分支）：

```cpp
int8_t buckets[50] = { 0,0,..., 1,1,..., 2,2,..., 3,3,..., 4,4,... };
int8_t mapToBucket(unsigned v) {
  if (v < (sizeof(buckets) / sizeof(int8_t)))
    return buckets[v];
  return -1;
}
```

**为什么**：7 级阈值比较对随机输入全部不可预测；值域仅 [0,150]，用 101 项表直接索引，`std::min` 钳位越界值且 -O3 下编译为无分支代码。大范围映射用分段表/interval map（Boost interval_map、LLVM IntervalMap）。
**来源**：perf-ninja/labs/bad_speculation/lookup_tables_1/；perf-book/chapters/10/10-1

### 无条件写 + 谓词计数（4.36x：286µs→65.6µs，误预测 23.54%→0.03%）

**反例（慢）**：

```cpp
std::size_t select(std::array<S, N> &output, const std::array<S, N> &input,
                   const std::uint32_t lower, const std::uint32_t upper) {
  std::size_t count = 0;
  for (const auto item : input) {
    if ((lower <= item.first) && (item.first <= upper)) {
      output[count++] = item;
    }
  }
  return count;
}
```

**正例（快）**：

```cpp
for (const auto item : input) {
    output[count] = item;
    count += lower <= item.first && item.first <= upper;
}
```

**为什么**：key 随机导致 if 谓词不可预测；改为每个元素无条件写入 `output[count]`，再把谓词的 0/1 整数值加到 count——不命中范围的预写会被后续迭代覆盖，分支消失（IPC 0.56→3.34）。
**来源**：perf-ninja/labs/bad_speculation/conditional_store_1/

### cmov 条件传送 + __builtin_unpredictable（生命游戏 lab）

**反例（慢）**（switch 分支随机不可预测）：

```cpp
switch(aliveNeighbours) {
    case 0:
    case 1:
        future[i][j] = 0;   // lonely and dies
        break;
    case 2:
        future[i][j] = current[i][j];  // remains the same
        break;
    case 3:
        future[i][j] = 1;   // a new cell is born
        break;
    default:
        future[i][j] = 0;   // dies due to over population
}
```

**正例（快）**——第一步先收敛分支（除 2/3 外恒 0）：

```cpp
int cell_value = 0;
if (aliveNeighbours == 2) {
    cell_value = current[i][j];
} else if (aliveNeighbours == 3) {
    cell_value = 1;
}
future[i][j] = cell_value;
```

第二步加提示生成 cmov（Clang 17+，x86 生成 `cmov`，ARM 生成 `csel`）：

```cpp
int cell_value = 0;
if (__builtin_unpredictable(aliveNeighbours == 2)) {
    cell_value = current[i][j];
} else if (__builtin_unpredictable(aliveNeighbours == 3)) {
    cell_value = 1;
}
future[i][j] = cell_value;
```

书的标准模式与汇编对照：

```cpp
// 反例                          // 正例（branchless）
int a;                           int x = computeX();
if (cond) { a = computeX(); }    int y = computeY();
else      { a = computeY(); }    int a = cond ? x : y;
foo(a);                          foo(a);
```

```bash
# original（有跳转）             # branchless（无跳转指令，cmovne 收尾）
400506: je 400514                40054d: test ebx,ebx
400508: call <computeX>          40054f: cmovne eax,ebp  # override a with x
400512: jmp 40051e
400519: call <computeY>
```

**为什么**：把控制流依赖转成数据流依赖，CPU 不必推测。**适用边界**：仅当分支难预测且两侧计算很小（几条指令）才划算；两侧函数大（>20 条指令）时误预测代价反而可能低于双份计算。
**来源**：perf-ninja/labs/bad_speculation/branches_to_cmov_1/；perf-book/chapters/10/10-3

### 算术替换分支（书）

**反例（慢）**：

```cpp
int8_t mapToBucket(unsigned v) {
  constexpr unsigned BucketRangeMax = 50;
  if (v < BucketRangeMax)
    return v / 10;
  return -1;
}
```

**正例（快）**（Clang-17 将除法强度削减为乘法+移位）：

```bash
mov al, -1
cmp edi, 49
ja .exit
movzx eax, dil
imul eax, eax, 205
shr eax, 11
.exit:
ret
```

**为什么**：可用一个算术公式表达的映射不需要分支；编译器通常不会自己找这种捷径，需程序员手工改写。
**来源**：perf-book/chapters/10-Optimizing-Branch-Prediction/10-2

### 多比较合并单分支 / AVX2 批量找换行（书 4x+ / lab 344µs→86.3µs 约 4x）

**反例（慢）**（每字符一比较一分支）：

```cpp
unsigned solution(const std::string &inputContents) {
  unsigned longestLine = 0;
  unsigned curLineLength = 0;
  for (auto s : inputContents) {
    curLineLength = (s == '\n') ? 0 : curLineLength + 1;
    longestLine = std::max(curLineLength, longestLine);
  }
  return longestLine;
}
```

**正例（快）**（AVX2 一次比较 32 字符，非零掩码才标量处理）：

```cpp
using v32i = __m256i;
const v32i eol = _mm256_set1_epi8('\n');
uint32_t curr_begin = 0;
auto* ptr = inputContents.data();

for (; pos + 32 < len; pos += 32) {
  v32i v = _mm256_loadu_si256(reinterpret_cast<const v32i*>(ptr));
  v32i v_mask = _mm256_cmpeq_epi8(v, eol);
  uint32_t mask = _mm256_movemask_epi8(v_mask);
  while (mask) {
    uint32_t chars = _tzcnt_u32(mask); // C++20: std::countr_zero(mask)
    uint32_t curr_len = (pos - curr_begin) + chars;
    if (pos < curr_begin) {
      // 一个 chunk 内多个 '\n'：行长就是相邻 '\n' 的间隔 chars
      curr_len = chars;
    }
    curr_begin += curr_len + 1;
    longestLine = std::max(longestLine, curr_len);
    chars++;
    if (chars > 31) break;
    mask >>= chars;
  }
  ptr += 32;
}
// 尾部不足 32 字符走原标量循环
```

**为什么**：`cmpeq_epi8` 比较 32 个字符 + `movemask_epi8` 压成 32-bit 位图 + `tzcnt` 定位每个换行符——分支指令减少 5–6 倍（i7-1260P 实测 >4x）。注意收益依赖输入分布，最坏情况（全是 `\n` 的零长行）标量版反而快。
**来源**：perf-ninja/labs/core_bound/compiler_intrinsics_2/；perf-book/chapters/10/10-4

### 虚调用对象分组（3.17x：589µs→186µs，误预测 21.50%→0.12%）

**反例（慢）**（A/B/C 随机乱序插入）：

```cpp
void generateObjects(InstanceArray& array) {
    std::default_random_engine generator(0);
    std::uniform_int_distribution<std::uint32_t> distribution(0, 2);

    for (std::size_t i = 0; i < N; i++) {
        int value = distribution(generator);
        if (value == 0) {
            array.push_back(std::make_unique<ClassA>());
        } else if (value == 1) {
            array.push_back(std::make_unique<ClassB>());
        } else {
            array.push_back(std::make_unique<ClassC>());
        }
    }
}

void invoke(InstanceArray& array, std::size_t& data) {
    for (const auto& item: array) {
        item->handle(data);      // 间接调用目标随机
    }
}
```

**正例（快）**（按派生类分组连续存放，`invoke` 不变）：

```cpp
void generateObjects(InstanceArray& array) {
    std::default_random_engine generator(0);
    std::uniform_int_distribution<std::uint32_t> distribution(0, 2);

    InstanceArray a, b, c;
    for (std::size_t i = 0; i < N; i++) {
        int value = distribution(generator);
        if (value == 0) {
            a.push_back(std::make_unique<ClassA>());
        } else if (value == 1) {
            b.push_back(std::make_unique<ClassB>());
        } else {
            c.push_back(std::make_unique<ClassC>());
        }
    }
    array.insert(array.end(), std::make_move_iterator(a.begin()), std::make_move_iterator(a.end()));
    array.insert(array.end(), std::make_move_iterator(b.begin()), std::make_move_iterator(b.end()));
    array.insert(array.end(), std::make_move_iterator(c.begin()), std::make_move_iterator(c.end()));
}
```

**为什么**：乱序混合的 `unique_ptr<BaseClass>` 使间接调用目标随机、预测器必然失败；按派生类分组后调用目标长期稳定（生成阶段的少量分支代价远小于虚调用 mispredict 惩罚）。更彻底方案是 CRTP/概念静态多态，但异构容器做不了。
**来源**：perf-ninja/labs/bad_speculation/virtual_call_mispredict/

<!-- 来源: external/perf-book/chapters/9-Optimizing-Computations/ -->
<!-- 来源: external/perf-book/chapters/10-Optimizing-Branch-Prediction/ -->
<!-- 来源: external/perf-book/chapters/12-Other-Tuning-Areas/12-1 -->
<!-- 来源: external/perf-ninja/labs/core_bound/ -->
<!-- 来源: external/perf-ninja/labs/bad_speculation/ -->
