# 第 6 章 固定功能最小集

> 本章把"顶点 → 图元 → 像素"三段固定功能一次串起来，是初级的重心（L 级，章内拆成 `M6.1` IA / `M6.2` 裁剪视口 / `M6.3` 光栅 / `M6.4` 合并四个可单独验收的里程碑，与 `../roadmap.md` §2.1 登记一致，Q-41）：非索引 IA → VS 分派 → 裁剪/视口 → 光栅 → 深度/颜色写回。一条 `DRAW` 命令从此能产出像素。
> 与本章相关的设计说明：`../architecture-design.md` §3.7 前置（顶点布局：初级固定 `RenderVertex`，第 22 章换 `VertexInputLayout`）、`../design/isa-abi.md` §3（VS 输出区 / PS 输入输出区 / uniform 区）；第 4、5 章（波执行与指令语义）为着色段消费方。
> 落地模块：`hw_sim/frontend/{ia,clip,viewport}`、`hw_sim/raster/{rasterizer,interp}`、`hw_sim/om`、`hw_sim/core/cp` 追加 `DRAW`/`SHADER_PSO` 分支、`apps/scene_e2_triangle`、`apps/scene_e4_mesh`。

---

## 1. 目标与非目标

### 1.1 目标

1. 一条非索引 `DRAW` 的完整流水：IA 装配 → VS 逐波 → 视口体/近平面裁剪 → 光栅（像素中心 + 共享边规则）→ PS 逐波 → OM（深度+颜色写回）。
2. 冻结 `RenderVertex` 布局（stride 32：`float4 position` @0 + `float4 color` @16）与阶段间数据契约。
3. 冻结初级 `SHADER_PSO` / `DRAW` 包载荷与 PSO 暂存区最小字段，作为初级固定功能状态入口；高级 HLI `Pipeline` 对象统一 lowering 到此序列化契约，PSO 包仍保留为低层兼容入口。
4. 光栅正确性：共享边不重不漏、退化图元丢弃、像素中心规则、VS-out 区（`v[16..27]`）重心插值。
5. 深度与颜色写回：固定深度比较（LESS）+ 写掩码开关、PS 输出区 → BGRA 内存（截断口径，ADR-015）。
6. 统计可观测：图元数、VS 波数、PS 波数、像素写入数，供硬判据断言与 `--trace` 人工核对。

### 1.2 非目标

- 不做索引/实例化/restart/step-rate（第 22 章）；不做 HS/DS/GS（第 28 章）；不做 MSAA（第 27 章）。
- 不做完整裁剪语义（guard band、多视口、Clip/Cull Distance、参考裁剪器对照——第 24 章）。
- 不做 tile/并行/双策略/occupancy（第 25、31 章）；不做混合/blend/MRT（第 26 章）。
- 不做纹理采样（第 7 章）；不做贴图四边形示例中的 `SAMPLE`。
- 不做运行时替换 VS/PS 的 binder、不做描述符（第 20 章）。

---

## 2. 单元框图

```
 DRAW 包 ──► CP ──► 固定功能流水（单线程、同步）
                                                  ┌─────────────┐
  图元/顶点入口                                     │  IShaderArray │（第 4/5 章）
     IA ──► v[0..15] ──► VS 逐波 ──► v[16..27]      │  波执行      │
      │                       │                   └─────────────┘
      ▼                       ▼
  视口体/近平面裁剪（退化丢弃）        └──► 光栅：像素中心 + 共享边
                                            │ 重心插值 v[16..27] → PS 输入区 v[0..15]
                                            ▼
                                        PS 逐波 ──► OM(深度/颜色写回) ──► RT（BGRA）+ 深度
```

各阶段只认包载荷与统一的状态对象（PSO）；单元之间不共享具体类型，只用 `v[..]` 寄存器区段与 `{x,y,z,w}` 像素载荷。

---

## 3. 输入输出契约

### 3.1 顶点布局（本章冻结；第 22 章替换）

```cpp
// RenderVertex —— 固定布局，stride 32
struct RenderVertex { float4 position; float4 color; };  // 内存顺序 = 声明顺序
static_assert(sizeof(RenderVertex) == 32);
static_assert(offsetof(RenderVertex, position) == 0 && offsetof(RenderVertex, color) == 16);
```

### 3.2 `SHADER_PSO` 包（0x06，本章最小载荷，字段只增不改）

| 字段 | 类型 | 说明 |
|---|---|---|
| `vsEntry` / `vsEntryHi` | u64 分两段 | VS 程序 GpuVAddr（第 3 章地址空间） |
| `psEntry` / `psEntryHi` | u64 分两段 | PS 程序 GpuVAddr |
| `vsSizeDwords` / `psSizeDwords` | u32 | 各程序 dword 数（越界读 → `InvalidIsa`） |
| `psOutSigma` | u32 | 保留（0） |
| `depthBaseVa` / `depthBaseVaHi` | u64 分两段 | 深度缓冲 GpuVAddr（写 PSO 暂存区 `0x87/0x88`，Q-34） |
| `viewport{ x, y, w, h }` | i32×2 + u32×2 | NDC → 像素；w/h ≥ 1 |
| `depthRange{ min, max }` | f32 | NDC z 映射后写回深度缓冲 |
| `depthFunc` | u8 | 本章固定 `Less`；`LessEqual` 等其余值返回 `NotSupported` |
| `flags` | u32 | `bit0` 深度测试、`bit1` 深度写、`bit2` 背面剔除（由 PA 判定，OM 只保存状态，Q-61）、`bit3` 启用视口裁剪 |
| `zNear/zFar` | f32 | 裁剪近/远边界 |
| `primitiveTopology` | u8 | 本章仅 0=TriangleList；其余 `NotSupported` |

PSO 暂存区挂 MMIO `0x80~0xDF`：CP 收到 `SHADER_PSO` 即解析校验并将字段写暂存区；后续 `DRAW` 从暂存区取状态。字段 → 偏移冻结如下（偏移一律为 dword 索引，Q-31/Q-34）：

| 字段 | 暂存区偏移 | 宽度 |
|---|---|---|
| `vsEntry` LO / HI | `0x80` / `0x81` | 2 dword |
| `psEntry` LO / HI | `0x82` / `0x83` | 2 dword |
| `vsSizeDwords` | `0x84` | 1 dword |
| `psSizeDwords` | `0x85` | 1 dword |
| `psOutSigma` | `0x86` | 1 dword |
| `depthBaseVa` LO / HI | `0x87` / `0x88` | 2 dword |
| `viewport.x` / `.y` / `.w` / `.h` | `0x89`~`0x8C` | 4 dword |
| `depthRange.min` / `.max` | `0x8D` / `0x8E` | 2 dword |
| `zNear` / `zFar` | `0x8F` / `0x90` | 2 dword |
| `depthFunc` | `0x91` | 1 dword（u8） |
| `primitiveTopology` | `0x92` | 1 dword（u8） |
| `flags` | `0x93` | 1 dword |

- `0x80~0xDF` 容量 = 96 dword（384 B）；上表占用 `0x80~0x93`（20 dword），`0x94~0xDF`（76 dword）保留，供第 20~26 章按"只增不改"追加字段。

### 3.3 `DRAW` 包（0x03）

| 字段 | 类型 | 说明 |
|---|---|---|
| `firstVertex` | u32 | 起始顶点号 |
| `vertexCount` | u32 | 顶点数，≥3 |
| `flags` | u32 | `bit0` indexed（0=非索引、1=索引；本章要求恒 0，为 1 返回 `NotSupported`）；其余位保留（0） |
| `vertexBufVa` / `vertexBufVaHi` | u64 分两段 | 顶点流基址（已上传，第 3 章） |
| `instanceOffset` | u32 | 恒 0（实例化在第 22 章） |

固定前缀共 6 个载荷 dword（`count=7`），精确位置以 [命令包 §2.5](../design/command-mmio.md) 为准。`SHADER_PSO.flags` 的高 28 位必须为 0，MMIO `0x93` 与包状态共用这一掩码。

- RT 基址来自 `stateId=0`（第 3 章 `SET_STATE_BASE`）；深度基址由 PSO 暂存区的 `depthBaseVa` 字段承载（正式 `stateId=2` 归第 26 章）。
- `vertexCount` 可大于单次派发容量：IA 以 32 为一批分批送 VS，容量不再成为 draw 上限（非专化为第 23 章的"切片"，这里只是分批）。

### 3.4 阶段间数据（内部不动点，测试可单点观察）

| 载荷 | 字段 | 来源 |
|---|---|---|
| 顶点输入 `v[0..15]` | position + color（float4×2） | IA 装配 |
| VS 输出 `v[16..27]` | position.x, y, z(NDC)，w=1；color 随重心插值 | VS 写 |
| 光栅像素载荷 | `{x, y, fragW, 深度z}` | 光栅 |
| PS 输入区 `v[0..15]` | 插值后的 position/color | 光栅插值 |
| PS 输出区 `v[36..39]` | RT0 的 RGBA（float4）；`v[40..43]` 深度 | PS 写 |
| 写回 | RT 内存 `[B,G,R,A]` 截断；深度内存 | OM |

---

## 4. 内部数据流 / 状态机

### 4.1 流水执行顺序（同步）

```
CP: DRAW → 锁读 PSO 暂存区 → 校验参数
IA:  i ∈ [0, vertexCount) 分批（每批 ≤ 32）
       装配 v[0..15]（addr = vertexBufVa + (firstVertex+i)*32）
VS:  batch → Dispatch(active 掩码按剩余顶点裁剪)
CLIP: 每 3 顶点成三角形 → near/zFar/视口 6 平面 Sutherland-Hodgman
       输出 ≥1 子三角形（顶点仍带插值属性）；面积≈0 丢弃
RASTER: 每子三角形包围盒扫描像素中心
       共享边左上规则判定 → 存活 lane（每像素 1 lane，32 像素/批）
       重心插值 → PS 输入区 v[0..15]
PS:  存活像素批 → Dispatch → v[36..39] + v[40..43]
OM:  深度测试(w/d 从 PSO flags)；通过者颜色截断写 RT，深度写回
STATS: 图元数 / 子三角形数 / VS 波数 / PS 波数 / 像素写入数
```

### 4.2 共享边与像素中心规则（单一代码落点）

- 像素中心取 `(x+0.5, y+0.5)`；采样由输出像素坐标驱动。
- 边函数 `E(x,y) = (y1-y0)·x + (x0-x1)·y + (x1·y0 - x0·y1)`；判定规则：对每条边，`E>0` 或 (`E==0` 且该边属"上或左"边，左上角规则)。所有图元共享同一规则 ⇒ 共享边不重不漏。
- 深度按重心系数线性插值（本章 w=1、无透视除法，属性插值同规则）。

### 4.3 裁剪与退化

- 对 NDC 空间 6 平面逐面 Sutherland-Hodgman；插值属性随新顶点按平面交点参数线性生成。
- 退化丢弃三条件（任一成立即丢，计入图元统计但像素写入数不增）：子面积 `|E| < epsilon`、所有顶点被近平/远平面裁掉、所有像素中心落视口外。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T6.1 | ① 契约 | `RenderVertex`/`SHADER_PSO`/`DRAW`/阶段间载荷/PSO 暂存区字段冻结到头文件与断言 |
| T6.2 | ② 主体 | `SHADER_PSO` 解析校验 + PSO 暂存区锁存（MMIO 0x80~0xDF 最小字段表） |
| T6.3 | ② 主体 | `DRAW` 解析、渲染目标/D 绑定（stateId 0 + depthBase）、视口换算 |
| T6.4 | ② 主体 | 非索引 IA：地址计算、`RenderVertex` → `v[0..15]`、越界（顶点流越界）定位 |
| T6.5 | ② 主体 | VS 分批派发：≤32 分批、active 掩码裁剪、VS 波数统计 |
| T6.6 | ② 主体 | 裁剪：6 平面 Sutherland-Hodgman、退化解、属性插值 |
| T6.7 | ② 主体 | 光栅 1：包围盒扫描、像素中心、共享边左上规则、存活掩码 |
| T6.8 | ② 主体 | 光栅 2：重心系数、`v[16..27]` → PS 输入区 `v[0..15]` 插值 |
| T6.9 | ② 主体 | PS 分批派发：存活像素批 → Dispatch、PS 波数统计 |
| T6.10 | ② 主体 | OM：深度比较（Less）+ 写开关、颜色截断写 RT（BGRA）、深度写回 |
| T6.11 | ② 主体 | 退化/出界判定与丢弃路径 + 图元/像素统计汇总（背面剔除仅保存/回读 `flags.bit2` 状态，判定归第 24 章 PA） |
| T6.12 | ② 主体 | 错误定位：无效 PSO 程序 VA、RT/深度越界、顶点流越界、Topology 不支持；`--trace` 观测字段 |
| T6.13 | ③ 接入 | 门面一条完整 draw：`scene_e2` 渐变三角全链路逐字节匹配基线；`scene_e4` 千三角形跑通 |
| T6.14 | ④ 验收 | A6：E2 与 E4 全绿 + 本章专有判据 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D6.1 | `static_assert(sizeof(RenderVertex)==32)`；PSO/DRAW 载荷偏移与 §3.2/§3.3 表一致（脚本比对） | L1 契约 |
| D6.2 | `SHADER_PSO` 锁存**本章登记字段**后可逐字段回读；非法（程序 VA 越界、视口 w=0、`depthFunc!=Less`、`primitiveTopology!=0`）各返回对应 `Result` 且不改写已提交状态。旧判据“0x80..0xDF 每个 dword 均与本章载荷逐一相等”已退役；后续章节可在登记流程下扩展 PSO | L1 契约 |
| D6.3 | 空转（`vertexCount<3`）不产生像素与波；`vertexCount=3` 恰 1 图元；顶点越界 1 个即 `NotMapped` 且错误带地址 | L1 契约 |
| D6.4 | 装配后的 `v[0..15]` 与源 `RenderVertex` 位一致（f32）；`firstVertex=K` 时地址基准 = `+K*32` | L1 契约 |
| D6.5 | 波数 = `ceil(vertexCount/32)`（对 31/32/33 顶点分别为 1/1/2）；最后一波 active=剩余数 | L1 契约 |
| D6.6 | 部分出界三角形（如顶点 z 在近平面后）裁剪后像素计数与手工 Sutherland-Hodgman 结果一致；完全在视口外的三角形像素写入数=0 | L2 解析 |
| D6.7 | 共享边：成对相邻三角形共用边像素被且仅被判给一方（覆盖率与 CPU 参考模型一致，见 ch7 的 `ref_model` 引入后细追踪；ch6 先以手工断言核） | L1/ L2 |
| D6.8 | 插值：像素中心重心系数推导的颜色与 `ExpectedBgra` 中心值 ±2 一致（256 基线中心 (128,128) 期望偏蓝灰）；单色顶点纯色三角形内部逐字节一致 | L2 解析 |
| D6.9 | PS 波数 = `ceil(像素数/32)`；存活掩码按像素批递减正确 | L1 契约 |
| D6.10 | 深度 `z_base + 插值*Δ` 写回正确；`depthTest=off` 时背景深度保持初值；`depthWrite=off` 时深度缓冲不变，颜色正常写 | L1 契约 |
| D6.11 | 退化三角形（三点共线）像素写入数=0 且图元计数 +1；`flags.bit2`（背面剔除状态）可写入并原值回读，本章不做背面判定（像素差异由第 24/26 章验收） | L1 契约 |
| D6.12 | 六类非法输入各触发一次：定位到 `{unit, packetType, dwordOffset, va}`，`GD_LOG` 与 MMIO 错误码一致；`--trace=frontend|raster|om` 逐阶段一行、字段确定性 | L1 契约 |
| D6.13 | `scene_e2`：全图逐字节匹配固定基线（256 基线）；背景非渲染区 = `(63,63,63,255)`；`scene_e4`：单条 draw 且非背景像素比例 > 5%、包围盒输出可复算 | L1 契约 + A6 |
| D6.14 | A6 全绿：E2 逐字节 + E4 非背景比例/包围盒覆盖（A6 处收口）+ 专有判据（共享边/退化/深度）零容差；连续两次运行全图逐字节一致 | L1 契约 |

---

## 7. 场景与判据（A6）

**绑定场景：E2 `scene_e2_triangle`（渐变三角）、E4 `scene_e4_mesh`（非索引大网格）。**

| 项 | 内容 |
|---|---|
| 输入 | E2：固定三顶点 NDC（`scene-specs.md` §2 E2：v0(-0.8,-0.8,0) 红 / v1(0.8,-0.8,0) 绿 / v2(0,0.8,0) 蓝）+ 手写 VS/PS；E4：约 1 千三角形非索引顶点流 |
| 产物 | E2 全图 RGBA；E4 非背景像素比例 + 包围盒；全链路 `--trace` 样例 |
| 主判据 | E2 全图逐字节匹配固定基线；E4 非背景像素比例 >5% + 包围盒可复算 |
| 本章专有判据 | ① 共享边不重不漏（成对三角形探针）；② 退化图元丢弃（三角形覆盖 0 像素）；③ 深度写回正确 |
| 回归项 | R06：E0 契约自检、E2 三角形与 E4 网格基线保持稳定；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | E4 的逐字节基线在 A9 全场景收口；纹理 `SAMPLE` 与贴图四边形 E3 在第 7 章 |

> 参考期望来源：ch6 先以进程内手工解析值（`ExpectedBgra`：视口变换 + 像素中心重心权重 + 截断）断言；第 7 章把参考期望集中到 `tools/ref_model`，ch6 判据迁移到 `ref_model` 生成值并维持同一套像素口径。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 顶点/图元/像素遍历顺序固定（顶点号升序、子三角形顺序、像素包围盒行主序）；光栅与统计无随机 |
| 时间无关 | 无时间输入 |
| 可观测 | `Stats{图元数, 子三角形数, VS波数, PS波数, 像素写入数}`；`--trace=frontend/raster/om` 各一行 |
| 契约影响 | PSO/DRAW 载荷与暂存区字段冻结；后续章只能追加字段（第 20~26 章维护注记） |
| 简化登记 | 非索引、固定深度比较、无 blend/MSAA/GS（见 §9） |
| 计算兼容 | IA/光栅/OM 都是图形专用阶段（明确允许）；**共用执行单元**（IShaderArray）不特化 |
| 配置与回退 | `exec_mode=parallel` 仍报未实现；无缓存开关 |
| 调试钩子 | 阶段 `--trace` 均单行、字段确定，仅供人工 |
| trace 契约 | 不进入判据 |
| 契约变更 | 顶点布局/PSO 字段/填充规则为冻结项；变更走 ADR 并回归 E2/E4 |
| 校验归属 | 全图逐字节（L1）+ 手工解析期望（L2 起步，第 7 章迁 ref_model） |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 固定功能 | 非索引 IA | 22（IA 全功能 + 实例化） |
| 裁剪 | 近平面 + 视口 6 平面，无 guard band/多视口/距离 | 24 |
| 光栅 | 像素中心 + 左上规则，无 tile/点线/MSAA | 25、27 |
| OM | 单一深度比较（Less）、无 blend/MRT/格式转换面 | 26 |
| 原语类型 | 仅 TriangleList | 22、25（点/线）、28（Tess/GS） |
| 顶点布局 | 固定 `RenderVertex` | 22（`VertexInputLayout`） |
| 着色器 | 手写 ISA 全汇编 | 17~19 |

---

## 10. 待定项

- **Q-77（已定案）** 深度插值在无透视（w=1）下为线性；若后续顶点引入透视除法（第 24 章起），插值需切换为透视校正并回归 E2/E4 基线——第 24 章引入透视除法时按 ADR 记录基线迁移策略。
- **Q-78（已定案）** E2 基线的覆盖率数值（256 基线 20808、色权重等）在 ref_model 落地（第 7 章）前以手工解析值留存；ref_model 落地后以 `ref_model` 为准，禁止手改期望，差异先修 ref_model 再回归。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A6 和适用的 R6 后提交。分配核对项包括DRAW v1 前缀、PSO flags 与 MMIO。变更按已有分配流程登记。

```text
feat(gpu-ch06): render triangles with the fixed pipeline

- 串通 IA、VS、三角形光栅、PS 和最小 OM。
- 对照 E2/E4 的 RT 输出、退化图元和深度状态。

Tasks: T6.1–T6.14
Validation: <实际执行的命令、用例和结果>
```
