# 第 36 章 错误路径

> 本章收口全项目错误处理：10~35 各章保留的"单元 + 包类型 + dword 偏移"定位钩子统一接入 watchdog、复位路径与故障注入。三类故障分层（硬件/驱动/应用，ADR-039）、按引擎 watchdog、`reset 后可重跑一致`（S13）是本章核心专有判据。设计说明见 `../design/boot-reset.md` §2~§4。
> 与本章相关的设计说明：`../architecture-design.md` §3.9（错误契约）；ADR-011/031/039/040；`../design/logging.md`（日志与断言）；`../design/boot-reset.md`；`tools/{log_analyze, mem_audit, state_inspector, spec_runner}`。
> 落地模块：`kmd/reset`、`firmware/watchdog`、`hw_sim` 故障注入钩子、`tools/{log_analyze, mem_audit}`、`apps/scene_s13_fault`。

---

## 1. 目标与非目标

### 1.1 目标

1. 三类故障分层清晰：硬件（`ResetRequired`）/ 驱动（`Result`）/ 应用（UMD 拦截），互不混淆（ADR-039）。
2. 按引擎 watchdog：每引擎独立活跃标记，watchdog 只复位对应引擎，不做全设备一刀切（ADR-039）。
3. 复位路径：检测 → 停止 CP → 丢弃未完成包 → 清空队列/fence/保留错误记录 → 重新 boot → 重跑一致（`boot-reset.md` §2）。
4. 故障注入：仅高级硬件单元（ADR-031）；错误包/非法 ISA/死循环/未映射/越界 → 定位到 `{unit, packetType, dwordOffset, va, code}`。
5. reset 后 S2 重跑成功（结果与首次一致）；错误记录可靠、日志可分析（`log_analyze`）。
6. 内存泄漏/UAF 检测：`mem_audit` 解析快照。
7. 错误不回滚 GPU 内存内容承诺；不承诺 wave 级中间状态恢复（范围外）。

### 1.2 非目标

- 不做细粒度页错误恢复/wave 暂停-恢复（呈现复位即可）。
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
 引擎超时/无进展 → watchdog tick      → 停止该引擎 → 复位该引擎 → 重建上下文
    │                                │测试：注入故障 → 定位 → reset → 重跑
    ▼
 应用/CI：S13 → log_analyze/mem_audit/spec_runner
```

- watchdog 按引擎：`active[engineId]` 每 tick 更新；无进展超限才触发该引擎复位。

---

## 3. 输入输出契约

### 3.1 错误记录（`common/result.h` 扩展）

```cpp
struct ErrorRecord {
    u32  unit;           // 枚举：cp/ia/frontend/raster/om/texture/shader/copy/display/...
    u32  packetType;
    u32  dwordOffset;    // 包内偏移（含头）
    u64  va;             // 出错资源 VA（0 表示不适用）
    Result code;         // NotMapped/InvalidPacket/InvalidIsa/ResetRequired...
};
```

- 错误记录进入 `kmd` 保留表；`ResetRequired` 是硬件复位唯一入口（ADR-031）。

### 3.2 复位管理

```cpp
class ResetManager {   // kmd/reset
public:
    Result ArmWatchdog(u32 engineId, u32 timeoutTicks);
    Result OnEngineProgress(u32 engineId);   // watchdog 喂狗
    Result ResetEngine(EngineType);          // 单引擎复位：停 CP→丢包→清 fence→reboot ring
    const ErrorRecord& LastError() const;
};
```

- 复位动作：该引擎 ring 停止、内部 FIFO 清空、fence 监视清零、重新 boot（`firmware/watchdog` 配合）。

**引擎复位时序伪代码（`ADR-041` 双引用计数 × 复位语义，3 行演示引用计数 / fence / 句柄有效性）：**

```
① 提交 A/B/C（ring）→ 均未完成，A/B/C 各自 hwRef++ 且入 ring；
② ResetEngine(Graphics) → 该引擎丢弃未完成包、fence 监视清零 → A/B/C 的硬件提交引用立即归零（hwRef=0），in-flight fence 不再可等；
    应用此时 Destroy 缓冲 B（userRef--，userRef=0 且 hwRef 已归零）→ 立即 FreeGpuMem，无悬垂；对象句柄（ID）对外标记 invalid/reuse 禁；
③ 重新提交 D（新命令）→ D 的 hwRef 从 0 重新建立（=1）、新 fence timeline 单调，旧句柄（B）再提交/绑定 → 返回 InvalidArgument（句柄失效，不静默复用）。
```

- 语义要点：复位只丢**该引擎**在飞引用；其它引擎引用不受影响；句柄有效性以"复位即失效"为界，禁止复位后无需重建就复用旧资源句柄。新增 **L1 契约单测（正常+非法边界）**：①复位瞬间在飞提交的 hwRef 立刻归零且 fence 不可等；②`Destroy` 后资源内存被真正回收（`mem_audit` 无 UAF）；③复位后复用旧句柄 → `InvalidArgument`。

### 3.3 故障注入菜单（测试入口）

```cpp
enum FaultInject { kBadPacket, kBadIsa, kOomLoop, kUnmappedVa, kDeadlockGlitch,
                   kRingOverflow, kBadVersion, kDescriptorOOB };
Result InjectFault1(FaultInject, const FaultSite&);  // 指定单元/包
```

- 注入只影响注入点（高级单元）；初级不经注入。

---

## 4. 内部数据流

### 4.1 错误流（正常→复位→重跑）

```
引擎检测故障 → 构造 ErrorRecord → 置 ResetRequired → 停止提交
→ KMD 记录错误 → 复位该引擎（丢包/清 fence/重 boot）
→ 应用可重跑同一固定输入 → 与首次一致（若输入确定无 fault: 重跑成功）
S13 断言：注入每类故障 → 错误记录字段精确 → reset 完成 → 重跑 S2 成功
```

### 4.2 watchdog（按引擎）

```
每 tick：各引擎 OnEngineProgress（若有包在处理）
超时（progress 未推进超过 timeoutTicks）→ 该引擎标记 stuck → ResetEngine(engine)
（不复位其它健全引擎——不整设备一刀切）
```

- 死循环着色器（`DeadlockGlitch` 注入）触发 watchdog → 只复位 shader/该引擎（见 fault 菜单）。

### 4.3 日志与审计

- 错误日志：`[level][unit][submodule] ... packetType/dwordOffset/va/code` 单行。
- `log_analyze`：错误计数、去重、Top 单元报告。
- `mem_audit`：快照解析泄漏/UAF。
- `spec_runner`：跑 S13 回归矩阵（每个故障注入子 case + reset 重跑）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T36.1 | ① 契约 | `ErrorRecord`/`ResetManager`/`FaultInject` 枚举/错误记录保留表落头文件 |
| T36.2 | ② 主体 | 错误记录构造与 `Result` 规范化（全单元统一定位钩子） |
| T36.3 | ② 主体 | `ResetRequired` → 单引擎复位路径（停 CP/丢包/清 fence/reboot ring） |
| T36.4 | ② 主体 | watchdog：每引擎活跃标记 + 超时决定只复位对应引擎 |
| T36.5 | ② 主体 | 故障注入菜单（BadPacket/BadIsa/OomLoop/UnmappedVa/Deadlock/RingOverflow/BadVersion/DescriptorOOB） |
| T36.6 | ② 主体 | 错误记录保留表 + LastError + 日志定位（unit/type/offset/va/code） |
| T36.7 | ② 主体 | 复位后上下文重建 + 重跑一致性（固定输入） |
| T36.8 | ② 主体 | `log_analyze`/`mem_audit` 工具接入（错误统计、泄漏/UAF） |
| T36.9 | ③ 接入 | `scene_s13_fault` 全故障矩阵 + reset 后重跑 S2 接入 |
| T36.10 | ④ 验收 | A36：S13 全绿 + 专有判据 + 全部 E/S 回归 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D36.1 | 接口独立编译；`ErrorRecord` 字段固定；`Result` 枚举扩展冻结 | L1 契约 |
| D36.2 | 每类故障定位到 `{unit, packetType, dwordOffset, va, code}`（注入→断言字段） | L1 契约 |
| D36.3 | `ResetRequired` → 单引擎复位完成（FIFO 空/fence 清零/ring 重 boot），其余引擎数据完好 | L1 契约 |
| D36.4 | watchdog：死循环只在对应引擎触发复位；其它引擎提交正常 | L1 契约 |
| D36.5 | 8 类注入菜单各命中（BadPacket/BadIsa/OomLoop/UnmappedVa/Deadlock/BadVersion/DescriptorOOB/RingOverflow） | L1 契约 |
| D36.6 | 错误保留表在复位后仍可读（LastError）；日志含 unit/type/offset/va/code | L1 契约 |
| D36.7 | reset 后重跑 S2：输出与首次一致（固定输入） | 横切零容差 |
| D36.8 | `log_analyze` 报告正确错误计数；`mem_audit` 标识泄漏/UAF | L1 契约 |
| D36.9 | `scene_s13_fault` 全故障矩阵探针通过（8 类 × 目标单元；reset 后 S2 一致） | L2 + A36 |
| D36.10 | A36 全绿：故障定位到单元、reset 后 S2 重跑成功（watchdog 按引擎监控、不做全设备一刀切）+ 全部 E/S 回归 | L1 + L2 + 横切 |

---

## 7. 场景与判据（A36）

**绑定场景：S13 `scene_s13_fault`（故障矩阵）。**

| 项 | 内容 |
|---|---|
| 输入 | 8 类故障注入 × 若干目标单元；每类之后 reset + 重跑 S2；双重配置（serial/parallel） |
| 产物 | 错误记录（unit/type/offset/va/code）、watchdog 日志、reset 后 S2 结果 |
| 主判据 | 每类故障：定位到单元与位置 + reset 后 S2 重跑成功；watchdog 只复位对应引擎 |
| 本章专有判据 | ① 故障定位到单元；② reset 后 S2 重跑成功；③ watchdog 按引擎监控、不做全设备一刀切 |
| 回归项 | 全部 E/S 场景（含双策略）无因错误处理回退；A1~A35 全绿 |
| 后续补齐 | 项目收口：E/S 全场景在双策略、多后端、故障注入下不退化（全回归矩阵） |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 错误记录构造序、复位动作序、重跑一致序固定 |
| 时间无关 | watchdog 用逻辑 tick（无进展判定），非墙钟 |
| 可观测 | ErrorRecord、watchdog 计时、`log_analyze` 报告、`--trace=error` |
| 契约影响 | `ErrorRecord`/`ResetManager`/复位语义/特权边界冻结 |
| 简化登记 | 初级"错误即中止" → 定位 + 按引擎复位 + watchdog |
| 计算兼容 | 错误钩子覆盖计算引擎（37/stage3 38~42，原 35.5）同样可注入 |
| 配置与回退 | `fault_inject`（测试）、`watchdog_timeout` 配置 |
| 调试钩子 | `--trace=error`（每次错误一行）；`--watchdog-log` |
| trace 契约 | 不进入判据 |
| 契约变更 | 复位/故障分层/特权边界/watchdog 为冻结项；变更走 ADR 回归 S13/S1 |
| 校验归属 | L1（结构/定位字段）+ L2（S13 矩阵）+ 横切（reset 重跑一致） |
| 双策略 | 复位与重跑在 `serial/parallel` 下一致（并行下故障仍可定位） |
| 阶段衔接 | 全项目收口：发动全部 E/S 场景回归矩阵 = 项目最终验收面 |

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
- watchdog timeout（tick 数）默认值是否与具体引擎速率耦合——按 tick 无色约束，默认取自配置，不改语义。
- 复位期间在飞提交的处理（排队 vs 拒绝）本期取"复位引擎丢弃未完成包 + 后续提交排队重跑"；若 E/S 双策略回归暴露顺序依赖，再补充确定性策略（ADR 记录）。
