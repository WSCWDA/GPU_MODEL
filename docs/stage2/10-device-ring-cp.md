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

### 3.4 MMIO 设备寄存器（ADR-054、B-042）

[公共 MMIO 登记 §4](../design/command-mmio.md)维护偏移和权限。第 10 章沿用初级寄存器，新增 CP 控制、中断、上下文、设备与 ring 标识、诊断和特权复位入口。PSO 与 stateId 引用同一登记。`CMD_VERSION` 仅回显近期接受值，版本判定依据 `STREAM_HEADER`。

---

## 4. 内部数据流 / 状态机

### 4.1 CP 状态机（FSM，非过程式）

```
IDLE → WAIT_RING（ring 非空？）→ FETCH_HEADER（读 1 dword 包头）
     → PARSE（PKT_TYPE/count 校验，count>0 且 ≤4095）
     → DISPATCH（按类型：内部处理 or Push 到引擎）
     → ADVANCE（head 推进 count 个 dword；跨尾前由生产者用 NOP 填充）
     → WRITE_FENCE（若包为 FENCE）→ IDLE
错误转移：非法类型/版本/长度 → ERROR（记录并失败完成当前提交）→ IDLE；显式诊断 ResetEngine 路径设置 ResetRequired，实际恢复由第 36 章接入。
```

断言：`count` 含头、`count≥1`、包不跨环尾、`PKT_TYPE` 在登记表内。

### 4.2 环读写指针

- `tail`（KMD 写）与 `head`（CP 读）均为 u32 dword 偏移，单调推进；可用容量 = `size-1`，满 = `(tail+1)%size == head`，空 = `head == tail`。
- 入队前检查余量；不足则先写 `NOP(count=1)` 填满到环尾，再在环头续包。
- 回绕正确性判据：`N > size` 的连续提交必须完整执行，fence 序列不重不漏。

### 4.3 fence 与完成通知

```
FENCE 包载荷 {u64 value} → CP 更新 FENCE_MON_LO/HI 与 FENCE_STATUS → 触发中断
→ KMD 查询指定提交，成功读回 64 bit fence，失败返回该提交的错误
```

- 值单调递增；同一队列内提交顺序 = fence 写序。
- `WaitFence` 退化为"提交同步完成后读回断言"（本章单上下文、串行）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T10.1 | ① 契约 | 准备 DDI、固件和引擎接口及错误记录。B-042 核对 dword 偏移、权限、位宽，并审计第 2、3、11 章使用的子集 |
| T10.2 | ② 主体 | 设备创建/销毁、能力查询（`config.commandVersion`、引擎列表）、MMIO bucket 复位清零 |
| T10.3 | ② 主体 | 实现 §3.4 启用子集，验证读写权限、门铃等副作用及非法偏移拒绝 |
| T10.4 | ② 主体 | HW→FW→KMD 创建及失败回滚。关闭时停止接收，推进已接受工作至终态，再停 CP 并逆序释放 |
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
| T10.15 | ④ 验收 | A10：S1 空环 + S13 报错子集全绿（场景主判据 + 本章专有判据 + R10 回归项） |

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
| D10.1 | 接口独立编译并核对包头字段。按 B-042 将实现和启用子集分别对照公共表，检查偏移、权限、位宽和碰撞。旧整表相等判据为 **SUPERSEDED** | L1 契约 |
| D10.2 | 设备创建两次得到两实例；能力查询 `version` 与构造参数一致；第二次创建前 MMIO 全 0 | L1 契约 |
| D10.3 | RW 项可回读，RO 项通过设备事件更新并拒绝外部写入，WO 项验证触发效果。非法偏移返回 `InvalidArgument` 且保持状态 | L1 契约 |
| D10.4 | HW→FW→KMD 创建顺序正确；任一级创建失败只回滚已建对象。带一个待处理提交关闭时先产生成功或失败完成，再停 CP；资源引用归零 | L1 契约 |
| D10.5 | `ResetEngine` 使 CP 停止且 `ResetRequired` 置位；`scene` 模拟 UMD 写 `RESET_CTRL(0xF0)` 返回 `NotSupported`（特权拒绝） | L1 契约 |
| D10.6 | `N=size-1` 填满环（可用容量）后 `tail+1==head`（满）；消费后 `head==tail`（空）；回绕一次后读写指针差值正确 | L1 契约 |
| D10.7 | `version != commandVersion` 的首包：返回 `InvalidPacket`，错误记录 `{unit=cp, packetType=0x00, dwordOffset=0}`；合法首包继续 | L1 契约 |
| D10.8 | `count=0` 拒绝；`count>4095` 拒绝；未知 `PKT_TYPE` 拒绝且报类型与偏移 | L1 契约 |
| D10.9 | 环余量 < 包长时，环尾出现 `NOP(count=1)` 填充、包从环头完整落地，fence 值不减 | L1 契约 |
| D10.10 | Push/Pump/Pop 的队列状态可以检查；FIFO 满返回 `Busy` 且可重推；非法包产生失败完成和位置记录。独立复位及重建验收归 A36 | L1 契约 |
| D10.11 | 连续两个 `FENCE(1)`/`FENCE(2)`：`WaitFence(2)` 返回后 `FENCE_MON==2`；先等待大值再小值不跳序 | L1 契约 |
| D10.12 | `ProgramRing` 后从 `RING_BASE_LO/HI`、`RING_SIZE` 还原地址和容量。`OnDoorbell` 后 `CP_STATUS` 为 running | L1 契约 |
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
| 回归项 | R10：E0–E5 不因新软件栈退化；本阶段门禁冻结 ring、包格式和 KMD/FW 接口；未列明历史 A 门禁不自动阻断 |
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
| 错误 | 当前提交进入失败终态；显式诊断 ResetEngine 入口验证 `ResetRequired`（见 `architecture-design.md` §3.9） | 36 |

---

## 10. 待定项

- `DISPATCH` 包的载荷布局（grid/block/paramBufVA）在第 20/37（原 35.5） 章定案；本章只登记类型号 0x04，具体字段后续追加，不反向修改包头。
- `IEngine::Stats` 的软观测口径（占用率是否加权重）在第 25/31 章按 FIFO 级细化；本章只保证可输出与单调性。
- `WaitFence` 在本章为"同步完成后值断言"；第 32 章引入真异步提交后是否保留同步便捷层，待第 32 章场景 S12 定案。

---

## 2026-11-21 评审同步说明

公共偏移见 命令与 MMIO 登记 §4，实现审计见 D10.1。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A10 和适用的 R10 后提交。分配核对项包括版本字段、ring/MMIO 寄存器。变更按已有分配流程登记。

```text
feat(gpu-ch10): route submissions through FW and CP

- 建立高级设备、FW/CP 边界与 v2 环消费。
- 以 S1 观测空环和完成值，以 S13 子例验证非法包或版本。

Tasks: T10.1–T10.15
Validation: <实际执行的命令、用例和结果>
```
