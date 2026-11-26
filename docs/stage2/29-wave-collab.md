# 第 29 章 波内协作

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章引入波内（subgroup）协作原语：`ballot/shuffle/broadcast/reduction/deriv`，对应 ISA 区段 0x70~0x7D（opcode 7 bit 重编号，ADR-043，`../design/isa-abi.md` §2）。wave 内 lane 间按固定 lane 序协作（ADR-018）；归约顺序固定（lane 升序）；结果为确定值。它是计算线（S10/S18）波内共享的第一步，也为第 30 章组内协作提供波内基础。
> 与本章相关的设计说明：`../design/isa-abi.md` §2（subgroup opcode 0x70~0x7D，7 bit 重编号）、§3（lane 序、mask）；ADR-034（发散分析）；`../design/execution-parallel.md` §5.5（归约顺序固定）；`../design/compute-line.md` §4（共用单元不特化）。
> 落地模块：`hw_sim/shader/subgroup`、`apps/scene_s10_reduce`；计算波内行为由本章单元用例覆盖，S18 场景主验收归第 37 章。

---

## 1. 目标与非目标

### 1.1 目标

1. `BALLOT`：返回 u32 掩码，bit i = lane i 的谓词结果。
2. `SHFL`：指定 lane 间数值换出；`SHFL_UP/DOWN`、`SHFL_IDX`。
3. `BROADCAST`：从指定 lane 广播值到全波。
4. `REDUCE`（wave 归约）：`ADD/MIN/MAX/AND/OR/XOR`，lane 升序确定；identity 按 op 定义（ADD=0、MIN=+∞、MAX=−∞、AND=全 1、OR/XOR=0），允许部分 lane 活跃（Q-62）。
5. `DERIV`：波内导数（相邻 lane 差分），供纹理 LOD。
6. 内建：`SUBGROUP_SIZE`（=32）、`SUBGROUP_INVOCATION`（=lane id）、`ballot` 等 subgroup 内建对 CPU 参考一致。
7. 结果确定：任何并行度/调度下探针零容差一致。

### 1.2 非目标

- 不做跨 wave 的组内协作（第 30 章 shared/atomic/barrier）。
- 不做可变 subgroup 大小（固定 32）。
- 不做 `voteAll/voteAny/voteBallot` 之外的选举（用 ballot 组合即可）。
- 不做 shuffle 的掩码特殊值扩展（只 lane 序 + 前向方向）。

---

## 2. 单元框图

```
 wave（32 lane）── subgroup 指令 → 波内协作单元
   │  LanedMask 固定
   ▼
 ballot/shuffle/broadcast/reduce/deriv
   │  每条指令在 "当前活跃 lane" 上执行
   ▼
 结果：某 lane 或 mask → 用于 SIMT 后续
（Lane id = 0..31；归约按 lane 升序）
```

- 所有 subgroup 指令作用于"当前栈顶掩码中的活跃 lane"（保守：非活跃 lane 结果未定义但确定 = 0）。
- 不作为跨 wave 同步点（第 30 章 barrier 才是）。

---

## 3. 输入输出契约

### 3.1 指令集（isa-abi §2，subgroup 0x70~0x7D）

| 指令 | 语义 | 输出 |
|---|---|---|
| `BALLOT dst, pred` | `dst = Σ (lane i 的 pred ? 1<<i : 0)` | u32 mask |
| `SHFL_IDX dst, val, srcLane` | `dst = val[srcLane]` | 值 |
| `SHFL_UP dl, val, delta` | `dst_lane = val[lane - delta]`（下界外 = 0） | 值 |
| `SHFL_DOWN dl, val, delta` | `dst_lane = val[lane + delta]`（上界外 = 0） | 值 |
| `BROADCAST dst, val, srcId` | 广播 srcId 的值 | 值 |
| `REDUCE_ADD/MIN/MAX/AND/OR/XOR dst, val` | 波内归约（lane 升序；identity 按 op，MIN=+∞、MAX=−∞） | 值 |
| `DERIV dPos, dUv` | quad 差分 | 向量 |

- 归约顺序固定：对 `ADD` `(0+1+...+31)` 次序累加；`MIN/MAX` 按 lane 序比较。REDUCE 只在活跃 lane 上归约，identity 按 op 定义（Q-62）。

### 3.2 内建（编译器映射）

| 内建 | 映射 |
|---|---|
| `SubgroupSize()` | 常量 32 |
| `SubgroupInvocationId()` | lane id（0..31） |
| `Ballot(pred)` / `Shuffle` / `Broadcast` | 对应指令 |
| `SubgroupReduce` | REDUCE 族 |

---

## 4. 内部数据流

### 4.1 执行模型

```
对每 subgroup 指令：
  mask_c = 当前活跃掩码（lanes 存活）
  ballot: out = OR(mask (i in mask_c) : (pred_i ? 1i : 0), 0)
  shuffle: out_lane = val[uint(target)]（target 由公式）
  reduce: acc = identity(op)  // ADD=0, MIN=+inf, MAX=-inf, AND=all1, OR/XOR=0
          for lane in 0..31: if lane in mask_c: acc op= val[lane]
  写回 dst（每 lane）
```

- 非活跃 lane 不参与归约、shuffle 读它返回 0（确定未定义）；允许部分 lane 活跃，REDUCE 的 identity 与活跃集共同决定结果（Q-62）。
- 结果无跨波次依赖 → 天然确定。

### 4.2 与流水交互

- subgroup 指令在执行阶段同步完成（不触发挂起，不涉 barrier）；不改变控制流。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T29.1 | ① 契约 | subgroup 指令集、内建映射、`SubgroupSize=32` 常量落头文件 |
| T29.2 | ② 主体 | `BALLOT`：掩码生成（活跃 lane 限制） |
| T29.3 | ② 主体 | `SHFL_IDX/UP/DOWN`：lane 间取值、边界 0 |
| T29.4 | ② 主体 | `BROADCAST` |
| T29.5 | ② 主体 | `REDUCE_ADD/MIN/MAX/AND/OR/XOR`：固定 lane 序归约 |
| T29.6 | ② 主体 | `DERIV`：quad 相邻差分 |
| T29.7 | ② 主体 | 编译器前端映射：内建 → subgroup opcode（17 章扩展） |
| T29.8 | ② 主体 | 波内统计：`ballot/reduce` 调用计数 |
| T29.9 | ③ 接入 | `scene_s10_reduce`（ballot/shuffle/reduce/deriv）接入；另以固定 wave 输入向量单测计算侧 subgroup 内建，不运行 S18 端到端场景 |
| T29.10 | ④ 验收 | A29：S10 全绿 + subgroup 单元用例 + 专有判据 + R29 回归项 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D29.1 | 接口独立编译。lane 0 和 17 的 `SubgroupSize()` 都返回 32，`SubgroupInvocationId()` 分别返回 0 和 17，沿用当前 opcode 映射 | L1 契约 |
| D29.2 | `Ballot((lane%2)==0)` → 偶数位模式 `0x55555555`（活跃全波）；部分活跃仅含活跃位 | L1 契约 |
| D29.3 | `ShuffleIdx(v, lane+1)` 各 lane 得 v[lane+1]；下界/上界外为 0 | L1 契约 |
| D29.4 | `Broadcast(v, 5)` → 全波 v[5] | L1 契约 |
| D29.5 | `ReduceAdd(v[i]=i)` →（活跃全波）Σ=496；`Min`=0，`Max`=31；部分活跃时按 op identity 归约（MIN/MAX 不因死 lane 得 0） | L1 契约 |
| D29.6 | `Deriv`：quad 内相邻差 = v[+1]-v[-1]（屏幕导数）| L1 契约 |
| D29.7 | 编译 `SubgroupReduce` 等内建成功并映射到对应 opcode | L1 契约 + L2 |
| D29.8 | S10 探针：ballot 模式、shuffle 顺序、reduce 值 = ref_model | L2 解析 |
| D29.9 | A29 全绿：ballot/shuffle/归约/导数结果确定；计算 subgroup 内建与 CPU 参考一致 + R29 回归项 | L1 + L2 |

---

## 7. 场景与判据（A29）

**绑定场景：S10 `scene_s10_reduce`。计算侧 wave 输入向量只用于 subgroup 单元测试，不运行 S18 端到端场景；S18 主验收归第 37 章。**

| 项 | 内容 |
|---|---|
| 输入 | S10：PS 中用 `ballot/SubgroupReduce/Shuffle` 探针；计算单测：固定 32-lane wave 输入验证 `SubgroupReduce`、mask 与 lane 顺序 |
| 产物 | ballot mask、reduce 值、shuffle 结果、导数 |
| 主判据 | S10 输出与固定 wave 单测均符合 ref_model（ballot 模式、归约值零容差） |
| 本章专有判据 | ① ballot/shuffle/归约/导数结果确定（两次一致）；② 计算 subgroup 内建与 CPU 参考一致 |
| 回归项 | R29：E1–E5 图形基线（无 subgroup shader）不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 第 30 章 shared/barrier 在波之上做组内；第 31 章策略 B 可并行化 subgroup |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 归约 lane 序固定、shuffle target 公式固定、ballot 位序固定 |
| 时间无关 | 无时间输入 |
| 可观测 | subgroup 指令计数、`--trace=subgroup` |
| 契约影响 | subgroup 指令/内建/掩码语义冻结 |
| 简化登记 | 无波内协作 → ballot/shuffle/归约/导数 |
| 计算兼容 | subgroup 是图形/计算共用硬件能力；计算驱动在第 37 章之后消费 |
| 配置与回退 | warp 大小固定 32（`WARP_SIZE` 编译期常量，非配置键）；无其他开关 |
| 调试钩子 | `--trace=subgroup`：每指令一行 `{op, lane, in, out}` |
| trace 契约 | 不进入判据 |
| 契约变更 | subgroup 指令/内建/归约序为冻结项；变更走 ADR 回归 S10 和计算 wave 单测 |
| 校验归属 | L1（结构/掩码/值）+ L2（S10 与 wave 单测对 ref_model） |
| 双策略 | 归约序在两策略下一致（波内原子性天然确定） |
| 阶段衔接 | 为第 30 章组内协作提供"wave 内已完成"的事实；第 31 章策略 B 可并行化 subgroup |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 波内 | 无 → subgroup 原语 | — |
| 归约 | 无 → 固定 lane 序 | —（31 复核） |
| 导数 | quad 已有导数 → subgroup DERIV 统一 | — |

---

## 10. 待定项

- `REDUCE` 在部分 lane 活跃时按 op 定义 identity（MIN=+∞、MAX=−∞），结果只由活跃 lane 决定；该口径已冻结（Q-62），stage3 38~42（MUSA kernel，原 35.5）的"仅活跃归约"语义直接沿用。
- `DERIV` 与第 21 章纹理 LOD 的导数来源是否合并为同一实现点（quad 差分）——本期分别实现，若 S10 探针显示不一致再合并（属重构，不动契约）。
- `Shuffle` 的边界外取值定 0；若某 API（35.x）要求实为"undefined but deterministic"，保持 0 并登记即可，不要求可配置。
