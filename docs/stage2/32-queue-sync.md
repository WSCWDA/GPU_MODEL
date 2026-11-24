# 第 32 章 队列 + 同步

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章多队列、多环落地：Graphics/Compute/Copy 三队列家族（每家族独立 ring 实例），`SIGNAL`/`WAIT` timeline semaphore、事件完成标记、跨队列家族所有权转移、以及异步计算队列。队列内命令按记录顺序、跨队列只由 fence/semaphore 定义顺序（ADR-024）。设计说明见 `../design/sync-primitives.md` §1~§2。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（IEngine）、§3.5（KMD Submit 扩展 queueId）；ADR-024（屏障/所有权转移）；`../design/command-mmio.md`（SIGNAL/WAIT/PREDICATE/BARRIER 包登记）；第 10 章单环扩展为多环。
> 落地模块：`hw_sim/core/multiqueue`、`kmd/submission`（多队列）、`umd/graphics/pal/queue`、`apps/scene_s12_multiengine`、`apps/scene_s16_barrier_ownership`。

---

## 1. 目标与非目标

### 1.1 目标

1. 三队列家族：Graphics/Compute/Copy，每家族独立 ring、独立 fence timeline。
2. `SIGNAL`（timeline semaphore）写值；`WAIT` 阻塞后续包直到满足；等待不得依赖中断到达顺序。
3. fence：每队列单调 timeline；`WaitFence`/`WaitSemaphore`（值断言与真阻塞两用）。
4. 跨队列家族所有权转移：释放（release）/获取（acquire）独立于普通内存屏障；转移必须显式。
5. 异步计算队列：计算队列入队不阻塞图形；结果由 fence 提供给图形后合并。
6. 多引擎交错依赖正确：图形 → 计算 → 图形通过 semaphore 排序，最终 RT 正确。
7. 队列内记录顺序执行；命令包（`SIGNAL/WAIT/PREDICATE/BARRIER`）语义确定。

### 1.2 非目标

- 不做优先级/抢占调度（14 章预占、36 复位配合；本期 FIFO）。
- 不做 IB 链/间接命令（33 章）。
- 不做多 command-list 交织（35.1 后）。
- 不做全功能 semaphore 二元/二元组全套（timeline 值语义）。

---

## 2. 单元框图

```
                    KMD Submission
  ┌──────────────┬───────────────┬───────────────┐
  ▼              ▼               ▼               ▼
Gfx Queue       Compute Queue   Copy Queue     (每队列各自 ring+tail+fence)
  │              │                │
  └──── SIGNAL/WAIT 在命令流间互连（timeline semaphore）
  └──── BARRIER（执行/内存/布局，粗粒度）
  │
 所有权转移：release(QueueB 可写) → acquire(QueueC 读)
```

- CP 多 Ring 仲裁：每队列实例独立取指；引擎空闲接收排空。
- `SIGNAL`/`WAIT` 包在目标队列 timeline 上读写。

---

## 3. 输入输出契约

### 3.1 队列对象（`pal/queue/queue.h`）

```cpp
struct QueueDesc { u32 familyId; uint8_t priority; }; // Graphics=0, Compute=1, Copy=2
class Queue {
public:
    Result Submit(const CommandList&);   // 写本 ring + doorbell
    Result Signal(u64 value);            // 命令流内 SIGNAL
    Result WaitSemaphore(u64 value);     // 命令流内 WAIT（阻塞至值）
    Result WaitHost(u64 value);          // 主机等待（值断言）
};
```

- 每队列一个 fence timeline：`currentValue/maxValue`；`SIGNAL` 推进、`WAIT` 挂起后续。

### 3.2 包（`command-mmio.md` §2 登记扩展）

| 包 | 载荷 | 语义 |
|---|---|---|
| `SIGNAL`(0x0A) | `{queueId, semaphoreVa/Id, value}` | 写 timeline 值；完成信号 |
| `WAIT`(0x0B) | `{queueId, semaphoreVa/Id, value}` | 阻塞至 `sem >= value` |
| `PREDICATE`(0x0D) | `{predVa:u64, mask:u64, cond:u64, skipDwords:u32, reserved0:u32}` | 条件执行；载荷与跳过范围唯一口径见 `command-mmio.md` §2.4 |
| `BARRIER`(0x0E) | 3-dword envelope `{kind,scope,memOrder,accessMask; srcStageMask; dstStageMask}`；ownership kind=2 | 粗粒度 |

- `SIGNAL` 为 `0x0A`、`WAIT` 为 `0x0B`，`BARRIER` 为 `0x0E`。包号和载荷以 [命令包登记](../design/command-mmio.md) 为准。
- `WAIT` 只挂起本队列后续包。等待以 [同步原语 §2.1](../design/sync-primitives.md#21-提交完成与失败传播) 定义的成功值或失败完成唤醒。

> 一期队列沿用第 10 章直连 ring 寄存器模型，多队列依赖通过内存 semaphore 表达。间接环描述表和 FW 活动窗口切换作为后续扩展，见 [评审裁决 §2.2](../design/review-sync-20261121.md)。
>
> 本章沿用 `stateId` 0~7 的直接槽与 `0x50~0x7F` 保留段，semaphore 通过设备内存 VA 引用。

### 3.3 所有权转移

```
释放（release）：QueueA.Signal(value); BARRIER(release accessMask)
获取（acquire）：QueueB.WaitSemaphore(value); BARRIER(acquire accessMask)
```

- 转移独立于普通内存屏障（ADR-024）。

---

## 4. 内部数据流

### 4.1 多环调度

```
提交到 Queue F：
  KMD 写 ringF + doorbell → CP_fid 取指执行
  引擎单元 3D/Copy/Compute 由包类型派发（10 章）；不同 ring 可交错推进
Waiter 在 ringF 上：SIGNAL/WAIT/BARRIER 作为命令流元素
```

- 队列间顺序唯一由 semaphore；无墙钟。

### 4.2 semaphore 计时线

```
Queue.Signal(v): 若 v < 当前成功值则报 InvalidArgument；否则发布 v
Queue.WaitSemaphore(v): 达到目标则继续；生产者失败则结束依赖提交；否则仅挂起本队列
主机 WaitHost(v): 查询已指定提交的成功或失败状态
```

- timeline 单调递增（release 语义）。

### 4.3 异步计算

```
计算 Queue C 提交 compute 命令：
  C.Signal(v1) ；图形 Queue G 命令流含 WaitSemaphore(v1)
  G 在 WAIT 处挂起，G 的后续包按原序等待，其他队列仍可推进
场景 S12：多引擎交错依赖正确。
```

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T32.1 | ① 契约 | `QueueDesc`/`Queue`/SIGNAL/WAIT/PREDICATE/BARRIER 载荷/所有权转移语义落头文件 |
| T32.2 | ② 主体 | 多队列实例 + 多 ring 初始化（Graphics/Compute/Copy 独立 ring） |
| T32.3 | ② 主体 | timeline semaphore：SIGNAL 写值、WAIT 阻塞、值单调 |
| T32.4 | ② 主体 | 提交接受、成功/失败完成和主机等待。已知生产信号失败时结束依赖提交 |
| T32.5 | ② 主体 | BARRIER 在队列内（执行/内存/布局）粗粒度语义 |
| T32.6 | ② 主体 | 所有权转移（release/acquire 显式） |
| T32.7 | ② 主体 | 用 Compute 家族队列的受控生产者与图形/Copy 验证跨队列推进和失败传播。实际 kernel 接入归 T37.10 |
| T32.8 | ③ 接入 | `scene_s12_multiengine`（三队列交错依赖）、`scene_s16_barrier_ownership`（所有权转移 + 跨队列 barrier）接入 |
| T32.9 | ④ 验收 | A32：S12/S16 全绿 + 专有判据 + R32 回归项 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D32.1 | 接口独立编译；`QueueDesc.familyId` 枚举冻结 | L1 契约 |
| D32.2 | 三队列独立：输出互不串扰；`Signal` 只推进本 timeline | L1 契约 |
| D32.3 | WAIT 观察对应成功或失败终态；对同一 semaphore 测试递增、等值幂等、递减 `SIGNAL` 返回 `InvalidArgument` 且成功值保持 | L1 契约 |
| D32.4 | 失败提交唤醒主机等待并传出错误，后续 fence 成功值不掩盖失败。释放在飞引用 | L1 契约 |
| D32.5 | BARRIER：release 后 acquire 读可见；`READ→WRITE` 不越级 | L1 契约 |
| D32.6 | 所有权：QueueB 在 acquire 前不读 QueueA 写的数据；acquire 后读到 | L1 契约 |
| D32.7 | WAIT 停驻当前队列；晚到生产者可唤醒；已绑定失败信号不能被后续成功值覆盖；独立队列持续推进。关闭时无法满足的等待进入失败终态 | L1 契约 |
| D32.8 | S12/S16 探针通过 | L2 + A32 |
| D32.9 | A32 全绿：三引擎交错依赖正确、timeline semaphore 值序正确、跨队列家族所有权转移、计算异步队列不破坏依赖 + R32 回归项 | L1 + L2 |

---

## 7. 场景与判据（A32）

**绑定场景：S12 `scene_s12_multiengine`（多队列依赖）、S16 `scene_s16_barrier_ownership`（所有权转移/跨队列 barrier）。均为既有 S 入口的子 case，不另立入口（Q-67）。**

| 项 | 内容 |
|---|---|
| 输入 | S12：图形 → 计算 → 图形 semaphore 链；S16：Copy 产中间缓冲 → 图形读（release/acquire 显式 + 跨队列 barrier） |
| 产物 | 各队列 timeline 序列、RT 输出、所有权转移日志 |
| 主判据 | S12/S16 输出 = ref_model（依赖顺序零容差） |
| 本章专有判据 | ① 三引擎交错依赖正确；② timeline semaphore 值序正确；③ 跨队列家族所有权转移；④ 计算异步队列不破坏依赖 |
| 回归项 | R32：单队列基线（S1）不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | IB 链/间接（33）；事件完成标记（34 查询） |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 队列内顺序=记录序；跨队列只由 semaphore 值定义；无墙钟 |
| 时间无关 | WaitHost 值断言（busy-等待逻辑），无超时语义进入判断 |
| 可观测 | 每队列 timeline、SIGNAL/WAIT 次数、`--trace=queue` |
| 契约影响 | Queue/SIGNAL/WAIT/BARRIER/所有权转移冻结 |
| 简化登记 | 单队列同步 → 三队列 + timeline semaphore + 所有权转移 |
| 计算兼容 | 计算队列与图形队列语义一致（异步是排序不是特化） |
| 配置与回退 | `queue.familyCount`、`queue.priority` 创建期声明 |
| 调试钩子 | `--trace=queue`：`{queueId, signal/wait, value}` |
| trace 契约 | 不进入判据 |
| 契约变更 | 队列/semaphore/所有权语义为冻结项；变更走 ADR 回归 S12/S16 |
| 校验归属 | L1（结构/timeline 单调）+ L2（S12 交错结果） |
| 双策略 | 队列依赖两策略一致（semaphore 定义序）；多线程只执行单个提交 |
| 阶段衔接 | 为 33（IB 链/多引擎）、stage3 38~42（stream/event 对队列，原 35.5）提供队列基座 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 队列 | 单队列 → Graphics/Compute/Copy 三队列 | — |
| 同步 | fence 单值 → timeline semaphore + 所有权转移 | — |
| 异步 | 无 → 计算异步队列 | — |
| 屏障 | 队列内 BARRIER → 粗粒度 | 33（间接/predicate 复用） |

---

## 10. 待定项

- 无进展和失败传播以 [同步原语 §2.1](../design/sync-primitives.md#21-提交完成与失败传播) 为准。局部验收用确定性故障替身，A36 再覆盖真实引擎失败。
- timeline semaphore 的初始/销毁语义（值从 0 开始、不可逆）是否需 `Reset` 支持，留 stage3 38~42（计算驱动 event，原 35.5）收口。
- 所有权转移的"自动"（无显式 release/acquire）是否默认存在：本期要求显式；若 S16 显示弱序用例难写，再评估默认所有权归当前提交者。

---

## 2026-11-21 评审同步说明

本章 BARRIER 使用 `review-sync-20261121.md` §2.3 的唯一 3-dword信封；一期采用第 10 章直连寄存器 ring 模型，多队列内存描述表/活动窗口切换为后续 capability（§2.2）。
