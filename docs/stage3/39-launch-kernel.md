# 第 39 章 LaunchKernel——kernel 参数打包与首次派发

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章是 **stage3 计算驱动开发线（38~42）** 的第二章。在第 38 章 module/function 对象骨架之上，实现计算驱动的**参数打包与首次派发**：应用 `kernel(args...)` → 驱动收集 `ArgPack{ptr,bytes}[]` → 六条校验 → 写 GART `HOST_VISIBLE|HOST_COHERENT` 参数缓冲 → 生成 `KernelArgsLayout` → `IHliPipeline(kernel).SetKernel` → `IHliCommandList.Dispatch(grid,block)` → 队列提交；Compute 引擎派发主路径（CP 解析 `DISPATCH` → Compute 引擎 → 读 `SET_KERNEL_ARGS` → 装载 `paramBufVa/grid/block` 到 VGPR uniform 区 `v[56..67]` → workgroup 展开到 `se`）。以 Demo C2 `vector_add` 双入口（直连 HLI / 计算驱动）等价验证为核心验收。
> 设计权威：参数打包 ABI 与六条校验见 `../design/compute-driver.md` §5；`KernelArgsLayout` 见 `../architecture-design.md` §3.8；`DispatchDesc` 与 D-01~D-13 见 §3.11；映射公式与 uniform 区见 §3.12、`../design/isa-abi.md` §3；错误码映射见 §3.9；WA-01 见 `../architecture-design.md` §7.9，计算通路见 `../design/compute-line.md` §1/§2。
> 编号纪律：本章不新增编号；复用 `DISPATCH=0x04`、`SET_KERNEL_ARGS=0x0F/0x0002`（ADR-043/047）；不新增 HLI 接口（ADR-032）；开工前对照 `../design/bit-budget.md` 审计（ADR-045）。

---

## 1. 目标与非目标

### 1.1 目标

1. **参数打包 ABI 落地**：`ArgPack{ptr,bytes}[]` → 按签名推导偏移 → 写入参数缓冲 → `KernelArgsLayout`；六条校验全部命中非法样例（对齐 / 重叠 / 越界 / 越区 / 未映射）返回确定错误且**不产生包**。
2. **`LaunchKernel` 主路径**：`SetKernel → Dispatch(grid,block) → 队列提交`；Compute 引擎把 `paramBufVa / grid / block` 装载到 VGPR uniform 区 `v[56..67]`。
3. **双入口等价**：同一 kernel（Demo C2 `vector_add`，WA-01）经计算驱动入口与直连 HLI 手动 Dispatch 命令流的关键操作与参数语义一致，探针零容差一致。
4. **首次端到端计算通路**（第 37 章通路契约 + 本章驱动面的闭合）：从 kernel 源码/ISA → 参数 → dispatch → 内存写回 → fence → 回读。

### 1.2 非目标

- 最小 Stream 与首次派发在本章闭合。多 Stream、Event 与异步互操作在第 40 章扩展。第 41 章接入协作、间接与 UAV，第 42 章汇总双入口矩阵。
- 不做 `LaunchWithParamBuffer`（二期）；不做动态并行；不做真实 occupancy 调优（只软观测 wave 数）。
- 不为计算特化硬件单元；不新增 HLI 接口（ADR-032）。

---

## 2. 单元框图

```
apps（Demo C2 vector_add）
   │  kernel(a_vec, b_vec, c_vec, N)        // ArgPack[] 
   ▼
umd/compute::LaunchKernel(stream, fn, grid, block, args[])
   ├─ 校验 §3.2 六条 → 失败不得产生 SET_KERNEL_ARGS/DISPATCH
   ├─ 写 GART HOST_VISIBLE|HOST_COHERENT 参数缓冲 → KernelArgsLayout
   ├─ IHliPipeline(fn).SetKernel
   └─ IHliCommandList.Dispatch(grid, block) → 队列提交（第 32 章）
   ▼
CP 解析 DISPATCH(0x04) → Compute 引擎（薄壳）
   ├─ 读 SET_KERNEL_ARGS{paramBufVa,size,paramVgprBase}
   ├─ 装载 paramBufVa/grid/block → VGPR uniform 区 v[56..67]（isa-abi §3）
   └─ workgroup 展开 → IShaderArray::Dispatch（29~31 共用单元，ADR-032）
   ▼
fence / 回读 → 探针比对（计算驱动入口 vs 直连 HLI）
```

说明：

- 派发主路径与第 37 章 37.1 完全一致；本章的增量是**驱动侧**的参数打包与命令流构造，并在驱动 / HLI 创建侧集中执行六条校验（D-07 引用 §3.8，不自造第二套）。
- 参数缓冲与 scratch 共用第 3/11 章 VA 与三堆（§3.13 Global/Param 行），零拷贝共享。

---

## 3. 输入输出契约

- **最小 Stream**：在 `umd/compute` 为已创建 Context 的一个 Compute 队列提供 `StreamCreate`、`StreamSynchronize`、`StreamDestroy`。Stream 与队列 1:1，`LaunchKernel` 经该 Stream 提交。同步等待提交时快照的队列目标值，并返回其终态。存在未完成提交时销毁返回 `Busy`，显式同步后销毁。第 40 章扩展同一接口。
- **参数归属**：第 20 章校验预构造参数缓冲的记录契约，第 37 章消费参数并验证 uniform 装载，本章从应用参数生成缓冲及布局。提交成功后，参数资源保持有效至完成或失败回收。
### 3.1 kernel 参数（权威：`architecture-design.md` §3.8 + `compute-driver.md` §5）

```cpp
struct KernelArgsLayout {
    GpuVAddr paramBufVa;           // 参数区 VA（HOST_VISIBLE|HOST_COHERENT）
    u32      paramBufByteSize;
    u32      paramVgprBase;        // 默认 56（uniform 区起点 v[56]，合法域 v[56..67]）
    std::vector<u32> paramOffsets; // 每参数在参数区内的字节偏移
};
```

### 3.2 六条输入校验规则（驱动创建侧集中执行；违反返回 `InvalidArgument` / `NotMapped` / `NotResident`，**不下发硬件**）

1. `paramBufVa` 已映射且驻留（`HOST_VISIBLE | HOST_COHERENT`）；
2. `paramBufByteSize > 0` 且 ≤ `kMaxParamBufBytes` = 4 KiB（B-041，独立契约常量）；`v[56..67]` 只承载 `paramBufVa/grid/block` 描述符，不参与参数容量计算（与 VGPR uniform 分区容量解耦）；
3. `paramOffsets` 严格递增且互不重叠，且每条 `offset + size ≤ paramBufByteSize`（size 由驱动按签名推导）；
4. 参数按类型自然对齐：标量 4 B、`float2/vec2/int2` 8 B、`float4/vec4` 16 B；
5. `paramVgprBase` 默认 56，合法域 `v[56..67]` 且不与 ABI 分区冲突（RT `v[44..55]`、临时量 `v[76..]` 拒绝）；
6. 只做静态结构校验，不断言取值语义（`DISPATCH` 包生成前完成）。

### 3.3 DispatchDesc（权威：`architecture-design.md` §3.11）

- 复用 `compute::DispatchDesc{kernelVa, paramBufVa, paramBufByteSize, paramVgprBase, paramOffsets, grid, block, ldsStaticBytes, sgprCount, vgprCount, subgroupSize, indirectArgsVa(=0), flags}`；
- D-01~D-13 全部生效：block 各维 ≥1 且积 ≤512（允许非 32 倍数，末波 LaneMask，M-02）；grid 各维 ≥1；`subgroupSize=32`；包不得携带 IA/光栅/OM 状态（D-08）；`flags` 未知位必须 0（D-09）等。
- D-13 组级静态可调度性（§3.11）：`wavesPerGroup ≤ 16`（B-037）、`ldsStaticBytes ≤ 64 KiB`（B-036）、grid 总量 ≤ 2^20（B-038）；**超限记录期 `InvalidArgument` 拒绝、不产生包、不进入排队**（M-06）；仅"需求可满足但当前被占"才按 ADR-038 排队。
- 校验在**记录 / 创建期执行**：失败 `InvalidArgument`，不产生包、不下发硬件；执行期只做确定性中止（§3.3 错误流）。

### 3.4 直连 HLI 对照入口

```cpp
// 直连 HLI（等价基准）：同一 kernel 由场景直接构造命令流
ihliPipeline.SetKernel(fn);                       // 或 IHliPipeline(kernel)
cmdList.Dispatch(grid, block);                    // 手动 Dispatch
queue.Submit(cmdList, fenceVal); wait(fenceVal);
// 计算驱动入口：按第 42 章 §3.1 比对关键命令语义
```

---

## 4. 内部数据流或状态机

### 4.1 LaunchKernel 数据流

```
LaunchKernel(stream, fn, grid, block, args[])
  ├─ 校验：KernelArgsLayout 六条 + DispatchDesc D-01~D-13
  ├─ 分配/复用 HOST_VISIBLE|HOST_COHERENT 参数缓冲
  ├─ 按签名推导 paramOffsets → 逐参数 memcpy（host→缓冲）
  ├─ SetKernel(fn) → cmdList.Dispatch(grid, block)
  └─ stream 提交 → IHliQueue.Submit(SIGNAL(fenceVal)) → 回读/同步
```

### 4.2 Compute 引擎派发主路径（GO 与直连 HLI 共用，第 37 章 37.3）

```
CP 解析 DISPATCH(0x04) → Compute 引擎
  → 读 SET_KERNEL_ARGS{paramBufVa, size, vgprBase}
  → 装载 paramBufVa / grid / block → VGPR uniform 区 v[56..67]
  → 三维 grid/workgroup 映射（blockIdx/threadIdx，越界 work-item 不写，M-01/M-02）
  → 每 workgroup 生成 wave 投递 IShaderArray::Dispatch
  → LDS/原子/barrier（第 30 章共用）→ 内存写回 → fence/查询
```

### 4.3 错误状态

- 结构 / 边界失败（未对齐、重叠、越界、越区、`paramBufByteSize` > 4 KiB）→ `InvalidArgument`（映射 `MUSA_ERROR_INVALID_VALUE`）；
- 映射 / 驻留失败 → `NotMapped` / `NotResident`（映射 `MUSA_ERROR_INVALID_VALUE`，执行期走设备丢失类）；
- 任一失败均不产生 `SET_KERNEL_ARGS` / `DISPATCH` 包；执行期越界由 GPU 侧确定性中止（不做语义修复）。

---

## 5. 任务清单

> 任务编号 `T39.1~T39.5`，四段式。

| 子里程碑 | 任务 | 段 | 内容 |
|---|---|---|---|
| M39.1 | T39.1 | ① 契约 | 先接通已有 Compute 队列的最小 Stream 生命周期，再固化 `packArg` / `KernelArgsLayout` 接口及六条校验 |
| M39.1 | T39.2 | ② 主体 | `ArgPack[]` → 参数缓冲写入与偏移推导（对齐/簇对齐/重叠检测；失败 `InvalidArgument` 且命令流不含 `SET_KERNEL_ARGS`/`DISPATCH`） |
| M39.1 | T39.3 | ② 主体 | `LaunchKernel` 在最小 Stream 上记录 SetKernel → Dispatch 并提交，保活参数缓冲至终态；复用第 37 章 uniform 装载 |
| M39.2 | T39.4 | ③ 接入 | Demo C2 `vector_add` 双跑直连 HLI 与计算驱动入口，比较关键命令语义和探针 |
| M39.2 | T39.5 | ④ 验收 | A39：C2 双入口、六条参数非法样例、S18 `case-3` 与同输入 `case-1` 对照、`case-0/1` 回归 |

### 子里程碑

| 子里程碑 | 范围 | 承载任务号 |
|---|---|---|
| M39.1 最小 Stream 与首次派发 | 单 Compute 队列、参数打包与校验、LaunchKernel | T39.1~T39.3 |
| M39.2 双入口与验收 | Demo C2 双跑、A39 | T39.4~T39.5 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D39.1 | 最小 Stream 头文件独立编译；单队列绑定、重复绑定拒绝、pending 销毁返回 `Busy`、显式同步后销毁；参数接口头文件独立编译，六条校验含参数缓冲 4096/4097 B 边界，D13-a/b/c 记录期单测全绿 | L1 契约 |
| D39.2 | 每 `{ptr,bytes}` 按 `paramOffsets` 写入 GART；重叠/越区/无效参数区/未映射 → 报错且命令流不含 `SET_KERNEL_ARGS` | L1 契约 |
| D39.3 | `LaunchKernel` 提交发出正确的 `DISPATCH`；延迟执行仍读到提交时的参数，终态后释放；`sa` 非空，`v[56..67]` 与第 37 章 `paramBufVa/grid/block` 探针一致 | L1 + L2 |
| D39.4 | C2 整型逐字节一致，fp32 与 CPU 参考 ≤1 ULP，同输入两入口探针零容差；关键包按第 42 章 §3.1 等价 | 横切 |
| D39.5 | A39 全绿：六条非法输入产生确定错误且无对应包；S18 `case-3` 与 `case-1` 同输入探针零容差及关键包等价；`case-0/1` 回归 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A39）

**绑定：Demo C2 `vector_add`（WA-01，本阶段主判据）；S18 `case-3`（经计算驱动，= case-1 同参数）为首次双入口命令流等价入口；case-0/1（1D/2D）作为前序能力回归。S18/S19 双入口矩阵验收在第 42 章。**

| 项 | 内容 |
|---|---|
| 输入 | `vector_add` kernel（1D grid/block；`a/b/c` 缓冲 + N）；六条校验非法样例集；`api_backend ∈ {hli, compute}` |
| 产物 | kernel 启动日志、命令包序列、参数缓冲布局、`v[56..67]` 快照、探针表 |
| 主判据 | C2 整型逐字节一致，fp32 与 CPU 参考 ≤1 ULP，同输入驱动与 HLI 探针零容差；关键包按 §3.1（第 42 章）等价 |
| 本章专有判据 | 六条校验全部命中非法样例返回确定错误且不下发；`v[56..67]` = paramBufVa/grid/block（快照比对）；不新增配置键（`api_backend=compute` 复用） |
| 回归项 | R39：S18 case-0/1（1D/2D）在接入后不退化；D13-a/b/c 拒绝边界回归；未列明历史 A 门禁不自动阻断 |

> 场景输入与探针单份规约见 `../design/scene-specs.md` §3（S18 节）与 §4（第 39 行）。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 参数偏移与命令流顺序只由签名与 grid/block 决定；两次运行一致；无时间输入 |
| 时间无关 | 装配不依赖墙钟/帧序 |
| 可观测 | `--trace=compute`：参数布局、命令包序列、`v[56..67]` 快照；只供人工 |
| 契约影响 | `KernelArgsLayout` 六条（§3.8）与 D-01~D-13（§3.11）为冻结项，变更走 ADR 与 `roadmap.md` §7.4 |
| 简化登记 | 一期不做 `LaunchWithParamBuffer`；不做 16 B 以上对齐的复杂布局（WA-01~12 覆盖） |
| 计算兼容 | 共用单元按包类型派发，无"图形专用/计算专用"分支（ADR-032） |
| 配置与回退 | 不新增配置键；入口复用 `api_backend=hli / compute`；`exec_mode` 双策略 |
| 调试钩子 | 参数快照、命令流 dump、uniform 区 dump；只供人工，不进硬判据 |
| trace 契约 | `--trace` 不保证字段稳定、不机器解析、不进判据 |
| 契约变更 | 六条校验/偏移推导走 ADR，回归 S18 与受影响 S 场景 |
| 校验归属 | L1（结构/状态）+ L2（vector_add 探针）+ 横切（双入口命令流等价） |
| 双策略 | `vector_add` 在 `serial`/`parallel` 探针零容差一致（S18 回归项） |

---

## 9. 简化登记

| 项 | 状态 | 替换章 / 备注 |
|---|---|---|
| `LaunchWithParamBuffer` | 二期（应用直给参数 VA） | `compute-driver.md` §5；第 42 章"后续补齐" |
| 动态并行 | 范围外 | 第 37 章非目标 |
| 真实 occupancy 调优 | 只软观测 wave 数 | ADR-038、`compute-line.md` §2 |

---

## 10. 待定项

- kernel 参数布局的显式寄存器直传（部分标量直接进 SGPR uniform，减少参数缓冲读取）——一期统一走参数缓冲，性能优化二期。
- 第 40 章扩展多个 Stream 和 Event，沿用本章的 `LaunchKernel` 提交路径。

---

## 2026-11-21 评审同步说明

block 容量上限见 [架构设计 §3.11](../architecture-design.md)。本章交付应用参数打包、最小 Stream 和首次 LaunchKernel。硬件指令执行与 wave 生成属第 37 章。
