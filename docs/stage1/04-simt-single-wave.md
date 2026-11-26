# 第 4 章 SIMT 单波

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章建立全项目的执行模型：warp（波）宽度 32、VGPR/SGPR 固定分区、活动掩码/谓词、无分歧线性执行、内建 ID，以及着色核的唯一接口 `IShaderArray::Dispatch`。着色器程序在本章以"内存内指令结构"表示（第 5 章把它规范化为二进制编码），执行语义与寄存器终态是本章验收对象。
> 与本章相关的设计说明：`../architecture-design.md` §3.1（`IShaderArray::Dispatch` 契约）、§3.3（`IEngine.Push` 携带着色核）；`../design/execution-parallel.md` §1~§3（时间无关与确定性）；`../design/isa-abi.md` §3（VGPR/SGPR 分区、LaneMask、控制流栈）；`../design/isa-abi.md` §1（内存内指令结构 → 第 5 章编码过渡）。
> 落地模块：`hw_sim/shader/wave`（波状态与已构造 Instr 的执行）和 `apps/scene_e2_triangle`（寄存器终态）。第 5 章从 `hw_sim/isa` 接入二进制解码。

---

## 1. 目标与非目标

### 1.1 目标

1. 冻结着色核接口 `IShaderArray::Dispatch` 与 `DispatchArgs`，作为 VS/PS 唯一执行入口（第 6 章起被固定功能各阶段调用）。
2. 波的创建与终止：32 lane 的同构执行、`LaneMask` 活动集合的维护（全真、取反与全集、按 lane 位操作）。
3. VGPR/SGPR 分区落地：每 lane 独立 VGPR bank（`v0..v127`）、波内一致 SGPR（`s0..s31`）；固定分区不被临时量覆盖。
4. 无分歧线性执行：按 pc 顺序执行，不存在分支指令，活动掩码在拍间不变（本章），逐 lane 完成指令语义。
5. 内建 ID：`GET_THREAD_ID` 等按 `DispatchArgs` 提供；派发参数按 CU 语义复用，不随 warp 数按栈放大。
6. 越界访存定位：程序访问越界地址必须报出地址并记录，不静默返回 0。

### 1.2 非目标

- 不做分歧/控制流/重收敛（第 15 章）；本章程序内禁止出现分支。
- 不做波调度、多波、CU 多 SIMD（第 15、31 章）——每次 `Dispatch` 恰一整个波。
- 不做 LDS/shared/原子/屏障（第 29、30 章）。
- 不做正式二进制编码与汇编器（第 5 章）；但本章留下的"指令结构"须能被第 5 章编码器无损双向映射，避免返工。
- 不做 ILP/超线程/延迟隐藏（第 15、21 章）。

---

## 2. 单元框图

```
 IEngine / 门面（第 6 章起由 IA 等阶段调用）
        │ Dispatch(stage, args, program, laneMask)
        ▼
┌──────────────────── hw_sim/shader/wave ─────────────────────┐
│ Wave          32 lane；每 lane：pc、谓词、VGPR bank          │
│ Registers     VGPR v0..v127（每 lane 独立）；SGPR s0..s31    │
│ LaneMask      u32 活动集合：全真 / 与 / 或 / 取反&全集        │
│ ExeUnit       译码"内存内指令" → 按 lane 执行语义            │
│ BuiltinIds    threadId / vertexId / instanceId / pixelPos    │
└───────────────────────────────┬─────────────────────────────┘
                                │ 结果：寄存器终态 + 写回请求（内存系 统）
                                ▼
                     VS 输出区 / PS 输出区（写回由调用方决定）
```

---

## 3. 输入输出契约

### 3.1 着色核接口（`hw_sim/shader/wave/dispatch.h`，冻结于本章）

```cpp
namespace shader {

using common::Stage;   // 唯一定义在 common/types.h，不得重复定义（Q-33）

struct DispatchArgs {
    Stage       stage;
    u32         threadIdBase;   // 本章恒 0；warp 内 lane 线性对应 threadId
    u32         vertexIdBase;   // 非索引 VS 用：首顶点全局 id
    u32         instanceId;
    i32         pixelX, pixelY; // PS 用；VS 用 0
    const u8*   program;        // 内存内指令结构（第 5 章改为编码后入口）
    u32         programBytes;
    const u32*  uniformData;    // uniform 标量区（v[56..67] 之外，可选）
    u32         uniformDwords;
};

class IShaderArray {
public:
    virtual ~IShaderArray() = default;
    // 同步执行一个 32-lane 波；结果写回调用方提供的寄存器视图。
    virtual Result Dispatch(const DispatchArgs&, const LaneMask& active,
                            const SymbolBindings&, ShaderRegisters* out) = 0;
    virtual u32  WaveSize() const = 0;                 // 恒 32
    virtual void PushEvent(/* 软观测事件 */ const EventSample&) = 0;
};

} // namespace shader
```

约束：

- `Dispatch` 同步执行完一整个波；`active` 为本次的初始活动集合，执行期间不变（无分歧前置）。
- `ShaderRegisters` 按 `../design/isa-abi.md` §3 的固定分区暴露：输入属性 `v[0..15]`、VS 输出 `v[16..27]`、PS 输出 RT0 `v[36..39]`/深度 `v[40..43]`、uniform `v[56..67]`、临时量 `v[76..]`。
- `SymbolBindings` 只是分区的机器编排，不得改变分区（ADR-034 硬规则）。
- 非着色引擎（Copy 等）**不实现**本接口，`sa == nullptr` 时引擎不得调用。

### 3.2 寄存器与掩码（`common/shader_abi.h`）

| 符号 | 值 | 约束 |
|---|---|---|
| `WARP_SIZE` | 32 | 固定；`WaveSize()` 恒 32 |
| `kLaneMaskAll` | `0xFFFFFFFF` | 取反结果必须 `& kLaneMaskAll` |
| `kVGprCount` | 128 | 每 lane 独立 |
| `kSGprCount` | 32 | 波内一致 |
| 输入属性区 | `v[0..15]` | IA 装配（第 6 章） |
| VS 输出区 | `v[16..27]` | 光栅插值源（第 6 章） |
| HS 因子区 | `v[28..35]` | 曲面细分因子（第 28 章消费） |
| PS 输出区 | RT0 `v[36..39]`、深度 `v[40..43]` | OM 消费（第 6 章） |
| PS 输出区 | RT1~3 `v[44..55]` | OM 消费（第 6 章多 RT） |
| uniform 区 | `v[56..67]` | 只放小体积高频标量 |
| 距离区 | `v[68..75]` | 预留给 ClipDistance / CullDistance（第 24 章启用），本章分配器禁止占用 |
| 临时量起点 | `v[76]` | 寄存器分配不得侵占上述固定区 |

### 3.3 内存内指令结构（本章专用，第 5 章可无损映射）

```cpp
enum class Op : u8 {
    Mov, Add, Sub, Mul, Div, And, Or, Xor, Not, Cmp,        // ALU/谓词，10 条初值
    GetThreadId, GetVertexId, GetInstanceId, GetPixelPos,   // 内建 ID
    Exit,
};
struct Instr {
    Op   op;       // 语义种类
    u8   dst;      // 目标 VGPR 或谓词寄存器
    u8   srcA, srcB; // 源 VGPR（0xFF = 立即数哨兵）
    f32  imm;      // srcA==0xFF 时使用（按打包规则见 §3.4）
};
```

> 注：内存指令（`LD/ST`、纹理 `SAMPLE`）不在本章落地，见 §3.4；保持 `Op` 可扩展即可。

### 3.4 越界与未定义行为（本章范围）

- 程序访问 `VDst > kVGprCount-1`、`Vsrc > kVGprCount-1`、`pc` 越过程序末尾未遇 `Exit`，一律返回 `InvalidIsa` 并记录 `{unit=Graphics3D, va=程序基址+偏移}`。
- 未支持 opcode（`SAMPLE` 等）返回 `NotSupported`，走能力四档语义，不静默当作 NOP。
- **派发参数按 CU 复用**：`DispatchArgs` 不含按 warp 序号放大的栈/参数副本；`threadIdBase=0`，波内 id 由 lane 决定。违反此规则加倍复制参数视为架构违规。

---

## 4. 内部数据流

### 4.1 单波执行轨迹

```
Dispatch(stage, args, active, bindings)
  ├─ 为 32 lane 创建寄存器视图（VGPR 清零、SGPR ← uniformData、输入区 ← bindings/args）
  ├─ pc = 0；LaneMask = active
  ├─ 循环：
  │    instr = Decode(program[pc])          // 语义结构
  │    if op == Exit: break
  │    对该 instr 检查 dst/src 越界 → 越界则 ErrorRecord 并中止
  │    对每个 active lane 执行 instr 语义（ALU 按 f32；谓词位操作按 u32）
  │    pc += 1
  ├─ 写出 ShaderRegisters（固定分区快照）
  └─ PushEvent({stage, 执行的 lane·拍数总和})   // 软观测
```

- 所有算术按 `f32` 单精度 IEEE-754 语义；不引入舍入模式配置，默认 round-to-nearest-even，跨编译选项一致（确定性规则）。
- 谓词：`Cmp` 把结果写谓词寄存器（每 lane 1 bit），不写 VGPR；`Mov` 的谓词版本（`Mov(Cond)`）本章不引入。
- 本章 `GetThreadId = lane`，写入指定 VGPR 的每 lane 值。

### 4.2 寄存器间搬运（Mov）

本章不引入内存 `LD/ST`；VS/PS 的数据搬运靠"寄存器区段映射"完成（IA 直接写 `v[0..15]`、VS 结果直接读 `v[16..27]`）。区段复制由 `Mov` 承担，`Mov` 的 src 可指向任意固定区段（Q-37、Q-73），与内存无关；真正的 `LD/ST` 在第 5 章 opcode 区段补齐。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T4.1 | ① 契约 | `Stage`、`DispatchArgs`、`IShaderArray`、`ShaderRegisters` 分区、`LaneMask`/`WARP_SIZE` 常量落头文件；`Stage::Compute` 一并冻结（暂不消费） |
| T4.2 | ② 主体 | Wave 创建/销毁：32 lane 寄存器视图、VGPR 清零、初始掩码、终止后寄存器快照导出 |
| T4.3 | ② 主体 | LaneMask 运算集：`& / \| / ^ / ~&kAll`、`IsActive(lane)`、迭代活跃 lane（升序固定） |
| T4.4 | ② 主体 | SGPR 与 uniform 通路：uniform 写入 SGPR/`v[56..67]`，标量读不按 lane 分化 |
| T4.5 | ② 主体 | 指令语义执行器：`Mov/Add/Sub/Mul/Div/And/Or/Xor/Not/Cmp` 按 lane 执行，`Div` 除零语义确定（返回 `+Inf/NaN` 按 IEEE，不崩溃） |
| T4.6 | ② 主体 | 内建 ID：`GetThreadId/GetVertexId/GetInstanceId/GetPixelPos`，`vertexIdBase`/`instanceId` 从 `DispatchArgs` 注入 |
| T4.7 | ② 主体 | 越界与未支持路径：`dst/src` 越界、`pc` 越程序、未支持 opcode 的错误记录与 `GD_LOG` |
| T4.8 | ③ 接入 | `IShaderArray` 经门面暴露：`scene_e2` 用内存内指令写一个"输入→输出区搬运 + 逐 lane 写唯一值"样例，验证寄存器终态 |
| T4.9 | ④ 验收 | A4：E2 寄存器终态子集 + 专有判据（终态全等、无分支、越界报地址、参数按 CU 复用） |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D4.1 | 头文件独立编译；`Stage` 取值含 `Compute` 且无重复；`ShaderRegisters` 分区表与 `../design/isa-abi.md` §3 逐项一致（脚本比对）；`WaveSize()==32` | L1 契约 |
| D4.2 | 10 次 `Dispatch` 后寄存器视图单调复用、无跨波残留（前波终止值不影响后波清零）；终止时 `active` 外 lane 的 VGPR 为初始化值 | L1 契约 |
| D4.3 | `(~m)&kLaneMaskAll` 与按位取反在 `m=0/0xFFFFFFFF/0x55555555` 三种样本下结果相等；活跃 lane 迭代顺序与 lane 升序一致 | L1 契约 |
| D4.4 | 同一 uniform 数据对不同 lane 读取一致（标量通路）；写入 `v[56..67]` 后 `slot<56` 或 `>67` 不被动 | L1 契约 |
| D4.5 | 对 10 条 ALU 各构造"全 lane 同输入"与"逐 lane 不同输入"两组用例：结果与 CPU 逐 lane 手工计算逐字节相等（f32 位级相等）；`Div(1,0)` 得 `+Inf`、`Div(0,0)` 得 `NaN`，不崩溃 | L1 契约 |
| D4.6 | `GetThreadId` 写入 `v[76]` 后 `v[76][lane]==lane`；`GetVertexId` 在 `vertexIdBase=100` 时 `v[76][lane] == 100 + lane`（`vertexIdBase` 注入正确、lane 与线 id 一一对应） | L1 契约 |
| D4.7 | 四类非法程序各触发一次（`dst` 越界、`src` 越界、`pc` 越程序、未支持 opcode）：`dst/src` 越界与 `pc` 越程序返回 `InvalidIsa`、未支持 opcode 返回 `NotSupported`；`GD_LOG` 含地址、`ErrorRecord.{unit,pkt,va}` 定位到出错指令索引 | L1 契约 |
| D4.8 | `scene_e2` 的着色样例：32 lane 全部执行、寄存器终态与参考值全等；门面只依赖 `IShaderArray` 接口头 | L1 契约 + A4 |
| D4.9 | A4 全绿：终态断言 N 条通过（N 由用例数定义）；无分支指令出现（解码器拒绝分支 opcode）；越界路径有对应测试用例 | L1 契约 |

---

## 7. 场景与判据（A4）

**绑定场景：E2 `scene_e2_triangle`（渐变三角形）——本章只验"着色寄存器终态"子集。**

| 项 | 内容 |
|---|---|
| 输入 | 3 顶点 `RenderVertex`（stride 32：`float4 position` + `float4 color`）+ 一个 VS 程序（内存内指令结构） |
| 本章产物 | VS 输出区寄存器终态（每顶点/每 lane） |
| 主判据 | 3 个顶点的 VS 输出区与参考值逐字节（f32）相等 |
| 本章专有判据 | ① 寄存器终态全等；② 程序内无分支指令；③ 越界访存报出地址；④ 派发参数按 CU 复用，不随 warp 数按栈放大（`DispatchArgs` 内存占用与波内 lane 无关） |
| 回归项 | R04：E0（D1.5→D5.8 后并入）；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | E2 的"全图逐字节匹配固定基线""退化三角形不写像素"依赖第 6 章光栅；A6 收口 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | ALU 全程 f32 IEEE-754 RN；活跃 lane 迭代顺序固定（升序）；`Dispatch` 无时间输入 |
| 时间无关 | 无墙钟；无 per-lane 随机；`frame_index` 不进入 `DispatchArgs` |
| 可观测 | `PushEvent({stage, 执行 lane·拍})` 软观测；暂停时寄存器快照可导出；`--trace=wave` 打印每拍活跃掩码 |
| 契约影响 | `IShaderArray::Dispatch` 签名冻结；`DispatchArgs` 字段只许追加 |
| 简化登记 | 单波、单 SIMD、无分支（§9） |
| 计算兼容 | 线程 ID 空间由第 31/37 章扩展（`blockIdx/threadIdx` 三维映射），本章不新增字段；`Stage::Compute` 已冻结，第 31/37 章（stage3 38~42，原 35.5）直接消费同一执行核 |
| 配置与回退 | 无独立开关；`exec_mode=parallel` 报未实现 |
| 调试钩子 | `--trace=wave`：`{拍, laneMask, pc, op}` 单行，仅人工 |
| trace 契约 | 字段不承诺稳定、不进入判据 |
| 契约变更 | ABI 分区、`DispatchArgs` 布局、算术规则为冻结项；变更需 ADR 并回归 E0/E2 |
| 校验归属 | 全部 L1 契约断言（寄存器终态逐字节）；无探针点（探针模型第 7 章起对像素生效） |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 执行 | 单波、单 SIMD、组内线性执行 | 15（分歧）、31（CU/SE 阵列） |
| 分支 | 着色器内禁止分支（无分歧概念） | 15、16 |
| 调度 | 无波调度器，`Dispatch` 即一个波同步完成 | 15、31 |
| 片上存储 | 无 LDS/shared/原子/屏障 | 29、30 |
| 延迟隐藏 | 无挂起/唤醒，访存同步阻塞 | 15、21 |
| 程序表示 | 内存内指令结构 | 5（编码化） |

---

## 10. 待定项

- **Q-73**：删除 `LoadReg/StoreReg`；区段复制由 `Mov` 承担（`Mov` 的 src 可指向任意固定区段），减少指令面（与 Q-37 同案）。
- **Q-74**：除零/NaN 语义维持 IEEE-754（`0/0→NaN`、`1/0→+Inf`）；如需改为"断言 + 报告"，须先登记。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A4 和适用的 R4 后提交。分配核对项为lane mask、VGPR ABI 与保留区，新增或改动按 [编码与位段校验表](../design/allocation-audit.md)登记。

```text
feat(gpu-ch04): execute one SIMT wave

- 按执行掩码调度单 wave 的取指、寄存器读写与退出。
- 用 E2 着色输入检查寄存器终态、不活跃 lane 与确定错误。

Tasks: T4.1–T4.9
Validation: <实际执行的命令、用例和结果>
```
