# 模块设计资料索引

当前教学任务见 [项目主干](../README.md) 与 [U0–U8 路线图](../roadmap.md)，共享边界见 [架构设计](../architecture-design.md)。本目录保留旧版全栈方案的模块细节、参数与测试样例；它们可用于设计练习，但不自动构成当前教学单元的前置任务或全局门禁。

| 教学问题 | 可取用的设计资料 | 当前用法 |
|---|---|---|
| 命令、完成与设备状态（U1/U6） | [command-mmio](command-mmio.md)、[boot-reset](boot-reset.md)、[bit-budget](bit-budget.md) | 选择最小 clear/draw/dispatch/fence 语义；旧偏移和位宽为具体实现例子 |
| 内存、资源与暂存（U3） | [memory-system](memory-system.md)、[texture-unit](texture-unit.md) | 讲解地址、缓存、数据复用和可见性 |
| 执行与协作（U2/U5） | [isa-abi](isa-abi.md)、[execution-parallel](execution-parallel.md)、[sync-primitives](sync-primitives.md) | 选用小型 wave/ISA、barrier/worker 练习 |
| 图形与计算驱动（U6/U7） | [compute-line](compute-line.md)、[compute-driver](compute-driver.md)、[compiler](compiler.md) | 借鉴对象关系、参数与分层；编译器可独立扩展 |
| 场景与教学观测（U0/U8） | [scene-specs](scene-specs.md)、[validation-model](validation-model.md)、[logging](logging.md) | 复用输入、参考模型、分级日志和诊断思路 |

[2026-11-21 评审同步记录](review-sync-20261121.md)解释旧版 42 章方案的冲突与修订历史。本版只把“图形/计算共享命令、资源、执行入口、完成与可见性”提升为必须互通的核心。选用旧文档中的某个具体 ABI 或常量时，请在相应 U 单元练习里写清输入、输出和适用范围。