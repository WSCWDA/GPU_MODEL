# 第 10 章 设备 + 环 + CP

> 本章是高级阶段的入口：把初级薄提交门面替换为 KMD/FW 驱动的真实提交路径，落地最小 KMD DDI、Firmware boot/ring/doorbell、以及按功能级握手定义的 `IEngine`。一条提交从"写环 → doorbell → CP 取指 → 派发 → fence"被完整建模，环回绕、包头版本、非法包定位成为本章核心专有判据。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（`IEngine` 握手签名）、§3.5（KMD/FW 契约）、§3.10（回调链生命周期）；`../design/command-mmio.md` §1~§4（包头/包表/stateId/MMIO 分段）；`../design/boot-reset.md` §1、§5（启动序列、逻辑特权分层）；`../design/execution-parallel.md` §4（功能级流水线/背压）。
> 落地模块：`hw_sim/core/{device,mmio,ring,cp,fence,interrupt}`、`firmware/{boot,scheduler}`、`kmd/{ddi,init,submission,interrupt}`、`umd/os`、`apps/scene_s1_empty_submit`。

---

## 1. 目标与非目标

### 1.1 目标

1. 落地最小 KMD DDI 与 Firmware：`CreateDevice/Submit/WaitFence` 写环 + doorbell，`ProgramRing/OnDoorbell/LoadContext` 启动 CP（契约 `architecture-design.md` §3.5）。
2. 冻结高级 SW stack：`apps → kmd → firmware → hw_sim/core`，回调链按创建/销毁时序执行（`architecture-design.md` §3.10）。
3. 环回绕正确：读/写指针单调推进、包不跨环尾（不足用 `NOP` 填充）、`N > 环容量` 提交仍完整执行（工程红线 #6）。
4. CP 按包类型派发：`STREAM_HEADER/FENCE/SET_STATE_BASE/CLEAR/NOP` 内部处理，`COPY_MEM/DRAW/DISPATCH` 走引擎（`../design/command-mmio.md` §2）。
5. `IEngine` 按 Push/Pump/Pop 握手实现（输入/输出 FIFO + valid/ready + 背压软观测，ADR-035）；`Execute` 为同步便捷封装。
6. fence 语义：`FENCE` 包写 MMIO 监视寄存器；提交完成顺序与 fence 值单调一致（`../design/sync-primitives.md` §1）。
7. 特权分层：`reset/页表根/watchdog` 经 KMD-DDI 封装由 Firmware 执行，UMD 无权直接下发（ADR-040）。

### 1.2 非目标

- 不做多队列/多引擎仲裁、不做 timeline semaphore（第 32 章）；环只有图形 Ring。
- 不做地址翻译/页表/TLB（第 11 章）；`IMemorySubsystem` 仍用初级平坦实现。
- 不做多引擎 watchdog/完整复位状态机（第 36 章），仅保留 `ResetRequired` 入口与错误记录。
- 不做上下文切换/占用率（第 14、31 章）；单上下文。
- 不做 setup 派发语义（第 22~28 章）；`DRAW` 派发先落到占位引擎。

---

## 2. 单元框图

```
 KMD.Submit ── 写环(增长tail) ──► 环内存(Vec<u32>) ──► doorbell(MMIO)
                                                          │
   FENCE 包 ──► FW ──► CP 取指 ──► 解析包头 ──► 按类型派发
                                                     │
        ┌────────────────────────────────────────────┤
        ▼
  STREAM_HEADER/FENCE/SET_STATE_BASE/CLEAR/NOP  →  CP 内部处理（置 MMIO / 写 fence）
  COPY_MEM/DRAW/DISPATCH  →  Push(Pump/Pop 握手) →  IEngine（功能级 FIFO）
                                                     │  Count == 0 排空
                                                     ▼
                                          中断通知 KMD WaitFence 值断言
```

单元间依赖：`kmd → firmware（只经 DDI/MMIO/环）→ hw_sim/core`；`firmware` 不解析 draw 语义。

---

## 3. 输入输出契约

### 3.1 KMD / FW 契约（冻结于本章，后续只追加不改签名）

```cpp
namespace kmd {
class IKmDdi {
public:
    virtual Result CreateDevice(const DeviceDesc&, IDevice** out) = 0;
    virtual Result AllocGpuMem(const AllocDesc&, GpuVAddr* outVa, void** outHost) = 0;
    virtual Result FreeGpuMem(GpuVAddr va) = 0;
    virtual Result CreateContext(u32 asid, IContext** out) = 0;
    virtual Result Submit(u32 queueId, const SubmitDesc&) = 0;  // 写环 + doorbell
    virtual Result WaitFence(u32 queueId, u64 value) = 0;
    virtual Result ResetEngine(EngineType) = 0;
};
}

namespace firmware {
class IFirmware {
public:
    virtual Result Boot(const BootDesc&) = 0;
    virtual Result ProgramRing(u32 ringId, GpuVAddr base, u32 sizeDwords) = 0;
    virtual Result OnDoorbell(u32 ringId) = 0;   // 加载页表基址，启动 CP
    virtual Result LoadContext(ContextId, GpuVAddr pageTableBase) = 0;
    virtual Result Halt() = 0;
};
}
```

规则：

- `Submit` 只写环、更新 tail、写 doorbell；取指由 CP 完成，KMD 不得直接调用引擎。
- doorbell 处理中 FW 只做"加载页表基址 + 启动 CP + 记录上下文"，不解析 draw。
- 中断只做通知；KMD 从 MMIO 读回完成值，不依赖中断载荷。

### 3.2 `IEngine` 握手签名（冻结于本章）

```cpp
namespace hw {
class IEngine {
public:
    virtual ~IEngine() = default;
    virtual EngineType Type() const = 0;
    virtual Result Push(const u32* pkt, u64 dwordCount, ContextId ctx,
                        IMemorySubsystem* mem, shader::IShaderArray* sa) = 0;
    virtual Result Pump() = 0;             // 功能级"周期"
    virtual Result Pop() = 0;              // 结果出队
    virtual bool   IsIdle() const = 0;     // 全部 FIFO 为空
    virtual CycleStats Stats() const = 0;  // FIFO 占用率、反压周期计数（软观测）
    virtual Result Execute(const u32* pkt, u64 dwordCount, ContextId ctx,
                           IMemorySubsystem* mem, shader::IShaderArray* sa) = 0; // 同步封装
};
}
```

- `FENCE/SET_STATE_BASE/CLEAR/NOP` 由 CP 处理，不经过引擎。
- `COPY_MEM` → Copy 引擎且 `sa=nullptr`；`DRAW` → 3D；非着色引擎必须容忍 `sa == nullptr`。
- 输入 FIFO 满 → `Push` 返回 `Busy`，触发上游背压停发。

### 3.3 包头与版本（本章收费包）

| 项 | 约定 |
|---|---|
| 包头 | `[31:28]=PKT_TYPE`、`[27:16]=count(含头,dword)`、`[15:0]=opcode/载荷`（`../design/command-mmio.md` §1） |
| 首包 | 命令流首包必须 `STREAM_HEADER`(0x00)，载荷 `{u32 version; u32 engine; u32 flags}` |
| 版本 | 与设备 `config.commandVersion` 不匹配 → 拒绝并报包类型 + dword 偏移 |
| 环填充 | 包不得跨环尾；空间不足时在环尾写 `NOP`(0x07,count=1) 填充后再续包 |
| 包表 | `STREAM_HEADER` 与版本/引擎/标志、`FENCE`(0x01)、`SET_STATE_BASE`(0x02)、`DRAW`(0x03)、`COPY_MEM`(0x05)、`SHADER_PSO`(0x06)、`NOP`(0x07)、`CLEAR`(0x08)——初章派发表的最小集合 |

### 3.4 MMIO 分段（`../design/command-mmio.md` §4，本章落地 0x00~0x1F、0x20~0x2F、0x30、0x38、0xE0~0xE1、0xF0）

本节 MMIO 偏移一律为 **dword 索引**（stateId n → `0x40+2n`，`0x50~0x7F` 保留，Q-31）。

> ⚠️ stateId 编号纪律（三档，`../design/bit-budget.md` §3.1）：①`0~7` 直接 MMIO 映射（`0x40+2n`）冻结、兼容 E0~E5；②`0x50~0x7F`（stateId 8~31）只是**临时过渡缓冲**，不得作为长期设计；一旦耗尽必须切换为③GPU 内存间接状态表（MMIO 只存根指针）。**禁止继续在 MMIO 硬编码新增 stateId**；本章不新增 stateId，只读 `0x40~0x4F` 段。

| 段 | 本章寄存器 | 用途 |
|---|---|---|
| 0x00~0x0F | RING_WPTR(0x00)、RING_RPTR(0x01)、RING_SIZE(0x02)、DOORBELL(0x04)、FENCE_MON(0x08) | ring 指针、doorbell、fence 监视 |
| 0x10~0x1F | CP_CTRL(0x10)、INT_STATUS(0x14)、CTX_STATE(0x18) | CP 控制、中断、上下文状态 |
| 0x20~0x2F | ERROR_CODE(0x20)、ERROR_PKT_TYPE(0x21)、ERROR_DWORD_OFFSET(0x22)、DEBUG0(0x24)、HW_STATS_BASE(0x28) | 错误定位、调试、性能计数 |
| 0x30~0x3F | DEV_RING_ID(0x30)、CMD_VERSION(0x38) | 设备/ring ID、命令流版本回显 |
| 0x40~0x4F | stateId 0~7 LO/HI 基址 | 当前上下文状态基址 |
| 0x80~0xDF | PSO 暂存区 | SHADER_PSO 状态副本（第 6 章） |
| 0xE0 | RT_WIDTH(0xE0) | RT 宽度（完整 32 bit） |
| 0xE1 | RT_HEIGHT(0xE1) | RT 高度（完整 32 bit） |
| 0xE2~0xEF | — | 多上下文 / 多队列 / SE / 并行统计预留 |
| 0xF0~0xFF | RESET_CTRL(0xF0) | 系统保留 / 紧急复位（特权） |

`CMD_VERSION`(0x38) 只回显 `STREAM_HEADER` 版本；判定以包为准。

---

## 4. 内部数据流 / 状态机

### 4.1 CP 状态机（FSM，非过程式）

```
IDLE → WAIT_RING（ring 非空？）→ FETCH_HEADER（读 1 dword 包头）
     → PARSE（PKT_TYPE/count 校验，count>0 且 ≤4095）
     → DISPATCH（按类型：内部处理 or Push 到引擎）
     → ADVANCE（tail 推进 count*dword；跨尾用 NOP 填充）
     → WRITE_FENCE（若包为 FENCE）→ IDLE
错误转移：非法类型/版本/长度 → ERROR_{report,ResetRequired} → IDLE（后续提交被拒直到复位）
```

断言：`count` 含头、`count≥1`、包不跨环尾、`PKT_TYPE` 在登记表内。

### 4.2 环读写指针

- `tail`（KMD 写）与 `head`（CP 读）均为 u32 dword 偏移，单调推进；可用容量 = `size-1`，满 = `(tail+1)%size == head`，空 = `head == tail`。
- 入队前检查余量；不足则先写 `NOP(count=1)` 填满到环尾，再在环头续包。
- 回绕正确性判据：`N > size` 的连续提交必须完整执行，fence 序列不重不漏。

### 4.3 fence 与完成通知

```
FENCE 包载荷 {u64 value} → CP 写 FENCE_MON(0x08) → 触发中断(INT_STATUS)
→ KMD WaitFence(queueId, v) 读回 FENCE_MON，v > 已读值 → 完成
```

- 值单调递增；同一队列内提交顺序 = fence 写序。
- `WaitFence` 退化为"提交同步完成后读回断言"（本章单上下文、串行）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T10.1 | ① 契约 | `IKmDdi`/`IFirmware`/`IEngine`/包头发送/MMIO 分段/错误记录结构落头文件并独立编译。**含 A 阶段前置执行项 B-042**：`mmio.h` 按 dword 索引重排 stateId/PSO/多上下文 + 回写第 2/10 章偏移表（裁定见 `bit-budget.md` §9 第 6 项、`roadmap.md` §2.3 A 行/§7.3，A 门禁无豁免） |
| T10.2 | ② 主体 | 设备创建/销毁、能力查询（`config.commandVersion`、引擎列表）、MMIO bucket 复位清零 |
| T10.3 | ② 主体 | MMIO 寄存器映射：`0x00~0x3F`、`0xF0` 读写 + 回读一致性、非法偏移拒绝 |
| T10.4 | ② 主体 | 回调节点：创建 HW→FW boot→KMD→（HLI/UMD 后置）逆序释放；Shutdown 必停 CP、等队列空闲 |
| T10.5 | ② 主体 | KMD DDI 入口 + 特权封装：`ResetEngine` 走 FW、页表根仅 FW 可写（ADR-040） |
| T10.6 | ② 主体 | 环初始化/写读指针、tail/head 推进、满/空判定、回绕 |
| T10.7 | ② 主体 | `STREAM_HEADER` 解析：版本/引擎/标志校验，不匹配拒绝报类型+偏移 |
| T10.8 | ② 主体 | 包头解析：`count` 校验、`PKT_TYPE` 登记表查表、`NOP` 跳过 |
| T10.9 | ② 主体 | 环尾填充：包不跨尾、余量不足写 `NOP`；跨尾续包正确 |
| T10.10 | ② 主体 | `IEngine` 握手化：Push/Pump/Pop 驱动输入/输出 FIFO、`Busy` 背压、Stats 软观测 |
| T10.11 | ② 主体 | `FENCE` 包：写 `FENCE_MON`、触发中断、值单调 |
| T10.12 | ② 主体 | FW boot/ring 编程/`ProgramRing`/`OnDoorbell` 启动 CP、`LoadContext` 占位 |
| T10.13 | ② 主体 | `Submit`：写环 + 更新 tail + doorbell；中断读回完成值 |
| T10.14 | ③ 接入 | `scene_s1` 空环提交：`Submit([]) → WaitFence` 命令链不断链、无包执行仍推进 fence |
| T10.15 | ④ 验收 | A10：S1 空环 + S13 报错子集全绿（场景主判据 + 本章专有判据 + A1~A9 回归） |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M10.1 | T10.1~T10.5 | 接口/设备/MMIO/回调/特权契约可编译并过 D10.1~D10.5 |
| M10.2 | T10.6~T10.10 | 环/包解析/握手全链路，D10.6~D10.10 全绿 |
| M10.3 | T10.11~T10.15 | fence/FW/提交/接入/验收，A10 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D10.1 | 三接口头文件独立编译；包头发送字段偏移 `static_assert`；MMIO 段号与 §3.4 表一一对应；**B-042 判据**：`mmio.h` 全部偏移为 dword 索引（脚本/`static_assert` 校验），第 2/10 章偏移表与代码逐项一致（脚本比对） | L1 契约 |
| D10.2 | 设备创建两次得到两实例；能力查询 `version` 与构造参数一致；第二次创建前 MMIO 全 0 | L1 契约 |
| D10.3 | 写后读回每个已登记寄存器逐字节相等；非法 MMIO 偏移访问返回 `InvalidArgument` 且不改状态 | L1 契约 |
| D10.4 | 创建序列回调顺序 = HW→FW→KMD；在 KMD 阶段注入 boot 失败 → 已建对象全释放且无泄漏计数 | L1 契约 |
| D10.5 | `ResetEngine` 使 CP 停止且 `ResetRequired` 置位；`scene` 模拟 UMD 写 `RESET_CTRL(0xF0)` 返回 `NotSupported`（特权拒绝） | L1 契约 |
| D10.6 | `N=size-1` 填满环（可用容量）后 `tail+1==head`（满）；消费后 `head==tail`（空）；回绕一次后读写指针差值正确 | L1 契约 |
| D10.7 | `version != commandVersion` 的首包：返回 `InvalidPacket`，错误记录 `{unit=cp, packetType=0x00, dwordOffset=0}`；合法首包继续 | L1 契约 |
| D10.8 | `count=0` 拒绝；`count>4095` 拒绝；未知 `PKT_TYPE` 拒绝且报类型与偏移 | L1 契约 |
| D10.9 | 环余量 < 包长时，环尾出现 `NOP(count=1)` 填充、包从环头完整落地，fence 值不减 | L1 契约 |
| D10.10 | `IsIdle()` 在 Push 后为 false、排空后为 true；FIFO 满时 `Push` 返回 `Busy` 且包不丢；`Stats` 可输出占用率与反压计数（软观测）。**新增 L1 契约单测（正常+非法边界）**：①FIFO 满 `Push`→`Busy` 且包不丢可重推；②Pump 遇非法包 → 引擎中止并报 `{unit,packetType,dwordOffset}`（不静默跳过）；③复位后重新 `Push/Pump` 握手恢复 | L1 契约 |
| D10.11 | 连续两个 `FENCE(1)`/`FENCE(2)`：`WaitFence(2)` 返回后 `FENCE_MON==2`；先等待大值再小值不跳序 | L1 契约 |
| D10.12 | `ProgramRing` 后 MMIO 回读 `base/size`；`OnDoorbell` 后 `CP_STATE` 为 running | L1 契约 |
| D10.13 | `Submit` 后 ring tail 推进恰等于包长 dword；doorbell 计数 +1；`WaitFence` 读回完成值 | L1 契约 |
| D10.14 | `scene_s1` 空命令流：无元素执行但 fence 从 a → b 单调推进且两次运行一致；命令链从场景到 CP 无断链 | L1 契约 + A10 |
| D10.15 | A10 全绿：S1 空环专有判据（回绕/填充/非法包/fence 顺序）零容差；S13 报错子集（非法包、版本拒绝）可定位 `{unit,pktType,dwordOffset}` | L1 契约 |

---

## 7. 场景与判据（A10）

**绑定场景：S1 `scene_s1_empty_submit`（空环验收）、S13 报错子集（本章只收非法包/版本拒绝，完整复位在第 36 章）。**

| 项 | 内容 |
|---|---|
| 输入 | S1：空命令流（仅 `STREAM_HEADER` + `FENCE`）、环容量 `size∈{64, 1024}`、`N>size` 的连续提交序列；S13 子集：未知 `PKT_TYPE`、`version` 不匹配首包 |
| 产物 | ring 快照、fence 值序列、错误记录 `{unit,pktType,dwordOffset,va,code}` |
| 主判据 | S1：`N>size` 提交后 fence 序列单调完整；每次运行 ring/fence 快照一致 |
| 本章专有判据 | ① 环回绕正确（见 D10.6）；② 包不跨环尾、`NOP` 填充正确（D10.9）；③ 非法包报类型+偏移（D10.8）；④ 版本不匹配拒绝（D10.7）；⑤ fence 语义顺序（D10.11） |
| 回归项 | A1~A9 全绿（E0~E5 不因新 SW stack 退化；本阶段门禁：环/包格式/KMD-FW 接口冻结） |
| 后续补齐 | 多引擎 ring 仲裁在第 32、33 章；S13 完整故障注入与复位在第 36 章收口 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 环读写指针单调；包按提交顺序派发；fence 值与提交序一致；无随机 |
| 时间无关 | 无墙钟/超时进入硬件状态；`WaitFence` 是值断言 |
| 可观测 | `Stats{FIFO 占用率, 反压周期}`、`ERROR_CODE(0x20)`、ring 快照、`--trace=cp` 单行 |
| 契约影响 | `IKmDdi/IFirmware/IEngine/包头/MMIO 分段`冻结；后续章只追加方法/字段 |
| 简化登记 | 单环形、单上下文、无 TLB、无多引擎（见 §9） |
| 计算兼容 | `DISPATCH` 在包表登记（0x04），派发点与 DRAW 同构；引擎不判断上层类型 |
| 配置与回退 | `commandVersion`、`ringSize` 配置化；策略 B 仍报未实现（第 31 章前） |
| 调试钩子 | `--trace=cp/kmd/fw` 仅供人工；单行、字段确定性 |
| trace 契约 | 不进入判据 |
| 契约变更 | 包头/包表/MMIO 段/DDI 形状为冻结项；变更走 ADR 并回归 S1/S13 |
| 校验归属 | L1 契约（结构/状态机/fence 顺序）为主；S1 全链路由 `RefModel` 对 fence/ring 做解析断言（L2） |
| 双策略 | CP 是单点消费者，环与 fence 在 `serial/parallel` 下行为一致（S1 两策略零容差） |
| 阶段衔接 | A 阶段门禁 1（S1/S2/S13）+ 门禁 2（契约冻结）+ 门禁 3（E0~E5 不退化）在此章起算 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 提交通道 | 初级门面 → 真实 KMD 写环 + FW doorbell + CP 取指 | 35.1 |
| 内存 | 初级平坦 `IMemorySubsystem` 原样保留 | 11 |
| 引擎 | Copy/3D 引擎暂为占位，`Execute` 同步封装 | 22~28、13 |
| 队列 | 单图形 Ring；`DISPATCH` 仅登记不派发 | 32、33、37（原 35.5） |
| 错误 | 只记录 `{unit,pktType,dwordOffset}` + 置 `ResetRequired`（人工诊断例外，见 `architecture-design.md` §3.9）；无复位状态机 | 36 |

---

## 10. 待定项

- `DISPATCH` 包的载荷布局（grid/block/paramBufVA）在第 20/37（原 35.5） 章定案；本章只登记类型号 0x04，具体字段后续追加，不反向修改包头。
- `IEngine::Stats` 的软观测口径（占用率是否加权重）在第 25/31 章按 FIFO 级细化；本章只保证可输出与单调性。
- `WaitFence` 在本章为"同步完成后值断言"；第 32 章引入真异步提交后是否保留同步便捷层，待第 32 章场景 S12 定案。

---

## 2026-11-21 评审同步说明

本章 MMIO 表为唯一硬件寄存器映射落地点，必须按 `review-sync-20261121.md` §2.2 同步：RT_WIDTH/HEIGHT=0xE0/0xE1，DEV_RING_ID=0x30；PSO 与保留偏移按 canonical table。CP ADVANCE 推进 head/read pointer；第 2 章是子集视图。
