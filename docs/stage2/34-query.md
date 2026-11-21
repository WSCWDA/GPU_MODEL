# 第 34 章 查询

> 项目范围、阶段位置和交付以[任务规划](../CPU模拟GPU教学项目_任务规划与实施规格.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章落地 GPU 查询子系统：遮挡查询（occlusion）、时间戳（timestamp）、管线统计（pipeline stats）、以及查询作为 GPU 侧谓词（`PREDICATE` 回读）。查询值进探针/断言面（计数零容差）；归约顺序固定；timestamp 只做单调性判定。可与 32/33 并行推进（roadmap §3.2）。
> 与本章相关的设计说明：`../architecture-design.md` §3.4（`IHliQueryPool`）；`../design/command-mmio.md` §2（`QUERY` 0x09、`stateId 6` 查询池基址）；ADR-027（查询为硬判据）；第 27 章（遮挡与 early-Z/Hi-Z 计数）、第 31 章（并行波/像素计数）。
> 落地模块：`hw_sim/query`、`apps/scene_s7_occlusion`、`apps/scene_s2_clear`。

---

## 1. 目标与非目标

### 1.1 目标

1. 遮挡查询：`QUERY_OCCLUSION begin/end` 统计通过深度测试的像素数（可带 early-Z）。
2. 时间戳：在命令流中打点 `QUERY_TIMESTAMP` 记录逻辑执行序号（`frame_index`/fence 相关），只做单调性判定（不做墙钟时间）。
3. 管线统计：顶点数/图元数/VS 波数/PS 像素数/深度通过等硬计数（零容差断言），与硬件统计 MMIO 分开（查询 → 显存；统计 → MMIO 软观测）。
4. 查询池：`QUERY(stateId 6)` 基址 + 池对象（`IHliQueryPool`）；`begin/end/resolve`。
5. GPU 侧谓词：查询结果作 `PREDICATE` 输入（跳过后续命令，33 章语义）。
6. 查询归约顺序固定：多单元计数在同一查询点累加顺序确定（lane 升序）。

### 1.2 非目标

- 不做条件渲染链的多级嵌套（一期单 begin/end 一段）。
- 不做时间戳的实际时钟读取（墙钟）；只逻辑序。
- 不做管线统计的 API 特化打包（35.2 映射可调整布局，语义不变）。

---

## 2. 单元框图

```
 命令流（ring/IB）── QUERY(begin) ──► 查询单元（累计器）
                     ◄ 中间命令 ── 目标引擎计数上报
                     QUERY(end) ──► 写入查询池（stateId6 + offset）
  HOST：QueryPool.GetResult(offset) → 值（零容差断言）
  GPU：PREDICATE 引 查询池 → 条件跳过
```

- 查询池 `{u64 values[N]}`：写由硬件累加器完成，无并发写序列。

---

## 3. 输入输出契约

### 3.1 QUERY 包（`command-mmio.md` §2 登记扩展）

```cpp
enum QueryType { Occlusion, Timestamp, PipelineStatsBegin, PipelineStatsEnd, PatchStats };
struct QueryPkt {  // QUERY(0x09)
    u32 type;
    u32 poolOffset;   // 相对于查询池基址的 u64 偏移
    u32 flags;        // 如 'resolve now'?
};
```

- `begin` 清 0（或为 stats 做累计基线）；`end` 写值。
- resolve：读回查询池对应槽（`stateId 6` 基址 + offset）。

### 3.2 查询池接口（`hw_sim/query/query_pool.h`）

```cpp
class QueryPool {
public:
    Result Begin(u32 type, u32 poolOffset);
    Result End(u32 type, u32 poolOffset);
    Result GetResult(u32 poolOffset, u64* out) const;   // 主机读回
    Result Reset(u32 poolOffset);
};
```

- 值 `u64`；`GetResult` 返回最终计数（零容差断言面）。
- 未 begin 就 End → `InvalidArgument`。

### 3.3 谓词回读

```
PREDICATE { predVa=查询池槽 }：判定逻辑与 33 章一致
```

---

## 4. 内部数据流

### 4.1 遮挡查询

```
QUERY.Pool[o].begin → 累计器归零
引擎处理像素：depth 通过 → cum++
QUERY.end → Pool[o] = cum（若期间有 begin/end 跨多提交，值持续累计）
GetResult(o) → 计数
```

- 计数含 early-Z 拒绝后通过的（27 章）；口径 = "实际写出的像素"。

### 4.2 时间戳（逻辑序）

```
TIMESTAMP 点 → counter = 逻辑执行序列号（由 fence/frame 推导）
GetResult → 单调（非必须累计，只判 >=）
```

- 不做墙钟；只判单调 `value[i+1] >= value[i]`。

### 4.3 管线统计

```
stat 与查询绑定：记录了 图元数/VS波数/PS像素等（对齐 23/25/26 章统计）
PipelineStatsBegin 记录基线，End 时写差值（= 这段新增）
```

- 与 MMIO `HW_STATS`（软观测）分开：查询是硬判据。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T34.1 | ① 契约 | `QueryType`/`QueryPkt`/`QueryPool`/查询池布局/`stateId 6` 落头文件 |
| T34.2 | ② 主体 | 查询池（`{u64[N]}` 基址、Reset、GetResult） |
| T34.3 | ② 主体 | 遮挡查询：begin/end/深度通过累计 |
| T34.4 | ② 主体 | 时间戳：逻辑序打点 + 单调判定 |
| T34.5 | ② 主体 | 管线统计：between begin/end 差值 |
| T34.6 | ② 主体 | 查询归约顺序固定（多单元累计序） |
| T34.7 | ② 主体 | 查询作 GPU 侧谓词（`PREDICATE` 回读） |
| T34.8 | ③ 接入 | `scene_s7_occlusion`（遮挡 + predicate + 管线统计）、`scene_s2_clear`（时间戳打点）接入 |
| T34.9 | ④ 验收 | A34：S7/S2 全绿 + 专有判据 + A33 回归 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D34.1 | 接口独立编译；`QueryType` 枚举冻结 | L1 契约 |
| D34.2 | `GetResult` 默认 0；`Reset` 后 0；begin/end 次序错 → `InvalidArgument` | L1 契约 |
| D34.3 | 遮挡：全遮挡三角形 → 0；半遮挡 → 正计数 = 深度通过数（与 27 章一致） | L2 解析 |
| D34.4 | 时间戳：两打点值 `v2>=v1`（单调）| L1 契约 |
| D34.5 | 管线统计：`VS 波数` = Σ（同 23 章）；`像素通过` = 同 26 章 | L1 契约 |
| D34.6 | 同场景两次运行查询值一致（归约序固定） | L1 契约 |
| D34.7 | `PREDICATE`：遮挡=0 → 后续命令被跳过（计数为 0）；非 0 → 执行 | L1 契约 |
| D34.8 | S7/S2 探针通过 | L2 + A34 |
| D34.9 | A34 全绿：遮挡 query=0、timestamp 单调不减、统计不参与正确性、查询作 GPU 侧谓词 + 回归 | L1 + L2 |

---

## 7. 场景与判据（A34）

**绑定场景：S7 `scene_s7_occlusion`（遮挡 + predicate + 管线统计）、S2 `scene_s2_clear`（时间戳逻辑序）。**

| 项 | 内容 |
|---|---|
| 输入 | S7：遮挡金字塔场景 + 全遮挡游标 + 统计计数；S2：清屏命令流中插入 `QUERY_TIMESTAMP` 打点序列 |
| 产物 | 查询池槽值、predicate 跳转日志、时间戳序列 |
| 主判据 | 查询值零容差（遮挡数、统计数）；timestamp 单调 |
| 本章专有判据 | ① 遮挡 query=0（全遮场景）；② timestamp 单调不减；③ 统计不参与正确性（其值不误导）；④ 查询作 GPU 侧谓词（predicate 正确） |
| 回归项 | A1~A33 全绿；早期统计（23/25/27）口径不变 |
| 后续补齐 | 35.2 三套 UMD 的查询 API 映射以本池为基础 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 累计序固定（lane/+引擎序）；查询值与执行序对应 |
| 时间无关 | 时间戳用逻辑序，非墙钟 |
| 可观测 | 查询池快照、`--trace=query` |
| 契约影响 | `QueryType/QueryPkt`/`QueryPool`/谓词回读冻结 |
| 简化登记 | 无查询 → 遮挡/时间戳/管线统计/谓词 |
| 计算兼容 | 查询对图形/计算统计均适用（共用累加器） |
| 配置与回退 | 查询池容量与谓词开关为 `QueryPool` 构造选项（非 §5.1 配置键） |
| 调试钩子 | `--trace=query`：begin/end/value 行 |
| trace 契约 | 不进入判据 |
| 契约变更 | 查询布局/语义/谓词回读为冻结项；变更走 ADR 回归 S7/S2 |
| 校验归属 | L1（结构/单调）+ L2（遮挡/统计与 ref_model） |
| 双策略 | 查询累计在两策略下一致（计数序固定） |
| 阶段衔接 | 为 35.2~35.4（API 查询）、stage3 38~42（event 用 fence 非 query，原 35.5）提供硬件查询面 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 查询 | 无 → 遮挡/时间戳/统计/谓词 | — |
| 谓词 | 显存 bool → 查询池槽 | — |
| 时间戳 | 无 → 逻辑序 | — |

---

## 10. 待定项

- 遮挡查询的 early-Z 交互口径（"通过 early-Z 且写出的像素" vs "通过 depth 测试"）以 27 章为准；S7 若对该口径敏感，明确写入 spec 判定（记录决策不留歧义）。
- 时间戳的单调基准（按 fence timeline 而非帧）在 35.2（Vulkan timestamp）可能需要换算，需在其映射层做比例，硬件只保逻辑序。
- 查询池多引擎并发写同一池槽竞争：本期语义为"合并累计"，若 S12 出现竞争差异，采用逐引擎分区再合并（登记）。
