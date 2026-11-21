# 第 27 章 MSAA + 可见性

> 项目范围、阶段位置和交付以[任务规划](../CPU模拟GPU教学项目_任务规划与实施规格.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章在 OM（26）之上补两块：多采样（MSAA：子样本覆盖、每样本深度/颜色、resolve）与可见性单元（early-Z、Hi-Z、fast clear）。early-Z 的"早丢弃"在本章真正生效，且必须与 OM 的"写 UAV 禁用 early-Z"标记配合。MSAA 执行频率分层（PS 每像素、深度/混合每样本）是本章专有判据。
> 与本章相关的设计说明：`../design/isa-abi.md` §3（PS 输出区）；第 25 章（quad 语义）；第 26 章（OM 状态）；`../design/execution-parallel.md`（occupancy 相关）；第 20 章（UAV）。
> 落地模块：`hw_sim/raster/msaa`、`hw_sim/om`（每样本扩展）、`hw_sim/visibility/`、`apps/scene_s9_msaa`、`apps/scene_s7_occlusion`。

---

## 1. 目标与非目标

### 1.1 目标

1. MSAA：`sampleCount ∈ {1,2,4,8}`；子样本坐标表固定；覆盖采样（像素中心 ±（每样本 offset））→ coverage mask。
2. 每样本操作：深度/模板测试与颜色混合每样本独立；PS 默认每像素执行（仅 `perSample` 插值时每样本）。
3. resolve：多样本 RT → 单样本 RT（平均/多数裁决；本期对颜色取平均、深度取最近）。
4. early-Z：光栅前或后（光栅后执行前）做深度粗测：`earlyZEnabled` 且无 UAV 写 → 提前拒绝，减少 PS 执行（优化）。
5. Hi-Z：深度层次（mip 层级测）粗测拒绝；`hiZ` 计数统计。
6. fast clear：清屏通过元数据（复杂度 O(1)）而非逐像素；clear color 元数据在 resolve 时扩展。
7. 可见性：`early-Z 不改结果`（正确性判据：early-Z 开 vs 关在探针像素集一致；全图差分仅告警）；`fast clear` 元数据与 `Hi-Z 拒绝计数可输出`。

### 1.2 非目标

- 不做 conservative raster / center-based expansion（范围外）。
- 不做 resolve 的硬件专用滤波器优化（平均/最近即可）。
- 不做深度压缩格式（Hi-Z 只做粗测拒绝统计，不压缩深度缓冲本体）。

---

## 2. 单元框图

```
 光栅（25）── coverage（子样本）→ MSAA 状态
    │
    ▼
visible（27.2）：early-Z 粗测 → Hi-Z 略测
    │ 拒绝（不计 PS）
    ▼
 PS 执行（每像素）→ OM（每样本深度/混合）
    │
    ▼
 resolve：样本RT → 单样本 RT
 fast clear：元数据路径（O(1)）→ 数据懒展开
```

- early-Z 位于 PS 之前，需光栅器提供覆盖样本与深度候选；fast clear 元数据独立于渲染路径。

---

## 3. 输入输出契约

### 3.1 MSAA 配置

```cpp
struct MsaaState {
    u32 samples;         // 1/2/4/8
    SampleLayout layout; // 固定子样本偏移表（每样本坐标）
    bool perSample;      // PS 是否每样本执行（仅 per-sample 插值时）
};
```

- 子样本偏移：像素 (0,0) 内 `{(0.25,0.25),(0.75,0.75),(0.25,0.75),(0.75,0.25),...}`（8x 复用 4x+更细）。固定表供 `ref_model` 对照。

### 3.2 执行频率分层（专有判据，`../design/execution-parallel.md`）

| 单元 | 执行粒度 |
|---|---|
| depth/stencil test | 每样本 |
| color blend | 每样本 |
| PS | 每像素（默认）；`perSample` 时每样本 |
| resolve | RT 层级 |

- `predicate`：`psPerSample` 场景 vs 默认。

### 3.3 可见性单元接口（`hw_sim/visibility/`）

```cpp
enum EarlyZResult { Clear, Unknown, Rejected };
struct EarlyZQuery { Vec2 uv; float z; Coverage cov; bool uavWritten; };
EarlyZResult TestEarlyZ(const EarlyZQuery&);   // clear/unknown 通过，rejected 拒绝
struct HiZStats { u32 rejectedPrims; u32 acceptedPrims; };
// fast clear：
struct FastClearMeta { bool isClear; Color clearColor; u32 version; };
```

---

## 4. 内部数据流

### 4.1 MSAA 管线

```
光栅给 coverage mask（每子样本）→ 判定 0 覆盖 → 丢弃
earlyZ：
  if earlyZEnabled && !uavWritten:
    for sample in mask: depth test(z_i vs existing[i]) → 全败 → Rejected（跳过 PS）
  else → 未知，走 PS
PS 每像素一次（多样本只 1 次），插值用像素中心（perSample 例外每样本一次）
OM 每样本：blend/depth/stencil 按样本
```

### 4.2 resolve

```
resolve(sampleRT[n], outputRT):
  color: 平均样本；depth: 最近样本；stencil: 多数
  对 fast-clear 元数据：样本值由元数据展开
```

- resolve 输出写入单样本 RT（through OM 或专用路径，本期经 OM 语义）。MSAA resolve 的指定探针像素（子样本平均结果）零容差；其余浮点颜色按容差口径：覆盖 ±1/16、颜色 ±2/通道、深度 ±1 LSB；全图逐字节差分仅告警（`roadmap.md` §1.4）。

### 4.3 early-Z 正确性

- 判据："early-Z 开 vs 关在探针像素集一致"（全图差分仅告警）——early-Z 只是优化，不得改变任何可观测结果（含 UAV 写计时序在下述 note）。
- 若 PS 写 UAV（影响全局状态），early-Z 必须禁用（26 章标记）→ 否则吞掉后写。

### 4.4 fast clear / Hi-Z

- `Clear` on MSAA RT → 写元数据 `{isClear=true, color}`，version 递增；随后 first-touch resolve 物化。
- Hi-Z：维护深度粗测（每 tile 最近小 min/max），早拒绝 `rejectedPrims++`（软观测）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T27.1 | ① 契约 | `MsaaState`/样本表/`EarlyZResult`/`HiZStats`/`FastClearMeta` 落头文件 |
| T27.2 | ② 主体 | 子样本覆盖：coverage mask 生成（固定偏移表） |
| T27.3 | ② 主体 | 每样本深度/模板测试与颜色混合 |
| T27.4 | ② 主体 | PS 每像素执行 + perSample 插值时每样本（频率分层） |
| T27.5 | ② 主体 | resolve：颜色平均、深度最近、stencil 多数、元数据展开 |
| T27.6 | ② 主体 | early-Z 粗测：`Clear/Unknown/Rejected` + 与 UAV 互斥 |
| T27.7 | ② 主体 | Hi-Z 粗测拒绝统计 |
| T27.8 | ② 主体 | fast clear 元数据 + O(1) clear + version |
| T27.9 | ③ 接入 | `scene_s9_msaa`（多样本 + resolve）、`scene_s7_occlusion`（early-Z 开/关 + 计数） |
| T27.10 | ④ 验收 | A27：S7/S9 全绿 + 专有判据 + A26 回归 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D27.1 | 接口独立编译；样本表固定；`samples ∈ {1,2,4,8}` | L1 契约 |
| D27.2 | 覆盖率：4x 子样本覆盖（部分像素）与 `ref_model` ±1/16 一致 | L2 解析 |
| D27.3 | 每样本 depth：混合深度不同时 resolve 取最近；blend 每样本输出正确 | L1 + L2 |
| D27.4 | PS 执行次数：默认 δ（像素）；perSample 时 ×samples（`stats.psExecCount`） | L1 契约 |
| D27.5 | resolve 后单样本 RT 的指定探针像素 = ref_model（平均/最近），零容差；全图差分仅告警（§1.4） | L1 + L2 |
| D27.6 | early-Z 开/关在探针像素集逐字节一致（含 UAV 场景：关时 UAV 后写仍在）；全图差分仅告警 | L1 契约 |
| D27.7 | Hi-Z 拒绝计数：构造被 early-Z/Hi-Z 拒绝的图元 → `rejectedPrims>0` | L1 契约 |
| D27.8 | fast clear：clear 后为元数据 `{isClear,color}`；首访 resolve 后像素 = color；version 递增 | L1 契约 |
| D27.9 | A27 全绿：coverage/resolve 正确；early-Z 不改结果；fast clear 元数据与 Hi-Z 拒绝计数可输出 | L1 + L2 |

> 注：D27.1~D27.9 与 T27.1~T27.9 一一对应；T27.10 为验收任务，其判据即 D27.9（A27）。

---

## 7. 场景与判据（A27）

**绑定场景：S9 `scene_s9_msaa`（MSAA + resolve）、S7 `scene_s7_occlusion`（early-Z / Hi-Z / fast clear）。**

| 项 | 内容 |
|---|---|
| 输入 | S9：4x MSAA 边缘触边三角形 + resolve；S7：early-Z 开 vs 关、UAV 深情景、fast clear + Hi-Z 图元 |
| 产物 | 覆盖掩码、resolve RT、early-Z 计数、`{isClear}` 元数据 |
| 主判据 | S7/S9：L1 契约（coverage/psExecCount/元数据/计数）+ L2 探针（覆盖 ±1/16、颜色 ±2/通道、深度 ±1、resolve 指定像素零容差）一致；全图差分仅告警（§1.4） |
| 本章专有判据 | ① coverage/resolve 正确；② early-Z 不改结果（开/关探针一致；全图差分仅告警）；③ fast clear 元数据与 Hi-Z 拒绝计数可输出 |
| 回归项 | A1~A26 全绿；S3 光栅 1x 覆盖基线不因 MSAA 引入变化 |
| 后续补齐 | 深度压缩格式（范围外）；resolve 专用硬件（本期经 OM 语义） |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 样本表固定、resolve 公式固定、early-Z 判定确定 |
| 时间无关 | 无时间输入 |
| 可观测 | `psExecCount`、`HiZ.rejectedPrims`、`--trace=msaa/visibility` |
| 契约影响 | MSAA 样本表、early-Z 语义、fast clear 元数据冻结 |
| 简化登记 | 1x → MSAA + early-Z + Hi-Z + fast clear |
| 计算兼容 | resolve/earlyZ 图形专用（明确允许） |
| 配置与回退 | `samples` 为 `MsaaState::samples` PSO 字段；`early_z`/`fast_clear` 为 `VisibilityOptions` 内部选项（默认开，非全局配置键） |
| 调试钩子 | `--trace=msaa`（覆盖/resolve 行）、`--trace=visibility`（earlyZ 判定行） |
| trace 契约 | 不进入判据 |
| 契约变更 | 样本表/earlyZ/HiZ/fast-clear 语义冻结；变更走 ADR 回归 S7/S9 |
| 校验归属 | L1（结构/覆盖率）+ L2（resolve/earlyZ 与 ref_model） |
| 双策略 | 每样本/每像素执行既并行策略也确定；两策略下计数一致 |
| 阶段衔接 | 为 28（GS 在 MSAA 之前）、31（tile 缓冲与 Hi-Z 交互）提供 coverage/earlyZ 语义 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| MSAA | 无 → 覆盖/每样本/resolve | — |
| early-Z | 语义标记 → 真早测 | — |
| Hi-Z | 无 → 粗测统计 | — |
| fast clear | 无 → 元数据路径 | — |

---

## 10. 待定项

- resolve 的深度/颜色采样时机与 fast-clear 展开是"首访物化"；若多 RT 交叉清屏，元数据 version 冲突处理需在 31 章 SE/tile 并行下复核（多线程同时物化），登记到 31 章 §10。
- Hi-Z 的精度级别（每 tile min/max 是否分层）只影响 `rejectedPrims` 软观测；正确性永远以逐样本为准。快速实现取单层，若统计无判别力再分层。
- `perSample` 插值仅当 PS 声明 `sample` 修饰符时启用；该类 shader 由编译器（17~19）标注，本章只消费标注位，编译器侧语义在 19 章后定。
