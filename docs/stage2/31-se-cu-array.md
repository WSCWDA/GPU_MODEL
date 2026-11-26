# 第 31 章 SE + CU 阵列（策略 B 落地）

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把"单线程串行 + 固定轮转"升级为真实硬件并行：SE 屏幕 tile 静态分区（ADR-019）、CU 阵列与波原子领取、线程本地 TLB + 全局失效、三维 workgroup 映射、双执行策略 B（硬件对齐并行）落地，以及 occupancy 资源上限（ADR-038）。`se_count × worker_threads` 全组合硬判据一致是本章（及 D 阶段）的核心专有判据。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（IEngine 握手）；ADR-006/007/019/038；`../design/execution-parallel.md` §1~§6；`../design/memory-system.md` §7（确定性）；`../design/compute-line.md` §4（三维映射）。
> 落地模块：`hw_sim/se`、`apps/scene_s11_parallel`；第 31 章只提供可复用的 workgroup 映射单元，不承担 S18 场景主验收。

---

## 1. 目标与非目标

### 1.1 目标

1. 策略 B 落地：N 工作线程（`worker_threads`，与 `se_count` 独立配置）执行 tile/波。
2. SE 分区：屏幕按 tile 静态分给 N 个 SE（`se_count`），区内顺序固定；SE 间 tile 不重叠。
3. CU 阵列波领取：多个 worker 通过原子 slot claim 并发领取 wave/workgroup；任务身份和 CU 映射固定，不冻结实际调度顺序。
4. 线程本地 TLB + 全局失效：每线程工作线程本地 TLB，`InvalidateTlb` 全局广播（无脏读）。
5. 三维 workgroup 映射：`DISPATCH(grid(x,y,z), block(x,y,z))` → work-id 线性化顺序固定；grid 边界部分 workgroup、越界 work-item 确定。
6. occupancy 上限（ADR-038）：CU 驻留 wave/SGPR/VGPR/LDS 总容量固定且可配置；超限排队等待释放而非报错。
7. 双策略对照：`serial`/`parallel` 在探针零容差一致；`se_count×worker_threads` 全组合 `{1,2,4,8}×{1,2,4}` 一致。
8. 31 章前策略 B 必须报"未实现"；本实现后才可 `parallel`。

### 1.2 非目标

- 不做真实硬件时钟/能耗建模（只功能级并行）。
- 不做 cache 亲和/线程间通信优化（原子领取已足够确定性）。
- 不做 SE 间动态负载均衡（静态 tile 分区）。
- 不做细粒度页错误恢复（记录 + 中止提交，ADR-031）；线程本地 TLB 只影响并行一致性。

---

## 2. 单元框图

```
 提交（DRAW/DISPATCH）→ SE 分区器 → N 个 SE
   │                              │
   ▼                              ▼
 tile 池（静态分区）          CU 阵列（每 SE 多 CU × 每 CU 多 SIMD）
原子 slot claim（不冻结领取/完成顺序）
   （图元 → 指定 SE tile）       ▼
                                CU 上 SIMD 波（threads 池）
                                   │  线程本地 TLB（读多读少不脏）
                                   ▼
                              输出写回（tile 顺序固定）
```

- 工作线程池：每个 `worker_threads` 绑一个"调度循环"；SE 数与线程数独立（`se_count` 分区，`worker_threads` 谁执行）。
- tile 静态分配表：`se_id = (tileIndex % se_count)`（或按行色带，本期取模，固定表）。

---

## 3. 输入输出契约

### 3.1 并行配置

```cpp
struct ParallelConfig {
    u32 seCount;          // 1/2/4/8
    u32 workerThreads;    // 1..N 实际线程数
    u32 cuPerSe;          // 每 SE CU 数（如 4）
    u32 simdPerCu;        // 每 CU SIMD（如 4）
    u32 maxWavesPerCu;    // occupancy：驻留 wave（默认 16）
    u32 maxSgprPerCu, maxVgprPerCu, maxLdsPerCu;  // occupancy 上限
    u32 tileW, tileH;     // tile 尺寸
};
```

- `se_count` 默认 `auto = min(hardware_concurrency,4)`；CI 固定 1。
- `tileW` / `tileH` 绑定配置键 `tile_w` / `tile_h`（`../roadmap.md` §5.1）：默认 4×4，取值 1..`rt_width` / `rt_height`；只决定 SE 分块粒度与统计，不改变像素/探针结果。第 25 章的 2×2 quad 是固定的导数执行粒度，不受此配置影响。
- 屏幕边缘按部分 tile 处理：`tilesX = ceil(rt_width/tileW)`、`tilesY = ceil(rt_height/tileH)`；任意取值都必须满足 D31.2（并集=全屏、互不重叠）。
- 配置键 `simd_per_cu ∈ {1,2,4}`（对应 `simdPerCu`）：只影响波调度分布与软观测，不改变硬判据（`roadmap.md` §1.4：软观测不升级为阻断）。
- 资源可满足但被占 → 进入等待队列（ADR-038），不报错；需求本身超上限（grid > B-038、参数 > B-041）→ 记录期 `InvalidArgument` 拒绝，不排队（D-13/M-06）。

**硬件上限六件套（B-036~B-041，`../design/bit-budget.md` §7.1）**：

| 常量 | 值 | 语义 |
|---|---|---|
| B-036 | LDS 64 KiB/CU | 每 CU shared memory 容量（组驻留占用） |
| B-037 | 16 waves/CU | `maxWavesPerCu` 默认值 |
| B-038 | `kMaxWorkgroups = 2^20` | 单次 DISPATCH grid 组数上限 |
| B-039 | FIFO 8 项 | 引擎请求 FIFO 深度 |
| B-040 | 描述符堆 64K 项 | Bindless 堆容量 |
| B-041 | `kMaxParamBufBytes = 4 KiB` | kernel 参数区独立上限（与 VGPR uniform 分区解耦） |

### 3.2 双执行策略入口

```cpp
// exec_mode = serial → 策略 A（单线程，固定轮转，黄金基线）
// exec_mode = parallel → 策略 B（worker 线程池 + SE 分区 + 原子领取）
Result RunDispatch(DispatchDesc, const ParallelConfig&);
```

- 策略 A 与 B 共用 `IShaderArray` 语义；输出只差调度细节、探针相同。

### 3.3 线程本地 TLB

```cpp
class ThreadLocalTlb {  // 每工作线程
    // 本地 TLB 条目（va→pa）
    LocalTlbEntry entry[64];
    u64  globalEpoch;   // 全球失效纪元（IncrementGlobalEpoch 全局 +1）
    bool IsValid(va) const { return entry.epoch == globalEpoch; }
};
// InvalidateTlb() == ++globalEpoch（一次全局广播，本地 TLB 顺势失效）
```

- 全局失效 epoch：所有工作线程看到同一计数，保证读过期 PA 不可能（标记 invalid 后不命中）。

---

## 4. 内部数据流

### 4.1 tile 静态分区（SE）

```
图元在投影后入 screen 域 → tileIndex = (py/tileH)*tilesX + px/tileW
tile → se = tileIndex mod seCount
tileQueue[se] 存图元列表（区内升序合并）
工作线程从 tileQueue[se] 领取处理（每 tile 一个线程粒度）
```

- 全 SE 的 tile 并集覆盖整个屏幕、不重叠（静态分区保证）。

### 4.2 CU slot 并发领取

多个 CPU worker 从 CU 的可运行 wave/workgroup slot 队列并发领取任务。实现使用原子状态转移保证每个任务最多领取一次；不得用持有至 wave 执行完成的全 CU slot 原子领取。任务 ID、CU 归属和输出索引由静态映射确定；实际领取及完成顺序不冻结。

- 正确性断言：每个任务恰好执行一次，无遗漏、重复和越界写。
- 并行性断言：两个独立 slot 可重叠推进；barrier 参与 wave 能交错运行并最终到齐。
- 确定性断言针对 race-free workload 的结果，不要求调度顺序或原子竞争顺序逐次相同。

### 4.3 occupancy 上限

```
新波 request：检查 per-CU 已驻留 wave/SGPR/VGPR/LDS 余量
  足够 → 进 CU；不足 → 进等待队列（qm）
  等待释放（波结束）→ 重试入队
统计：驻留 wave 数、等待次数（软观测）
```

- 上限可配置默认值（`maxWavesPerCu=16`，每 CU 16 wave、每波 32 lane）；超限排队非报错。

### 4.4 三维 workgroup 映射

```
work linear id = gz*totalGridXY + gy*totalX + gx （先 x, 后 y, 后 z — 固定序）
block 内 lane 映射：lane = (wz*blockY + wy)*blockX + wx
grid 边界：work-id >= gridTotal → 越界 → 不执行（提前空转 EXIT）
```

- 映射表固定，`ref_model` 与双策略一致。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T31.1 | ① 契约 | `ParallelConfig`/ThreadLocalTlb/双策略入口/映射常量落头文件 |
| T31.2 | ② 主体 | SE 静态 tile 分区器（tile→se 固定表） |
| T31.3 | ② 主体 | tileQueue 每 SE 管理与区内容并序 |
| T31.4 | ② 主体 | CU runnable slot 并发领取（原子唯一性，不冻结调度顺序） |
| T31.5 | ② 主体 | 工作线程池调度循环（worker_threads） |
| T31.6 | ② 主体 | CU 阵列执行实例化（每 SE 多 CU × 多 SIMD 波） |
| T31.7 | ② 主体 | 线程本地 TLB + 全局失效 epoch（无脏读） |
| T31.8 | ② 主体 | work-item id 映射（三维 → 线性，lane 序固定） |
| T31.9 | ② 主体 | grid 边界部分 workgroup / 越界 work-item 处理 |
| T31.10 | ② 主体 | occupancy：wave/SGPR/VGPR/LDS 驻留上限 + 等待队列 |
| T31.11 | ② 主体 | 双策略对照：策略 A/B 在探针零容差 |
| T31.12 | ② 主体 | `se_count×worker_threads` 全组合一致（{1,2,4,8}×{1,2,4}） |
| T31.13 | ② 主体 | 并行统计（每线程负载/tile 数，软观测） |
| T31.14 | ② 主体（收口） | 31 章前 "策略 B 未实现" 行为移除、可并行开关打通 |
| T31.15 | ③ 接入 | 接入 S11，以章内独立二维/三维向量验证映射。正式 S18 输入由第 37 章验收 |
| T31.16 | ④ 验收 | A31：S11 全绿 + 三维 workgroup 映射单测 + 全组合专有判据 + R31 回归项 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M31.1 | T31.1、T31.8~T31.10 | 用串行支架验证三维映射、末波、容量账本和整组等待 |
| M31.2 | T31.2~T31.6 | 以合成任务验证 tile 分区、区内顺序、slot 领取和 worker 循环 |
| M31.3 | T31.7、T31.11~T31.16 | 先验证线程本地 TLB 失效，再开放并行策略并完成 S11 对照 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D31.1 | 接口独立编译；`ParallelConfig` 默认与 §3.1 一致 | L1 契约 |
| D31.2 | SE 分区：tile 并集=全屏、且互不重叠（脚本遍历验证） | L1 契约 |
| D31.3 | tileQueue：SE 内图元序升序合并、无丢失 | L1 契约 |
| D31.4 | 波领取：每波唯一被一个 CU 领取；无遗漏/重复（计数校验）；领取无重复/遗漏；顺序不作为硬判据 | L1 契约 |
| D31.5 | worker_threads=2/4 下输出与 serial 一致（探针零容差） | 横切 |
| D31.6 | 多 CU：CU/workgroup slot 执行波数分布正确（可观测） | L1 契约 |
| D31.7 | 线程本地 TLB 失效 epoch：失效后无脏读（构造"失效后读旧 PA"负例）| L1 契约 |
| D31.8 | 三维 work-id：`(1,2,3)` 工件 = 线性序正确；lane 序固定 | L1 契约 |
| D31.9 | 越界 work-id 不执行不写；grid 边界部分 workgroup 数据完整 | L1 契约 |
| D31.10 | occupancy：超上限波进等待队列、不报错；释放后完成 | L1 契约 |
| D31.11 | 策略 A/B 探针零容差一致（S11） | 横切 |
| D31.12 | `se×threads` 全 12 组合每组合探针一致，且 调度顺序不冻结；race-free 结果一致且无遗漏/重复 | 横切 |
| D31.13 | 每线程负载统计输出（软观测） | L1 契约 |
| D31.14 | `exec_mode=parallel` 不再报"未实现"，正常执行 | L1 契约 |
| D31.15 | S11 出图与统计在并行下正确；workgroup 映射输入向量与 ref_model 一致，不在本章验收 S18 端到端路径 | L2 + A31 |
| D31.16 | A31 全绿：全并行组合逐字节一致（se 1/2/4/8 × threads 1/2/4）、线程本地 TLB + 全局失效无脏读、三维 workgroup 映射/边界部分/越界确定 + R31 回归项 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A31）

**绑定场景：S11 `scene_s11_parallel` 及章内独立二维/三维映射单测。S18 正式端到端验收由第 37 章负责。**

| 项 | 内容 |
|---|---|
| 输入 | S11：`se_count ∈ {1,2,4,8}` × `worker_threads ∈ {1,2,4}`。局部 `map_2d` 用 block `(8,8,1)`，`map_3d` 用 block `(2,2,2)`。两者为章内向量 |
| 产物 | 每组合探针、tile 分布、线程负载、occupancy 等待计数 |
| 主判据 | 全部组合探针零容差一致（与策略 A/基线）；S11 出图 = ref_model；映射单测匹配 ref_model |
| 本章专有判据 | ① 全并行组合逐字节一致（se 1/2/4/8 × worker 1/2/4）；② 线程本地 TLB + 全局失效无脏读；③ 三维 workgroup 映射/grid 边界部分/越界 work-item 确定；④ slot 唯一领取与可并发推进 |
| 回归项 | R31：D 阶段已交付场景 S10/S11 与 C 阶段图形基线在并行执行下不退化；S18/S19 端到端门禁留在第 37 章；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 多引擎/队列在 32/33；故障注入（36）在并行下也可定位 |

- `tile_w/tile_h` 不进入 `se_count × worker_threads` 全组合矩阵；单独以 `8×8` 对默认 `4×4` 做一次探针逐项一致性检查（分块只影响调度与统计，不改变像素结果）。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | tile 分区表、静态任务映射、work-id 线性序固定；并行不引入随机 |
| 时间无关 | 无墙钟；epoch 为逻辑计数 |
| 可观测 | 每线程负载、tile 数、occupancy 等待、`--trace=se` |
| 契约影响 | `ParallelConfig`、三维映射、occupancy 上限冻结 |
| 简化登记 | 串行 → 硬件对齐并行（策略 B）+ SE/CU + 线程本地 TLB |
| 计算兼容 | 波领取/三维映射图形与计算共用（S18） |
| 配置与回退 | `exec_mode/worker_threads/se_count/simd_per_cu` 为 §5.1 配置键；`tile_w/tile_h` 同；occupancy 上限为 `ParallelConfig` 字段（B-036~B-041），CI 固定 se=1 |
| 调试钩子 | `--trace=se`（tile/波领取行）；`--trace=tlb`（失效/命中） |
| trace 契约 | 不进入判据 |
| 契约变更 | 并行确定性规则、三维映射、occupancy 上限为冻结项；变更走 ADR 回归 S11 与三维映射单测 |
| 校验归属 | L1（结构/计数）+ L2（S11 出图）+ 横切（全组合差异） |
| 双策略 | 本章实现策略 B 并证明 A/B；此后两策略作为标准 CI 矩阵维度 |
| 阶段衔接 | D 阶段收口只验 S10/S11；S18/S19 端到端由第 37 章 F 门禁验收；本章为 32（多队列栅栏）、34（查询）提供并行语义 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 执行 | 策略 A 串行 → 策略 B 并行落地 | — |
| 分区 | 固定轮转 → SE tile 静态分区 + CU runnable slot 并发领取 | — |
| 内存 | 单一 TLB → 线程本地 TLB + 全局失效 | — |
| 三维 | 1D → 三维 workgroup 映射 | — |
| occupancy | 无上限 → 资源上限 + 等待队列 | — |

---

## 10. 待定项

- `se_id = tileIndex % seCount` 的取模分区是简化静态方案；若 S11 显示某 SE 空载不均衡，可考虑行色带/哈希，但**不改确定性**（只需保证分区表固定），登记做软观测优化项。
- 原子 slot claim 保证唯一领取：每个 runnable slot 通过原子状态转移只允许一个 worker 成功领取；任务集合唯一性与 race-free 结果是硬判据，不冻结调度顺序。
- occupancy 上限的默认 wave 数（16）与 LDS 容量耦合（第 30 章），若 S18 大 block 触发等待，登记调整默认值但不改变语义。

---

## 2026-11-21 评审同步说明

全 CU ticket/servingTicket 锁包围执行的旧方案已退役。并行验收检查 CU/workgroup slot 可并发推进、barrier 可达和 race-free 结果稳定，见 [并行执行设计](../design/execution-parallel.md)。
