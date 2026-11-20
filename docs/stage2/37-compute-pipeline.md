# 第 37 章 计算管线（CS 计算管线；直连 HLI 入口验收，不含 MUSA 计算驱动对象模型）

> 本章是计算线**直连 HLI 入口的收口章**（ADR-013 修订 / ADR-048，2026-11-19/20）：把散落在第 20（kernel 参数）、29~31（波内/组内/并行协作）、32（异步计算）、33（间接派发）、34（查询统计）中的**计算专属开发任务**整合为一条完整、可独立验收的"计算管线"主线；共用硬件单元（subgroup / LDS / atomic / barrier / occupancy / 三维 workgroup 硬件映射 / 队列 / IB / 查询）仍在原章作为**图形 + 计算共用能力**落地（ADR-032），本章只在其上收口计算专属的调度、作用域、间接与统计，并以**直连 HLI 入口**（A37）验收 S18/S19。原 35.5 的 MUSA 计算驱动对象模型**不并入本章**，整体归 stage3 38~42 承载（ADR-047，`../design/compute-driver.md`）。
> 设计权威：通路与对象模型见 `../design/compute-line.md`；kernel 参数契约见 `../architecture-design.md` §3.8（KernelArgsLayout 六条校验规则）；计算 pipeline 对象面见 `architecture-design.md` §3.4；同步原语与内存作用域见 `../design/sync-primitives.md` §3；执行与并行模型见 `../design/execution-parallel.md`；Wave/Workgroup 拆分与 ABI 分区见 `../design/isa-abi.md` §2/§3。
> 编号纪律：本章**不新增包类型 / opcode / stateId / MMIO 槽**，全部复用既有登记（`DISPATCH`=0x04、`SET_KERNEL_ARGS`=0x0F/0x0002、`DISPATCH_INDIRECT`、`BARRIER`=0x0E、`PREDICATE`=0x0D）；开工前仍须对照 `../design/bit-budget.md` 审计复用号的所有权（`../roadmap.md` §7.3/§7.4、ADR-043/045）。

---

## 1. 目标与非目标

### 1.1 目标

1. **计算最小通路**（线贯通）：`kernel → module/function → 参数缓冲 → dispatch(grid, block) → CP 派发 → workgroup 分配到 CU/SIMD → wave 调度 → LDS/原子/barrier → 全局内存/图像写回 → fence/查询`，全链路可定位、可探针校验。
2. **直连 HLI 计算入口**（A37 验收入口）：`SetKernel` → `Dispatch` / `DispatchIndirect` 命令流闭环接入 S18/S19；（MUSA 计算驱动对象模型 `context / module / function / stream / event` 与 `LaunchKernel` 属原 35.5，**归 stage3 38~42**（ADR-047），本章不实现。）
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
- `DISPATCH` 包、`SET_KERNEL_ARGS` 包、`KernelArgsLayout` 契约在第 20 章形成（图形/计算共用）；本章在**驱动与验收侧**收口，不重新定义包布局。

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
5. `paramVgprBase` 默认 56，必须落在 uniform 分区 `v[56..67]` 且不与其他 ABI 分区冲突（如落入 RT1~3 `v[44..55]` 或临时量区 `v[76..]` 均拒绝）；
6. 校验只做静态结构校验，不断言参数语义（值域由 kernel 自证）。

### 3.2 计算管线对象（权威：`architecture-design.md` §3.4）

| 对象 | 本章用法 | 落点 |
|---|---|---|
| `IHliPipeline(kernel)` | `SetKernel` 创建计算管线（与图形 stages 同构，创建不按上层类型分支） | 35.1 冻结、本 M37.1 消费 |
| `IHliCommandList.Dispatch(grid, block)` / `DispatchIndirect()` | 命令记录后统一经 32 章队列提交 | M37.1 / M37.3 |
| `IHliQueue.Submit(SIGNAL/WAIT)` | stream/event 的底层载体 | stage3 38~42 复用 32 章原语 |
| `IHliQueryPool` | 计算统计与谓词回读 | 37.3 |

### 3.3 计算驱动对象（**归 stage3 38~42 实现**；权威 `../design/compute-driver.md`；本章只作概念引用，A37 验收不走计算驱动入口）

- `context` = HLI device + KMD context（ASID），一期 1:1:1；销毁前所有 stream 必须已同步。
- `module` = 自有头 `MUSA_MODULE_HDR` + 代码段 + 符号表（`name / entryOffset / paramBytes / vgprCount / ldsBytes`），常驻 GART；不引入 ELF/COFF，不做重定位。
- `function` = module 内入口（`codeBaseVa + entryOffset` + 参数布局）；分支用 pc 相对偏移，加载不需地址修补。
- `stream` = 1:1 映射一个 HLI 队列；stream 内顺序 = 队列顺序；不允许多队列，优先级创建时声明、不可变。
- `event` = 独立对象 `{queueId, timelineValue, state}`；Record 插 `SIGNAL`、Wait 插 `WAIT`、Query 非阻塞读；值域复用队列 timeline，不复用 fence 对象。
- kernel 参数由驱动打包：应用按签名提交 `ArgPack{ptr, bytes}[]`，驱动按 §3.1 六条规则校验并写入 GART `HOST_VISIBLE | HOST_COHERENT` 参数缓冲、生成 `KernelArgsLayout`；`LaunchWithParamBuffer`（应用直给参数 VA）留二期。

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
  ├─ 校验 §3.4 → 打包 ArgPack[] → 写 GART 参数缓冲 → KernelArgsLayout
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
| M37.1 | T37.1 | ① 契约 | 计算入口契约集中：`Stage::Compute` 派发、`DISPATCH`→Compute 引擎主路径对接、计算管线对象（kernel）统一表、`SET_KERNEL_ARGS` 消费封装、`KernelArgsLayout` 六条校验单测 + D13-a/b/c 拒绝边界单测落头文件（吸收 20 章契约面；35.5 对象契约归 stage3 38~42） |
| M37.1 | T37.2 | ② 主体 | kernel 参数打包与校验：`ArgPack[]` → 写 GART `HOST_VISIBLE\|HOST_COHERENT` 参数缓冲 → 生成 `KernelArgsLayout`；六条规则集中执行，非法样例不下发（吸收 20 章驱动侧打包 + 35.5 launch config） |
| M37.1 | T37.3 | ② 主体 | Compute 引擎派发主路径：CP 解析 `DISPATCH` → Compute 引擎 → `IShaderArray::Dispatch`（复用 29~31 共用单元；非着色引擎 `sa` 规则不适用，Compute 必填） |
| M37.1 | T37.4 | ② 主体 | `DISPATCH` 统一区装载：`paramBufVa` / grid / block 写入 VGPR uniform 区 `v[56..67]`（isa-abi §3 权威分区） |
| M37.1 | T37.5 | ② 主体 | 直连 HLI 计算入口打通：`IHliPipeline(kernel)` + `IHliCommandList.Dispatch` → S18/S19（HLI 入口）闭环（M37.1 收口） |
| M37.2 | T37.6 | ② 主体 | 三维 workgroup/grid 映射**驱动侧封装**：blockIdx/threadIdx 语义、grid 边界部分 workgroup、越界 work-item 不写（硬件映射在 31 章；本章收口为 kernel 入口语义） |
| M37.2 | T37.7 | ② 主体 | 组内 barrier 与内存作用域收口：组内全部 wave 到达放行（跨 wave）、workgroup/device 作用域在 kernel 启动面的语义（同步原语在 30 章；`sync-primitives.md` §3） |
| M37.2 | T37.8 | ② 主体 | 计算内建 id（THREAD/GROUP/WAVE）编译器前端映射与 kernel 侧暴露（内建寄存器面在 15/17 章；本章收口计算侧） |
| M37.2 | T37.9 | ② 主体 | 计算统计口径：每 dispatch wave 数、每 workgroup 波数、occupancy 占用观测（软观测，ADR-038；31 章统计的计算侧口径） |
| M37.3 | T37.10 | ② 主体 | 异步计算：计算队列与图形队列并发提交、计算 → 图形 fence 依赖、`SIGNAL/WAIT` 值序（队列机制在 32 章；本章收口计算/图形互操作的入口语义） |
| M37.3 | T37.11 | ② 主体 | 间接派发计算侧：`DISPATCH_INDIRECT` 从显存读 `{gridX,Y,Z}`，与直接 dispatch 结果等价（间接层在 33 章；本章收口计算侧场景 S19） |
| M37.3 | T37.12 | ② 主体 | 计算查询与统计：dispatch 计数、timestamp 单调、查询可作 GPU 侧谓词（查询单元在 34 章；本章收口计算口径） |
| M37.3 | T37.13 | ③ 接入 | 全阶段 UAV kernel 写回收口接入 S19 `case-4`：kernel 经 UAV 视图写全局/图像内存（视图在 20 章、格式转换表在 26 章；本章收口 kernel 侧主路径） |
| M37.4 | T37.14 | ④ 验收 | 直连 HLI 计算入口接 S18/S19：`A37` 全绿（S18/S19 双策略硬判据一致、直连 HLI 计算入口跑通、受影响 S 场景回归）；双入口等价（原 35.5 计算驱动吸收点）归 stage3 42（A42） |

### 子里程碑（本章 ≤ 6 任务/块）

| 子里程碑 | 范围 | 承载任务号 |
|---|---|---|
| M37.1 计算通路与派发 | 入口契约、参数打包校验、Compute 引擎派发、uniform 装载、直连 HLI 接入 | T37.1~T37.5 |
| M37.2 计算调度与协作收口 | 三维映射驱动侧、组内 barrier 作用域、内建 id、统计口径 | T37.6~T37.9 |
| M37.3 计算系统收口 | 异步计算、间接派发、查询统计、UAV 写回 | T37.10~T37.13 |
| M37.4 端到端验收 | 直连 HLI 计算入口接 S18/S19、A37 | T37.14 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D37.1 | 计算入口相关头文件独立编译；`KernelArgsLayout` 六条规则单测全绿（正常 + 非法边界，含 `paramBufByteSize` 4096 通过 / 4097 拒绝）；D13-a/b/c（block 513 / LDS 128 KiB / grid 2^20+1）记录期 `InvalidArgument` 单测全绿 | L1 契约 |
| D37.2 | 每 `{ptr,bytes}` 按 `paramOffsets` 写入 GART；重叠 / 越区 / 无效参数区 / 未映射 → 报错且命令流不含 `SET_KERNEL_ARGS` | L1 契约 |
| D37.3 | `DISPATCH` 能驱动 Compute 引擎产出与 29~31 章单测一致的结果；`sa` 非空 | L1 契约 |
| D37.4 | `DISPATCH` 后 VGPR uniform 区 `v[56..67]` 内容 = `paramBufVa/grid/block`（快照比对） | L1 契约 |
| D37.5 | 直连 HLI 计算入口下 `scene_s18_compute_basic` 探针通过（整数/索引零容差；含 fp32 归约的 case 按 ≤1 ULP + 顺序断言） | L2 + A37 |
| D37.6 | 三维 block/grid 的 blockIdx/threadIdx=ref_model；grid 边界部分 workgroup 有效、越界 work-item 不写任何内存 | L2 |
| D37.7 | block > warp（如 64 = 2 wave / 256 = 8 wave，对齐 S19 `case-0`）跨 wave barrier 只在全部 wave 到达后放行（原子计数断言）；workgroup/device 作用域语义确定 | L2 |
| D37.8 | 内建 `THREAD/GROUP/WAVE` id 与 ref_model 一致；编译器产物不产生非法布局 | L2 |
| D37.9 | 统计输出可观测（wave 数/占用），只作软观测、不进硬判据 | L1 |
| D37.10 | 计算队列与图形队列并发提交，图形在 `WAIT` 前不读计算产物；timeline 值序正确 | L1 + L2 |
| D37.11 | `DISPATCH_INDIRECT(参数=直接 dispatch)` 与直接 dispatch 结果逐点零容差一致 | 横切 |
| D37.12 | 计算 dispatch 计数值正确、timestamp 单调不减；查询可作谓词回读 | L1 |
| D37.13 | kernel UAV 写回按 `(x,y,format)` 与 20 章 image 描述符 + 26 章格式转换表逐字节一致（S19 `case-4`）；不依赖 26 章 OM 写回探针 | 横切 |
| D37.14 | A37 全绿：S18/S19 在 `serial`/`parallel` 硬判据逐项一致（整数/索引零容差；fp32 归约 ≤1 ULP + 归约顺序断言）、直连 HLI 计算入口跑通、受影响 S 场景不退化（双入口等价归 stage3 42 / A42） | L1 + L2 + 横切 |

---

## 7. 场景与判据（A37）

**绑定场景：S18 `scene_s18_compute_basic`、S19 `scene_s19_compute_memory`（本章为主验收章；S10 组内归约与原子、S11 并行、S12 多引擎、S16 所有权为回归项）。**

| 项 | 内容 |
|---|---|
| 输入 | S18：1D/2D/3D grid + block、kernel 参数、越界 work-item；S19：LDS / 原子 / barrier + 间接 dispatch + 异步计算；两者跑 `api_backend=hli`（直连）；（`api_backend=compute` 计算驱动入口归 stage3 42 双入口等价验收） |
| 产物 | kernel 启动日志、命令包序列、RT/UAV/参数缓冲读回、S18/S19 探针表 |
| 主判据 | S18/S19 的 blockIdx/threadIdx/越界/计数/shared/barrier/内存序 = ref_model（整数/索引零容差）；fp32 归约 ≤1 ULP + 归约顺序断言（`scene-specs.md` §1.2）；间接 dispatch 与直接等价；计算→图形 fence 后结果可见 |
| 本章专有判据 | ① S18/S19 在 `serial` / `parallel` 下硬判据逐项一致（整数/索引零容差；fp32 归约 ≤1 ULP + 归约顺序断言）；② 直连 HLI 计算入口跑通（双入口等价归 stage3 42 / A42 主验收）；③ 共用单元无图形/计算特化分支；④ 六条参数校验全部命中非法样例返回确定错误且不下发，D13-a/b/c 与组驻留（M-03，16 waves/CU，B-037）、参数区 4 KiB（B-041）、grid ≤ 2^20（B-038）上限断言命中 |
| 回归项 | A1~A36 全绿；S10（波内/组内归约计算侧）、S11（并行）、S12（多引擎）、S16（所有权）在计算线接入后不退化 |
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

本章按 `review-sync-20261121.md` §2.4–2.5 重划：最大 workgroup 512 threads；indirect ABI 28 bytes；本章只交付硬件 Compute 通路与直连 HLI 验收。参数打包、stream/event、驱动侧 indirect/UAV/query API 的重复任务/DoD 标记 `SUPERSEDED → stage3 39–41`。
