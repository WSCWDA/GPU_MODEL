# 第 42 章 双入口等价与端到端验收

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章是 **stage3 计算驱动开发线（38~42，ADR-047）** 的收口 / 验收章。把 stage3 全部能力（对象骨架 / LaunchKernel / stream-event 异步 / 组内协作与间接/UAV 写回）经**计算驱动入口**（`umd/compute`，`api_backend=compute`）与**直连 HLI 计算入口**（`api_backend=hli`）双跑：S18 `scene_s18_compute_basic`（blockIdx/threadIdx、越界 work-item 不写、grid 边界部分 workgroup、wave 归约与 CPU 参考一致）+ S19 `scene_s19_compute_memory`（LDS/原子/barrier、间接 dispatch、异步计算、作用域语义、计算→图形 fence 后结果可见）。主判据：**双入口探针零容差等价（同输入、关键命令语义等价、探针一致）**；A-stage3（A38~A42）全绿；E/S 场景回归不退化；`serial`/`parallel` 双策略一致。
> 设计权威：stage3 全链路契约见 `../design/compute-driver.md`；WA 目录见 `../architecture-design.md` §7.9，计算通路见 `../design/compute-line.md` §1/§2；kernel 参数与 `DispatchDesc` 见 `../architecture-design.md` §3.8/§3.11；映射与作用域见 §3.12/§3.13；错误码映射见 §3.9；场景与判据唯一权威 `../design/scene-specs.md` §3（S18/S19）、§4（第 42 行）。
> 编号纪律：本章不新增编号（复用全量既有登记）；不新增配置键（入口选择复用 `api_backend=hli / compute`，ADR-047）；开工前对照 `../design/bit-budget.md` 审计（ADR-045）。

---

## 1. 目标与非目标

### 1.1 目标

1. **双入口贯通**：使用第 39~41 章已有接口运行本章用例矩阵，比较关键命令语义。
2. **双入口等价主判据**：S18/S19 经 `compute` 与 `hli` 两入口探针**逐值等价（零容差）**；同输入、关键命令语义等价、探针一致。
3. **A-stage3（A38~A42 汇总）全绿**：`serial`/`parallel` 硬判据零容差一致；E/S 场景回归不退化（S10/S11/S12/S16/S17 等）。
4. **错误路径回归**：六类参数非法输入、坏 module/function、已失败 Event 正确返回错误，未达成 Query 返回 `NotReached` 状态。
5. **能力档边界登记**（ADR-025）：`LaunchWithParamBuffer` 二期、动态并行 / API-二进制兼容 / 真实 occupancy 二期或范围外，仅留 capability bit / 文档登记；二期候选未获 ADR 前 `NotSupported`。

### 1.2 非目标

- 不做新功能实现：本章只做**贯通与验收**；功能缺陷一律回溯到 38~41 相应章修正。
- 不做二期 / 范围外能力（grid-wide barrier、动态并行、API 兼容等）。
- 不做性能调优判断：wave 数 / 占用只软观测，不进硬判据。

---

## 2. 单元框图

```
S18 / S19 场景
   │  api_backend = hli（直连）          api_backend = compute（计算驱动）
   ▼                                    ▼
IHliCommandList.Dispatch 手动序列     umd/compute::LaunchKernel → 同构命令流
   └───────────────┬───────────────────┘
                   ▼
        (同输入、关键命令语义等价、探针一致) → 双入口等价比对（零容差）
                   ▼
CP → Compute 引擎（薄壳）→ IShaderArray::Dispatch（29~31 共用单元）
   → LDS/原子/barrier/作用域（30）→ memory/texture 写回 + query（34）
                   ▼
A-stage3 验收表：A38~A42 全绿 + 双入口等价 + 回归不退化
```

说明：

- 直连 HLI 是双入口的判定基准。固定输入资源内容、kernel、网格形状与同步依赖，按 §3.1 比对命令语义与探针。
- S18 `case-3`（计算驱动路径，HLI 参考用 `case-1` 的等价场景配置）为本章主验收子 case；S19 `case-0..4` 全部进入双入口矩阵：作用域/barrier、gather/scatter、异步、间接 dispatch + 谓词、图形↔计算 UAV 互操作（`scene-specs.md` §3）。

---

## 3. 输入输出契约

### 3.1 双入口等价判定

| 维度 | 判据 |
|---|---|
| 命令流 | 对比操作种类、相关顺序和依赖、语义参数、作用域与失败类别。`SET_KERNEL_ARGS` 比布局和入参内容，`DISPATCH` 比 grid/block，`SIGNAL`/`WAIT` 比生产者、消费者及目标关系 |
| 探针 | S18 `case-0..3` / S19 `case-0..4` 探针表两入口**逐值零容差**一致 |
| 配置矩阵 | `api_backend ∈ {hli, compute} × exec_mode ∈ {serial, parallel}` 全组合一致 |

按两次运行中对应的资源和队列角色映射句柄。GPU VA 解析为逻辑资源与偏移，保留范围、别名与无效地址语义。timeline 值按对应对象比较初值相对增量、单调顺序、WAIT 目标绑定和失败终态。原始包作为定位材料。CPU 参考的数值容差沿用场景规定，同一输入的两入口硬判据保持零容差。

### 3.2 错误路径映射（权威：`architecture-design.md` §3.9 + `compute-driver.md` §9）

| 非法输入样例 | 底层 `Result` | 计算驱动错误码 |
|---|---|---|
| 参数六条校验失败（重叠/越界/未对齐/越区/未映射） | `InvalidArgument` / `NotMapped` / `NotResident` | `MUSA_ERROR_INVALID_VALUE`（映射/驻留直传） |
| 坏 module / 未命中符号 / 越段 | `InvalidIsa` | `MUSA_ERROR_INVALID_DEVICE_FUNCTION` |
| 未达成值 `Query` | （状态枚举非错误） | 非阻塞返回 NotReached |
| 二期 / 范围外调用 | `NotSupported` | `MUSA_ERROR_NOT_SUPPORTED` |

- 硬规则：不吞错、不降级为成功、不静默降级；`DeviceLost` 类映射设备丢失错误（§3.9）。

### 3.3 能力档登记（ADR-025）

- 二期（capability bit 恒 false / 文档登记）：`LaunchWithParamBuffer`、grid-wide barrier、image 原子、async copy；调用返回 `NotSupported`。
- 范围外：MUSA/CUDA API 或二进制兼容、动态并行、可变 subgroup、真实 occupancy 调优。

---

## 4. 内部数据流或状态机

### 4.1 S18/S19 双入口跑法

```
对 (backend, exec_mode) ∈ {hli, compute} × {serial, parallel}:
  ① 构造同一输入（grid/block/kernel/参数/越界样例）
  ② compute → LaunchKernel（39/40/41 已实现的统一路径）
     hli    → 手动 Dispatch 等价序列
  ③ 提交 → fence → 回读探针（LDS 结果/全局结果/UAV/查询）
  ④ 两入口探针比对 + case-3 主验收
```

### 4.2 A-stage3 汇总

```
A-stage3 = A38（对象/装载）∧ A39（打包/派发）∧ A40（stream/event 异步）
           ∧ A41（协作/间接/UAV）∧ A42（双入口等价；A42 = 本章 D42.1~D42.4，不含自身汇总）
评定：硬判据零容差；统计软观测；任一失败定位到 38~41 对应 DoD 回溯
```

### 4.3 错误处理

- 双入口任一不一致按正确性缺陷处理（先定位是命令流差异还是执行差异）；
- 计算驱动入口错误按 `compute-driver.md` §9 映射，日志携带适用诊断字段。有效 Query 的 `NotReached` 和 `Reached` 是状态值。

---

## 5. 任务清单

> 任务编号 `T42.1~T42.5`，四段式。

| 子里程碑 | 任务 | 段 | 内容 |
|---|---|---|---|
| M42.1 | T42.1 | ① 契约 | 双入口等价验收的探针/判据单测与配置矩阵（`api_backend ∈ {hli,compute} × exec_mode ∈ {serial,parallel}`） |
| M42.1 | T42.2 | ② 主体 | 实现 §3.1 关键命令语义对照，覆盖不同 VA/句柄、timeline 起点和真实依赖差异 |
| M42.1 | T42.3 | ③ 接入 | 运行 S18 `case-0~3` 与 S19 `case-0~4` 的双入口矩阵，保留 S18 其余 case 的 HLI 回归 |
| M42.2 | T42.4 | ② 主体 | 验证参数/对象/执行失败映射及未达成 Query 状态 |
| M42.2 | T42.5 | ④ 验收 | A42/A-stage3 全绿：S18/S19 双入口逐值等价（L2 + 横切）、`serial`/`parallel` 零容差一致、E/S 场景不退化、共用单元无"图形专用/计算专用"分支审查（ADR-032） |

### 子里程碑

| 子里程碑 | 范围 | 承载任务号 |
|---|---|---|
| M42.1 双入口贯通 | 矩阵、关键命令语义对照、S18/S19 代表用例 | T42.1~T42.3 |
| M42.2 错误与验收 | 错误与待完成状态回归、A42/A-stage3 全绿 | T42.4~T42.5 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D42.1 | 等价矩阵单测全绿（4 组合 × 本章覆盖子集：S18 `case-0~3` / S19 `case-0~4`）；配置键不新增 | L1 契约 |
| D42.2 | 语义对照接受同一逻辑资源的地址/句柄重分配和时间线基值变化；拒绝参数、作用域、依赖或失败终态的变化；uniform 快照与逻辑地址对应 | 横切 |
| D42.3 | S18 `case-0~3` 与 S19 `case-0~4` 四组合硬探针零容差，含计算→图形 fence 后可见与图形→计算 UAV；S18 `case-4~12` 维持 HLI 基线 | L2 + 横切 |
| D42.4 | 非法参数、坏 module/function 和已失败 Event 命中对应错误；有效未达成 Query 非阻塞返回 `NotReached`；重复非法调用返回确定结果 | L1 |
| D42.5 | A-stage3 全绿：A38~A41 + 本章 A42（D42.1~D42.4）汇总 + E1~E5/S0~S17 全部适用回归（E0 单独 L1 自检）+ 适用场景 `serial`/`parallel` 零容差 + ADR-032 分支审查 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A42 / A-stage3）

**绑定：S18、S19（本阶段与 stage3 主验收）；回归 S10、S11、S12、S16、S17。**

| 项 | 内容 |
|---|---|
| 输入 | S18（1D/2D/3D grid+block、kernel 参数、越界 work-item，本章覆盖 `case-0~3`）；S19（LDS/原子/barrier + gather/scatter + 间接 dispatch + 异步计算 + UAV 互操作，本章覆盖 `case-0~4`）；`api_backend ∈ {hli, compute} × exec_mode ∈ {serial, parallel}` |
| 产物 | 双入口命令包序列比对、S18/S19 探针表（双入口）、A-stage3 汇总表、错误映射日志 |
| 主判据 | S18 `case-0~3`、S19 `case-0~4` 双入口关键命令语义等价，硬探针逐值零容差，`serial`/`parallel` 一致；S18 其余 case 保留 HLI 回归 |
| 本章专有判据 | 非法参数和对象、执行失败有明确错误；有效未达成 Query 返回状态；共用单元按 ADR-032 审查 |
| 回归项 | R42：E1~E5 与 S0~S17 不退化（E0 单独执行 L1 自检）；所有适用场景运行 `serial`/`parallel`，S10/S11/S12/S16/S17 重点复核；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | `LaunchWithParamBuffer`、动态并行、可变 subgroup、真实 occupancy —— 二期 / 范围外，仅留 capability bit / 文档登记 |

> 场景输入与探针单份规约见 `../design/scene-specs.md` §3（S18/S19 节）与 §4（第 42 行）；A-stage3 汇总以本节 + §4 表为唯一权威。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 双入口等价、双策略一致都只靠 id/命令流/数值判定；无时间输入 |
| 时间无关 | S18/S19 无 `uTime`/墙钟 |
| 可观测 | 双入口命令流 dump、探针表、错误映射日志、A-stage3 汇总 |
| 契约影响 | 不新增编号/配置键；映射/打包/对象契约为冻结项（ADR-043/047） |
| 简化登记 | 阶段收口：stage3 在 1~37 章之上新增对象与驱动层，硬件共用不特化 |
| 计算兼容 | 双入口等价证明共用单元无"图形专用/计算专用"分支（ADR-032） |
| 配置与回退 | 不新增配置键；`api_backend=hli / compute` 等价，无静默回退 |
| 调试钩子 | `--trace=compute`、命令流 diff、探针比对；只供人工 |
| trace 契约 | `--trace` 不保证字段稳定、不机器解析、不进判据 |
| 契约变更 | stage3 冻结契约变更走 ADR 与 `roadmap.md` §7.4，回归 A-stage3 |
| 校验归属 | L1（结构/错误映射）+ L2（S18/S19 探针）+ 横切（双入口等价、回归） |
| 双策略 | S18/S19 在 `serial`/`parallel` 零容差一致（第 37 章与本章双重收口） |

---

## 9. 简化登记

| 项 | 状态 | 替换章 / 备注 |
|---|---|---|
| 双入口 | 计算驱动入口 ∨ 直连 HLI 入口，无第三条路径 | `api_backend` 仅 `{hli, compute}` |
| 二期能力 | capability bit 恒 false / 文档登记 | `LaunchWithParamBuffer`、grid-wide barrier、image 原子、async copy |
| 范围外 | 永不实现 | MUSA/CUDA 兼容、动态并行、可变 subgroup、真实 occupancy |
| wave/占用统计 | 软观测，不进硬判据 | ADR-038 |

---

## 10. 待定项

- `LaunchWithParamBuffer`（应用直给参数 VA）：二期，本期只登记正确映射（`compute-driver.md` §5）。
- 二期候选领取：grid-wide barrier（协 kernel）、async copy、image 原子需先领 ADR 扩 `Capability` 位后才可启用，此前一律 `NotSupported`（`architecture-design.md` §7.9）。
- stage3（38~42）若需回填 29~34 共用单元能力（如新增 subgroup op），按 `roadmap.md` §7.4 流程在共用章 DoD 登记并回归。

---

## 2026-11-21 评审同步说明

A42 汇总计算驱动和直连 HLI 的双入口验收。矩阵、输入和探针以 [场景规约](../design/scene-specs.md) 为准，硬件能力继续由 A37 验收，驱动子能力由 A38~A41 验收。
