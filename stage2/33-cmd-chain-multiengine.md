# 第 33 章 命令链 + 多引擎

> 本章在 10（单环 CP）、13（Copy 引擎）、32（多队列）之上补齐命令间接层：IB（Indirect Buffer）链、间接绘制/派发（`INDIRECT`）、GPU 侧谓词（`PREDICATE`）、以及多引擎（Copy/DMA）并发。间接与直接结果等价是核心判据；DMA 切片拷贝正确。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（IEngine 握手）；`../design/command-mmio.md` §2（`INDIRECT` 0x0C / `PREDICATE` 0x0D / `stateId 7` IB 基址）；ADR-024（屏障延续）；第 20 章（描述符配合）、第 32 章（队列）。
> 落地模块：`hw_sim/core/cp` 扩展（IB/INDIRECT/PREDICATE）、`hw_sim/periph/copy`（DMA 切片）、`umd/graphics/pal/cmd_gen`、`apps/scene_s12_multiengine`、`apps/scene_s19_compute_memory`。

---

## 1. 目标与非目标

### 1.1 目标

1. IB 链：`INDIRECT` 指向一片命令缓冲（`stateId 7` 基址），CP 顺序读入执行；IB 内可再含 `INDIRECT`（链式，深度上限防循环）。
2. 间接绘制：`DRAW_INDIRECT` 从显存读 `{vertexCount, instanceCount, firstVertex, firstInstance}`，与直接等值。
3. 间接派发：`DISPATCH_INDIRECT` 从显存读 `{gridX,Y,Z}`，与直接等值。
4. GPU 侧谓词：`PREDICATE` 从显存读谓词（或查询结果，34 章）→ 条件跳过后续命令。
5. DMA 切片：CopyEngine 支持 `COPY_MEM` 的切片（stride/深度）语义（`DMA_COPY`），切片区域正确。
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
    COPY_MEM 切片（stride/depth）──► 区间拷贝
```

- IB 址基于 `stateId 7`（`SET_STATE_BASE` 设置）；IB 内计 `count` 截断。

---

## 3. 输入输出契约

### 3.1 包族（`command-mmio.md` §2 登记扩展）

| 包 | 载荷 | 语义 |
|---|---|---|
| `INDIRECT`(0x0C) | `{ibVa, ibDwords, flags}` | 读入并执行 IB（链式深度 ≤4） |
| `EXTENDED`(0x0F) `SubOp=0x0010` → `DISPATCH_INDIRECT` | `{paramVa}` | 显存 grid → DISPATCH 等价（B-034） |
| `EXTENDED`(0x0F) `SubOp=0x0011` → `DRAW_INDIRECT` | `{paramVa, stride}` | 显存参数 → DRAW 等价（`B-0nn` 待登记，`command-mmio.md` §2.1） |
| `PREDICATE`(0x0D) | `{predVa, flags(cond)}` | `if (pred==cond) 执行后续；else skip` |
| `EXTENDED`(0x0F) `SubOp=0x0012` → `DMA_COPY` | `{dstVa, srcVa, bytes, stride, sliceCount}` | CopyEngine 切片拷贝（13 章 `COPY_MEM` 的切片扩展，B-035） |

- **取号纪律（ADR-043/045）**：顶层 `PKT_TYPE` 已冻结，仅 `INDIRECT`(0x0C)、`PREDICATE`(0x0D) 有顶层号；`DRAW_INDIRECT`/`DISPATCH_INDIRECT`/`DMA_COPY` 不再从 `0x0F` 预留区领取顶层号，统一走 `EXTENDED`(0x0F)+ SubOp（`0x0010~0x001F` 间接族，`../design/bit-budget.md` §2.3）。SubOp 分配先在 `../roadmap.md` §7.3 登记（`B-nnn`，先对照 bit-budget 审计）再使用（`../design/command-mmio.md` §7 契约变更）。

### 3.2 间接参数布局

```
drawIndirectArgs { u32 vertexCount, instanceCount, firstVertex, firstInstance; }
dispatchIndirectArgs { u32 gridX, gridY, gridZ; }
```

- 与直接相比不含拓扑/绑定等（复用当前绑定状态）。

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
PREDICATE(pVa, cond)：读 bool → p==cond ?
  是 → 执行后续命令（当前 IB/ring 不跳过）
  否 → 跳过后续直到本命令流的 `END`/NOP 段（跳过量由 flags/包给出）
```

- 仅单谓词；谓词读越界 → `NotMapped` 定位。

### 4.4 DMA 切片拷贝

```
params: dst, src, bytes, stride, sliceCount
for s in [0,sliceCount):
  行拷贝 limbs：src + s*stride → dst + s*stride（等 stride）
（overlap/未映射 按 COPY_MEM 规则）
```

- 切片与普通 `COPY_MEM`（无 stride）语义一致，overlap/未映射按第 13 章 `COPY_MEM` 规则。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T33.1 | ① 契约 | IB/DRAW_INDIRECT/DISPATCH_INDIRECT/PREDICATE/DMA_COPY 载荷与布局落头文件 |
| T33.2 | ② 主体 | IB 读入与执行：切换/返回、链深度限制、越界定位 |
| T33.3 | ② 主体 | DRAW_INDIRECT：显存参数读 + 等价 DRAW |
| T33.4 | ② 主体 | DISPATCH_INDIRECT：显存 grid + 等价 DISPATCH |
| T33.5 | ② 主体 | PREDICATE：谓词读 + 条件跳过 |
| T33.6 | ② 主体 | DMA_COPY 切片：stride/slice 拷贝 + overlap 校验 |
| T33.7 | ② 主体 | IB 与 ring 混合执行序固定（+下推 state 保持） |
| T33.8 | ③ 接入 | `scene_s12_multiengine`（IB 链 + 间接 DRAW）、`scene_s19_compute_memory`（`case-3` 间接 DISPATCH + 谓词）接入 |
| T33.9 | ④ 验收 | A33：S12/S19 全绿 + 专有判据 + A32 回归 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D33.1 | 接口独立编译；参数布局 `static_assert`（与 §3.2 一致） | L1 契约 |
| D33.2 | `INDIRECT` 执行完全等价于把 IB 内容直挂到 ring（同一包序列）→ 探针一致 | 横切零容差 |
| D33.3 | `DRAW_INDIRECT(参数=直接DRAW)` 与直接 DRAW 同一拓扑同一顶点结果逐像素一致 | 横切零容差 |
| D33.4 | `DISPATCH_INDIRECT(grid)` 与直接 DISPATCH 输出一致 | 横切零容差 |
| D33.5 | `PREDICATE(true=true)` 全执行；`false` 全跳过（无副作用计数为 0） | L1 契约 |
| D33.6 | DMA 切片：`sliceCount=4, stride=64` 各切片正确；区域重叠按规则 | L1 契约 |
| D33.7 | IB 嵌套 ≤4 处理；>4 → `InvalidPacket`；IB 越界 → `NotMapped`+VA | L1 契约 |
| D33.8 | S12/S19 探针通过（v33 主线） | L2 + A33 |
| D33.9 | A33 全绿：IB 链/间接绘制/predication 与直接等价、间接 dispatch 与直接等价、DMA 切片拷贝正确 + 回归 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A33）

**绑定场景：S12 `scene_s12_multiengine`（IB + 间接绘制）、S19 `scene_s19_compute_memory`（`case-3` 间接派发 + 谓词）。**

| 项 | 内容 |
|---|---|
| 输入 | S12：IB 包装的绘制（引用同一顶点；直接对照）；S19 `case-3`：GPU 侧参数 buffer 生成的间接 DISPATCH + 谓词（DMA 切片以 L1 单测覆盖，D33.6） |
| 产物 | IB 执行日志、间接参数读回、切片拷贝结果 |
| 主判据 | 间接路径与直接路径探针零容差一致 |
| 本章专有判据 | ① IB 链/间接绘制/predication 与直接等价；② 间接 dispatch 与直接等价；③ DMA 切片拷贝正确 |
| 回归项 | A1~A32 全绿；复制切片负例不改变 COPY_MEM 基线 |
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
| 契约变更 | IB/DRAW_INDIRECT/PREDICATE/DMA 切片为冻结项；变更走 ADR 回归 S12/S19 |
| 校验归属 | L1（结构/计数）+ L2（间接输出一比直接）+ 横切（等价） |
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
| DMA | 块拷贝 → 切片/stride | — |

---

## 10. 待定项

- IB 链的 `stateId 7` 基址与 `DRAW_INDIRECT` 的 paramVa 相对/绝对寻址语义：本期绝对 VA；35.2（Vulkan `vkCmdDrawIndirect`）可能要求相对，届时由 ADR 记录切换。
- PREDICATE 的"跳过分量"精确边界（跳过至 IB 尾 vs 命令段尾）本期取 IB/ring 段尾；若 34 章查询谓词需要"跳过 N 条"则扩展 flags。
- DMA 切片与普通 COPY_MEM 的 overlap 语义统一后，若出现负 stride 场景（镜像拷贝）本期拒绝，留能力位。
