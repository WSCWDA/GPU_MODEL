# 第 20 章 资源与绑定

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章建立图形与计算的统一对象模型：资源（描述存储）与视图（描述用途/切片）分离（ADR-023）、描述符堆（Bindless 一期）、管线对象（图形 stages + 计算 kernel 同构）、root signature 类绑定、资源状态与屏障（粗粒度访问掩码，ADR-024/031）、以及 kernel 参数区封装。它是 21（纹理）、22~26（几何/光栅/OM）与 37/stage3 38~42（计算，原 35.5）的公共底座。
> 与本章相关的设计说明：`../architecture-design.md` §3.4（HLI 对象面将承此对象模型）、§3.8（`KernelArgsLayout`）；`../design/command-mmio.md` §3（`stateId 3/5` 纹理/绑定/采样器描述符堆基址）；`../design/sync-primitives.md` §2（屏障粗粒度）；ADR-023/024/025（资源视图分离/屏障/能力四档）、ADR-042（地址确定性）。
> 落地模块：`umd/graphics/pal/{resource,binding,pipeline}`、`hw_sim`（`USE_RESOURCE/VIEW/DESCRIPTOR/STATES` 数据面）、`apps/scene_s4_textured_quad`、`apps/scene_s5_model`（借用资源）、`apps/scene_s14_bindless_uav`、`apps/scene_s18_compute_basic`（kernel 参数）。

---

## 1. 目标与非目标

### 1.1 目标

1. `Resource`：`{va, sizeBytes, heap, format, flags(UNORM/UINT/SINT/sRGB...)}`——只描述存储；`View`：`{resource, viewType(SRV/UAV/CBV/RTV/DSV), slice[firstLayer, layerCount], firstMip, mipCount, format(reinterpret), size}`——用途与切片。
2. 描述符堆（Bindless 一期）：堆 `{descEntry[heap], capacity}`、`Write(index, view)`、`GetGpuVa(index)`；提交后改堆 → 下一次 draw 生效；**命令包不拷贝描述符**（ADR-023 附注）。堆登记 GART（GPU 可读，Q-51），为设备内存语义。
3. 管线对象：图形 `{stages(VS/PS/HS/DS/GS), fixedState(Depth/Stencil/Blend/Rasterizer/Viewport/Scissor)}`；计算 `{kernel}`。二者同构（`IEShaderStage` + common 统一对象）。
4. root signature 类绑定：绑定表（`stateId 5`）+ 常量区（`stateId 4`）+ push constant（35.1 收口）；`BindRootDescriptors` 在命令流生成 `BIND_DESCRIPTOR_TABLE` 等包。采样器并入 `stateId 5` 描述符堆，复用 `BIND_DESCRIPTOR_TABLE`，不另立独立包（Q-52）。
5. 资源状态与屏障：执行/内存/图像布局屏障；粗粒度 `READ/WRITE/ATTACHMENT/TRANSFER` 掩码（ADR-031）；`BARRIER` 包。
6. kernel 参数区：`KernelArgsLayout{paramBufVa, paramBufByteSize, paramVgprBase=56, paramOffsets[]}`（`architecture-design.md` §3.8，默认值对齐 isa-abi.md §3 uniform 区起点 `v[56..67]`）；推送参数到 GART 参数缓冲。
7. 全阶段 UAV：像素着色器可读写 UAV（写 UAV 时 early-Z 禁用语义在 26/27 章收口）；`S14` 覆盖 bindless + UAV。

### 1.2 非目标

- 不做 RenderPass 对象（LoadOp/StoreOp 由管线/命令描述，26 章实现语义）。
- 不做稀疏资源/压缩纹理（范围外 ADR-025）。
- 不做每 `View` 的 API 特化（Vk/D3D12/GL 专属视图在第 35 章 UMD 层）。
- 不做纹理采样器对象（21 章创建后写入 `stateId 5` 描述符堆）；不做完整 `IA/光栅`（22~26）。

---

## 2. 单元框图

```
 资源表（Umd/PAL 层）
   Resource ──► View(SRV/UAV/CBV/RTV/DSV)
                 │
   DescriptorHeap ── Write(index) ──► 描述符表（stateId 5 基址）
       │                              │
       ▼                              ▼
   BIND_DESCRIPTOR_TABLE(slot) ◄── root signature（stateId 绑定槽映射）
   Pipeline(PSO) ◄── 管线对象（图形固定状态 + 着色器）
       │
       │ STATES（depth/stencil/blend/raster）── resource 转换
       ▼
   BARRIER 包（粗粒度掩码）→ Cache Flush/Invalidate（12 章）
   kernel args（参数缓冲 → uniform 区 v[56..67]）
```

- 硬件单元只消费例行包（`BIND_DESCRIPTOR_TABLE` 等），不认识 UMD 概念；描述符/资源表是 UMD/PAL 侧对象。

---

## 3. 输入输出契约

### 3.1 资源/视图（`pal/resource/resource.h`）

```cpp
struct ResourceDesc {
    GpuVAddr  va;
    u64       sizeBytes;
    Heap      heap;        // Vram/Gart/System
    Format    format;      // 主格式（UNDEFINED 允许 reinterpret）
    u32       width, height, depth;  // 或 buffer bytes
    ResourceUsage usage;   // Tex2D/Rt/Depth/Stencil/Buffer/etc
    Capability flags;      // UAV/RT/DepthStencil 位（ADR-025）
};

struct ViewDesc {
    ResourceId res;
    ViewKind   kind;       // SRV/UAV/CBV/RTV/DSV
    u32 firstSlice, sliceCount;   // layer 切片
    u32 firstMip,  mipCount;
    Format format;         // reinterpret格式（格式重解释，见 D20.x）
    u32   bufferOffset, bufferSize; // CBV 使用
};
```

### 3.2 描述符堆（`pal/binding/descriptor_heap.h`）

```cpp
struct DescriptorHeapDesc { u32 capacity; Heap heap; }; // 登记 GART（GPU 可读）
class DescriptorHeap {
public:
    Result Write(u32 index, const ViewDesc&);    // 普通视图
    Result GetGpuVa(u32 index, GpuVAddr* out) const;
    u32    Capacity() const;
    // 设备内存语义：heap 内容随提交进入硬件视口，不拷贝进命令包
};
```

- Bindless 一期：`GetGpuVa(index)` 直接给硬件 base + offset；允许在应用运行中改写 index 处视图；堆位于 GART，GPU 可读（Q-51）。
- `Write` 越界 → `InvalidArgument`。

### 3.3 管线对象（`pal/pipeline/pipeline.h`）

```cpp
struct GraphicsStages { ShaderBin vs, ps, hs, ds, gs; ShaderIo io; };
struct FixedState {
    DepthState  ds;  // {depthTest, depthWrite, depthFunc}
    StencilState ss; // {front/back, func, ops}
    BlendState  bs;  // {blendEnable, srcFactor, dstFactor, equation}
    RasterState rs;  // {cullMode, fillMode, frontFace, depthBias...}
    Viewport vp; U32  viewportCount; Scissor sc;  // 第 24 章多视口
};
enum PipelineKind { kGraphics, kCompute };
struct ComputePipeline { ShaderBin kernel; KernelArgsLayout paramLayout; };
class Pipeline {
public:
    Result Create(GraphicsStages, FixedState);  // 或 ComputePipeline
    Result Bind(const CommandList&);           // 设置固定状态 PSO 暂存区
    PipelineKind Kind() const;
};
```

- 图形/计算同构：`CreatePipeline` 对两者用同一对象面（ADR-032 不特化）。

### 3.4 包族（`../design/command-mmio.md` §2 登记扩展）

| 包 | 作用 | 载荷要点 |
|---|---|---|
| `EXTENDED`(0x0F) `SubOp=0x0001` → `BIND_DESCRIPTOR_TABLE` | 绑定描述符表基址到槽位 + root slot | `{slot, tableVa, heapIndex}` |
| `EXTENDED`(0x0F) `SubOp=0x0002` → `SET_KERNEL_ARGS` | 设置 kernel 参数基址/布局到 uniform 区 | `{paramBufVa, size, vgprBase}` |
| `BARRIER`(0x0E) | 粗粒度资源状态转换 | 3-dword envelope `{kind,scope,memOrder,accessMask; srcStageMask; dstStageMask}`；resource kind=0 |

- **取号纪律（ADR-043/045）**：顶层 `PKT_TYPE` 已冻结，`0x0F` 定义为 `EXTENDED`（1 个顶层号 + 16 bit 子操作码），见 `../design/bit-budget.md` §2.3 与 `../design/command-mmio.md` §2；新绑定类命令**不再抢占顶层包号**，统一走 `EXTENDED` + SubOp 或 `INDIRECT`(0x0C) IB。本节 SubOp 分配须在 `../roadmap.md` §7.3 登记（编号示例 `B-nnn`，先对照 bit-budget 审计再登记）。
- `BARRIER` 顶部号为 0x0E（`../design/command-mmio.md` §2 已登记，与 32 章共用）；本章实现屏障语义子集（资源状态/布局），队列级 barrier 归 32 章，不争用 0x0F/0x0D。

先登记再使用（`../design/command-mmio.md` §7）。采样器不新增独立包：由第 21 章 `CreateSampler` 写入 `stateId 5` 描述符堆，复用 `BIND_DESCRIPTOR_TABLE`（Q-52）。

> ⚠️ stateId 编号纪律（三档，`../design/bit-budget.md` §3.1）：本章只使用已冻结的 `stateId 3/5`（纹理/绑定/采样器描述符堆）。`0x50~0x7F`（stateId 8~31）为**临时过渡缓冲**，不得作为长期设计；**禁止新增 stateId 到 MMIO 硬编码**，新状态一律走 GPU 内存间接状态表（`SET_STATE_TABLE_INDEX`）。

### 3.5 Kernel 参数（`../architecture-design.md` §3.8）

```cpp
struct KernelArgsLayout {
    GpuVAddr paramBufVa;
    u32      paramBufByteSize;
    u32      paramVgprBase;        // 默认 56（= uniform 区起点 v[56]，isa-abi.md §3 权威分区）
    std::vector<u32> paramOffsets;  // 每参数在参数区内字节偏移
};
```

- 驱动按签名打包 `{ptr, bytes}[]` → 写 GART 参数缓冲 → 生成 layout。
- 硬件 `DISPATCH` 时把参数基址写入 VGPR uniform 区（`v[56..67]`，起点由 `paramVgprBase` 指定）。

**KernelArgsLayout 校验规则（`../architecture-design.md` §3.8，创建侧 HLI / 计算驱动执行，违反即 `InvalidArgument` 且不下发硬件）：**

| 规则 | 违例示例 | 返回 |
|---|---|---|
| 参数缓冲须已映射且驻留 | `paramBufVa` 未映射 | `NotMapped` |
| `paramOffsets` 严格递增、互不重叠 | 参数 A 占 `[0,16)`，参数 B 也占 `[16,32)` 但 A 实际覆盖到 `[8,24)` → 重叠；或两个偏移相等 | `InvalidArgument` |
| 按类型自然对齐 | 8 字节参数落在 4 字节对齐偏移 | `InvalidArgument` |
| `paramVgprBase` 落在 uniform 分区（`v[56..67]`）内且不与其他 ABI 分区冲突 | `paramVgprBase=52`（落入 RT1~3 `v[44..55]`）或 `=76`（临时量区） | `InvalidArgument` |
| `paramBufByteSize > 0` 且不超参数区容量上限 | `paramBufByteSize == 0` 或超过 `kMaxParamBufBytes` 参数区容量上限 | `InvalidArgument`/`OutOfMemory` |

> 非法样例（帮助 35.1/37 实现对照，原 35.5 归 stage3 38~42）：`KernelArgsLayout{va=Mapped, size=64, paramVgprBase=52, offsets={0,16,16}}` → ①A/B 偏移重叠（两个参数都从 `[16,...)` 起）→ `InvalidArgument`；②`paramVgprBase=52` 越出 uniform 分区 → `InvalidArgument`，均不下发硬件、命令流不含 `SET_KERNEL_ARGS`。

---

## 4. 内部数据流

### 4.1 绑定流

```
CreateResource → ResourceDesc（登记资源表）
CreateView → ViewDesc（登记视图表）→ DescriptorHeap.Write(index, view)
BindRootDescriptors(slot=index) → 命令流产生 BIND_DESCRIPTOR_TABLE 包（含 heap base + index）
DRAW/DISPATCH → 引擎按 stateId 5 基址读描述符
约束：命令包只含 index/表基址，不含描述符快照（D20.x 拒绝拷贝）
```

### 4.2 资源状态与屏障

```
资源初始状态（PSO 声明的初始布局）
提交中使用 → 记录当前状态（READ/WRITE/ATTACHMENT/TRANSFER）
跨阶段/跨提交转换 → BARRIER 包 → HW 执行 Cache FlushBank/Invalidate（12 章）→ 状态推进
粗粒度掩码：不做 per-resource 位（ADR-031）
```

- 滥用（未按状态使用资源）不报错（无完整验证资源状态机，本期）——留待 pipeline validation（35.2 起）。但非法 subresource 范围（切片/层/mip 越界）必须报错。

### 4.3 描述符堆设备内存语义

```
提交 1：Write(5, texA)；draw → 取 texA
提交 2（在提交 1 入队后）：Write(5, texB)；draw → 取 texB（非提交 1 的拷贝）
验证：命令包序列不含 texA/texB 数据副本；堆地址不变
```

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T20.1 | ① 契约 | 资源/视图/描述符堆/管线/KernelArgsLayout/包族结构落头文件 |
| T20.2 | ② 主体 | Resource 登记表：创建/销毁/查询 `GetVa/GetSize` |
| T20.3 | ② 主体 | View 分层：SRV/UAV/CBV/RTV/DSV 创建 + 切片/mip/格式重解释 |
| T20.4 | ② 主体 | DescriptorHeap：容量、Write/GetGpuVa、越界拒绝、设备内存语义 |
| T20.5 | ② 主体 | Pipeline 图形对象：stages + fixedState + `Bind` 到 PSO 暂存区 |
| T20.6 | ② 主体 | Pipeline 计算对象：kernel + paramLayout（同构） |
| T20.7 | ② 主体 | root signature 绑定槽：slot ↔（tableVa,index）映射、`BIND_DESCRIPTOR_TABLE` |
| T20.8 | ② 主体 | 常量区/push constant：`SET_CONSTANT` 或 push 常量路径（35.1 收口） |
| T20.9 | ② 主体 | 资源状态机情况：状态下推与跟踪（粗粒度） |
| T20.10 | ② 主体 | BARRIER 包：执行/内存/布局掩码 → 12 章缓存刷/失效 |
| T20.11 | ② 主体 | 定义 `KernelArgsLayout` 的记录期校验，使用预先写好的参数缓冲记录 `SET_KERNEL_ARGS`。按函数签名打包主机字节由 T39.2 实现 |
| T20.12 | ② 主体 | 视图/描述符非法：越界、未绑定、重解释非法格式 → 确定报错 |
| T20.13 | ② 主体 | 管线替换完整性：`Bind` 新 PSO 后旧状态清、新状态生效、删除旧资源引用安全 |
| T20.14 | ② 主体 | 绑定用例数据面与自检：S4 描述符堆+Bindless、S14 常量/UAV 场景；S5 资源视图与 S18 case-0~2 仅作独立绑定/参数布局单测输入（供 ③ 接入，不跑完整场景） |
| T20.15 | ③ 接入 | 用第 20 章 Pipeline 和共用 lowering 接入已交付且适用的图形场景，生成既有 `SHADER_PSO`。通过章内提交支架验证，正式 HLI 对象入口由 T35.04 接入 |
| T20.16 | ④ 验收 | A20：S4/S14 场景通过；S5 资源视图与 S18 参数布局单测通过；本章专有判据与 R20 通过 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M20.1 | T20.1~T20.5 | 资源/视图/描述符堆/图形管线，D20.1~D20.5 全绿 |
| M20.2 | T20.6~T20.8 | 计算管线/root signature/常量区，D20.6~D20.8 全绿 |
| M20.3 | T20.9~T20.16 | 资源状态/屏障/kernel 参数/场景接入验收，A20 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D20.1 | 接口独立编译；`ResourceDesc/ViewDesc/DescriptorHeap` 字段与 §3 一致 | L1 契约 |
| D20.2 | 创建/销毁资源在资源表登记/移除；两次创建同参数得不同 id | L1 契约 |
| D20.3 | View 切片（firstLayer/count）与 mip 范围越界返回 `InvalidArgument`；格式重解释成非法（如 R8G8B8A8→R32F 超尺寸）返回 `NotSupported` | L1 契约 |
| D20.4 | 描述符堆越界 `Write` → `InvalidArgument`；`GetGpuVa` 未写 index 返回 `NotMapped`（或 0 位标记）；容量固定 | L1 契约 |
| D20.5 | 图形 PSO `Bind` 后暂存区字段 = 传入 fixedState（逐一比对）；换管线后旧字段清 | L1 契约 |
| D20.6 | 计算 PSO 同构：`CreatePipeline(kernel)` + `Bind` 后 `SET_KERNEL_ARGS` 可查 | L1 契约 |
| D20.7 | `BindRootDescriptors(slot, heapVa, idx)` → 命令流包内容与参数一致；slot 越界拒 | L1 契约 |
| D20.8 | push constant 越界（超常量区）→ `InvalidArgument`；合法值进入 SB/常量区 | L1 契约 |
| D20.9 | 资源状态随提交推进（READ→WRITE 需 BARRIER）；非法 subresource 范围报错 | L1 契约 |
| D20.10 | `BARRIER(kind=resource)` 的唯一 3-dword envelope 触发 12 章 Flush/Invalidate；`READ→WRITE` 后新值读回 | L1 契约 |
| D20.11 | 用构造的缓冲、大小与偏移验证记录期校验。合法描述生成预期 `SET_KERNEL_ARGS`。重叠、越界、未映射、未驻留或 uniform 区不合法时返回对应错误且无包。硬件装载由 D37.4 验收 | L1 契约 |
| D20.12 | 越界视图/未绑定描述符 → 确定报错（`NotMapped`/`InvalidArgument`），非静默 0 | L1 契约 |
| D20.13 | 绑定新 PSO 后旧资源引用不被新提交使用；删除旧资源时提交未引用（引用计数/安全） | L1 契约 |
| D20.14 | S4/S14 场景断言通过；S5 资源视图、S18 case-0~2 参数布局独立单测通过（不验收 S5/S18 端到端场景） | L1 + L2 |
| D20.15 | Pipeline/状态输入经共用 lowering 生成预期 `SHADER_PSO` 字段，已接入的图形场景输出零容差。通过章内包支架复核，HLI 接线归 35.1 | 横切 |
| D20.16 | A20 全绿：S4/S14 场景、S5 资源视图与 S18 参数布局单测，及专有判据（常量/结构化缓冲/SSBO 取值、越界/未绑定报错、视图切片与格式重解释、管线替换完整、push constant 越界、非法子资源屏障报错、描述符堆设备内存语义）+ R20 | L1 + L2 |

---

## 7. 场景与判据（A20）

**场景与单测：S4 `scene_s4_textured_quad`、S14 `scene_s14_bindless_uav` 为本章场景；S5 `scene_s5_model` 的资源视图断言与 S18 `scene_s18_compute_basic` case-0~2 仅作独立绑定/参数布局单测向量，不运行完整 S5/S18 场景。完整 S5 场景由第 22 章验收，S18 端到端由第 37/42 章验收。**

| 项 | 内容 |
|---|---|
| 输入 | S4：2 层数组 + Cubemap + 独立采样器（21 章配色）；S14：常量/结构化/UAV；S5 资源视图与 S18 kernel 参数布局仅使用独立构造输入 |
| 产物 | 描述符堆内容、命令包序列、kernel args、RT/UAV 读回 |
| 主判据 | 绑定/描述符/管线对象接入后全部 E/S 场景输出 = 基线（零容差） |
| 本章专有判据 | ① 常量/结构化缓冲/SSBO 与 kernel 参数取值正确；② 越界/未绑定返回确定报错；③ 视图切片与格式重解释正确；④ 管线对象替换完整；⑤ push constant 越界报错；⑥ 屏障非法子资源范围报错；⑦ 描述符堆设备内存语义（提交后改 → 下次 draw 生效，命令包不拷快照） |
| 回归项 | R20：E0~E5 不因资源/绑定引入变化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 独立采样器对象在 21 章；全阶段 UAV 写入路径在 26 章收口；真实 root signature API 特化在 35.2~35.4 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 描述符堆写入顺序、绑定槽映射、参数打包顺序固定 |
| 时间无关 | 无墙钟；提交顺序决定下次 draw 生效 |
| 可观测 | 描述符堆 dump、命令包序列复核、kernel args 快照、`--trace=pal` |
| 契约影响 | 资源/视图/描述符/管线/kernel args 契约冻结；包族登记扩展 |
| 简化登记 | 应用侧 PSO 直发 → HLI Pipeline 对象；既有 `SHADER_PSO` 包保留为驱动/硬件间的序列化契约，不删除或更换其硬件职责 |
| 计算兼容 | 图形/计算共用资源视图描述符；kernel 参数走统一路径（37/stage3 38~42，原 35.5） |
| 配置与回退 | `bindless` 为 PAL 能力/视图选项（`ResourceFlags::Bindless`）；`descriptor_heap_capacity` 为 `DescriptorHeapDesc::capacity` 结构字段（非全局配置键） |
| 调试钩子 | `--trace=pal` 绑定/状态每步一行；`--dump-descriptor-heap` |
| trace 契约 | 不进入判据 |
| 契约变更 | 资源/视图/绑定/屏障掩码为冻结项；变更走 ADR 回归 S4/S14 与本章资源视图/参数布局单测，不触发 S18 端到端门禁 |
| 校验归属 | L1（结构/状态）+ L2（S4/S14 输出比对 ref_model）+ 横切（管线替换） |
| 双策略 | 描述符堆与屏障语义与执行策略无关；两策略下场景一致 |
| 阶段衔接 | 为 21（纹理消费描述符）、22~26（管线/固定状态）、35.1（HLI 收口）提供对象与包基础 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 资源 | 无对象 → Resource/View 分离 | —（定案） |
| 绑定 | 无 → 描述符堆 + root sig | — |
| 管线对象 | 直发 PSO 包 → 管线对象 | — |
| 屏障 | 无 → 粗粒度 BARRIER | — |
| kernel 参数 | 无 → KernelArgsLayout | 37（参数打包）/ stage3 38~42（计算驱动打包，原 35.5） |

---

## 10. 待定项

- `BIND_DESCRIPTOR_TABLE`（`EXTENDED` 0x0F `SubOp=0x0001`）/`SET_KERNEL_ARGS`（`SubOp=0x0002`）沿用 §3.4 的取号方案：**不占顶层包号**，走 `EXTENDED` 子操作码（`../design/bit-budget.md` §2.3、`../design/command-mmio.md` §2）。SubOp 号先在 `../roadmap.md` §7.3 登记（`B-nnn`，先对照 bit-budget 审计）再使用，不写死。
- 资源状态机的"完整跟踪 + 验证"（像真实 API）本期只做"状态随提交推进"不做违规检测，是否补检测待 35.2（Vulkan UMD validation）时评估，涉及 `--validate` 项。
- Bindless 二期的"描述符堆 GPU 更新 + 无 CPU 写"是否纳入，留 T35.1（HLI）能力面决定；一期只做 CPU 侧 Write。

---

## 2026-11-21 评审同步说明

BARRIER 使用 [命令包登记 §2.3](../design/command-mmio.md) 的 3-dword 载荷。Pipeline 对象经 T20.15 转换成既有 `SHADER_PSO`。
