# 第 40 章 Stream/Event 与异步计算

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章是 **stage3 计算驱动开发线（38~42）** 的第三章。在第 38/39 章基础上实现计算驱动的**异步执行语义**：`stream` = 1:1 映射一个 HLI 队列（优先级创建时声明、不可变；一个 stream 不绑定多个队列，不同队列家族可各有 stream）；`event` 拥有独立 timeline semaphore（Record 插 `SIGNAL`、Wait 插 `WAIT`、Query 非阻塞读；event value 单调且可跨队列等待，不复用队列 fence 值域）；**async compute**：计算队列与图形队列并发提交、计算→图形 fence 依赖（队列家族所有权转移复用第 32 章）、timeline 值序。全程复用第 32 章队列/同步原语，不新增 HLI 接口、不新增包号。
> 设计权威：Stream/Event 语义见 `../design/compute-driver.md` §6；队列与 timeline、所有权转移见第 32 章与 `../architecture-design.md` §3.13（图形↔计算行）；同步原语与内存作用域见 `../design/sync-primitives.md` §3；WA-11 见 `../architecture-design.md` §7.9，计算通路见 `../design/compute-line.md` §1/§2。
> 编号纪律：本章不新增编号；复用 `SIGNAL`/`WAIT`（第 32 章）、`BARRIER=0x0E`、`PREDICATE=0x0D`（ADR-043/047）；不新增 HLI 接口（ADR-032）；开工前对照 `../design/bit-budget.md` 审计（ADR-045）。

---

## 1. 目标与非目标

### 1.1 目标

1. **stream 对象落地**：`StreamCreate/Destroy/Synchronize`（优先级声明且不可变、每队列一个 stream wrapper）；stream 内顺序 = 命令流顺序；`Synchronize` 显式 drain，pending 时 `StreamDestroy`/有 live Stream 时 `ContextDestroy` 返回 `Busy`。
2. **event 对象落地**：Event 自有 timeline semaphore 与单调 `u64` 值；`Record` 为该 Event 预留值并插 `SIGNAL`，`Wait` 可跨队列等待最近一次已记录值并插 `WAIT`，`Query` 非阻塞读取 Event 完成值。
3. **async compute 与互操作**：计算队列与图形队列并发提交；计算 → 图形 fence 依赖；`WAIT` 前图形不读计算产物；跨队列家族所有权转移（第 32 章）后可见性正确。
4. **Demo C3 闭环**：`async_pipeline`（WA-11 多 kernel 流水 + async compute）在双队列下数值等价（与串行硬判据一致）。

### 1.2 非目标

- 不做组内协作 / 间接派发 / UAV 写回的驱动侧收口（第 41 章）；不做 S18/S19 双入口矩阵验收（第 42 章）。
- 一个 stream 不绑定多个队列；Graphics 与 Compute 等不同队列可并存各自的 stream。动态改优先级与计算专用队列优先级策略不做（ADR-032 `core/cp/ring` 边界）。
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
- event Event 自有 timeline semaphore，不复用队列 timeline 值域，**不复用 fence 对象**（`compute-driver.md` §6）；`Query` 非阻塞、不隐式同步。

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
struct EventDesc { SemaphoreHandle timeline; u64 nextValue; u64 lastRecordedValue; };
// Record(stream): first accepted record binds producer; reserve value and emit SIGNAL
// Wait(stream):   snapshot lastRecordedValue; emit WAIT(event.timeline, value)
// Query(): inspect completion of lastRecordedValue; return pending, reached, or failure without blocking
```

### 3.3 异步与互操作契约（第 32 章 + `architecture-design.md` §3.13"图形 ↔ 计算"行）

- 计算队列与图形队列并发提交；**依赖只由 fence / semaphore 定义**，不依赖提交次序；
- 计算写入后按命令顺序记录整资源 release 和 Event SIGNAL，图形依次 WAIT、acquire、读资源。描述和缓存维护见 [同步原语 §2.2](../design/sync-primitives.md)。
- 图形在 WAIT 成功且 acquire 完成后访问目标。
- 若图形消费后将资源交还计算，按相反的源、目标家族再移交一次。

### 3.4 错误码映射

- `StreamCreate` 优先级非法 / 队列不可用 → `InvalidArgument` → `MUSA_ERROR_INVALID_VALUE`；
- `EventQuery` 对有效目标非阻塞。目标尚未达成返回 `NotReached`，达成返回 `Reached`，目标失败时返回对应错误。无有效 Record、无效句柄等返回 `InvalidArgument`。
- 首次被接受的 `Record` 绑定生产者 Stream。后续 `Record` 必须使用该 Stream。计算侧 `Wait` 可跨队列，图形侧通过同设备的共享 semaphore/资源建立依赖。
- `Wait` 在记录时快照最后一个已接受的 Record 值。已满足的历史值可立即通过。正向同队列自等待、无效 semaphore 和递减 SIGNAL 按第 32 章处理。

---

## 4. 内部数据流或状态机

### 4.1 stream/event 时序

```
t0 计算队列提交: LaunchKernel(k1) → Release → Event.Record(e1) 对应 SIGNAL
t1 图形队列: graphics UMD 提交 WAIT(e1) → 获取资源所有权 → draw/读 UAV
t2 计算队列: LaunchKernel(k2，使用独立资源) → Event.Record(e2)
t3 回读:   主机显式同步 Stream → Query(e2)=Reached → 数值比对
```

### 4.2 event 状态

```
Event.state: NotReached → Reached 或失败（对应 Record 值）
Query：只读目标状态，主机主动同步使用 StreamSynchronize 或 fence
```

### 4.3 错误处理

- 目标家族在 acquire 前访问返回 `InvalidArgument`，由 S16 的负向用例验证。
- timeline 回卷 / 值序错误在提交期 `InvalidArgument`（值序单调由第 32 章保证）。

---

## 5. 任务清单

> 任务编号 `T40.1~T40.6`，四段式。

| 子里程碑 | 任务 | 段 | 内容 |
|---|---|---|---|
| M40.1 | T40.1 | ① 契约 | 扩展 39 章最小 Stream 支持不同队列，冻结 Event 目标值、生产者绑定和查询状态 |
| M40.1 | T40.2 | ② 主体 | 沿用提交路径，验证多 Stream 相互独立、各自 FIFO、显式同步和资源销毁顺序 |
| M40.1 | T40.3 | ② 主体 | 实现 Event Record/Wait/Query，独立 semaphore 值域、Wait 快照及失败传播 |
| M40.2 | T40.4 | ② 主体 | 计算 UMD 生产、35.1 图形 UMD 消费，复用第 32 章队列同步与所有权转移 |
| M40.2 | T40.5 | ③ 接入 | Demo C3 `async_pipeline` 跑通双 UMD、双队列，接入 S12/S16 回归 |
| M40.2 | T40.6 | ④ 验收 | A40 汇总 D40.1~D40.5 与 `serial`/`parallel` 硬判据 |

### 子里程碑

| 子里程碑 | 范围 | 承载任务号 |
|---|---|---|
| M40.1 stream/event | Stream/Event 头文件、Stream 提交封装、Record/Wait/Query | T40.1~T40.3 |
| M40.2 异步与验收 | async compute、Demo C3、A40 | T40.4~T40.6 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D40.1 | Stream/Event 头文件独立编译；优先级不可变、Stream 与 HLI 队列 1:1 映射；同一队列重复创建 Stream → `InvalidArgument`，不同家族队列（Graphics/Compute）可并存 | L1 契约 |
| D40.2 | 多 Stream 独立推进、各自 FIFO，沿用第 39 章提交路径；同步返回完成结果，pending 时 `StreamDestroy` 与有 live Stream 时 `ContextDestroy` 返回 `Busy`，显式同步后可逆序销毁 | L1 契约 |
| D40.3 | 包序含正确 `SIGNAL`/`WAIT`；重新 Record 后旧 WAIT 仍保留目标快照；换生产者 Record 被拒；Query 非阻塞区分未达成、达成和失败 | L1 契约 |
| D40.4 | 图形与计算 UMD 经不同队列执行 release→SIGNAL→WAIT→acquire。延迟 release 时图形继续等待，信号前生产者失败时依赖绘制失败，正常移交后读到计算结果 | L1 + L2 |
| D40.5 | 双 UMD 的 C3（WA-11）结果与串行参考逐值一致；S12/S16 回归不退化 | 横切 |
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
| 回归项 | R40：S12（多引擎交错）、S16（屏障 + 所有权转移）在接入后不退化；未列明历史 A 门禁不自动阻断 |

> 场景输入与探针单份规约见 `../design/scene-specs.md` §3（S19 节）与 §4（第 40 行）。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 依赖只由 fence/semaphore 定义；归并无时间输入；两次运行依赖序一致 |
| 时间无关 | 双队列推进不依赖墙钟；`WAIT` 由 timeline 值而非时间触发 |
| 可观测 | `--trace=compute`：提交日志、SIGNAL/WAIT 序列、event 状态机 |
| 契约影响 | 不新增编号/队列/配置键；timeline 值域复用第 32 章 |
| 简化登记 | 每个 stream 仅绑定一个队列；不同家族队列可并存各自 stream；优先级不可变；无抢占 |
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
| 多队列 stream | 不支持单个 stream 跨多个队列；不同队列各自的 stream 可并存（1:1 映射） | `compute-driver.md` §6 |
| 优先级 | 创建时声明、不可变 | 同上 |
| 计算抢占 / TDR | 二期 | ADR-025 二期候选（capability bit） |
| async copy（直通 LDS） | 二期候选 | §7.9 二期候选，未获 ADR 前 `NotSupported` |

---

## 10. 待定项

- 跨 stream 隐式依赖（应用语义层面）一期不做：所有依赖显式经 `Event`/fence 表达（M-07 同源）。
- `Event` 复用池化与价值域基线化属于工程优化，不影响契约。

---

## 2026-11-21 评审同步说明

本章负责多 Stream 与 Event 的驱动编排。队列与异步推进复用第 32 章，BARRIER 编码见 [命令包登记 §2.3](../design/command-mmio.md)。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A40 和适用的 R40 后提交。分配核对项为引用现有同步包、完成值与事件代际，新增或改动按 [编码与位段校验表](../design/allocation-audit.md)登记。

```text
feat(gpu-ch40): schedule asynchronous streams and events

- 实现 stream/event、跨流依赖及回收生命周期。
- 用 C3 对照事件次序、失败传播和资源回收。

Tasks: T40.1–T40.6
Validation: <实际执行的命令、用例和结果>
```
