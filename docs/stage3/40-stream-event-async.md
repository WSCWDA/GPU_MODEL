# 第 40 章 Stream/Event 与异步计算

> 本章是 **stage3 计算驱动开发线（38~42）** 的第三章。在第 38/39 章基础上实现计算驱动的**异步执行语义**：`stream` = 1:1 映射一个 HLI 队列（优先级创建时声明、不可变；不允许多队列）；`event` = `{queueId, timelineValue, state}`（Record 插 `SIGNAL`、Wait 插 `WAIT`、Query 非阻塞读，值域复用队列 timeline、不复用 fence 对象）；**async compute**：计算队列与图形队列并发提交、计算→图形 fence 依赖（队列家族所有权转移复用第 32 章）、timeline 值序。全程复用第 32 章队列/同步原语，不新增 HLI 接口、不新增包号。
> 设计权威：Stream/Event 语义见 `../design/compute-driver.md` §6；队列与 timeline、所有权转移见第 32 章与 `../architecture-design.md` §3.13（图形↔计算行）；同步原语与内存作用域见 `../design/sync-primitives.md` §3；WA-11 见 `../architecture-design.md` §7.9，计算通路见 `../design/compute-line.md` §1/§2。
> 编号纪律：本章不新增编号；复用 `SIGNAL`/`WAIT`（第 32 章）、`BARRIER=0x0E`、`PREDICATE=0x0D`（ADR-043/047）；不新增 HLI 接口（ADR-032）；开工前对照 `../design/bit-budget.md` 审计（ADR-045）。

---

## 1. 目标与非目标

### 1.1 目标

1. **stream 对象落地**：`StreamCreate/Destroy`（优先级声明且不可变、1:1 队列映射）、stream 内顺序 = 命令流顺序、`LaunchKernel(stream,...)` 顺序提交。
2. **event 对象落地**：`Record`（插 `SIGNAL`）、`Wait`（插 `WAIT`）、`Query`（非阻塞状态读，未达值返回未完成）；值域复用队列 timeline。
3. **async compute 与互操作**：计算队列与图形队列并发提交；计算 → 图形 fence 依赖；`WAIT` 前图形不读计算产物；跨队列家族所有权转移（第 32 章）后可见性正确。
4. **Demo C3 闭环**：`async_pipeline`（WA-11 多 kernel 流水 + async compute）在双队列下数值等价（与串行硬判据一致）。

### 1.2 非目标

- 不做组内协作 / 间接派发 / UAV 写回的驱动侧收口（第 41 章）；不做 S18/S19 全量双入口验收（第 42 章）。
- 不允许多队列 / 动态改优先级；不做计算专用队列优先级策略（ADR-032 `core/cp/ring` 边界）。
- 不做抢占（二期）；不打乱提交次序来满足依赖（依赖只由 fence/semaphore 定义）。

---

## 2. 单元框图

```
apps（Demo C3 async_pipeline）
   │  图形队列（绘制）+ 计算队列（多 kernel 流水）并发
   ▼
umd/compute::Stream（1:1 → IHliQueue）
   ├─ LaunchKernel(stream, fn, grid, block, args[])   // 顺序提交（第 39 章）
   └─ Event.Record(stream) → 命令流插 SIGNAL(timelineValue)
       Event.Wait(stream)  → 命令流插 WAIT(timelineValue)
       Event.Query()       → 非阻塞读 state（队列 timeline 值域）
   ▼
IHliQueue.Submit（第 32 章队列）──► kMD/FW → ring → CP
   │  计算→图形依赖：SIGNAL/WAIT + 队列家族所有权转移 + BARRIER(scope=Device)
   ▼
fence / 回读 → 断言：顺序语义 + 数值等价（serial/parallel 一致）
```

说明：

- stream 是驱动对第 32 章队列的**薄封装**（1:1），所有同步语义仍由 `SIGNAL/WAIT`/timeline 定义，驱动不自造提交次序假设（M-07、§3.13）。
- event 值域复用队列 timeline，**不复用 fence 对象**（`compute-driver.md` §6）；`Query` 非阻塞、不隐式同步。

---

## 3. 输入输出契约

### 3.1 Stream（权威：`compute-driver.md` §6、第 32 章）

```cpp
class Stream {
    IHliQueue* queue;         // 1:1 映射；构造时声明 priority，不可变
    QueuePriority priority;   // {Default}（一期仅 Default；枚举域以 compute-driver.md 为准），创建后改动 → InvalidArgument
};
// 提交顺序 = 命令流顺序；单 stream 内不支持多任务并行组装
```

### 3.2 Event（权威：`compute-driver.md` §6）

```cpp
struct EventDesc { u32 queueId; u32 timelineValue; EventState state; };
// Record(stream): 命令流插 SIGNAL(timelineValue)
// Wait(stream):   命令流插 WAIT(timelineValue)
// Query():        仅读状态；未达值返回 NotReached（非阻塞，不等 fence）
```

### 3.3 异步与互操作契约（第 32 章 + `architecture-design.md` §3.13"图形 ↔ 计算"行）

- 计算队列与图形队列并发提交；**依赖只由 fence / semaphore 定义**，不依赖提交次序；
- 计算 → 图形可见性 = 队列家族所有权转移（第 32 章）+ fence + `BARRIER(scope=Device)`；
- 图形在 `WAIT` 前不得读计算产物（读旧值/竞态即判据失败）；
- 所有权转移后同一缓冲的多读路径（图形采样 + 计算读）必须经显式同步建立（§3.13 一致性矩阵）。

### 3.4 错误码映射

- `StreamCreate` 优先级非法 / 队列不可用 → `InvalidArgument` → `MUSA_ERROR_INVALID_VALUE`；
- `EventQuery` 不产生错误码（状态枚举驱动）；
- `Record` 的 `stream.queueId` 必须等于 `event.queueId`（否则 `InvalidArgument`）；`Wait` 允许跨队列（timeline semaphore 语义，§4.1 的 `t1` 即跨队列等待）；
- 非法仅限：未映射事件、同队列自等待、等待值回退（`<` 当前 timeline）。

---

## 4. 内部数据流或状态机

### 4.1 stream/event 时序

```
t0 计算队列: LaunchKernel(k1) → Event.Record(e1)      [提交]
t1 图形队列: 绘制 draw → Event.Wait(e1) → 读 UAV    [依赖 e1≥v]
t2 计算队列: LaunchKernel(k2) → Event.Record(e2)
t3 回读:   Event.Wait(e2) 阻塞至完成 → Query(e2)=Reached → 数值比对
```

### 4.2 event 状态

```
Event.state: NotReached（timelineValue 未达） → Reached（值 ≥ timelineValue）
Query：只读，不推进（主动同步用 Wait / fence）
```

### 4.3 错误处理

- 依赖错误（如 `WAIT` 前读计算产物）为**确定性判据失败**（探针/数值比对），非驱动崩溃；
- timeline 回卷 / 值序错误在提交期 `InvalidArgument`（值序单调由第 32 章保证）。

---

## 5. 任务清单

> 任务编号 `T40.1~T40.6`，四段式。

| 子里程碑 | 任务 | 段 | 内容 |
|---|---|---|---|
| M40.1 | T40.1 | ① 契约 | `umd/compute` Stream/Event 头文件（对齐 `compute-driver.md` §6；stream 1:1 队列、优先级不可变、event 字段 `{queueId,timelineValue,state}`） |
| M40.1 | T40.2 | ② 主体 | Stream 提交封装：`LaunchKernel(stream,...)` 顺序提交 → `IHliQueue.Submit`；stream 内顺序 = 命令流顺序 |
| M40.1 | T40.3 | ② 主体 | Event Record/Wait/Query：命令流插 `SIGNAL/WAIT`、Query 非阻塞读；值域复用队列 timeline |
| M40.2 | T40.4 | ② 主体 | async compute：计算队列与图形队列并发、计算 → 图形 fence 依赖、`WAIT` 前图形不读计算产物（复用第 32 章 + 所有权转移） |
| M40.2 | T40.5 | ③ 接入 | Demo C3 `async_pipeline` 双队列并发跑通；S12/S16 回归接入 |
| M40.2 | T40.6 | ④ 验收 | A40：event Record/Wait/Query 值序正确、未达值 Query 非阻塞返回未完成；stream 提交顺序 = 命令流顺序；异步交叉下数值等价（serial/parallel 硬判据一致） |

### 子里程碑

| 子里程碑 | 范围 | 承载任务号 |
|---|---|---|
| M40.1 stream/event | Stream/Event 头文件、Stream 提交封装、Record/Wait/Query | T40.1~T40.3 |
| M40.2 异步与验收 | async compute、Demo C3、A40 | T40.4~T40.6 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D40.1 | Stream/Event 头文件独立编译；优先级不可变、1:1 队列映射单测（改优先级/第二队列 → `InvalidArgument`） | L1 契约 |
| D40.2 | stream 内提交顺序 = 命令流顺序（提交日志按序断言）；`LaunchKernel` 经 stream 提交包序列正确 | L1 契约 |
| D40.3 | Record 后命令流含 `SIGNAL(v)`、Wait 含 `WAIT(v)`；Query 未达值非阻塞返回 NotReached（不等 fence 不卡死） | L1 契约 |
| D40.4 | 计算队列与图形队列并发提交；图形 `WAIT` 前不读计算产物（探针因竞态读旧值即失败）；timeline 值序正确 | L1 + L2 |
| D40.5 | Demo C3 `async_pipeline`（WA-11）数值等价：与串行参考逐值一致；S12/S16 回归不退化 | 横切 |
| D40.6 | A40 全绿（D40.1~D40.5 + `serial`/`parallel` 硬判据一致） | L1 + L2 + 横切 |

---

## 7. 场景与判据（A40）

**绑定：Demo C3 `async_pipeline`（WA-11，本阶段主判据）；S19（计算内存与并发）相关能力验证 + S12/S16 回归。**

| 项 | 内容 |
|---|---|
| 输入 | 图形队列绘制 + 计算队列多 kernel 流水；event/fence 交叉；`exec_mode ∈ {serial, parallel}` |
| 产物 | 双队列提交日志、SIGNAL/WAIT 序列、命令包序列、回读探针表 |
| 主判据 | event 值序正确；`WAIT` 前图形不读计算产物；异步交叉下数值等价（与串行参考一致，零容差） |
| 本章专有判据 | Query 非阻塞（未达值返回未完成）；stream 顺序 = 命令流顺序；不新增队列/配置键 |
| 回归项 | A1~A39 全绿；S12（多引擎交错）、S16（屏障 + 所有权转移）在接入后不退化 |

> 场景输入与探针单份规约见 `../design/scene-specs.md` §3（S19 节）与 §4（第 40 行）。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 依赖只由 fence/semaphore 定义；归并无时间输入；两次运行依赖序一致 |
| 时间无关 | 双队列推进不依赖墙钟；`WAIT` 由 timeline 值而非时间触发 |
| 可观测 | `--trace=compute`：提交日志、SIGNAL/WAIT 序列、event 状态机 |
| 契约影响 | 不新增编号/队列/配置键；timeline 值域复用第 32 章 |
| 简化登记 | 单 stream 单队列（不允许多队列）；优先级不可变；无抢占 |
| 计算兼容 | stream 不做计算专用队列优先级策略（ADR-032 CP 边界） |
| 配置与回退 | 不新增配置键；`api_backend=hli / compute` 均支持异步 |
| 调试钩子 | 双队列日志、event 状态 dump；只供人工 |
| trace 契约 | `--trace` 不保证字段稳定、不机器解析、不进判据 |
| 契约变更 | event/stream 语义走 ADR，回归 S12/S16/S19 |
| 校验归属 | L1（值序/阻塞）+ L2（数值等价）+ 横切（S12/S16 回归） |
| 双策略 | API 交互在 `serial`/`parallel` 下硬判据一致（依赖由值定义、非次序） |

---

## 9. 简化登记

| 项 | 状态 | 替换章 / 备注 |
|---|---|---|
| 多队列 stream | 不做（1:1 队列） | `compute-driver.md` §6 |
| 优先级 | 创建时声明、不可变 | 同上 |
| 计算抢占 / TDR | 二期 | ADR-025 二期候选（capability bit） |
| async copy（直通 LDS） | 二期候选 | §7.9 二期候选，未获 ADR 前 `NotSupported` |

---

## 10. 待定项

- 跨 stream 隐式依赖（应用语义层面）一期不做：所有依赖显式经 `Event`/fence 表达（M-07 同源）。
- `Event` 复用池化与价值域基线化属于工程优化，不影响契约。
