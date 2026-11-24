# 第 33 章 命令链 + 多引擎

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章在 10（单环 CP）、13（Copy 引擎）、32（多队列）之上补齐命令间接层：IB（Indirect Buffer）链、间接绘制/派发（`INDIRECT`）、GPU 侧谓词（`PREDICATE`）、以及多引擎（Copy/DMA）并发。间接与直接结果等价是核心判据；DMA 切片拷贝正确。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（IEngine 握手）；`../design/command-mmio.md` §2（`INDIRECT` 0x0C / `PREDICATE` 0x0D / `stateId 7` IB 基址）；ADR-024（屏障延续）；第 20 章（描述符配合）、第 32 章（队列）。
> 落地模块：`hw_sim/core/cp` 扩展（IB/INDIRECT/PREDICATE）、`hw_sim/periph/copy`（DMA 切片）、`umd/graphics/pal/cmd_gen`、`apps/scene_s12_multiengine`；计算间接包只做单元级解析/参数等价测试，S19 主验收归第 37 章。

---

## 1. 目标与非目标

### 1.1 目标

1. IB 链：`INDIRECT` 指向一片命令缓冲（`stateId 7` 基址），CP 顺序读入执行；IB 内可再含 `INDIRECT`（链式，深度上限防循环）。
2. 间接绘制：`DRAW_INDIRECT` 从显存读 `{vertexCount, instanceCount, firstVertex, firstInstance}`，与直接等值。
3. 间接派发：`DISPATCH_INDIRECT` 从显存读 28-byte、16-byte 对齐的 `{grid[3],block[3],reserved0}`；reserved0 必须为 0，与直接派发语义等价。
4. GPU 侧谓词：`PREDICATE` 从显存读谓词（或查询结果，34 章）→ 条件跳过后续命令。
5. DMA 切片：CopyEngine 支持 `DMA_COPY` 独立的源/目标行步长，完整校验各行范围后执行；重叠区域按本章契约拒绝。
6. 多引擎并发：Copy 与图形可在不同队列并发，属 32 章语义的硬件侧；本章落地 Copy 引擎多实例与 DMA 执行真实重叠。
7. 间接与直接结果等价（探针零容差）。

### 1.2 非目标

- 不做 `ExecuteIndirect` 的命令表多级展开（本期单级 IB + 控件）。
- 不做谓词的复杂链路（只单谓词判定，不嵌套）。
- 不做 DMA 多环复制队列调度优化（复用 32 的 Copy 队列）。
- 不做安全沙箱的地址校验深度（越界即 `NotMapped` 定位）。

---

## 2. 单元框图

```
 命令流（ring）── INDIRECT(ibVa) ──► IB 读（stateId7 基址）
      │  每 IB 同 CP 取指解析执行
    DRAW_INDIRECT ──► 显存: {vertexCount, ...} → 等价 DRAW
    DISPATCH_INDIRECT ──► grid → 等价 DISPATCH
    PREDICATE(pVa) ──► bool → 跳过后续
 DMA（CopyEngine 多实例）：
    DMA_COPY {bytesPerRow, srcStride, dstStride, sliceCount} ──► 行切片拷贝
```

- IB 址基于 `stateId 7`（`SET_STATE_BASE` 设置）；IB 内计 `count` 截断。

---

## 3. 输入输出契约

### 3.1 包族（`command-mmio.md` §2 登记扩展）

| 包 | 载荷 | 语义 |
|---|---|---|
| `INDIRECT`(0x0C) | `{ibVa, ibDwords, flags}` | 读入并执行 IB（链式深度 ≤4） |
| `EXTENDED`(0x0F) `SubOp=0x0010` → `DISPATCH_INDIRECT` | `{paramVa}` | 显存 grid → DISPATCH 等价（B-034） |
| `EXTENDED`(0x0F) `SubOp=0x0011` → `DRAW_INDIRECT` | `{paramVa, stride}` | 显存参数 → DRAW 等价（B-044，`command-mmio.md` §2.1） |
| `PREDICATE`(0x0D) | `{predVa:u64, mask:u64, cond:u64, skipDwords:u32, reserved0:u32}` | `(load64(predVa)&mask)==cond` 执行后续；否则按 skipDwords 跳过完整包序列 |
| `EXTENDED`(0x0F) `SubOp=0x0012` → `DMA_COPY` | `{dstVa:u64, srcVa:u64, bytesPerRow:u32, srcStride:u32, dstStride:u32, sliceCount:u32}`；payload=32 B；只支持行切片，不支持压缩格式或重叠区域 | CopyEngine 切片拷贝（13 章 `COPY_MEM` 的切片扩展，B-035） |

- **取号纪律（ADR-043/045）**：顶层 `PKT_TYPE` 已冻结，仅 `INDIRECT`(0x0C)、`PREDICATE`(0x0D) 有顶层号；`DRAW_INDIRECT`/`DISPATCH_INDIRECT`/`DMA_COPY` 不再从 `0x0F` 预留区领取顶层号，统一走 `EXTENDED`(0x0F)+ SubOp（`0x0010~0x001F` 间接族，`../design/bit-budget.md` §2.3）。SubOp 分配先在 `../roadmap.md` §7.3 登记（`B-nnn`，先对照 bit-budget 审计）再使用（`../design/command-mmio.md` §7 契约变更）。

### 3.2 间接参数布局

```
drawIndirectArgs { u32 vertexCount, instanceCount, firstVertex, firstInstance; }
dispatchIndirectArgs = DispatchArgsIndirect { u32 gridX,gridY,gridZ, blockX,blockY,blockZ, reserved0; } // 7 u32 / 28 B / 16 B-aligned
```

- 计算间接参数必须完整携带 grid 与 block，唯一布局为 28 B、16 B 对齐的 `{grid[3],block[3],reserved0=0}`；不得仅读取 12 B grid。与直接 dispatch 的差异限于参数来源，绑定/管线状态沿用当前状态。

### 3.3 IB 链约束

- `ibDwords` 必须 ≥1；IB 内首包可为任何已登记类型（含 `INDIRECT`）。
- 链循环检测：深度 counter（≤4）防无限嵌套；超限 → `InvalidPacket`。

---

## 4. 内部数据流

### 4.1 IB 执行

```
CP push INDIRECT → 保存当前 ring 位置 → 切换到 IB 读指（ibCursor）
逐包执行（与 ring 语义相同：count/type 校验）→ IB 结束（无 INDIRECT 时）
→ 返回原 ring 位置继续
深度→ SB 计数
```

### 4.2 间接绘制/派发

```
DRAW_INDIRECT：从 paramVa 读 4 u32 → 校验 → 等价构造 DRAW（拓扑用当前 PSO）
DISPATCH_INDIRECT：读 3 u32 → 校验 → 等价 DISPATCH
```

- 校验失败（如 vertexCount 巨大）→ `InvalidArgument` 定位。

### 4.3 谓词

```
PREDICATE(pVa, mask, cond, skipDwords)：value=LoadU64(pVa)
  match=(value & mask)==cond
  match → 从 predicate 之后继续解析
  mismatch → 跳过恰好 skipDwords，并验证跳转目标是包边界
```

- 仅单谓词；谓词读越界 → `NotMapped` 定位。

### 4.4 DMA 切片拷贝

```
params: dstVa, srcVa, bytesPerRow, srcStride, dstStride, sliceCount
validate sliceCount >= 1; bytesPerRow >= 1; srcStride >= bytesPerRow; dstStride >= bytesPerRow
validate arithmetic does not overflow; map every copied row before writing
sourceSpan = [srcVa, srcVa + (sliceCount-1)*srcStride + bytesPerRow)
destSpan   = [dstVa, dstVa + (sliceCount-1)*dstStride + bytesPerRow)
reject intersecting sourceSpan/destSpan unless srcVa == dstVa and srcStride == dstStride
for s in [0,sliceCount):
  copy bytesPerRow bytes: srcVa + s*srcStride → dstVa + s*dstStride
```

- `sliceCount` 与 `bytesPerRow` 必须大于 0，行跨度必须不小于 `bytesPerRow`；地址/跨度加法溢出或源/目标实际覆盖区间重叠时整条命令返回 `InvalidArgument`，任一复制行未映射时返回 `NotMapped`，均不得产生部分写入。同址、同步长视为 no-op。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T33.1 | ① 契约 | IB/DRAW_INDIRECT/DISPATCH_INDIRECT/PREDICATE/DMA_COPY 载荷与布局落头文件 |
| T33.2 | ② 主体 | IB 读入与执行：切换/返回、链深度限制、越界定位 |
| T33.3 | ② 主体 | DRAW_INDIRECT：显存参数读 + 等价 DRAW |
| T33.4 | ② 主体 | DISPATCH_INDIRECT：显存 grid + 等价 DISPATCH |
| T33.5 | ② 主体 | PREDICATE：谓词读 + 条件跳过 |
| T33.6 | ② 主体 | DMA_COPY 切片：srcStride/dstStride 拷贝 + overlap 拒绝校验 |
| T33.7 | ② 主体 | IB 与 ring 混合执行序固定（+下推 state 保持） |
| T33.8 | ③ 接入 | `scene_s12_multiengine`（IB 链 + 间接 DRAW）接入；S19 `case-3` 仅提供间接 DISPATCH 参数与 predicate 输入向量的单元测试数据，不运行计算场景 |
| T33.9 | ④ 验收 | A33：S12 全绿 + 间接计算包单元测试 + 专有判据 + R33 回归项 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D33.1 | 接口独立编译；参数布局 `static_assert`（与 §3.2 一致） | L1 契约 |
| D33.2 | `INDIRECT` 执行完全等价于把 IB 内容直挂到 ring（同一包序列）→ 探针一致 | 横切零容差 |
| D33.3 | `DRAW_INDIRECT(参数=直接DRAW)` 与直接 DRAW 同一拓扑同一顶点结果逐像素一致 | 横切零容差 |
| D33.4 | `DISPATCH_INDIRECT` 解析所得 `DispatchArgs` 与同参数直接 DISPATCH 描述符一致；端到端输出等价由第 37/42 章验收 | 横切零容差 |
| D33.5 | mask/cond 匹配时不跳过；不匹配时恰跳过 skipDwords 指定的完整包序列；目标非包边界、超出 IB/ring 或 reserved0 非零均返回 `InvalidPacket` | L1 契约 |
| D33.6 | DMA 切片：`sliceCount=4, bytesPerRow=32, srcStride=64, dstStride=80` 四行分别写入目标偏移 0/80/160/240，目标间隙保持原值；跨度不足、算术溢出、未映射和源/目标重叠均整条失败且无部分写入 | L1 契约 |
| D33.7 | IB 嵌套 ≤4 处理；>4 → `InvalidPacket`；IB 越界 → `NotMapped`+VA | L1 契约 |
| D33.8 | S12 探针与间接计算包解析/参数向量单测通过；不在本章验收 S19 端到端计算路径 | L2 + A33 |
| D33.9 | A33 全绿：IB 链/间接绘制/predication 与直接等价、间接 dispatch 参数解析与直接包描述符等价（端到端输出等价留第 37/42 章）、DMA 切片拷贝正确 + R33 回归项 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A33）

**绑定场景：S12 `scene_s12_multiengine`（IB + 间接绘制）。S19 `case-3` 只为间接计算包提供参数向量，S19 端到端主验收归第 37 章。**

| 项 | 内容 |
|---|---|
| 输入 | S12：IB 包装的绘制（引用同一顶点；直接对照）；间接计算包单测：S19 `case-3` 的 28 B 参数与 predicate 向量（DMA 切片另以 L1 单测覆盖，D33.6） |
| 产物 | IB 执行日志、间接参数读回、切片拷贝结果 |
| 主判据 | 间接路径与直接路径探针零容差一致 |
| 本章专有判据 | ① IB 链/间接绘制/predication 与直接等价；② 间接 dispatch 与直接等价；③ DMA 切片拷贝正确 |
| 回归项 | R33：复制切片负例不改变 COPY_MEM 基线；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | GPU 侧谓词回读查询（34 章）；多条 IB 间 fences（32 语义延续） |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | IB 执行序=IB 内包序=直接等价序；谓词读序固定 |
| 时间无关 | 无时间输入 |
| 可观测 | IB 深度、间接参数、predicate 值、DMA 切片计数、`--trace=cp/indirect` |
| 契约影响 | IB/DRAW_INDIRECT/PREDICATE/DMA_COPY 载荷冻结 |
| 简化登记 | 单命令直发 → IB 链 + 间接 + 谓词 + DMA 切片 |
| 计算兼容 | `DISPATCH_INDIRECT` 计算共用；谓词对 DRAW/DISPATCH 同构 |
| 配置与回退 | 内部选项 `IbOptions::depthLimit`（默认 4）、`IbOptions::indirectEnable`（默认开），非 §5.1 配置键 |
| 调试钩子 | `--trace=ib`（进入/返回深度）；`--trace=predicate`（值/跳过量） |
| trace 契约 | 不进入判据 |
| 契约变更 | IB/DRAW_INDIRECT/PREDICATE/DMA 切片为冻结项；变更走 ADR 回归 S12 与间接包单测 |
| 校验归属 | L1（包格式/IB/间接计算参数）+ L2（S12）；S19 场景级计算结果归第 37 章 |
| 双策略 | 间接子任务在策略 A/B 下等价成立（执行序确定） |
| 阶段衔接 | 为 34（谓词回读查询）、35.2（API 间接绘制）、36（IB 越界故障注入）提供间接层 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 命令 | 单流直发 → IB 链 | — |
| 绘制 | 直接 → 间接参数 | — |
| 派发 | 直接 → 间接 grid | — |
| 谓词 | 无 → PREDICATE | — |
| DMA | 块拷贝 → 支持独立 `srcStride` / `dstStride` 的行切片 | — |

---

## 10. 待定项

- IB 链的 `stateId 7` 基址与 `DRAW_INDIRECT` 的 paramVa 相对/绝对寻址语义：本期绝对 VA；35.2（Vulkan `vkCmdDrawIndirect`）可能要求相对，届时由 ADR 记录切换。
- PREDICATE 的跳过边界已冻结为 `skipDwords` 个 DWORD、目标必须为同一 IB/ring 段内合法包边界；本期不支持越段跳转。
- 负步长不在 `DMA_COPY` 的无符号字段范围内；镜像拷贝不属于本期能力。

---

## 2026-11-21 评审同步说明

DISPATCH_INDIRECT 参数与 BARRIER 载荷分别见 [命令包登记 §2.1、§2.3](../design/command-mmio.md)。旧 12-byte grid-only 间接参数布局已退役。
