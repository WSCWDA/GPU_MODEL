# 第 19 章 LLVM 对照后端

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章将同一 SSA 语义分别映射到手写后端与 LLVM 路径，对照编码合法性、ABI 与规范程序的执行结果。两套后端允许合法指令选择和寄存器分配不同，不要求 blob 逐字节相同；本章不阻塞第 20 章对象层开发。
> 与本章相关的设计说明：`../design/compiler.md` §1~§3（Mesa NIR 形状对照 / 对照管线 / 任务归属）；ADR-012（LLVM 对照后置在 18 之后）。
> 落地模块：`umd/shader/llvm`、`tools/llvm_compare`、`apps/scene_s3_triangle`（对照回归）。

---

## 1. 目标与非目标

### 1.1 目标

1. SSA IR → LLVM IR 映射：指令、CFG、φ 映射为 LLVM phi；一致性标注（uniform→scalar path）降级为标量/向量选择。
2. 两后端对同一 IR 快照须满足相同 ABI、合法 ISA 与规范测试程序语义；允许编码和寄存器分配不同。
3. JIT 编译 LLVM IR 执行与解释器执行对照：同输入输出一致（对照语义层）。
4. 对照报告：`tools/llvm_compare` 逐指令映射差异输出（供调试）。
5. 回归：S3 全场景两后端一致；不一致即阻断。

### 1.2 非目标

- 不做生产路径替换：LLVM 不进入运行时，只作对照回归。
- 不做 LLVM 优化调优：LLVM IR 只保证语义正确对照，不做优化对齐（优化是本项目后端自己家的）。
- 不做真机/外部编译后端参照：只对照自身 IR（ADR-027）。
- 不引入 Mesa/第三方编译器代码（对照形状参考）。

---

## 2. 单元框图

```
 前端 IR 快照 ──► 手写后端（18）──► blob_A
      │  同一快照
      ├─► SSA→LLVM IR 映射 ──► LLVM opt(无优化) ──► LLVM JIT
      │                                          │ 执行 → 输出 ref（对照）
      └─► blob_A ──► 解释器执行 ──► 输出 A
   对照：blob_A 与 blob_LLVM 的 ABI/合法编码及规范程序语义对照
        （两后端发射同一 16 章编码表；`blob_LLVM` 的发射实现独立于第 18 章）
```

- LLVM IR 是语义对照路径；比较 ABI/符号/合法 ISA 和规范程序执行输出。后端允许不同编码、寄存器分配和指令序列。
- 一致性失败 → 阻断本章通过。

---

## 3. 输入输出契约

### 3.1 对照接口（`tools/llvm_compare/compare.h`）

```cpp
struct CompareConfig { bool enableJit; bool emitReport; u32 diffLimit; };
struct CompareResult {
    bool abiIsaLegal;           // 两后端都遵守 ISA/ABI
    bool semanticEqual;        // 规范程序输出满足同一参考判据
    std::vector<Diff> diffs;   // 首 D 处 `{insnIndex, a, b}`
    Stats stats;               // {insnCountA, insnCountB, semanticMismatchCount}
};
Result CompareBackends(const IrModule&, const CompareConfig&, CompareResult* out);
```

- `Diffs` 报告逐指令：`{index, blobA_dword, blobB_dword, phase}`。

### 3.2 映射表（SSA → LLVM IR）

| SSA | LLVM IR | 说明 |
|---|---|---|
| 算术 | `fadd/fsub/fmul/fdiv` | 按 type |
| 比较 | `fcmp`/`icmp` → select | mask 语义 |
| 常量 | constant-fold 立即数 | |
| φ | `phi` 节点 | LLVM phi |
| 分支 | `br` | CFG 直接映射 |
| 内建 ID | 本模块内调用 | 本项目内建 → 内部函数 |

- 映射不蕴含"优化对齐"，只做语义等价对照。

---

## 4. 内部数据流

### 4.1 对照流程

```
IR 快照 → 手写后端 / LLVM 对照后端分别生成合法 ISA
  → 检查 module/symbol/ABI 与指令编码
  → 分别执行规范程序并与 CPU reference 按 scene-specs 容差比较
  → 语义不符时报告首个可定位差异并阻断本章 L2 判据
LLVM JIT vs interpreter 仅作语义定位辅证
```

- 后端可以选择不同指令、寄存器分配和 blob 编码；主判据是两后端分别满足 ABI/ISA 合法性且规范程序语义满足 reference 判据，不要求二进制相同。

### 4.2 失败定位

- `diffs` 报告首个差异指令：映射相位（arithmetic/compare/branch）、A/B dword、指令 mnemonics（16 章反汇编）。
- 定位到映射 bug 或手写后端 emit bug，修后重跑直到 `abiIsaLegal`。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T19.1 | ① 契约 | `CompareConfig`/`CompareResult`/映射表与 diff 结构落头文件 |
| T19.2 | ② 主体 | SSA IR → LLVM IR 映射引擎：算术/比较/φ/分支 |
| T19.3 | ② 主体 | LLVM 后端 emit：LLVM IR → 本项目 ISA blob（复用 16 章编码表；发射实现独立于第 18 章，不调用其 emit） |
| T19.4 | ② 主体 | `tools/llvm_compare`：比对、diff 报告、统计 |
| T19.5 | ② 主体 | 内建 ID/发散标注在 LLVM IR 的映射（标量/向量选择） |
| T19.6 | ② 主体 | JIT 执行 vs 解释器执行对照（语义层） |
| T19.7 | ② 主体 | 差异定位辅助：反汇编标注首个差异指令 |
| T19.8 | ③ 接入 | `scene_s3_triangle`：规范 shader 用例两后端 ABI/ISA 合法性与执行语义对照 |
| T19.9 | ④ 验收 | A19：S3 两后端 ABI/ISA 合法且规范程序语义一致 + R19 回归项 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D19.1 | 接口独立编译；`CompareResult` 字段冻结 | L1 契约 |
| D19.2 | 模块/符号/ABI 与编码合法性通过 L1；规范程序运行结果与 CPU reference 的语义比对按 scene-specs 容差执行，归 L2 阻断；不要求二进制逐字节相同 | L1 + L2 |
| D19.3 | JIT 执行与解释器执行同输入时输出满足 S3 参考判据 | L2 解析 |
| D19.4 | `tools/llvm_compare` 输出 ABI/指令合法性与语义对照报告；映射注入错误时规范程序结果不一致且定位到 IR 操作/指令 | L1 + L2 |
| D19.5 | 发散 uniform 在 LLVM IR 映射为标量；发散值向量 | L1 契约 |
| D19.6 | JIT 语义对照：相同输入帧，输出与解释器一致（helper） | L2 解析 |
| D19.7 | 反汇编标注首差异指令 mnemonics 与映射相位 | L1 契约 |
| D19.8 | S3 规范探针与参考模型通过；仅各自后端的确定性输出需重复运行稳定 | L2 + A19 |
| D19.9 | A19 全绿：规范测试程序语义一致 + S3 硬判据不退化 + 本章 `R19` 回归项通过 | L1 + L2 |

---

## 7. 场景与判据（A19）

**绑定场景：S3 `scene_s3_triangle`（编译对照）。S0 编译器自检扩展属第 17 章覆盖，不并入本章判据。**

| 项 | 内容 |
|---|---|
| 输入 | S3 每源码：同一 IR 快照两后端；`tools/llvm_compare` 报告 |
| 产物 | `abiIsaLegal` 判定、diff 报告（若失败）、JIT vs 解释器 RB |
| 主判据 | 两后端在规范测试程序上满足 ABI、指令合法性与执行语义一致；代码质量指标为非阻断观测 |
| 本章专有判据 | ① 两后端 ABI/ISA 均合法；② 规范程序与参考模型语义一致；③ JIT 与解释器对照作为辅证 |
| 回归项 | R19：S3 规范探针与手写基线 E2/E4 不退化；只回归受本章后端映射影响且仍有效的判据，历史 A 门禁不自动阻断 |
| 后续补齐 | 第 20 章基于已冻结的手写后端/ABI 契约可并行开工；第 19 章完成后补入后端语义对照回归，不把 A19 设为第 20 章开工门槛 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 映射表固定、规范输入固定；每个后端重复运行结果稳定，跨后端只比较语义判据 |
| 时间无关 | 编译/对照无时间输入 |
| 可观测 | `tools/llvm_compare` diff 报告、反汇编标注 |
| 契约影响 | 映射表、ABI/ISA 合法性和语义对照集为冻结项（新增映射走 §7.3 登记） |
| 简化登记 | 手写后端是本项目唯一生产后端；LLVM 只对照不替换 |
| 计算兼容 | 规范 compute kernel 可作为 LLVM/ISA 语义对照输入；本章只验编译产物的 ABI/合法编码与语义，不运行 S18 端到端场景（S18 场景级验收归第 37/42 章） |
| 配置与回退 | `enableJit`、`emitReport`、`diffLimit` 为 `CompareConfig` 对照工具选项（非全局配置键）；对照失败阻断，不静默 |
| 调试钩子 | `--dump-llvm-ir`、`--dump-llvm-blob`、diff 报告 |
| trace 契约 | 不进入判据 |
| 契约变更 | 一致性判据/映射表为冻结项；变更走 ADR 回归 S3/S0 |
| 校验归属 | L1（模块/符号/ABI/编码合法性）+ L2（规范程序语义，按 scene-specs 阻断） |
| 双策略 | 对照与执行策略无关；一致性两策略下成立 |
| 阶段衔接 | 19 与 20 共用已冻结 ISA/ABI 契约，可并行开发；A19 语义判据用于 LLVM 对照质量与回归，不阻塞 20 开工 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 对照 | 无 → LLVM IR 对照后端 | —（第十九章定案） |
| 一致性 | 无 → 两后端 ABI/ISA 合法性与规范程序语义对照 | — |
| 生产后端 | LLVM 不替换手写 | —（永远） |

---

## 10. 待定项

- 后端编码、寄存器分配或块序差异只进入诊断报告，不作为失败；映射差异导致规范程序语义不符时按 D19.2 定位并阻断。
- `enableJit` 在 CI（无 LLVM JIT 基础设施）时默认关、走"emit 对照"，须保证两种配置下判据一致——待 CI 环境可用性验证后定是否默开。
- 是否需要支持 `-O0` 之外的 LLVM 优化作为交叉验证，属可选项；若做，仅作软观测，不改变语义一致性阻断判据。
