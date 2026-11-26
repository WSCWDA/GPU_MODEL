# 第 18 章 后端

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把前端 SSA IR 变为最终 ISA blob：优化（常量折叠/DCE/CSE/复制传播，不做循环变换）→ 线性扫描寄存器分配（SGPR/VGPR 分区、溢出）→ 指令选择 + 64 bit 发射。手写后端在本章定稿，作为第 19 章 LLVM 对照的唯一基线与"逐字节一致"的唯一参照。设计说明见 `../design/compiler.md` §1~§3。
> 与本章相关的设计说明：`../design/compiler.md`（后端三里程）；ADR-012（19 必须在 18 后）；ADR-034（发散值→VGPR）；`../design/isa-abi.md` §3（ABI 分区不覆盖）。
> 落地模块：`umd/shader/opt`、`umd/shader/regalloc`、`umd/shader/emit`、`apps/scene_s3_triangle`（优化开关差分）。

---

## 1. 目标与非目标

### 1.1 目标

1. 优化（M18.1）：常量折叠、死代码消除（DCE）、公共子表达式消除（CSE）、复制传播；**禁止任何循环变换（含 LICM/循环不变量外提）**；优化不得改变可观测结果（优化开/关一致）。
2. 分歧分析从标注落实到分配：一致值（uniform）进 SGPR，发散值进 VGPR；溢出写 GART scratch（per-thread ≤4 KiB、4 B 对齐、不清零；每 wave ≤128 KiB；不占 CU-local occupancy 容量；按有效线程数分配，GART 不足则 dispatch 前 `OutOfMemory`）。
3. 线性扫描寄存器分配（M18.2）：VGPR 从 `v[76..]` 起（临时量区，isa-abi §3），可用上限 `v[127]`（`vgprLimit=128`）；SGPR `s0..s31` 限定；溢出（寄存器不足）→ GART scratch，不占用 ABI 分区。
4. 指令选择 + 发射（M18.3）：SSA → 指令序列 → 64 bit 编码（第 16 章）；编码逐位一致。
5. 发射顺序确定：IR 块序 + 指令选择顺序固定 → blob 确定。
6. 溢出统计可输出：`spillCount/spillBytes` 软观测（供 19 章对照与 S3 调试）。

### 1.2 非目标

- 不做循环优化/向量化/内联，亦不做 LICM/循环不变量外提（第 18 章明确排除）。
- 不做 SSA→LLVM（第 19 章）。
- 不做寄存器复用启发调优（若溢出频繁，由源程序/IR 改善，不做超复杂分配）。
- 不做多后端并存（19 章对照完成后手写后端定稿、LLVM 只对照不替换）。

---

## 2. 单元框图

```
 SSA IR（前端）──► Opt（M18.1）──► 分歧标注 → 寄存器分配（M18.2）
                │                         │   溢出(spill)
                │                         ▼
                └─► 指令选择 + 发射（M18.3）──► ISA blob（16 编码）
```

- 优化 pass 串行执行固定顺序（CSE→复制传播→DCE→常量折叠）；开关控制执行与否，保证"优化开/关一致"。
- 分配前：检查临时量区容量（`v[76..end]`）能否容纳活跃集；不足 → 溢出。

---

## 3. 输入输出契约

### 3.1 后端接口（`umd/shader/backend/backend.h`）

```cpp
struct BackendOptions {
    bool enableOpt;    // 优化开关
    u32  vgprLimit;    // VGPR 上限（默认 128；临时区从 v[76] 起）
    u32  sgprLimit;    // 32（s0..s31）
    u32  spillSlots;   // 溢出槽容量（如 256）
    bool emitRelf;     // 相对 pc 分支
};

struct BackendOutput {
    Result result;             // Ok / RegisterOverflow / OutOfMemory / InvalidIr
    std::vector<u64> blob;     // 编码后明文
    Stats stats;               // {optPasses, vgprUsed, sgprUsed, spillCount, spillBytes, insnCount}
};

Result CompileBackend(const IrModule&, const BackendOptions&, BackendOutput* out);
```

### 3.2 寄存器分区（发射期执行 ABI 约束）

| 分区 | 范围 | 本后端分配策略 |
|---|---|---|
| 输入/VS 输出/HS 因子/PS/深度 | `v[0..55]` | 固定，不分配 |
| uniform | `v[56..67]` | 常量直接引用，不进分配（kernel 参数区起点 `v[56]`） |
| 距离 | `v[68..75]` | clip/cull 保留 |
| 临时 | `v[76..]` | 线性扫描主区（`vgprLimit`） |
| SGPR | `s0..s31` | uniform 值；超过 → 溢出到 GART scratch |

- 任何分配不得覆盖固定分区（ABI 硬规则，分配器 assert）。

### 3.3 指令选择映射

SSA op 与 ISA op 一对一（本章设计指令表固定）：

```
add/sub/mul/div/frac → ADD/SUB/MUL/DIV
min/max → CMP+SELECT（或 MIN/MAX 若 ISA 有）——统一映射，避免两后端分歧
smoothstep → 序列
load/store → LD/ST（偏移 sX+imm）
分支/ф → BRC 与基础块排布
```

- 指令选择固定表在 `common/isa.h` 侧登记（新增必须走 §7.3 登记）。

---

## 4. 内部数据流

### 4.1 优化管线（固定顺序）

```
CSE（公共子表达式）→ 复制传播 → 常量折叠 → DCE
每 pass 一轮，直到无变化（固定最大迭代 8 防止不收敛）。
```

- `enableOpt=false`：跳过 pass，直接进入分配（结果应与开优化"逻辑等价"——优化只删冗余，不改语义）。
- 禁止循环变换：优化前后 IR 的循环结构与块序保持不变，不实现 LICM/循环不变量外提。

### 4.2 线性扫描

```
活跃区间：按块序线性扫描，记录每 SSA 生存期[start,end]
→ 若 overlap set ≤ vgprLimit 可用范围 → 直接分配 v[76..]
→ 否则：最老活跃值 spill 到 GART scratch 槽（per-thread ≤4 KiB、每 wave ≤128 KiB；GART scratch 不占 CU-local occupancy），spillCount++
→ uniform（发散标注 uniform）→ SGPR 或常量池
→ 最终验证分区不重叠（assert）
```

- 溢出槽映射到 GART scratch（per-thread ≤4 KiB、每 wave ≤128 KiB、4 B 对齐、不清零），发射 `LD/ST` 访问；dispatch scratch 分配超出 GART 可用容量时提交前返回 `OutOfMemory`。

### 4.3 发射

```
对每基本块翻译选择：
  线性顺序（块序固定）→ 指令序列
  φ → 拷贝移动到块入口/合并（简化：φ 各 incoming 在跳转前拷贝到目标）
分支：非条件 BR / 条件 BRC（偏移 = 目标 - pc - 1）
→ 每指令 encode → blob 追加；EXIT 收尾
→ 校验：blob 长度、相对偏移 s32 范围
```

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T18.1 | ① 契约 | `BackendOptions`/`BackendOutput`/`BackendStats` 与指令选择映射表落头文件 |
| T18.2 | ② 主体 | 常量折叠 + 复制传播 |
| T18.3 | ② 主体 | DCE + CSE |
| T18.4 | ② 主体 | 优化 pass 管理：固定顺序、`enableOpt` 开关、最大迭代 |
| T18.5 | ② 主体 | 活跃区间计算（块序线性） |
| T18.6 | ② 主体 | 线性扫描分配：VGPR 主区、SGPR uniform、分区保护 assert |
| T18.7 | ② 主体 | 溢出：spill 槽分配、LD/ST 生成、`SpillStats` |
| T18.8 | ② 主体 | 指令选择：SSA→ISA 固定表、内建展开 |
| T18.9 | ② 主体 | φ → 拷贝移动；控制流（BR/BRC/相对偏移） |
| T18.10 | ② 主体 | 发射编码：每指令 encode、blob 组装 |
| T18.11 | ② 主体 | 后端自检：合法 IR 编译成功、非法 IR（溢出/超限）对应错误 |
| T18.12 | ② 主体 | 后端差分自检：优化开/关两跑 + 输出 RB 一致 + 溢出统计（`scene_s3_triangle` 数据源） |
| T18.13 | ③ 接入 | S3 编译着色器执行链路接入：blob 喂解释器（16 章）出图 |
| T18.14 | ④ 验收 | A18：S3 编译执行 = ref_model + 专有判据（优化开/关一致、溢出统计可输出）+ R18 回归项 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M18.1 | T18.1~T18.4 | 后端契约/优化管线，D18.1~D18.4 全绿 |
| M18.2 | T18.5~T18.7 | 活跃区间/线性扫描/溢出，D18.5~D18.7 全绿 |
| M18.3 | T18.8~T18.14 | 指令选择/发射/场景接入验收，A18 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D18.1 | 接口独立编译；`vgprLimit/sgprLimit` 默认与 ABI 一致（128 / 32） | L1 契约 |
| D18.2 | 常量表达式折叠为立即数；`x=x` 复制传播消除 | L1 契约 |
| D18.3 | 未用赋值删除（DCE）；重复表达式提为一个 SSA（CSE） | L1 契约 |
| D18.4 | `enableOpt=false` 与 `=true` 的 blob 执行输出逐字节一致（优化只删冗余） | 横切零容差 |
| D18.5 | 活跃区间计算正确（跨块生存期合并） | L1 契约 |
| D18.6 | 分配不覆盖分区（assert/静态检查）；uniform 值进 SGPR/S 常量 | L1 契约 |
| D18.7 | 触发溢出的 IR：`spillCount>0`、`spillBytes>0`；溢出前后执行输出一致 | L1 契约 |
| D18.8 | 指令选择表每 SSA op 映射确定；`smoothstep` 展开序列固定 | L1 契约 |
| D18.9 | φ 拷贝移动正确；BR/BRC 偏移与第 16 章编码一致 | L1 契约 |
| D18.10 | blob 逐字与手写汇编（16 章）对同一 IR 二进制一致 | L1 契约 + L2 |
| D18.11 | 非法 IR（超 $vgprLimit 活跃）→ `RegisterOverflow`；分配 assert 不触发 | L1 契约 |
| D18.12 | S3 优化开/关两跑 RB 探针逐字节一致；`SpillStats` 输出非负数 | 横切零容差 |
| D18.13 | S3 编译管线出图 = ref_model（blob 喂解释器） | L2 解析 |
| D18.14 | A18 全绿：优化开/关一致 + 溢出统计可输出 + S3 编译路径出图 == 手写基线 | L1 + L2 |

---

## 7. 场景与判据（A18）

**绑定场景：S3 `scene_s3_triangle`（编译场景）。S0 编译器自检扩展属第 17 章覆盖，不并入本章判据。**

| 项 | 内容 |
|---|---|
| 输入 | S3：多源码着色器（分支/循环/内建）；同一 IR 优化开 vs 关两跑；一个放开 `vgprLimit` 触发溢出 |
| 产物 | 优化前/后 IR、blob、`SpillStats`、两跑 RT 探针 |
| 主判据 | S3 编译出图 = ref_model 期望；优化开/关输出一致 |
| 本章专有判据 | ① 优化开/关一致（逻辑等价性，探针零容差）；② 寄存器溢出统计可输出且溢出不破坏结果 |
| 回归项 | R18：手写 ISA 基线（第 6/7 章）不因编译器链路引入变化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 第 19 章 LLVM 对照以此为唯一基线与"逐字节一致"唯一参照 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | pass 固定顺序、迭代上限、块序固定、发射顺序确定 |
| 时间无关 | 编译无时间/随机输入 |
| 可观测 | `SpillStats`、`--dump-backend`(IR->blob)、`--trace=backend` 每 pass 一行 |
| 契约影响 | 指令选择映射表、分区策略、溢出语义冻结（登 `common/isa.h`） |
| 简化登记 | 手写 ISA → 编译后端（优化+分配+发射） |
| 计算兼容 | 后端不假设图形 stage；compute kernel 同一条管线（S18 消费） |
| 配置与回退 | `enableOpt`、`vgprLimit`、`spillSlots` 为 `BackendOptions` 编译期选项（非全局配置键）；手写 ISA 路径保留 |
| 调试钩子 | `--dump-blob` 二进制、`--trace=backend` 行化 |
| trace 契约 | 不进入判据 |
| 契约变更 | 指令选择表/分区/溢出语义为冻结项；变更走 ADR 回归 S3/S0 |
| 校验归属 | L1（结构）+ L2（编译路径出图与 ref_model） |
| 双策略 | blob 与执行策略无关；两策略下 S3 输出一致 |
| 阶段衔接 | 为第 19 章唯一基线；19 未过不进入第 20 章（ADR-012） |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 着色器来源 | 手写 ISA → 编译后端 | — |
| 寄存器 | 手工/单 wave → 线性扫描 | — |
| 优化 | 无 → CSE/DCE/折叠/复制传播 | — |
| 溢出 | 无 → spill 槽 | — |

---

## 10. 待定项

- 指令选择"逐 SSA 一对一"策略在多数场景合适，但 `min/max` 若 ISA 缺原生指令，则用 CMP+SELECT 展开；是否在后端发射表加 `MIN/MAX` 原生指令由硬件范围定，第 19 章 LLVM 对照前冻结（防两后端分歧）。
- 溢出写 GART scratch（per-thread，≤ 4 KiB、4 B 对齐、不清零）；若后续出现递归/深调用，可能改用软件栈帧，待 29/31 章软件栈定案。
- φ 拷贝移动到跳转前的简单实现会在某些循环引发多余拷贝；不做循环优化，成本可接受，若 S3 显示性能异常另议（仅软观测）。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A18 和适用的 R18 后提交。分配核对项为寄存器保留区、opcode、分支偏移，新增或改动按 [编码与位段校验表](../design/allocation-audit.md)登记。

```text
feat(gpu-ch18): compile IR to executable shader ISA

- 实现指令选择、寄存器分配和 ISA 发射。
- 用 S3 对照手写路径的执行与探针，覆盖寄存器溢出。

Tasks: T18.1–T18.14
Validation: <实际执行的命令、用例和结果>
```
