# 计算驱动对象模型与 ABI — 独立模块设计

> 项目目标、范围、阶段与交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

> 定位：定义 **stage3 计算驱动开发线（新章 38~42，`docs/stage3/`）** 的驱动专属设计——`umd/compute` 的 MUSA 风格对象模型（context / module / function / stream / event）、`MUSA_MODULE_HDR` 二进制布局与装载、kernel 参数打包 ABI、stream/event 同步、生命周期与错误码映射。它是 stage3 各章文档的设计权威。
> 落地章：38（对象骨架与 Module 装载）、39（LaunchKernel 与参数打包）、40（Stream/Event 与异步）、41（组内协作/间接/UAV 写回）、42（双入口等价与验收）。
> 权威来源：`architecture-design.md` §3.8（KernelArgsLayout 六条校验）、§3.9（错误契约与 MUSA 错误码映射）、§3.11（DispatchDesc 与 D-01~D-12）、§3.12（映射公式 M-01~M-08）、§3.13（地址空间与作用域 S-01~S-06）、§3.10（生命周期）；`compute-line.md` §2/§7（对象粒度与 HLI 映射）；ADR-009 / ADR-013 / ADR-022 / ADR-030 / ADR-032 / ADR-039 / ADR-041 / ADR-047。
> 与 `compute-line.md` 的分工：计算最小通路、workgroup/wave/CU 映射、内存地址空间与作用域、共用单元复用表、WA 业务目录保留在 `compute-line.md`；**对象模型二进制细节、参数打包 ABI、stream/event 同步、生命周期与错误码**为驱动专属，迁到本文件。

---

## 1. 定位与模块归属

- `umd/compute` 是**唯一计算专属目录**（ADR-009），与 `umd/graphics/api` 并列；计算驱动**只依赖 HLI 与 `umd/os`**（ADR-022），不得直接调用硬件单元（访问硬件唯一路径是"写环 → doorbell → CP 取指"，与图形 UMD 相同）。
- **不新增 HLI 接口**：dispatch 走既有 `DISPATCH` 包与 compute pipeline（`IHliPipeline.SetKernel`、`IHliCommandList.Dispatch/IndirectDispatch`）；队列与同步走第 32 章原语（`IHliQueue.Submit(SIGNAL/WAIT)`、timeline）。不新增 `CreateComputePipeline` 平行入口（ADR-032）。
- **不新增编号**：不新增包类型 / opcode / stateId / MMIO / 配置键；`DISPATCH=0x04`、`SET_KERNEL_ARGS=0x0F/0x0002`、`DISPATCH_INDIRECT`、`BARRIER=0x0E`、`PREDICATE=0x0D` 均复用既有登记（ADR-043）；开工前对照 `bit-budget.md` 审计（ADR-045）。
- **一期可交付**（ADR-025 高级一期）：Driver API（context / module / function / stream / event）+ Runtime API 核心子集（alloc / memcpy / launch）；`LaunchWithParamBuffer`、动态并行、API/二进制兼容、真实 occupancy 调优均二期 / 范围外（只留 capability bit 或文档登记）。
- **能力只加不改**：subgroup / shared / atomic / 三维 workgroup 等计算专属能力在 29~31 章以图形+计算共用单元落地，本模块不复制实现、不给共用单元开计算特化分支（ADR-032）。

## 2. 对象模型

| 对象 | 职责 | 一期属性 | 映射到 HLI / KMD |
|---|---|---|---|
| `Context` | 设备、内存域、module/stream 的容器与生命周期根 | 一个 HLI device 支持 N 个 Context；每个 Context 绑定唯一 KMD context/ASID（Context:ASID=1:1）。销毁不隐式同步；仍有子对象或硬件引用时返回 `Busy` | `IHliDevice` + 资源表 |
| `Module` | 一份 ISA blob（自有头 + 代码段 + 符号表） | 常驻 GART（`HOST_VISIBLE`）；不引入 ELF/COFF；不做重定位 | `IHliPipeline`（compute）+ `umd/os` 内存分配 |
| `Function` | module 内一个 kernel 入口 + 参数布局 | `codeBaseVa + entryOffset`；分支用 pc 相对偏移，加载不需地址修补 | `IHliPipeline` + `KernelArgsLayout` |
| `Stream` | 顺序提交队列 | 1:1 映射一个 HLI 队列；`Synchronize` 显式 drain；有 in-flight 时 Destroy 返回 `Busy`；Context 销毁时仍有 stream 子对象返回 `Busy`；优先级不可变 | `IHliQueue` |
| `Event` | 跨队列完成标记 | 自有 timeline semaphore 与单调 `u64` value；Record 绑定发信号的 stream，Wait 可在任意 stream 上等待已记录值 | Event timeline semaphore |

**状态机（简化为三态，能定位即可）**：

- `Module`：`Loaded`（blob 解析通过）→ `Resident`（GART 常驻并可被 CP 访问）→ `Bound`（至少一个 Function 被 `SetKernel` 引用）；销毁或 context 销毁回到释放。
- `Function`：`Resolved`（符号表解析出入口）→ `Launched`（至少一次 `Dispatch` 引用）；入口解析失败为错误终态（`InvalidIsa` / `NotMapped`）。
- `Stream`：`Idle` → `Pending`（有 in-flight 提交）→ 完成回到 `Idle`；`Event` 状态由其自有 semaphore 的完成值与最近记录值比较（未达 = NotReached，达到 = Reached）。

**生命周期（权威 `architecture-design.md` §3.10，ADR-041）**：

1. 创建顺序：`Context → Module → Function → Stream/Event`，HLI 侧先建 device → queue；销毁严格逆序。
2. 驱动对象销毁只减**用户引用**；底层 GPU 内存仅在"用户引用 AND 硬件提交引用双归零"后释放（硬件引用由 fence 完成回调释放），销毁不做隐式同步。
3. 设备丢失 / 引擎复位期间（第 36 章）：被丢弃命令的硬件提交引用即刻归零；驱动对象销毁必须安全且幂等（底层内存已回收时不得二次 free）。

## 3. `MUSA_MODULE_HDR` 二进制布局

module blob 布局（little-endian，字段 4 B 对齐；`entryOffset`/`codeBaseVa` 一律 4 B 对齐）：

```
偏移   大小  字段            说明
0x00   u32   magic          'MUSA'（0x4D555341）
0x04   u32   version        一期 = 1；不支持时返回 NotSupported
0x08   u32   codeOffset     代码段相对 file base 的字节偏移
0x0C   u32   codeBytes      代码段字节数
0x10   u32   symTabOffset   符号表区相对 file base 的字节偏移
0x14   u32   symTabCount    符号条目数
0x18   u32   reserved0      必须 0（非法返回 InvalidIsa）
0x1C   u32   reserved1      必须 0
0x20   ...   code[]         代码段（ISA 编码，`common/isa.h`）
...    ...   symtab[]       符号表条目数组（见下）
```

符号表条目（每项 28 B，`symTabCount` 项）：

```
偏移  大小  字段
0     u32   nameOffset    名称字符串相对 file base 偏移（NUL 结尾，≤127 B）
4     u32   nameLen       名称长度（不含 NUL）
8     u32   entryOffset   UTF-8 名称 → module 代码段内入口偏移（相对 codeOffset）
12    u32   paramBytes    该 kernel 参数区字节数（供打包校验）
16    u32   vgprCount     资源声明，仅进 occupancy 预算（ADR-038）
20    u32   ldsBytes      静态 LDS 需求声明
24    u32   flags         位 0 = 组内使用 barrier；位 1 = 需 device 作用域；其余必须 0
```

**装载规则**：

- 校验顺序：magic → version → reserved=0 → `codeOffset+codeBytes ≤ blobSize` 且各偏移 4 B 对齐 → 逐符号 `entryOffset < codeBytes`、`nameOffset/nameLen` 在 blob 内。任一失败返回 `InvalidIsa` / `NotMapped`，**不产生任何 HLI 资源**。
- module 装入 GART `HOST_VISIBLE` 常驻堆（第 11 章确定性分配器，VA 由 `translation.seed` 复现）；CP 通过 `kernelVa = codeBaseVa + entryOffset` 访问，不做重定位（分支为 pc 相对偏移（第 16 章））。

## 4. 符号表解析与 Function 入口

- `Function` = `Module` + 符号命中（`name` 匹配）+ 参数布局。解析后得到 `DispatchDesc.kernelVa = codeBaseVa + entryOffset`。
- 非法输入：符号未命中 → `InvalidIsa`；`kernelVa` 未映射 → `NotMapped`；`entryOffset` 未 4 B 对齐或越出代码段 → `InvalidIsa`（与 §3.11 D-06 同口径）。
- 重复解析同一符号返回同一 `Function`（幂等）；`Function` 生命周期跟随 `Module`。

## 5. kernel 参数打包 ABI

```
应用: kernel(args...)           驱动（umd/compute）
  └─> 按签名收集 ArgPack{ptr, bytes}[]   └─> 分配 HOST_VISIBLE|HOST_COHERENT 参数缓冲
                                          └─> 按 §3.8 六条校验 → 生成 KernelArgsLayout
                                          └─> SET_KERNEL_ARGS{paramBufVa,size,paramVgprBase}
                                              + DISPATCH{grid, block}
```

- 字段：`KernelArgsLayout{ paramBufVa, paramBufByteSize, paramVgprBase(=56), paramOffsets[] }`（`architecture-design.md` §3.8）。
- **六条校验规则**：完全复用 `architecture-design.md` §3.8 规则 1~6，**本节不重复定义**（§3.11 D-07，禁止两处各写一套口径）；失败返回确定错误、**不产生包、不下发硬件**。本节只补**本地增量**：
  1. 符号表字段参与校验：`paramBytes`（§3 符号条目）作为打包后 `paramBufByteSize` 的期望值对照；`ldsBytes`/`vgprCount` 进 D-13 静态可调度性预算（不新增独立规则）；
  2. `LaunchKernel(args[])` 按签名收集 `ArgPack{ptr, bytes}`，size 推导规则以 §3.8 规则 3 为准；
  3. 参数容量上限引用契约常量 `kMaxParamBufBytes`（§3.8 规则 2 / `bit-budget.md` §7.1 B-041），本节不另设值。

- 写入后 Compute 引擎把 `paramBufVa / grid / block` 装载到 VGPR uniform 区 `v[56..67]`（`isa-abi.md` §3 权威分区），kernel 按 `paramOffsets` 读取。
- 二期 `LaunchWithParamBuffer(GpuVAddr)`：应用直接给参数 VA，校验收敛为只做映射/驻留与对齐检查，不在此期落地（登记于第 42 章"后续补齐"）。

## 6. Stream 与 Event

- `Stream` = 1:1 映射一个 HLI 队列（`IHliQueue`）；stream 内提交顺序 = 命令流顺序；不同队列可各自创建 stream，但同一队列不重复包装；优先级创建时声明且不可变。
- `StreamSynchronize` 是唯一阻塞 drain 操作；`StreamDestroy` 存在 in-flight 提交时返回 `Busy`，不隐式同步；`ContextDestroy` 存在任一子对象（含 Stream）或硬件引用时返回 `Busy`。
- `Event` 是拥有独立 timeline semaphore 的对象，不复用任何队列的 fence/timeline 值域：
  - `Record(stream)`：为该 Event 预留严格递增的 `u64` 值，并在该 stream 命令流插入 `SIGNAL(eventSemaphore,value)`；
  - `Wait(stream)`：快照 Event 最近一次已记录值，在该 stream 插入 `WAIT(eventSemaphore,value)`；未记录过的 Event 返回 `InvalidArgument`。等待可跨队列；
  - `Query()`：非阻塞读取 semaphore 的已完成值，与最近一次已记录值比较；未达到返回 `NotReached`，不隐式同步。
- 异步计算语义：计算队列与图形队列以 `SIGNAL/WAIT` 定义依赖（不依赖提交次序）；计算→图形可见性 = 队列家族所有权转移（第 32 章）+ fence + device barrier（§3.13 一致性矩阵"图形 ↔ 计算"行）。

## 7. Runtime API 面（alloc / memcpy / launch）

| API | 本项目实现 | 映射 |
|---|---|---|
| `ContextCreate/Destroy` | context 生命周期（§2） | `IHliDevice` + KMD context |
| `ModuleLoad(blob)` | §3 装载 | GART 常驻资源 |
| `FunctionGet(module, name)` | §4 符号解析 | `IHliPipeline` |
| `StreamCreate/Destroy` | stream 生命周期 | `IHliQueue` |
| `EventCreate/Record/Wait/Query` | §6 | timeline SIGNAL/WAIT |
| `Alloc / Free` | HOST_VISIBLE/DEVICE 内存分配 | 第 11 章堆 + 确定性分配器 |
| `MemcpyHtoD / DtoH` | 参数与结果搬运 | Copy 引擎（第 13 章，`sa=nullptr`） |
| `LaunchKernel(stream, function, grid, block, args[])` | §5 打包 → Dispatch | `IHliCommandList.Dispatch` |

## 8. 正确性、确定性与非目标

- **正确性关键词**：workgroup 是调度单位、wave 是执行单位（`compute-line.md` §1）；内存作用域与 barrier 是正确性核心（`sync-primitives.md` §3、§3.13）。
- **确定性**：`tid/gtid/waveId/workgroupId` 只由 grid/block 决定（M-01）；归约顺序写死（lane 升序或原子累加）；随机固定种子按 `gtid` 升序消费（M-08）。
- **非目标**：MUSA/CUDA API 或二进制兼容、动态并行、可变 subgroup、周期精确时序、真实 occupancy 调优、grid-wide barrier（协 kernel）——均二期 / 范围外。
- **数据模型一致性（S-04）**：计算 buffer 写回直写 memory、image 写回复用第 21 章 tiling 编码与 `common/format` 转换，与 RT 写回共用同一实现，不复制两份（ADR-032 实现级推论）。

## 9. 错误码映射（权威 `architecture-design.md` §3.9）

| 底层 `Result` | 计算驱动（MUSA 风格，示意） | 场景 |
|---|---|---|
| `InvalidArgument` | `MUSA_ERROR_INVALID_VALUE` | 参数/句柄/数据契约非法（创建期为主） |
| `NotSupported` | `MUSA_ERROR_NOT_SUPPORTED` | 二期 / 范围外能力 |
| `NotMapped` / `NotResident` | `MUSA_ERROR_INVALID_VALUE`（执行期走设备丢失类） | GPU 地址未映射 / 未驻留 |
| `InvalidPacket` / `InvalidIsa` | `MUSA_ERROR_INVALID_DEVICE_FUNCTION` | 命令包或 ISA 非法、module/function 非法 |
| `OutOfMemory` | `MUSA_ERROR_OUT_OF_MEMORY` | GPU 内存不足 |
| `DeviceLost` / `ResetRequired` | `MUSA_ERROR_DEVICE_LOST` | 设备/引擎复位需重跑 |
| `Internal` | `MUSA_ERROR_UNKNOWN` | 未归类内部错误 |

硬规则（§3.9）：映射方向单向（底层 → 上层）、在 UMD 层一次性完成、禁止吞错/降级为成功、创建期与执行期区分、`DeviceLost` 类必须映射为设备丢失类错误。

## 10. 验收要点与契约变更规则

- **验收（stage3 A38~A42，唯一权威 `scene-specs.md` §4 38~42 行）**：
  - Demo C1 `module_probe`：`MUSA_MODULE_HDR` 逐字节一致、符号表解析正确、异常输入确定错误（A38）；
  - Demo C2 `vector_add`（WA-01）：六条校验全命中、`v[56..67]` 快照一致、整型逐字节 + fp32 ≤1 ULP 通过、>1 ULP 阻断（同序 FMA 下期望 0 ULP 是诊断信息，`scene-specs.md` §1.2）、命令流与直连 HLI 零容差一致（A39）；
  - Demo C3 `async_pipeline`（WA-11）：event 值序、非阻塞 Query、异步交叉数值等价（A40）；
  - Demo C4 `reduce` / Demo C5 `indirect_dispatch`（WA-02 / WA-12）：归约 ≤1 ULP + 顺序断言（强制）、间接等价（A41）；
  - S18/S19 经计算驱动入口与直连 HLI 入口逐值等价（A42 / A-stage3）。
- **契约变更规则**：本文件任何字段 / ABI / 校验规则若需改动，属于冻结契约（对应 `architecture-design.md` §3.8/§3.9/§3.11），**必须走 ADR 并配套 `roadmap.md` §7.4 流程**，回归 S18/S19 与受影响 S 场景；本文件每节登记"落地章、相关层间接口"以便追踪。

---

## 附：每节落位表

| 节 | 内容 | 落地章 | 相关层间接口 |
|---|---|---|---|
| §2 | 对象模型与生命周期 | 38、39、40 | §3.4、§3.10（ADR-041） |
| §3 | `MUSA_MODULE_HDR` 布局与装载 | 38 | `common/isa.h`、§3.11 D-06 |
| §4 | 符号表解析与 Function 入口 | 38 | §3.11 D-06 |
| §5 | 参数打包 ABI（六条校验） | 39 | §3.8、§3.11 D-07 |
| §6 | Stream / Event | 40 | §3.4、第 32 章 timeline |
| §7 | Runtime API 面 | 39~42 | `IHliDevice` / `IHliQueue` |
| §9 | 错误码映射 | 38~42 | §3.9 |
