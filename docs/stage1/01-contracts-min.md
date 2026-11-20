# 第 1 章 契约最小集

> 本章是全项目契约的起点：后续 35 章的结构体、枚举、命令包头、错误码、像素格式与字节序、日志格式、确定性规则，均以本章冻结的条款为准。本章不含任何可执行硬件行为，只交付头文件、编译期断言与自检场景。
> 与本章相关的设计说明：`../architecture-design.md` §3.9（错误契约）、`../design/command-mmio.md` §1~§2（包头与包类型）、`../design/isa-abi.md` §1~§3（编码与 ABI 分区）、`../design/logging.md` §1/§3（日志与断言）、`../design/memory-system.md` §1（三类地址）。
> 落地模块：`common/include/common/{types,result,packet,format,log,config}.h`、`apps/scene_e0_selftest`。

---

## 1. 目标与非目标

### 1.1 目标

1. 冻结**基础类型与句柄**：整数/浮点别名、`GpuVAddr`/`PhysAddr`/`HostVa`、`ContextId`、`EngineType`、`Stage`、`WARP_SIZE`、`LaneMask`。
2. 冻结**错误契约**：`Result` 枚举的全部取值与跨层返回约定。
3. 冻结**命令包头**：16 字节内的 4 bit 类型 + 12 bit 计数布局、包类型登记表、每个命令流首包 `STREAM_HEADER` 及其版本校验入口。
4. 冻结**像素与颜色契约**：RT 格式 `R8G8B8A8_UNORM`、内存字节序 BGRA、Clear 色编码、浮点→字节的截断口径。
5. 冻结**日志与断言基座**：`GD_LOG` 格式与等级、`static_assert`/`DEV_ASSERT`/硬件行为校验三套断言的分工。
6. 冻结**配置键与判定分类**：全部运行期配置键的名称、取值、默认值，以及"硬判据 / 软观测"的定义。
7. 交付 E0 自检场景，使上述冻结项在 CI 中逐条可判。

### 1.2 非目标

- 不实现设备、MMIO、环、内存、执行、光栅、显示（第 2 章起）。
- 不定义具体 ISA 指令与编码细化（第 5 章初版、第 16 章冻结），本章只冻结编码位宽与 opcode 区段边界。
- 不引入编译器、不引入计算通路、不做 API 兼容层。
- 不实现任何容器、线程、文件 IO 抽象（`umd/os` 属高级阶段）。

---

## 2. 单元框图

```
┌───────────────────────── common（纯头文件 + 少量 constexpr） ─────────────────────────┐
│ types.h    标量别名 / 地址类型 / ContextId / EngineType / Stage / WARP_SIZE / LaneMask │
│ result.h   Result 枚举 + ToString + 错误记录结构 ErrorRecord                            │
│ packet.h   PacketHeader 位域 / PktType 登记表 / STREAM_HEADER 载荷 / 版本校验           │
│ format.h   PixelFormat / 内存字节序 / ClearColor / FloatToUnorm8                        │
│ log.h      LogLevel / GD_LOG / DEV_ASSERT / GD_CHECK（硬件行为校验）                    │
│ config.h   ConfigKey 清单 / 默认值 / 取值域 / 判定分类（硬判据|软观测）                 │
└───────────────────────────────┬──────────────────────────────────────────────────────┘
                                │ 编译期断言 + 常量
                                ▼
                    apps/scene_e0_selftest（E0）
                    断言结构体尺寸/枚举取值/包头往返/颜色换算/日志格式/配置键完整性
```

本章所有单元都是**值类型与常量**，没有运行时状态机；"数据流"体现为 E0 的断言流水线（§4）。

---

## 3. 输入输出契约

### 3.1 基础类型（`types.h`）

| 符号 | 定义 | 约束 |
|---|---|---|
| `u8/u16/u32/u64`、`i32/i64`、`f32` | 定宽整数与 32 位浮点 | `static_assert` 校验尺寸 |
| `GpuVAddr` | `u64` | 硬件单元唯一可用的地址类型 |
| `PhysAddr` | `u64` | 只在页表内部出现（第 11 章起） |
| `HostVa` | `void*` | 只允许出现在 KMD/UMD/工具层 |
| `ContextId` | `u32` | 0 为无效上下文 |
| `EngineType` | `enum class : u8 { Graphics3D, Compute, Copy, Display }` | 与包派发目标一一对应 |
| `Stage` | `enum class : u8 { Vertex, Hull, Domain, Geometry, Pixel, Compute }` | `Compute` 在本章即冻结，暂无人消费 |
| `WARP_SIZE` | `constexpr u32 = 32` | 固定不开放（ADR-018） |
| `LaneMask` | `u32`、`kLaneMaskAll = 0xFFFFFFFF` | 取反后必须与 `kLaneMaskAll` 相与 |

- `Stage` 唯一定义在 `common/types.h`；`namespace shader` 只做 `using common::Stage;`，不得重复定义（Q-33）。

### 3.2 错误契约（`result.h`）

`Result` 共 16 值：`Ok / InvalidArgument / NotSupported / NotMapped / NotResident / InvalidPacket / InvalidIsa / OutOfMemory / DeviceLost / ResetRequired / Internal / Busy / OutOfStack / RegisterOverflow / InvalidIr / Timeout`。此集合按 ADR 快照冻结；细分诊断放入错误记录，不新增全局返回值。

- 所有跨层接口返回 `Result`，禁止异常跨层传播。
- 错误记录结构：

```cpp
struct ErrorRecord {
    EngineType unit;        // 出错单元
    u32        packetType;  // 包类型，非包路径填 0xFFFFFFFF
    u32        dwordOffset; // 包内偏移
    GpuVAddr   va;          // 相关地址，无则 0
    Result     code;
};
```

- `NotSupported` 只用于二期/范围外能力；不得用 `Ok` 掩盖未实现。

### 3.3 命令包头与包类型（`packet.h`）

```
 31        28 27                 16 15                        0
┌──────────┬──────────────────────┬───────────────────────────┐
│ PKT_TYPE │ count（含头，dword） │ packet-specific / opcode  │
└──────────┴──────────────────────┴───────────────────────────┘
```

- `count` 12 bit，单包最大 4095 dword；包不得跨环尾，空间不足以 `NOP` 填充（第 2 章执行）。
- 包类型登记表（本章冻结编号，落地章只在已登记编号上实现）：

| PKT_TYPE | 名称 | 落地章 |
|---|---|---|
| 0x00 | `STREAM_HEADER` | 1 |
| 0x01 | `FENCE` | 2 |
| 0x02 | `SET_STATE_BASE` | 3 |
| 0x03 | `DRAW` | 6 |
| 0x04 | `DISPATCH` | 37（原 35.5） |
| 0x05 | `COPY_MEM` | 3 |
| 0x06 | `SHADER_PSO` | 6 |
| 0x07 | `NOP` | 1 |
| 0x08 | `CLEAR` | 3 |
| 0x09 | `QUERY` | 34 |
| 0x0A | `SIGNAL` | 32 |
| 0x0B | `WAIT` | 32 |
| 0x0C | `INDIRECT` | 33 |
| 0x0D | `PREDICATE` | 33 |
| 0x0E | `BARRIER` | 32 |
| 0x0F | `EXTENDED` | 顶层号仅剩此号；新命令**不再抢占顶层号**，走 `EXTENDED`(0x0F)+ 子操作码（`../design/bit-budget.md` §2.3）或 `INDIRECT`(0x0C) IB，先登记再使用（ADR-043/045） |

- `STREAM_HEADER` 载荷：`{u32 version; u32 engine; u32 flags;}`，初级 `STREAM_HEADER.version=1`；高级升为 2 并拒绝 v1（Q-28）。不匹配返回 `InvalidPacket` 并携带 `{packetType, dwordOffset}`。
- 扩展规则：新增字段追加在载荷尾部；解析按 `count` 截断，未用字段视为 0。
- **扩容规则（ADR-043/045，`../design/bit-budget.md` §2）**：`PKT_TYPE` 4 bit 已冻结、已登记 15 个（0x00~0x0E），顶层仅剩 `0x0F`；新增命令一律不新增顶层包号，优先①载荷 `SubOp` 子操作码（`0x0F=EXTENDED` + 16 bit packet-specific）②`INDIRECT`(0x0C) IB ③必要时 0x0F 定义转义。任何新编号先对照 bit-budget 审计再在 `../roadmap.md` §7.3 登记（编号 `B-nnn`）。
- 错误不走包类型 0xFF（类型域只有 4 bit），统一走 MMIO 错误码 + `GD_LOG`。

### 3.4 像素与颜色（`format.h`）

| 项 | 约定 |
|---|---|
| RT 格式 | `R8G8B8A8_UNORM` |
| 内存字节序 | `[B, G, R, A]`（ADR-015），Clear/光栅/OM/Present/校验全栈一致 |
| Clear 色编码 | `u32 0xAARRGGBB`；背景色 `0xFF3F3F3F` |
| 浮点→字节 | `clamp(x, 0.0, 1.0) * 255` 后**截断**取整（0.25→63、1.0→255） |
| 全项目背景 | `(R,G,B,A) = (63,63,63,255)`，内存 `[63,63,63,255]` |

### 3.5 日志与断言（`log.h`）

- 等级：`Error / Warn / Info / Debug / Trace`；单行格式 `[level][module][submodule] msg k1=val1 k2=val2`，纯 ASCII。
- 控制台只出 `Error/Warn` 与标志性 `Info`；`Debug/Trace` 编译期可裁剪。
- 三套断言：
  1. `static_assert`：契约、结构体、枚举；
  2. `DEV_ASSERT()`：内部不变量，Release 移除；
  3. `GD_CHECK(cond, record)`：硬件行为校验，Release 也执行，不 abort，返回 `Result` 并写 `ErrorRecord`。
- 红线：不得用原生 `assert()` 做硬件正确性校验；热点循环禁止 `Debug/Trace` 日志；日志与注释一律 ASCII。

### 3.6 配置键（`config.h`）

| 键 | 取值 | 默认 | 判定分类 |
|---|---|---|---|
| `exec_mode` | `serial` / `parallel` | `serial` | 硬判据（本章只接受 `serial`，`parallel` 报"策略 B 未实现"） |
| `worker_threads` | `1..N` | 1 | 硬判据 |
| `se_count` | `auto`/`1`/`2`/`4`/`8` | `auto` | 硬判据 |
| `warp_size` | 固定 32 | 32 | 硬判据（只读） |
| `cache_enable` | `on`/`off` | `on` | 横切差分（开关前后结果必须一致） |
| `rt_size` | `256x256`/`1080x720`/自定义 | `256x256` | 硬判据 |
| `api_backend` | `hli`/`vulkan`/`d3d12`/`gl` | `hli` | 硬判据 |
| `trace` | `off`/`<unit>` | `off` | 软观测（仅人工） |
| `dump_raw_output` | `off`/`on` | `off` | 软观测（诊断制品） |
| `max_allowed_bad_pixels` | `0..8192` | `256` | 软观测（只告警） |
| `kft_allow` | `off`/`on` | `off` | 软观测（只告警） |
| `log_level` | `Error`/`Warn`/`Info`/`Debug`/`Trace` | `Warn` | 软观测 |
| `hw_counters_enable` | `on`/`off` | `off` | 软观测 |
| `mem_sanitizer` | `on`/`off` | `off` | 软观测（调试工具） |
| `tile_w` | `1..128` | `4` | 硬判据 |
| `tile_h` | `1..128` | `4` | 硬判据 |
| `translation.seed` | `u32` | `0x5EED` | 硬判据（地址翻译确定性种子，第 11 章使用） |
| `memory_budget` | `auto`/`<MiB>` | `auto` | 硬判据（第 14 章驻留/逐出使用） |
| `simd_per_cu` | `1`/`2`/`4` | `1` | 硬判据（第 31/37 章使用） |

**判定分类定义**：硬判据 = 任何取值下必须完全一致的数值/字节/顺序；软观测 = 允许波动、只记录、不阻断。全项目只允许这两类进入验收，禁止以"日志看起来对""耗时合理"作为判据。

---

## 4. 内部数据流

本章没有运行时流水线，E0 的执行流程固定为六段断言，任一段失败即退出码非 0：

```
[1] 类型与尺寸断言   → sizeof(u32)==4、GpuVAddr 为 8 字节、WARP_SIZE==32
[2] 枚举完整性断言   → Result 取值数 = 16、与 ADR 快照相等且无重复；EngineType/Stage/PktType 唯一
[3] 包头往返         → 构造 header → 序列化 → 解析 → 字段全等；count 边界 4095/4096
[4] 版本校验         → version=1 通过；version≠1 返回 InvalidPacket 且记录 packetType/dwordOffset
[5] 颜色换算         → 0.25→63、1.0→255、-0.1→0、1.5→255；0xFF3F3F3F 拆解为 [63,63,63,255]
[6] 日志与配置       → 格式串匹配；log_level=Warn 时 Info 不输出；配置键数量与默认值全等
```

---

## 5. 任务清单

| 任务 | 段 | 内容 | 产出文件 |
|---|---|---|---|
| T1.1 | ① 契约 | 基础类型、地址类型、`ContextId`、`EngineType`、`Stage`、`Result`、`ErrorRecord` 落头文件，补编译期断言 | `common/types.h`、`common/result.h` |
| T1.2 | ② 主体 | 包头位域、包类型登记表、`STREAM_HEADER` 载荷与版本校验函数（含 `packetType/dwordOffset` 报错） | `common/packet.h` |
| T1.3 | ② 主体 | 像素格式、BGRA 字节序、Clear 色、浮点截断函数 | `common/format.h` |
| T1.4 | ③ 接入 | `GD_LOG`、三套断言、配置键表接入 E0 场景骨架（场景可编译可运行，输出断言摘要） | `common/log.h`、`common/config.h`、`apps/scene_e0_selftest/main.cpp` |
| T1.5 | ④ 验收 | 跑通 A1：E0 全绿 + 冻结清单与代码逐项比对 | `apps/scene_e0_selftest`、基线输出 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D1.1 | `static_assert` 全通过：定宽类型尺寸、`GpuVAddr`/`PhysAddr` 8 字节、`WARP_SIZE==32`；`Result` 恰 16 个取值且与 ADR 快照一致、无重复；`EngineType`/`Stage` 无重复值；`-Wall -Wextra` 零警告 | L1 契约 |
| D1.2 | 包头序列化/反序列化往返字段全等；`count=4095` 可解析、`4096` 被拒；`PktType` 枚举值与 §3.3 表逐项相等（脚本比对）；`version=2` 返回 `InvalidPacket` 且 `ErrorRecord` 中 `packetType=0x00`、`dwordOffset` 精确指向版本字段 | L1 契约 |
| D1.3 | `FloatToUnorm8(0.25)==63`、`(1.0)==255`、`(-0.1)==0`、`(1.5)==255`；`0xFF3F3F3F` 按 `[B,G,R,A]` 拆解得到 `(63,63,63,255)`；`PixelFormat` 到字节序的映射函数对 `R8G8B8A8_UNORM` 返回 BGRA | L1 契约 |
| D1.4 | `GD_LOG` 输出行可被正则 `^\[(Error\|Warn\|Info\|Debug\|Trace)\]\[[a-z0-9_]+\]\[[a-z0-9_]+\] .+$` 匹配；`log_level=Warn` 时 Info 行计数为 0；`GD_BUILD_TRACE=0` 编译后二进制中不含 Trace 字符串；配置键数量、名称、默认值与 `roadmap.md` §5.1 权威 19 键表全等（脚本比对，含 `tile_w/tile_h/translation.seed/memory_budget/simd_per_cu`） | L1 契约 |
| D1.5 | `scene_e0_selftest` 退出码 0 且打印"断言 N 条全绿"，N 等于 §4 六段断言的用例总数；契约清单（§3 各表行数）与头文件公开符号一一对应，无缺项无冗余 | L1 契约 + A1 |

---

## 7. 场景与判据（A1）

**绑定场景：E0 `scene_e0_selftest`（契约与工具自检）**

| 项 | 内容 |
|---|---|
| 输入 | 无外部输入；全部断言用例内建 |
| 产物 | 断言结果摘要（stdout）、E0 退出码 |
| 主判据 | 结构体/枚举/命令包头断言全绿；退出码 0 |
| 本章专有判据 | 冻结项清单与代码一致（§3 六张表逐行有对应符号）；任何契约改动有决策记录（ADR） |
| 回归项 | 无（首章） |
| 后续补齐 | "汇编→二进制→反汇编→再汇编二进制相同"在 T5.1 落地后并入 E0，本章只保留占位用例并标记 `SKIP_PENDING_CH5`，不计入失败 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 本章不含遍历与分配；浮点→字节只允许截断规则，禁止舍入策略随编译选项变化 |
| 时间无关 | 头文件与常量不得引用时钟；`frame_index` 类型在本章冻结，硬件路径不使用墙钟 |
| 可观测 | `GD_LOG` 格式与等级、`ErrorRecord` 字段在本章冻结；E0 打印断言摘要 |
| 契约影响 | 本章即冻结契约本身；改动走 ADR + `../roadmap.md` §7.4 工作流，并回归 E0~E5 |
| 简化登记 | 见 §9 总表（本章登记全项目初级简化与替换章） |
| 计算兼容 | `Stage::Compute`、`EngineType::Compute`、`DISPATCH` 包号在本章冻结，暂无人消费 |
| 配置与回退 | 配置键全部默认值即简化基线；`parallel` 在本章明确报"未实现"，不静默退化 |
| 调试钩子 | 本章无 shader printf / wave trace，只提供 `GD_LOG`；不进入硬判据 |
| trace 契约 | `trace` 键已登记，字段不做稳定承诺，不参与判据 |
| 契约变更 | 包头布局、包类型编号、`Result` 取值、颜色口径均为冻结项，变更需新建 ADR 并回归 E0~E5 |
| 校验归属 | 本章全部为 L1 契约/结构断言，零容差、阻断 |

---

## 9. 简化登记（全项目初级简化总表）

本章作为契约章登记全项目初级简化；第 2~9 章各自在自己的 §9 摘录相关行。

| 项 | 初级实现 | 替换章 |
|---|---|---|
| 地址 | 单一连续 VA，直接映射物理，无 TLB、无页错误 | 11 |
| 存储 | 扁平显存字节数组，无 L1/L2、无写缓冲 | 12 |
| 提交 | 单队列、单环、容量足够不回绕、无中断/watchdog | 10 |
| 执行 | 单波、单 SIMD、着色器内禁止分支 | 15、16 |
| ISA | 约 20 条指令子集，线性编码，无压缩编码 | 16 |
| 着色器来源 | 不引入编译器，全部手写 ISA | 17~19 |
| 工作量类型 | 只覆盖图形 draw，无 dispatch、无 workgroup | 29~31、37（原 35.5） |
| 固定功能 | 非索引 IA、无 HS/DS/GS、无 MSAA、固定深度比较 | 22~28 |
| 纹理 | RGBA8、nearest、单层、无 mip、无 sRGB、无采样器对象 | 21 |
| 资源绑定 | 常量/顶点/纹理直接用指针传递，无描述符堆 | 20 |
| 显示 | 单 RT、双缓冲、提交后读回，不建模 vsync/翻转链 | 13 |
| 上下文 | 单上下文，无驻留逐出、无抢占 | 14 |
| 同步 | 单队列顺序完成，无 semaphore/fence 硬件对象 | 32 |
| 执行策略 | 仅 `serial`，无缓存/TLB 统计 | 31 |
| API 层 | 无 HLI/UMD，场景直接调用引擎 C++ 接口 | 35 |
| 校验 | 全图逐字节 + 少量结构断言 | 7 起逐步引入参考模型与探针 |

规则：每条简化必须在代码中有唯一实现点；替换章只能替换实现，不得改本章冻结的契约与 E0~E5 的输入数据。

---

## 10. 待定项

无。本章条款按冻结契约管理。后续若需调整，按 roadmap §7.4 登记 ADR、同步全局登记表与受影响章节判据；被替代判据须标记 `SUPERSEDED` 并指向新判据，不能作为阻断回归继续执行。

---

## 2026-11-21 评审同步说明

本章的 `Result=11`、`D1.1` 恰 11 值、回归全局冻结条款均被 `review-sync-20261121.md` §2.1 替代。终版为 16 个 Result；契约变更按 roadmap §7.4 的 ADR + SUPERSEDED 流程。

> 旧 `D1.1` 中“恰 11 个取值”及 `§4` 的“取值数 = 11”属于已退役判据，更新为对 §3.2 16 值集合做唯一性与快照比对。
