# Memory Bound 修复代码样例（反例 → 正例）

> 配合 references/optimization-playbook.md §2 使用。每条含实测收益（若有）。
> 注：perf-ninja 本地仓库 solution.cpp 为 baseline（即反例），优化正例出自各 lab README 的 Worked Solution / 上游 golden 分支。

## A. 缓存友好数据结构

### 行列遍历顺序（列优先 → 行优先）

**反例（慢）**：

```cpp
// Column-major order
for (row = 0; row < NROWS; row++)
  for (col = 0; col < NCOLS; col++)
    matrix[col][row] = row + col;
```

**正例（快）**：

```cpp
// Row-major order
for (row = 0; row < NROWS; row++)
  for (col = 0; col < NCOLS; col++)
    matrix[row][col] = row + col;
```

**为什么**：列优先每次内迭代跳过 NCOLS 个元素，预取的缓存行未用尽即被逐出；行优先按内存布局顺序访问，完整利用每个缓存行并触发硬件预取。
**来源**：perf-book/chapters/8-Optimizing-Memory-Accesses/8-2 Cache-Friendly Data Structures.md

### 字段重排消 padding + 位域压缩（lab 实测 sizeof 40B → 24B → 8B）

**反例（慢）**：

```cpp
struct S {
  int i;
  long long l;
  short s;
  double d;
  bool b;
  bool operator<(const S &s) const { return this->i < s.i; }
};
// sizeof(S) = 40：3 处 padding 共 17 字节
```

**正例（快）**——第一步仅按大小降序排列到 24 字节：

```cpp
struct S {
  long long l;
  double d;
  int i;
  short s;
  bool b;
  bool operator<(const S &s) const { return this->i < s.i; }
};
```

第二步利用取值域 + 位域压到 8 字节（i、s 最大 99，l = i*s <= 10000）：

```cpp
struct S {
  unsigned l:14;
  unsigned i:7;
  unsigned s:7;
  bool b:1;
  float d;
  bool operator<(const S &s) const { return this->i < s.i; }
};
```

书的极简对照（3 字节 → 1 字节）：

```cpp
// 反例：3 bytes
struct S { unsigned char a; unsigned char b; unsigned char c; };
// 正例：1 byte
struct S { unsigned char a:4; unsigned char b:2; unsigned char c:2; };
```

**为什么**：成员乱序导致编译器插入 padding，40 字节中 17 字节是废流量；按大小降序消除 padding 后，再用位域按实际取值范围压缩，排序搬运的字节数降为 1/5。位域代价是编译器需插入移位/掩码指令，适合访存代价大于计算代价的场合。
**来源**：perf-ninja/labs/memory_bound/data_packing/；perf-book/chapters/8/8-2

### 字段按访问阶段分组（Soldier 结构体）

**反例（慢）**：

```cpp
struct Soldier {
  2DCoords coords;   /*  8 bytes */
  unsigned attack;
  unsigned defense;
  /* other fields */ /* 64 bytes */
  unsigned speed;
  unsigned money;
  unsigned health;
};
```

**正例（快）**：

```cpp
struct Soldier {
  unsigned attack;  // 1. battle
  unsigned defense; // 1. battle
  unsigned health;  // 1. battle
  2DCoords coords;  // 2. move
  unsigned speed;   // 2. move
  // other fields
  unsigned money;   // 3. trade
};
```

**为什么**：同一阶段一起访问的字段相邻存放（battle/move/trade 各自聚簇），避免一次逻辑操作拉两条缓存行。可用 `perf mem record` + `perf annotate --data-type`（内核 6.8+）发现重排机会。
**来源**：perf-book/chapters/8-Optimizing-Memory-Accesses/8-2

### 结构体拆分（Structure Splitting）

**反例（慢）**：

```cpp
struct Point {
  int X; int Y; int Z;
  /*many other fields*/
};
std::vector<Point> points;
```

**正例（快）**：

```cpp
struct PointCoords { int X; int Y; int Z; };
struct PointInfo { /*many other fields*/ };
std::vector<PointCoords> pointCoords;
std::vector<PointInfo> pointInfos;
```

**为什么**：只需坐标时不必把 PointInfo 一并载入缓存，单条缓存行容纳更多点。
**来源**：perf-book/chapters/8-Optimizing-Memory-Accesses/8-2

### 指针内联（热字段搬进父结构）

**反例（慢）**：

```cpp
struct GraphEdge {
  unsigned int from;
  unsigned int to;
  GraphEdgeProperties* prop;
};
struct GraphEdgeProperties {
  float weight;
  std::string label;
};
```

**正例（快）**：

```cpp
struct GraphEdge {
  unsigned int from;
  unsigned int to;
  float weight;               // 高频访问，内联进来
  GraphEdgeProperties* prop;
};
struct GraphEdgeProperties {
  std::string label;
};
```

**为什么**：图算法高频访问 weight，原布局要解引用 prop（多一次访存、可能缓存未命中）；内联后省掉这次额外访问。
**来源**：perf-book/chapters/8-Optimizing-Memory-Accesses/8-2

### AoS ↔ SoA（双向转换，视访问模式定方向）

**反例（慢）**（若只遍历字段 b，则 AoS 为劣）：

```cpp
// Array of Structures (AOS)
struct S { int a; int b; int c; };
S s[N];
```

**正例（快）**：

```cpp
// Structure of Arrays (SOA)
struct S {
  int a[N];
  int b[N];
  int c[N];
};
S s;
```

**为什么**：只迭代部分字段时 SOA 访问全顺序、且是自动向量化的前提；若对全部字段做重型操作，AOS 因成员同缓存行反而带宽利用率更高。多数场景 SOA 更优。
**来源**：perf-book/chapters/8-Optimizing-Memory-Accesses/8-0

## B. 循环变换

### 循环交换（教科书模式 + lab 实测 1261ms→126ms 约 10x）

**反例（慢）**（矩阵幂 i-j-k）：

```cpp
void multiply(Matrix &result, const Matrix &a, const Matrix &b) {
  zero(result);
  for (int i = 0; i < N; i++) {
    for (int j = 0; j < N; j++) {
      for (int k = 0; k < N; k++) {
        result[i][j] += a[i][k] * b[k][j];
      }
    }
  }
}
```

**正例（快）**（i-k-j，只交换 j/k 两层）：

```cpp
void multiply(Matrix &result, const Matrix &a, const Matrix &b) {
  zero(result);
  for (int i = 0; i < N; i++) {
    for (int k = 0; k < N; k++) {
      for (int j = 0; j < N; j++) {
        result[i][j] += a[i][k] * b[k][j];
      }
    }
  }
}
```

**为什么**：原最内层按 `b[k][j]` 的 k 步进，每次跳一整行，缓存行里相邻元素全浪费；交换后 `b[k][j]` 与 `result[i][j]` 都沿行连续访问，顺带可被向量化。
**来源**：perf-ninja/labs/memory_bound/loop_interchange_1/；通用模式见 perf-book/chapters/9/9-3

### 循环交换 + 标量升维为数组（290ms→71.4ms 约 4x，Memory_Bound 49.7%→5.4%）

**反例（慢）**（高斯模糊垂直 pass，累加器是标量）：

```cpp
for (int c = 0; c < width; c++) {
  for (int r = radius; r < height - radius; r++) {
      int dot = 0;
      for (int i = 0; i < radius + 1 + radius; i++) {
          dot += input[(r - radius + i) * width + c] * kernel[i];
      }
      int value = (dot + rounding) >> shift;
      output[r * width + c] = static_cast<uint8_t>(value);
  }
}
```

**正例（快）**（`dot` 扩成数组打破迭代间依赖，c 变最内层）：

```cpp
for (int r = radius; r < height - radius; r++) {
  int dot[width];
  for (int c = 0; c < width; c++) {
    dot[c] = 0; // <- INITIALISATION
  }
  for (int i = 0; i < radius + 1 + radius; i++) {
    for (int c = 0; c < width; c++) {
      // <- LOOP OVER `c` INSIDE `i`; sequential accesses are cache-friendly
      dot[c] += input[(r - radius + i) * width + c] * kernel[i];
    }
  }
  for (int c = 0; c < width; c++) {
    int value = (dot[c] + rounding) >> shift;
    output[r * width + c] = static_cast<uint8_t>(value);
  }
}
```

**为什么**：垂直滤波时最内层 i 每步进一次就在 `input` 中跳 `width` 字节，跨行大步长大量 cache miss；把累加变量 `dot` 扩成 `int dot[width]` 打破迭代间依赖，才能把 c 挪到最内层使访问完全连续。
**来源**：perf-ninja/labs/memory_bound/loop_interchange_2/

### 循环分块 Tiling（lab 实测 16.8ms→5.72ms 约 3x）

**反例（慢）**（矩阵转置线性遍历）：

```cpp
bool solution(MatrixOfDoubles &in, MatrixOfDoubles &out) {
  int size = in.size();
  for (int i = 0; i < size; i++) {
    for (int j = 0; j < size; j++) {
      out[i][j] = in[j][i];
    }
  }
  return out[0][size - 1];
}
```

**正例（快）**（16×16 tile）：

```cpp
static constexpr int TILE_SIZE = 16;
for (int i = 0; i < size; i+= TILE_SIZE) {
  for (int j = 0; j < size; j+= TILE_SIZE) {
    for (int k = i; k < std::min(i + TILE_SIZE, size); k++) {
      for (int l = j; l < std::min(j + TILE_SIZE, size); l++) {
        out[k][l] = in[l][k];
      }
    }
  }
}
```

书的通用模式（8×8 块）：

```cpp
for (int ii = 0; ii < N; ii+=8)
  for (int jj = 0; jj < N; jj+=8)
    for (int i = ii; i < ii+8; i++)
     for (int j = jj; j < jj+8; j++)
       a[i][j] += b[j][i];
```

**为什么**：整列扫 `in` 时，加载 (0,0) 顺带取入的 (0,1)(0,2)… 在 2000 次迭代后被挤出，回来访问即 miss；分块后块内数据常驻 L1 直至复用（GEMM 标准手法）。块大小依赖具体 CPU 的 L1/L2，需实测。
**来源**：perf-ninja/labs/memory_bound/loop_tiling_1/；perf-book/chapters/9/9-3

### 循环合并 Fusion（反向为 Fission 拆分）

**反例（慢）**：

```cpp
for (int i = 0; i < N; i++)
  a[i].x = b[i].x;
for (int i = 0; i < N; i++)
  a[i].y = b[i].y;
```

**正例（快）**：

```cpp
for (int i = 0; i < N; i++) {
  a[i].x = b[i].x;
  a[i].y = b[i].y;
}
```

**为什么**：若 x、y 同属一条缓存行，合并后一次加载同时喂两处；反向的拆分（fission）可降寄存器压力、改善 icache，各有适用场景。
**来源**：perf-book/chapters/9-Optimizing-Computations/9-3

## C. 软件预取

### 模式一：软件流水 + 下一迭代预取

**反例（慢）**：

```cpp
for (int i = 0; i < N; ++i) {
  size_t idx = random_distribution(generator);
  int x = arr[idx]; // cache miss
  doSomeExtensiveComputation(x);
}
```

**正例（快）**：

```cpp
size_t idx = random_distribution(generator);
for (int i = 0; i < N; ++i) {
  int x = arr[idx];
  idx = random_distribution(generator);
  // prefetch the element for the next iteration
  __builtin_prefetch(&arr[idx]);
  doSomeExtensiveComputation(x);
}
```

**为什么**：随机索引下硬件预取器失效、OOO 窗口太小，访存延迟落在关键路径；提前生成下一迭代随机数并预取，用循环内计算时间覆盖缓存未命中延迟。
**来源**：perf-book/chapters/8-Optimizing-Memory-Accesses/8-6 Memory Prefetching.md

### 模式二：多迭代前瞻（lookAhead=8，稀疏图建边）

**正例（快）**（无逐行对照，单向示例）：

```cpp
template <int lookAhead = 8>
void Graph::update(const std::vector<Edge>& edges) {
  for(int i = 0; i + lookAhead < edges.size(); i++) {
    VertexID v = edges[i].from;
    VertexID u = edges[i].to;
    this->out_neighbors[u].push_back(v);
    this->in_neighbors[v].push_back(u);

    // prefetch elements for future iterations
    VertexID v_next = edges[i + lookAhead].from;
    VertexID u_next = edges[i + lookAhead].to;
    __builtin_prefetch(this->out_neighbors.data() + v_next);
    __builtin_prefetch(this->in_neighbors.data()  + u_next);
  }
  // process the remainder of the vector `edges` ...
}
```

**为什么**：edges 顺序访问可被硬件预取，但对邻居向量的随机访问不行；预取需足够早但不能过早污染缓存，lookAhead 做成模板参数便于实验调优。
**来源**：perf-book/chapters/8-Optimizing-Memory-Accesses/8-6

### lab 完整版：哈希表随机查找提前 16 步预取（96.2ms→43.5ms 约 2.2x）

**反例（慢）**：

```cpp
int solution(const hash_map_t *hash_map, const std::vector<int> &lookups) {
  int result = 0;
  for (int val : lookups) {
    if (hash_map->find(val))
      result += getSumOfDigits(val);
  }
  return result;
}
// find(): 随机访问 ~32MB 向量 → m_vector[val % N_Buckets]
```

**正例（快）**：

```cpp
// hash_map 中新增：
void prefetch_find(int val) const {
    int bucket = val % N_Buckets;
    __builtin_prefetch(&m_vector[bucket]);
}

static constexpr auto prefetch_step = 16;
int solution(const hash_map_t *hash_map, const std::vector<int> &lookups) {
  int result = 0;
  const auto size = lookups.size();

  if (size <= prefetch_step) {
    for (std::size_t i = 0; i < size; i++) {
      if (const int val = lookups[i]; hash_map->find(val)) {
        result += getSumOfDigits(val);
      }
    }
    return result;
  }

  for (auto i = 0; i + prefetch_step < size; i++) {
    if (const int val = lookups[i]; hash_map->find(val)) {
      result += getSumOfDigits(val);
    }
    hash_map->prefetch_find(lookups[i + prefetch_step]);
  }

  for (auto i = size - prefetch_step; i < size; i++) {
    if (const int val = lookups[i]; hash_map->find(val)) {
      result += getSumOfDigits(val);
    }
  }

  return result;
}
```

**为什么**：访问模式随机，硬件预取器无能为力，每次 find 都是一记 DRAM 停顿；提前 16 步（一个缓存行宽的 int 数）把未来桶拉进缓存，取数与计算重叠（Memory_Bound 37.7%→24.0%）。
**来源**：perf-ninja/labs/memory_bound/swmem_prefetch_1/

## D. 大页（DTLB miss）

### 显式大页 EHP（mmap MAP_HUGETLB）

**正例（快）**（单向示例）：

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
if (ptr == MAP_FAILED)
  throw std::bad_alloc{};
// ...
munmap(ptr, size);
```

**为什么**：2MB 大页把 20MB 映射从 5120 页降到 10 页，显著减少 DTLB miss；收益上限约 30%（SPEC2006 多数 <1%，最高两项 22%/27%）。EHP 不会被换出、延迟确定，HFT 首选；需提前 `echo 128 > /proc/sys/vm/nr_hugepages` 预留。
**来源**：perf-book/chapters/8-Optimizing-Memory-Accesses/8-5 Reducing DTLB misses.md

### 透明大页 THP（madvise）

**正例（快）**（单向示例）：

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE | PROT_EXEC,
                MAP_PRIVATE | MAP_ANONYMOUS, -1 , 0);
if (ptr == MAP_FAILED)
  throw std::bad_alloc{};
madvise(ptr, size, MADV_HUGEPAGE);
// use the memory region `ptr`
munmap(ptr, size);
```

**为什么**：`MADV_HUGEPAGE` 提示内核把区域提升为 2MB 大页；也可经 jemalloc：`MALLOC_CONF="thp:always"`。THP 易用但有内核后台整理的非确定延迟。
**来源**：perf-book/chapters/8-Optimizing-Memory-Accesses/8-5

### lab 完整版：大页分配器（7.09s→6.02s，吞吐 206.5→243.3 Mi/s 约 +20%）

**反例（慢）**：

```cpp
inline auto allocateDoublesArray(size_t size) {
  double *alloc = new double[size];
  auto deleter = [](double *ptr) { delete[] ptr; };
  return std::unique_ptr<double[], decltype(deleter)>(alloc, deleter);
}
```

**正例（快）**：

```cpp
#include <sys/mman.h>
inline auto allocateDoublesArray(size_t size) {
  size_t total_array_size = size * sizeof(double);
  constexpr size_t page_2mb = 1UL << 21;
  auto pages_needed = total_array_size / page_2mb + (total_array_size % page_2mb != 0);
  size_t total_to_alloc = pages_needed * page_2mb;

  void* ptr = mmap(nullptr, total_to_alloc, PROT_READ | PROT_WRITE,
                MAP_PRIVATE | MAP_ANONYMOUS, -1 , 0);
  if (ptr == MAP_FAILED) {
    throw std::bad_alloc{};
  }
  madvise(ptr, total_to_alloc, MADV_HUGEPAGE);

  auto deleter = [total_to_alloc](double *p) { munmap(p, total_to_alloc); };
  return std::unique_ptr<double[], decltype(deleter)>(static_cast<double*>(ptr), std::move(deleter));
}
```

**为什么**：随机 gather/scatter 访问数千万 double，4KB 小页需要的映射数远超 TLB 容量（约 4000 项）；2MB 大页后单个 TLB 项覆盖面积扩大 512 倍。
**来源**：perf-ninja/labs/memory_bound/huge_pages_1/

## E. 对齐

### 矩阵对齐 + 填充哑列（512 列比 511/513 快 15–20%）

**反例（慢）**：

```cpp
// 矩阵起始地址未对齐
using Matrix = std::vector<float>;
// 行宽 = 列数，行尾不落缓存行边界
int n_columns(int N) { return N; }
```

**正例（快）**：

```cpp
// 改用缓存行对齐分配器（CACHELINE_SIZE = 64）
using Matrix = AlignedVector<float>;

// 行宽向上取整到每缓存行元素数（16 个 float），插入哑列
inline constexpr int ELEMS_PER_CACHE_LINE = CACHELINE_SIZE / sizeof(float);

int get_next_multiple(int N) {
  const auto y = ELEMS_PER_CACHE_LINE - 1;
  return N + y & ~y;
}

int n_columns(int N) {
  return get_next_multiple(N);
}
```

**为什么**：SIMD 加载未对齐且跨缓存行时拆成两次访问（split load/store，TMA 下钻 Memory Bound → L1_Bound → Split Loads，事件 `mem_inst_retired.split_loads/split_stores`）；矩阵头 alignas 到 64B、每行填充哑列使行首行尾都压在缓存行边界。对齐标准：AVX2 32B、SSE/Neon 16B、AVX-512 64B；Apple L2 行 128B。
**来源**：perf-ninja/labs/memory_bound/mem_alignment_1/

## F. 内存序违例（store-to-load 冒险）

### 多副本直方图（lab 4 副本 / 书 2 副本，实测 10%–50% 提速）

**反例（慢）**：

```cpp
std::array<uint32_t, 256> computeHistogram(const GrayscaleImage& image) {
  std::array<uint32_t, 256> hist;
  hist.fill(0);
  for (int i = 0; i < image.width * image.height; ++i)
    hist[image.data[i]]++;
  return hist;
}
```

**正例（快）**（lab 版：4 个独立直方图交错累加再合并）：

```cpp
std::array<uint32_t, 256> computeHistogram(const GrayscaleImage& image) {
  std::array<uint32_t, 256> hist1{};
  std::array<uint32_t, 256> hist2{};
  std::array<uint32_t, 256> hist3{};
  std::array<uint32_t, 256> hist4{};

  int i = 0;
  for (; i + 3 < image.width * image.height; i+=4) {
    hist1[image.data[i]]++;
    hist2[image.data[i+1]]++;
    hist3[image.data[i+2]]++;
    hist4[image.data[i+3]]++;
  }
  for (; i < image.width * image.height; i++) {
    hist1[image.data[i]]++;
  }
  for (int j = 0; j < hist1.size(); ++j) {
    hist1[j] += hist2[j] + hist3[j] + hist4[j];
  }
  return hist1;
}
```

书的极简版（2 副本，代价多 1KB 内存）：

```cpp
int i = 0;
for (; i + 1 < N; i += 2) {
  hist1[image[i+0]]++;
  hist2[image[i+1]]++;
}
for (; i < N; ++i) hist1[image[i]]++;
for (int i = 0; i < hist1.size(); ++i) hist1[i] += hist2[i];
```

**为什么**：连续同灰度像素使对同一 `hist[x]` 的读-改-写形成 store-to-load 数据冒险（内存消歧预测失败触发流水线冲刷），乱序引擎被迫串行等待前次退休；拆成多个独立直方图后增量互不依赖，末尾合并循环还被编译器自动向量化（YMM 一次加 8 个 uint32）。纯色图最坏情形快一倍。
**来源**：perf-ninja/labs/memory_bound/mem_order_violation_1/；perf-book/chapters/12/12-2 Microarchitecture-Specific Issues.md

## G. I/O 供数

### 整文件 mmap 替代逐字节 fstream 读（CRC 计算是纯 I/O 瓶颈）

**反例（慢）**：

```cpp
uint32_t solution(const char *file_name) {
  std::fstream file_stream{file_name};
  if (!file_stream.is_open())
    throw std::runtime_error{"The file could not be opened"};

  uint32_t crc = 0xff'ff'ff'ff;
  char c;
  while (true) {
    file_stream.read(&c, 1);
    if (file_stream.eof())
      break;
    update_crc32(crc, static_cast<uint8_t>(c));
  }
  crc ^= 0xff'ff'ff'ff;
  return crc;
}
```

**正例（快）**（用仓库自带 MappedFile.hpp RAII 封装）：

```cpp
#include "MappedFile.hpp"

uint32_t solution(const char *file_name) {
  MappedFile mapped_file{file_name};
  const auto contents = mapped_file.getContents();

  uint32_t crc = 0xff'ff'ff'ff;
  for (const char c : contents)
    update_crc32(crc, static_cast<uint8_t>(c));

  crc ^= 0xff'ff'ff'ff;
  return crc;
}
```

**为什么**：CRC 单字节计算极便宜（x86 有 SSE4.2 crc32 硬件指令，成本≈一次乘法），瓶颈全在供数：每次 `read(&c,1)` 都付一次流提取开销；mmap 后由缺页按页搬数据，循环只剩纯内存访问（另一路线：一次 read 大块进缓冲区再遍历）。
**来源**：perf-ninja/labs/misc/io_opt1/

<!-- 来源: external/perf-book/chapters/8-Optimizing-Memory-Accesses/ -->
<!-- 来源: external/perf-book/chapters/9-Optimizing-Computations/9-3 Loop Optimizations.md -->
<!-- 来源: external/perf-book/chapters/12-Other-Tuning-Areas/12-2 -->
<!-- 来源: external/perf-ninja/labs/memory_bound/ -->
<!-- 来源: external/perf-ninja/labs/misc/io_opt1/ -->
