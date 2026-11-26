# 第 21 章 纹理单元

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把初级"nearest 单层 RGBA8"的纹理采样替换为完整硬件纹理单元：格式解码（含 sRGB）、线性/tile 两种布局、mip/LOD/过滤、数组/Cubemap 寻址、独立采样器对象、以及异步纹理采样（延迟隐藏，ADR-036）。设计说明见 `../design/texture-unit.md` §1~§4。
> 与本章相关的设计说明：`../design/texture-unit.md`（M21.1~M21.3 / 错误语义 / 延迟隐藏）；ADR-036（波挂起唤醒）；`../design/memory-system.md` §8（采样走 IMemory + 缓存）；`../design/command-mmio.md` §3（`stateId 3/5` 纹理/采样器描述符堆基址；采样器并入 `stateId 5`）。
> 落地模块：`hw_sim/texture`（采样、布局和格式）、`hli/detail` 的采样器数据和章内支架、`hw_sim/shader`（采样挂起及唤醒）、S4/S14。35.1 的 HLI 复用采样器数据。

---

## 1. 目标与非目标

### 1.1 目标

1. 格式解码：一期支持 `RGBA8/BGRA8/R8/RG8/D24S8`；sRGB 采样时解码到线性空间、分值重编码回 sRGB。
2. 布局：线性与 tile 两种；tile 尺寸由格式与 mip 决定，寻址对上层透明（`tile` 模式 `2x1` 或 `2x2` 按格式）。
3. mip/LOD：`uv` 与导数均为归一化纹理坐标；纹理单元按纹理宽高将导数换算为 texel 空间，计算 rho 与 LOD，再按采样器 bias 与有效 mip 范围裁剪；相邻 mip 三线性过滤（Q-53）。
4. 过滤：nearest、bilinear、trilinear；各向异性只留 capability bit，一期拒参（返回 `NotSupported`，Q-79）。
5. 寻址：`wrap/clamp/mirror/border`；数组层 `layer` 与 Cubemap 面显式选择。
6. 比较采样：`SAMPLE_CMP` 返回比较结果；shadow 路径不改变格式解码。
7. 独立采样器对象：`CreateSampler`；采样器写入 `stateId 5` 描述符堆、复用 `BIND_DESCRIPTOR_TABLE`，采样器状态不再全在 PSO/SHADER_PSO（Q-52）。
8. 异步纹理采样（ADR-036）：波发出请求即挂起、让出 CU、结果返回唤醒（波调度器维护就绪波队列）；只影响性能/调度观测，不影响采样结果。

### 1.2 非目标

- 不做各向异性过滤（capability bit 保留、一期拒参返回不支持）。
- 不做压缩纹理、稀疏纹理（范围外 ADR-025）。
- 不做纹理阵列动态创建/更新（资源绑定责任在 20 章）。
- 不做 per-lod 逐层存储的自动展开（mip 链由 `CreateTexture` 提供）。
- 不做 4D/体积模型（depth 维度仅用于 D24S8 保留）。

---

## 2. 单元框图

```
 SAMPLE 指令（着色器）──► 纹理单元（独立）
        │  请求{texDesc, sampler, float2 uv, dPdx/dPdy, layer, face}
        ▼
┌──────────────────────── hw_sim/texture ─────────────────────────┐
│ AddressCalc(地址/寻址) → Tiling(布局) → FormatDecode → Filter    │
│   │  mip/LOD 选择；Cubemap face 选择；border                         │
│   └─ IMemorySubsystem.Read（翻译+缓存）                          │
│ MipChain：从 base + offset 读多级 → Trilinear 合（可选）          │
└────────────────────────────────┬────────────────────────────────┘
                                 ▼
                    结果 RGBA/深度 → 返回 lane（或挂起唤醒）
```

- 采样独立模块：`SAMPLE`/`SAMPLE_CMP` 由波调度器触发，非内联在着色器执行。
- 纹理只读；写入/绑定/视图走 20 章。

---

## 3. 输入输出契约

### 3.1 纹理描述（`hw_sim/texture/texture.h`）

```cpp
struct TextureDesc {
    GpuVAddr baseVa;
    u32      baseAlignShift;   // tile 对齐
    Format   format;           // RGBA8/BGRA8/R8/RG8/D24S8
    u32      width, height, depth;
    u32      mipLevels;        // ≥1；层内连续
    u32      arrayLayers;      // 1 或 N（数组/Cubemap）
    bool     isCubemap;        // layers=6 的倍数
    bool     sRgb;             // 采样解码色域
    Layout   layout;           // Linear / Tiled
};
```

### 3.2 采样器（共用绑定数据）

```cpp
struct SamplerDesc {
    Filter  magFilter, minFilter, mipFilter;  // nearest / linear / linear_mipmap
    Address wrapS, wrapT, wrapR;              // repeat / clamp / mirror / border
    float   mipLodBias;
    float   minLod, maxLod;
    CompareFunc compareFunc;   // SAMPLE_CMP 用
    float   borderColor[4];
};
Result CreateSampler(const SamplerDesc&, SamplerId* out);   // 独立采样器对象
```

- 采样器对象创建后写入 `stateId 5` 描述符堆，复用 `BIND_DESCRIPTOR_TABLE`（Q-52）；不新增 `BIND_SAMPLER`。

### 3.3 SAMPLE 指令接口

```cpp
struct SampleRequest {
    GpuVAddr texBufBase;      // 纹理描述符堆基址（stateId 3）
    float    uv[2];           // float2 归一化 uv
    float    dPdx[2];         // uv 对屏幕 x 的偏导
    float    dPdy[2];         // uv 对屏幕 y 的偏导
    u32      layer, face;     // 数组/Cubemap
    SamplerId sampler;        // stateId 5 描述符堆索引
};
struct SampleResult { float rgba[4]; float depth; Result code; };
// LOD 由纹理单元按 uv/dPdx/dPdy 与采样器状态计算（Q-53）
// 波调度器：SendSample(req) ; void OnSampleReturn(req, res)
```

- `SAMPLE_CMP` 载荷多一个 `refValue`；返回 `(ref cmp tex) ? 1.0 : 0.0`。

---

## 4. 内部数据流

### 4.1 采样流水（同步子步骤）

```
ReceiveRequest → AddressCalc(寻址模式→像素坐标)
  → Tiling: 线性 → base+stride*row+col*px ; tile → (tileRow,tileCol,pxInTile)
  → FormatDecode: 位域解码 → 线性 f32 / UNORM 量化解码；sRGB→线性
  → Filter: nearest取1 / bilinear取4 / trilinear取8(两 mip)
  → (SAMPLE_CMP) 与 ref 比较 → 0/1
  → 返回 Result
```

### 4.2 LOD 计算

- `uv`、`dPdx`、`dPdy` 均使用归一化纹理坐标；导数换算到 texel 空间：`rhoX = sqrt((dudx * width)^2 + (dvdx * height)^2)`，`rhoY = sqrt((dudy * width)^2 + (dvdy * height)^2)`，`rho = max(rhoX, rhoY)`。
- `lod = log2(max(rho, 2^-24)) + mipLodBias`，再裁剪到 `[max(0,minLod), min(maxLod,mipLevels-1)]`；合法区间为空或导数为 NaN/Inf 时返回 `InvalidArgument`。三线性过滤使用 `floor(lod)` 与下一 mip，并按小数部分插值。
- 点/线等无导数路径使用 `lod=0`；25 章 quad 路径必须为边缘 lane 提供确定导数。

### 4.3 数组/Cubemap 寻址

- 数组：层索引 = `layer`；越界 clamp 到最后有效层。
- Cubemap：正面 `face` 显式（`RIGHT/LEFT/UP/DOWN/FRONT/BACK`），层基准 = `faceIndex*每面层数`。
- 越界 layer/face → 按 clamp（合法寻址），不报错（非法层数属绑定侧 20 章验证）。

### 4.4 异步（ADR-036）接入

```
SAMPLE → 波发出请求 → SendSample(req) → 波 Suspend（进挂起队列）
  → 就绪波继续执行 → 纹理单元处理（异步 tick）→ OnSampleReturn(req,res)
  → Calendar 上唤醒该波（回到就绪波队列）
```

- `waveSwitch` 软观测；采样结果不受调序影响（结果只与输入绑定）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T21.1 | ① 契约 | `TextureDesc`/`SamplerDesc`/`Request/Result`/格式枚举索引落头文件 |
| T21.2 | ② 主体 | 格式解码：RGBA8/BGRA8/R8/RG8/D24S8 位域 → f32 |
| T21.3 | ② 主体 | sRGB 解码/重编码（采样→线性、写回→sRGB） |
| T21.4 | ② 主体 | 线性布局寻址（base + stride 计算） |
| T21.5 | ② 主体 | tile 布局寻址（tileRow/col/pxInTile，按格式 tile 尺寸） |
| T21.6 | ② 主体 | mip 链读取 + LOD 计算（导数 + bias + clamp） |
| T21.7 | ② 主体 | nearest / bilinear 过滤 |
| T21.8 | ② 主体 | trilinear（双 mip 合成） |
| T21.9 | ② 主体 | 寻址模式 wrap/clamp/mirror/border 实现 |
| T21.10 | ② 主体 | 数组/Cubemap 层索引与面选择、越界 clamp |
| T21.11 | ② 主体 | 独立采样器对象创建并写入 `stateId 5` 描述符堆（`SAMPLE` 引用 samplerId） |
| T21.12 | ② 主体 | 先按 [分配校验](../design/allocation-audit.md) 登记 `SAMPLE_CMP` 实际 opcode 和操作数，再接入比较采样。可先用类型化请求验证比较语义 |
| T21.13 | ② 主体 | 异步采样接入：Suspend/OnSampleReturn/waveSwitch |
| T21.14 | ③ 接入 | `scene_s4_textured_quad`（数组/Cubemap/LOD）、`scene_s14_bindless_uav`（bindless 中纹理）+ S4 采样器 |
| T21.15 | ④ 验收 | A21：S4/S14 全绿 + 本章专有判据 + R21 回归项 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M21.1 | T21.1~T21.5 | 格式/tiling/线性布局，D21.1~D21.5 全绿 |
| M21.2 | T21.6~T21.10 | mip/LOD/过滤/寻址/数组与 Cube，D21.6~D21.10 全绿 |
| M21.3 | T21.11~T21.15 | 采样器/CMP/异步/场景接入验收，A21 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D21.1 | 接口独立编译；格式/寻址/采样器枚举对齐 §3 | L1 契约 |
| D21.2 | RGBA8/BGRA8 各格式 `Sample` 返回逐通道与手工解码值一致；D24S8 深度返回正确 | L1 契约 + L2 |
| D21.3 | sRGB 纹理：采样解码线性值 = 期望（对照公式）；重编码后写回一致 | L1 契约 |
| D21.4 | 线性布局：`(x,y)` 采样地址 = `base + y*stride + x*px` 精确 | L1 契约 |
| D21.5 | tile 布局：`(x,y)` 定位 tile 行/列与 tile 内偏移正确（构造随机访问比对线性参考） | L1 契约 |
| D21.6 | 两级 mip：LOD=0 用 mip0、LOD=1 用 mip1；bias/clamp 边界正确 | L1 契约 |
| D21.7 | bilinear：`(2.5,2.5)` 采样四邻混值 = 手工公式 ±1 ULP | L2 解析 |
| D21.8 | trilinear：两 mip 混合系数正确 | L2 解析 |
| D21.9 | wrap 越界 → 周期回绕；clamp → 边界值；mirror → 镜像；border → 边界色 | L1 契约 |
| D21.10 | 数组 index=3/层，返回第 3 层数据；Cubemap 各 face 数据独立正确；越界 face → 最后有效面（clamp） | L1 契约 |
| D21.11 | 采样器对象：不同 sampler（nearest/linear）绑定同一纹理结果不同且正确 | L1 契约 |
| D21.12 | `SAMPLE_CMP` 与 `ref` 比较 `{Less,Leq,Greater,Geq,Equal,NotEqual,Always,Never}` | L1 契约 |
| D21.13 | 异步开启前后同输入采样结果逐像素一致；`waveSwitch` 计数增加（软观测） | 横切零容差 |
| D21.14 | S4：数组/Cubemap/LOD 全覆盖 + 独立采样、S14：bindless 纹理路径 | L2 + A21 |
| D21.15 | A21 全绿：LOD/sRGB/寻址正确、数组/Cubemap 层索引与越界确定、独立采样器、SAMPLE_CMP 正确 + R21 回归项 | L1 + L2 |

---

## 7. 场景与判据（A21）

**绑定场景：S4 `scene_s4_textured_quad`（数组纹理 + Cubemap）、S14 `scene_s14_bindless_uav`（bindless 中纹理采样路径）。**

| 项 | 内容 |
|---|---|
| 输入 | S4：两列 2 层数组纹理 + Cubemap（6 面 RGB 分色）；S14：bindless 纹理（含 sRGB 一张、比较采样一张） |
| 产物 | 采样后端 RT/UAV、采样器绑定记录、LOD 探针 |
| 主判据 | S4/S14：可寻址采样后的 RT 输出 = ref_model 期望（覆盖角点与 LOD 边界探针） |
| 本章专有判据 | ① LOD/sRGB/寻址正确；② 数组/Cubemap 层索引与越界确定；③ 独立采样器对象（并入 `stateId 5`）与比较采样正确；④ SAMPLE 请求携带 float2 uv + 导数（dPdx/dPdy），LOD 由纹理单元计算；⑤ 异步采样挂起/唤醒后结果一致且 `waveSwitch` 计数 > 0（延迟隐藏生效） |
| 回归项 | R21：初级 E3 nearest 基线不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 各向异性只留 capability bit；深度纹理/比较在 26/27 章 OM/可见性消费断章 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | LOD 计算、寻址、滤波公式确定；异步不改变结果 |
| 时间无关 | 波形调度逻辑，非墙钟；LOD 用帧导数 |
| 可观测 | `waveSwitch`、采样计数、`--trace=texture` 每次采样一行 |
| 契约影响 | `TextureDesc/SamplerDesc`/SAMPLE 接口冻结 |
| 简化登记 | 初级 nearest/RGBA8/clamp → 完整纹理单元 + 采样器 |
| 计算兼容 | 采样单元与图形/计算共用：compute kernel 可采样（37/stage3 38~42，原 35.5） |
| 配置与回退 | `texture.async` 为 `TextureUnitOptions::async` 内部选项（默认开，用于调试确定性）；各向异性走 `Capability::Aniso` 能力位（恒 false，非 0 拒参），均不是全局配置键 |
| 调试钩子 | `--trace=texture`：`{req, texAddr, lod, filter, out}` 单行 |
| trace 契约 | 不进入判据 |
| 契约变更 | 格式列表/tiling/寻址/采样器语义为冻结项；变更走 ADR 回归 S4/S14 |
| 校验归属 | L1（结构/寻址）+ L2（滤波/LOD 与 ref_model 手算值比对） |
| 双策略 | 纹理单元两策略输出一致；异步选序只在调度软件观 |
| 阶段衔接 | 为 25（quad 导数）、26（深度纹理）、35.2（Vulkan 纹理对象）提供采样基础 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 采样 | 初级 nearest/RGBA8 → 完整格式/布局/filter/sampler | —（定案） |
| 异步 | 同步 → Suspend/Resume 延迟隐藏 | — |
| 寻址 | clamp → wrap/clamp/mirror/border + 数组/Cube | — |
| 各向异性 | capability bit 保留，一期不支持 | —（二期） |

---

## 10. 待定项

- tile 尺寸（`2x1` vs `2x2` vs 按格式变量）的具体表在 M21.1 实现时依据格式族固定；若 S4 出现格式与布局组合未覆盖的情况，新增组合走 §7.3 登记，不改变既有寻址语义。
- sRGB 线性化的精度（用查找表 vs 公式）影响 `±2/通道`容差内是否稳定；选择公式计算（确定性更优），若 CI 抖动再评估 LUT。
- 异步采样在主循环模拟里"单步处理完还是分批"，只影响 `waveSwitch` 软件观测数、不影响结果；同时保留一个"async=off"模式用于调试确定性。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A21 和适用的 R21 后提交。分配核对项为SAMPLE_CMP、sampler 编码与索引，新增或改动按 [编码与位段校验表](../design/allocation-audit.md)登记。

```text
feat(gpu-ch21): sample mipmapped array and cube textures

- 完成纹理解码、寻址、LOD/过滤、独立采样器与比较。
- 用 S4/S14 采样子例和边界坐标差分，先登记比较采样编码。

Tasks: T21.1–T21.15
Validation: <实际执行的命令、用例和结果>
```
