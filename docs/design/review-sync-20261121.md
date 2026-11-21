# 设计评审同步裁决（2026-11-21）

> 本文是设计同步记录，不是项目范围、接口或场景判据的新增权威来源。项目范围以任务规划为主干，专业细节继续由架构、模块、场景和 roadmap 对应文档维护。

> 状态：本文件是对 `设计文档问题评审-20261120.md` 与 `GPU 软模型设计评审改进建议文档.txt` 的裁决记录，不是待选方案。与旧章节冲突时，以本文件及下列同步后的权威文档为准。本文只裁决文档契约；不代表代码已实现或测试已通过。

## 1. 总体判断与任务划分

当前分层方向合理：`hw_sim → firmware/KMD → HLI → UMD → apps`，CP 按包类型派发、图形与计算复用硬件单元、单一场景规约均应保留。主要缺陷是契约没有统一登记、章节任务/DoD 没有落实分层边界、同一功能多章重复验收，以及前序 DoD 没有退役机制。

| 交付边界 | 唯一负责内容 | 不再重复交付 |
|---|---|---|
| 第 1 章 / design | 类型、枚举、包、MMIO、stateId、ABI 等全局契约登记 | 后续章节私自定义第二套契约 |
| 第 10 章 | KMD/FW 启动、环、CP、单队列提交及 MMIO 首次落地 | 多队列策略（第 32 章） |
| 第 29–31 章 | 可复用执行原语、LDS/同步、SE/CU 调度及并行执行模型 | 第 37/41 章再实现一份 wave/barrier 硬件 |
| 第 32–36 章 | 多队列/IB、查询、HLI/API、故障复位等系统能力 | 把异步/错误行为复制进计算驱动章节 |
| 第 37 章 | Compute 硬件入口、CP→Compute engine 派发、硬件侧资源与指令执行；通过 HLI 直连验硬件通路 | kernel 参数打包（39）、stream/event API（40）、驱动侧三维映射/间接/UAV/query 包装（41） |
| 第 38–42 章 | 38 上下文；39 launch/参数布局；40 stream/event；41 驱动侧协作 API 与 indirect/UAV/query 包装；42 双入口端到端等价 | 重复实现第 37 章硬件单元 |
| 第 35 章 | HLI 是一期必需；外部 API 前端作为独立、可选适配器，按能力矩阵逐个交付 | 一次性要求 Vulkan、D3D12、GL 三套完整兼容实现 |

任务粒度统一为契约/接口、主体实现、接入、验收四段；每个任务应能在不超过约 2 人日内独立评审。第 35 章保留章节编号以免全局重排，但将三套 UMD 分成 35.2/35.3/35.4 子里程碑；一期只要求 HLI，Vulkan 仅实现明确列出的子集时须标注“API-inspired，不宣称兼容”，D3D12/GL 默认 `NotSupported`，不阻塞核心验收。场景迁移按场景逐步双入口，不能整体切换。

## 2. 已冻结的跨章裁决（替代旧条款）

### 2.1 Result 与契约演进（P0-01、P0-03）

`Result` 最终一期集合共 16 值：原 11 值加 `Busy`、`OutOfStack`、`RegisterOverflow`、`InvalidIr`、`Timeout`。第 1 章枚举数量、快照与 DoD 应按此集合更新。错误细节仍由 `ErrorRecord`/模块错误对象提供，不继续向全局枚举添加细分错误。今后变更必须登记 ADR、同步全局登记表、列明受影响判据，并将旧 DoD 标为 `SUPERSEDED → 新判据`；不得既要求新契约又保留旧判据为阻断项。每章回归仅检查该章登记的合同和能力，不得把“保留空间全部 NotSupported”“枚举数量永久不变”写成全局断言。

### 2.2 MMIO 唯一表（P0-02、P1-04、P1-09）

所有偏移以 dword index 计，唯一映射如下：`0x00 RING_WPTR`、`0x01 RING_RPTR/HEAD`、`0x02 RING_SIZE`、`0x04 DOORBELL`、`0x08 FENCE_MON`；`0x10 CP_CTRL`、`0x14 INT_STATUS`、`0x18 CTX_STATE`；`0x20 ERROR_CODE`、`0x21 ERROR_PKT_TYPE`、`0x22 ERROR_DWORD_OFFSET`、`0x24 DEBUG0`、`0x28 HW_STATS_BASE`；`0x30 DEV_RING_ID`、`0x38 CMD_VERSION`；`0x40–0x4F stateId 0–7`；`0x50–0x7F`保留；`0x80–0xDF`为 PSO 暂存区；`0xE0 RT_WIDTH`、`0xE1 RT_HEIGHT`，`0xE2–0xEF`为上下文/队列/SE 统计；`0xF0 RESET_CTRL`，其余系统保留。RT 宽高分别完整 32 bit，不再复用 0x30。第 2 章仅断言自身登记的寄存器子集；第 10 章映射表是硬件寄存器权威；第 32 章的一期实现沿用直连 ring 寄存器，内存 ring 描述表与活动窗口切换降为后续 capability。

### 2.3 BARRIER 包唯一载荷（P0-04）

`PKT_TYPE=0x0E` 固定为 3 dword：dword0 按字节编码 `kind, scope, memOrder, accessMask`；dword1 为 `srcStageMask`；dword2 为 `dstStageMask`。`kind`: resource=0、execution=1、ownership=2；scope/memOrder/access 的枚举按 `sync-primitives.md` 登记；未知值/位返回 `InvalidPacket`。Workgroup barrier 的执行点仍是 ISA/LDS 同步原语，包只承载 CPU/CP 可见的资源、设备或队列级依赖。第 20/30/32/41 章只能使用此信封，不得另定义 payload。

### 2.4 Compute 容量及 indirect ABI（P0-05、P0-06）

`WARP_SIZE=32`、`maxWavesPerGroup=16`，故 `maxThreadsPerGroup=512`。记录期检查 `block.x*block.y*block.z ≤ 512`；1024 线程样例改为 513 或 `(32,32,1)` 并预期 `InvalidArgument`。显式登记容量项，任何提升上限须同时重新证明 wave/CU、LDS、barrier 驻留条件。SubOp `DISPATCH_INDIRECT(0x0010)` 唯一输入为 28 字节、16 字节对齐的 `{grid[3], block[3], reserved0}`（7 个 u32）；reserved 必须为 0。指针字段对齐按 ABI 规定；直接/间接路径对相同参数必须语义等价。

### 2.5 计算工作分层（P0-07）

第 37 章任务表中与 T39/T40/T41 重复的参数打包、三维驱动封装、驱动侧 barrier、stream/event、indirect、query、UAV API 项全部标记 `SUPERSEDED → 第 39/40/41 章`。第 37 章保留硬件 Compute engine、命令解释、波/workgroup 执行、资源访问及硬件级验收；第 39/40/41 章只做驱动包装和既有硬件能力组合。A37 验收直连 HLI 硬件路径；A42 验收直连 HLI 与计算驱动入口的端到端等价，禁止两处对相同功能重复承担实现 DoD。

### 2.6 编译器双后端判据（P0-08）

两套后端不要求逐字节生成相同二进制。L1 验证模块/符号/ABI/合法编码；L2 以规范测试程序运行结果对照 CPU 参考模型，逐值验证确定语义；L3 记录代码尺寸、寄存器压力等质量指标，不作阻断门槛。允许寄存器分配、NOP 和合法指令选择不同；规范化 IR/指令语义必须一致。第 19 章不再阻塞第 20 章对象层开发。

### 2.7 并行执行与 watchdog（P0-09、P0-10）

移除以 ticket/serving 锁包住 CU 全部执行的串行化方案。并行模式允许多个 CU/workgroup slot 同时由 CPU worker 执行；同一 CU 的 wave 通过确定性指令步进/ready queue 交错推进，以便跨 wave barrier 可达。静态 workgroup 编号保证输出索引稳定，不承诺调度顺序相同；race-free 用例结果必须一致，原子竞争只按明确定义的原子语义验收。串行模式保留为参考实现。watchdog 不读取墙钟；每 wave 退休指令计数，达到 `kMaxInstructionsPerWave=2^24` 返回 `Timeout`，错误含 wave/PC/dispatch 标识，并走引擎复位路径。

## 3. P1/P2 同步规则与逐章修订项

下列评审问题纳入本次同步，不得默认为已实现；受影响章节提交实现前必须按这里收口：

- P1-01/02/05/10–12/14–20/34–45：配置键、stateId、opcode、缓存、容量、语义和 DoD 的修订以第 1 章、bit-budget 及对应单章契约为准；将每条判据写成可机器判定且只覆盖本章登记集合。
- P1-06/21–33、P2-03/06/09–11/13：固定功能与描述符、LOD 尺寸、索引越界、patch 切片、容差、LDS/STS 和 DMA stride 的章节内定义须补全后方可进入对应阶段；旧判据与新口径并存时标记 SUPERSEDED。
- P1-03/07–09/13/17–19/40–45、P2-02/04/14–18：前向依赖必须在 roadmap 标注硬依赖或软依赖；端到端门禁依赖未交付模块时不得挂在前序章；S18/S19 case 只在 scene-specs 设锚定定义；A42 覆盖范围与目标一致。
- P2-01/05/07/08/12/15：过期章数、双错误映射、类型笔误、后端判据级别、递减 fence 语义及空指针契约在相应章节修订时一并清理。

本同步记录本身不能替代各章节的详细输入、输出、错误码和测试向量。第 1 章、MMIO/bit-budget、stage 章节中的旧条款若与上述决策冲突，旧条款状态为 `SUPERSEDED`，实现与评审使用本文件给出的唯一口径。代码实现状态仍需独立验证。
