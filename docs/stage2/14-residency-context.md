# 第 14 章 驻留 + 上下文

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章补齐 A 阶段最后一块：GPU 内存驻留管理与多上下文切换。驻留表记录 `{资源, 页范围, 优先级}`、LRU 逐出、逐出前 flush；上下文以 ASID 隔离、可保存/恢复（寄存器/控制流栈/谓词/PC/绑定基址/SE 状态），两上下文交替提交互不污染；并为抢占/恢复（S12）预留接口。本章是 A 阶段收口章，以 S5 驻留与 S11 占用的独立输入向量验收。
> 与本章相关的设计说明：`../architecture-design.md` §3.6（`IsResident` 语义）、§3.5（KMD `context`）；`../design/memory-system.md` §5（驻留与上下文）；ADR-031（页错误记录并中止当前提交，不复位）；`../design/boot-reset.md` §2（复位路径）。
> 落地模块：`hw_sim/memory/residency`、`kmd/context`、`kmd/scheduler`（上下文选择/抢占）、`firmware/scheduler`（上下文加载）、`apps/scene_s5_model`、`apps/scene_s11_parallel`。

---

## 1. 目标与非目标

### 1.1 目标

1. 驻留表：`{资源, 页范围[begin,end), 优先级}`；访问未驻留页 → 页错误入口（记录 `{va, ctx}` 后中止当前提交、不复位，ADR-031）。
2. LRU 逐出：内存预算用尽时按最低优先级 + 最久未用逐出；逐出前必须调用第 12 章 `CacheSubsystem` 完成 flush 写回，再 `InvalidateTlb`（12 必须先于 14，Q-47）。
3. 多上下文：每上下文独立页表根 + 独立驻留视图；ASID 区分；切换保存/恢复全部执行状态（通用寄存器、控制流栈、谓词、PC、绑定基址、SE 状态、shared memory 布局）。
4. 上下文隔离：两个上下文交替提交，结果互不污染、页错误不串扰。
5. 抢占/恢复：上下文抢占（S12 前预案）——被抢上下文可完整恢复继续；此处只做恢复正确性面，真调度在第 32 章。
6. 页错误入口：未驻留记录 `{va, ctx}` 后中止当前提交（不复位）；修正映射或完成逐出后重跑一致（复位状态机归第 36 章）。

### 1.2 非目标

- 不做硬件支持的多级优先级抢占/上下文装换时间测量（第 32 章）。
- 不做按资源位图的细粒度驻留（预算按页粒度即止）。
- 不做计算 context 的流/事件对象（stage3 38~42，原 35.5）；ASID 为上下文粒度（ADR-042）。
- 不做缓存随上下文驻留的持久化；上下文切换时 `InvalidateTlb` 全清。

---

## 2. 单元框图

```
                    KMD（context 表，ASID 分配）
        ┌─────────────────────────────┴─────────────────────────────┐
        ▼                                                           ▼
  contextA(S0) 页表根A            contextB(S1) 页表根B
        │                                                           │
        ▼                                                           ▼
┌───────────────────────── hw_sim/memory/residency ─────────────────────────┐
│ ResidencyTable{资源, 页范围, 优先级}    MemoryBudget 预算                │
│ LRU 逐出（flush→引退PTE→TLB失效）     页错误入口(NotResident→记录,中止提交) │
└────────────────────────────────────┬──────────────────────────────────────┘
                                     ▼
                    IMemorySubsystem（翻译 + 缓存 + 存储）
```

- 上下文切换由 KMD scheduler 触发：`SaveContext(ctx)` / `RestoreContext(ctx)`；恢复后 `RestoreContext` 写活 `pageTableBase` 并 `InvalidateTlb`。
- 页错误记录当前上下文后中止提交（不复位）；复位状态机见 `../design/boot-reset.md` §2。

---

## 3. 输入输出契约

### 3.1 驻留接口（`hw_sim/memory/residency/residency.h`）

```cpp
struct ResidencyEntry { ResourceId resId; u64 beginVa; u64 endVa; u32 priority; };

class ResidencyManager {
public:
    Result ResidencyAdd(ResourceId, u64 va, u64 bytes, u32 priority, ContextId ctx);
    Result ResidencyEvict(ContextId ctx, u64 budgetBytesLimit);
    bool   IsResident(ContextId ctx, GpuVAddr va) const;
    u64    UsedBytes(ContextId ctx) const;
    Result OnPageFault(ContextId ctx, GpuVAddr va);   // 记录 {va,ctx} → 中止提交（不复位）
    // 上下文
    Result SaveContext(ContextId ctx);   // 汇编终态：SVG/SGPR/PC/控制流栈/谓词/绑定基址/SE 状态/shared
    Result RestoreContext(ContextId ctx, GpuVAddr pageTableBase);
};
```

- `IsResident`：`[begin,end)` 内均有一个 `ResidencyEntry` 覆盖才算驻留。
- 预算：`MemoryBudget` 由配置给出（如 32 MiB → 可反复触发逐出/页错误测试）。

### 3.2 上下文保存/恢复状态清单

| 状态 | 保存字段 |
|---|---|
| 通用寄存器 | 当前上下文 activate 的 VGPR 镜像（波暂停时） |
| 控制流栈 | `{pc, mask}[]` 栈 |
| 谓词 | 当前活跃掩码 |
| PC | 下一条指令 |
| 绑定基址 | `SET_STATE_BASE` stateId 0~7 表 |
| SE 状态 | 波到 SIMD/CU 的绑定位（第二上下文共享 CU 时分离） |
| shared memory | LDS 内容（按上下文隔离） |

- 恢复前 `InvalidateTlb`；恢复后 `RestoreContext` 更新页表基址 MMIO（`stateId=1`）。

> 上下文仅保存和恢复直接基址槽 0..7。MMIO `0x50..0x7F` 整段保留，禁止用于临时分配。新增状态需求先设计内存间接表并走 [分配校验](../design/allocation-audit.md) 与登记流程。

---

## 4. 内部数据流

### 4.1 驻留 → 访问 → 页错误

```
提交 → 访问 [va, va+n)
  → IsResident(ctx, va)? 否 → OnPageFault(记录 va+ctx → 中止提交，不复位)
  → 是 → Translate（页表在，命中）→ 缓存/存储访问
驻留表更新：提交前显式 ResidencyAdd（UMD/驱动负责），不自动补。
```

### 4.2 LRU 逐出

```
预算紧张且新 acquire 需要 → 选择候选：
  priority 最低 → 其中 LRU 最久未用（时间戳由逻辑访问序）
逐出：第 12 章 `CacheSubsystem` flush（脏行写回）→ 释放 PTE → InvalidateTlb → 从驻留表移除
```

- `UsedBytes` 与预算差值 = 可逐出上限；逐出不跨上下文（带 ctx 隔离）。
- 页错误率统计（`--stats`）供 `scene_s11_parallel`。

### 4.3 上下文切换

```
KMD 选择 ctxB 运行 → SaveContext(A)（拷贝终态到 A 的保存区）
→ RestoreContext(B)：B.pageTableBase 写入 MMIO + InvalidateTlb + 恢复寄存/PC/栈
→ B 的 pending 波可继续
验证互不污染：A 与 B 各写各的 RT，读回互不混串。
```

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T14.1 | ① 契约 | `ResidencyEntry`/`ResidencyManager`/上下文状态结构/预算常量落头文件 |
| T14.2 | ② 主体 | 驻留表增删查、`IsResident` 区间覆盖判定 |
| T14.3 | ② 主体 | 页错误入口：`OnPageFault` 记录 `{va,ctx}` 后中止当前提交（不复位） |
| T14.4 | ② 主体 | LRU 逐出：优先级 + LRU 选择、调用第 12 章 `CacheSubsystem` flush 写回、TLB 致废 |
| T14.5 | ② 主体 | 预算管理：`UsedBytes`、逐出上限、逐出不跨 ctx |
| T14.6 | ② 主体 | 上下文创建/销毁：ASID 分配、页表根持有 |
| T14.7 | ② 主体 | `SaveContext`/`RestoreContext` 状态读写：寄存器/PC/栈/谓词/绑定基址 |
| T14.8 | ② 主体 | 上下文切换：MMIO 页表基址切换 + `InvalidateTlb` 触发面 |
| T14.9 | ② 主体 | 页错误与逐出路径的确定性：同一输入重跑一致 |
| T14.10 | ② 主体 | `scene_s5_model`：超预算提交 → 页错误/逐出 → 逐出后重跑一致（不复位） |
| T14.11 | ② 主体 | 抢占恢复正确性面：`Preempt(ctxA)` → `Restore(ctxA)` 后续提交完整 |
| T14.12 | ② 主体 | 两上下文隔离自测：A/B 各自写 RT/读回，互不混串 |
| T14.13 | ② 主体 | 逐出 flush 与缓存一致性：逐出后调用第 12 章 `CacheSubsystem` flush 保证存储可见 |
| T14.14 | ③ 接入 | `scene_s5`/`scene_s11` 接入：驻留统计、两上下文提交、逐出后重跑 |
| T14.15 | ④ 验收 | A14：独立驻留/ASID/保存恢复与 flush 向量，S5 驻留和 S11 占用子例，以及阶段门禁 3 的 E0~E5 回归 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M14.1 | T14.1~T14.5 | 驻留表/页错误入口/LRU 逐出/预算，D14.1~D14.5 全绿 |
| M14.2 | T14.6~T14.10 | 上下文保存恢复/切换/确定性/S5 接入，D14.6~D14.10 全绿 |
| M14.3 | T14.11~T14.15 | 抢占恢复/双 ctx 隔离/flush 一致性/接入验收，A14 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D14.1 | 接口独立编译；`ResidencyEntry` 字段/预算默认值冻结 | L1 契约 |
| D14.2 | `IsResident(va)` 仅当区间全覆盖 true；区间部分驻留返回 false | L1 契约 |
| D14.3 | 访问未驻留页 → `OnPageFault` 记 `{va,ctx}` 并中止当前提交（不复位）；修正映射/逐出后重跑成功 | L1 契约 |
| D14.4 | 预算 1 MiB、驻留 2 MiB：逐出发生、逐出块 = 最低优先级 + LRU；逐出后 `Translate` 返回 `NotMapped` | L1 契约 |
| D14.5 | `UsedBytes` 恒 ≤ 预算；跨 ctx 不允许逐出（逐出不减他 ctx） | L1 契约 |
| D14.6 | 创建 ctxB：新 ASID、独立页表根；销毁后页表释放、引用计数归零 | L1 契约 |
| D14.7 | `SaveContext(A)` 后 `RestoreContext(A)` 状态逐字段一致（寄存器/PC/栈/谓词/绑定基址） | L1 契约 |
| D14.8 | 切换后页表基址 MMIO 回读 = B 的 root；之后访问 B 的页全命中、A 的页 `NotMapped` | L1 契约 |
| D14.9 | 相同输入两次运行驻留/逐出/页错误序列、数量完全一致 | L1 契约 |
| D14.10 | S5 驻留向量：预算不足走逐出或确定页错误，清理后重跑与首次结果一致 | L1 + A14 |
| D14.11 | `Preempt(A)→运行 B→Restore(A)→A 后续提交`：A 的 RT 结果 = 未抢占基线 | L1 契约 |
| D14.12 | A/B 各写 RT（不同区域）+ 各读回：A 结果不混入 B，B 同 | L1 契约 |
| D14.13 | 逐出后访问驻留页有脏行：第 12 章 `CacheSubsystem` flush 后存储可见；缓存无残留 | L1 契约 |
| D14.14 | 用已交付的 `Download` 读回驻留探针结果，独立输出 S11 所需的驻留占用字段并核对计数 | L1 契约 + A14 |
| D14.15 | A14：预算恢复、双 ASID 隔离、上下文保存/恢复、逐出前 flush 与后续读取可见、E0~E5。完整 S5/S11 分别归 A22/A31 | L1 契约 + L2 |

---

## 7. 场景与判据（A14）

**A14 使用独立 S5 驻留输入、独立 S11 占用输入。可用清屏或内存探针生成确定的读回结果。**

| 项 | 内容 |
|---|---|
| 输入 | 独立构造正常驻留、1 MiB 预算下请求驻留 2 MiB、两上下文按相同 VA 写不同值、恢复后再读回，占用计数使用单一上下文对照 |
| 产物 | 驻留表快照、逐出事件、页错误事件、逐出后重跑结果、双 ctx RT 读回 |
| 主判据 | 已驻留提交读回参考值，超预算报告逐出或页错误，恢复后结果一致，双 ASID 结果隔离 |
| 本章专有判据 | ① 超显存（预算<需求）时结果一致（逐出重跑 vs 直接满足预算基线）；② 两上下文互不污染；③ 抢占恢复正确；④ 逐出前经第 12 章 `CacheSubsystem` flush 完成，`Flush/Invalidate` 后可见性正确 |
| 回归项 | R14：阶段门禁 3（E0~E5 不因驻留/上下文退化）；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 真异步抢占/调度在第 32 章 S12；复位状态机收口在第 36 章 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 驻留表更新顺序、LRU 时间戳用逻辑访问序、逐出候选选择确定 |
| 时间无关 | 无墙钟；抢占由逻辑事件调度，不依赖 timer |
| 可观测 | 驻留/逐出/页错误计数、`UsedBytes`、上下文切换次数、`--trace=residency` |
| 契约影响 | `ResidencyManager`、上下文状态机、ASID 语义冻结 |
| 简化登记 | 单上下文 → 多上下文 ASID；常驻 → 驻留/逐出/页错误入口 |
| 计算兼容 | ASID 为上下文粒度（算归计算共享）；逐出不因 workload 差异改变 LRU 语义 |
| 配置与回退 | `memory_budget` 配置；页错误默认记录 + 中止提交（不复位，ADR-031） |
| 调试钩子 | `--trace=residency` 单行 `{event, va, ctx}`；`--dump-residency` |
| trace 契约 | 不进入判据 |
| 契约变更 | 驻留/逐出/页错误/上下文保存恢复为冻结项；变更走 ADR 回归 S5/S11 |
| 校验归属 | L1（结构/状态机）+ L2（超预算 exec 结果与 ref_model 解析一致） |
| 双策略 | 上下文切换在 `serial/parallel` 下行为一致；抢占恢复在并行下不得发生脏读（cache 已 flush） |
| 阶段衔接 | A 阶段收口：门禁 1（S1/S2/S13）+ 门禁 2（契约冻结）+ 门禁 3（E0~E5）全绿后进 B 阶段 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 上下文 | 单 ctx → 多 ctx + ASID + 保存/恢复 | —（定案） |
| 驻留 | 常驻 → 驻留表/LRU 逐出/页错误入口 | —（定案） |
| 抢占 | 预占正确性面（恢复完整） | 32（真异步调度） |
| 逐出一致性 | 第 12 章 flush→写回→引退→TLB 失效（12 先于 14） | — |
| 页错误恢复 | 记录 + 中止提交（不复位） | 36（复位状态机按引擎级接入） |

---

## 10. 待定项

- `Preempt` 的触发条件（何时抢占、抢占优先级）在本章只验证"可恢复"，调度策略本身留第 32 章 S12 定案。
- 逐出时的共享缓存 `Invalidate` 粒度：本章整缓存失效；若 S11 统计显示逐出频繁，可评估按段失效，仅影响软观测不改变正确性。
- 上下文切换保存的"SE 状态"在 CU 阵列落地（第 31 章）前为空；待 31 章补波到 SIMD 绑定的持久化。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A14 和适用的 R14 后提交。分配核对项为ASID、页表与保留 MMIO，新增或改动按 [编码与位段校验表](../design/allocation-audit.md)登记。

```text
feat(gpu-ch14): isolate resident resources by context

- 实现驻留预算、逐出前 flush、ASID 切换和状态恢复。
- 以独立驻留/占用向量校验，两上下文同 VA 读回结果互相隔离。

Tasks: T14.1–T14.15
Validation: <实际执行的命令、用例和结果>
```
