# 命令包、stateId 与 MMIO — 独立模块设计

> 项目目标、范围、阶段与交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

> 定位：定义驱动与硬件之间的唯一命令载体：命令包（`STREAM_HEADER` 版本化包头 + 包类型登记表）、`stateId` 基址索引、以及 MMIO 寄存器分段。CP 按包类型派发，硬件单元只认命令包与自身状态。
> 落地章：第 1 章冻结包头与版本、第 2 章 CP 最小派发、第 10 章 MMIO/KMD-FW 契约；包类型与 MMIO 段在后续章追加。
> 权威来源：从 `architecture-design.md` §4.5 拆出，并吸收编号空间审计（原 §5.6，权威在 `bit-budget.md` §9 与 `roadmap.md` §7.4）、命令包版本（原 §10.2，现在 `roadmap.md` §2.8）、ADR-004 / ADR-008（命令缓冲位于宿主内存、取指方式）与 `roadmap.md` §2.7（CP 按包类型分派）。

---

## 1. 包头（第 1 章冻结）

```
 31        28 27                 16 15                        0
┌──────────┬──────────────────────┬───────────────────────────┐
│ PKT_TYPE │ count（含头，dword） │ packet-specific / opcode  │
└──────────┴──────────────────────┴───────────────────────────┘
```

- `count` 12 bit，最大 4095 dword；包不得跨环尾，空间不足用 `NOP` 填充。
- 每个命令流首包必须是 `STREAM_HEADER`（0x00），载荷含 `version/engine/flags`；版本策略：初级 `version=1`；高级 `version=2` 且拒绝 v1（Q-28）。版本不匹配拒绝，并报出包类型与偏移（`roadmap.md` §2.8「命令包版本号」）。
- 新增字段一律追加在载荷尾部；解析器按 `count` 截断，未用字段视为 0/默认值。

## 2. 包类型登记表

| PKT_TYPE | 名称 | 首次落地 | 作用 |
|---|---|---|---|
| 0x00 | `STREAM_HEADER` | 1 | 命令流版本、引擎、标志 |
| 0x01 | `FENCE` | 2 | 写 fence 值并触发中断 |
| 0x02 | `SET_STATE_BASE` | 3 | 设置状态基址，载荷含 `stateId` |
| 0x03 | `DRAW` | 6 | 绘制；`indexed` 标志区分索引/非索引 |
| 0x04 | `DISPATCH` | 37 | 派发 workgroup（载荷见 §2.2） |
| 0x05 | `COPY_MEM` | 3 | 块拷贝（Copy Engine） |
| 0x06 | `SHADER_PSO` | 6 | 着色入口 + 固定功能状态，变长载荷 |
| 0x07 | `NOP` | 1 | 空包 / 环尾填充 |
| 0x08 | `CLEAR` | 3 | 清屏（CP 级） |
| 0x09 | `QUERY` | 34 | begin/end/resolve |
| 0x0A | `SIGNAL` | 32 | timeline semaphore 信号，单调与失败传播见 [同步原语 §2.1](sync-primitives.md#21-提交完成与失败传播) |
| 0x0B | `WAIT` | 32 | 等待 fence / semaphore，只停驻当前队列，按完成结果推进 |
| 0x0C | `INDIRECT` | 33 | IB 链 / 间接绘制与派发 |
| 0x0D | `PREDICATE` | 33 | 条件执行 |
| 0x0E | `BARRIER` | 32 | 屏障 / 同步（载荷见 §2.3） |
| 0x0F | `EXTENDED` | 20 | 转义：`packet-specific` 16 bit 作子操作码（子表见 §2.1） |

### 2.1 `EXTENDED`（0x0F）子操作码表

`packet-specific` 16 bit 取值；先登记再取号（ADR-043，登记位 `roadmap.md` §7.3）：

| SubOp | 名称 | 用途 | 落地章 | 登记 |
|---|---|---|---|---|
| `0x0001` | `BIND_DESCRIPTOR_TABLE` | 绑定描述符表基址 | 20 | B-032 |
| `0x0002` | `SET_KERNEL_ARGS` | 记录已准备的 kernel 参数区（paramBufVa/size/paramVgprBase） | 20 记录契约、37 硬件消费、39 主机参数打包 | B-033 |
| `0x0010` | `DISPATCH_INDIRECT` | 从 GPU 内存读取 28-byte/16-byte-aligned `{grid[3], block[3], reserved0}` | 33、41 | B-034 |
| `0x0011` | `DRAW_INDIRECT` | 间接绘制 | 33 | B-044 |
| `0x0012` | `DMA_COPY` | `{dstVa:u64, srcVa:u64, bytesPerRow:u32, srcStride:u32, dstStride:u32, sliceCount:u32}` 行切片；32 B payload | 33 | B-035 |
| 其余 | 保留 | 先登记再取号 | — | — |

`DMA_COPY` 行切片约束：`bytesPerRow`、`sliceCount` 非零；两种 stride 均不小于 `bytesPerRow`；任一地址运算溢出或源/目标覆盖区间重叠返回 `InvalidArgument`，任一源/目标行未映射返回 `NotMapped`。执行前须验证完整区间，失败不得部分写入；相同源/目标地址且 stride 相同为 no-op。

### 2.2 `DISPATCH`（0x04）二进制载荷（dword 索引）

> 唯一来源 `architecture-design.md` §3.11 `DispatchDesc`（排除宿主侧 `paramOffsets`/`subgroupSize`：偏移表不进包，`subgroupSize` 固定 32 不进包）；校验规则 D-01~D-13 由 HLI/计算驱动在记录期执行；字段顺序 = `DispatchDesc` 成员顺序。

| dword | 字段 | 说明 |
|---|---|---|
| 0~1 | `kernelVa` LO/HI | module 代码段内 ISA 入口（4 B 对齐，D-06） |
| 2~3 | `paramBufVa` LO/HI | 参数区（§3.8 `HOST_VISIBLE|HOST_COHERENT`） |
| 4 | `paramBufByteSize` | 参数区字节数（≤ `kMaxParamBufBytes`，§3.8 规则 2） |
| 5 | `paramVgprBase` | 默认 56，合法域 uniform 区 `v[56..67]`（§3.8 规则 5） |
| 6~8 | `grid.x/y/z` | workgroup 数（D-02） |
| 9~11 | `block.x/y/z` | 每 workgroup 线程数（D-01） |
| 12 | `ldsStaticBytes` | 静态 LDS 需求（16 B 对齐，D-03/D-13） |
| 13 | `sgprCount` | 资源声明，仅 occupancy 预算（D-04） |
| 14 | `vgprCount` | 资源声明，仅 occupancy 预算（D-04） |
| 15~16 | `indirectArgsVa` LO/HI | 0 = 直接 dispatch（D-10） |
| 17 | `flags` | bit0 = 组内 barrier；bit1 = device 作用域；其余必须 0（D-09） |

### 2.3 `BARRIER`（0x0E）二进制载荷（dword 索引）

固定为 3 dword，所有资源、执行依赖、队列所有权转换共用此信封：

| dword | 字段 | 编码 |
|---|---|---|
| 0 | `kind` [7:0]、`scope` [15:8]、`memOrder` [23:16]、`accessMask` [31:24] | kind: resource=0 / execution=1 / ownership=2；其余枚举以 sync-primitives.md 为准 |
| 1 | `srcStageMask` | 来源阶段位图 |
| 2 | `dstStageMask` | 目标阶段位图 |

未知枚举、保留位或非法组合返回 `InvalidPacket`。Workgroup 执行屏障在 ISA/LDS 同步点实现，不通过该 CP 包模拟；此包仅表达资源、设备、队列可见依赖。20/30/32/41 章不得另定义载荷布局。

### 2.4 `PREDICATE`（0x0D）载荷

固定 8 dword payload：`{u64 predVa, u64 mask, u64 cond, u32 skipDwords, u32 reserved0}`。CP 读取 `value=LoadU64(predVa)`；条件为 `(value & mask) == cond`，并要求 `(cond & ~mask)==0`。条件成立时执行后续命令流；条件不成立时跳过紧随其后的 `skipDwords`，该范围必须完整由命令包组成并落在当前 ring/IB 边界内，否则返回 `InvalidPacket`。`reserved0` 必须为 0。`skipDwords=0` 表示不跳过；不支持嵌套 predicate。

## 3. stateId 基址索引

| stateId | 用途 | 落地章 |
|---|---|---|
| 0 | 渲染目标基址 | 3 |
| 1 | 当前上下文页表基址（初级由驱动写入，第 10 章迁 FW） | 3、11 |
| 2 | 深度/模板缓冲基址 | 26 |
| 3 | 纹理/采样器描述符表基址 | 20、21 |
| 4 | 常量区基址（初级提前落地：第 5 章以 stateId=4 承载常量区） | 5、17、18 |
| 5 | 绑定表 / 描述符堆基址（描述符堆登记 GART，GPU 可读，Q-51） | 20 |
| 6 | 查询池基址 | 34 |
| 7 | IB / 间接参数区基址 | 33 |

`SET_STATE_BASE` 的 `stateId` 是唯一索引；不得用"地址直接写寄存器"绕过。

## 4. MMIO 分段（256 dword = 1 KiB）

| 段 | 用途 |
|---|---|
| 0x00~0x0F | ring 读写指针、doorbell、fence 监视 |
| 0x10~0x1F | CP 控制、中断状态、上下文状态 |
| 0x20~0x2F | 错误码、调试、性能计数 |
| 0x30~0x3F | 设备/ring ID、命令流版本回显、引擎标志 |
| 0x40~0x4F | `stateId` 0~7 的 LO/HI 基址（stateId n → `0x40+2n`，各 2 dword） |
| 0x50~0x7F | 保留（`stateId` 扩展备用，Q-31） |
| 0x80~0xDF | PSO 暂存区（`SHADER_PSO` 工作副本） |
| 0xE0 | RT_WIDTH（32 bit） |
| 0xE1 | RT_HEIGHT（32 bit） |
| 0xE2~0xEF | 多上下文 / 多队列 / SE / 并行统计 |
| 0xF0~0xFF | 系统保留（boot、紧急中断） |

规则：

- **MMIO 偏移一律为 dword 索引**，`stateId` n → `0x40+2n`，`0x50~0x7F` 保留（Q-31）。
- RT 尺寸由 `CLEAR` 锁存到 `RT_WIDTH(0xE0)` 与 `RT_HEIGHT(0xE1)` 两个完整 32 bit 寄存器，`DRAW` 不携带尺寸。
- PSO 暂存区字段顺序由第 20~26 章共同维护，新增字段只能追加。
- `0x38` 的命令流版本只是回显，判定以 `STREAM_HEADER` 为准。
- 段内具体偏移由落地章定义；新增先在本节登记。

PSO 暂存区字段 → 偏移（dword 索引，第 6 章冻结，含 `depthBaseVa`，Q-34）：

| 字段 | 偏移 | 宽度 |
|---|---|---|
| `vsEntry` LO / HI | `0x80` / `0x81` | 2 dword |
| `psEntry` LO / HI | `0x82` / `0x83` | 2 dword |
| `vsSizeDwords` | `0x84` | 1 dword |
| `psSizeDwords` | `0x85` | 1 dword |
| `psOutSigma` | `0x86` | 1 dword |
| `depthBaseVa` LO / HI | `0x87` / `0x88` | 2 dword |
| `viewport.x` / `.y` / `.w` / `.h` | `0x89`~`0x8C` | 4 dword |
| `depthRange.min` / `.max` | `0x8D` / `0x8E` | 2 dword |
| `zNear` / `zFar` | `0x8F` / `0x90` | 2 dword |
| `depthFunc` | `0x91` | 1 dword（u8） |
| `primitiveTopology` | `0x92` | 1 dword（u8） |
| `flags` | `0x93` | 1 dword |

- `0x80~0xDF` 容量 = 96 dword（384 B）；第 6 章冻结占用 `0x80~0x93`（20 dword），`0x94~0xDF`（76 dword）保留，按"只增不改"追加。

## 5. MMIO 唯一硬件访问接口（ADR-037）

硬件配置、状态、统计统一走 MMIO 寄存器读写；单元内部成员全部私有，禁止外部直接访问（`roadmap.md` §0.7、§2.7）。

## 6. 与日志 / 错误的联动

- 错误定位需要 `{unit, packetType, dwordOffset, va, code}`（`../docs/design/logging.md`、`architecture-design.md` §3.6 错误契约）。
- 非法包 / 非法 ISA / 越界 = 硬件故障：记录 `{unit, packetType, dwordOffset, va, code}` 后**中止当前提交**；**初级不产生 `ResetRequired`**，复位仅第 36 章按引擎级接入（Q-30、Q-69）。

## 7. 契约变更

- 包类型、包头布局、stateId、MMIO 分段均为冻结契约，变更走 `roadmap.md` §7.4；包类型从预留区取号后**先登记再使用**。
- 包类型 / count / packet-specific、stateId 与 MMIO 的当前余量和扩容选项见 `bit-budget.md` §2~§3。

---

## 2026-11-21 评审同步说明

命令包、stateId 与 MMIO 分段以本文件为准。寄存器的具体偏移见 [第 10 章 §3.4](../stage2/10-device-ring-cp.md)。旧版载荷的退役流程见 [路线图 §7.4](../roadmap.md)。
