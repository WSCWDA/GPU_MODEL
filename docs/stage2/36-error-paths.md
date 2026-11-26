# 第 36 章 错误路径

> 本章收口全项目错误处理：10~35 各章保留的"单元 + 包类型 + dword 偏移"定位钩子统一接入 watchdog、复位路径与故障注入。三类故障分层（硬件/驱动/应用，ADR-039）、按引擎 watchdog、`reset 后可重跑一致`（S13）是本章核心专有判据。设计说明见 `../design/boot-reset.md` §2~§4。
> 与本章相关的设计说明：`../architecture-design.md` §3.9（错误契约）；ADR-011/031/039/040；`../design/logging.md`（日志与断言）；`../design/boot-reset.md`；`tools/{log_analyze, mem_audit, state_inspector, spec_runner}`。
> 落地模块：`kmd/reset`、`firmware/watchdog`、`hw_sim` 故障注入钩子、`tools/{log_analyze, mem_audit}`、`apps/scene_s13_fault`。

---

## 1. 目标与非目标

### 1.1 目标

1. 三类故障分层清晰：硬件（`ResetRequired`）/ 驱动（`Result`）/ 应用（UMD 拦截），互不混淆（ADR-039）。
2. 按引擎 watchdog：每引擎独立活跃标记，watchdog 只复位对应引擎，不做全设备一刀切（ADR-039）。
3. 复位只停故障引擎。丢弃未完成包并通知等待者，保留错误记录，只重编程该引擎 ring，再由应用重建队列并显式重提。
4. 故障注入：仅高级硬件单元（ADR-031）；错误包/非法 ISA/死循环/未映射/越界 → 定位到 `{unit, packetType, dwordOffset, va, code}`。
5. reset 后 S2 重跑成功（结果与首次一致）；错误记录可靠、日志可分析（`log_analyze`）。
6. 内存泄漏/UAF 检测：`mem_audit` 解析快照。
7. 错误不回滚 GPU 内存内容承诺；不承诺 wave 级中间状态恢复（范围外）。

### 1.2 非目标

- 普通页错误结束当前提交，修正映射后可显式重提。
- 不做完整容错/ECC。
- 不做中断驱动的恢复握手（中断只通知）。
- 不做应用层错误穿透到硬件（红线：应用逻辑错误不触发设备复位）。

---

## 2. 单元框图

```
 任何引擎单元（3D/Copy/Compute）――故障信号 → ErrorRecord{unit, type, offset, va, code}
    │                                │
    ▼                                ▼
 firmware/watchdog（每引擎活跃标记）  kmd/reset（复位管理器）
 wave 超出指令预算/引擎错误 → watchdog 判定 → 停止该引擎 → 复位该引擎 → 重建上下文
    │                                │测试：中止后继续提交，或复位后显式重提
    ▼
 应用/CI：S13 → log_analyze/mem_audit/spec_runner
```

- Watchdog 按单 wave 的退休指令数判断，预算和正常退出边界见 [启动与复位 §4](../design/boot-reset.md)。

---

## 3. 输入输出契约

### 3.1 错误记录

引用 [第 1 章 §3.2](../stage1/01-contracts-min.md) 定义的公共 `Result` 和 `ErrorRecord`。KMD 保留原错误及提交、引擎与命令位置。公共 `dwordOffset` 沿用包内偏移，定位所需的包起点随提交诊断保存。

### 3.2 复位管理（ADR-052/057）

先停驻故障引擎的 CP，等待已分派的 worker 停止访问该引擎的在飞资源。再丢弃未完成包、登记失败终态与未执行的信号、释放对应硬件引用，然后重编程该 ring。

| 操作 | 可观察行为 |
|---|---|
| 等待已接受的旧提交 | 按队列 generation 与提交序号得到原终态。预算耗尽的提交保留 `Timeout`，因复位被丢弃的其余提交返回 `DeviceLost`。等待开始于复位前、查询开始于复位后都得到同一结果。 |
| 复位期间提交 / 复位后使用旧队列或 Stream 提交 | 分别返回 `Busy` / `DeviceLost`。新队列使用新的逻辑身份。 |
| 使用已销毁对象 | 返回 `InvalidArgument`。未销毁对象的用户引用仍遵循原生命周期。 |
| 引用释放 | 故障引擎停稳后逐条释放丢弃提交的引用。其他引擎与有效引用继续工作。两类引用归零才回收资源。 |

旧记录保留至设备提交查询生命周期结束，后来的成功 fence 保持其独立终态。受影响的产物按架构 §3.10 处理，应用初始化所需资源后显式重提。

### 3.3 故障注入菜单（测试入口）

```cpp
enum FaultInject { kBadPacket, kBadIsa, kOomLoop, kUnmappedVa, kDeadlockGlitch,
                   kRingOverflow, kBadVersion, kDescriptorOOB };
Result InjectFault1(FaultInject, const FaultSite&);  // 指定单元/包
```

- 注入只影响注入点（高级单元）；初级不经注入。

---

## 4. 内部数据流

### 4.1 故障分流（S13）

| 故障 | 结果与后续行为 |
|---|---|
| 未映射、未驻留 | 分别返回 `NotMapped` / `NotResident`，记录 VA，当前提交失败，同一队列继续接收合法提交。 |
| 可定位的非法包、版本、ISA 或描述符越界 | 沿入口契约拒绝或中止当前提交，并记录对应错误。 |
| 分配耗尽、ring 容量不足 | 分别返回 `OutOfMemory`、`Busy`，保留其他有效提交。 |
| 死循环超时、显式注入引擎级故障 | 根因分别记录 `Timeout` 或原错误，按 §3.2 复位，应用显式重提。 |

### 4.2 Watchdog

每个 wave 独立累计退休指令。最后一条指令耗尽预算并正常退出时先判定成功。预算耗尽且仍运行时返回 `Timeout`，保留 wave、PC 和 dispatch，触发对应引擎复位。workgroup 进展和墙钟用于观测。

### 4.3 日志与审计

- 错误日志：`[level][unit][submodule] ... packetType/dwordOffset/va/code` 单行。
- `log_analyze`：错误计数、去重、Top 单元报告。
- `mem_audit`：快照解析泄漏/UAF。
- `spec_runner` 运行 S13 故障分流、复位终态及 watchdog 边界回归。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T36.1 | ① 契约 | 引用第 1 章 Result 与 ErrorRecord，定义复位管理、故障注入和完成映射 |
| T36.2 | ② 主体 | 错误记录构造与 `Result` 规范化（全单元统一定位钩子） |
| T36.3 | ② 主体 | 停驻并等待故障引擎 worker，按旧提交登记失败、释放引用并重编程 ring |
| T36.4 | ② 主体 | watchdog：每引擎活跃标记 + 超时决定只复位对应引擎 |
| T36.5 | ② 主体 | 故障注入菜单（BadPacket/BadIsa/OomLoop/UnmappedVa/Deadlock/RingOverflow/BadVersion/DescriptorOOB） |
| T36.6 | ② 主体 | 错误记录保留表 + LastError + 日志定位（unit/type/offset/va/code） |
| T36.7 | ② 主体 | 复位后上下文重建 + 重跑一致性（固定输入） |
| T36.8 | ② 主体 | `log_analyze`/`mem_audit` 工具接入（错误统计、泄漏/UAF） |
| T36.9 | ③ 接入 | 接入 S13 的提交中止、引擎复位重提和 watchdog 边界 |
| T36.10 | ④ 验收 | A36：S13 全绿 + 专有判据 + 已交付硬件范围内的 E1~E5/S0~S17 回归（E0 单独 L1 自检；S18/S19 归 37/42） |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D36.1 | 接口独立编译并引用第 1 章 16 值快照。逐项核对调用错误、ErrorRecord 与提交终态，旧枚举独立冻结判据为 **SUPERSEDED** | L1 契约 |
| D36.2 | 每类故障定位到 `{unit, packetType, dwordOffset, va, code}`（注入→断言字段） | L1 契约 |
| D36.3 | 复位前等待与复位后查询均得到同一旧提交终态，未执行信号的失败可达依赖方。停稳后释放丢弃提交引用一次，其他引擎及有效引用持续可用 | L1 契约 |
| D36.4 | 单 wave 在预算内、恰好耗尽并退出均成功，耗尽且仍运行返回 `Timeout`，保留 wave/PC/dispatch。多 wave 合法长派发按单 wave 计数通过，仅超时引擎复位 | L1 契约 |
| D36.5 | 8 类注入菜单各命中（BadPacket/BadIsa/OomLoop/UnmappedVa/Deadlock/BadVersion/DescriptorOOB/RingOverflow） | L1 契约 |
| D36.6 | 错误保留表在复位后仍可读（LastError）；日志含 unit/type/offset/va/code | L1 契约 |
| D36.7 | 复位期间提交返回 `Busy`，旧 queue/stream 重新提交返回 `DeviceLost`，已销毁资源句柄返回 `InvalidArgument`。重建并显式重提 S2 后结果一致 | 横切零容差 |
| D36.8 | `log_analyze` 报告正确错误计数；`mem_audit` 标识泄漏/UAF | L1 契约 |
| D36.9 | S13 普通页错误后同一队列继续提交，复位子用例验证旧终态、其他引擎持续运行及显式重提结果 | L2 + A36 |
| D36.10 | A36 通过故障分流、复位终态、重提、watchdog 边界及已交付场景回归，E0 单独 L1 自检 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A36）

**绑定场景：S13 `scene_s13_fault`（故障矩阵）。**

| 项 | 内容 |
|---|---|
| 输入 | S13 故障分流、旧提交终态和 watchdog 边界，执行策略按场景规约 |
| 产物 | 错误记录（unit/type/offset/va/code）、watchdog 日志、reset 后 S2 结果 |
| 主判据 | 普通页错误保持队列可用，复位仅影响故障引擎，旧终态可查询，显式重提结果一致 |
| 本章专有判据 | 故障位置、错误映射、提交终态、引用生命周期及单 wave 预算边界可独立断言 |
| 回归项 | R36：已交付 E1–E5、S0–S17 的适用判据不因错误处理退化；E0 单独 L1 自检；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 项目最终 E/S 全场景回归与双入口计算验证由第 42 章 A-stage3 收口 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 错误记录构造序、复位动作序、重跑一致序固定 |
| 时间无关 | watchdog 使用每 wave 退休指令预算，不依赖墙钟/tick |
| 可观测 | ErrorRecord、watchdog 指令预算、`log_analyze` 报告、`--trace=error` |
| 契约影响 | `ErrorRecord`/`ResetManager`/复位语义/特权边界冻结 |
| 简化登记 | 初级"错误即中止" → 定位 + 按引擎复位 + watchdog |
| 计算兼容 | 错误钩子覆盖计算引擎（37/stage3 38~42，原 35.5）同样可注入 |
| 配置与回退 | `fault_inject`（测试）、`kMaxInstructionsPerWave=2^24` 固定预算 |
| 调试钩子 | `--trace=error`（每次错误一行）；`--watchdog-log` |
| trace 契约 | 不进入判据 |
| 契约变更 | 复位/故障分层/特权边界/watchdog 为冻结项；变更走 ADR 回归 S13/S1 |
| 校验归属 | L1（结构/定位字段）+ L2（S13 矩阵）+ 横切（reset 重跑一致） |
| 双策略 | 复位与重跑在 `serial/parallel` 下一致（并行下故障仍可定位） |
| 阶段衔接 | 本章验证已交付硬件范围；全部 E/S 项目回归由第 42 章在 S18/S19 和 stage3 完成后收口 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 错误 | 初级"中止即停" → 定位 + 复位 + watchdog | —（定案） |
| 复位 | 整设备一刀切 → 按引擎 watchdog/复位 | — |
| 故障注入 | 初级无 → 高级单元注入菜单 | — |
| 恢复 | 无需 auto-recovery → reset 后重跑一致 | — |

---

## 10. 待定项

- `log_analyze`/`mem_audit`/`state_inspector`/`spec_runner` 的产物格式（JSON/HTML 汇总）在 36 章实现时与 `validation-model.md` §7 对齐；格式细节归档于 `tools` 文档（属工具层，不进硬件契约）。
- `kMaxInstructionsPerWave` 的默认值是否需要按 shader 复杂度配置——当前固定为 2^24；以后变更需登记 ADR。
- 复位语义固定：故障引擎的在飞提交丢弃并标记失败；复位期间新提交返回 `Busy`；复位完成后旧 queue/stream generation 句柄返回 `DeviceLost`。不隐式重放。应用重建受影响队列/stream/资源后显式重提。

---

## 2026-11-21 评审同步说明

watchdog 的每 wave 指令预算见 启动与复位。公共 Result 集合见 第 1 章 §3.2。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A36 和适用的 R36 后提交。分配核对项包括错误枚举、MMIO 权限与完成状态。变更按已有分配流程登记。

```text
feat(gpu-ch36): report deterministic submission faults

- 统一错误、调试上下文、失败 fence 与最小复位。
- 用 S13 故障矩阵核对错误来源和引擎恢复。

Tasks: T36.1–T36.10
Validation: <实际执行的命令、用例和结果>
```
