# 第 24 章 裁剪 + PA（图元装配）

> 项目范围、阶段位置和交付以[任务规划](../CPU模拟GPU教学项目_任务规划与实施规格.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把初级"近平面 + 视口 6 平面 Sutherland-Hodgman"扩展为完整裁剪管线：guard band 裁剪、Clip/Cull Distance（`kVgprBaseDistance`=`v[68..75]`，8 项，clip 在前、cull 在后，ADR-033/Q-57/Q-64）、多视口索引、退化丢弃，并与 CPU 参考裁剪器对照。PA 在此负责把 VS/GS 后的顶点流装配成带拓扑的图元列表交光栅，并判定背面剔除（Q-61）。
> 与本章相关的设计说明：`../design/isa-abi.md` §3（距离区 `kVgprBaseDistance`=`v[68..75]`）；ADR-033（顶点间固定距离区）；第 15（分歧）、第 22（拓扑）、第 23（切片）章为上游；第 25 章光栅为下游。
> 落地模块：`hw_sim/frontend/clip_pa`、`apps/scene_s3_triangle`、`apps/scene_s5_model`。

---

## 1. 目标与非目标

### 1.1 目标

1. NDC 裁剪：视口 6 平面 + 近/远平面；Sutherland-Hodgman 逐平面。
2. guard band 裁剪：允许超出视口但不超 guard band（`gMaxX/Y`）的顶点不切，减少裁剪开销；超过 guard band 才实际裁剪。
3. Clip/Cull Distance：`kVgprBaseDistance`（`v[68..75]`）存储至多 8 个距；`clipCount + cullCount ≤ 8`（PSO 声明），写超声明 count 报 L1 错（Q-57/Q-64）；clip 在前、cull 在后（ADR-033）；`clipDistance < 0` 的顶点被裁，`cullDistance < 0` 的图元被丢弃。
4. 多视口：`viewportIndex` 由 PS 输入或内建决定；每个视口独立 `{x,y,w,h}` 与 scissor。
5. 退化丢弃：面积 ~0、顶点被裁剪耗尽、全视口外 → 丢弃并计统计。背面剔除（`cullMode/frontFace`）在本章 PA 判定，OM（第 26 章）只保存状态、不重复判定（Q-61）。
6. 与 CPU 参考裁剪器（`tools/ref_model` 的 clip 模块）逐 case 对照（出界/跨近平面无裂缝）。

### 1.2 非目标

- 不做三角形细分/曲面（28 章）；不做多视口数组的 viewport transform 立体扩展（只对象索引）。
- 不做深度范围 clamped 特殊 NV 行为；深度范围单 `[near,far]`。
- 不做 guard band 的 "不裁剪+后光栅裁剪" 加速路径，只做判定分支。

---

## 2. 单元框图

```
 VS/DS/GS 顶点流(v[16..27]+距离 `kVgprBaseDistance`=`v[68..75]`)
     │
     ▼
PA：按拓扑装配图元列表（顶点流 → primitive 顶点元组 + 附加属性）
     │
     ▼
CLIP：NDC 6 平面 + guard band 判定
   ├─ 全部在 → 直接过
   ├─ 超 guard band → Sutherland-Hodgman 裁到视口
   └─ 全被近/远/裁剪丢弃 → 丢弃
CULL：cullDistance<0 → 丢弃
VIEWPORT：选择 viewportIndex → 变换到像素坐标（+ scissor）
     │
     ▼
光栅器（25 章）
```

- 每个出界/裁剪图元保留插值属性随交点生成（属性插值在 Pascal 坐标）。
- 背面剔除（`cullMode/frontFace`）在 PA 装配后判定；第 26 章 OM 只保存状态、不重复判定（Q-61）。

---

## 3. 输入输出契约

### 3.1 输入（来自 VS 顶点流）

```
PosAndAttrib { float4 clipPos; float4 attribs[...]; float clipDist[8]; float cullDist[8]; u32 viewportIdx; }
```

- `viewportIdx` 每顶点可不同（GS/VS 写）；最终图元取首顶点索引。
- 视口数组：`PSO.fixedState.viewportCount`（1~4），每个 `{x,y,w,h,depthMin,depthMax}`.

### 3.2 输出（光栅输入）

```cpp
struct ClippedPrimitive {
    u32  primIndex;
    Vec4 verts[3];          // 已变换到像素坐标
    float attribs[3][MAX];  // 插值属性（3 顶点）
    u32  viewportIdx;
    u32  flags;             // 是否裁剪过 / guard band 内
};
```

### 3.3 guard band 参数

- `guardBandScale`（默认 2.0）：`gLimit = viewportExtent * guardBandScale`。
- 顶点 clip 空间在 `[-gLimit,+gLimit]` 内不裁，之外才裁。

---

## 4. 内部数据流

### 4.1 裁剪流程（逐图元）

```
PA 装配出 primitive(v0,v1,v2)
对每平面（x=±1,y=±1,z=near,z=far，x=±gLimit,y=±gLimit 作 guard band 扩展）:
   Sutherland-Hodgman：
     遍历当前边形顶点，对每条边求交参数 t = (d0 / (d0-d1))
     生成新顶点 = interp(属性按 t)
   （guard band 判定：先按视口 6 平面判定；若顶点在 `±gLimit` 内且位于视口外，则本平面不裁剪、保留原顶点直送光栅；仅超出 guard band 的边做求交）
```

- 判定优先级：先做普通 NDC 6 平面；对其余顶点若在 guard band 内 → 保留。
- 退化：面积 ~0 或顶点数 < 3 → 丢弃。
- 背面剔除：PA 按 `cullMode/frontFace` 判定缠绕序，剔除的图元不进入裁剪/光栅（Q-61）；26 章只保存 cull 状态。

### 4.2 Clip/Cull Distance（ADR-033）

- 装载：VS 写 `kVgprBaseDistance`（`v[68..75]`，至多 8 项）；PSO 声明 `clipCount/cullCount`（`clipCount+cullCount≤8`）；写超声明 count → L1 错（Q-57/Q-64）。
- clip：图元任一顶点 `clipDist[i] < 0`（i<clipCount）→ 交点裁剪（边属性插值）。
- cull：图元所有顶点 `cullDist[i] < 0` → 整图元丢弃（不裁插值）。
- 未声明区保留。

### 4.3 多视口变换

```
prim 顶点 clipPos → 按 viewportIdx 命中的 viewport 做视口变换：
  x' = (x+1)/2 * w + x0 ; y' = (1-y)/2 * h + y0 ; z' = lerp(depthMin, depthMax, z)
scissor 属于光栅（25 章）；这里只做视口与深度范围。
```

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T24.1 | ① 契约 | `ClippedPrimitive`、距离区声明、`viewportCount`、guard band 参数落头文件 |
| T24.2 | ② 主体 | NDC 6 平面 Sutherland-Hodgman（含属性插值） |
| T24.3 | ② 主体 | guard band 判定分支（band内保留/out裁剪） |
| T24.4 | ② 主体 | Clip Distance 交点裁剪（`kVgprBaseDistance` + 属性插值） |
| T24.5 | ② 主体 | Cull Distance 整图元丢弃判定 |
| T24.6 | ② 主体 | 退化丢弃（~0 面积/顶点耗尽/全视口外）+ 背面剔除（`cullMode/frontFace`，PA 判定）+ 统计 |
| T24.7 | ② 主体 | 多视口：viewport 数组选择 + 视口变换 + 深度范围 |
| T24.8 | ② 主体 | 与 CPU 参考裁剪器集成（`ref_model` 的 clip 模块）逐 case 对照 |
| T24.9 | ③ 接入 | `scene_s3_triangle`/`scene_s5_model` 接入（出界/跨近平面/guard band/多视口） |
| T24.10 | ④ 验收 | A24：S3/S5 全绿 + 专有判据 + L1 clip/cull distance 单测 + A23 回归 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D24.1 | 接口独立编译；距离区规范（`kVgprBaseDistance`=`v[68..75]`、clip 前 cull 后、总数≤8、写超声明 count → L1 错）与 §3 一致 | L1 契约 |
| D24.2 | 出界/跨近平面三角形裁剪后像素集 = 参考裁剪器（探针 ±1/16） | L2 解析 |
| D24.3 | guard band：band 内顶点不产新顶点；跨 band 顶点裁剪产生交点；无裂缝（相邻图元共享边像素一致） | L1 + L2 |
| D24.4 | Clip Distance：某顶点 clipDist<0 → 沿边插值出新顶点属性（对比手工）；cull：全负 → 图元 0 像素 | L1 契约 |
| D24.5 | 退化丢弃与背面剔除：面积~0 不产出像素；图元统计 +1 但像素 0；`cullMode/frontFace` 镜像剔除结果正确（PA 判定，OM 不重复） | L1 契约 |
| D24.6 | 视口索引越界（>viewportCount-1）→ `InvalidArgument`（或 clamp，本期报错语义取拒绝，登记待定项） | L1 契约 |
| D24.7 | 视口变换：NDC→像素坐标与公式逐一相等；多视口两窗口各自正确 | L1 契约 |
| D24.8 | 取 20 个典型裁剪 case（出界/跨平面/band/退化）：`ref_model` clip 输出与本实现图元列表逐 case 一致 | L2 解析 |
| D24.9 | A24 全绿：出界/跨近平面与参考裁剪器一致 + guard band 无裂缝 + 退化丢弃 + 多视口索引校验 + Clip/Cull Distance 逐顶点逐图元正确 + 回归 | L1 + L2 |

---

## 7. 场景与判据（A24）

**绑定场景：S3 `scene_s3_triangle`、S5 `scene_s5_model`；Clip/Cull Distance 以 L1 单测（D24.4）覆盖，不并入场景输入（S6 属第 28 章）。**

| 项 | 内容 |
|---|---|
| 输入 | S3/S5：三角形出界/跨近平面的多种 case；S5：多视口两 RT 子窗；Clip/Cull Distance 负例走 L1 单测（D24.4） |
| 产物 | 裁剪后图元列表、插值属性探针、clip/cull 计数 |
| 主判据 | 裁剪后输出 = ref_model 参考（像素探针零容差） |
| 本章专有判据 | ① 出界/跨近平面与参考裁剪器一致；② guard band 无裂缝；③ 退化丢弃；④ 多视口索引校验；⑤ Clip/Cull Distance 逐顶点逐图元（`kVgprBaseDistance`=`v[68..75]`，L1 单测）；⑥ 背面剔除在 PA 判定、OM 只保存状态 |
| 回归项 | A1~A23 全绿；E2/E4（初级 6 平面裁剪基线）不退化 |
| 后续补齐 | GS 生成顶点（28）可复用本裁剪；MSAA（27）单独覆盖 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 平面处理序、交点插值序固定；guard band 阈值固定 |
| 时间无关 | 无时间输入 |
| 可观测 | clip/cull 计数、裁剪前后图元统计、`--trace=clip` |
| 契约影响 | 距离区/视口数组/guard band 契约冻结 |
| 简化登记 | 6 平面 → 完整 + guard band + clip/cull distance + 多视口 |
| 计算兼容 | 裁剪仅图形（明确允许） |
| 配置与回退 | `guardBandScale` 为 `RasterState::guardBandScale`（默认 2.0，非全局配置键）；`viewportCount` 来自 PSO.fixedState（默认 1） |
| 调试钩子 | `--trace=clip`：每平面一行 `{planesequ, inV, outV}`；`--trace=pa` 图元一行 |
| trace 契约 | 不进入判据 |
| 契约变更 | 距离区、裁剪语义、guard band、多视口为冻结项；变更走 ADR 回归 S3/S5 |
| 校验归属 | L1（结构/计数）+ L2（text 参考裁剪 diff 对照） |
| 双策略 | 裁剪与并行策略无关（独立单元）；两策略输出一致 |
| 阶段衔接 | 为 25（光栅消费图元列表）、26（OM 消费裁剪残差）、31（SE 分区在 PA 之后）提供图元边界 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 裁剪 | 6 平面 → +guard band + clip/cull distance | —（定案） |
| 视口 | 单视口 → 多视口数组 | — |
| 退化 | 面积 ~0 → +全视口外判定 | — |
| PA | 顶点流 → 图元列表（含属性插值交点） | — |

---

## 10. 待定项

- 多视口索引越界语义：真实 GPU 通常 clamp 到最后一个视口；本期报错取拒绝更利于定位。若 35.2（Vulkan 语义）要求 clamp 行为，需 ADR 记录切换。
- guard band 具体数值（`guardBandScale=2.0`）是默认值，随 ref_model 容差是否稳定微调；不改变语义只变阈值时直接参照容差表。
- 距离插值的精度（交点 t 由 f32 计算）可能造成边缘 ±1/16 浮动，先以容差容忍；若 CI 抖动评估提升到 f64 计算插值裁剪。
