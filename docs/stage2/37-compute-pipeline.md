# 第 37 章 计算管线（CS 计算管线；直连 HLI 入口验收，不含 MUSA 计算驱动对象模型）

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章定义 Compute 硬件从已冻结的 HLI 命令包到 Compute engine、workgroup/wave 执行与内存写回的主通路，并以直连 HLI 场景验收硬件入口。第 29–34 章负责共用的 SIMT、LDS/atomic/barrier、队列、IB、query 等模块机制；第 37 章复用这些模块，不再复制一套实现。Host 参数打包和 LaunchKernel 归第 39 章，stream/event 归第 40 章，驱动侧协作/indirect/UAV/query 包装归第 41 章，双入口端到端对照归第 42 章（ADR-047/048）。
> 设计权威：通路与对象模型见 `../design/compute-line.md`；kernel 参数契约见 `../architecture-design.md` §3.8（KernelArgsLayout 六条校验规则）；计算 pipeline 对象面见 `architecture-design.md` §3.4；同步原语与内存作用域见 `../design/sync-primitives.md` §3；执行与并行模型见 `../design/execution-parallel.md`；Wave/Workgroup 拆分与 ABI 分区见 `../design/isa-abi.md` §2/§3。
> 编号纪律：本章**不新增包类型 / opcode / stateId / MMIO 槽**，全部复用既有登记（`DISPATCH`=0x04、`SET_KERNEL_ARGS`=0x0F/0x0002、`DISPATCH_INDIRECT`、`BARRIER`=0x0E、`PREDICATE`=0x0D）；开工前仍须对照 `../design/bit-budget.md` 审计复用号的所有权（`../roadmap.md` §7.3/§7.4、ADR-043/045）。

---

## 1. 目标与非目标

### 1.1 目标

1. **计算最小通路**（线贯通）：`kernel → module/function → 参数缓冲 → dispatch(grid, block) → CP 派发 → workgroup 分配到 CU/SIMD → wave 调度 → LDS/原子/barrier → 全局内存/图像写回 → fence/查询`，全链路可定位、可探针校验。
2. **直连 HLI 计算入口**通过 `SetKernel` → `Dispatch` / `DispatchIndirect` 跑通已准备参数的计算场景。最小 Stream 和主机参数打包由第 39 章接入。
3. **计算调度与协作收口**（37.2）：三维 workgroup/grid 映射的**驱动侧语义**、grid 边界部分 workgroup、越界 work-item 不写、组内 barrier 组内全部 wave 到达放行、workgroup/device 作用域内存序——把 29~31 的共用单元收口为 kernel 入口语义。
4. **计算系统收口**（37.3）：异步计算（计算队列并发 + 计算→图形 fence 依赖）、间接派发计算侧（`DISPATCH_INDIRECT` 与直接等价）、计算查询与统计、全阶段 UAV kernel 写回收口。
5. **端到端验收（A37，直连 HLI 入口）**：S18/S19 在 `serial` / `parallel` 下硬判据零容差一致、直连 HLI 计算入口跑通；计算驱动入口与直连 HLI 入口的**双入口等价归 stage3 42（A42）**主验收。

### 1.2 非目标

- 不做 MUSA / CUDA 的 API 或二进制兼容（只借鉴对象模型与调用路径，ADR-009/030）。
- 不做周期精确建模、不做真实 occupancy 调优（只输出 wave 数与占用统计，ADR-038/`compute-line.md` §2）。
- 不做动态并行（kernel 内部再派发 kernel，ADR-013 范围外）。
- 不做可变 subgroup / lane swizzle（固定 warp32，lane 序 = 线性序）。
- 不为计算特化硬件单元：所有共用单元不得出现"图形专用 / 计算专用"分支（ADR-032）。

---

## 2. 单元框图

```
apps（计算入口：直连 HLI / umd::compute 驱动）
   │  KernelArgsLayout + DISPATCH(grid, block)
   ▼
hli::IHliCommandList.Dispatch / IHliPipeline(kernel)          （35.1 冻结对象面）
   │  SET_KERNEL_ARGS(0x0F/0x0002) + DISPATCH(0x04)
   ▼
kmd → firmware → ring → CP 按包类型派发（非着色引擎 sa=nullptr 等规则，架构 §3.3）
   ▼
hw_sim::engines（Compute 引擎）──► IShaderArray::Dispatch
   │
   ├─ workgroup 分配到 CU / SIMD（31 章 SE/CU 阵列，共用）
   ├─ wave 调度 → 分歧（15 章，共用）
   ├─ subgroup 协作（29 章 BALLOT/SHFL/REDUCE/DERIV，共用）
   ├─ LDS / atomic / barrier / 内存作用域（30 章，共用）
   └─ 全局内存 / UAV 图像写回（3/11 内存子系统、26 章格式转换表，共用）
   │
   ▼
fence / timeline（32 章队列同步，共用）→ 查询与统计（34 章，共用）
   ▼
S18 / S19 探针表比对（ref_model 生成期望，37 章主验收）
```

说明：

- 本章的"单元"主要是**组装与驱动**：硬件执行单元都是既有共用单元，本章不复制实现；每个收口点都对应一个"驱动/入口侧"任务，硬件侧任务在原章 DoD 中仍是权威落点。
- `DISPATCH` 包、`SET_KERNEL_ARGS` 包、`KernelArgsLayout` 契约在第 20 章形成（图形/计算共用）；本章只负责 Compute 硬件管线消费已冻结的包布局；计算驱动的参数打包和 API 对象包装分别由第 39–41 章负责。

---

## 3. 输入输出契约

### 3.1 kernel 参数（权威：`architecture-design.md` §3.8，第 20 章契约、本章驱动侧收口）

```cpp
struct KernelArgsLayout {
    GpuVAddr paramBufVa;          // 参数区 VA（host 预写或 H2D 拷贝后）
    u32      paramBufByteSize;
    u32      paramVgprBase;       // 默认 56（= uniform 区起点 v[56..67]，isa-abi.md §3 权威分区）
    std::vector<u32> paramOffsets; // 每参数在参数区内的字节偏移
};
```

**六条输入校验规则（第 20 章冻结，本章驱动侧集中执行）**——违反任何一条返回 `InvalidArgument`/`NotMapped`/`NotResident`，**不下发硬件**：

1. `paramBufVa` 已映射且驻留（`HOST_VISIBLE | HOST_COHERENT` 参数缓冲）；
2. `paramBufByteSize > 0` 且 ≤ `kMaxParamBufBytes` = 4 KiB（B-041，独立契约常量）；`v[56..67]` 只承载 `paramBufVa/grid/block` 描述符，不参与参数容量计算（与 VGPR uniform 分区容量解耦）；
3. `paramOffsets` 严格递增、互不重叠，且 `offset + size ≤ paramBufByteSize`；
4. 参数按类型自然对齐（标量 4 B、float2/vec2 8 B、float4/vec4 16 B）；
5. `paramVgprBase` 默认 56，合法基址 `56..60`，8 个槽依次装载参数 VA LO/HI、`grid.xyz`、`block.xyz`。CP 校验整个写入范围，再初始化 VGPR 和派发工作组。
6. 校验只做静态结构校验，不断言参数语义（值域由 kernel 自证）。

### 3.2 计算管线接口（权威：`architecture-design.md` §3.4）

| 对象 | 本章用法 | 落点 |
|---|---|---|
| `IHliPipeline(kernel)` | `SetKernel` 创建计算管线（与图形 stages 同构，创建不按上层类型分支） | 35.1 冻结、本 M37.1 消费 |
| `IHliCommandList.Dispatch(grid, block)` / `DispatchIndirect()` | 命令记录后统一经 32 章队列提交 | M37.1 / M37.3 |
| `IHliQueue.Submit(SIGNAL/WAIT)` | stream/event 的底层载体 | stage3 38~42 复用 32 章原语 |
| `IHliQueryPool` | 计算统计与谓词回读 | 37.3 |

### 3.3 计算驱动边界（归 stage3 38~42）

本章不定义或实现 `Context/module/function/stream/event` 的对象生命周期、模块格式、参数打包 ABI 或驱动 API；这些契约的唯一来源是 `../design/compute-driver.md`，实现与验收归 stage3 38~42。此处只规定硬件消费已冻结的 `DISPATCH` / `SET_KERNEL_ARGS` 描述符，并通过既有 `IShaderArray::Dispatch` 执行 workgroup/wave。stage3 只能组合已冻结的 HLI 接口，不新增硬件包、opcode、MMIO 或 HLI 接口。
### 3.4 输入校验（驱动 / HLI 创建侧执行，与第 20 章一致）

- 未映射 / 未驻留参数缓冲 → `NotMapped` / `NotResident`；
- 偏移重叠 / 越界 / 未对齐 / `paramVgprBase` 越区 → `InvalidArgument`；
- 上述任何失败**不产生** `SET_KERNEL_ARGS` / `DISPATCH` 包；
- 硬件侧对越界访问只做确定性中止（`architecture-design.md` §3.3 错误流），不做语义修复。

---

## 4. 内部数据流或状态机

### 4.1 直连 HLI 计算入口主路径（A37 验收入口）

```
hli 计算入口（SetKernel → Dispatch/ DispatchIndirect）
  ├─ HLI/计算驱动（39）负责校验 §3.4 → 打包 ArgPack[] → 写 GART 参数缓冲 → KernelArgsLayout
  ├─ IHliPipeline(kernel).SetKernel
  ├─ IHliCommandList.Dispatch(grid, block)〔或 DispatchIndirect〕
  └─ HLI 队列 Submit → fence / 查询
```

> umd/compute 计算驱动代码路径（`LaunchKernel` → stream/event：stream 1:1 映射 HLI 队列、Record 插 `SIGNAL`、Wait 插 `WAIT`、Query 非阻塞读）**整体归 stage3 38~42**（ADR-047，对象与 ABI 权威 `../design/compute-driver.md`、章文档 `docs/stage3/`）；本章只以**直连 HLI 入口**做 A37 验收，双入口等价归 stage3 42（A42）。

### 4.2 DISPATCH 派发主路径（收编第 20 章 kernel 参数 + 31 章映射的驱动面）

```
CP 解析 DISPATCH(0x04) → Compute 引擎
  → 读 SET_KERNEL_ARGS {paramBufVa, size, vgprBase}
  → 装载 paramBufVa / grid / block 到 VGPR uniform 区 v[56..67]（isa-abi §3）
  → 三维 grid/workgroup 映射（blockIdx/threadIdx，越界 work-item 不写）
  → 每 workgroup 生成 N = ceil(block / warp) 个 wave 投递 IShaderArray::Dispatch
  → 组内 barrier：组内全部 wave 到达才放行（跨 wave 同步）
  → LDS / 原子 / 写回 → 完成 → fence / 查询
```

### 4.3 可观测状态

- 输入 FIFO（运行单位）→ workgroup 队列 → CU/SIMD 波队列 → 完成队列（软观测 wave 数、占用）。
- 错误统一走 `architecture-design.md` §3.3 错误流（`{unit, packetType, dwordOffset, va, code}`），不做重试恢复。

---

## 5. 任务清单

> 任务编号 `T37.1~T37.14`，采用四段式（① 契约 → ② 主体 → ③ 接入 → ④ 验收）。**吸收来源**标注了该任务对应于原散落位置（20 / 29~31 / 32 / 33 / 34），原章保留共用单元任务与契约，验收判据在本章以**直连 HLI 入口**收口；原 35.5 计算驱动对象模型归 stage3 38~42（ADR-047），不占本章任务号。

| 子里程碑 | 任务 | 段 | 内容（吸收来源） |
|---|---|---|---|
| M37.1 | T37.1 | ① 契约 | 冻结 Compute 引擎消费所需的 DISPATCH/SET_KERNEL_ARGS 硬件载荷字段与入口；参数打包、KernelArgsLayout 记录期校验由架构 ABI 与第 39 章负责 |
| M37.1 | T37.2 | ② 主体 | 硬件侧消费 `SET_KERNEL_ARGS`：按已冻结的 `KernelArgsLayout` 读取 GART 参数区、检查映射/边界并装载 uniform；不负责 `ArgPack[]` 解释或 Host 参数缓冲打包（归第 39 章） |
| M37.1 | T37.3 | ② 主体 | Compute 引擎派发主路径：CP 解析 `DISPATCH` → Compute 引擎 → `IShaderArray::Dispatch`（复用 29~31 共用单元；非着色引擎 `sa` 规则不适用，Compute 必填） |
| M37.1 | T37.4 | ② 主体 | `DISPATCH` 统一区装载：`paramBufVa` / grid / block 写入 VGPR uniform 区 `v[56..67]`（isa-abi §3 权威分区） |
| M37.1 | T37.5 | ③ 局部接入 | 直连 HLI 记录 kernel 与简单 dispatch，先用 S18 基本子集检查命令和写回。完整 S18/S19 在 M37.4 汇总 |
| M37.2 | T37.6 | ② 主体 | 硬件侧按 DISPATCH grid/block 展开 workgroup/wave，并复用第 31 章索引映射；驱动侧不复制 blockIdx/threadIdx 算法（计算驱动只消费统一定义） |
| M37.2 | T37.7 | ② 主体 | 计算引擎用第 30 章 ISA/LDS 同步原语运行跨 wave barrier 用例；本章只验收硬件执行路径可达，不重做 barrier 或驱动作用域 API（归 30/41 章） |
| M37.2 | T37.8 | ② 主体 | 计算内建 id（THREAD/GROUP/WAVE）编译器前端映射与 kernel 侧暴露（内建寄存器面在 15/17 章；本章收口计算侧） |
| M37.2 | T37.9 | ② 主体 | 计算统计口径：每 dispatch wave 数、每 workgroup 波数、occupancy 占用观测（软观测，ADR-038；31 章统计的计算侧口径） |
| M37.3 | T37.10 | ② 主体 | Compute engine 消费已冻结的 queue/fence/SIGNAL/WAIT 命令并按 32 章调度；本章仅验证硬件命令执行和依赖，不实现 stream/event API（归第 40 章） |
| M37.3 | T37.11 | ② 主体 | Compute engine 解释第 33 章登记的 `DISPATCH_INDIRECT` 与 28 B 参数结构，执行与直接派发语义等价的硬件路径；驱动侧参数构造/校验由第 41 章负责 |
| M37.3 | T37.12 | ② 主体 | 计算查询与统计：dispatch 计数、timestamp 单调、查询可作 GPU 侧谓词（查询单元在 34 章；本章收口计算口径） |
| M37.3 | T37.13 | ③ 接入 | Compute shader 对已绑定的全局/图像资源执行写回，验证硬件 memory/image store 通路；UAV view 创建、描述符打包与驱动调用包装归第 41 章 |
| M37.4 | T37.14 | ④ 验收 | A37 跑 `api_backend=hli` 的 S18 `case-0~2/4~12` 与 S19 `case-0~4`，对照双执行策略并回归受影响场景；S18 `case-3` 从 A39 启用 |

### 子里程碑（本章 ≤ 6 任务/块）

| 子里程碑 | 范围 | 承载任务号 |
|---|---|---|
| M37.1 计算通路与派发 | 入口载荷、Compute 引擎派发、uniform 装载、直连 HLI 硬件路径 | T37.1~T37.5 |
| M37.2 计算调度与协作集成 | 复用三维映射与 barrier 原语，验证内建 id 和资源调度路径 | T37.6~T37.9 |
| M37.3 计算系统收口 | 异步计算、间接派发、查询统计、UAV 写回 | T37.10~T37.13 |
| M37.4 端到端验收 | 直连 HLI 计算入口接 S18/S19、A37 | T37.14 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D37.1 | Compute 载荷头独立编译，DISPATCH 字段正确解析；包长、版本、保留位和引擎类别错误返回确定状态；输入数据的记录期校验归 20/35.1，主机参数打包归 39 | L1 契约 |
| D37.2 | 硬件消费已提交参数描述符时，未映射/越界参数区产生确定错误；合法参数正确装入 uniform。Host `ArgPack[]` 的布局/重叠校验与禁止下发由 D39.1/D39.2 验收 | L1 契约 |
| D37.3 | `DISPATCH` 能驱动 Compute 引擎产出与 29~31 章单测一致的结果；派发入口必须持有非空 `IShaderArray* sa` 并对每个有效 wave 调用 `IShaderArray::Dispatch`，缺失 `sa` 在执行前返回 `InvalidArgument` | L1 契约 |
| D37.4 | 基址 `56`、`60` 时逐槽核对参数 VA LO/HI、grid、block，检查 v[68] 与其他非目标槽保持原值。直接构造 `55`、`61`、`UINT32_MAX` 的包，确认 CP 返回 `InvalidArgument` 且无工作组派发 | L1 契约 |
| D37.5 | 直连 HLI 运行 S18 case-0，索引和写回与参考逐项一致。后续子 case 由 M37.2/M37.3 接入，完整场景归 D37.14 | L2 |
| D37.6 | 三维 block/grid 的 blockIdx/threadIdx=ref_model；grid 边界部分 workgroup 有效、越界 work-item 不写任何内存 | L2 |
| D37.7 | block > warp（如 64 = 2 wave / 256 = 8 wave，对齐 S19 `case-0`）跨 wave barrier 只在全部 wave 到达后放行（原子计数断言）；workgroup/device 作用域语义确定 | L2 |
| D37.8 | 内建 `THREAD/GROUP/WAVE` id 与 ref_model 一致；编译器产物不产生非法布局 | L2 |
| D37.9 | 统计输出可观测（wave 数/占用），只作软观测、不进硬判据 | L1 |
| D37.10 | 计算队列与图形队列并发提交，图形在 `WAIT` 前不读计算产物；timeline 值序正确 | L1 + L2 |
| D37.11 | `DISPATCH_INDIRECT(参数=直接 dispatch)` 与直接 dispatch 结果逐点零容差一致 | 横切 |
| D37.12 | 计算 dispatch 计数值正确、timestamp 单调不减；查询可作谓词回读 | L1 |
| D37.13 | kernel UAV 写回按 `(x,y,format)` 与 20 章 image 描述符 + 26 章格式转换表逐字节一致（S19 `case-4`）；不依赖 26 章 OM 写回探针 | 横切 |
| D37.14 | A37：直连 HLI 的 S18 `case-0~2/4~12` 与 S19 `case-0~4` 通过场景判据和双策略对照。S18 `case-3` 进入 A39/A42 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A37）

**绑定场景：直连 HLI 的 S18 `case-0~2/4~12` 与 S19 `case-0~4`。S18 `case-3` 在计算 UMD 入口交付后执行。S10/S11/S12/S16 为回归项。**

| 项 | 内容 |
|---|---|
| 输入 | S18：1D/2D/3D grid + block、kernel 参数、越界 work-item；S19：LDS / 原子 / barrier + 间接 dispatch + 异步计算；两者跑 `api_backend=hli`（直连）；（`api_backend=compute` 计算驱动入口归 stage3 42 双入口等价验收） |
| 产物 | kernel 启动日志、命令包序列、RT/UAV/参数缓冲读回、S18/S19 探针表 |
| 主判据 | S18/S19 的 blockIdx/threadIdx/越界/计数/shared/barrier/内存序 = ref_model（整数/索引零容差）；fp32 归约 ≤1 ULP + 归约顺序断言（`scene-specs.md` §1.2）；间接 dispatch 与直接等价；计算→图形完成整资源移交后可见 |
| 本章专有判据 | 直连 HLI 双策略结果符合本章场景；以现有 D13-a/b/c 检查 HLI 记录期拒绝和无包，A39 复核计算驱动入口的错误映射；有效提交与组驻留可正常推进 |
| 回归项 | R37：S10（波内/组内归约计算侧）、S11（并行）、S12（多引擎）、S16（所有权）在计算线接入后不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | `LaunchWithParamBuffer`、动态并行、可变 subgroup、真实 occupancy 调优 —— 均二期 / 范围外，只留 capability bit 或文档登记 |

> 场景输入与探针单份规约见 `../design/scene-specs.md` §3（S18/S19 节）与 §4 章节验收判据表（第 37 行）。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | workgroup 分配、lane 线性序、归约 lane 序、barrier 放行序显式定义；不使用线程完成顺序或时间输入 |
| 时间无关 | kernel 与位姿无 `uTime`/墙钟；帧序由 `frame_index` 驱动 |
| 可观测 | 每 dispatch wave 数、每 workgroup 波数、SIGNAL/WAIT 序列、命令包序列、`--trace=compute` |
| 契约影响 | 不新增编号（复用既有包号/SubOp/stateId/MMIO/配置键）；`KernelArgsLayout` 六条规则为冻结项，变更走 `roadmap.md` §7.4 |
| 简化登记 | 计算线无初级简化（高级引入）；本章替换"35.5 前计算任务散落各章"的中间状态，见 §9 |
| 计算兼容 | 本章即计算线本体；共用单元全部按包类型派发，无特化分支 |
| 配置与回退 | 不新增配置键；计算场景同样服从 `exec_mode` / `api_backend`（`hli` / `compute`） |
| 调试钩子 | kernel 参数快照、`--trace=compute`、命令流 dump；只供人工，不进硬判据 |
| trace 契约 | `--trace` 不保证字段稳定、不做机器解析、不进判据 |
| 契约变更 | 涉及冻结契约（参数校验 / 内存序）一律走 ADR，回归 S18/S19 与受影响 S 场景；计算驱动对象模型契约归 stage3 38~42（ADR-047） |
| 校验归属 | L1（结构/状态）+ L2（S18/S19 与 ref_model）+ 横切（间接等价；双入口等价归 stage3 42） |
| 双策略 | S18/S19 在 `serial`/`parallel` 逐项一致（整数/索引零容差；fp32 归约 ≤1 ULP + 顺序断言）；`parallel` 前 31 章已实现（31 章起策略 B） |

---

## 9. 简化登记

| 项 | 状态 | 替换章 / 备注 |
|---|---|---|
| 计算线散落任务 | 原散落 20/29~31/32/33/34 → 本章统一收口（35.5 计算驱动对象模型归 stage3 38~42，ADR-047） | ADR-013（2026-11-19 修订）/ ADR-048；30/31 章保留共用单元实现，验收判据在本章以直连 HLI 入口收口 |
| 单波、WARP32、无分支 | 初级简化（ch4），未提供给计算使用 | 15（分歧）、16（完整 ISA）、37（kernel 入口） |
| 手写 ISA，无编译器 | 初级简化（ch5） | 17~19（编译器三件套）→ 37 消费 ISA |
| 无 shared/atomic/barrier | D 阶段前无组内协作 | 30（共用单元）、37（计算侧收口） |
| 无计算驱动 | 35.5 前无 MUSA 对象面 | stage3 38~42（计算驱动开发线，ADR-047） |

---

## 10. 待定项

- `LaunchWithParamBuffer`（应用直接给参数 VA）留二期，不阻塞一期验收（`compute-line.md` §2）。
- 计算驱动 API 错误码命名以计算驱动头文件为准；映射方向与禁止项权威在 `architecture-design.md` §3.9（计算驱动列）。
- "35.1 HLI 图形侧先冻结，37 章启用计算 pipeline 创建"的先后纪律：35.1 DoD 全绿后才允许写计算驱动（stage3 38，ADR-047），不反向修改 HLI 接口。

---

## 2026-11-21 评审同步说明

workgroup 容量和间接派发 ABI 见 [架构设计 §3.11](../architecture-design.md)。本章交付 Compute 硬件通路和直连 HLI 验收。旧参数打包、Stream/Event 和驱动侧 API 任务已退役，实施归第 39–41 章。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A37 和适用的 R37 后提交。分配核对项为DISPATCH dword、VGPR 区段和容量，新增或改动按 [编码与位段校验表](../design/allocation-audit.md)登记。

```text
feat(gpu-ch37): dispatch compute through direct HLI

- 接通 DISPATCH、8 槽参数初始化、workgroup 执行与写回。
- 先以原始包测 CP 边界，再跑直连 S18/S19 的归属用例。

Tasks: T37.1–T37.14
Validation: <实际执行的命令、用例和结果>
```
