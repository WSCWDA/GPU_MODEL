# 第 2 章 设备 + 单环

> 本章建立"提交 → 取指 → 派发 → 完成"的最小闭环：设备对象与 MMIO 寄存器块、单个命令环的读写指针、doorbell、CP 状态机、fence 写值与完成通知。本章结束时，一条只含 `STREAM_HEADER`/`NOP`/`FENCE` 的命令流可以从薄门面提交并确定性地完成。
> 与本章相关的设计说明：`../architecture-design.md` §3.2（薄提交门面）、§3.3（`IEngine` 握手签名）、§3.10（回调链）；`../design/command-mmio.md` §1~§2、§4（包头、包类型、MMIO 分段）；`../design/sync-primitives.md` §2（fence 规则）；`../design/logging.md` §3（硬件行为校验）。
> 落地模块：`hw_sim/core/{device,mmio,ring,cp,fence}`、`hw_sim/submit/facade`、`apps/scene_e1_clear_present`（本章只跑其提交子集）。

---

## 1. 目标与非目标

### 1.1 目标

1. 设备对象：创建/销毁、能力标志、MMIO 寄存器块的分配与读写（配置、状态、统计的唯一通道）。
2. 单环：环基址/大小/读指针(head)/写指针(tail) 的编程与回绕，`count` 不跨环尾。
3. doorbell：写寄存器唤醒 CP；CP 从 head 取包、推进 head、写回 MMIO。
4. CP 状态机：按包类型分派的最小版本，内部处理 `STREAM_HEADER` / `NOP` / `FENCE`。
5. fence：CP 写 64 bit 单调值并置完成位；`Wait` 退化为值断言；完成计数 = 提交计数。
6. 薄提交门面骨架：`submit::Submit(desc)` / `submit::Wait()` 可编译、可提交空命令流。
7. 非法输入定位：非法包类型、`count` 越界、版本不匹配、包跨环尾，均返回带 `{unit, packetType, dwordOffset}` 的错误。

### 1.2 非目标

- 不做内存分配、资源上传、清屏（第 3 章）；不做执行与着色（第 4~6 章）。
- 不做中断控制器、watchdog、多环仲裁（第 10、32、36 章）。
- 不做 KMD/FW 分层；本章所有结构位于 `hw_sim` 内，由薄门面直接驱动。
- 不做真实并发：单线程消费，CP 的 `Pump` 由提交路径同步推进。

---

## 2. 单元框图

```
 apps/scene_e1（提交子集）
        │ Submit(desc) / Wait()
        ▼
 hw::submit::Facade ──打包──► 命令流（STREAM_HEADER … FENCE）
        │                          │ 写入 ring 内存
        ▼                          ▼
 ┌──────────────────── hw_sim/core ─────────────────────┐
 │ Ring    base / sizeDwords / head / tail / wrap 检查   │
 │ Mmio    256 dword 寄存器块（只经 Read/Write 访问）    │
 │ Doorbell 写 0x00 段 → Cp::OnDoorbell()               │
 │ Cp      FSM: Idle→Fetch→Decode→Dispatch→Fence→Idle   │
 │ Fence   64 bit 单调值 + 完成位（0x08 段）              │
 │ Device  生命周期 / 能力 / 单元持有者（私有成员）        │
 └──────────────────────────────────────────────────────┘
        │ 派发目标（本章只有 CP 内部处理）
        ▼
   NOP / STREAM_HEADER / FENCE  ──► 完成写值
```

---

## 3. 输入输出契约

### 3.1 设备与环描述（`hw_sim/core/device.h`）

```cpp
struct DeviceDesc {
    u32  rtWidth;      // 只保存尺寸用于环/资源校验；RT_SIZE 唯一由 CLEAR 锁存（Q-35）
    u32  rtHeight;     // 同上，不写 MMIO
    u32  ringSizeDwords;   // 必须为 2 的幂，>= 64
    u32  streamVersion;    // 期望的命令流版本，当前 1
};

struct RingDesc {
    GpuVAddr baseVa;      // 环内存起始（第 3 章起由内存子系统分配）
    u32      sizeDwords;  // 2 的幂
    u64      initialFence;
};

class IDevice {
public:
    virtual Result Configure(const DeviceDesc&) = 0;
    virtual Result CreateRing(const RingDesc&, u32* outRingId) = 0;
    virtual Result WriteMmio(u32 reg, u32 value) = 0;
    virtual Result ReadMmio(u32 reg, u32* outValue) const = 0;
    virtual Result Shutdown() = 0;
};
```

### 3.2 薄提交门面（`hw_sim/submit/facade.h`）

```cpp
namespace hw::submit {
struct DrawDesc {                 // 本章只使用 rtWidth/rtHeight/clearColor，其余字段冻结待用
    const void* vertexData;  u32 vertexCount;  u32 stride;
    const void* indexData;   u32 indexCount;
    const void* textureData; u32 texWidth; u32 texHeight;
    u32 rtWidth; u32 rtHeight; u32 clearColor;
};
Result Submit(const DrawDesc& desc);  // 同步执行完命令
Result Wait();                        // fence 值断言
u64    LastSubmittedFence();          // 诊断用，不参与判据
}
```

### 3.3 MMIO 寄存器（本章只实现 0x00~0x3F 段）

| 偏移 | 名称 | 读写 | 语义 |
|---|---|---|---|
| 0x00 | `RING_BASE_LO` | RW | 环基址低 32 bit（`GpuVAddr`） |
| 0x01 | `RING_BASE_HI` | RW | 环基址高 32 bit |
| 0x02 | `RING_SIZE` | RW | 环大小（dword，2 的幂） |
| 0x03 | `RING_HEAD` | RO | CP 已消费位置（dword 索引） |
| 0x04 | `RING_TAIL` | RW | 驱动写入位置（dword 索引） |
| 0x05 | `DOORBELL` | WO | 写任意值触发 CP 取指 |
| 0x08 | `FENCE_VALUE_LO` | RO | 当前完成值低 32 bit |
| 0x09 | `FENCE_VALUE_HI` | RO | 当前完成值高 32 bit |
| 0x0A | `FENCE_STATUS` | RO | bit0 = 完成位；bit1 = 忙碌 |
| 0x10 | `CP_STATUS` | RO | CP 状态机当前状态（见 §4） |
| 0x20 | `ERROR_CODE` | RO | 最近一次 `Result` 数值 |
| 0x21 | `ERROR_PKT_TYPE` | RO | 出错包类型 |
| 0x22 | `ERROR_DWORD_OFFSET` | RO | 出错 dword 偏移 |
| 0x30 | `RT_SIZE` | RW | 宽低 16 bit、高高 16 bit（由 `CLEAR` 锁存，本章只写入） |
| 0x38 | `STREAM_VERSION_ECHO` | RO | 最近接受的命令流版本（仅回显） |

约束：
- MMIO 偏移一律为 dword 索引（Q-31）。
- 单元内部成员全部私有，外部只能经 MMIO 读写（ADR-037）。
- 段内偏移由本章定义并登记；新增偏移必须在 `../design/command-mmio.md` §4 对应段登记。
- `0x08~0x0A` 是 fence 的最小载体：CP 写值即完成通知，不引入中断载荷。

### 3.4 错误语义

| 场景 | 返回 | 记录 |
|---|---|---|
| 环大小非 2 的幂或 < 64 | `InvalidArgument` | `{unit=Graphics3D, pkt=0xFFFFFFFF, dwordOffset=0}` |
| 包 `count==0` 或 > 4095 | `InvalidPacket` | `{unit, packetType, dwordOffset}` |
| 首包不是 `STREAM_HEADER` | `InvalidPacket` | `dwordOffset=0` |
| 版本不匹配 | `InvalidPacket` | `dwordOffset` 指向版本字段 |
| `head/tail` 越界或写入越界 | `InvalidArgument` | 带当前 head/tail |
| 包跨环尾 | 由写入侧 `NOP` 填充消除；若仍出现则 `InvalidPacket` | `dwordOffset` 指向包起始 |

---

## 4. 内部数据流与状态机

### 4.1 CP 状态机（本章最小版）

```
        ┌────────┐  tail != head   ┌────────────┐
        │  Idle  │────────────────►│ FetchHeader│
        └────────┘   doorbell      └─────┬──────┘
             ▲                            │ 读 1 dword，解析 type/count
             │ 环空                        ▼
        ┌────┴─────┐   非首包且未见 HEADER ┌─────────┐
        │  Halted  │◄──────────────────────│ Validate│──count/version 非法→错误
        └──────────┘                       └────┬────┘
                                                │ 合法
                                                ▼
                                          ┌──────────┐
                                          │ Dispatch │ 按 type：NOP/HEADER 内部处理
                                          └────┬─────┘ FENCE → 写 fence
                                               ▼
                                          ┌──────────┐
                                          │AdvanceHead│ head += count（模环大小）
                                          └────┬─────┘
                                               └──► Idle
```

规则：
- 一个 `Pump()` 推进一次状态转移，便于单步观测；提交路径同步循环 `Pump` 直到 `IsIdle()`。
- `head` 是"已消费"的 dword 索引，按环大小取模；`tail` 由门面在写入后更新。
- 环空判定统一为 `head == tail`。

### 4.2 写入与填充

```
门面计算命令流总 dword → 若 tail 到环尾剩余 < 总长：
    写 NOP 包填满到环尾（count = 剩余 dword）→ tail 回绕到 0
再写入各包（每包 count 含头）→ 更新 RING_TAIL → 写 DOORBELL
```

### 4.3 fence 语义

- 每个 `FENCE` 包载荷为 64 bit 目标值；CP 写 `FENCE_VALUE_LO/HI` 并置完成位，随后清忙碌位。
- `Wait()` 只做断言：`FENCE_VALUE >= LastSubmittedFence` 且完成位置位，否则返回 `Internal` 并记录。
- 完成计数以"已写 fence 的包数"为准，本章规定**一次提交恰产生一个 fence**。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T2.1 | ① 契约 | `DeviceDesc`/`RingDesc`/`IDevice`/薄门面 `DrawDesc`+`Submit/Wait` 签名冻结；MMIO 偏移表与错误语义表落头文件与断言 |
| T2.2 | ② 主体 | MMIO 寄存器块实现：分配、按偏移读写、越界偏移拒绝；`ERROR_*` 与 `CP_STATUS` 只读语义 |
| T2.3 | ② 主体 | Ring：初始化、`head/tail` 推进与取模、空/满判定、`NOP` 填充跨尾规则 |
| T2.4 | ② 主体 | CP 状态机：Fetch → Validate（首包/版本/count）→ Dispatch（`NOP`/`STREAM_HEADER` 内部处理）→ AdvanceHead；`Pump` 单步与 `IsIdle` |
| T2.5 | ② 主体 | Fence：64 bit 值写入、完成位/忙碌位、`Wait` 断言、完成计数 |
| T2.6 | ② 主体 | 非法输入路径：非法 `count`、非首包 `STREAM_HEADER`、版本不匹配、跨尾残留、越界 `tail`、环大小非法，全部产出 `ErrorRecord` 并写 MMIO 错误码 |
| T2.7 | ③ 接入 | 薄门面接入：`Submit` 组装 `STREAM_HEADER`+`NOP`+`FENCE` 写入环并敲门铃，`Wait` 断言完成；`scene_e1` 以空命令流跑通 |
| T2.8 | ④ 验收 | A2：E1 提交子集全绿 + 本章专有判据（命令不跨环尾、完成计数 = 提交数） |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D2.1 | 头文件可独立编译；MMIO 偏移表与 §3.3 逐行一致（脚本比对）；`DrawDesc` 字段顺序与字节偏移有 `static_assert` | L1 契约 |
| D2.2 | 对 0x00~0x3F 内任一登记偏移读写往返一致；越界偏移（≥0x100）返回 `InvalidArgument`；`0x40~0xFF` 保留段（合法窗口内未实现）读写返回 `NotSupported` 且不改写任何状态；`RING_HEAD`/`ERROR_*` 写入被拒绝且值不变 | L1 契约 |
| D2.3 | 以 `sizeDwords=64` 连续写入 100 个包后 `head/tail` 始终在 `[0,64)`；任意包不跨环尾（逐包检查 `offset+count ≤ size`）；空环时 `Pump` 不发生状态变化 | L1 契约 |
| D2.4 | 对 `[HEADER, NOP, FENCE]` 序列，状态转移轨迹为 `Idle→Fetch→Validate→Dispatch→Advance→Idle`（`CP_STATUS` 逐次匹配）；`count=0`、`count=4096`、首包非 HEADER 三种输入均被拒且错误偏移精确 | L1 契约 |
| D2.5 | 提交 N 次（N=1、8、64）后 `FENCE_VALUE` 严格等于最后一次提交的 fence 值且单调递增；完成位在 `Wait` 前可能为 0、在 `Wait` 后必为 1；完成计数 == N | L1 契约 |
| D2.6 | 六类非法输入各触发一次（环大小非 2 幂/<64、非法 `count`、非首包 `STREAM_HEADER`、版本不匹配、跨尾残留、越界 `tail`）：返回码、`ERROR_CODE/ERROR_PKT_TYPE/ERROR_DWORD_OFFSET` 三寄存器与 `GD_LOG` 行三者一致；不崩溃、不静默继续 | L1 契约 |
| D2.7 | `scene_e1` 以 `rt_size=256x256` 提交 1 次空命令流：`Submit` 返回 `Ok`、`Wait` 返回 `Ok`、`LastSubmittedFence()==1`；场景只 include 门面头文件（编译期检查 include 图） | L1 契约 + A2 |
| D2.8 | A2 全绿：E1 提交子集（读回缓冲尚未引入，本章只判提交与完成）+ 专有判据；连续两次运行 fence 序列与错误记录完全一致 | L1 契约 |

---

## 7. 场景与判据（A2）

**绑定场景：E1 `scene_e1_clear_present`（清屏上屏）——本章只跑"提交 → 完成"子集。**

| 项 | 内容 |
|---|---|
| 输入 | 背景色 `0.25` 灰（`0xFF3F3F3F`）、`rt_size=256x256`、单次提交 |
| 本章产物 | 环内存快照、`FENCE_VALUE`、`CP_STATUS` 轨迹、错误记录（无错误时为空） |
| 主判据 | 提交返回 `Ok`；`Wait` 返回 `Ok`；fence 值 = 提交序号 |
| 本章专有判据 | ① 命令不跨环尾：任意 `count` 下 `packetStart + count ≤ ringSize`；② 完成计数 = 提交数 |
| 回归项 | 无（首章实现） |
| 后续补齐 | 完整 E1（全图逐字节等于背景色、读回缓冲 = 显存、双缓冲前后帧可区分）依赖第 3 章内存与第 8 章帧输出，在 A8 收口；本章不得声称 E1 已全绿，只记录子集通过 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 单线程顺序消费；`head/tail` 推进与包派发顺序固定；包序不受任何容器遍历影响 |
| 时间无关 | `Wait` 是值断言，不轮询时钟；无超时参数 |
| 可观测 | `CP_STATUS`、`FENCE_VALUE/STATUS`、`ERROR_*` 三组 MMIO 只读寄存器；`--trace=cp` 打印状态转移 |
| 契约影响 | 新增 MMIO 偏移必须在 `command-mmio.md` §4 登记；`DrawDesc` 字段只许追加 |
| 简化登记 | 单环不回绕压测、无中断、无 watchdog（§9） |
| 计算兼容 | CP 按包类型分派，本章虽只有 `NOP/HEADER/FENCE`，但派发表结构预留 `DISPATCH` 分支 |
| 配置与回退 | `rt_size` 只用于环/资源尺寸校验（`RT_SIZE` 唯一由 `CLEAR` 锁存）；`exec_mode=parallel` 报"策略 B 未实现" |
| 调试钩子 | `--trace=cp` 输出每步 `{state, head, tail, pktType, dwordOffset}`，仅供人工 |
| trace 契约 | 字段不承诺稳定、不参与判据、CI 不解析 |
| 契约变更 | 包头布局、MMIO 分段、fence 语义为冻结项；变更需 ADR 并回归 E0/E1 |
| 校验归属 | 本章全部 L1 契约断言，零容差、阻断；E1 全图比对推迟到 A8 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 提交 | 单队列、单环、同步完成、无中断 | 10（KMD/FW + 中断）、32（多队列） |
| 环容量 | 容量足够，只做单元测试级回绕，不做持续压测 | 10（S1 用 N > 环容量压测） |
| 同步 | fence 只是 MMIO 值断言，无 semaphore/event 对象 | 32、33 |
| 异常 | 只记录并中止当前提交，不复位、不恢复 | 36（复位路径） |
| 设备 | 无电源/时钟/热模型，无 watchdog | 36 |

---

## 10. 待定项

无。若后续需要引入"环优先级"或"doorbell 批量合并"，先以 ADR 登记再改本章 §3.3 与 §4.2。
