# 设计评审剩余问题

本文跟踪 [设计文档问题评审（2026-11-20）](../设计文档问题评审-20261120.md) 与 [GPU 软模型设计评审改进建议](<../GPU 软模型设计评审改进建议文档.txt>) 中仍需处理的问题。沿用原评审编号。`BACK-*` 是对照[旧功能总览](https://github.com/WSCWDA/GPU_MODEL/blob/c80b321ce55ab7b6f0a167245d381b450cc7c8d3/docs/back/GPU_MODEL_%E8%AE%BE%E8%AE%A1%E6%80%9D%E8%B7%AF%E4%B8%8E%E7%AB%A0%E8%8A%82%E5%8A%9F%E8%83%BD%E6%80%BB%E8%A7%88.md)与现行设计时发现的待修项。文档问题以责任章节中的契约、任务和验收条件一致为关闭依据，实施验证见第 2 节。

## 1. 文档待修项

| 评审项 | 当前缺口 | 关闭判据 |
|---|---|---|
| P0-01 · Result 验收口径 | [第 1 章](../stage1/01-contracts-min.md)定义 16 个一期取值，[第 36 章](../stage2/36-error-paths.md) D36.1 仍要求“Result 枚举扩展冻结”，与本章末尾的退役说明并存。 | D36.1 引用公共 Result 快照，只验错误记录与已有状态的映射。 |
| P0-02 / P1-09 / P2-02 · MMIO | [第 2 章](../stage1/02-device-and-ring-min.md) §3.3 把 Doorbell 设为 `0x05`，`0x04` 给了 ring 写指针；[第 10 章](../stage2/10-device-ring-cp.md) §3.4 把 Doorbell 设为 `0x04`，写指针为 `0x00`。[第 3 章](../stage1/03-flat-memory.md)仍把 RT 尺寸写入 `0x30`，并在数据流中把宽高压成两个 16-bit 字段。D10.1 仍未明确两章各自的比对范围。 | 核对公共 MMIO 登记、初级子集、CLEAR 锁存和脚本判据。RT 宽高使用 `0xE0/0xE1` 的完整 32-bit 字段。`0x30` 保持设备/ring ID。 |
| P1-11 · 普通页错误与复位 | [地址与翻译](../stage2/11-translation-heaps.md)和[驻留](../stage2/14-residency-context.md)都规定页错误只中止当前提交。[第 36 章](../stage2/36-error-paths.md) §4.1 和 [S13](scene-specs.md)仍对“每类故障”要求 reset，而注入菜单包含未映射地址。 | 将普通页错误的记录、失败完成与复位类故障分开验收，并同步 T/D36 和 S13 对应入口。 |
| P1-30 · S3 覆盖判据 | [场景规约](scene-specs.md)中 S3 的覆盖结构探针为零容差。[第 25 章](../stage2/25-rasterizer.md) D25.2 对覆盖像素写 `±1/16`，同章 §7 又把 1× 覆盖率定义为零容差。 | 明确 S3 的覆盖集合零容差及颜色探针容差，并单列允许 `±1/16` 的子样本或诊断指标，使 D25.2 和章验收可直接执行。 |
| P1-45 · 复位后完成与引用 | [第 36 章](../stage2/36-error-paths.md) §3.2 仍写“in-flight fence 不再可等”及笼统的“复位即失效”。D36.3、[架构 §3.10](../architecture-design.md)及[同步原语 §2.1](sync-primitives.md)要求失败完成可被等待方观察，并保持用户引用与未受影响的提交引用。 | 将被丢弃提交、旧队列/Stream generation、已销毁的资源对象分别写成可测状态，覆盖复位前已等待、复位后再查询和其他队列继续推进。 |
| P2-04 · Flip 生效时机 | [第 13 章](../stage2/13-copy-display.md) §3.2 仍写“扫描线超过 flip 地址后才可见”，§4.3 和 D13.4 按 VBlank/下一 vsync 切换 front。 | 使用同一个生效点，并让 D13.3 能检查同一帧的扫描线来自同一个 front 快照。 |
| P2-16 · S18 输入与期待输出 | [第 31 章](../stage2/31-se-cu-array.md)把 S18 `case-1` 的 block 写成 `(8,8,1)`，[场景规约](scene-specs.md)使用 `(16,16)`。场景表给出 `8×8` 个 workgroup、每组 `16×16` 个线程以及 1,024 个输出元素，但未写明有效域或输出映射。`case-3` 沿用 `case-1`。 | 标明早期单测采用完整场景输入还是自选的小尺寸向量。冻结实际 dispatch、有效域及预期写入数量，使 `case-1/3` 可复用同一输入。 |
| A41/A42 · WA-02 回归范围 | [第 41 章](../stage3/41-workgroup-collab-indirect.md) §10 将 WA-02 全集交给第 42 章回归；[A42](../stage3/42-end-to-end-acceptance.md)中 S18 `case-5`（WA-02）保留为 HLI 回归，计算驱动归约由 C4 验收。 | 明确归约边界样例分别落在 C4 或 HLI 回归，并统一第 41 章待定项与第 42 章矩阵。 |
| BACK-01 · ISA 代码归属与升级路径 | [架构 §2.4](../architecture-design.md)将第 5、16 章的编码和解释器归入 `hw_sim/isa`；[第 5 章](../stage1/05-isa-min.md)却将解码和 `ToInstr` 放在 `hw_sim/shader/isa`，[第 16 章](../stage2/16-isa-assembler.md)改用 `hw_sim/isa`，章节间未说明复用或迁移。 | 明确第 5、16 章的编码、解码、指令语义和 wave 状态分别由谁负责。保留一套公共编码契约，并明确初级工具和解释器如何接入高级路径。 |
| BACK-02 · 35.1 前后的录包与提交边界 | [第 20 章](../stage2/20-resource-binding.md)和[第 33 章](../stage2/33-cmd-chain-multiengine.md)把资源绑定或命令生成放在 PAL 目录，[第 32 章](../stage2/32-queue-sync.md)甚至在 PAL 的 `Queue::Submit` 标注写 ring 和 doorbell；[架构 §3.4](../architecture-design.md)要求 PAL 经 `IHliCommandList`，且 35.1 前允许场景使用薄门面或测试支架。 | 指定正式 HLI 就绪前的资源、队列和命令支架及各自的共用转换代码；35.1 明确接入点，PAL 按对象接口记录，实际命令序列化和 ring/doorbell 的责任与 HLI、KMD/FW 分层一致。早期验收能独立执行。 |
| BACK-03 · 所有权移交顺序 | [第 32 章](../stage2/32-queue-sync.md) §3.3 在生产者队列先写 `SIGNAL`、再执行 release 屏障。消费队列因此可能在资源释放前通过 `WAIT`，与 [S16](scene-specs.md) 的所有权判据不一致。 | 生产者完成写入与 release 后发布信号，消费者依次完成 WAIT、acquire 和访问。S16 用延迟 release 的交错验证消费者不能提前读。 |
| BACK-04 · 屏障目标与验收范围 | [D20.9](../stage2/20-resource-binding.md)和 [S16](scene-specs.md)要求区分合法与非法子资源范围，并验证转移前后的访问。[命令包 §2.3](command-mmio.md)的三 dword 载荷只有类型、作用域、内存序、访问和阶段掩码，当前正文未规定资源、子资源范围或队列所有权如何关联到这条屏障。 | 结合教学范围，明确记录层校验、执行层状态及目标来源。若一期仅在执行层建模粗粒度屏障，判据需明确哪些检查在记录层完成，哪些场景验证队列移交与可见性。 |

## 2. 实施待验证项

- **P0-03 · 回归影响集**。待实现和 CI 落地后，按 [路线图 §7.4](../roadmap.md)验证契约变更会执行对应的 `R_n`、更新登记和快照，并且退役判据不会继续阻断门禁。
