# 第 15 章 控制流 + 波调度

> 本章为完整的 SIMT 执行补齐两条纵向能力：分歧控制流（控制流栈 `{pc, mask}` 与重收敛）与多波调度（CU 多 SIMD、就绪波队列、延迟隐藏挂起/唤醒）。初级单波概念在此扩展到"一波较多并发 + 分支轨迹确定"。基线的波执行模型见 `../design/isa-abi.md` §3。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（`IEngine` 握手与 `IShaderArray`）；`../design/execution-parallel.md` §4~§5（功能级流水线、延迟隐藏 ADR-036）；ADR-006/007（执行与波调度双策略）、ADR-018（warp32、掩码 u32、`{pc,mask}` 栈）。
> 落地模块：`hw_sim/shader/{divergence, scheduler}`、`hw_sim/shader/wave`（扩展多波容量）、`apps/scene_s3_triangle`。

---

## 1. 目标与非目标

### 1.1 目标

1. 分歧控制流：分支指令基于 lane 谓词分裂 `{pc, mask}`；`RECONVERGE` 重收敛；`SYNC_PUSH/POP` 栈管理；不发散区单 pc 快速执行。
2. 控制流栈结构 `{pc, mask}`（u32 掩码、全 1 为全活泼；取反必须 `& 0xFFFFFFFF`，工程红线 #1/#2）。
3. 波调度器：多波就绪队列、每 CU 多 SIMD、波领取固定轮转（策略 A）或原子领取（策略 B，第 31 章）。
4. 延迟隐藏：高延迟操作（纹理采样等）波挂起 → 让出 CU → 结果返回唤醒（就绪波队列维护，软观测波切换次数）。
5. 波执行数/Divergence 轨迹可观测；掩码轨迹与多次调度顺序确定（SM 可复现）。
6. `DIVERGENCE` 相关内建（`GET_LANE_ID` 等）正确；`LaneMask` 返回 u32；`ballot` 语义延至第 29 章。

### 1.2 非目标

- 不做 CU 阵列多线程与原子领取（第 31 章）；本章策略 B 仍报"策略 B 未实现"。
- 不做 subgroup ballot/shuffle（第 29 章）；不做 shared/atomic（第 30 章）。
- 不做纹理异步（第 21 章才做）；本章延迟隐藏用"合成延迟"占位验证。
- 不做可变 warp（固定 32）。

---

## 2. 单元框图

```
 波调度器 (Scheduler)
   │  ready queue(W0,W1,...)  挂起队列(等待高延迟)
   ▼
┌──────────────────────── shader/divergence ────────────────────────┐
│ CSw{cpc, stack[{pc,mask}]}   谓词掩码 dst                         │
│ BR: split / merge             RECONVERGE: 收敛检查                 │
│ EXIT: 出栈完成                                                   │
└────────────────────────────────┬──────────────────────────────────┘
                                 │ 每步取指+执行
                    wave::Dispatch（多波：每波独立执行实例）
```

- 每条指令执行在"当前栈顶 `{pc,mask}`"上：mask 中所有 lane 同步执行（SIMT）。
- 延迟隐藏：波发出高延迟请求 → 移到挂起队列 → 调度器选就绪波 → 请求返回 → 唤醒。

---

## 3. 输入输出契约

### 3.1 控制流指令集（`../design/isa-abi.md` §2 0x30~0x3F 区段）

| 指令 | 语义 | 栈作用 |
|---|---|---|
| `BR` (brc) | 条件分支到目标 | 分裂为 `{taken, mask∩cond}` 压新帧 |
| `RECONVERGE` | 重收敛点 | 出帧、合并掩码 |
| `SYNC_PUSH` / `SYNC_POP` | 子程序入口/出口 | push/pop `{pc, mask}` |
| `CALL` / `RET` | 调用/返回 | push/pop 返回地址 + mask |
| `EXIT`(0x7F) | 结束本波 | —（终端） |

- 当 mask 为全 0（全 lane 死亡）→ 波立即结束（不继续取指）。
- 分支目标偏移为有符号 32 bit，相对当前 pc。

### 3.2 波调度器接口

```cpp
struct WaveDesc { ContextId ctx; u32 simdId; u32 warpSize; GpuVAddr entryVa; u32 vgprBase; };
class WaveScheduler {
public:
    Result Enqueue(const WaveDesc&);          // 就绪波队列
    Result Tick();                             // 一轮调度：执行就绪波、挂起检查
    Result Suspend(WaveId, ResId);             // 高延迟挂起
    Result Resume(WaveId, ResId);              // 结果返回唤醒
    SchedStats Stats() const;                  // {readyCount, suspendedCount, dispatches, waveSwitch}
};
```

- 每 CU N 个 SIMD（`simd_per_cu` ∈ {1,2,4}）；策略 A 固定轮转挑选；波领取顺序确定。
- `waveSwitch` 供软观测（延迟隐藏效率）。

### 3.3 波内谓词

- 条件掩码保存在 `s_pred`（标量寄存器）或 `v_pred`（每 lane），由 `SETP`/`CMP` 写入。
- `BR` 使用谓词计算 `mask' = mask ∩ pred` 与 `mask'' = mask ∩ ¬pred`。

---

## 4. 内部数据流 / 状态机

### 4.1 分歧执行循环（每波）

```
while (true):
  (pc, mask) = top
  if mask == 0: EXIT
  inst = fetch(pc); decode
  case 执行束：switch (opcode):
    BR: taken=mask∩(pred)，notTaken=mask∩¬pred
        if taken && notTaken: push {target, taken}; push {fallthrough, notTaken}
        elif taken: pc=target
        elif notTaken: pc=fallthrough
    RECONVERGE: pop 至该分支起点帧，mask |= 栈底掩码
    SYNC_PUSH: push {pc+1, mask}
    SYNC_POP : pop → mask = 外层掩码 & 本次完成
    EXIT: break
  ...
```

- 栈深上限 256；超限唯一返回 `OutOfStack` 并产生可定位 ABORT 记录，不再同时返回 `InvalidIsa`。

### 4.2 调度与延迟隐藏

```
Tick:
  for each CU:
    SIMD[i] 空闲 && ready 非空 → 领取最前面波 → 执行一步
  sus 波：高延迟操作完成后 → ready 队列 + Resume（waveSwitch++）
```

- 判定顺序：ready FIFO 先进先出；波间无乱序（策略 A 确定性）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T15.1 | ① 契约 | `{pc,mask}` 栈、WaveDesc、SchedulerStats、控制流指令集落头文件 |
| T15.2 | ② 主体 | 条件分支：taken/notTaken 掩码分裂与两条路径派发 |
| T15.3 | ② 主体 | `RECONVERGE` 重收敛：出帧合并掩码 |
| T15.4 | ② 主体 | `SYNC_PUSH/POP/CALL/RET` 子程序与返回地址 |
| T15.5 | ② 主体 | 全 0 掩码提前结束、掩码位宽 u32 断言、栈深超限错误 |
| T15.6 | ② 主体 | 多波容量：波调度器多波队列 + 每 CU 多 SIMD 轮转 |
| T15.7 | ② 主体 | 延迟隐藏：Suspend/Resume、就绪波队列、`waveSwitch` 统计 |
| T15.8 | ② 主体 | Divergence 轨迹输出：每波 `{pc, mask}` 序列（调试） |
| T15.9 | ② 主体 | 策略 A 固定轮转确定：多次运行调度顺序一致 |
| T15.10 | ③ 接入 | `scene_s3_triangle` 分歧着色器全流程接入（if/else、loop、discard） |
| T15.11 | ④ 验收 | A15：S3 主判据 + 专有判据（LaneMask 轨迹、调度顺序）+ R15 回归项 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M15.1 | T15.1~T15.5 | 控制流栈/分支/重收敛/子程序/边界错误，D15.1~D15.5 全绿 |
| M15.2 | T15.6~T15.11 | 多波调度/延迟隐藏/策略 A 确定性/场景接入验收，A15 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D15.1 | `{pc,mask}` 结构体 u32 位宽、不做位域（工程红线 #2 代码审查）；接口独立编译 | L1 契约 |
| D15.2 | 单 if/else：一个分支两个路径 lane 各执行一遍、`RECONVERGE` 后 mask=全1 | L1 契约 |
| D15.3 | 嵌套 if：内层分裂/合并后外层掩码不被污染（栈 LIFO 正确） | L1 契约 |
| D15.4 | `CALL/RET`：返回地址正确、`RET` 后 mask 恢复 | L1 契约 |
| D15.5 | 全 0 掩码波提前 `EXIT`；栈超 256 返回 `OutOfStack`（定位） | L1 契约 |
| D15.6 | 单次 dispatch 的 32 个 wave 均可入队并最终完成；任一 CU 同时驻留不超过 B-037 的 16 waves，超出部分保持 pending 并在资源释放后推进；每 SIMD 顺序符合固定轮转，`Stats.dispatches` 正确 | L1 契约 |
| D15.7 | 挂起后该 SIMD 执行其他就绪波（`waveSwitch++`）；`Resume` 后波从中断点继续 | L1 契约 |
| D15.8 | 分歧轨迹两次运行完全一致（`{pc,mask}` 快照序列） | L1 契约 |
| D15.9 | 策略 A 下多次调度 `dispatches`/顺序位图一致；`exec_mode=parallel` 报"策略 B 未实现" | L1 契约 |
| D15.10 | S3 主线（分支着色的输出 = ref_model）+ 专有判据（LaneMask 轨迹、调度顺序） | L2 解析 + A15 |

---

## 7. 场景与判据（A15）

**绑定场景：S3 `scene_s3_triangle`（分歧着色器）。**

| 项 | 内容 |
|---|---|
| 输入 | S3：带 if/else、loop、`discard` 的 PS/VS；control flow 探针（每 lane 分支结果、收敛掩码） |
| 产物 | 分歧轨迹 `{pc,mask}` 快照、调度统计、最终 RT |
| 主判据 | S3 分支场景输出 = ref_model 期望（探针零容差）；全图掩码差分仅告警 |
| 本章专有判据 | ① LaneMask 轨迹符合预期（if/else 分裂收敛、discard 置零掩码）；② 多次调度（重启）顺序确定 |
| 回归项 | R15：E2（初级无分支）仍逐字节一致；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 第 29/30 章的 subgroup/shared 在 S10/S18 用；第 31 章策略 B 原子领取替换轮转 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 栈操作 LIFO、分支路径派发顺序固定、调度轮转固定 |
| 时间无关 | 无墙钟；延迟隐藏用逻辑挂起事件，不靠 timer |
| 可观测 | `{pc,mask}` 轨迹、`Stats.dispatches/waveSwitch`、`--trace=divergence` |
| 契约影响 | 控制流指令集与波调度接口冻结；`{pc,mask}` 结构体冻结 |
| 简化登记 | 单波无分支 → 多波 + 分歧 + 延迟隐藏 |
| 计算兼容 | 分歧/调度不假设 PS 阶段；compute kernel 同样适用（S18 消费） |
| 配置与回退 | `exec_mode=parallel` 仍报未实现（31 章前）；`simd_per_cu`（1/2/4）可配置，仅影响调度分布与软观测 |
| 调试钩子 | `--trace=divergence` 单行；`--trace=scheduler` 每调度步一行 |
| trace 契约 | 不进入判据 |
| 契约变更 | 控制流指令 opcode、栈语义、调度顺序为冻结项；变更走 ADR 回归 S3 |
| 校验归属 | L1（结构/掩码/栈）+ L2（分支着色输出与 ref_model 比对） |
| 双策略 | 策略 A 轮转确定；策略 B（原子领取）延至 31，S3 两策略零容差是最终目标 |
| 阶段衔接 | 为第 16 章汇编器、17 章编译器提供分歧控制流后端目标 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 分支 | 无分支 → 完整分歧控制流 | —（定案） |
| 调度 | 单波 → 多波队列 + 轮转 | 31（策略 B 原子领取） |
| 延迟隐藏 | 合成延迟挂起/唤醒 | 21（真纹理异步） |
| 波领取 | 固定轮转 | 31 |

---

## 10. 待定项

- `discard`/`clip` 在分歧控制流里的掩码归零语义与光栅覆盖率交互，需在 OM（第 26 章）和可见性（第 27 章）定案时统一口径（以 `discard` 置 0 lane 的像素不写回为准）。
- 延迟隐藏的"合成延迟"时长配置仅影响软观测；真实纹理延迟在第 21 章替代后删除占位。
- 栈深上限 256 是否合理，取决于第 17/18 章编译器生成的调用图深度；如超限频繁以配置可调，不冻结数值。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A15 和适用的 R15 后提交。分配核对项包括控制流 opcode、PC 偏移单位、lane mask。变更按已有分配流程登记。

```text
feat(gpu-ch15): schedule waves with divergent control flow

- 扩展分支、调用和重收敛，并按规则调度波。
- 对照 S3 分歧 trace 与恢复掩码。

Tasks: T15.1–T15.11
Validation: <实际执行的命令、用例和结果>
```
