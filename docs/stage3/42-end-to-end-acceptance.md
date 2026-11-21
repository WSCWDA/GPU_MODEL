# 第 42 章 双入口等价与端到端验收

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章是 **stage3 计算驱动开发线（38~42，ADR-047）** 的收口 / 验收章。把 stage3 全部能力（对象骨架 / LaunchKernel / stream-event 异步 / 组内协作与间接/UAV 写回）经**计算驱动入口**（`umd/compute`，`api_backend=compute`）与**直连 HLI 计算入口**（`api_backend=hli`）双跑：S18 `scene_s18_compute_basic`（blockIdx/threadIdx、越界 work-item 不写、grid 边界部分 workgroup、wave 归约与 CPU 参考一致）+ S19 `scene_s19_compute_memory`（LDS/原子/barrier、间接 dispatch、异步计算、作用域语义、计算→图形 fence 后结果可见）。主判据：**双入口探针零容差等价（同参数、同命令流、同探针）**；A-stage3（A38~A42）全绿；E/S 场景回归不退化；`serial`/`parallel` 双策略一致。
> 设计权威：stage3 全链路契约见 `../design/compute-driver.md`；WA 目录见 `../architecture-design.md` §7.9，计算通路见 `../design/compute-line.md` §1/§2；kernel 参数与 `DispatchDesc` 见 `../architecture-design.md` §3.8/§3.11；映射与作用域见 §3.12/§3.13；错误码映射见 §3.9；场景与判据唯一权威 `../design/scene-specs.md` §3（S18/S19）、§4（第 42 行）。
> 编号纪律：本章不新增编号（复用全量既有登记）；不新增配置键（入口选择复用 `api_backend=hli / compute`，ADR-047）；开工前对照 `../design/bit-budget.md` 审计（ADR-045）。

---

## 1. 目标与非目标

### 1.1 目标

1. **双入口贯通**：38~41 的 demos 与 S18/S19 全部接到计算驱动入口（`LaunchKernel` 统一路径），与直连 HLI 手动 Dispatch 命令流逐条对齐。
2. **双入口等价主判据**：S18/S19 经 `compute` 与 `hli` 两入口探针**逐值等价（零容差）**；同参数、同命令流、同探针。
3. **A-stage3（A38~A42 汇总）全绿**：`serial`/`parallel` 硬判据零容差一致；E/S 场景回归不退化（S10/S11/S12/S16/S17 等）。
4. **错误路径回归**：六条参数校验非法样例、坏 module/function、未达成值 Query 经计算驱动入口正确映射 `MUSA_ERROR_*`，不吞错、不静默降级。
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
        (同参数、同命令流、同探针) → 双入口等价比对（零容差）
                   ▼
CP → Compute 引擎（薄壳）→ IShaderArray::Dispatch（29~31 共用单元）
   → LDS/原子/barrier/作用域（30）→ memory/texture 写回 + query（34）
                   ▼
A-stage3 验收表：A38~A42 全绿 + 双入口等价 + 回归不退化
```

说明：

- 双入口等价的**判定基准**是直连 HLI（黄金入口）：计算驱动入口不得产生与直连不同的命令流或数值（`compute-driver.md` §10 验收要点）。
- S18 `case-3`（经计算驱动，= case-1 同参数）为本章主验收子 case；S19 `case-0..4` 全部进入双入口矩阵：作用域/barrier、gather/scatter、异步、间接 dispatch + 谓词、图形↔计算 UAV 互操作（`scene-specs.md` §3）。

---

## 3. 输入输出契约

### 3.1 双入口等价判定

| 维度 | 判据 |
|---|---|
| 命令流 | 计算驱动入口生成命令包序列与直连 HLI 手动 Dispatch **同参数、同顺序、同值**（`SET_KERNEL_ARGS`/`DISPATCH`/`SIGNAL`/`WAIT` 逐条可比） |
| 探针 | S18 `case-0..3` / S19 `case-0..4` 探针表两入口**逐值零容差**一致 |
| 配置矩阵 | `api_backend ∈ {hli, compute} × exec_mode ∈ {serial, parallel}` 全组合一致 |

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
- 错误全部经计算驱动入口映射为 `MUSA_ERROR_*`，日志含 `{unit, packetType, dwordOffset, va, code}`（ADR-011 口径）。

---

## 5. 任务清单

> 任务编号 `T42.1~T42.5`，四段式。

| 子里程碑 | 任务 | 段 | 内容 |
|---|---|---|---|
| M42.1 | T42.1 | ① 契约 | 双入口等价验收的探针/判据单测与配置矩阵（`api_backend ∈ {hli,compute} × exec_mode ∈ {serial,parallel}`） |
| M42.1 | T42.2 | ② 主体 | 把 39~41 的 demos 全部接到计算驱动入口（`LaunchKernel` 统一路径），与直连 HLI 手动 Dispatch 命令流逐条对齐 |
| M42.2 | T42.3 | ② 主体 | 错误路径回归：六条校验非法样例、坏 module/function、未达成值 Query 经计算驱动入口正确映射 `MUSA_ERROR_*`（不吞错、不静默降级） |
| M42.2 | T42.4 | ③ 接入 | S18/S19 经计算驱动入口跑通本章覆盖子集（S18 `case-0~3`、S19 `case-0~4`；S18 `case-3` 主验收） |
| M42.2 | T42.5 | ④ 验收 | A42/A-stage3 全绿：S18/S19 双入口逐值等价（L2 + 横切）、`serial`/`parallel` 零容差一致、E/S 场景不退化、共用单元无"图形专用/计算专用"分支审查（ADR-032） |

### 子里程碑

| 子里程碑 | 范围 | 承载任务号 |
|---|---|---|
| M42.1 双入口贯通 | 等价探针/矩阵、demos 接入计算驱动入口、命令流逐条对齐 | T42.1~T42.2 |
| M42.2 接入与验收 | S18/S19 覆盖子集跑通、错误路径回归、A42/A-stage3 全绿与 E/S 回归 | T42.3~T42.5 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D42.1 | 等价矩阵单测全绿（4 组合 × 本章覆盖子集：S18 `case-0~3` / S19 `case-0~4`）；配置键不新增 | L1 契约 |
| D42.2 | 计算驱动入口命令流 = 直连 HLI 手动 Dispatch 命令流（同参数、同顺序）；`v[56..67]` / SIGNAL-WAIT 快照一致 | 横切 |
| D42.3 | S18 `case-3` 探针与直连 HLI 逐值零容差一致；S19 `case-0..4`（作用域/barrier、WA-10 gather/scatter、异步、间接 dispatch + 谓词、图形↔计算 UAV 互操作）经计算驱动入口与直连 HLI 等价；计算→图形 fence 后结果可见 | L2 + 横切 |
| D42.4 | 错误路径全部命中正确 `MUSA_ERROR_*`，无吞错/降级；二次非法调用不崩溃 | L1 |
| D42.5 | A-stage3 全绿：A38~A41 + 本章 A42（D42.1~D42.4）汇总 + E1~E5/S0~S17 全部适用回归（E0 单独 L1 自检）+ 适用场景 `serial`/`parallel` 零容差 + ADR-032 分支审查 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A42 / A-stage3）

**绑定：S18、S19（本阶段与 stage3 主验收）；回归 S10、S11、S12、S16、S17。**

| 项 | 内容 |
|---|---|
| 输入 | S18（1D/2D/3D grid+block、kernel 参数、越界 work-item，本章覆盖 `case-0~3`）；S19（LDS/原子/barrier + gather/scatter + 间接 dispatch + 异步计算 + UAV 互操作，本章覆盖 `case-0~4`）；`api_backend ∈ {hli, compute} × exec_mode ∈ {serial, parallel}` |
| 产物 | 双入口命令包序列比对、S18/S19 探针表（双入口）、A-stage3 汇总表、错误映射日志 |
| 主判据 | S18/S19 全部纳入双入口矩阵；探针逐值等价（同参数、同命令流、同探针，零容差；S18 `case-3` 为驱动入口主验收）；`serial`/`parallel` 零容差一致 |
| 本章专有判据 | 六条校验非法样例 / 坏 module / 未达成值 Query 经计算驱动入口正确映射 `MUSA_ERROR_*`（不吞错、不静默降级）；共用单元无特化分支（ADR-032） |
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

A42 是唯一的计算驱动端到端等价门禁：比较直连 HLI 与计算驱动两入口；不复验并重复承担 A37 的硬件单元实现任务。用例范围与 S18/S19 case 锚定以 `review-sync-20261121.md` §3 及 scene-specs 为准。
