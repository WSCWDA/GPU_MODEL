# ISA 与 Shader ABI — 独立模块设计

> 定位：定义 GPU 虚拟 ISA 的 64 bit 定长编码、opcode 区段与着色器寄存器 ABI 分区，是全栈共享（解释器、汇编器、反汇编器、编译器、LLVM 对照）的唯一权威。
> 落地章：第 5 章初版（最小 ISA）、第 16 章冻结（完整编码与汇编器）；编译器 17~19 只消费本 ABI，不共享内部数据结构。
> 权威来源：从 `architecture-design.md` §4.1 拆出；决策约束见 ADR-018 / ADR-033 / ADR-034、`roadmap.md` §4.3（Shader ABI 冻结）与 §2.8（补强项落位）。
> 权威顺序：`roadmap.md` → `architecture-design.md` → `../docs/design/isa-abi.md` → `stageN` 章文档。

---

## 1. 指令编码（64 bit 定长）

```
[63]    SFLAG   1 bit（标量指令标志，向量指令为 0）
[62:56] opcode  7 bit
[55:48] dstReg  8 bit（SGPR/VGPR 统一编号）
[47:40] src0    8 bit
[39:32] src1    8 bit
[31: 0] imm32   32 bit（立即数 / 地址偏移 / 分支偏移）
```

规则：

- opcode 为 7 bit（上限 `0x7F`）、`[63]=SFLAG`（ADR-034、Q-32）；§2 区段表已按 ADR-043 / `bit-budget.md` §6 在 7 bit 内完成重编号（`0x7E=UNDEFINED`、`0x7F=EXIT`），具体指令号在第 16 章冻结时按 §2 落表。
- 分支偏移为有符号 32 bit，相对当前 `pc`。
- `MOV` 的 `src0 == 0xFF` 表示"写立即数"哨兵。
- 编码为唯一权威；汇编器、反汇编器、解释器、编译器共用 `common/isa.h`。
- `dstReg` 是"SGPR/VGPR 统一编号"字段：实际空间由 `src0/src1` 侧的标量/向量区分（ADR-034，见 §3），`[63]=SFLAG` 区分标量指令与向量指令。

## 2. opcode 区段

| 区段 | 用途 | 代表指令 |
|---|---|---|
| 0x00 | 空操作 | `NOP` |
| 0x01~0x0F | 特殊/保留 | — |
| 0x10~0x1F | ALU / 谓词 | `ADD/SUB/MUL/DIV/AND/OR/XOR/NOT/CMP/MOV/MAD/SETP` |
| 0x20~0x2F | 内存 / 纹理 / 片上 | `LD/ST/SAMPLE/LDS/STS` |
| 0x30~0x3F | 控制流 | `BR/BRC/CALL/RET/SYNC_PUSH/SYNC_POP/RECONVERGE` |
| 0x40~0x4F | 同步 / 原子 | `BARRIER/ATOMIC_ADD/ATOMIC_CAS` |
| 0x50~0x5F | 内建 ID | `GET_THREAD_ID/GET_GROUP_ID/GET_VERTEX_ID/GET_PIXEL_POS/GET_INSTANCE_ID` |
| 0x60~0x6F | 曲面 / 几何 | `TSETUP/EVAL/EMIT/RESTART` |
| 0x70~0x7D | subgroup（第 29 章） | `BALLOT/SHFL/BROADCAST/REDUCE/DERIV` |
| 0x7E | `UNDEFINED` | 解码失败 |
| 0x7F | `EXIT` | 程序结束 |

> 上表为 **ADR-043 / `bit-budget.md` §6 已裁定的 7 bit 重编号**（原"0x80~0xFF"区段按 7 bit 压缩后落位，`0x7E=UNDEFINED`、`0x7F=EXIT`）；具体指令号在第 16 章冻结时按本表细化，增量 opcode 走 `roadmap.md` §7.3 登记（`B-nnn`，先对照 bit-budget 审计）。

> 计算线不新增 opcode 区段：subgroup / shared / 原子 / 内建 ID 区段在图形线推进时就预留到位（`roadmap.md` §2.1 共用单元清单、`docs/design/compute-line.md` §1~§2 共用单元原则）。

- 谓词统一为 **u64 值 + mask/cond 比较**（Q-70）：`SETP` 写完整 u64 谓词值，分支与选择按 mask/cond 解释；bool 仅为其特例，不再单独定义 bool 寄存器具名。

## 3. Shader ABI（第 16 章冻结）

| 项 | 约定 |
|---|---|
| 波宽 | `WARP_SIZE = 32`，不开放配置（ADR-018） |
| lane 顺序 | lane id 与 `ThreadId` 线性对应，不做 swizzle；归约与 ballot 按 lane 升序 |
| 活跃掩码 | `LaneMask = u32`，`kLaneMaskAll = 0xFFFFFFFF`；取反后必须 `& kLaneMaskAll` |
| 控制流栈 | `struct { u32 pc; LaneMask mask; }`，不做位域打包 |
| 输入属性 | `kVgprBaseAttrib`（`v[0..15]`），由 IA 按 `VertexInputLayout` 装配（`architecture-design.md` §3.7） |
| VS 输出 | `kVgprBaseVsOut`（`v[16..27]`） |
| HS 因子 | `kVgprBaseTessFactor`（`v[28..35]`）：`(inner,e0,e1,e2)` 四项结构共 8 个槽 |
| PS 输出 | RT0 `v[36..39]`；深度 `v[40..43]`；RT1~3 `v[44..55]`（RT0 起分区整体后移） |
| uniform | `v[56..67]`，只放小体积高频标量；大块常量走常量区 + `LD` |
| 距离区 | `kVgprBaseDistance`（`v[68..75]`）：先 clip 后 cull，`clipCount + cullCount ≤ 8`，count 由 PSO 声明；未声明时该区仍保留 |
| 临时量 | `v[76..]`，线性扫描从此起步 |
| 属性描述 | 描述项含 `stage` 字段；`location` 按 `stage` 基址换算到对应 `kVgprBase*` 分区（Q-54） |
| 内建 ID | 只读 SGPR / 专用寄存器（`GET_THREAD_ID` 等），不占 VGPR 分区（Q-55） |
| SGPR | `s0..s31`，波内各 lane 一致的值：返回地址、统一常量、共享内存地址、波元数据；只在标量指令中出现 |
| shared | 独立地址空间，只有 `LDS/STS` 可访问；大小经 PSO 下发 |

硬规则：

1. 固定分区不允许被临时量覆盖；`SymbolBindings` 只是分区的机器编排，不得改变分区本身；分区调整必须走 ADR。
2. Clip/Cull Distance 固定落在 `kVgprBaseDistance`（`v[68..75]`，ADR-033）：只有 VS/DS/GS 可写、PA 在裁剪阶段按 count 读取，PS 不读；写超声明 count 视为着色器契约错误（L1 阻断）。
3. 标量（`s`）与向量（`v`）寄存器**独立编号空间**、`[63]=SFLAG` 区分；编译器前端按发散分析把一致值放 SGPR、发散值放 VGPR（ADR-034）。初级可只实现向量指令，标量指令留空编码位，ABI 分区先冻结后期不动编码格式。
4. `VertexInputLayout` 的结构体定义见 `architecture-design.md` §3.7（层间数据契约），IA 只消费该布局，不感知 API 顶点格式。
5. VGPR 分区一律以 `kVgprBase*` 符号常量表达；文档与代码一律引用符号，不写裸寄存器号（Q-57/Q-64）。分区数值以本表为唯一定义处，跨阶段重排与临时量占用一律禁止。

## 4. 汇编器 / 反汇编器

- 落地：初级第 5 章（约 20 条指令子集、线性编码）、高级第 16 章（完整编码 + 汇编器 + 指令集自检 + 计算指令与特殊寄存器）。
- 活验收：汇编 → 二进制 → 反汇编 → 再汇编二进制完全相同（E0/S0 的 `--trace` 与 L1 断言）；全指令集 smoke 无解码失败。
- 工具：`tools/isa_asm` / `tools/isa_disasm`（`docs/design/validation-model.md` §7）。

## 5. 与编译器 / LLVM 对照的关系

- 编译器（`../docs/design/compiler.md`）与 `hw_sim` 的唯一连接是 `common` 的 ISA 定义与 Shader ABI，不共享内部数据结构。
- SSA ↔ LLVM IR 对照（第 19 章）以"两后端逐字节一致"为唯一判据，底层就是本文件编码必选一致。

## 6. 契约变更

- 任何 opcode 增删、ABI 分区调整、宏/编码变更都视为冻结契约变更，走 `roadmap.md` §7.4 工作流（新建 ADR）并回归 E0~E5 与受影响 S 场景。
- opcode 区段表已按 ADR-043 / `bit-budget.md` §6 完成 7 bit 重编号（subgroup 落 `0x70~0x7D`、曲面/几何 `0x60~0x6F`、`0x7E=UNDEFINED`、`0x7F=EXIT`）；具体指令号在第 16 章冻结时按 §2 落表，增量登记见 `roadmap.md` §7.3（`B-nnn`）。
- 编码/分区登记表见 `roadmap.md` §7.3。
