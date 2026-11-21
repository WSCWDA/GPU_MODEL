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

本轮已将原评审里仍残留在章节正文的判据回写责任章节。实施顺序仍以 roadmap 依赖图为准；审查状态与风险如下：

本同步记录本身不能替代各章节的详细输入、输出、错误码和测试向量。第 1 章、MMIO/bit-budget、stage 章节中的旧条款若与上述决策冲突，旧条款状态为 `SUPERSEDED`，实现与评审使用本文件给出的唯一口径。代码实现状态仍需独立验证。


## 4. 结构重整后的 P0 复核（2026-11-21）

本节对原评审中 11 项 P0 按当前正文是否存在可执行的互斥要求复核。这里的“已闭环”须以权威正文已改写为准；仅追加说明、未删除旧 DoD 不计闭环。

| 问题 | 复核结果 | 主要同步位置 |
|---|---|---|
| P0-01 Result 冻结与扩展 | 已闭环：16 值为唯一一期集合，旧 11 值 DoD 已标退役 | stage1/01、architecture-design、roadmap §7.4 |
| P0-02 MMIO 地址表 | 已闭环：使用统一偏移表；stage1/02 只测本章登记子集，旧保留段断言退役 | design/command-mmio、stage1/02、stage2/10 |
| P0-03 冻结/扩展与前序 DoD | 文档侧已闭环：各章已改用明确 `R_n`，roadmap 不再把任一历史 `A<x>` 作为后续章的无差别全量门禁；旧 DoD 可经 ADR 标记 `SUPERSEDED`。实现侧 CI 影响集验证尚未执行 | README §17.2、roadmap §7.4、stage1~stage3 §7 |
| P0-04 BARRIER 多载荷 | 已闭环：包级 BARRIER 固定 3-dword envelope；ISA Workgroup BARRIER 明确为另一种执行原语 | design/command-mmio、stage2/20/30/32、stage3/41 |
| P0-05 block 上限 | 已闭环：maxThreadsPerGroup=512；513 及以上在记录期拒绝 | architecture-design §3.11、scene-specs D-13、stage2/37、stage3/39 |
| P0-06 间接参数布局 | 已闭环：唯一 28-byte/16-byte-aligned {grid[3],block[3],reserved0}；旧 12-byte grid-only 表述退役 | architecture-design、stage2/33、stage3/41、design/command-mmio |
| P0-07 37 与 39–41 重复工作 | 已收窄：37 只验硬件解析/执行，39 做 host 参数打包，40 做 stream/event API，41 做驱动协作包装；集成 DoD 验证组合路径 | stage2/37、stage3/39–41 |
| P0-08 LLVM 双后端逐字节判据 | 已闭环：改为 ABI/ISA 合法性与规范程序语义，允许合法编码差异，19 不阻塞 20 | stage2/19、design/compiler、design/isa-abi、roadmap |
| P0-09 CU ticket 锁串行化 | 已闭环：移除包围执行的 ticket 锁要求，改为原子唯一 slot claim、多个 worker 并发、race-free 结果一致 | stage2/31、design/execution-parallel |
| P0-10 同步模型无 watchdog tick | 已闭环：每 wave 退休指令预算 2^24 触发 Timeout；墙钟/tick 不参与判定；工作组完成只作进展观测 | stage2/36、design/boot-reset、design/review-sync |
| P0-11 第 35 章工作量 | 已收窄：HLI 核心对象面必需；三套 API-inspired adapter 按 capability 选择用例且可选，不要求完整 E/S 或厂商兼容 | README WP6、stage2/35、roadmap §0.5 |


## 5. 两份评审文档对新结构的逐项复核（2026-11-21）

复核范围：`设计文档问题评审-20261120.md` 中 P0/P1/P2 与任务分工建议，以及 `GPU 软模型设计评审改进建议文档.txt` 的契约治理、MMIO、计算分层、编译器、并行、资源绑定、阶段门禁和错误复位建议。结论：**新目录和主干分工大体合理，但重组本身没有消除章节正文里的旧冲突。** 本轮仍找到查询口径、LOD、Context/Event 生命周期、reset replay、A42 case 覆盖等残留问题，并已回写。

### 5.1 P1 复核结果

| 评审项 | 复核结论 | 主要落点 |
|---|---|---|
| P1-01~20 | 已回写可裁决项：配置/stateId/opcode、Submit/CP、front 快照、缓存验收、容量/责任边界、CompilerResult 映射与 scratch 上限。 | `README.md`、`architecture-design.md`、stage1/02/05/08、stage2/10~20、design/compiler、memory-system |
| P1-21~30 | 已明确纹理尺寸参与 LOD、索引边界、IA/VGT 分工、完整图元/patch 切片、距离 ABI 与结构/颜色探针容差。 | stage2/21~28、`isa-abi.md`、`scene-specs.md` |
| P1-31~40 | 已固定 8×8 Hi-Z 块、Tess 因子与 patch staging、LDS/STS 和 SubOp 登记、作用域、PREDICATE 跳过范围、深度通过 sample 查询口径与 ULP 阻断规则。 | stage2/27~34、`command-mmio.md`、`isa-abi.md`、`roadmap.md` |
| P1-41~45 | 已裁决多 Context/ASID、Context 与 Stream 销毁边界、Event 自有 timeline semaphore、A42 覆盖 S19 case-0..4、reset 不隐式 replay。 | `compute-driver.md`、stage3/38/40/42、`boot-reset.md`、stage2/36 |

### 5.2 对《GPU 软模型设计评审改进建议文档》的问题域复核

| 问题域 | 新结构中的判断 | 说明与边界 |
|---|---|---|
| 契约治理 | 文档侧已闭环 | `Result` 终版与集中登记规则已写入；跨章变更走 ADR、权威契约、影响集更新和 `SUPERSEDED`；各章改用 `R_n`，roadmap 只按硬依赖检查前序 DoD，不自动继承所有历史 A 门禁。实现侧的编号审计仍待代码落地。 |
| MMIO / stateId | 主要冲突已闭环 | 唯一寄存器/包契约与 bit-budget、stage1/stage2 引用统一；`stateId 0~7` 冻结，扩展状态走内存间接表。未采用 512 dword / 双 BAR 重构，这是软模型一期的范围选择。 |
| 计算分层 | 已按任务表硬切分 | 29~34 只做可复用共用硬件单元的局部用例，不提前验收 S18/S19；37 收口 Compute 硬件通路；38~42 负责计算驱动对象、ABI 与双入口验收，不新造 HLI/硬件接口。 |
| 编译器双后端 | 冲突已消除 | 19 章比较 ABI/ISA 合法性和规范程序语义，不要求二进制相同；第 20 章可依据冻结的手写后端/ABI 契约并行开工。 |
| 并行执行 | 旧 ticket 串行化方案已移除 | 以唯一 slot claim 和多个 worker 并行；硬判据限定 race-free 负载与明确定义的原子语义，不要求未定义竞争程序有固定结果。 |
| 资源绑定 | 一期无互斥定义，保留扩展取舍 | 根槽与内存间接描述符满足本期能力；统一 Bindless 全 VA 化、资源列表屏障未作为一期交付，属于后续架构选项，不再与当前能力并列作为硬要求。 |
| 阶段门禁 | 历史全量 A 门禁问题已在文档侧修订 | 前序验收集按 `R_n` 和受影响场景明确；D 门禁收口 S10/S11，F 门禁主验收 S18/S19；19 不阻塞 20；E 阶段不要求尚未交付的 S18/S19。roadmap 依赖表仍需在实现前转成 CI 检查。 |
| 错误与复位 | 保留必要支撑能力，不作为主展示线 | 指令预算、确定性中止和显式重建/重提已定义；不采纳完整 RAS/故障注入系统作为一期主任务。错误路径不替代图形/计算 UMD、FW 衔接与内存管理的主线。 |

### 5.3 新结构中的剩余风险

- 项目文档中的 `R_n` 与 ADR 流程已明确，但代码、CI 是否按影响集运行回归尚未验证。
- 一期资源描述符容量和 API-inspired adapters 仍有限；这属于声明过的能力边界，新增能力需另行登记，不能默认扩展为厂商兼容。
- 并行结果一致性仅对 race-free 用例与已定义原子/同步语义成立，不能把任意竞争程序要求为确定结果。

### 5.4 P2 复核

本轮同时清理过期章数、Doorbell 图示、AS6 与 SSA 字段笔误、ISA 语义责任、切片占位文字、dontCare 越界条件、Hi-Z/PS 执行频率术语、递减 SIGNAL 行为、DMA 独立源/目标步长与失败原子性、case 锚定引用和 A42 回归范围；并将发现的 P2 口径逐项回写至权威章节。其他局部事项仍按 `scene-specs.md` 和模块契约在实现前复核。

本节是文档审查与同步，不代表实现完成或测试通过；代码状态由对应章 DoD 单独验证。
