# 第 31 章 SE + CU 阵列（策略 B 落地）

> 本章把"单线程串行 + 固定轮转"升级为真实硬件并行：SE 屏幕 tile 静态分区（ADR-019）、CU 阵列与波原子领取、线程本地 TLB + 全局失效、三维 workgroup 映射、双执行策略 B（硬件对齐并行）落地，以及 occupancy 资源上限（ADR-038）。`se_count × worker_threads` 全组合硬判据一致是本章（及 D 阶段）的核心专有判据。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（IEngine 握手）；ADR-006/007/019/038；`../design/execution-parallel.md` §1~§6；`../design/memory-system.md` §7（确定性）；`../design/compute-line.md` §4（三维映射）。
> 落地模块：`hw_sim/se`、`apps/scene_s11_parallel`、`apps/scene_s18_compute_basic`。

---

## 1. 目标与非目标

### 1.1 目标

1. 策略 B 落地：N 工作线程（`worker_threads`，与 `se_count` 独立配置）执行 tile/波。
2. SE 分区：屏幕按 tile 静态分给 N 个 SE（`se_count`），区内顺序固定；SE 间 tile 不重叠。
3. CU 阵列波领取：SE 内 CU 波经 ticket 锁领取（`nextTicket/servingTicket`）确定非重叠领取，固定波领取序（Q-68）。
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
   每个 tile 一个 work item      │  ticket 锁领取（固定序，Q-68）
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

### 4.2 波 ticket 锁领取（CU）

```
每 CU 一个 ticket 计数器（nextTicket[cuId] / servingTicket[cuId]）
波生成 → ticket = fetch_add(nextTicket) → 自旋等待 servingTicket == ticket
→ 领取并执行；完成后 servingTicket++（固定波领取序，Q-68）
```

- 领取序由 ticket 锁固定、可复现；该确定性是 S11 硬判据（非软观测，Q-68）。

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
| T31.4 | ② 主体 | CU 波 ticket 锁领取（固定波领取序，S11 硬判据，Q-68） |
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
| T31.15 | ③ 接入 | `scene_s11_parallel`、S18 `case-1`（2D）/`case-2`（3D 映射）接入 |
| T31.16 | ④ 验收 | A31：S11/S18 全绿 + 全组合专有判据 + 回归 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M31.1 | T31.1~T31.6 | 契约/SE 分区/CU 阵列，D31.1~D31.6 全绿 |
| M31.2 | T31.7~T31.11 | TLB/三维映射/occupancy/双策略，D31.7~D31.11 全绿 |
| M31.3 | T31.12~T31.16 | 全组合一致/接入验收，A31 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D31.1 | 接口独立编译；`ParallelConfig` 默认与 §3.1 一致 | L1 契约 |
| D31.2 | SE 分区：tile 并集=全屏、且互不重叠（脚本遍历验证） | L1 契约 |
| D31.3 | tileQueue：SE 内图元序升序合并、无丢失 | L1 契约 |
| D31.4 | 波领取：每波唯一被一个 CU 领取；无遗漏/重复（计数校验）；ticket 领取序确定（Q-68） | L1 契约 |
| D31.5 | worker_threads=2/4 下输出与 serial 一致（探针零容差） | 横切 |
| D31.6 | 多 CU：每 CU 执行波数分布正确（可观测） | L1 契约 |
| D31.7 | 线程本地 TLB 失效 epoch：失效后无脏读（构造"失效后读旧 PA"负例）| L1 契约 |
| D31.8 | 三维 work-id：`(1,2,3)` 工件 = 线性序正确；lane 序固定 | L1 契约 |
| D31.9 | 越界 work-id 不执行不写；grid 边界部分 workgroup 数据完整 | L1 契约 |
| D31.10 | occupancy：超上限波进等待队列、不报错；释放后完成 | L1 契约 |
| D31.11 | 策略 A/B 探针零容差一致（S11） | 横切 |
| D31.12 | `se×threads` 全 12 组合每组合探针一致，且 ticket 波领取序逐项一致（S11 硬判据，Q-68） | 横切 |
| D31.13 | 每线程负载统计输出（软观测） | L1 契约 |
| D31.14 | `exec_mode=parallel` 不再报"未实现"，正常执行 | L1 契约 |
| D31.15 | S11/S18 出图与统计均在并行下正确 | L2 + A31 |
| D31.16 | A31 全绿：全并行组合逐字节一致（se 1/2/4/8 × threads 1/2/4）、线程本地 TLB + 全局失效无脏读、三维 workgroup 映射/边界部分/越界确定 + 回归 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A31）

**绑定场景：S11 `scene_s11_parallel`（并行一致性）、S18 `scene_s18_compute_basic`（`case-1` 2D / `case-2` 3D workgroup 映射）。**

| 项 | 内容 |
|---|---|
| 输入 | S11：`se_count ∈ {1,2,4,8}` × `worker_threads ∈ {1,2,4}`；构造多 tile 图形；S18 `case-1`：2D block (8,8,1)；S18 `case-2`：3D block (2,2,2) |
| 产物 | 每组合探针、tile 分布、线程负载、occupancy 等待计数 |
| 主判据 | 全部组合探针零容差一致（与策略 A/基线）；S11 出图 = ref_model |
| 本章专有判据 | ① 全并行组合逐字节一致（se 1/2/4/8 × worker 1/2/4）；② 线程本地 TLB + 全局失效无脏读；③ 三维 workgroup 映射/grid 边界部分/越界 work-item 确定；④ ticket 锁波领取序确定（S11 硬判据，Q-68） |
| 回归项 | A1~A30 全绿；D 阶段门禁（S10/S11/S18 全绿 + 并行确定性）达成 |
| 后续补齐 | 多引擎/队列在 32/33；故障注入（36）在并行下也可定位 |

- `tile_w/tile_h` 不进入 `se_count × worker_threads` 全组合矩阵；单独以 `8×8` 对默认 `4×4` 做一次探针逐项一致性检查（分块只影响调度与统计，不改变像素结果）。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | tile 分区表、ticket 锁波领取序、work-id 线性序固定；并行不引入随机 |
| 时间无关 | 无墙钟；epoch 为逻辑计数 |
| 可观测 | 每线程负载、tile 数、occupancy 等待、`--trace=se` |
| 契约影响 | `ParallelConfig`、三维映射、occupancy 上限冻结 |
| 简化登记 | 串行 → 硬件对齐并行（策略 B）+ SE/CU + 线程本地 TLB |
| 计算兼容 | 波领取/三维映射图形与计算共用（S18） |
| 配置与回退 | `exec_mode/worker_threads/se_count/simd_per_cu` 为 §5.1 配置键；`tile_w/tile_h` 同；occupancy 上限为 `ParallelConfig` 字段（B-036~B-041），CI 固定 se=1 |
| 调试钩子 | `--trace=se`（tile/波领取行）；`--trace=tlb`（失效/命中） |
| trace 契约 | 不进入判据 |
| 契约变更 | 并行确定性规则、三维映射、occupancy 上限为冻结项；变更走 ADR 回归 S11/S18 |
| 校验归属 | L1（结构/计数）+ L2（S11 出图）+ 横切（全组合差异） |
| 双策略 | 本章实现策略 B 并证明 A/B；此后两策略作为标准 CI 矩阵维度 |
| 阶段衔接 | D 阶段收口（S10/S11/S18 门禁）；为 32（多队列栅栏）、34（查询）提供并行语义 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 执行 | 策略 A 串行 → 策略 B 并行落地 | — |
| 分区 | 固定轮转 → SE tile 静态分区 + CU ticket 锁领取 | — |
| 内存 | 单一 TLB → 线程本地 TLB + 全局失效 | — |
| 三维 | 1D → 三维 workgroup 映射 | — |
| occupancy | 无上限 → 资源上限 + 等待队列 | — |

---

## 10. 待定项

- `se_id = tileIndex % seCount` 的取模分区是简化静态方案；若 S11 显示某 SE 空载不均衡，可考虑行色带/哈希，但**不改确定性**（只需保证分区表固定），登记做软观测优化项。
- ticket 锁固定波领取序：每 CU 的 `nextTicket/servingTicket` 严格单调，跨线程领取顺序可复现；该确定性作为 S11 硬判据（Q-68），不再以"领取结果集合唯一"替代顺序判据。
- occupancy 上限的默认 wave 数（16）与 LDS 容量耦合（第 30 章），若 S18 大 block 触发等待，登记调整默认值但不改变语义。

---

## 2026-11-21 评审同步说明

本章的 ticket/servingTicket 包围 CU 执行条款被 `review-sync-20261121.md` §2.7 替代。并行 DoD 应证明 CU/workgroup slot 可并发推进、barrier 可达、race-free 结果稳定；不可只证明静态分配顺序。
