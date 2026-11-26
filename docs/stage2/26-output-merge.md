# 第 26 章 输出合并（OM）

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把初级"单一深度比较 + 截断写 BGRA"扩展为完整输出合并：深度/模板测试、背面剔除、颜色混合（含双源混合与逻辑 Op）、MRT、PS 输出到 RT 的格式转换（UNORM/SNORM/UINT/SINT/sRGB）、写 UAV 时 early-Z 禁用语义，以及 LoadOp/StoreOp 载入/存储操作。
> 与本章相关的设计说明：`../architecture-design.md` §3.4（HLI 对象改）、ADR-015（BGRA 内存序）；`../design/isa-abi.md` §3（PS out `v[36..39]`/RT1~3 `v[44..55]`/深度 `v[40..43]`）；`../design/memory-system.md` §8（写走 IMemory，BGRA）。
> 落地模块：`hw_sim/om`、`apps/scene_s7_occlusion`、`apps/scene_s8_blend_mrt`、`apps/scene_s14_bindless_uav`（全阶段 UAV 写入路径）。

---

## 1. 目标与非目标

### 1.1 目标

1. 深度测试：`depthFunc ∈ {Never,Less,Lequal,Greater,Gequal,Equal,NotEqual,Always}`；写掩码开关；与模板联动。
2. 模板测试：front/back 独立 `{func, stencilWriteMask, depthFuncFail/stencilFail/stencilPass}` 三 op；逐像素。
3. 背面剔除：只保存 `cullMode ∈ {none, front, back}`、`frontFace`（cw/ccw）状态；实际判定在第 24 章 PA，OM 不重复判定（Q-61）。
4. 颜色混合：`blendEnable`、`srcFactor/dstFactor`（SRC_ALPHA/ONE_MINUS_SRC_ALPHA/...）、方程（ADD/SUB）；双源混合（blendFactor 双通道）与逻辑 Op（`AND/OR/XOR` 等）。
5. MRT：4 个 RT（`R8G8B8A8` 等），各 RT 独立 blend/writeMask；PS 输出 `v[36..39,44..55]` 映射。
6. 格式转换：PS 输出 float → 后端格式（UNORM 量化、SNORM、UINT/SINT 取整、sRGB 重编码）；写掩码。
7. 写 UAV 时 early-Z 标识：若 PS 写 UAV，则 early-Z（27 章）禁止/退回 late-Z 语义（本章标记状态，27 收实质优化）。
8. LoadOp/StoreOp：`loadOp ∈ {load, clear, dontCare}`、`storeOp ∈ {store, dontCare}`；不做 RenderPass 对象（命令/PSO 描述）。

### 1.2 非目标

- 不做 early-Z 优化本身（27 章可见性单元）；本章只定义 early-Z 与写 UAV 的互斥约束。
- 不做 MSAA resolve（27 章）；本章样本=像素（1x）。
- 不做深度 CAS 的压缩（27 章 fast clear 元数据）。
- 不做 subpass/DEPTH 并行读写（本期单一深度缓冲）。

---

## 2. 单元框图

```
 PS 输出（v[36..39] RT0 等）＋ 深度 v[40..43]
     │
     ▼
┌─────────────── om ──────────────────────────┐
│ 早期：stencil test（front/back）→ depth test │
│ 失败丢弃（不写）                              │
│ 通过 → blend（color0..3）＋ writeMask        │
│       → 格式转换→ 写 RT（VGPR[36..]→内存）    │
│ cull 状态在 OM 保存、判定在 24 章 PA（Q-61）      │
│ 写 UAV 时：标记 earlyZDisable                │
└──────────────────────┬─────────────────────┘
                       ▼
      RT 内存（BGRA/格式）+ 深度缓冲（+模板）
```

- OM 顺序：模板 → 深度 →（写入掩码+混合）→ 格式转换 → 写回；早期丢弃不写任何缓冲。

---

## 3. 输入输出契约

### 3.1 OM 状态（`hw_sim/om/om.h`）

```cpp
struct DepthState  { bool enabled; CompareFunc func; bool writeMask; };
struct StencilState{ bool enabled; CompareFunc frontFunc, backFunc;
                     u8  frontMask, backMask, ref;
                     StencilOp depthFail, stencilFail, stencilPass;
                     u8  frontOpMask, backOpMask; };
struct BlendState  { bool enabled[4]; BlendFactor src[4], dst[4]; BlendOp op[4];
                     bool logicOpEnable; LogicOp logicOp;
                     bool dualSrcEnable; /* blendFactor 双通道 */ };
struct RasterStateOpts { CullMode cull; FrontFace front; bool earlyZDisable; };
struct TargetFormat  { Format rt[4]; u32 writeMask[4]; }; // 格式转换目标
```

- `earlyZDisable`：若 PS 包含 UAV 写 → `c.Flags.earlyZ=false`（真 early-Z 优化在 27 章知情）。

### 3.2 PS 输出区映射（ABI）

| 输出 | VGPR | 用途 |
|---|---|---|
| RT0 | `v[36..39]` | 颜色 |
| 深度 | `v[40..43]` | 深度（1x 用 x） |
| RT1~3 | `v[44..55]` | 额外 RT |

- 写 mask：对每 RT 位掩；`writeMask & (1<<i)` 只写 i 通道。

### 3.3 触发条件

- 每像素（存活 lane）→ OM 一次；样本与像素在 1x 相同。
- 深度缓冲为独立存储；模板存储于深度缓冲高字节（D24S8 布局；本期 S8 部分）。

---

## 4. 内部数据流

### 4.1 每像素流程

```
lane 存活：
  stencil test：过？→ else 丢弃（不写）
  depth test：过？→ else 丢弃（不写）
  通过：
    for rt in 0..3: if writeMask[rt]:
      color = PS out[rt]
      if blend: color = mix(src factor, dst factor)
      formatConvert(color, TargetFormat[rt])  // UNORM/SNORM/UINT/SINT/sRGB
      写 RT 内存（BGRA for R8G8B8A8）
    depth write 若 writeMask → 写深度
```

- 早期丢弃与后丢弃都计数（`stats.depthTestFail/stencilFail`）。

### 4.2 格式转换

| 目标 | 转换 |
|---|---|
| UNORM | `quantize(clamp(c,0,1), n)` = `round(c * (2^n-1))` |
| SNORM | `quantize(clamp(c,-1,1), n)` |
| UINT/SINT | 直接取整（`convert` 到整型） |
| sRGB | 线性 → sRGB 重编码（0.0031308 分段公式） |

- 舍入取 `round-to-nearest`；确定性来源：像素遍历序、混合公式与舍入规则固定。

### 4.3 MRT 与写 UAV

```
写 UAV（第 20 章描述符）在 PS 内部经 LD/ST；OM 不拦截（UAV 写属于着色器，不经 OM）。
OM 状态初：若 pipeline 声明写 UAV → earlyZ 标记禁用（27 章据此不提前丢弃）。
```

- 全阶段 UAV 路径（S14）在此收口 OM 侧配合。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T26.1 | ① 契约 | `DepthState/StencilState/BlendState/RasterStateOpts/TargetFormat` 与输出区映射落头文件 |
| T26.2 | ② 主体 | 深度测试全量比较 + 写掩码 |
| T26.3 | ② 主体 | 模板测试：front/back 三 op 状态机 |
| T26.4 | ② 主体 | 背面剔除状态：保存 `cull/frontFace` 供 24 章 PA 判定（本章不重复判定） |
| T26.5 | ② 主体 | 颜色混合（方程 + 因子） |
| T26.6 | ② 主体 | 双源混合 + 逻辑 Op |
| T26.7 | ② 主体 | MRT：4 RT 独立 blend/writeMask/格式转换 |
| T26.8 | ② 主体 | PS 输出 → 格式转换（UNORM/SNORM/UINT/SINT/sRGB 舍入确定） |
| T26.9 | ② 主体 | LoadOp/StoreOp：load/clear/dontCare + store/dontCare |
| T26.10 | ② 主体 | 写 UAV 时 early-Z 互斥标记（27 章消费标记） |
| T26.11 | ② 主体 | 统计：depth/stencil 通过/失败、像素写计数 |
| T26.12 | ② 主体 | 与 `ref_model` 的 OM 模块对照（blend/公式） |
| T26.13 | ③ 接入 | `scene_s7_occlusion`（混合/MRT/格式）、`scene_s8_blend_mrt`（Load/Store）、`scene_s14_bindless_uav`（UAV 写路径） |
| T26.14 | ④ 验收 | A26：S7/S8/S14 全绿 + 专有判据 + R26 回归项 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M26.1 | T26.1~T26.4 | 深度/模板/剔除状态，D26.1~D26.4 全绿 |
| M26.2 | T26.5~T26.8 | 混合/双源/逻辑 Op/MRT/格式转换，D26.5~D26.8 全绿 |
| M26.3 | T26.9~T26.14 | LoadOp/StoreOp/early-Z 标记/统计/接入验收，A26 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D26.1 | 接口独立编译；枚举/结构与 §3 一致 | L1 契约 |
| D26.2 | 深度：8 种 func 各测（构造深度=目标值 near）；`writeMask` 关时不写深度 | L1 契约 |
| D26.3 | 模板：fail/深度fail/深度pass 三 op 轨迹正确（逐像素探针） | L1 契约 |
| D26.4 | 剔除状态：OM 保存 `cull/frontFace` 并向 24 章 PA 提供同一状态；镜像后的实际剔除结果由 PA 判定，OM 不重复判定 | L1 契约 |
| D26.5 | blend：`SRC_ALPHA/ONE_MINUS_SRC_ALPHA` 半透明场景输出 = 公式 ±2/通道 | L2 解析 |
| D26.6 | 双源混合/逻辑 Op：`AND` 等逻辑输出与按位期望一致 | L1 契约 |
| D26.7 | MRT：4 RT 各写不同格式，读回各自正确 | L1 契约 |
| D26.8 | 格式转换：0.5→R8 UNORM=128；-0.5→R8 SNORM=192；UINT/SINT 取整正确；sRGB 重编码 ±1 LSB | L2 解析 |
| D26.9 | LoadOp=load 读旧值、clear 写指定值；load/store 的 `dontCare` 不读取或承诺目标内容，测试不比对其值，但所有访存必须在资源范围内且 `mem_sanitizer` 零越界 | L1 契约 |
| D26.10 | PS 写 UAV → pipeline.earlyZ=false（27 章据此禁用 early-Z）；非 UAV → true | L1 契约 |
| D26.11 | depthTestFail/stencilFail/写计数两次运行一致 | L1 契约 |
| D26.12 | 混合/格式 20 例与 `ref_model` 对照 | L2 解析 |
| D26.13 | S7/S8/S14 出图 = ref_model | L2 + A26 |
| D26.14 | A26 全绿：深度/模板/混合/MRT/剔除正确、LoadOp/StoreOp 正确、双源混合与逻辑 Op、PS 输出格式舍入确定、写 UAV 时 early-Z 禁用 + R26 回归项 | L1 + L2 |

---

## 7. 场景与判据（A26）

**绑定场景：S7 `scene_s7_occlusion`（混合/格式）、S8 `scene_s8_blend_mrt`（Load/StoreOp）、S14 `scene_s14_bindless_uav`（写 UAV 全阶段）。**

| 项 | 内容 |
|---|---|
| 输入 | S7：半透明多层叠加（blend）、MRT 多 RT、UNORM/UINT 目标；S8：`load`/`clear`/`store` 组合；S14：PS 写 UAV + depth 场景 |
| 产物 | RT 多张读回、深度快照、blend 探针 |
| 主判据 | S7/S8/S14：L1 契约（深度/模板/blend/MRT/格式舍入断言）+ L2 探针（颜色 ±2/通道、深度 ±1 LSB、指定像素）一致；全图差分仅告警（§1.4） |
| 本章专有判据 | ① depth/stencil/blend/MRT 正确，cull 状态由本章保存、由 24 章判定；② LoadOp/StoreOp 正确；③ 双源混合与逻辑 Op；④ PS 输出 UNORM/SNORM/UINT/SINT/sRGB 舍入确定；⑤ 写 UAV 时 early-Z 禁用 |
| 回归项 | R26：E2/E4 基线不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | early-Z/Hi-Z/fast clear（27 章）；MSAA resolve（27） |

> 容差口径同 Q-60：覆盖 ±1/16、颜色 ±2/通道、深度 ±1 LSB；**探针比对为阻断判据，全图逐字节差分仅告警（`roadmap.md` §1.4）**；零容差仅限 1x 覆盖率与 L1 结构/计数断言。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 像素处理序、混合公式、舍入规则固定 |
| 时间无关 | 无时间输入 |
| 可观测 | depth/stencil 通过失败计数、写计数、`--trace=om` |
| 契约影响 | OM 状态结构、输出区映射、格式转换规则冻结 |
| 简化登记 | 截断写 → 完整测试/混合/MRT/转换/LoadStoreOp |
| 计算兼容 | OM 仅图形（明确允许）；UAV 写路径（着色器侧）与计算共用 |
| 配置与回退 | blend/逻辑 Op 开关、`writeMask`、`loadOp/storeOp` 均为 `OMState`/PSO 载荷字段（`BlendState`/`TargetFormat`/`LoadStoreOp`），非全局配置键 |
| 调试钩子 | `--trace=om`：每像素一行 `{lane, stencil, depth, blendOut}` |
| trace 契约 | 不进入判据 |
| 契约变更 | OM 状态/格式/混合/LoadStoreOp 为冻结项；变更走 ADR 回归 S7/S8/S14 |
| 校验归属 | L1（结构/掩码）+ L2（blend/格式与 ref_model） |
| 双策略 | OM 与执行策略无关；两策略输出一致（写序固定） |
| 阶段衔接 | 为 27（early-Z/fast clear 消费标记、MSAA 解析）、33（UAV 间接）提供 OM 语义 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| OM | 单一深度 → 深度+模板+混合+MRT+转换+LoadStoreOp | —（定案） |
| 深度 | 固定 Less → 8 func | — |
| 格式 | 截断 → UNORM/SNORM/UINT/SINT/sRGB | — |
| early-Z | 仅语义标记 | 27（真实优化） |

---

## 10. 待定项

- 模板操作 `opMask` 与三 op 的组合语义（尤其 depthFail≠stencilFail 时）SM 撮合，先按 D24S8 独立字节模拟；若 35.2（Vulkan stencil 语义）需更细，走 ADR。
- 逻辑 Op 与 blend 仅启用二者之一（真实 GPU 亦互斥）；一期按互斥处理，不实现同时启用错误检测（校验留 35.2）。
- `dontCare` 的 load 不保证旧内容可用，store 不保证写入任何特定值；引用结果不得参与数值比对，但地址计算仍须合法且不得越界。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A26 和适用的 R26 后提交。分配核对项为RT/深度槽、PS 输出 VGPR 与 PSO flags，新增或改动按 [编码与位段校验表](../design/allocation-audit.md)登记。

```text
feat(gpu-ch26): merge render outputs and storage writes

- 实现深度模板、混合、多 RT 与 UAV 输出。
- 用 S7/S8 与 S14 已支撑子例比对 RT/UAV 读回。

Tasks: T26.1–T26.14
Validation: <实际执行的命令、用例和结果>
```
