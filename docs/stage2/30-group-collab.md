# 第 30 章 组内协作（workgroup + LDS）

> 本章引入 workgroup 级协作：shared memory（LDS/STS）、原子操作、组内 barrier（workgroup 作用域同步）与内存序/作用域。workgroup 是调度单位、wave 是执行单位；barrier 要等组内全部 wave 到达后才放行。本章以独立 workgroup 单元用例验收，不提前承担 S19 场景主验收（归第 37 章）。设计说明见 `../design/sync-primitives.md` §3、`../design/compute-line.md` §1。
> 与本章相关的设计说明：`../architecture-design.md` §3.8（kernel 参数）；ADR-034；`../design/isa-abi.md` §2（opcode 区段）与 §3（LDS/atomic 默认作用域）；第 15 章（Suspend/Resume）；`../design/execution-parallel.md` §4/§5（wave 是执行单位）。
> 落地模块：`hw_sim/shader/shared`、`apps/scene_s10_reduce`（子 case）、`apps/scene_s19_compute_memory`（`case-0`/`case-1`）。

---

## 1. 目标与非目标

### 1.1 目标

1. shared memory（LDS）：独立地址空间，`LDS/STS` 访问；大小经 PSO 下发（`psO.ldsBytes`）。
2. bank 布局：LDS 按 32 bank × 4B；bank 冲突影响软观测（`--stats`），不改变结果。
3. 原子操作：`ATOMIC_ADD/CAS`（0x40~0x4F）在 LDS/全局；workgroup 作用域原子；结果确定（原子累加序固定）。
4. 组内 barrier：`BARRIER` 在 workgroup 内同步全部 wave（block ÷ warp 向上取整 + 部分块）；全部到达才放行。
5. 内存序/作用域：`Relaxed/Acquire/Release/AcqRel/SeqCst` + `Workgroup/Device`；shared 栅栏与全局资源屏障是两套机制。
6. workgroup 是调度单位，wave 是执行单位：DISPATCH 拆分、block 大小 >32（如 64/96）时多 wave/组。
7. 跨 wave barrier 正确：独立 workgroup 单测以 block=256（8 wave）验证。

### 1.2 非目标

- 不做 device 作用域的细粒度 shared 访问（那是全局内存）。
- 不做硬件无锁算法优化（本期顺序执行+确定时间）。
- 不做 LDS 的动态分配/稀疏；大小静态每 PSO。
- 不做 barrier 的 fence-only 复杂化（两套机制明确）。

---

## 2. 单元框图

```
 workgroup(组内 N wave，每 wave 32 lane)
   │  LDS（shared memory）
   ▼
┌─────────────── hw_sim/shader/shared ────────────────┐
│ LDS Array: 每 workgroup 一个；bank 布局 32×4B      │
│ AtomUnit（LDS/global atom，workgroup 作用域）         │
│ BarrierUnit：wave 到达计数，N 全到 → 放行              │
│ MemOrder：Relaxed/Acquire/Release… → 各指令标注      │
└────────────────────────┬───────────────────────────┘
                         │ 与 WaveScheduler 交互（Suspend/Barrier）
```

- workgroup 的 LDS 在各波间共享；跨 workgroup 不共享（隔离）。`LDS/STS` 默认只寻址当前 workgroup LDS，默认 scope=`Workgroup`；global `ATOMIC_*` 默认 scope=`Device`，显式缩小 scope 才可采用 Workgroup。默认值与 ISA 编码见 `isa-abi.md` §2/§4。
- Barrier 挂起各 wave，等全部 wave 到齐后一起放行（跨 wave barrier）。

---

## 3. 输入输出契约

### 3.1 shared memory / 指令

```cpp
struct SharedMemState { u32 ldsBytes; };  // PSO 下发
// LDS/STS：
//   LDS  dst, [addr+imm]
//   STS  [addr+imm], src
//   addr 为 LDS 空间的 4B 对齐偏移
```

- 独立地址空间：`addr` 不经过 VA 翻译，只经 LDS 单元。
- 越界（≥ ldsBytes）→ `InvalidArgument`（可定位）。

### 3.2 原子

```cpp
// 指令：
ATOMIC_ADD dst, addr, inc       // LDS 或 global（workgroup 作用域）
ATOMIC_CAS dst, addr, cmp, new
// result = 旧值（dst），内存位置按地址寻址
```

- workgroup 作用域：对 LDS 原子天然 workgroup；对 global 原子需声明作用域（device 需要全局屏障）。
- 原子执行次序：**原子加固定按 wave 领取序累加**（每原子一次同序），保证 `ref_model` 一致。

### 3.3 barrier / 内存序

```cpp
enum class MemOrder { Relaxed, Acquire, Release, AcqRel, SequentiallyConsistent };
enum class Scope { Workgroup, Device };

ISA `BARRIER` { memOrder, scope }  // workgroup 指令；不是 PKT_TYPE=0x0E 命令包
// workgroup 作用域：该 workgroup 内各 wave 到达后放行
// ISA barrier 只同步执行 wave；设备级内存可见性由包级 BARRIER(kind=resource/ownership) + fence 建立
```

- 两套机制：shared 栅栏（workgroup）与全局屏障（device）不混用。

---

## 4. 内部数据流

### 4.1 DISPATCH → workgroup → wave

```
DISPATCH(grid, block)：
  workgroup 数 = ceil(grid / block)；每 workgroup 内 wave 数 = ceil(block/32)
  workgroup 分配属性统一（每 workgroup 一个执行块 + 一个 LDS）
  越界 work-item（grid 边界部分块）→ 按 Id 过滤（31 章细化）
```

- 本组内：所有 wave 调度到同一逻辑 CU 虚拟化执行。

### 4.2 barrier 语义（跨 wave）

```
wave 执行 BARRIER(Workgroup)：
  barrier.await(wave_id)
  若组内 wave 数未到齐 → Suspend（Into 挂起队列，让出 CU）
  到齐（arrival_count == expected）→ 全部 Resume（放行）
```

- 若只有一个 wave（block ≤ 32），barrier 几乎自足（无跨 wave 等待），但仍需 fence 语义（Acquire/Release）。

### 4.3 内存序/作用域

```
Relaxed：无同步。
Acquire：该指令后的 shared 读/写对同 workgroup 内释放者可见。
Release：该指令后的写对同 workgroup 内获取者可见。
AcqRel/SeqCst：双向可见序。
作用域 Workgroup：LDS 栅栏（组内）。
作用域 Device：全局屏障（12 章 flush/invalidate + fence）。
```

- 实现上：在顺序模拟内所有字即刻一致，但**语义标注**必须落在指令/barrier 上以证明正确性（未来并行/软化依赖此标注）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T30.1 | ① 契约 | `SharedMemState`/MemOrder/Scope/barrier 结构/`LDS=0x23`、`STS=0x24`、`ATOMIC` 指令落头文件（B-043） |
| T30.2 | ② 主体 | LDS 分配与寻址（每 workgroup）、bank 布局 |
| T30.3 | ② 主体 | `LDS/STS` 读写指令 + 越界定位 |
| T30.4 | ② 主体 | LDS bank 冲突统计（软观测） |
| T30.5 | ② 主体 | `ATOMIC_ADD/CAS` 在 LDS 与 global（workgroup 作用域） |
| T30.6 | ② 主体 | 原子执行序固定（wave 内 lane 升序 + wave 序） |
| T30.7 | ② 主体 | 组内 `BARRIER`：跨 wave 到达/放行 |
| T30.8 | ② 主体 | 单 wave 组时的 barrier（fence 语义） |
| T30.9 | ② 主体 | 内存序标注（Acquire/Release/...）→ LDS 栅栏与全局屏障分离 |
| T30.10 | ② 主体 | workgroup 作用域到 device 作用域的显式升级（BARRIER device） |
| T30.11 | ② 主体 | block>32（64/96）多 wave 组 + 跨 wave barrier 完整 |
| T30.12 | ② 主体 | grid 边界部分 workgroup 的越界 work-item 过滤 |
| T30.13 | ③ 接入 | `scene_s10_reduce`（LDS 转置 + 原子归约）；S19 `case-0/1` 仅提供 barrier/原子的独立 workgroup 单测向量，不运行 S19 端到端场景 |
| T30.14 | ④ 验收 | A30：S10 全绿 + workgroup 单元用例 + 专有判据 + R30 回归项 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M30.1 | T30.1~T30.4 | LDS 布局/bank 冲突/shared 指令，D30.1~D30.4 全绿 |
| M30.2 | T30.5~T30.8 | 原子操作/组内 barrier，D30.5~D30.8 全绿 |
| M30.3 | T30.9~T30.14 | 内存序/作用域/跨 wave barrier/接入验收，A30 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D30.1 | 接口独立编译；`MemOrder/Scope` 取值冻结；LDS 常量 | L1 契约 |
| D30.2 | LDS 分配：`ldsBytes` PSO 每组唯一；越界访问返回 `InvalidArgument` | L1 契约 |
| D30.3 | `LDS/STS` 读写：写后同组读回相等；越界访问定位到 `{unit, offset}` | L1 契约 |
| D30.4 | bank 冲突计数：同 bank 并发访问计数 ≥0，且两次运行一致 | L1 契约 |
| D30.5 | `ATOMIC_ADD` 在 LDS：64 lane（2 wave）各加 → 总和确定（与序 → 一致）| L1 契约 + L2 |
| D30.6 | 原子序固定：同输入两次运行全局原子值一致 | L1 契约 |
| D30.7 | barrier：block=64（2 wave）跨 wave 同步，放行后才有一致可见 | L1 契约 |
| D30.8 | block≤32 单波组 barrier 放行语义 | L1 契约 |
| D30.9 | acquire/release 顺序语义：释放写 → barrier → 获取读，值可见 | L1 契约 |
| D30.10 | device 作用域：全局屏障被触发（Flush/Invalidate 调用）| L1 契约 |
| D30.11 | 独立 workgroup 单测：block=256（8 wave）跨 wave barrier 全部 wave 到达后才放行；不作为 S19 场景验收 | L2 |
| D30.12 | grid 边界越界 work-item 不执行/不写；部分 workgroup 正常 | L1 契约 |
| D30.13 | S10 + 独立 workgroup 单测：转置/原子/barrier 与 ref_model 一致；统计（barrier 次数、原子计数、bank 冲突）两次一致 | L2 |
| D30.14 | A30 全绿：shared 转置、原子累加与 CPU 参考一致、barrier 顺序正确、workgroup/device 作用域释放获取语义确定（跨 wave barrier）+ R30 回归项 | L1 + L2 |

---

## 7. 场景与判据（A30）

**绑定场景：S10 `scene_s10_reduce`（LDS 转置/原子归约子 case）。S19 `case-0`/`case-1` 的参数与期望结果可作为 workgroup 单元测试向量，但本章不运行 S19 端到端场景；S18/S19 主验收归第 37 章。**

| 项 | 内容 |
|---|---|
| 输入 | S10：LDS 矩阵转置 + 组内原子归约；独立 workgroup 向量：block=256（8 wave）跨 wave barrier + workgroup/device 作用域，并覆盖 gather/scatter 原子 |
| 产物 | LDS 内容、原子结果、barrier 放行证据（访问次序快照） |
| 主判据 | S10 与独立 workgroup 单测输出符合 ref_model（转置/归约/barrier 有序性零容差） |
| 本章专有判据 | ① shared 转置；② 原子累加与 CPU 参考一致；③ barrier 顺序正确；④ workgroup/device 作用域释放获取语义确定（跨 wave barrier） |
| 回归项 | R30：无 subgroup/group 的图形基线不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 31 章把本组内多 wave 调度到 CU 阵列/线程本地 TLB；37/stage3 38~42（计算驱动，原 35.5）消费 S18/S19 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 原子序（wave+lane 升序）、barrier 放行后序、LDS 访问序固定 |
| 时间无关 | barrier 是逻辑到达计数，非墙钟 |
| 可观测 | barrier 次数、原子执行次数、bank 冲突、`--trace=shared` |
| 契约影响 | LDS/原子/barrier/内存序作用域契约冻结（`sync-primitives`） |
| 简化登记 | 无共享 → LDS + 原子 + barrier + 内存序/作用域 |
| 计算兼容 | workgroup/wave 属计算与图形共用；barrier 语义不按 workload 分叉 |
| 配置与回退 | `ldsBytes` 为 PSO 字段；`block` 来自 DISPATCH 载荷；`mem_sanitizer` 为内部调试选项（均非全局配置键） |
| 调试钩子 | `--trace=shared`（bank/原子/barrier 行） |
| trace 契约 | 不进入判据 |
| 契约变更 | LDS 布局/原子序/barrier/内存序为冻结项；变更走 ADR 回归 S10 与 workgroup 单测 |
| 校验归属 | L1（结构/计数）+ L2（转置/归约/有序性与 ref_model） |
| 双策略 | 原子归约序与 barrier 放行序两策略一致（31 章并行为前提） |
| 阶段衔接 | 为 31（CU 阵列、三维 workgroup 映射、occupancy 上限影响 LDS 容量）提供组内语义与容量模型 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 共享 | 无 → LDS/STS | — |
| 原子 | 无 → ATOMIC_ADD/CAS | — |
| barrier | 无 → 组内跨 wave barrier | — |
| 内存序 | 无 → 作用域/acquire-release 标注 | — |

---

## 10. 待定项

- global 地址上的 `ATOMIC_ADD/CAS` 默认 Device scope；指令显式携带 Workgroup scope 且目标是 LDS 时才缩小作用域。LDS/STS 与 BARRIER 的默认语义见 `../design/isa-abi.md` §3，本章不另设默认值。
- LDS bank 布局仿真如引入 31 章并行后产生真实 bank 冲突差异，只影响软观测计数，不影响正确性；届时以同一观察基准对齐。
- 跨 wave barrier 的"放行后谁先执行"在本期顺序模拟固定 wave 领取序；31 章 CU 并行后在放行点强制确定性。

---

## 2026-11-21 评审同步说明

组内执行 barrier 使用本章的 ISA/LDS 原语。宿主侧 BARRIER 包见 [命令包登记 §2.3](../design/command-mmio.md)，跨 wave 步进见 [并行执行设计](../design/execution-parallel.md)。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A30 和适用的 R30 后提交。分配核对项包括LDS/STS、同步/原子 opcode 与作用域。变更按已有分配流程登记。

```text
feat(gpu-ch30): synchronize waves within a workgroup

- 实现组内 LDS、barrier、原子操作和调度恢复。
- 用 S10 组级用例及独立 S19 向量对照结果。

Tasks: T30.1–T30.14
Validation: <实际执行的命令、用例和结果>
```
