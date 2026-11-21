# 第 19 章 LLVM 对照后端

> 项目范围、阶段位置和交付以[任务规划](../CPU模拟GPU教学项目_任务规划与实施规格.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把手写后端（第 18 章）产出的 SSA IR 映射到真实 LLVM IR，用真实编译器（JIT）验证我们的后端结果——"两后端逐字节一致"是本单元的唯一判据，也是 ADR-012 的收口点：19 未过不进入第 20 章。它**不是**生产路径，只做对照，不替换手写后端。
> 与本章相关的设计说明：`../design/compiler.md` §1~§3（Mesa NIR 形状对照 / 对照管线 / 任务归属）；ADR-012（LLVM 对照后置在 18 之后）。
> 落地模块：`umd/shader/llvm`、`tools/llvm_compare`、`apps/scene_s3_triangle`（对照回归）。

---

## 1. 目标与非目标

### 1.1 目标

1. SSA IR → LLVM IR 映射：指令、CFG、φ 映射为 LLVM phi；一致性标注（uniform→scalar path）降级为标量/向量选择。
2. 手写后端与 LLVM 后端对**同一 IR 快照**产出**逐字节一致的 blob**（唯一判据，D19 核心）；`blob_LLVM` 由独立于第 18 章的发射实现产生。
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
   对照：blob_A(手写)  vs  blob_LLVM(LLVM 回测输入) 逐字节一致
        （两后端发射同一 16 章编码表；`blob_LLVM` 的发射实现独立于第 18 章）
```

- "逐字节一致"指两后端对 IR 的**编码输出**一致（手写 emit 与 LLVM 后端 emit 生成相同编码序列）；`blob_LLVM` 由独立于第 18 章的发射实现产生，不调用手写后端 emit（Q-50）；LLVM IR 本身只是中间对照物。
- 一致性失败 → 阻断本章通过。

---

## 3. 输入输出契约

### 3.1 对照接口（`tools/llvm_compare/compare.h`）

```cpp
struct CompareConfig { bool enableJit; bool emitReport; u32 diffLimit; };
struct CompareResult {
    bool byteEqual;            // blob_A 与 blob_LLVM 逐字节
    bool semanticEqual;        // JIT 执行 vs 解释器执行输出
    std::vector<Diff> diffs;   // 首 D 处 `{insnIndex, a, b}`
    Stats stats;               // {insnCountA, insnCountB, diffCount}
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
IR 快照 → 手写 emit → blob_A
IR 快照 → SSA→LLVM IR → opt(-O0) → (JIT) → blob_LLVM（独立于第 18 章的发射实现，不调用其 emit）
  → byteEqual 比对
  → 若等 → 通过
  → 若不等 → diffs 报告 → 阻断 + 定位指令
语义层：LLVM JIT 执行 vs 解释器执行（同样输入）输出相等（辅证）
```

- 两次比对互为冗余：字节一致是主判据；语义一致是兜底（防双双错误）。

### 4.2 失败定位

- `diffs` 报告首个差异指令：映射相位（arithmetic/compare/branch）、A/B dword、指令 mnemonics（16 章反汇编）。
- 定位到映射 bug 或手写后端 emit bug，修后重跑直到 `byteEqual`。

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
| T19.8 | ③ 接入 | `scene_s3_triangle`：全 S3 代码两后端逐字节一致回归 |
| T19.9 | ④ 验收 | A19：S3 两后端逐字节一致 + LLVM JIT 语义一致 + A18 回归 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D19.1 | 接口独立编译；`CompareResult` 字段冻结 | L1 契约 |
| D19.2 | 全 S3 着色器：手写 blob_A 与 LLVM 后端 blob_LLVM 逐字节一致 | L1 契约（横切） |
| D19.3 | 除字节一致外，JIT 执行与解释器执行（同输入）输出 RB 相同 | L1 契约 |
| D19.4 | `tools/llvm_compare` 输 `byteEqual=true` + `diffCount=0`；若人为插入 bug（改一个映射）→ `diffCount>0` 且定位到正确指令 index | L1 契约 |
| D19.5 | 发散 uniform 在 LLVM IR 映射为标量；发散值向量 | L1 契约 |
| D19.6 | JIT 语义对照：相同输入帧，输出与解释器一致（helper） | L1 契约 |
| D19.7 | 反汇编标注首差异指令 mnemonics 与映射相位 | L1 契约 |
| D19.8 | S3 全场景逐字节回归通过 | L1 契约 + A19 |
| D19.9 | A19 全绿：两后端逐字节一致（唯一判据）+ S3 硬判据不退化 + A1~A18 回归 | L1 + L2 |

---

## 7. 场景与判据（A19）

**绑定场景：S3 `scene_s3_triangle`（编译对照）。S0 编译器自检扩展属第 17 章覆盖，不并入本章判据。**

| 项 | 内容 |
|---|---|
| 输入 | S3 每源码：同一 IR 快照两后端；`tools/llvm_compare` 报告 |
| 产物 | `byteEqual` 判定、diff 报告（若失败）、JIT vs 解释器 RB |
| 主判据 | **两后端逐字节一致（唯一判据）** |
| 本章专有判据 | ① 两后端逐字节一致；②（辅证）JIT 与解释器语义一致 |
| 回归项 | A1~A18 全绿；手写基线 E2/E4（手写 ISA 路径）不退化 |
| 后续补齐 | ADR-012 门禁在此达成——19 过后才解锁第 20 章 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 映射表固定、发射顺序固定；字节一致性是确定判据 |
| 时间无关 | 编译/对照无时间输入 |
| 可观测 | `tools/llvm_compare` diff 报告、反汇编标注 |
| 契约影响 | 映射表与"两后端一致"为冻结项（新增映射走 §7.3 登记） |
| 简化登记 | 手写后端是本项目唯一生产后端；LLVM 只对照不替换 |
| 计算兼容 | compute kernel 同能对照（S18 消费） |
| 配置与回退 | `enableJit`、`emitReport`、`diffLimit` 为 `CompareConfig` 对照工具选项（非全局配置键）；对照失败阻断，不静默 |
| 调试钩子 | `--dump-llvm-ir`、`--dump-llvm-blob`、diff 报告 |
| trace 契约 | 不进入判据 |
| 契约变更 | 一致性判据/映射表为冻结项；变更走 ADR 回归 S3/S0 |
| 校验归属 | L1 契约（逐字节）+ L2（JIT 语义辅证） |
| 双策略 | 对照与执行策略无关；一致性两策略下成立 |
| 阶段衔接 | ADR-012 门禁：19 未过不进入 20；B 阶段门禁（ISA/ABI 冻结）前置满足 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 对照 | 无 → LLVM IR 对照后端 | —（第十九章定案） |
| 一致性 | 无 → 两后端逐字节 | — |
| 生产后端 | LLVM 不替换手写 | —（永远） |

---

## 10. 待定项

- LLVM IR 的 phi/收敛点与手写 φ 拷贝的不一致（如块排序差异）可能导致字节差异——先按 IR 块序固定 LLVM 输出顺序，若仍有差异以 diff 报告定位后固定映射顺序并记录。
- `enableJit` 在 CI（无 LLVM JIT 基础设施）时默认关、走"emit 对照"，须保证两种配置下判据一致——待 CI 环境可用性验证后定是否默开。
- 是否需要支持 `-O0` 之外的 LLVM 优化作为交叉验证，属可选项；若做，仅软观测，不改变"逐字节一致"冻结判据。
