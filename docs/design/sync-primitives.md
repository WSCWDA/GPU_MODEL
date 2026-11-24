# 同步原语 — 独立模块设计

> 项目目标、范围、阶段与交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

> 定位：定义跨单元 / 跨队列 / 跨引擎的正确性核心：fence、timeline semaphore、event、barrier 与内存序/作用域映射。同步语义与内存模型（`../docs/design/memory-system.md`）共同保证"时间无关 + 确定序"。
> 落地章：第 2 章 Fence 最小载体、第 32 章队列 + 同步、第 33 章命令链 + 多引擎、第 20 章资源屏障；内存序/作用域落在第 30、31 章工作负载侧。
> 权威来源：从 `architecture-design.md` §4.3 拆出，并吸收 `memory-system.md` §6（主机-GPU 缓存同步）、`roadmap.md` §0.5（能力边界，屏障粗粒度掩码）与 §2.8（屏障与所有权转移、内存序映射）、ADR-024 / ADR-031。

---

## 1. 原语总表

| 原语 | 语义 | 载体 |
|---|---|---|
| Fence | 单调递增 64 bit 时间线；提交完成写值 | MMIO fence 监视 + `FENCE` 包 |
| Timeline semaphore | 跨队列/跨引擎等待与信号 | `SIGNAL`/`WAIT` 包和内存中的信号量状态 |
| 计算 UMD Event | 可重复记录的跨队列标记 | Event 自有 timeline semaphore 与最近一次已接受的记录值 |
| Barrier | 执行/内存/图像布局约束 | `BARRIER` 包 |

队列家族固定为 Graphics / Compute / Copy；计算队列支持异步计算，与图形队列只通过 fence/semaphore 排序。

**并发口径（P1-7 澄清）**："计算队列与图形队列并发" = FW 在**包粒度 Round-Robin 交错推进**；硬判据对交错顺序**不敏感**（种子化交错顺序下结果不变），**同刻并行不在本期**——S19 与 stage3 40 章判据依此执行（`roadmap.md` §5.4、`scene-specs.md` S19）。

## 2. 规则

1. 初级 `Wait` 是 fence 值断言：提交同步完成，等待退化为检查。
2. 高级 `SIGNAL` 发布成功信号。`WAIT` 只暂停本队列在该命令后的执行，FW 继续推进其他就绪队列。等待和唤醒依据模型状态。
3. 屏障一期只支持 `READ / WRITE / ATTACHMENT / TRANSFER` 粗粒度集合（ADR-031，`roadmap.md` §0.5 能力边界）：第 20 章资源屏障、第 32 章队列屏障共用同一编码；不做 per-resource 位图。
4. 队列家族所有权转移独立于普通内存屏障；转移必须显式释放/获取（ADR-024）。
5. 同一队列内命令按记录顺序执行；跨队列只由 fence/semaphore 定义顺序。

### 2.1 提交完成与失败传播

| 状态或操作 | 一期行为 |
|---|---|
| 提交接受 | KMD 登记队列 generation 和提交序号，获得一个可查询的完成记录。提交持有的资源引用保留到终态，已拒绝的提交不生成完成记录 |
| 完成 | 执行成功更新完成状态。解析、执行失败或复位丢弃更新为失败，并保留原始错误和提交位置。主机等待读取终态，失败提交不会因为后续 fence 值变大而变成成功 |
| SIGNAL | 成功值单调。小于当前成功值的信号返回 `InvalidArgument`，原值保持，等值幂等。被失败提交跳过的信号保留失败记录，尚在等待该已登记信号的消费者得到失败完成 |
| WAIT | 已满足则继续。未满足时挂起本队列，其他队列仍可推进。观察到对应生产者失败时中止依赖提交，记录失败原因，阻止后续访问资源 |
| 关闭期间的无进展 | 停止接收后，所有未结束队列均无法推进且没有待处理的内部事件时，判定剩余等待无法满足，使用现有 `InvalidArgument` 报告等待失败，记录队列与信号目标。正常运行中允许生产者后续提交 |

队列层以提交记录跟踪 Pending/Success/Failed。`SIGNAL` 在提交接受时与命令同时登记。原始 WAIT 已被当前值满足时立即通过，否则绑定首个能覆盖目标的已登记信号。没有匹配时停驻，允许后续提交登记生产者；已绑定的等待按该生产者终态完成，后续成功值不能覆盖其失败。Event WAIT 固定使用其 Record 快照对应的信号。第 32 章用受控提交和故障替身验证，第 36 章接入复位。关闭时先处理可运行队列和在飞工作，所有队列均停驻且内部事件已清空才终结无法满足的等待。

资源消费方仍需完成对应的 release/acquire、所有权移交和缓存可见性操作。成功的 timeline 值证明依赖顺序，资源屏障证明写入可被读取。

## 3. 内存序与作用域映射（第 30、31 章）

统一映射 `Relaxed / Acquire / Release / AcqRel / SequentiallyConsistent`，作用域区分 `Workgroup / Device`：

- **workgroup 作用域**：shared 栅栏（`LDS/STS` + 组内 barrier），跨 wave barrier 同步——组内全部 wave 到达 barrier 才放行，不是单波内同步；
- **device 作用域**：全局资源屏障（`BARRIER` 包）+ Fence/Semaphore，跨组 / 跨引擎可见性；
- shared 栅栏与全局资源屏障是**两套机制**，不得混用；屏障不依赖线程到达顺序。

计算线的内存序与 barrier 是正确性核心（`../docs/design/compute-line.md` §4.3）；共用单元不得按 workload 分叉屏障语义。

## 4. 与确定性 / 时间无关的关系

- 硬件路径只依赖两类逻辑时间源：`frame_index` 与 timeline-semaphore / fence 逻辑时间戳；`std::steady_clock` / 系统时钟只用于性能耗时统计（`docs/design/execution-parallel.md` §2）。
- 跨队列依赖顺序由 fence/semaphore 定义，任何并行度下硬判据一致（双策略）。
- 竞争缺陷由策略 B（`../docs/design/execution-parallel.md`）暴露，但正确性判据永远落在同步语义上，不靠线程调度。

## 5. 契约变更

- 屏障编码、semaphore 语义、内存序取值均为冻结契约，变更走 `roadmap.md` §7.4（新建 ADR）并回归 S12/S16/S19 等同步相关场景。
- `SIGNAL` / `WAIT` / `BARRIER` 包类型见 [命令包登记 §2](command-mmio.md)。
