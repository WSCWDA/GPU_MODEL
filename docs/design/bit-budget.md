# 位宽与地址空间预算审计

> 项目目标、范围、阶段与交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

> 定位：对现有设计中的位域、编号空间、MMIO 窗口、GPU VA 与环/提交容量做一次集中盘点，回答"当前分配是否够用、哪里接近耗尽、扩容会牵动哪些契约"。
> 项目范围与阶段以任务规划为主干；本文件是位宽和地址空间余量的唯一审计基准。登记位置、层间接口与章级实施分别由 roadmap、architecture-design 和对应章节维护；代码现状只作为实现偏差证据，不自动升级为契约。
> §2–§8 中的碰撞示例和备选方案记录审计过程。现行分配见下表、§7.1 和 §9，具体字段以命令、ISA 和内存设计文档为准。

---

容量和历史裁决见本文。新增编号、字内位段与跨字段占用的查重方法、现行校验记录及实施前待登记项见 [编码与位段校验表](allocation-audit.md)。

## 1. 结论摘要

| 编号 | 分配项 | 当前口径 |
|---|---|---|
| BB-01 / BB-03 | 顶层包号与子操作码 | 顶层 4 bit，0x0F 为 EXTENDED。子操作码跟随实际功能登记 |
| BB-02 | 包长度 | 12 bit，计入包头。执行仍受队列容量限制 |
| BB-04 / BB-05 | stateId / MMIO | 直接槽 0~7。0x50~0x7F 保留。MMIO 256 dword，RT 宽高分别占一个 dword |
| BB-06 | 多队列 | 第 32 章沿用第 10 章直连 ring 寄存器模型。间接描述表与活动窗口切换是后续扩展 |
| BB-07 | GPU VA | 公共容器 u64，有效 36 bit。第 3 章用 flat，第 11 章启用 12+12+12 位翻译 |
| BB-08 | ISA | 7 bit opcode，bit 63 为 SFLAG |
| BB-09 / BB-10 / BB-11 | 队列、堆与其它域 | 容量与编码宽度分别核对。按实际分配和消费方启用，见 §5 和 §7.1 |

---

## 2. 命令提交编码

### 2.1 包头现状

`command-mmio.md` §1 冻结的布局：

```
 31        28 27                 16 15                        0
┌──────────┬──────────────────────┬───────────────────────────┐
│ PKT_TYPE │ count（含头，dword） │ packet-specific / opcode  │
└──────────┴──────────────────────┴───────────────────────────┘
```

- `PKT_TYPE`：4 bit，最多 16 个顶层类型。
- `count`：12 bit，单包最大 4095 dword。
- packet-specific：16 bit，当前只被各包当作零散 flags 使用，没有统一的子操作码定义。

### 2.2 历史冲突：最后一个顶层号曾被多处争用

`command-mmio.md` §2 已登记 `0x00`~`0x0E` 共 15 个类型，`0x0F` 为唯一预留。后续章节却把新包直接指向 `0x0F` 或更早的号：

| 来源 | 主张 | 冲突点 |
|---|---|---|
| `command-mmio.md` §2 | `0x0E=BARRIER`，`0x0F=预留` | 权威登记 |
| `stage2/20-resource-binding.md` §3.4 | `BIND_DESCRIPTOR_TABLE` 取 `0x0E` 扩展/`0x0F`；`SET_KERNEL_ARGS` 取 `0x0F`；`BARRIER` 取 `0x0F/0x0D` | 两个新包争 1 个号；`BARRIER` 与权威的 `0x0E` 不一致 |
| `stage2/32-queue-sync.md` §3.2 | `PREDICATE=0x0D`、`BARRIER=0x0E`、`0x0F` 预留 | 与权威一致，但 `0x0F` 已被 20 章占用 |
| `stage2/33-cmd-chain-multiengine.md` §3.1 | `INDIRECT=0x0C`、`PREDICATE=0x0D`；另列 `DRAW_INDIRECT`、`DISPATCH_INDIRECT`、`DMA_COPY` 未给号 | 至少还差 3 个顶层号 |

按最保守口径，尚未落号的包族至少有：`BIND_DESCRIPTOR_TABLE`、`SET_KERNEL_ARGS`、`DRAW_INDIRECT`、`DISPATCH_INDIRECT`、`DMA_COPY` 五个；可用顶层号只有 `0x0F` 一个。因此编号空间的「大段预留」表述与 `command-mmio.md` §2 原先的实际登记状态不符，当时仅 `0x0F` 可分配，现已登记为 `EXTENDED`（权威：`roadmap.md` §7.4 与本文件 §9）。

另外，`common/include/common/packet.h` 中 `PacketType::ERROR = 0xFF`。`0xFF` 在 4 bit 类型域中会被截断为 `0x0F`，不会报错，只会静默占用预留号；这也说明单靠"约定不要用 0xFF"不足以防止类型域越界。

### 2.3 三种扩容方案

| 方案 | 做法 | 容量 | 代价 | 评价 |
|---|---|---|---|---|
| A（推荐） | 保留 4+12 布局，`0x0F` 定义为 `EXTENDED`，用 packet-specific 的 16 bit 作子操作码 | 1 个顶层号 → 65536 个子操作码 | CP 多一级派发；错误上报需带 `{type=0x0F, subOpcode}`；新增 `STREAM_HEADER.version=2` 门控 | 不动已有包和 `count`，最适合"只增不改" |
| B | 改为 5 bit 类型 + 11 bit count | 32 个顶层类型，单包 2047 dword | 改包头布局；大载荷包上限下降 | 若坚持顶层类型、且确认单包不会超 2047 dword，可选 |
| C | 改为 8 bit 类型 + 8 bit count | 256 个顶层类型，单包 255 dword | count 太小，PSO/批量绑定会频繁拆分 | 不建议 |

方案 A 的子操作码分配建议（需在 `command-mmio.md` §2 新增子表，并在 `roadmap.md` §7.3 登记）：

| 子操作码 | 用途 |
|---|---|
| `0x0001` | `BIND_DESCRIPTOR_TABLE` |
| `0x0002` | `SET_KERNEL_ARGS` |
| `0x0010~0x001F` | 间接绘制/派发/拷贝族（`DRAW_INDIRECT`、`DISPATCH_INDIRECT`、`DMA_COPY` 等） |
| 其余 | 保留，先登记再使用 |

### 2.4 count 与 packet-specific

- `count` 当前最大包是 `SHADER_PSO` 的 23 dword，离 4095 很远。未来若出现大载荷，优先用 state block / IB / 描述符堆，不要为了载荷去压缩类型位。
- packet-specific 目前没有"按包类型分配 bit"的登记表。若采用方案 A，它同时承担子操作码；其余类型仍作 flags。建议在 `command-mmio.md` §2 增加"类型 → 16 bit 位定义"表，避免不同章节各自解释同一位。

---

## 3. stateId 与 MMIO

### 3.1 stateId 的直接槽位

包内字段宽度与 MMIO 实际槽数分别计算。现行直接槽为 0~7，每个基址占两 dword，映射到 `0x40+2n`。 `0x50~0x7F` 整段保留，未加入可分配直接槽。扩展先基于实际状态消费者设计内存间接表，再审计所需根指针和编号。

### 3.2 MMIO 有效分段

现行窗口 256 dword，以 [命令与 MMIO §4](command-mmio.md) 为分段登记。stateId 占 `0x40~0x4F`。PSO 暂存区为 `0x80~0xDF`，第 6 章使用 `0x80~0x93`。RT 宽、高各占一 dword（`0xE0` 与 `0xE1`），`0xE2~0xEF` 为上下文、队列与统计区。未启用字段保留，新增能力按实际用途登记。

### 3.3 历史地址口径冲突

以下为旧文档和历史代码的审计对照。现行偏移以 [公共 MMIO 登记 §4](command-mmio.md) 为准。

代码 `hw_sim/include/hw_sim/mmio.h` 使用**字节偏移**（`0x00/0x04/0x08...`），并在 `hw_sim/src/mmio.cpp` 用 `offset/4` 索引寄存器；而 `command-mmio.md` §4 明确规定"MMIO 偏移一律为 dword 索引"（Q-31）。同一数值在两边含义不同：

| 数值 | 文档含义（dword 索引） | 代码含义（字节偏移） |
|---|---|---|
| `0x40` | stateId 0 的 LO 基址 | stateId 1 的 LO 基址 |
| `0x80` | PSO `vsEntry` LO | PSO `vertexStride` |
| `0xC0` | 未定义（多上下文从 `0xE0` 开始） | `ACTIVE_ASID` |
| `0xE0` | 多上下文段起点 | 超出代码已定义的最高 `0xDC` |

具体重叠：

- 代码 stateId 0~4 占 `0x38~0x5F`（0x38/0x3C、0x40/0x44、0x48/0x4C、0x50/0x54、0x58/0x5C），其中 stateId 3/4 落入文档的 `0x50~0x7F` stateId 扩展区；文档的 stateId 0 在 `0x40`，与代码的 stateId 1 重合。
- 代码 PSO 占 `0x60~0xBC`，横跨文档的 stateId 扩展区（`0x50~0x7F`）和 PSO 区（`0x80~0xDF`）。
- 代码多上下文/抢占占 `0xC0~0xDC`，落在文档 PSO 区内。
- 代码最高只定义到 `0xDC`（dword 55），256 dword 中有约 200 dword 未定义；但这不是"余量"，因为偏移语义与文档不一致，不能直接拿来扩容。

**结论：在讨论"加位"之前，必须先冻结"dword 索引"这一口径，并按一张表重排 stateId / PSO / 多上下文；否则任何扩容都会与现有代码地址碰撞。**

### 3.4 多队列与 fence 寄存器不足

`stage2/32-queue-sync.md` 要求每个队列各自拥有 ring、tail、fence；代码 `MAX_RINGS=4`。按最小直接寄存器模型估算，每环需要：

| 寄存器 | dword |
|---|---|
| base LO/HI | 2 |
| size | 1 |
| head | 1 |
| tail | 1 |
| doorbell | 1 |
| fence value LO/HI | 2 |
| ring 状态/ID | 1 |
| 合计 | **9** |

4 环约 36 dword，再加每队列 fence 状态与 SE 统计，`0xE0~0xEF` 的 16 dword 不够。当前代码实际走的是"单 MMIO 窗口 + FW 保存/恢复"路线（`km_submission_impl.cpp` 每次 `SetupRing` 都写同一组 `MMIO_RING_*`），功能上可模拟，但硬件契约里没有"每环寄存器块"。

可选路线：

| 路线 | 做法 | 优点 | 代价 |
|---|---|---|---|
| 间接环表（历史备选，后续 capability） | MMIO 只保留一个活动 ring 窗口；内存中放 ring 描述表，FW 切换时重编程 | 不扩 MMIO；与现有 FW 路径一致 | 硬件不能同时看到多环；每队列 fence 需落在内存 semaphore |
| 直接每环寄存器块 | 扩到 512 dword，新增 `0x100~0x1FF`；每环 9~12 dword | 硬件模型直观，易调试 | 改 MMIO 窗口契约；需定义 ring ID 选择/门铃语义 |
| 混合 | 常用 ring 0 直接寄存器，其余走间接表 | 折中 | 两套路径，验证面变大 |

---

## 4. GPU VA 与页表

### 4.1 历史冲突：文档与代码的 VA 位宽

`memory-system.md` §2 写的是"GPU VA 64 bit；页 4 KiB；两级页表，每级 512 项"。按这个几何：

```
9 bit PD index + 9 bit PT index + 12 bit offset = 30 bit VA = 1 GiB / 地址空间
```

但同一套设计又把 KMD 堆基址放在 `0x1_0000_0000`（4 GiB）和 `0x2_0000_0000`（8 GiB）。这两个地址分别需要 33 bit 和 34 bit，**30 bit 页表根本覆盖不到**。

代码 `common/include/common/memory_types.h` 则使用：

```
12 bit PD index + 12 bit PT index + 12 bit offset = 36 bit VA = 64 GiB / 地址空间
```

代码注释明确说 36 bit 是为了覆盖 `0x1_0000_0000` 和 `0x2_0000_0000`。也就是说，**代码比文档多 6 bit，文档目前自相矛盾**。

### 4.2 两种修正方向

| 方向 | 做法 | 影响 |
|---|---|---|
| A（推荐） | 采用 36 bit：两级各 4096 项，更新 `memory-system.md` §2，把"64 bit VA"改为"软件类型 64 bit，硬件有效 VA 36 bit" | 与代码和 4 GiB/8 GiB 基址一致；每级页表 4096×16 B = 64 KiB（代码 PTE 为 8 B PA + 4 B flags，按 8 B 对齐），比 4 KiB 页表大 |
| B | 保留 30 bit：把堆基址降到 1 GiB 以下，删除 4 GiB/8 GiB 基址 | 改动小，但地址空间上限 1 GiB/AS，后续扩展要加第三级 |
| C | 直接上 48 bit / 四级页表 | 容量充足，但对 64 MB VRAM + 16 MB GART 的模型明显过度设计 |

若选 A，还需在文档中补一行"有效 VA 位宽 = 36 bit；PA 位宽按堆容量单独定义"，避免再出现"64 bit VA"被误读为硬件 64 bit。

### 4.3 TLB 与 ASID 的未分配位

- TLB 索引是 `(va>>12)&0x3F`，即 64 组 × 4 路 = 256 项。36 bit VA 下 VPN 为 24 bit，索引 6 bit，则 tag 至少 18 bit；文档没有写 tag 位宽。
- `AddressSpaceId` 在代码中是 u32，TLB 条目直接存 u32 ASID；如果未来做 RTL 或压缩 TLB tag，需要显式分配 ASID 位宽（例如 8 bit = 256 上下文）。当前只要求两个上下文互不污染，**不是立即不足，但属于未冻结的位预算**。

---

## 5. 环、提交与容量

### 5.1 环指针位宽够用，但默认环太小

- `RING_HEAD/RING_TAIL` 是 32 bit dword 索引，理论上限 `2^32` dword = 16 GiB；位宽不是瓶颈。
- 代码默认环大小 4 KiB = 1024 dword（`km_init_impl.cpp`），`WriteRingData` 要求**单个提交的 dword 数 ≤ 环大小**，否则返回 `OutOfMemory`；跨环尾用 NOP 填充，不允许一个包跨尾。
- `SubmitArgs.cmdBufSize` 是 u64，但落环路径在 `kmd_impl.cpp` 先转成 u32 dword（`cmdBufSize / 4`），再交给 `WriteRingData`。超过 16 GiB 的提交会静默截断，超过 4 KiB 的默认环会直接失败。

因此"提交命令的地址空间"实际受**环大小**限制，而不是 32 bit 指针限制。建议：

- 明确默认环大小与上限（例如 4 KiB 默认、64 KiB/256 KiB 可选）。
- 大命令流走 `INDIRECT`（0x0C）/ IB，而不是要求单次提交塞满环。
- 在 `Submit` 契约中补"单次提交 ≤ 环容量"的显式错误，避免依赖底层 `OutOfMemory`。

### 5.2 堆容量会先于位宽耗尽

- VRAM 64 MB、GART 16 MB；GART 是 bump 分配、不回收、不整理，描述符堆、顶点/索引/上传缓冲都在里面。
- 描述符堆 `capacity` 是 u32，但真实上限由 GART 决定；查询池 `poolOffset` 是 u32，按字节偏移最大 4 GiB，当前也远大于实际池。
- 这些不是"位不够"，而是"预算没写清"。建议 `memory_budget` 给出各堆默认值和告警水位，并在容量审计工具（`tools/mem_audit`）中输出高水位。

---

## 6. ISA 编码

### 6.1 历史冲突：7 bit opcode 与 8 bit 区段表

`isa-abi.md` §1 把 opcode 收窄为 `[62:56]` 7 bit（上限 `0x7F`），但 §2 的区段表仍按 8 bit 写：

| 区段 | 表内用途 | 7 bit 可否编码 |
|---|---|---|
| `0x00~0x7F` | NOP/ALU/内存/控制流/同步/内建 ID/曲面几何 | 可以 |
| `0x80~0x9F` | subgroup | **不可以** |
| `0xA0~0xFD` | 保留 | **不可以** |
| `0xFE` | `UNDEFINED` | **不可以** |
| `0xFF` | `EXIT` | **不可以** |

这不是"未来不够用"，而是**当前规范无法编码自己列出的后半段指令**。第 16 章冻结前必须解决，否则汇编器/解释器/反汇编器无法就同一份表达成一致。

### 6.2 建议：在 7 bit 内重编号（推荐）

`isa-abi.md` §1 已经写明"§2 表中 `0x80` 以上区段按 7 bit 重编号后生效"，缺的是一次完整落表。建议粗粒度映射如下（具体指令号在第 16 章冻结）：

| 区段 | 用途 |
|---|---|
| `0x00` | `NOP` |
| `0x01~0x0F` | 特殊/保留 |
| `0x10~0x1F` | ALU / 谓词 |
| `0x20~0x2F` | 内存 / 纹理 / 片上 |
| `0x30~0x3F` | 控制流 |
| `0x40~0x4F` | 同步 / 原子 |
| `0x50~0x5F` | 内建 ID |
| `0x60~0x6F` | 曲面 / 几何 |
| `0x70~0x7D` | subgroup（14 个槽，足够 BALLOT/SHFL/BROADCAST/REDUCE/DERIV） |
| `0x7E` | `UNDEFINED` |
| `0x7F` | `EXIT` |

这样全部 128 个值都有确定归属，且保持 64 bit 定长与 `[63]=SFLAG` 不变。

其他 ISA 位域：

- `dstReg/src0/src1` 各 8 bit，SGPR 32 + VGPR 128 = 160 个编号，够用；`src0==0xFF` 立即数哨兵也不会与现有寄存器号冲突。
- 若未来 VGPR 扩到 256 个以上，8 bit 字段会先于 opcode 耗尽；目前不需要动。
- `imm32`、分支有符号 32 bit 偏移均够用。

---

## 7. 其它位域与容量

| 项 | 位宽/容量 | 判断 |
|---|---|---|
| RT 尺寸 | `RT_WIDTH` 与 `RT_HEIGHT` 分占 32 bit，MMIO dword `0xE0/0xE1` | 合法尺寸与资源范围由 CLEAR / RT 约束验证 |
| fence 值 | 64 bit | 够用 |
| 页表项 flags | 5 bit 已用 / 32 bit | 余 27 bit，够用 |
| 队列优先级 | u8 | 够用；注意 `QueueDesc.priority` 是 8 bit，若未来要 256 级以上需改 |
| `QueueDesc.familyId` | u32 | 够用 |
| 查询池 `poolOffset` | u32 字节偏移 | 单池最大 4 GiB；按 u64 结果约 5.3 亿槽，够用 |
| 描述符堆 `capacity` | u32 | 编号够用，真实上限是 GART 16 MB |
| `SHADER_PSO.flags` / 各包 flags | u32，无位级登记 | 位宽够，**分配规则缺失**；建议按包类型补 bit 表 |
| `DRAW` 载荷 | 当前 7 dword | 位宽不是问题；`baseVertex/baseInstance` 尚未进包（第 22 章要求），需要追加 2 dword，count 有空间 |

### 7.1 硬件上限登记（契约，不改硬件实现）

> **教学模型硬件上限的契约声明**（本表为唯一权威，不复制到其它文档）：写入文档不扩张实现；实际能力低于上限时按上限登记、超上限拒绝。任一章节涉及包 / MMIO / ISA / 页表分配时先对照本表审计（ADR-043/045，`roadmap.md` §7.4）。

| 上限项 | 契约值 | 依据 / 登记位 |
|---|---|---|
| GPU 有效 VA / 页表 | **36 bit**（两级各 4096 项 × 4 KiB = 64 GiB/AS），软件类型 `GpuVAddr` 仍为 u64 | §4、`memory-system.md` §2、`common` 页表常量 |
| VGPR 总数 | **128**（`v0..v127`），编译器 / ABI 按此编址；`v128+` 不编址 | `isa-abi.md` §3、ADR-034 |
| SGPR 总数 | **32**（`s0..s31`） | `isa-abi.md` §3、ADR-034 |
| opcode | **7 bit**（`[62:56]` + `[63]=SFLAG`），`0x80~0xFD` 按 7 bit 重编号 | §6、`isa-abi.md` §1/§2、ADR-034 |
| VGPR 固定分区 | 一律 `kVgprBase*` 符号；远程量 / mesh / 距离等分区以 `isa-abi.md` §3 为唯一权威 | ADR-033 |
| WARP / LaneMask | `WARP_SIZE=32`、`LaneMask=u32`，取反 `& 0xFFFFFFFF` | ADR-018、`roadmap.md` §5.1 |
| CU LDS 总容量 | **64 KiB/CU** | ADR-038，**B-036**；`architecture-design.md` §3.11 D-03/D-13、§3.12 M-03/M-06 |
| CU 驻留 wave 数上限 | **16 waves/CU** | ADR-038，**B-037**；§3.11 D-13、§3.12 M-06 |
| `kMaxWorkgroups` | **`2^20`** | ADR-038/043，**B-038**；§3.11 D-02 |
| 输入/输出 FIFO 深度 | **8 项** | ADR-038，**B-039**；`execution-parallel.md` §4 |
| 描述符堆容量 | **64K 项**（受 GART 16 MB 约束） | ADR-038，**B-040**；`memory-system.md` §3、`roadmap.md` §7.1 |
| 参数区容量 `kMaxParamBufBytes` | **4 KiB（1 页）** | ADR-043，**B-041**；§3.8 规则 2（与 VGPR 分区解耦） |
| 寄存器编号位宽 | 8 bit，`src0==0xFF` 为 `MOV` 立即数哨兵；VGPR 128 < 255，无碰撞 | `isa-abi.md` §1 |

> 上表 6 行（CU LDS / 驻留波数 / `kMaxWorkgroups` / FIFO / 描述符堆 / 参数区容量）为 ADR-038/043 **固定容量**，已于 **2026-11-20 定案**并领 `B-036~B-041`；改值必须走 `roadmap.md` §7.4 契约变更。D-02/D-03/D-13 与 M-03/M-06 按本表上限判定，超限在 HLI/驱动记录期 `InvalidArgument` 拒绝（`architecture-design.md` §3.11 D-13），不进入排队。参数区容量与 VGPR uniform 分区（`v[56..67]`）**解耦**，见 `architecture-design.md` §3.8 规则 2 修订。

> 注意：历史表述「两级 512 项 → 30 bit / 1 GiB」与代码 36 bit 及 4 GiB/8 GiB 堆基址矛盾，**已裁定为 36 bit**（BB-07，见 §4 与 §9 决策 2）。

---

## 8. 建议的处理顺序

| 顺序 | 动作 | 影响面 | 建议时点 |
|---|---|---|---|
| 1 | ~~冻结 ISA opcode 的 7 bit 重编号~~ **已完成**：已按 §6 建议落表并回写 `isa-abi.md` §1/§2（`0x70~0x7D` subgroup、`0x7E=UNDEFINED`、`0x7F=EXIT`）与 `roadmap.md` §7.3（ADR-043） | 第 16 章按 `isa-abi.md` §2 细化具体指令号 | 第 16 章冻结前 |
| 2 | 定案包类型扩展方案（A/B/C），清理 20/32/33 章的包号冲突 | `command-mmio.md` §2、`roadmap.md` §7.3、相关章文档 | 第 1 章/新增包落地前 |
| 3 | 统一 MMIO 为 dword 索引，重排 stateId/PSO/多上下文 | `mmio.h`、`command-mmio.md` §4、第 2/10 章 | 下一次改 MMIO 前 |
| 4 | 把 VA 有效位宽统一为 36 bit（或降堆基址），补 ASID/TLB tag 位宽 | `memory-system.md` §2、`memory_types.h`、第 11/14 章 | 第 11 章前 |
| 5 | 核对直连 ring 与内存 semaphore 的边界。活动窗口方案保留为扩展 | `command-mmio.md` §4、第 32 章、FW | 第 32 章前 |
| 6 | 明确环默认大小/上限与大命令流走 IB 的规则 | `Submit` 契约、`km_submission`、场景 | 性能/大场景前 |
| 7 | 给出 VRAM/GART/描述符堆预算与告警水位 | `memory_budget`、`tools/mem_audit` | 高级一期联调前 |

> 编号空间说明（P1-5，**2026-11-20 已统一**）：本文件审计项编号 **`BB-01~BB-11`** 与 `roadmap.md` §7.3 的**分配登记编号 `B-nnn`** 是两套编号空间（同一事项两号，如 GPU VA = 本表 `BB-07` / §7.3 `B-007`）。审计项一律用 `BB-nn` 前缀（只作审计行号，不作分配登记）；分配值只引用 `B-nnn`。

---

## 9. 现行分配决策

> **裁决状态（ADR-043）**：以下 1~5 项已在主干文档定案，本审计按"已裁决"引用于 `roadmap.md` §0.3 ADR-043/045 与 §7.4 编号空间、`architecture-design.md` §7.4/§7.6；实际收缩实现时若有偏差须先改 ADR 再改本表。**第 6 项已裁定（2026-11-20）**。

1. **命令包类型**：**采用 A**——`0x0F` 定义 `EXTENDED` 转义 + 16 bit `packet-specific` 子操作码，或复用已有包载荷 `SubOp`；不新增顶层包号硬编码；取号前 `roadmap.md` §7.3 / §7.4。
2. **GPU VA**：**采用 36 bit**（两级各 4096 项 = 64 GiB/AS，与代码及 4 GiB/8 GiB 堆基址一致）；历史 30 bit/1 GiB 表述废止。
3. **MMIO 窗口**：**保持 256 dword**，不扩 512；大块状态一律移出 MMIO 放 GPU 内存，MMIO 只存根指针/控制/计数（扩展时复用未列满段 + 内存状态表）。
4. **多队列**：一期沿用 [第 10 章 §3.4](../stage2/10-device-ring-cp.md) 的直连 ring 寄存器模型，跨队列同步使用内存 semaphore。间接环描述表及活动窗口切换留作后续扩展。
5. **stateId**：直接槽为 0~7，`0x50~0x7F` 保留。索引需求扩展时优先考虑内存间接状态表，并按新增用例登记。

6. **MMIO 偏移口径（ADR-054、B-042）**：dword 索引，0x06/0x07 登记 ring 基址低高位，0x11 登记 CP_STATUS。位宽和权限见 [公共表 §4](command-mmio.md)。第 10 章汇总前序子集及实现的审计。

编号空间按 ADR-043/045 管理。公共 MMIO 偏移以 ADR-054 和当前登记为准，代码执行及前序子集回归由 B-042 跟踪。

---

## 2026-11-21 评审同步说明

公共 Result 见 [第 1 章 §3.2](../stage1/01-contracts-min.md)，包和 MMIO 见 [命令与 MMIO 登记](command-mmio.md)，计算容量与间接派发见 [架构设计 §3.11](../architecture-design.md)。契约变更及旧预算退役按 [路线图 §7.4](../roadmap.md) 执行。
