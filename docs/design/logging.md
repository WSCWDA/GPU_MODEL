# 轻量日志、硬件统计与断言 — 独立模块设计

> 项目目标、范围、阶段与交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

> 定位：定义普通运行日志（`GD_LOG`）、MMIO 只读硬件统计计数器、以及三套断言体系。所有观测手段都只是调试手段，**CI 校验只信任探针 + `ref_model`**（`../docs/design/validation-model.md`）。
> 落地章：第 1 章轻量日志（ADR-010）；硬件统计计数器高级 A 阶段作为 MMIO 只读段落地（`../docs/design/command-mmio.md` §4 0x20~0x2F）。
> 权威来源：从 `architecture-design.md` §4.6 拆出，吸收日志体系定义（原 §7.6，已并入本文件）与 ADR-010。

---

## 1. 普通运行日志 `GD_LOG`（初级实现，ADR-010）

- 等级：`Error / Warn / Info / Debug / Trace`，统一格式 `[level][module][submodule] msg k1=val1 k2=val2`，纯 ASCII。
- 通道：控制台只输出 `Error/Warn` 与标志性 `Info`（由 `log_level` 控制）；文件按模块 + 聚合分层输出。
- 开关：`GD_LOG_LEVEL` 环境变量 + 配置键（`log_level`，`roadmap.md` §5.1）；默认 `Warn`；CI 只输出 `Error/Warn`；Debug/Trace 编译期可裁剪。
- 禁止：
  - 热点路径（逐像素 / 逐 wave 循环）严禁打 Debug/Trace 日志；
  - 日志参与判据；
  - 中文进入日志与注释（工程红线，跨平台一致性）。
- **计算定位字段**（错误记录 `{unit, packetType, dwordOffset, va, code}` 的扩展，2026-11-20 补齐）：计算引擎错误日志附加 `kernelVa / workgroupId / gtid / paramOffset` 四键，支撑 M-01/M-02"越界 work-item 不写且可定位"与 D-13 拒绝定位（`architecture-design.md` §3.12）。

## 2. 硬件统计计数器（高级 A，MMIO 只读）

- wave 执行数、cache 命中 / 失效、图元数、barrier 执行次数等硬件内部累加统计，经 MMIO 寄存器读取。
- **计算统计项**（2026-11-20 补齐，MMIO `0x20~0x2F` 错误/调试/性能段内分配，位宽登记见 `bit-budget.md` §3.2）：`workgroup 完成数`、`occupancy 拒绝计数`（D-13 静态拒绝）、`LDS bank 冲突计数`（仅软观测，不判正确性，`roadmap.md` §6.8）。
- `hw_counters_enable` 开关，关闭零开销；仅软观测，不参与门禁阻断。
- `state_inspector` 只读 MMIO、队列与统计，**不反向写状态**。

## 3. 断言三套体系

1. 编译期 `static_assert`：用于契约、结构体、枚举；
2. 开发期 `DEV_ASSERT()`：Release 编译直接移除，只检查内部不变量；
3. 硬件行为校验：Release/Debug 都执行，不直接 abort，返回 `Result` 错误码进入日志 + 复位路径。

红线：不要用原生 `assert()` 做硬件正确性校验——Release 版本硬件违例不能直接进程崩溃，走错误码 + 复位流程（`../docs/design/boot-reset.md`）。

## 4. 与 trace 的关系

`--trace` 仅供人工阅读，不保证字段稳定、不做机器解析契约、不进入硬判据（`architecture-design.md` §7.7 横向关注点、ADR-010）；结构化硬件 trace（json-line）**不纳入开发范围**。

## 5. 契约变更

- 日志格式、统计寄存器布局由落地章定义；作为契约的一部分，变更走 `roadmap.md` §7.4。
