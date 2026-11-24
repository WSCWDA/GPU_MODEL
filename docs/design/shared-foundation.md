# 公共组件与渐进契约

项目范围、工作包和正式场景以 [README](../README.md) 为准。本页说明哪些设计需要提早确定，哪些功能在有实际消费者时才实现。模块内部机制继续在各自设计文档维护。

## 1. 放在 common 的内容

| 公共内容 | 所有者和使用方 | 专业细节 |
|---|---|---|
| 地址、句柄、Result 和格式值类型 | common 定义跨层值。HW 以 GPU VA 请求内存，KMD 管映射，HLI 管对象生命周期 | [架构 §3](../architecture-design.md)、[内存系统](memory-system.md) |
| 命令头、编号、编码与解码 | common 持有已登记的共享格式，HLI 或初级薄门面编码，CP 解码和派发 | [命令与 MMIO](command-mmio.md)、[位预算](bit-budget.md) |
| ISA、Shader ABI 与格式转换 | common 共享字段和数据表示，编译端产生载荷，着色执行器消费 | [ISA 与 ABI](isa-abi.md)、[编译器](compiler.md) |
| 完成标识和错误记录形状 | common 定义共享值，队列持有完成状态，调用方解释错误。日志只读观测值 | [同步原语](sync-primitives.md)、[日志与断言](logging.md) |

具体对象的生命周期、调度和存储状态各由一个运行时组件维护。多个章节使用同一机制时，沿用公开接口。需要跨层交换或跨模块保持一致的值类型、序列化与格式转换可提取到 common。额外的软件层或硬件状态由首次用到它的场景引入。

## 2. 提前确定的边界与实施顺序

| 边界 | 提前确定 | 能力启用时实现 | 核对示例 |
|---|---|---|---|
| GPU 地址 | 区分 HostVa、GpuVAddr、PhysAddr。公共 GPU 地址为 u64，设备有效 VA 为 36 bit。明确地址单位、范围检查与错误责任 | 第 3 章用单地址空间和容量内直映射。第 11 章接两级页表、TLB 和堆，第 14 章接 ASID、驻留与上下文切换 | 同一上传/读回接口在 flat 和 paged 路径可用。高位或跨范围访问可报告错误 |
| 编码位和编号 | 稳定字段宽度、单位、版本和所有者。命令包顶层号 4 bit，预定扩展入口 EXTENDED。ISA opcode 7 bit，MMIO 按 dword，stateId 直接槽 0~7 | 初级只编解码本阶段用到的包和字段。后续按 [命令登记](command-mmio.md) 和 [路线图 §7.3](../roadmap.md) 启用实际子操作码或寄存器 | 审计只覆盖本次使用和影响的范围。保留号与已登记但未启用的能力分别处理 |
| 提交与完成 | 区分接受、执行结束、失败和数据可见。命令携带足够身份供定位 | 第 2 章同步推进单环。第 10 章拆出 KMD/FW 和完成通知。第 32 章接多逻辑队列、timeline 和资源转移 | 单环序列可重现，跨队列消费者由显式依赖控制 |
| 执行与局部存储 | lane 身份、活动掩码、设备地址与局部地址各有定义。组内原语和队列同步各有入口 | 单 wave 串行起步。第 28 章用 patch 缓冲。第 30 章启用 LDS/barrier，第 31 章用 worker 承载已定义的工作 | 串行支架可核对掩码与组同步，宿主线程变化后检查相同的硬判据 |

编码登记包含所有者、单位或有效位宽、首个消费者、启用阶段、保留值和不支持行为。审计容量时区分编号空间与实现容量。36 bit 地址空间并不要求一次性分配 64 GiB，预留 opcode 也无需实现占位引擎。现行数值见专业登记，历史方案仅用于理解裁决。

## 3. 代码中的教学简化说明

实现中在做出简化的函数、状态机或数据结构旁写短注释，说明它为何足以支持本阶段原理。对后续扩展写出增加的状态、转换条件和接入模块，引用对应章节。无需把整份设计复制到注释中。

| 实现点 | 注释应解释的当前行为 | 扩展应补齐的状态或责任 |
|---|---|---|
| flat Translate | 单地址空间，GPU VA 在已分配范围内直映射 | 页表根、PTE 权限和 TLB 失效由第 11 章接管，多上下文由第 14 章接入 |
| 薄提交门面 | 同步推进 ring 直到成功或失败，Wait 验证完成 | KMD/FW 提交所有权、独立队列的等待和完成状态 |
| single-wave runner | 固定 lane 映射，已定义顺序推进 | 分歧栈、就绪/挂起、组到齐、成组驻留和 worker 领取 |
| patch 暂存 | 完整保存控制点直到消费完成 | 如扩展跨批次复用，补足容量、所有权、释放与背压 |
| 复位支持 | 当前提交失败后可定位，显式重提使用新队列 | 扩展到更复杂的恢复时需要记录可恢复状态、停止点和重放条件 |

示例可放在平坦内存翻译函数旁。

```cpp
// Teaching model: one address space with identity translation after a range check.
// The paged implementation supplies a page-table root, checks PTE permissions,
// and invalidates cached translations after remapping. Keep Read/Write callers
// on IMemorySubsystem so the implementation can change at that boundary.
```

验收时可运行最小输入并解释停顿、返回值或输出。只有共享边界行为改变时才更新受影响的调用方和回归集。局部数据结构或可观测统计可在各自设计中演进。
