# 第 7 章 纹理最小集

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章引入第一类采样资源：RGBA8 纹理的地址计算（UV → 纹素）、nearest 采样与越界 clamp 语义，以及 `SAMPLE` 指令的最小实现；同时把校验体系从"进程内手工断言"升级为"CPU 参考模型 + 全图逐字节 / 结构断言"（初级不引入探针表，Q-39），为高级阶段的双轨校验奠定基础。
> 与本章相关的设计说明：`../design/texture-unit.md` §1~§2（环节约定与初级最小集）；`../design/validation-model.md` §1~§4（校验分层、容差；初级不引入探针表）；`../design/memory-system.md` §7（分配确定性）；第 5 章（`LD/ST`/`SAMPLE` 区段）。
> 落地模块：`hw_sim/texture/nearest`（初级版）、`common/format.h` 追加纹理格式、`hw_sim/shader/isa` 追加 `SAMPLE` 语义、`tools/ref_model`、`tools/image_diff`、`apps/scene_e3_textured_quad`。

---

## 1. 目标与非目标

### 1.1 目标

1. 纹理资源：RGBA8 数据经 `Upload` 进入地址空间；纹理描述符（宽/高/格式/寻址）最小集合先以 PSO/`SAMPLE` 立即数形式随包下发（对象模型推迟到第 21 章）。
2. `SAMPLE` 指令：UV 坐标 → 纹素地址 → `IMemorySubsystem` 读回 → 输出；地址按 UV 取址公式（归一化 0..1 → 纹素中心采样）。
3. 越界 clamp：UV 超出 [0,1] 时 clamp 到边界纹素；**这是合法寻址，不是错误**。
4. 参考模型：`tools/ref_model` 生成期望纹素/像素值；`image_diff` 比对；初级比对口径为全图逐字节 + 结构断言，不引入探针表（Q-39）。
5. E3 贴图四边形全链路：复用第 6 章光栅，PS 增加 `SAMPLE`，nearest 全图逐字节。

### 1.2 非目标

- 不做 tiling、mip、LOD、bilinear/trilinear、sRGB、数组/Cubemap、比较采样、独立采样器对象（第 21 章）。
- 不做描述符堆与资源视图对象（第 20 章）；采样参数走常量区/`SAMPLE` 立即数字段。
- 不做各向异性（只留 capability bit，一期拒参，Q-79）。
- 不引入探针表：初级只用全图逐字节 + 结构断言，探针推迟到高级（Q-39）。

---

## 2. 单元框图

```
 scene_e3（2×2 纹素 + 四边形 UV）
     │ Upload(RGBA8)
     ▼
 纹理资源（地址空间内字节块 + 描述元数据）
     │ SAMPLE(uv) in PS
     ▼
┌──────────────────── hw_sim/texture/nearest ─────────────────────┐
│ 寻址：u,v ∈ [0,1] → clamp → 纹素坐标 (x = floor(u*W), y = floor(v*H)) │
│      纹素中心采样 → 地址 = base + (y*W + x)*4                    │
│ 取读：IMemorySubsystem::Read → RGBA 四字节（[B,G,R,A] 内存序）    │
│ 输出：重排为 float4 rgba → 写入 PS v[..]                        │
└───────────────────────────────┬─────────────────────────────────┘
                                ▼
   tools/ref_model（期望纹素） + image_diff（全图逐字节）
```

---

## 3. 输入输出契约

### 3.1 纹理描述与上传（最小集合）

```cpp
struct TexDescMin {
    GpuVAddr base;      // Upload 返回的 VA
    u32      width, height;   // 1..8192
    u8       format;    // 0 = R8G8B8A8_UNORM（本章仅此值，其余 NotSupported）
    u8       wrapS, wrapT;    // 0 = clamp（本章仅此值）
};
Result UploadTexture(const void* rgba8, u32 w, u32 h, TexDescMin* out);
Result SampleTexture(const TexDescMin&, f32 u, f32 v, float4* out);
```

- `UploadTexture` 内部走 bump 分配 + `IMemorySubsystem::Write`；纹素按 `[B,G,R,A]` 内存序写入（与 RT 同序，ADR-015）。
- 纹理字节布局：`base + (y*width + x)*4`，行无 padding（线性布局，未引入 tiling）。

### 3.2 `SAMPLE` 指令（编码扩展，opcode 区段 0x22）

| 字段 | 语义 |
|---|---|
| `dst` | 输出 `float4`（4 个连续 VGPR 槽：`v[dst..dst+3]`） |
| `src0` | 保存 `{u,v}` 的 VGPR（第 4/5 章已有几何），未用位 0 |
| `imm32[:8]=texId` | 纹理描述在"纹理绑定表"中的索引（本章表仅为 1 项，下一章 E4 可扩至 2 项） |
| 寻址 | `x=clamp(floor(u*W), 0, W-1)`、`y=clamp(floor(v*H), 0, H-1)`；纹素中心语义（u=1.0 → 最后一列纹素） |

- `SAMPLE` 是**内存读而非寄存器搬移**：必经 `IMemorySubsystem`，返回 `NotMapped` 时应定位。UV 为 NaN/Inf 视为非法坐标 → `InvalidArgument`（教材点：clamp 只覆盖有限值域）。
- 输出 `float4` 通道 = `{R/255, G/255, B/255, A/255}`（UNORM 归一化到 [0,1]）。

### 3.3 E3 场景输入（冻结）

| 项 | 值 |
|---|---|
| 纹素 | 2×2：`[0,0]=红`、`[1,0]=绿`、`[0,1]=蓝`、`[1,1]=黄`（RGBA 书写） |
| 四边形 | NDC ±0.8、UV `(0,0)-(1,1)` 全图映射（`scene-specs.md` §2 E3 冻结输入） |
| 采样 | nearest + clamp |
| 判定 | 四象限色块全图逐字节；UV 越界 clamp 行为确定（用 UV>1 的子区域逐字节断言） |

---

## 4. 内部数据流

### 4.1 采样路径

```
SAMPLE(uv, texId)
  → 查描述表 → w/h/format 校验
  → u,v finite ? 否则 InvalidArgument
  → clamp → texel = (cx, cy)
  → offset = (cy*w + cx)*4 → IMemorySubsystem::Read(base+offset, 4)
  → 内存 [B,G,R,A] → float4 {R/255, G/255, B/255, A/255}
  → 写 v[dst..dst+3]
```

### 4.2 参考模型 ↔ 被测

```
scene_e3 输入快照（纹素表 + UV 网格 + 配置） 
      ├─► tools/ref_model：按同一寻址公式离线计算期望全图
      └─► 模拟器渲染 → 实际全图
tools/image_diff：全图逐字节比较，输出 {坐标, 期望, 实际, 差值}；容差取 tolerances.toml（nearest 零容差）
CI：全图逐字节 + 结构断言，全部阻断（初级不引入探针表，Q-39）
```

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T7.1 | ① 契约 | `TexDescMin`/`UploadTexture`/`SampleTexture` 签名与内存序冻结；`SAMPLE` 编码与 `texId` 绑定表 |
| T7.2 | ② 主体 | 纹素地址计算：clamp + 纹素中心 + 字节偏移；NaN/Inf/宽高 0 错误路径 |
| T7.3 | ② 主体 | `SampleTexture` 读回与重排：内存 `[B,G,R,A]` → `float4 rgba`；`NotMapped` 定位 |
| T7.4 | ② 主体 | `SAMPLE` 指令接入 PS 执行器（第 5 章语义表扩展），越界 UV clamp、输出写 `v[dst..dst+3]` |
| T7.5 | ② 主体 | 纹理绑定表（本章 1 项，扩展机制留位）+ `UploadTexture` 与 bump 分配确定性校验 |
| T7.6 | ② 主体 | `tools/ref_model`：按 §3.1 公式生成期望纹素/像素；`tools/image_diff` 全图逐字节对比 |
| T7.7 | ③ 接入 | `scene_e3` 全链路：四象限贴图四边形、越界 clamp 子区域断言（初级不做探针表入库，Q-39） |
| T7.8 | ④ 验收 | A7：E3 nearest 全图逐字节 + clamp 行为确定 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D7.1 | 头文件独立编译；`TexDescMin` 偏移 `static_assert`；内存序 `[B,G,R,A]` 与 RT 同序断言 | L1 契约 |
| D7.2 | `(u,v)=(0,0)/(0.5,0.5)/(1,1)/(1.5,0.5)/(-0.5,0.5)` 的纹素坐标与公式逐项一致（clamp 前后） | L1 契约 |
| D7.3 | 读回纹素字节序 → float4 归一化逐字节可判（红 `{1,0,0,1}`、黄 `{1,1,0,1}`）；越界纹理 VA 读 → `NotMapped` 且定位 | L1 契约 |
| D7.4 | 单独重跑 E3 全图逐字节与固定基线一致（排除其它场景干扰后采样路径无回归）；注入 `u/v=NaN/Inf` 触发 `InvalidArgument`，`ErrorRecord.va` 定位到纹理描述符 | L1 契约 |
| D7.5 | `texId` 越界（≥绑定表长）→ `InvalidArgument`；两次 `UploadTexture` 同一内容所得 VA 序列一致（确定性） | L1 契约 |
| D7.6 | `ref_model` 对同一输入两次输出一致；`image_diff` 能区分"期望/实际/差值"三列并给出 0 差异报告 | L1 契约 |
| D7.7 | `scene_e3` 全链路：256 基线四象限色块全图逐字节；Clamp 子区域（UV>1 与 UV<0）纹素等于边界色；只用全图逐字节 + 结构断言，不引入探针表（Q-39） | L1 契约 |
| D7.8 | A7 全绿：E3 全图逐字节 + 专有判据全绿；回归 R07（含 E0~E2 场景不退化） | L1 契约 |

---

## 7. 场景与判据（A7）

**绑定场景：E3 `scene_e3_textured_quad`（纹理四边形）。**

| 项 | 内容 |
|---|---|
| 输入 | 2×2 纹素（红/绿/蓝/黄）+ 全图 UV 映射 + 手写 PS（含一条 `SAMPLE`） |
| 产物 | 全图 RGBA、clamp 子区域断言 |
| 主判据 | nearest 全图逐字节匹配固定基线 |
| 本章专有判据 | ① UV 越界 clamp 行为确定（0/负/超 1 三个子区域）；② 纹素中心采样一列边缘像素与参考一致 |
| 回归项 | R07：E0–E2 已交付基线不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 完整纹理单元（格式/tiling/mip/LOD/过滤/数组/采样器）在第 21 章；E3 输入在 A9 收口到端到端帧 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 寻址公式纯函数；绑定表顺序固定；参考模型与模拟器同一公式（无自证：公式以独立常量为输入） |
| 时间无关 | 无时间输入 |
| 可观测 | `--trace=tex`：`{texId, u, v, texel, readVa}`；纹理统计（采样请求数）软观测 |
| 契约影响 | `SAMPLE` 编码、纹理布局、寻址公式、内存序为冻结契约；变更走 ADR + 回归 E3 |
| 简化登记 | RGBA8 / nearest / clamp / 单层 / 无采样器对象（§9） |
| 计算兼容 | 纹理只读、绑定与视图归第 20 章共用；计算侧采样（可选能力）不改变纹理单元语义 |
| 配置与回退 | 无独立开关 |
| 调试钩子 | `--trace=tex` 仅人工 |
| trace 契约 | 不进入判据 |
| 契约变更 | 见"契约影响" |
| 校验归属 | 全图逐字节（L1）+ 结构断言（L2 起步，初级不引入探针表）；容差集中 `tolerances.toml`（nearest 阶段零容差） |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 格式 | 仅 RGBA8 | 21 |
| 过滤 | nearest | 21（bilinear/trilinear） |
| 层级 | 单层、无 mip、无 sRGB | 21 |
| 采样器 | 无采样器对象（固定 nearest+clamp） | 21 |
| 寻址 | clamp 单一模式 | 21（wrap/mirror/border） |
| 布局 | 线性、无 tiling | 21 |
| 对象模型 | 无描述符堆/视图 | 20、21 |

---

## 10. 待定项

- **Q-79（已定案）** 各向异性过滤只保留 capability bit（`../design/texture-unit.md` §1）；一期 `SAMPLE` 不携带各向异性参数、直接拒参，归二期项，不再保留"bit 位 + 拒参"的待定表述。
- **Q-80（已定案）** 初级不引入探针表，Q16 归一化探针记录要求取消；E 场景统一以全图逐字节 + 结构断言验收，探针格式对齐推迟到高级（Q-39）。
