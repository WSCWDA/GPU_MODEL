# 第 41 章 组内协作与间接触发

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章是 **stage3 计算驱动开发线（38~42）** 的第四章。在第 38~40 章基础上收口计算**驱动侧**的高级契约：三维 workgroup/grid 驱动侧封装（blockIdx/threadIdx 语义、grid 边界部分 workgroup、越界 work-item 不写内存）；组内 barrier（组内全部 wave 到达放行、跨 wave）与 workgroup/device 内存作用域；subgroup `ballot/shuffle/reduce` + LDS/原子；`DISPATCH_INDIRECT`（间接参数唯一真值、与直接 dispatch 逐值等价）；全阶段 UAV kernel 写回（buffer/image）；计算查询与统计（dispatch 计数 / timestamp 单调 / 可作 GPU 侧谓词）。所有能力建立在 29~31/33/34 章共用单元之上，驱动侧只做**打包、校验与入口语义收口**，不新增 `hw_sim` 单元、不新增包号。
> 设计权威：映射公式与 M-01~M-08 见 `../architecture-design.md` §3.12；地址空间与作用域 S-01~S-06 见 §3.13；`DispatchDesc`/D-01~D-13 与间接参数见 §3.11；对象与打包 ABI 见 `../design/compute-driver.md` §5；WA-02/WA-12 见 `../architecture-design.md` §7.9，计算通路见 `../design/compute-line.md` §1/§2；查询见第 34 章。
> 编号纪律：本章不新增编号；复用 `DISPATCH=0x04`、`DISPATCH_INDIRECT`（第 33 章）、`BARRIER=0x0E`、`PREDICATE=0x0D`（ADR-043/047）；不新增 `hw_sim` 单元/目录（ADR-009/032）；开工前对照 `../design/bit-budget.md` 审计（ADR-045）。

---

## 1. 目标与非目标

### 1.1 目标

1. **三维 workgroup 驱动侧封装**：`grid/block` 与 `blockIdx/threadIdx` 语义、grid 边界部分 workgroup 有效、越界 work-item 不写任何内存、末波 `LaneMask`（M-01/M-02）。
2. **组内 barrier 与作用域收口**：组内全部 wave 到达才放行（跨 wave，原子计数断言）；`scope ∈ {Workgroup, Device}` 在 `0x0E` 载荷内表达（S-03）；组驻留整体性防死锁（M-03）。
3. **subgroup / LDS / 原子计算侧包装**：`ballot/shuffle/reduce`（identity 按 op 定义：MIN=+∞ / MAX=−∞）、LDS 布局与原子、无效 lane 不执行（M-02）。
4. **间接 dispatch**：UMD 记录已绑定参数缓冲的命令，执行层在展开前读一次 28 B `{grid[3], block[3], reserved0}`，按 D-10 校验并派发；有效输入与直接 dispatch 逐值等价。
5. **全阶段 UAV kernel 写回**：buffer store 直写 memory（S-04）、image store 复用第 21 章 tiling 编码与 `common/format` 转换（与 26 章格式转换表共用实现，不依赖 OM 写回路径）。
6. **计算查询与统计**：dispatch 计数、timestamp 单调、可作 GPU 侧谓词（第 34 章 query 共用）（A41 贡献项）。

### 1.2 非目标

- 不做 MUSA/CUDA API 或二进制兼容；不做动态并行（kernel 内再派发）；不做 grid-wide barrier（协 kernel，二期候选）；不做可变 subgroup（固定 warp32）。
- 不新增硬件单元/包号；不为计算特化共用单元（ADR-032）；不做真实 occupancy 调优（软观测 wave 数）。
- 不做 S18/S19 双入口矩阵验收（第 42 章）。

---

## 2. 单元框图

```
apps（Demo C4 reduce / Demo C5 indirect_dispatch）
   ▼
umd/compute（驱动侧封装）
   ├─ 三维 workgroup 打包：grid/block → DispatchDesc（M-01/M-02 语义由驱动固化）
   ├─ barrier/作用域：flags bit0(组内 barrier)/bit1(device 作用域) → 0x0E 载荷
   ├─ DISPATCH_INDIRECT：indirectArgsVa（16 B 对齐）→ 展开前读一次
   └─ UAV 视图写回：buffer/image store（S-04）
   ▼
IHliCommandList.Dispatch / DispatchIndirect → 队列提交（第 32 章）
   ▼
CP → Compute 引擎（薄壳）→ IShaderArray::Dispatch
   ├─ workgroup 分配 CU/SIMD（31 章）
   ├─ subgroup ballot/shuffle/reduce（29 章）
   ├─ LDS / 原子 / barrier / 作用域（30 章）
   └─ 全局 / UAV 写回（3/11/21 章）
   ▼
query（34 章）→ 探针比对 + 统计断言
```

说明：

- 本章所有硬件能力都是既有共用单元（29~31、33、34），驱动侧**只做打包、校验与入口语义收口**；"组驻留整体性（M-03）"是防死锁的唯一手段，超限整个 workgroup 排队。
- 作用域/谓词/间接参数均在既有载荷位内表达，不新增顶层包号（ADR-043）。

---

## 3. 输入输出契约

本章交付 UMD 输入校验、命令记录和探针。下述索引计算、屏障、subgroup、UAV 写回和间接参数取数属于既有执行层契约，通过共用单元的回归验证。三维索引公式继续由架构设计 §3.12 定义。
### 3.1 三维映射（权威：`architecture-design.md` §3.12，唯一公式）

```
threadsPerGroup = block.x * block.y * block.z
wavesPerGroup   = ceil(threadsPerGroup / 32)
gtid            = tid.x + tid.y*block.x + tid.z*block.x*block.y      // z 最外层
waveIdInGroup   = gtid / 32                 lane = gtid % 32
workgroupId     = gx + gy*grid.x + gz*grid.x*grid.y
globalThreadId  = workgroupId * threadsPerGroup + gtid
```

- **M-01 身份唯一**：id 只由 grid/block 决定，与完成顺序/CU/SE 无关，`serial`/`parallel` 逐值相同。
- **M-02 末波部分有效**：`LaneMask` = 有效 lane；无效 lane 不执行访存/原子/barrier；**禁止移植第 25 章 2×2 quad 规则**。
- **M-03 组驻留整体性**：`wavesPerGroup` 波同 CU 驻留；预算超限整个 workgroup 排队；不得跨 CU/SE 拆组（D-11）。
- **M-04 barrier 一致到达**；**M-07 组边界不隐式同步**；**M-08 确定性**。
- **D-13 组级静态可调度性**：`wavesPerGroup ≤ 16`（B-037）、`ldsStaticBytes ≤ 64 KiB`（B-036）、grid 总量 ≤ 2^20（B-038）；超限记录期 `InvalidArgument` 拒绝、不产生包、不排队（M-06）；仅"可满足但被占"按 ADR-038 排队。

### 3.2 作用域与屏障（权威：`architecture-design.md` §3.13 + `sync-primitives.md` §3）

| 写 → 读 | 需要的机制 |
|---|---|
| 同组 LDS | ISA `BARRIER(Workgroup)` 指令（不是 PKT_TYPE=0x0E 包） |
| 同组 global（同 CU） | ISA `BARRIER` + 本模型的 workgroup 可见性规则；不得把局部指令当成命令包 |
| 跨组 / 跨 CU global | `PKT_TYPE=0x0E` resource/device envelope + fence（排空/失效） |
| 跨 dispatch / 跨队列 | fence/timeline + `PKT_TYPE=0x0E` ownership/device envelope |

- 若发出命令包级 `PKT_TYPE=0x0E`，必须使用 `command-mmio.md` §2.3 的唯一 3-dword envelope（kind/scope/memOrder/accessMask + stage masks）；本章不得定义另一种载荷。`scope/access` 是 envelope 的字段摘要，不是独立 ABI。
- LDS 永不经 VA/PA（S-01）；dispatch 边界不自动刷新（S-02）。

### 3.3 间接 dispatch（权威：§3.11 D-10）

- `indirectArgsVa` 已映射、16 B 对齐；`DispatchArgsIndirect{grid[3], block[3], reserved0=0}`；
- 展开 workgroup **前读一次**，作为本次 dispatch 唯一真值，执行中不重读；
- 取值非法（0/越限）由 GPU 侧确定性中止，不修复。

### 3.4 UAV kernel 写回收口（§3.13 S-04）

- buffer store 由 memory 子系统直写（不经 OM）；image store 复用第 21 章 tiling 编码 + `common/format` 转换，与 RT 写回共用同一格式转换/写掩码实现（不复制两份）；
- 写 UAV 时 Early-Z 逻辑不影响此路径（第 27 章边界）。

---

## 4. 内部数据流或状态机

### 4.1 组内 barrier 可达性（防死锁，M-03/M-04）

```
预算检查：wavesPerGroup × (VGPR+SGPR 占用) + ldsStaticBytes ≤ CU 容量
  超限 → 整个 workgroup 排队（不得部分驻留 / 跨 CU）
  不超限 → wavesPerGroup 波同时驻留 → barrier 永远可达（无死锁）
非一致控制流到达 BARRIER → ISA 层检 InvalidIsa（M-04）
```

### 4.2 DISPATCH_INDIRECT 数据流

```
建议：kernel-A 写 grid/block 到 indirectArgsVa（16 B 对齐）
  → DispatchIndirect(indirectArgsVa)
  → CP 展开前读一次 → 生成 workgroup 集合（唯一真值）→ 派发
  → 与直接 dispatch（同参数）逐值等价断言
```

### 4.3 组内协作数据流

```
workgroup → 首波执行 → LDS 初始化 → BARRIER(Workgroup) → 计算
  → 末波 reduce（subgroup/原子）→ BARRIER → 写回（memory/texture）
  → dispatch 结束（不隐式同步，M-07）
```

---

## 5. 任务清单

> 任务编号 `T41.1~T41.8`，四段式。

| 子里程碑 | 任务 | 段 | 内容 |
|---|---|---|---|
| M41.1 | T41.1 | ① 契约 | 封装三维 grid/block、同步作用域、间接参数及资源视图输入，引用已有契约 |
| M41.1 | T41.2 | ② 主体 | 校验三维 grid/block 并记录 Dispatch，使用执行层探针验证 ID、尾波与边界 |
| M41.1 | T41.3 | ② 主体 | 转发 workgroup/device 作用域和同步描述，验证已有屏障与可见性语义 |
| M41.1 | T41.4 | ② 主体 | 接入使用既有 subgroup/LDS/原子能力的 kernel，驱动负责对象、参数与派发 |
| M41.2 | T41.5 | ② 主体 | 记录 `DISPATCH_INDIRECT`，校验地址、16 B 对齐及资源映射；执行时读参、校验值并展开由已有执行层负责 |
| M41.2 | T41.6 | ② 主体 | 转发 UAV buffer/image 视图及生命周期引用，回归已有写回、tiling 与格式 |
| M41.2 | T41.7 | ③ 接入 | 接入既有查询/谓词，运行 C4 归约和 C5 间接派发 |
| M41.2 | T41.8 | ④ 验收 | 汇总 D41.1~D41.7，以 C4/C5 及代表性 UAV 用例验收 |

### 子里程碑

| 子里程碑 | 范围 | 承载任务号 |
|---|---|---|
| M41.1 派发与组内协作接入 | 三维封装、作用域转发、组内 kernel 用例 | T41.1~T41.4 |
| M41.2 资源、间接派发与验收 | 间接参数与 UAV 视图，查询/谓词与 C4/C5 | T41.5~T41.8 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D41.1 | UMD 包装契约可独立编译，输入合法性按 §3.12/§3.13，使用同一套执行层索引契约 | L1 契约 |
| D41.2 | 三维 grid/block 正确记录，非法输入无 dispatch；有效 ID、尾波 mask 和边界越界不写的探针与执行基准一致 | L2 |
| D41.3 | 作用域正确传到执行层；S19 `case-0` 的跨 wave barrier 在全部 wave 到场后放行，作用域回归通过 | L2 |
| D41.4 | 同 kernel 经 UMD 使用既有 ballot/shuffle/reduce、LDS 与原子后，探针与 HLI 基准一致 | L2 |
| D41.5 | 非法间接地址、对齐或映射在记录侧拒绝；合法地址的内容由执行层在执行时检查，直接/间接有效输出逐值一致 | 横切 |
| D41.6 | UAV 视图及资源引用正确传递；S19 `case-4` 代表性输出与 20 章描述符、26 章格式契约逐字节一致 | 横切 |
| D41.7 | 查询与谓词经既有路径记录；计算 dispatch 计数、timestamp 单调与谓词探针通过；C4/C5 可运行 | L1 |
| D41.8 | A41 汇总 D41.1~D41.7。C4 的 WA-02 基线及部分活跃 ADD、MIN/MAX 单波用例满足 CPU 参考 ≤1 ULP、顺序正确及串并行硬探针一致。C5 与直接派发逐值一致 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A41）

**绑定：Demo C4 `reduce`（WA-02）、Demo C5 `indirect_dispatch`（WA-12）；S18 case-2（3D + 越界）与 S19（作用域/间接/UAV/异步）在本章驱动侧贯通。**

| 项 | 内容 |
|---|---|
| 输入 | 三维 grid/block（含越界、部分 workgroup）；block=64/256 跨 wave barrier 样例（对齐 S19 `case-0` block=256）；归约 kernel（WA-02）；间接 dispatch 双 kernel 流水（WA-12）；UAV 写回样例 |
| 产物 | 三维 id 探针表、barrier 到达日志、归约结果、间接/直接命令流与探针对比、UAV 回读、查询统计 |
| 主判据 | blockIdx/threadIdx = ref_model（L2）；跨 wave barrier 只在全部 wave 到达放行；归约 ≤1 ULP 通过、>1 ULP 阻断；同序 FMA 下 0 ULP 是额外诊断期望，不改变通过门槛；归约顺序断言强制；间接与直接逐值等价（横切） |
| 本章专有判据 | 越界 work-item 不写任何内存；作用域语义确定；UAV 写回同 `(x,y,format)` 逐字节一致（对照 20 章 image 描述符 + 26 章格式转换表，ADR-049）；无新增模块/包号 |
| 回归项 | R41：S10（组内归约计算侧）、S12（多引擎）、S16（所有权）不退化；未列明历史 A 门禁不自动阻断 |

> 场景输入与探针单份规约见 `../design/scene-specs.md` §3（S18/S19 节）与 §4（第 41 行）。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | workgroup 分配、barrier 放行序、reduction lane 序、间接参数唯一真值显式定义；无时间输入 |
| 时间无关 | kernel 无 `uTime`/墙钟；帧序由 `frame_index` 驱动 |
| 可观测 | 每 dispatch wave 数、每 workgroup 波数、barrier 到达计数、dispatch/timestamp 统计 |
| 契约影响 | 不新增编号（复用既有包号/SubOp/stateId/MMIO/配置键）；映射/作用域/间接参数为冻结契约（ADR-043/045） |
| 简化登记 | 无 grid-wide barrier、无动态并行、无可变 subgroup；occupancy 只软观测 |
| 计算兼容 | 共用单元按包类型派发，无特化分支（ADR-032）；驱动侧只包装不复制 |
| 配置与回退 | 不新增配置键；`api_backend=hli / compute` |
| 调试钩子 | 三维 id 快照、barrier 日志、间接参数 dump；只供人工 |
| trace 契约 | `--trace` 不保证字段稳定、不进判据 |
| 契约变更 | 映射/作用域/间接参数走 ADR，回归 S18/S19 与受影响 S 场景 |
| 校验归属 | L1（结构/统计）+ L2（三维/归约/UAV 探针）+ 横切（间接等价） |
| 双策略 | 三维 id、归约、间接在 `serial`/`parallel` 零容差一致 |

---

## 9. 简化登记

| 项 | 状态 | 替换章 / 备注 |
|---|---|---|
| 组间同步 | 无隐式同步（M-07），只走显式 barrier/fence | §3.13 S-02 |
| grid-wide barrier（协 kernel） | 二期候选 | §7.9，未获 ADR 前 `NotSupported` |
| 动态并行 / child kernel | 范围外 | 第 37 章非目标 |
| image 原子 / async copy | 二期候选 | §7.9 |

---

## 10. 待定项

- C4 的 identity 和部分活跃边界归 A41 验收。A42 汇总其结果，S18 case-5 保留 HLI 回归。
- `DISPATCH_INDIRECT` 的谓词组合（PREDICATE + 间接）一期只保证与直接等价；复杂谓词链二期。

---

## 2026-11-21 评审同步说明

本章交付三维派发、间接、UAV 和查询的驱动接入。容量与间接 ABI 见 [架构设计 §3.11](../architecture-design.md)，BARRIER 载荷见 [命令包登记 §2.3](../design/command-mmio.md)。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A41 和适用的 R41 后提交。分配核对项为沿用 28 B 记录、barrier 位域与参数区，新增或改动按 [编码与位段校验表](../design/allocation-audit.md)登记。

```text
feat(gpu-ch41): launch collaborative and indirect compute kernels

- 通过计算 UMD 暴露组内协作、间接派发和对应校验。
- 用 C4/C5 检查驱动入口、结果及间接记录负例。

Tasks: T41.1–T41.8
Validation: <实际执行的命令、用例和结果>
```
