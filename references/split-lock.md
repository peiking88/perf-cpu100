# split lock / 总线锁：整机 CPI 突增排查与预防（perf-cpu100 深化篇）

> 症状入口见 SKILL.md"番外二"。适用：整机所有核 CPI 突增 3–4 倍、TMA Frontend 反常、或已确认 bus_lock 事件非零。
> 案例背景：阿里 AMD Turin 服务器混部环境（在线容器 + kata container 跑离线 ODPS），大促前最高优先级故障。

## 一、问题签名（与常见 CPI 升高的区别）

- 整机**所有业务 pod、所有核**的 CPI 从 1 以下正常态突增至 3 甚至 4；指令数不变 → 每条指令周期涨三四倍 → 在线容器 CPU 利用率同步涨三四倍，并进一步压制离线任务。
- 以往经验里整机 CPI 上涨一般是内存带宽/时延上涨导致；本故障 TMA 表现**截然相反**：
  - 前端取指极为异常：指令 L1I miss 极高，Inst Cache 命中大量来自 remote（其他 CCD），指令 dispatch 被堵住 → 整个 core 执行变慢；
  - 因此对 L3 cache 及内存的访问**大幅下降**（不是上升）。
- 见到"全核 CPI 突增 + Frontend 反常 + 内存访问反降"即应怀疑总线锁。

## 二、确认与定位

1. **bus lock 事件采集**（须在产生 bus lock 的上下文里执行，见 §三 PMU 上下文分离）：
   ```bash
   perf stat -e ls_locks.bus_lock        # 计数
   perf record -e ls_locks.bus_lock && perf script -F pid   # 定位到引发线程
   ```
2. **热点链判读**：top -Hp 找 100% 线程 → perf 采样时间几乎全耗在 `__lll_lock_wait_private`→`__x86_sys_futex` → pstack 采用户栈。
3. **bpftrace 抓 futex 参数**，看等待地址是否跨行/非法：
   ```bash
   bpftrace -e 'tracepoint:syscalls:sys_enter_futex /pid==12345/ {@cnt[tid,args->uaddr,args->op,args->val,args->uaddr2,ustack]=count();} interval:s:1 {print(@cnt); clear(@cnt);}'
   ```
   案例中 uaddr=0xffffffff（由 `__lll_lock_wait_private` 传入）——地址跨 64B 缓存行边界，即 split lock 实锤。
4. **元凶进程二分**：问题发生时对可疑进程逐个 `kill -SIGSTOP`，机器恢复正常即锁定；虚拟机场景先停整个 VM 确认，再串口进 VM 逐进程 SIGSTOP。
5. **现场留存**：gcore 生成 core；进程跑在独立 mount namespace 时，直接在 root namespace gdb 会因动态库路径找不到解析出错栈——先 `nsenter` 进业务的 mount namespace 并安装 glibc/python debuginfo 再解析。

## 三、平台差异（WHY AMD）

| 平台         | split lock 影响                                                                | 检测手段                                                                                                                              |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| AMD（Turin） | 直接产生 bus lock，**拖慢整机所有核**（预期内行为，微架构未做隔离）            | `perf stat -e ls_locks.bus_lock`；内核 split_lock_detect 需等 patch 合入（5.10 内核未含）                                             |
| Intel        | 微架构层面做了隔离（技术细节为商业机密），影响**仅限发生 split lock 的物理核** | `perf stat -e r102c`（EMR）；VM 内 split lock 宿主机内核日志可见：`#AC: fc_vcpu61/257319 took a split_lock trap at address: 0x40179d` |

- Intel 内核命令行 `split_lock_detect=ratelimit 500` 后，连 split lock 对单个物理核的影响也几乎消除（原因不明：该参数看似只控制探测频率，并未控制 split lock 产生频率）。
- AMD 内核检测依赖 patch：x86/cpu: Add Bus Lock Detect support for AMD（https://lwn.net/ml/all/20240806125442.1603-5-ravi.bangoria@amd.com/ ，把 Intel 的 split lock detect 抽成公共模块）；合入后可凭宿主机内核日志判定整机是否发生 split lock。
- **宿主机感知不到 VM 内 bus lock**：宿主机与虚拟机的 PMU 上下文是分开的，宿主机 perf record -e ls_locks.bus_lock 抓不到虚拟机里产生的 bus lock；Intel 上靠 #AC 日志可见，AMD 上在内核支持前完全无感——这正是该问题长期漏报的原因。

**复现验证**：malloc 一块 64 字节内存，地址 +15 使其跨缓存行，传入 `__lll_lock_wait_private`（内部 `xchgl %eax,(%rdi)` 隐含 LOCK 的原子交换）→ AMD 整机所有核 CPI 异常升高；Intel 最新机型及 Skylake 老机型不复现整机现象，仅跑该程序的物理核 CPI 升到 4 左右。

## 四、真实根因案例：分配器混用 + fork 竞态

问题本质是业务用 jemalloc 分配的内存走进了 glibc 的 free，释放过程中对损坏的 arena 指针（0xffffffff）做原子加锁，跨行 → split lock。完整传导链：

1. C++ 进程 dlopen(libpython.so) 时传入 **RTLD_DEEPBIND**（防止符号重定位，优先从当前 lib 及其下级依赖查找符号）→ 作业进程的 malloc/free 从 jemalloc 符号**变回 glibc 符号**。
2. glibc 通过 `__malloc_hook`/`__free_hook` 跳回 jemalloc（jemalloc 以 `JEMALLOC_EXPORT void (*__free_hook)(void* ptr) = je_free;` 劫持 hook）——正常路径下仍由 jemalloc 释放。
3. 业务调用了 **malloc_trim**（valloc/pvalloc/mallinfo/malloc/mallopt/malloc_info/malloc_stats/malloc_trim 任一都会触发 ptmalloc_init）→ glibc 全局变量 `__malloc_initialized` 从默认 -1 置为 1。
4. 某次 **fork** 时 `__libc_fork`→`__malloc_fork_lock_parent` 在 `__malloc_initialized>=1` 条件下把 `__free_hook` **临时改为 free_atfork**——竞态窗口出现。
5. 同时执行 Python 解释器的线程刚好 free 一段 jemalloc 申请的内存 → 走进 glibc `_int_free`，且 av 指针解析为 0xffffffff → `mutex_lock(&av->mutex)` 对 0xffffffff 原子加锁 → 跨行 → split lock → 总线锁 → 整机 CPI 飙高。

glibc 关键结构（gdb 解析 core 用）：

- `mutex` 是 `struct malloc_state` 的**第一个字段** → `av->mutex` 地址 == av 地址；
- `mem2chunk(mem) = mem - 2*SIZE_SZ`（即 mem-0x10）；
- `arena_for_chunk`：`p->size & NON_MAIN_ARENA(0x4)` 非零 → `heap_for_ptr(p)->ar_ptr`，否则 `&main_arena`；
- `HEAP_MAX_SIZE = 2 * DEFAULT_MMAP_THRESHOLD_MAX = 0x4000000`，`heap_for_ptr(ptr) = ptr & ~(HEAP_MAX_SIZE-1)`（即 ptr & 0xfffffffffc000000）——对问题 ptr 按此取整得到的 heap_info 里 ar_ptr 为 0xffffffff。

**修复与教训**：

- 止血：作业开 isolation 模式**强制全局 tcmalloc**，避免 jemalloc 与 glibc ptmalloc 混用（灰度半月后基本不再复现）；根治：避免调用 malloc_trim 等触发 ptmalloc_init 的函数，防止 __free_hook 失效。
- 教训：混用内存分配器 + dlopen(RTLD_DEEPBIND) + glibc hook 机制的组合极其脆弱，应避免类似用法。

## 五、预防：原子变量对齐编码规范（C/C++）

1. 原子变量对齐到自然边界：
   ```cpp
   alignas(64) atomic<uint64_t> counter;      // 64 字节对齐，避免跨行和伪共享
   alignas(16) atomic<__int128> big_counter;  // 128 位原子类型 16 字节对齐
   ```
2. 避免将大原子变量放在结构体中间（前面成员使其位移导致跨行）；对齐要求最高的成员放最前并显式声明：
   ```cpp
   struct Good { alignas(16) atomic<__int128> val; char a; };   // 推荐
   struct Bad  { char a; atomic<__int128> val; };               // 可能跨缓存行
   ```
3. 不需要真正 128 位原子性时，拆成两个 64 位原子：
   ```cpp
   struct PaddedCounter { alignas(64) atomic<uint64_t> low; alignas(64) atomic<uint64_t> high; };
   ```
4. 不要对未对齐指针做原子操作（malloc 不保证 16 字节对齐，老 libc 尤甚），用 aligned_alloc：
   ```cpp
   void* p = aligned_alloc(16, sizeof(atomic<__int128>));       // 正确
   new(p) atomic<__int128>;
   ```
5. static_assert 编译期检查对齐：
   ```cpp
   static_assert(alignof(atomic<__int128>) >= 16, "128-bit atomic must be 16-byte aligned");
   ```
6. 禁止在 packed 结构体中使用原子类型（紧凑布局使 8 字节原子也可能跨行）：
   ```cpp
   #pragma pack(push, 1)
   struct Packed { uint8_t flag; atomic<uint64_t> counter; };   // 错误
   #pragma pack(pop)
   ```

规则 1 的 alignas(64) 与多线程伪共享修复（见 optimization-playbook.md §6）是同一手法：既防跨行原子（split lock），也防同行多写（false sharing）。

## 参考

- 原文：https://mp.weixin.qq.com/s/4DtVUCPSz7UWQ-icIV830g
- 深入剖析 splitlocks：https://developer.volcengine.com/articles/7096405105133502471
- 规避 Split Lock 性能争抢最佳实践（阿里云）：https://help.aliyun.com/zh/ecs/user-guide/best-practices-for-avoiding-split-lock-performance-scramble
- AMD Bus Lock Detect patch：https://lwn.net/ml/all/20240806125442.1603-5-ravi.bangoria@amd.com/

<!-- 来源: https://mp.weixin.qq.com/s/4DtVUCPSz7UWQ-icIV830g -->
