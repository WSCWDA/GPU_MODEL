# 第 5 章 最小 ISA

> 本章把第 4 章的"内存内指令结构"规范化为 64 bit 定长二进制编码，并交付汇编器/反汇编器与指令语义表。约 20 条指令、线性编码（无压缩）；标量/向量分区位预留，本章只实现向量指令。E0 的"汇编往返"判据在此补全，E2 的着色样例改用编码投递。
> 与本章相关的设计说明：`../design/isa-abi.md` §1~§2、§3（64 bit 编码、opcode 区段、ABI 分区）；`../architecture-design.md` §3.1（`IShaderArray`）；第 4 章（执行器消费语义）。
> 落地模块：`common/include/common/isa.h`、`hw_sim/shader/isa`（解码/语义表）、`tools/isa_asm`、`tools/isa_disasm`、`apps/scene_e0_selftest`（往返子集补全）、`apps/scene_e2_triangle`。

---

## 1. 目标与非目标

### 1.1 目标

1. 冻结 64 bit 编码布局：`[62:56] opcode（7 bit）、[63] SFLAG、[55:48] dst、[47:40] src0、[39:32] src1、[31:0] imm32`。
2. 登记第 5 章指令子集（约 20 条），按 `../design/isa-abi.md` §2 的区段取号，**从预留区取号前先登记**。
3. 汇编器 `isa_asm`：文本/汇编程序 → 二进制dword 数组；支持 `vN/sN` 寄存器写、立即数哨兵 `0xFF`、立即数打包、相对分支偏移字段（本章禁止分支指令用于执行）。
4. 反汇编器 `isa_disasm`：二进制 → 规范化文本；汇编 → 二进制 → 反汇编 → 再汇编必须得到相同二进制（E0 活验收）。
5. 指令语义表：20 条指令逐一映射到第 4 章执行器（`decode → Instr` 结构），无解码失败。
6. smoke 用例集：每条指令至少"合法一次 + 边界一次"，覆盖 `-Wall -Wextra` 零警告构建。

### 1.2 非目标

- 不做压缩编码、变长指令、宏操作（第 16 章）。
- 不做完整 opcode 全集与计算指令（subgroup/atomic/barrier 等，第 16/29/30 章）。
- 不做标量指令执行（ADR-034 的 `s` 通路只在 ABI/编码位冻结，本章 `src/dst` 全为 `v` 编号；标量位保留并置 0，第 16 章启用）。
- 不做编译器（第 17~19 章）；本章只写手写汇编。
- 不做分支/循环/调用指令的执行（第 15 章）；本章 BR/BRC 等控制流指令**不列入**指令子集，解码遇未知 opcode 返回 `InvalidIsa`。

---

## 2. 单元框图

```
   文本汇编（手写 VS/PS）
        │ isa_asm（tools/）
        ▼
   二进制代码（dword 数组，放 GpuVAddr）
        │
   ┌────┴───────────────────────────────┐
   │ common/isa.h  编码布局 + opcode 表   │
   │ isa::Encode(Inst)→u64 / isa::Decode │
   │ tools/isa_disasm → 规范化文本         │
   │ shader/isa 语义表 Decode→Instr→执行器 │
   └──────────────────────────────────────┘
        │ ¼ 第 4 章 Wave 执行器执行
        ▼
   寄存器终态
   E0：往返断言；E2：VS 样例执行断言
```

---

## 3. 输入输出契约

### 3.1 编码布局（`common/isa.h`，本章冻结）

```
[62:56] opcode  7 bit  —— 区段按 ../design/isa-abi.md §2（已按 ADR-043 落 7 bit 区段表，原"0x80~0xFF"内容压缩进 0x60~0x7F：曲面/几何 0x60~0x6F、subgroup 0x70~0x7D、0x7E=UNDEFINED、0x7F=EXIT；具体指令号第 16 章细化）
[55:48] dstReg  8 bit  —— 统一编号字段；本章取 VGPR 号 0..127
[47:40] src0    8 bit   —— 0xFF = 立即数哨兵（MOV 写立即数）
[39:32] src1    8 bit
[31: 0] imm32   32 bit —— 立即数（f32 位型）或分支偏移（本章仅保留字段）
[63]    SFLAG   —— 标量/向量区分位（ADR-034），本章固定 0（向量）；第 16 章启用
```

- 分支偏移为有符号 32 bit、相对当前 `pc`；本章无分支指令，字段仅供 LOAD/STORE 的地址偏移复用。
- `MOV` 的 `src0 == 0xFF` 表示"写入 imm32"。
- 编码为唯一权威：汇编器、反汇编器、解释器、编译器共用 `common/isa.h`。

### 3.2 第 5 章指令子集（20 条）

| Opcode | 助记符 | 类别 | 语义（每 lane，活动 lane 内） |
|---|---|---|---|
| 0x00 | `NOP` | 空操作 | 无操作 |
| 0x10 | `MOV` | ALU | `v[ dst ] = srcB(src0==0xFF ? imm32 : v[ src0 ])` |
| 0x11 | `ADD` | ALU | `v[dst] = v[src0] + v[src1]`（f32） |
| 0x12 | `SUB` | ALU | `v[dst] = v[src0] - v[src1]` |
| 0x13 | `MUL` | ALU | `v[dst] = v[src0] * v[src1]` |
| 0x14 | `DIV` | ALU | `v[dst] = v[src0] / v[src1]`（IEEE；除零见第 4 章） |
| 0x15 | `AND` | ALU | 按位且（按 u32 解释两位） |
| 0x16 | `OR` | ALU | 按位或 |
| 0x17 | `XOR` | ALU | 按位异或 |
| 0x18 | `NOT` | ALU | 按位非 |
| 0x19 | `CMP` | 谓词 | 写谓词寄存器：`pred = v[src0] OP v[src1]`（OP 由 imm 低 4bit 编码 `EQ/NE/LT/LE/GT/GE`） |
| 0x1A | `MAD` | ALU | `v[dst] = v[src0] * v[src1] + imm32` |
| 0x1B | `SETP` | 谓词 | 谓词类型为 u64 值 + mask/cond 比较，bool 为特例（Q-70）；本章最小：SETP 恒真置位，供分支接管前占位 |
| 0x20 | `LD` | 内存 | `v[dst] = mem32(常数区基址 + src0 索引·4)`：基址 = 随包下发的常数区基址（stateId=4），src0 为按 lane 索引 |
| 0x21 | `ST` | 内存 | `mem32(基址+src0 索引) = v[src1]` |
| 0x50 | `GET_THREAD_ID` | 内建 | `v[dst] = lane` |
| 0x51 | `GET_VERTEX_ID` | 内建 | `v[dst] = vertexIdBase + lane（由 DispatchArgs 注入）` |
| 0x52 | `GET_INSTANCE_ID` | 内建 | `v[dst] = instanceId` |
| 0x53 | `GET_PIXEL_POS` | 内建 | `v[dst] = {pixelX, pixelY, 0, 1}`（4 槽写） |
| 0x7F | `EXIT` | 控制流 | 终止当前 lane；全部活动 lane EXIT 后波结束 |

> 说明：`LD/ST` 的访存必须经 `IMemorySubsystem`（第 3 章唯一通道），地址 = `常数基址 + src0 的按 lane 索引·4`；初级用 stateId=4 承载常量区基址（第 17/18 章正式落地，Q-36）。边界由宿主席位返回 `NotMapped`。`SETP/CMP` 只写谓词，不写 VGPR。

### 3.3 汇编语言最小语法

```
<label> : 可选，除引用留着给第 15 章
<line>  =  <mnem>  <opnd>[, <opnd>...]  [; <comment>]
opnd    =  v<0..127> | s<0..31>（本章仅 v 合法）| i<imm>（立即数）| i32:f<hex>
指令行以单一空格分隔字段；`;` 后为行注释（ASCII）。
```

- 立即数哨兵：汇编器把"立即数操作数"翻译为 `src0=0xFF + imm32` 组合；`f32` 立即数以 `f<hex>`（4 字节十六进制位型）书写以避免舍入歧义，`i<dec>` 整数按位型直接打包。

### 3.4 往返一致性（E0 主判据）

```
asm(规范化文本) ──isa_asm──► bin1
bin1           ──isa_disasm─► 规范化文本
规范化文本       ──isa_asm──► bin2
断言 bin1 == bin2（逐 64 bit 字）
```

规范化文本是反汇编器的唯一输出形式；文本中不允许出现字符别名层面就分叉的写法（寄存器一律小写 `vN`、立即数按既定格式）。

---

## 4. 内部数据流

### 4.1 汇编流水

```
词法（按空格/`;` 切分）→ 语法（mnem + operand 分类）→ 语义（寄存器号/哨兵/imm 打包）
→ Encode(dst/src/imm → 各 8 bit 字段 + imm32) → 输出 u64 数组
```

### 4.2 反汇编流水

```
每 64 bit：Decode → mnemon 查表 → 寄存器/立即数字段还原 → 规范化文本行
（dst/src 的 0xFF → 立即数形式；否则 vN；imm32 按 `f` 十六进制位型输出）
```

### 4.3 语义表 → 执行器

`shader/isa` 提供 `std::optional<Instr> ToInstr(u64 word)`：解码 + 字段拾取，产出第 4 章 `Instr`（`op/dst/srcA/srcB/imm`），随后由通道 4 执行器按 lane 执行。语义表必须与 `common/isa.h` 枚举**单向一致**（工具脚本校验：助记符、opcode、分区无漂移）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T5.1 | ① 契约 | 编码布局、opcode 区段表、20 条指令枚举与语法冻结到 `common/isa.h`；`Encode/Decode` 签名与位掩码常量 |
| T5.2 | ② 主体 | `Encode/Decode`：指令 ↔ 64 bit 字往返；位布局按 §3.1 断言；非法字段（`dst>127`、未知 opcode）译码失败路径 |
| T5.3 | ② 主体 | `isa_asm`：全语法解析、寄存器/哨兵/立即数、报错行号与列号；对给定样例与参考二进制逐字一致 |
| T5.4 | ② 主体 | `isa_disasm`：规范文本输出；对任意合法二进制逐行可再汇编 |
| T5.5 | ② 主体 | `ToInstr` 语义表：20 条助记符 ↔ `Instr` 映射，`CMP/SETP` 谓词写入执行器的桥接 |
| T5.6 | ② 主体 | smoke 集：每条指令合法用例 + 边界用例（`dst=0/127`、`count` 尾、`s` 编号被拒、`i/f` 立即数），无解码失败 |
| T5.7 | ③ 接入 | E0 往返子集补全（替换 ch1 的 `SKIP_PENDING_CH5` 占位用例）；`scene_e2` 的 VS 样例切换为编码投递 |
| T5.8 | ④ 验收 | A5：E0（往返 + 全指令 smoke）全绿 + E2 VS 执行子集 + 专有判据 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D5.1 | `static_assert(sizeof(编码字)==8)`；opcode 表与 §3.2 逐项一致（脚本比对）；`decode(dst>127)` 与未知 opcode 均返回明确错误码 | L1 契约 |
| D5.2 | 构造 20 条指令各 1 个实例：`Encode∘Decode` 还原 `{op,dst,src0,src1,imm}` 全等；`src0==0xFF` 用例 imm32 位型还原 | L1 契约 |
| D5.3 | 样本程序 P1/P2 汇编后与参考二进制逐 64 bit 字一致；语法错误（未知名、`v128`、缺操作数）返回行/列错误 | L1 契约 |
| D5.4 | 对 bin1 反汇编 → 再汇编 → bin2：逐字相等（E0 主判据）；对全部 smoke 指令重复该往返 | L1 契约 |
| D5.5 | `ToInstr`：20 条指令全部可映射；`CMP` 六种 OP 每种 assert 谓词位；未知 opcode → `std::nullopt` | L1 契约 |
| D5.6 | smoke 用例计数 = 2×20（合法+边界）；逐条 `Decode` 不抛异常、无失败返回；`s` 编号输入被汇编器日志级拒绝（非静默） | L1 契约 |
| D5.7 | E0 中"汇编往返"用例由 `SKIP_PENDING_CH5` 变为通过，计数并入 E0 断言总数；`scene_e2` VS 以汇编文本→编码投递，A4（`scene_e2` 寄存器终态）仍全等 | L1 契约 + A5 |
| D5.8 | A5 全绿：E0 断言数比 ch1 增加往返+smoke 相应条数；E2 VS 终态回归不退化 | L1 契约 |

---

## 7. 场景与判据（A5）

**绑定场景：E0 `scene_e0_selftest`（往返补全）、E2 `scene_e2_triangle`（编码投递）。**

| 项 | 内容 |
|---|---|
| 输入 | E0：20 条指令 smoke 程序 + P1/P2 样本；E2：手写 VS 汇编文本 |
| 产物 | 往返二进制、E2 VS 寄存器终态 |
| 主判据 | E0：汇编→二进制→反汇编→再汇编二进制相同；E2：VS 终态与第 4 章一致 |
| 本章专有判据 | ① 子集全指令 smoke 无解码失败；② `s` 编号在标量通路启用前被拒绝（不静默展开） |
| 回归项 | A1（类型/包头断言）、A4（终态） |
| 后续补齐 | 完整 E2 全图（A6）；完整编码与计算指令（第 16 章）；编译产物一致（第 17~19 章） |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 编码唯一；`i32:f<hex>` 立即数消除了文本舍入分叉；往返是纯函数 |
| 时间无关 | 无时间输入 |
| 可观测 | `isa_asm/isa_disasm` 标准输入输出；语义表差异检测脚本 CI |
| 契约影响 | 编码布局、opcode 号、语法均为冻结契约；变更需 ADR + 回归 E0/E2 |
| 简化登记 | 子集指令、线性编码、无标量执行、无分支（§9） |
| 计算兼容 | 计算指令区段（subgroup/atomic/barrier）在 `../design/isa-abi.md` §2 已预留，本章不动区段边界 |
| 配置与回退 | 无开关 |
| 调试钩子 | 无 shader trace（只有 `--trace=isa` 打印往返中间字，供人工） |
| trace 契约 | 不进入判据 |
| 契约变更 | 见"契约影响"行；第 16 章冻结前允许增补指令，增加必须从预留区段取号并登记 |
| 校验归属 | 全部 L1 契约断言；E0 往返是 L1 阻断判据 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| ISA | 约 20 条指令子集、线性编码、无压缩编码 | 16（完整编码 + 汇编器全集） |
| 标量通路 | 只冻结编码位，不实现执行 | 16 |
| 分支 | 无分支指令（禁止），`CMP/SETP` 谓词仅占位 | 15 |
| 访存指令 | `LD/ST` 走常数基址索引 1 种寻址模式 | 16 编址模式扩充 |
| 着色器来源 | 全手写汇编 | 17~19 编译器 |

---

## 10. 待定项

- **Q-75**：本章 `LD/ST` 只做 smoke，真实访存用例第 6 章有消费方时补。
- **Q-76**：`SETP` 维持恒真占位，第 15 章按 ADR 改写。
