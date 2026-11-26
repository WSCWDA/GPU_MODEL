# 第 38 章 计算驱动骨架——umd/compute 对象模型与 Module 装载

> 本章是 **stage3 计算驱动开发线（新章 38~42，ADR-047）** 的第一章。在 1~37 章图形管线（HLI、KMD、FW、内存、队列、Compute 引擎派发、第 37 章计算通路收口）之上，落地 MUSA 风格计算驱动（`umd/compute`）的**对象模型骨架**：`context / module / function` 对象头文件、`MUSA_MODULE_HDR` 二进制格式与装载、符号表解析与 function 入口。从此章起，计算驱动作为独立用户态驱动栈与图形 UMD 并列（ADR-009），只依赖 HLI 与 `umd/os`。
> 设计权威：驱动对象模型 / `MUSA_MODULE_HDR` 布局 / 符号表 / 生命周期见 `../design/compute-driver.md` §2~§4、§6、§9；kernel 参数与 `DispatchDesc` 契约见 `../architecture-design.md` §3.8、§3.11；对象销毁契约（逆创建、双维度引用计数）见 `../architecture-design.md` §3.10；错误码映射见 §3.9；WA 业务目录见 `../architecture-design.md` §7.9，计算通路见 `../design/compute-line.md` §1/§2。
> 编号纪律：本章**不新增包类型 / opcode / stateId / MMIO / 配置键**（`DISPATCH=0x04` 等全部复用，ADR-043/047）；开工前对照 `../design/bit-budget.md` 审计（ADR-045）；不新增 `hw_sim` 单元 / 目录（ADR-009）。

---

## 1. 目标与非目标

### 1.1 目标

1. **`umd/compute` 目录与对象骨架落地**：`Context / Module / Function` 的头文件、生命周期、创建 / 销毁（逆创建，ADR-041 双维度引用计数语义）。
2. **`MUSA_MODULE_HDR` 格式与装载**：magic / version / 代码段 / 符号表的逐字节解析；装载入 GART `HOST_VISIBLE` 常驻堆；**不引入 ELF/COFF、不做重定位**（`compute-driver.md` §3）。
3. **符号表解析与 Function 入口**：`codeBaseVa + entryOffset`（8 B 对齐）、越段 / 未映射确定报错（§3.11 D-06 同口径）、分支 pc 相对偏移不需地址修补。
4. **异常输入确定性**：截断 blob、坏 header、越界符号、非法 reserved → `InvalidIsa` / `NotMapped`，无泄漏、无部分创建。
5. **Demo C1 闭环**：`module_probe` 装载示例 kernel module，打印符号表与解析入口，`MUSA_MODULE_HDR` 与 blob 逐字节比对全等，输出供后续章使用的参数布局信息。

### 1.2 非目标

- 不做 kernel 参数打包与 `LaunchKernel`（第 39 章）；不做 stream/event（第 40 章）；不做组内协作 / 间接派发 / UAV 写回（第 41 章）。
- 不做 MUSA / CUDA API 或二进制兼容；不做动态并行；不做 ELF/COFF、不做重定位器。
- 不为计算特化硬件单元（ADR-032）；不新增 HLI 接口（dispatch 对象仍由既存 `IHliPipeline` 承载）。

---

## 2. 单元框图

```
apps（Demo C1 module_probe）
   │  MUSA_MODULE_HDR blob（示例 kernel ISA，手写或 17~19 章编译器产物）
   ▼
umd/compute::Context　←── IHliDevice + 独立 KMD context/ASID（Context:ASID 1:1）+ 资源表
   │  umd/compute::ModuleLoad(blob)
   ▼
module blob 解析：magic/version/reserved → 代码段 → 符号表逐条
   │  （代码偏移与入口均 8 B 对齐，符号表 4 B 对齐，完整指令与名称均在各自范围内）
   ▼
GART HOST_VISIBLE 常驻装载（第 11 章确定性分配器，VA 可复现）
   ▼
umd/compute::Function　= { codeBaseVa + entryOffset, paramBytes, vgprCount, ldsBytes }
   │  SetKernel（IHliPipeline compute，35.1 冻结对象面，本章只建对象不派发）
   ▼
Demo C1 断言：MUSA_MODULE_HDR 字段 ≡ blob；符号表解析正确；异常输入确定错误
```

说明：

- 本章**只建对象、不派发**：`IHliPipeline(kernel)` 对象在第 35.1 章冻结，本章通过 `SetKernel` 建立 kernel 管线对象；`Dispatch` 首次真正下发在第 39 章（`LaunchKernel`）。
- 装载内存走第 11 章堆与确定性分配器（`translation.seed` 可复现），与图形资源同一条 VA 空间，零拷贝共享（§3.13 Global/Param 同空间）。

---

## 3. 输入输出契约

### 3.1 对象头文件（权威：`compute-driver.md` §2、`architecture-design.md` §3.10）

```cpp
namespace compute {

struct ModuleHdr {           // MUSA_MODULE_HDR（compute-driver.md §3）
    u32 magic;               // 'MUSA'
    u32 version;             // 1
    u32 codeOffset, codeBytes;
    u32 symTabOffset, symTabCount;
    u32 reserved0, reserved1; // 必须 0
};
struct SymEntry {            // 符号表条目（28 B）
    u32 nameOffset, nameLen, entryOffset;
    u32 paramBytes, vgprCount, ldsBytes, flags;
};
struct ModuleInfo {          // 解析结果（装载后生成）
    GpuVAddr codeBaseVa;     // 代码段基址（GART 常驻）
    u32      codeBytes;
    std::vector<SymEntry> syms;
};
struct FunctionInfo {
    GpuVAddr kernelVa;       // codeBaseVa + entryOffset
    u32      paramBytes;     // 供第 39 章打包校验
    u32      vgprCount, ldsBytes;
};

class Context;
class Module;                // 持 GART 资源 + ModuleInfo + 归属 context
class Function;              // 持 Module* + FunctionInfo
}
```

### 3.2 校验规则（装载期执行，违反返回 `InvalidIsa` / `NotMapped`，不产生 HLI 资源）

| 编号 | 规则 |
|---|---|
| L-01 | `magic` 字节为 `4D 55 53 41`（小端 u32 `0x4153554D`），否则 `InvalidIsa`。`version` 为 1，其余版本 `NotSupported` |
| L-02 | `reserved0/1` 必须 0（`InvalidIsa`） |
| L-03 | `codeOffset` 与 `codeBytes` 按 8 B 对齐且代码至少 8 B，`symTabOffset` 按 4 B 对齐。代码段和全部固定 28 B 符号记录须落在 blob 内，按长度上限校验后再做乘加 |
| L-04 | `entryOffset` 8 B 对齐且 `<= codeBytes - 8`。名称字节范围在 blob 内且 `nameLen ≤ 127`，未知的 `SymEntry.flags` 位返回 `InvalidIsa` |
| L-05 | 装载后 `codeBaseVa` 已映射（`NotMapped`）；装载失败释放已分配资源（无泄漏、无部分创建） |

### 3.3 错误映射（权威：`architecture-design.md` §3.9）

- 坏 module / 未命中符号 / 越段 → `InvalidIsa` → `MUSA_ERROR_INVALID_DEVICE_FUNCTION`；
- 未映射 → `NotMapped` → `MUSA_ERROR_INVALID_VALUE`（执行期走设备丢失类）；
- 版本/能力不支持 → `NotSupported` → `MUSA_ERROR_NOT_SUPPORTED`。

---

## 4. 内部数据流或状态机

### 4.1 对象状态机（`compute-driver.md` §2 权威）

```
Context: Created → [可创建 Module/Function] → Destroyed（仍有子对象/硬件引用时返回 Busy；不隐式同步）
Module:  Loaded(解析通过) → Resident(GART 常驻) → Bound(被 SetKernel 引用) → 释放
Function:Resolved(符号命中) → Launched(第 39 章首次 Dispatch 引用) → 随 Module 释放
```

### 4.2 ModuleLoad 数据流

```
Context::ModuleLoad(blob)
  ├─ 校验 L-01~L-05 → 失败返回错误码（不产生对象）
  ├─ 解析 ModuleHdr + SymTable → ModuleInfo
  ├─ umd/os 分配 GART HOST_VISIBLE 常驻块 → 装载代码段 → codeBaseVa
  └─ 创建 Module 对象（归属 context，用户引用计数=1，硬件提交引用=0）
```

### 4.3 错误流程

- 统一的"记录 + 中止当前创建"（ADR-011）：打印 `{unit, packetType, dwordOffset, va, code}` 一条错误后返回确定错误码；module 装载无包路径时 `packetType/dwordOffset` 填 `N/A`、`va` 记 module blob/代码段基址；已分配资源按逆序释放。

---

## 5. 任务清单

> 任务编号 `T38.1~T38.6`，四段式（① 契约 → ② 主体 → ③ 接入 → ④ 验收）。

| 子里程碑 | 任务 | 段 | 内容 |
|---|---|---|---|
| M38.1 | T38.1 | ① 契约 | `umd/compute` 头文件：`Context/Module/Function` + `ModuleHdr/SymEntry/ModuleInfo/FunctionInfo`（对齐 `compute-driver.md` §2~§4，`static_assert` 布局单测落头文件） |
| M38.1 | T38.2 | ② 主体 | `Context` 创建/销毁：同一 `IHliDevice` 支持多个 Context，每个分配独立 ASID；销毁有 Module/Function 等存活子对象或硬件引用时返回 `Busy`，不隐式同步；逆创建顺序（§3.10） |
| M38.1 | T38.3 | ② 主体 | `MUSA_MODULE_HDR` 解析：magic/version/reserved、代码段、符号表逐条解码（L-01~L-04）；损坏/截断 → `InvalidIsa` |
| M38.1 | T38.4 | ② 主体 | `Function` 入口解析：`codeBaseVa + entryOffset`（8 B 对齐、越段 `InvalidIsa`、未映射 `NotMapped`）；pc 相对偏移不需地址修补 |
| M38.2 | T38.5 | ③ 接入 | module 装载经 HLI 资源（GART `HOST_VISIBLE`，第 11 章确定性分配器）接入；Demo C1 `module_probe` 跑通 |
| M38.2 | T38.6 | ④ 验收 | A38：`ModuleHdr` 字段与 blob 逐字节一致；符号表解析正确；截断/坏 header/越段/非法 reserved 返回确定错误且无泄漏；重复解析幂等 |

### 子里程碑

| 子里程碑 | 范围 | 承载任务号 |
|---|---|---|
| M38.1 对象骨架 | 头文件、Context 生命周期、Module 解析、Function 入口 | T38.1~T38.4 |
| M38.2 装载接入与验收 | GART 装载接入、Demo C1、A38 | T38.5~T38.6 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D38.1 | 计算驱动头文件独立编译；`ModuleHdr`/`SymEntry` 布局 `static_assert` 全绿 | L1 契约 |
| D38.2 | 同一 device 创建两个 Context 得到不同 ASID；逆序销毁无泄漏；存在存活子对象或硬件引用时 `ContextDestroy` 返回 `Busy`，清理后可销毁 | L1 契约 |
| D38.3 | `ModuleHdr` 字段与 blob 逐字节一致；6 种非法样例（bad magic/version/reserved/越界/截断/坏对齐）全部返回确定错误且无部分创建 | L1 契约 |
| D38.4 | 首条与末条完整指令入口成功。只满足 4 B 对齐、落在代码末尾或跨段时返回 `InvalidIsa`，未映射返回 `NotMapped` | L1 契约 |
| D38.5 | Demo C1 `module_probe` 输出符号表（name/entryOffset/paramBytes/vgprCount/ldsBytes）与解析入口；两次运行逐字节一致 | L2 |
| D38.6 | A38 全绿（D38.1~D38.5 + 异常样例回归矩阵；`serial`/`parallel` 一致） | L1 + L2 |

---

## 7. 场景与判据（A38）

**绑定：Demo C1 `module_probe`（本阶段主判据）；S18 提供 37 章计算通路前序能力验证（回归，不做端到端主判据）。**

| 项 | 内容 |
|---|---|
| 输入 | 示例 kernel module blob（手写 ISA 或第 17~19 章编译器产物；≥2 个符号）；异常样例集（截断 / bad magic / version=2 / reserved≠0 / 越界符号 / 仅满足 4 B 对齐 / 指令跨段） |
| 产物 | 符号表打印、每个 function 的 `kernelVa`、`MUSA_MODULE_HDR` 与 blob 字节比对、异常样例错误码与错误日志 |
| 主判据 | `ModuleHdr` 字段逐字节一致；符号表解析正确；异常输入全部命中确定错误且无泄漏（L1 + D38.2~D38.4） |
| 本章专有判据 | 对象生命周期逆序创建/销毁无泄漏；重复解析同一符号返回同一 Function（幂等）；装载不产生任何包 |
| 回归项 | R38：S18（作为 37 章通路能力验证）在接入后不退化；未列明历史 A 门禁不自动阻断 |

> 场景输入与探针单份规约见 `../design/scene-specs.md` §3（S18 节）与 §4（第 38 行）。

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | module 装载与符号解析顺序固定；两次运行符号表与入口逐字节一致；无时间输入 |
| 时间无关 | 装配/解析不依赖墙钟或帧序 |
| 可观测 | `--trace=compute`：module 头、符号表、装载地址快照；供人工、不进硬判据 |
| 契约影响 | 不新增编号（复用既有包号/SubOp/stateId/MMIO/配置键）；`ModuleHdr` 布局冻结，变更走 ADR 与 `roadmap.md` §7.4 |
| 简化登记 | 无 ELF/COFF、无重定位（与第 16 章 pc 相对分支一致）；一期不做 `LaunchWithParamBuffer` |
| 计算兼容 | 对象模型与图形 UMD 并列，共用 HLI；不新增计算专用 HLI 面（ADR-032） |
| 配置与回退 | 不新增配置键；装载行为不受 `exec_mode` 影响 |
| 调试钩子 | module blob dump、符号表打印；只供人工，不进硬判据 |
| trace 契约 | `--trace` 不保证字段稳定、不做机器解析、不进判据 |
| 契约变更 | 涉及 `ModuleHdr`/对象生命周期基本契约一律走 ADR，回归 S18 与受影响 S 场景 |
| 校验归属 | L1（结构/状态）+ L2（Demo C1 探针） |
| 双策略 | 装载与解析为 CPU 侧，双策略结果一致；S18 回归项验证 `serial`/`parallel` 不退化 |

---

## 9. 简化登记

| 项 | 状态 | 替换章 / 备注 |
|---|---|---|
| 对象模型 | stage3 落地 `context/module/function` | 37 章不含计算驱动对象模型（ADR-048）；本章首次落地，`stage1/01` §9 的"无计算驱动"由 38~42 替换 |
| `MUSA_MODULE_HDR` | 自有头 + 符号表，非 ELF/COFF | `compute-driver.md` §3 |
| 无重定位 | pc 相对分支（第 16 章） | 装载不需地址修补 |
| Driver API / Runtime API 简化 | 一期只做核心子集 | `compute-driver.md` §2/§7 |

---

## 10. 待定项

- module 多入口（`entryOffset` 数组化）与嵌入式常量区（`.rodata`）留二期：一期代码段 + 符号表即可覆盖 WA-01~12 全部 demo。
- `LaunchWithParamBuffer`（应用直给参数 VA）在第 39/42 章登记，不在本章实现。
- "35.1 HLI 冻结后才允许写计算驱动"纪律：本章依赖 35.1 DoD 全绿；如遇 HLI 接口缺口，不反向修改 HLI，走 ADR 流程。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A38 和适用的 R38 后提交。分配核对项包括MUSA_MODULE_HDR、SymEntry.flags、ASID。变更按已有分配流程登记。

```text
feat(gpu-ch38): load modules and resolve compute functions

- 实现计算 Context、模块字节解析及 Function 入口。
- 用 C1 对照 magic、符号范围、8 B 入口及释放负例。

Tasks: T38.1–T38.6
Validation: <实际执行的命令、用例和结果>
```
