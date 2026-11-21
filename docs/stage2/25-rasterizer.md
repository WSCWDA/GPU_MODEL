# 第 25 章 光栅器

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把初级"像素中心 + 左上规则"光栅升级为完整栅格化：fill rule（top-left）、2×2 quad（强制执行）、三种插值修饰符、点/线光栅规则、scissor，并在 M25.3 集成"无效 lane 仍执行的 quad 判据"。成熟的光栅规则在此定型并对接 CPU 参考模型（覆盖 ±1/16、颜色 ±2、深度 ±1，Q-60）。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（IEngine 握手）；ADR-035（流水线/背压）；第 24 章（裁剪输出图元）、第 21 章（quad 导数供纹理 LOD）。
> 落地模块：`hw_sim/raster/rasterizer`、`hw_sim/raster/interp`、`apps/scene_s3_triangle`、`apps/scene_s9_msaa`。

---

## 1. 目标与非目标

### 1.1 目标

1. fill rule：像素中心 + top-left（左上）规则，共享边不重不漏（沿用初级实现但正式冻结）。
2. 2×2 quad：光栅按 2×2 像素组扫描；quad 内任一 lane 有效 → 4 lane 全执行（即使其余 3 个不写回）；导数用相邻 lane 差值，与像素是否有效无关（quad 语义，21 章 LOD 前置）。
3. 三种插值：`perspective`（重心插值经透视校正）、`linear`（不校正）、`centroid`（覆盖率高样本插值）。
4. 点光栅：像素中心判定、尺寸（`pointSize`）可配置；线光栅：宽度 1（本期）、斜线规则（如 Bresenham/DDA 离散）。
5. scissor：像素在 scissor 外不产生（写保护）；`scissorEnabled` 开关。
6. 覆盖率：子样本覆盖与 CPU 参考 ±1/16 一致（MSAA 在 27 章，但 1x 覆盖率口径本章定）。
7. 强制执行判据：S3 边缘像素执行计数可断言。

### 1.2 非目标

- 不做 MSAA 覆盖/解析（27 章）；本章只 1x 像素中心覆盖。
- 不做 line 平滑/thin 线宽 >1（本期线宽 1）。
- 不做 multi-viewport 内的独立线性采样器等（光栅只负责像素）。
- 不做 tile 缓冲（31 章 SE 分区才引入 tile 概念）。

---

## 2. 单元框图

```
 图元列表（24 章输出，已裁剪/变换到像素域）
     │
     ▼
┌────────────── rasterizer ─────────────────┐
│ PrimSet → bbox → 2x2 quad 扫描            │
│   fill rule（top-left）：像素中心判定        │
│   activeMask（每像素 lane）                 │
│ quad 存活：任一 lane → 4 lane 全执行         │
│   只写回存活 lane、导数通 quad             │
└──────────────────────┬────────────────────┘
                       ▼
┌────────────── interp ─────────────────────┐
│ 重心系数 → 透视/线性/centroid 插值属性       │
└──────────────────────┬────────────────────┘
                       ▼
          覆写 mask + 属性 + 深度 → 送 PS（OM/26 章）
```

- quad = 2×2 像素；像素坐标 `(qx*2+i, qy*2+j)`。
- PS 执行每 quad，存活 lane 写回；`--stats.pixelWriteCount` 只统计存活 lane。

---

## 3. 输入输出契约

### 3.1 rasterizer 输入/输出

```cpp
struct RasterInput {
    ClippedPrimitive prim;   // 已到像素域（24 章）
    Viewport vp;             // 每图元视图
    u32 rtWidth, rtHeight;
    bool scissorEnabled; Scissor sc;
    InterpMode mode;         // Perspective / Linear / Centroid
    float pointSize;         // 点
};
struct RasterOutput {
    std::vector<Quad> quads;   // {x,y, activeMask[4], bary[4][3], depth[4]}
    u32 pixelWriteMaskCount;   // 存活像素
    u32 quadCount;
};
```

### 3.2 导数输出（供纹理 LOD / 21 章）

```cpp
struct QuadDeriv { // 每 quad 提供
    Vec2 dPos;     // 屏幕空间
    Vec4 dUv[4];   // 各 lane UV 导数（相邻 lane 差）
};
QuadDeriv ComputeQuadDeriv(const Quad&, const ShaderInputs&);
```

- 即使 lane 无效（不写回），quad 内 4 lane 的 UV 都执行以得导数（quad 语义，21 章 LOD 前置）。

### 3.3 插值修饰符

| 模式 | 公式要点 |
|---|---|
| perspective | `attrib = (a0*w0*B0 + a1*w1*B1 + a2*w2*B2) / (w0*B0+w1*B1+w2*B2)`（w = 1/w 透视校正） |
| linear | `attrib = a0*B0 + a1*B1 + a2*B2`（屏幕插值） |
| centroid | 用覆盖的子样本重心插值（27 章 MSAA 引入后细化） |

---

## 4. 内部数据流

### 4.1 光栅扫描（确定性）

```
for prim:
  bbox = 包围盒(像素对齐)  ∩  [0,rtSize)
  for qy in bbox.rows / 2:
    for qx in bbox.cols / 2:
      quad(4 pix)：每像素中心 (x+0.5,y+0.5)
        active = fillRule(在三角形内) && !scissorCulled
      if any(active): emit quad  +  ComputeQuadDeriv（4 lane 都算）
```

- bbox 遍历顺序（行/列升序）固定，是确定性来源。

### 4.2 顶点航运/重心

```
E(x,y) = (y1-y0)x + (x0-x1)y + (x1y0-x0y1)
inside: 每条边 E>0 或 (E==0 且边为左上边) → in
bary: B0 = E2/Etotal, B1 = E0/Etotal, B2 = E1/Etotal（规范化）
```

- top-left 判定：边向量 `(dx,dy)`，`dy<0 || (dy==0 && dx>0)` 为左上边。

### 4.3 点/线

- 点：中心 (cx,cy)，覆盖 `[cx - size/2, cx+size/2)` 像素中心；尺寸默认 1。
- 线：宽度 1，两端点到线段的像素距离 ≤ 0.5 覆盖（保留 DDA 一致性）；共享端点像素按规则归一边（避免重）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T25.1 | ① 契约 | `RasterInput/Output`、fill rule/quad/interp 枚举、scissor 落头文件 |
| T25.2 | ② 主体 | bbox 计算与 2×2 quad 扫描 |
| T25.3 | ② 主体 | fill rule（top-left）逐像素判定 + activeMask |
| T25.4 | ② 主体 | quad 存活强制：any-active → 4 lane 执行、导数计算（含无效 lane） |
| T25.5 | ② 主体 | 重心插值（perspective/linear）与属性插值输出 |
| T25.6 | ② 主体 | centroid 插值（覆盖样本重心） |
| T25.7 | ② 主体 | 点光栅与线光栅（宽度 1） |
| T25.8 | ② 主体 | scissor 裁剪（像素外不产出） |
| T25.9 | ② 主体 | 覆盖率统计（1x：覆盖子样本=像素中心）、像素写计数 |
| T25.10 | ② 主体 | 边缘 quad 判据可观测（执行计数/存活写回计数分离） |
| T25.11 | ② 主体 | 与 `ref_model` 的 raster 模块对照（覆盖率） |
| T25.12 | ② 主体 | 确定性遍历与统计：`quadCount`/写入/衍生一致 |
| T25.13 | ③ 接入 | `scene_s3_triangle`/`scene_s9_msaa` 接入（含点线、scissor、perspective 场景） |
| T25.14 | ④ 验收 | A25：S3/S9 全绿 + 专有判据 + R25 回归项 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M25.1 | T25.1~T25.4 | fill 规则/像素中心/2×2 quad，D25.1~D25.4 全绿 |
| M25.2 | T25.5~T25.9 | 插值/点线/scissor/覆盖率，D25.5~D25.9 全绿 |
| M25.3 | T25.10~T25.14 | 边缘 quad 判据/ref 对照/确定性/接入验收，A25 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D25.1 | 接口独立编译；枚举/结构与 §3 一致 | L1 契约 |
| D25.2 | 随机三角形：所有覆盖像素 = CPU 参考（±1/16）；无遗漏/多余 | L2 解析 |
| D25.3 | fill rule：像素在边上时 top-left 判定正确（构造重点共用边） | L1 契约 |
| D25.4 | quad 存活：`active[0]=1,active[1..3]=0` → 4 lane 执行、仅 lane0 写回；导数含 4 lane UV | L1 契约 |
| D25.5 | perspective 插值 vs linear：同时存在时区分正确（单图元两模式输出不同且各符公式） | L2 解析 |
| D25.6 | centroid：采样覆盖率高的子样本（27 章前仅 1x，等价像素中心） | L2 解析 |
| D25.7 | 点 (10,10,size=1) → 覆盖像素 `(10,10)`；线端点共享边像素不重复 | L1 契约 |
| D25.8 | scissor 外像素：`pixelWriteMaskCount` 为 0（该区）；scissor 内正常 | L1 契约 |
| D25.9 | 覆盖率：1x 覆盖数 = 存活像素数 | L1 契约 |
| D25.10 | 边缘 quad 判据：边缘像素的存活写回计数与执行 quad 计数分离可输出（S3 断言） | L1 契约 |
| D25.11 | 覆盖率对照：20 随机三角形 + 10 边界 pattern，`ref_model` 覆盖一致（±1/16） | L2 解析 |
| D25.12 | 两次运行 stats 一致（`quadCount` 等） | L1 契约 |
| D25.13 | S3/S9 出图 = ref_model；点/线/scissor/perspective 探针通过 | L2 + A25 |
| D25.14 | A25 全绿：共享边不重不漏 + 覆盖率 ±1/16 + scissor 外无写 + 三种插值可区分 + 点/线规则 + 边缘 quad 判据 | L1 + L2 |

---

## 7. 场景与判据（A25）

**绑定场景：S3 `scene_s3_triangle`（基本光栅）、S9 `scene_s9_msaa`（光栅覆盖基底，MSAA 在 27 章）。**

| 项 | 内容 |
|---|---|
| 输入 | S3：各种三角形 + 点/线 + scissor + perspective/linear 两图元；S9：边缘对齐三角形（覆盖探针） |
| 产物 | quad 列表、覆盖率、写回掩码、导数样例 |
| 主判据 | S3/S9：L1 契约（quad/写回掩码/覆盖率计数）+ L2 探针（角点/边缘/内部像素覆盖 ±1/16、颜色 ±2、深度 ±1）一致；全图差分仅告警（§1.4） |
| 本章专有判据 | ① 共享边不重不漏；② 覆盖率与 CPU 参考 ±1/16；③ scissor 外无写入；④ 三种插值可区分；⑤ 点/线规则正确 |
| 回归项 | R25：E2/E4（初级像素中心光栅基线）不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | MSAA 子样本覆盖与 resolver 在 27 章；SE tile 并行在 31 章复用本 quad 逻辑 |

> 容差口径（Q-60）：覆盖 ±1/16 子样本、颜色 ±2/通道、深度 ±1 LSB；**探针比对为阻断判据，全图逐字节差分仅告警（`roadmap.md` §1.4）**；所谓"像素零容差"仅适用于 1x 覆盖率与 L1 结构/计数断言，不适用于浮点滤波后的颜色。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | bbox 升序遍历、fill rule 固定、quad 分组固定 |
| 时间无关 | 无时间输入 |
| 可观测 | `pixelWriteMaskCount/quadCount`、覆盖率、`--trace=raster` |
| 契约影响 | fill rule/quad/interp/scissor 契约冻结（登 Raster 数据契约） |
| 简化登记 | 像素中心光栅 → +2×2 quad + 插值三模式 + 点线 + scissor |
| 计算兼容 | 光栅仅图形（明确允许） |
| 配置与回退 | `scissorEnabled`/`pointSize` 为 `RasterState` PSO 字段；`guardBandScale` 属 24 章 `RasterState`（均非全局配置键） |
| 调试钩子 | `--trace=raster`：quad 一行 `{x,y,activeMask,coverage}` |
| trace 契约 | 不进入判据 |
| 契约变更 | fill rule/quad/interp/scissor/点线为冻结项；变更走 ADR 回归 S3/S9 |
| 校验归属 | L1（结构/活掩码）+ L2（覆盖率与 ref_model） |
| 双策略 | quad 扫描与并行策略正交（31 章 tile 时复用）；两策略输出一致 |
| 阶段衔接 | 为 26（OM 消费精度/深度）、27（MSAA 覆盖）、31（tile/SE）提供 quad 与覆盖率基石 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 光栅 | 像素中心 → +2×2 quad 强制 + fill rule 冻结 | — |
| 插值 | 单一 w=1 → perspective/linear/centroid | — |
| 点/线 | 无 → 点尺寸/线宽1 | — |
| scissor | 无 → 裁剪 | — |
| MSAA | 无 | 27（覆盖/解析） |

---

## 10. 待定项

- quad 分组的对齐基准（0,0 起始 quad：quad (0,0) 覆盖 (0,0)~(1,1)）是否与 31 章 tile 对齐一致，若 tile 从 0.5 取整则需在 tile 化时固定；先以像素中心 0.5 为基准，31 章可复核不改正确性。
- linear 插值在深度方向的用例不多（仅当 shader 声明 linear），若 S3 未覆盖充分，补充探针即可。
- `centroid` 在 1x（27 章前）退化为像素中心；MSAA 落地后接用覆盖样本重心，需在 27 章回归本章 1x 基线。
