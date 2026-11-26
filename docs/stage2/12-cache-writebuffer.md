# 第 12 章 缓存 + 写缓冲

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章在翻译层之上落地 L1/L2 缓存与写缓冲：L1 每 CU 16 KiB 写穿入 L2、L2 全局 256 KiB 写回，仅 L2 脏行替换才进写缓冲（按 64 B 合并），I-Cache 命中统计。`cache_enable` 开关只影响统计与路径，不影响结果；缓存一致性由屏障与 fence 维护，不依赖时间或线程顺序（`../design/memory-system.md` §4）。
> 教学重点是缓存/写缓冲在 GPU 数据通路中的位置、读写与排空、flush/invalidate 后的数据可见性，以及生产者到消费者的数据流和复用。hit/miss 与 I-Cache 计数只作软诊断，不作为主要验收目标。GS/Tessellation 的 patch/control-point 缓冲属于阶段局部暂存，不等同于 L1/L2。
> 与本章相关的设计说明：`../architecture-design.md` §3.6（内存接口，缓存置于 `Translate` 之后、存储之前）；`../design/memory-system.md` §4（L1/L2/写缓冲/I-Cache、一致性、`cache_enable`）；`../design/validation-model.md` §3（缓存统计为软观测）。
> 落地模块：`hw_sim/memory/cache`、`kmd/memory`（缓存旁路）、`apps/scene_s3_triangle`（统计探针场景）。

---

## 1. 目标与非目标

### 1.1 目标

1. L1 缓存：每 CU 16 KiB、64 B 行、写穿入 L2；`Read` 经 L1，未命中下取 L2。
2. L2 缓存：全局 256 KiB、写回；L2 行 64 B；与三堆存储交互。
3. 写缓冲：仅 L2 脏行替换入队，按 64 B 合并；耗尽触发冲刷。
4. I-Cache：指令取指缓存，只做命中统计、不影响正确性（软观测）。
5. `cache_enable=on/off` 开关：关闭只停止缓存参与，结果逐字节一致（开关前后零容差）。
6. 统计只读软观测：命中/失效/写回计数进 MMIO `0x28` 软观测段（可选诊断），进不了硬判据。
7. 一致性：屏障/屏障 fence 后全局可见；不引入时间或线程顺序依赖。

### 1.2 非目标

- 不做多级缓存一致性目录/监听协议；一致性只经屏障与 fence 显式保证。
- 不做缓存容量/策略的动态配置（容量固定）；不做 `cache_enable` 之外的策略开关。
- 不做驻留/逐出联动（第 14 章）；缓存不清行不影响逐出。
- 不做 MSAA/纹理专属缓存策略；缓存策略不按 workload 分叉。

---

## 2. 单元框图

```
 IMemorySubsystem.Read/Write
        │  cache_enable?
        ▼
┌────────────── hw_sim/memory/cache ──────────────┐
│ L1 (per-CU 16KiB, 64B 行, 写穿入 L2)                 │
│   │ hit/miss                                       │
│   ▼                                               │
│ L2 (全局 256KiB, 写回, 行64B)                       │
│   │ hit/miss / evict                               │
│   ▼                                               │
│ WriteBuffer (64B 合并队列, 冲刷到存储)              │
│ I-Cache（指令取指统计，不参与正确性）                │
└───────────────────────┬───────────────────────────┘
                        ▼
                  存储（三堆后台）
```

- 何时数据回到存储：仅 L2 脏行被替换时经写缓冲冲刷到存储；屏障 fence 请求刷新 L2/写缓冲。
- `InvalidateTlb` 不影响缓存；缓存一致性靠屏障，不靠 TLB。
- 本章 `CacheSubsystem` 提供 `Flush`/`Invalidate`（接口为 `FlushBank/Invalidate`），第 11/14 章统一调用；`IMemorySubsystem` 不新增方法（Q-44）。

---

## 3. 输入输出契约

### 3.1 缓存接口（`hw_sim/memory/cache/cache.h`）

```cpp
struct CacheConfig {
    u32 l1SizeBytes;      // 16 KiB
    u32 l1LineBytes;      // 64
    u32 l2SizeBytes;      // 256 KiB
    u32 l2LineBytes;      // 64
    u32 assocL1;          // 4 路
    u32 assocL2;          // 8 路
};

class CacheSubsystem {
public:
    Result Read(GpuVAddr va, void* dst, u64 bytes);      // 经 L1/L2
    Result Write(GpuVAddr va, const void* src, u64 bytes); // 经 L1 写穿入 L2
    Result FlushBank(GpuVAddr begin, GpuVAddr end);   // 冲刷一段（屏障）
    Result Invalidate(GpuVAddr begin, GpuVAddr end);  // 无效一段（屏障/上下文切换）
    CacheStats Stats() const;   // {l1Hits, l1Misses, l2Hits, l2Misses, l2Evictions, wbMerges, wbFlushes}
    void   SetEnabled(bool on);
};
```

- 全部操作按 64 B 行对齐划分；跨行访问拆分为多行处理。
- `SetEnabled(false)`：旁路缓存，直接读存储；`Stats` 仍返回 0（或可选冻结计数）。
- 写穿：L1 写命中更新 L1 行并写入 L2（L1 写穿入 L2）；写未命中直接写 L2（不取回 L1 行）。仅 L2 脏行被替换时才进入写缓冲。

### 3.2 I-Cache（只统计）

- 取指路径挂 I-Cache：`iHits/iMisses` 计数。
- 不提供独立指令读缓冲；I-Cache 只记录取指统计，并随 `cache_enable` 一起启停，不新增 `icache_enable` 配置键。

### 3.3 屏障集成

- 内存屏障（第 20/33 章 `BARRIER` 包）→ 调用 `FlushBank(all)` + `Invalidate(all)`。
- 本章只提供 `FlushBank/Invalidate` 能力，调度在 `BARRIER` 落地章使用。

---

## 4. 内部数据流

### 4.1 L1 读路径

```
Read(va): 行号 = va>>6；组查找（assoc=4）→ 命中？返回/写穿更新 → 返回
          未命中 → 下 L2 → L1 装载（若 L1 该组满 → LRU 替换）→ 返回
```

- L1 写穿：写命中更新 L1 行并写入 L2；写未命中直接写 L2（不取回 L1 行）。
- L1 替换不需要回写（写穿）；仅 L2 脏行替换进入写缓冲。

### 4.2 L2 写回路径

```
L2 替换脏行 → 写缓冲入队（合并）→ 未来冲刷到存储
FlushBank(begin,end) → 写缓冲中被选区间全部落存储 → wbFlushes++
```

### 4.3 写缓冲合并

- 写缓冲项 `{pa, line[64B]}`；新写覆盖同行的部分字节则合并（wbMerges++）。
- 满则冲刷最旧项；屏障则全刷。

### 4.4 一致性（显式，不依赖时间）

- 两个内核/引擎之间依赖屏障：
  `Producer.Write → Barrier(Flush+Invalidate) → Consumer.Read` 在存储层可见。
- `cache_enable=off` 时 Barrier 为空操作但结果不变（直接存储语义本就一致）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T12.1 | ① 契约 | `CacheConfig`/`CacheSubsystem`/`CacheStats`/`I-Cache` 接口与容量常量落头文件 |
| T12.2 | ② 主体 | L1：组相联查找、写穿、LRU 替换 |
| T12.3 | ② 主体 | L2：写回、脏行替换、逐出回写 |
| T12.4 | ② 主体 | 写缓冲：合并、满冲刷、屏障全刷 |
| T12.5 | ② 主体 | 跨行/跨页访问拆分；边界与未对齐字节正确 |
| T12.6 | ② 主体 | I-Cache 统计；`SetEnabled` 旁路逻辑 |
| T12.7 | ② 主体 | `FlushBank/Invalidate` + 统计复位语义 |
| T12.8 | ③ 接入 | 接入 `IMemorySubsystem`：所有 `Read/Write/读指` 挂缓存（`cache_enable` 门控）；S3 统计探针场景 |
| T12.9 | ④ 验收 | 以写回、部分写合并的字节结果、flush/invalidate 可见性和缓存开关一致收口。统计用于诊断 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M12.1 | T12.1~T12.4 | L1/L2/写缓冲主体，D12.1~D12.4 全绿 |
| M12.2 | T12.5~T12.9 | 跨行/I-Cache/flush/接入验收，A12 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D12.1 | `CacheConfig` 常量 `static_assert`（16KiB/64B/256KiB/64B）；接口独立编译 | L1 契约 |
| D12.2 | 顺序读同一 4 KiB 页内 8 条不同 64 B cache line 两轮，再访问另一页并制造容量替换；数据始终正确，统计只作诊断且应反映第二轮复用与替换失效，不以固定 hit/miss 数作通过门槛 | L1 契约 |
| D12.3 | L2 写回：写 1 KB 后 `FlushBank`，存储可读回；未 flush 前存储不变（写回延迟语义） | L1 契约 |
| D12.4 | 同一行的部分写及重叠写回后，重叠字节取最后一次写入，其他字节保持预期，合并计数作为诊断 | L1 契约 |
| D12.5 | 跨行访问（如 va=0x3F8, bytes=16）正确拆分为两行 | L1 契约 |
| D12.6 | `SetEnabled(false)` 后读/写直接存储，`Stats` 返回 0（或冻结 0）| L1 契约 |
| D12.7 | 构造底层旧值、缓存脏值和消费端旧行。FlushBank 之后底层看到新值并排空目标写缓冲，Invalidate 后消费端重读新值。同步后的结果与直通路径一致 | L1 契约 |
| D12.8 | `cache_enable=on` 与 `off` 全场景逐字节一致（S2/S3/S11 退化）；单元里所有 `Read/Write` 过缓存 | L1 契约 + 横切 |
| D12.9 | A12 全绿：S2 主判据 + 缓存/写缓冲读写、flush/invalidate 可见性边界正确；统计仅作软观测；`cache_enable` 开关前后一致 + S3/S11 退化子集 | L1 契约 + L2 |

---

## 7. 场景与判据（A12）

**绑定场景：S2 `scene_s2_clear`（清屏路径的重复访问辅助探针）、S3 `scene_s3_triangle`（缓存探针）、S11 退化子集（`se_count=1`）。**

| 项 | 内容 |
|---|---|
| 输入 | S2：清屏前后对同一缓冲的重复访问探针；S3：顶点/常量流的 L1 命中、L2 命中、写合并四种模式；S11 仅 `se_count=1` |
| 产物 | 缓存统计快照（L1/L2 hit-miss、写合并）、`cache_enable` 前后输出、屏障后可见性 |
| 主判据 | `cache_enable=on/off` 开关前后所有场景输出逐字节一致；屏障后 Producer 数据对 Consumer 可见 |
| 本章专有判据 | L1 写穿、L2 写回、部分写合并数据、flush/invalidate 可见性及开关一致。命中和合并计数为诊断信息 |
| 回归项 | R12：S3 统计只在软观测，不阻断开销；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 多引擎缓存一致性与屏障频率统计在第 32、33 章；缓存在 `parallel` 下每线程本地无需拷贝（只共享存储） |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 缓存命中/替换顺序按访问序决定；LRU 序确定；写缓冲合并顺序确定 |
| 时间无关 | 一致性只由屏障决定，不引入时间；`FlushBank` 按地址段不是按时间 |
| 可观测 | `CacheStats` 软观测；`--trace=cache` 单行打印命中/失效/冲刷 |
| 契约影响 | `CacheSubsystem` 接口冻结；`IMemorySubsystem` 不新增缓存方法（缓存挂在内部） |
| 简化登记 | 无缓存 → L1/L2 + 写缓冲；I-Cache 统计外加 |
| 计算兼容 | 缓存策略不按 workload 分叉；CS 与 PS 共用同一 L1/L2（统一内存域） |
| 配置与回退 | `cache_enable` 门控；I-Cache 统计由 `CacheConfig.icacheStats` 选项控制（内部选项，非配置键） |
| 调试钩子 | `--trace=cache`、`--dump-cache-stats`；仅供人工 |
| trace 契约 | 不进入判据 |
| 契约变更 | 缓存容量/行宽/写策略（写穿 vs 写回）为冻结项；变更需 ADR 回归 S2/S3/S11 |
| 校验归属 | L1 结构断言 + L2 解析（刷新后可见性由存储比对）；统计属软观测 |
| 双策略 | 存储共享；L2 命中差异属软观测，探针点输出零容差（S11 退化子集先行） |
| 阶段衔接 | 为第 14 章驻留的"逐出前 flush"、第 32 章屏障落地提供 `FlushBank/Invalidate` 基础 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 缓存 | 无 → L1(16K 写穿) + L2(256K 写回) | —（定案；容量可配置） |
| 写缓冲 | 64 B 合并 + 满冲刷 | — |
| I-Cache | 命中统计（软观测） | —（一致性不依赖） |
| 一致粒度 | 屏障全段 `FlushBank/Invalidate` | 32/33（BARRIER 调度） |

---

## 10. 待定项

- `fetch-on-write`（写未命中是否先取回 L1 行）本章取否（直接写 L2，不取回 L1 行）。若后续场景出现"写后立即读同地址"性能异常，再评估开启；不改正确性。
- L2 替换算法固定 LRU；若未来要测"时钟/伪 LRU"等替代，仅影响软观测统计不改变输出，作为观测项延后。
- 缓存行与 4 KiB 页边界不对齐时的 `FlushBank` 处理：本章按行对齐覆盖，跨页部分显式纳入——若遇页错误交互（翻转页面对脏行的影响）在第 14 章定案。
