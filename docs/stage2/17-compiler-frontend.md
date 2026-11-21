# 第 17 章 编译器前端

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把"手写 ISA"替换为真实编译器：GLSL-like 子集源文 → 词法/语法/语义 → AST → SSA IR + CFG → 发散分析（一致值 SGPR / 发散值 VGPR）。前端不产出最终 ISA（后端第 18 章），但产出可快照、可重放、可对照 IR。IO 签名校验是本单元特有的树篱判据。设计说明见 `../design/compiler.md` §1~§3。
> 与本章相关的设计说明：`../design/compiler.md`（前端/S3 场景/IR 快照）；ADR-034（发散分析）；`../design/isa-abi.md` §3（ABI 分区，编译器不共享 hw_sim 内部结构）。
> 落地模块：`umd/shader/frontend`、`umd/shader/ir`、`apps/scene_s0_selftest`（编译自检）、`apps/scene_s3_triangle`（编译着色器）。

---

## 1. 目标与非目标

### 1.1 目标

1. GLSL-like 子集解析：类型（`float/vec2/vec3/vec4/int/uint/bool`）、变量、算术/位/比较/逻辑、if/else、for/while、函数（含递归上限保护）、`discard`。
2. 语义分析：类型检查（带行号的失败信息）、影子声明检查、未定义变量、四则运算类型提升。
3. AST → SSA IR + CFG：SSA 唯一赋值、显式 `φ` 节点、基本块 CFG；`phi` 归并只在收敛点。
4. 发散分析：标记每个值"波内一致（uniform）"或"发散（divergent）"；一致值可走 SGPR、发散走 VGPR（ADR-034）。
5. IR 快照稳定：同一源码编译产物（IR dump）两次一致，可用于回归与对照。
6. IO 签名校验：shader 输入/输出属性（`SV_Position`、`COLOR`、`TEXCOORD` 等）与 PSO 描述匹配；每个属性带 `stage` 字段，`location` 按 stage 基址换算；签名不匹配返回确定错误（带名称与序号）。

### 1.2 非目标

- 不做寄存器分配 / 指令选择 / 发射（第 18 章）。
- 不做优化 pass（第 18 章 M18.1）；本章 IR 不做重写控制流。
- 不做 LLVM IR（第 19 章）; 本章产出自有 SSA IR 快照。
- 不支持完整 GLSL 全量：只该子集 GLSL 词法；语法错误必须定位到行/列。
- 不产出最终 ISA blob（后端做）；前端到 IR 即交付点。

---

## 2. 单元框图

```
 源码(.comp) → 词法(tokens) → 语法(parser→AST) → 语义(类型/作用域签核)
             → 降级器(AST→SSA+CFG) → φ 完备化 → 发散分析 → IR 快照
                                                       │
                                                       ▼
                               （第 18 章后端消费 SSA）
```

- IR 快照含：模块级常量、全局数组、每函数 CFG + SSA 值表。
- 语法/语义错误一律带行号；`Result` 返回 `ParseError`/`SemanticError` 的映射（见错误契约 §3.9）。

---

## 3. 输入输出契约

### 3.1 前端接口（`umd/shader/frontend/frontend.h`）

```cpp
struct ShaderSource { const char* text; u32 len; const char* filename; };

struct IoSignatureAttrib {
    Stage stage;           // 属性所属阶段（VS/PS/HS/DS/GS/CS）
    u32 location;          // 阶段内槽位；按 stage 基址换算为 ABI 寄存器/分区
    std::string semantic;  // "POSITION"/"COLOR"/"TEXCOORD0"...
    TypeRef type;          // vec4/float/...
};

struct ShaderIo {
    std::vector<IoSignatureAttrib> inputs;
    std::vector<IoSignatureAttrib> outputs;
};

enum class CompilerResult { Ok, ParseError, SemanticError, IoMismatch, OutOfMemory };

struct FrontendOutput {
    CompilerResult result;
    std::string    error;        // 带 file:line:col
    std::vector<u64> irSnapshot; // 确定性序列化（快照）
    ShapeAnalysis  shape;        // 一致性标注
    ShaderIo       io;           // 提取的 PSO 匹配签名
};
Result CompileShader(const ShaderSource&, const ShaderIo& expectIo, FrontendOutput* out);
```

`CompileShader` 的双返回契约：API `Result` 只表示调用参数/输出缓冲是否有效；编译器诊断的唯一细节位于 `FrontendOutput`。参数非法或 `out==nullptr` → `Result::InvalidArgument`（无有效输出）；有效调用完成后函数返回 `Result::Success`，由 `out->result` 表示 `Ok/ParseError/SemanticError/IoMismatch/OutOfMemory`，具体源码位置与说明写入 `out->error`。前端编译错误不得再映射为第二套 `Result` 错误码。

- `expectIo`：由 PSO 描述提供（第 20 章管线）。io 校验：inputs/outputs 属性数量、语义名、类型必须与着色器声明一致；不匹配 → `IoMismatch`。
- `stage` 标明属性所属阶段；`location` 为阶段内槽位，按 stage 基址换算（VS 输入 `v[0..15]`、PS 输出 `kVgprBasePsOut`（`v[36..]`）等），不写裸号（Q-54）。

### 3.2 IR 结构（`umd/shader/ir/ir.h`）

```cpp
struct BasicBlock { u32 id; std::vector<u32> preds, succs; std::vector<Insn*> body; BlockKind kind; };
struct SsaValue { u32 id; TypeRef type; u8 kind; };   // uniform = 发散标注
struct PhiNode  { u32 dst; std::vector<std::pair<u32,u32>> incoming; }; // (BB,val)
struct IrModule { std::vector<Function> funcs; GlobalTable globals; };
```

- 序列化确定性：所有 id 按创建序、遍历序固定；快照可作为回归输入。

---

## 4. 内部数据流

### 4.1 语法制导 → SSA

```
tokens → parser（Pratt precedence）→ AST（表达式带类型标注）
→ 语义检查 → 函数内：符号表 {name→SsaValue}，变量每次赋值新 SSA id
→ CFG：控制流结构 → 基本块；if/for 产生合并块 + φ（简单 DCE 后保留）
→ φ 完备：对每 φ 读入块补一个拷贝（如果前端不做拷贝消除）——本实现直接产出 φ，不去重。
→ 发散分析：从输入属性/window 坐标根节点传播 uniform/divergent；
   控制流分支谓词本身视为发散种子。
```

### 4.2 发散分析规则

- 常量为 uniform；输入属性（非 `SV_Position` 之内的 stage 常量）按 stage 的 uniform/divergent 规则定种子：VS 的 `POSITION` 发散（每顶点不同），uniform 常量 uniform；PS 的 `POSITION` 发散。
- 二元/一元运算：结果 uniform ⇔ 两操作数 uniform。
- 分支条件发散 → 块内值发散（保守）；phi 的结果发散 ⇔ 任一 incoming uniform 不同源（保守标记发散）。
- 只标注，不改写控制流（ADR-034）。
- 内建 ID（`GET_THREAD_ID`/`GET_PRIM_ID` 等）只读 SGPR/专用寄存器，不得从属性区读取（Q-55）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T17.1 | ① 契约 | `ShaderSource`/`ShaderIo`/`CompilerResult`/`FrontendOutput`/IR 结构落头文件 |
| T17.2 | ② 主体 | 词法分析器：token 表、数值/标识符/运算符、行号列号 |
| T17.3 | ② 主体 | 语法分析器：表达式优先级、声明、语句块、if/for/while、函数 |
| T17.4 | ② 主体 | 语义分析：类型系统、作用域、未定义/重定义、四则提升 |
| T17.5 | ② 主体 | AST → SSA 降级：数值 SSA 化、函数调用展开、return 路径 |
| T17.6 | ② 主体 | CFG 构建 + φ 生成（if/for 合并） |
| T17.7 | ② 主体 | 发散分析引擎：uniform 传播、保守放大、`shape` 标注 |
| T17.8 | ② 主体 | IO 签名提取与校验（与 `expectIo` 比对） |
| T17.9 | ② 主体 | IR 确定性快照序列化 + 反序列化（回归用） |
| T17.10 | ② 主体 | 语法/语义错误定位（file:line:col）与 `Result` 映射 |
| T17.11 | ② 主体 | 函数递归深度上限与调用图检查（超限 `SemanticError`） |
| T17.12 | ② 主体 | 内置函数子集（`floor/frac/clamp/min/max/smoothstep` 等）实现为 IR 原语 |
| T17.13 | ② 主体 | 编译自检样本集与驱动：多源码（正确/错误）样本 + 自检断言（供 `scene_s0_selftest` 调用） |
| T17.14 | ③ 接入 | `scene_s0_selftest` 自检与 S3 编译着色器链路接入：源码 → IR 快照 → 后端（18 章）→ 执行 |
| T17.15 | ④ 验收 | A17：S3 编译场景 + 专有判据（IR 快照稳定、产物与手写一致、IO 签名错误确定）+ R17 回归项 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M17.1 | T17.1~T17.5 | 词法/语法/语义/SSA 降级，D17.1~D17.5 全绿 |
| M17.2 | T17.6~T17.10 | CFG/φ、发散分析、IO 签名校验、IR 快照、错误定位，D17.6~D17.10 全绿 |
| M17.3 | T17.11~T17.15 | 递归上限/内置函数/场景接入验收，A17 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D17.1 | 头文件独立编译；`CompilerResult` 枚举对齐 §3.1 | L1 契约 |
| D17.2 | token 序列正确（覆盖全部语法）；换行/注释跳过不计 token | L1 契约 |
| D17.3 | 合法表达式正确解析 AST（优先级、结合性断言）；非法语法抛 `ParseError` 且行号正确 | L1 契约 |
| D17.4 | 类型不匹配（`vec4 + float`）→ `SemanticError` + 行列；未定义变量 → 定位 | L1 契约 |
| D17.5 | SSA 唯一赋值：同变量三次赋值产生 3 个 SSA id；引用正确 | L1 契约 |
| D17.6 | 含 if/for 的代码 CFG 正确（preds/succs）；φ 就位于合并块 | L1 契约 |
| D17.7 | uniform 表达式标 uniform、发散表达式标 divergent；`a++`（发散）传播正确；分支条件发散保守放大 | L1 契约 |
| D17.8 | `expectIo` 与声明匹配 → Ok；适配缺失/多余属性/语义名错 → `IoMismatch`（报属性名与序号） | L1 契约 |
| D17.9 | 同源码两次编译 IR 快照逐字节一致；快照反序列化后重打印一致 | L1 契约 |
| D17.10 | 错误集所有错误定位到 `filename:line:col`；`Result` 映射确定 | L1 契约 |
| D17.11 | 深递归（>200 帧）编译返回 `SemanticError` 并自终止 | L1 契约 |
| D17.12 | 内置函数编译为 IR 原语；无未知内建 `SemanticError` | L1 契约 |
| D17.13 | S0 编译器自检：正确样本全部 Ok；错误样本得到对应错误类 | L1 契约 |
| D17.14 | S0 前端用例全绿：合法源码生成规范化 IR 快照，非法语法/语义/IO 签名返回明确错误与源码位置；不要求本章产物执行或出图 | L1 + L2 |
| D17.15 | A17 全绿：前端 IR、语法/语义错误、源码位置与 IO 签名单测通过；A17 不依赖第 18/19 章后端 | L1 + L2 |

---

## 7. 场景与判据（A17）

**绑定场景：S3 `scene_s3_triangle`（编译着色器）、S0 `scene_s0_selftest`（编译自检）。**

| 项 | 内容 |
|---|---|
| 输入 | S3：预置若干 GLSL-like VS/PS 源码（含分支）；S0：正确/错误样本集（每语法/语义类各一） |
| 产物 | IR 快照、编译产物 `.bin`（后端出）、错误报告 `{file:line:col, code}` |
| 主判据 | 本章只判源码解析、语义检查、规范化 IR 与 IO 签名；S0 前端自检全绿。S3 编译产物出图归第 18 章 A18 / 第 19 章 A19 收口 |
| 本章专有判据 | ① IR 快照稳定（同源码两次逐字节一致）；② 错误源码返回确定的 `CompilerResult` 和源码位置；③ IO 签名不匹配返回确定错误。编译产物与手写 ISA 的执行等价由 18/19 章验收 |
| 回归项 | R17：E0~E5（手写 ISA 路径仍可用）不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 后端寄存器分配/发射（18）；优化 pass（18.1）；LLVM 对照（19）把 IR 快照作为对照输入 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | IR id/遍历序、φ 序确定；快照序列化确定性 |
| 时间无关 | 编译不依赖时间/随机 |
| 可观测 | IR 快照 dump、错误定位、`--trace=compiler` 每阶段一行 |
| 契约影响 | 前端 IR 结构 + `CompilerResult` + IO 校验冻结；后继经 SSA 对接 |
| 简化登记 | 手写 ISA → 真实编译前端；ABI 分区不可被编译器覆盖 |
| 计算兼容 | 前端不假设图形语义；compute kernel 走同一前端（S18 消费 IR） |
| 配置与回退 | `use_compiler` 为 UMD 前端内部选项（默认开，字段 `FrontendOptions::useCompiler`），不是全局配置键；手写 ISA 路径保留兼容 |
| 调试钩子 | `--trace=compiler`：lex/parse/semantic/ir 各一行：`--dump-ir` |
| trace 契约 | 不进入判据 |
| 契约变更 | IR 形态、IO 签名语义、SSA/φ 语义为冻结项；变更走 ADR 回归 S3/S0 |
| 校验归属 | L1（结构/错误定位）+ L2（S3 编译路径输出与 ref_model） |
| 双策略 | IR 与执行策略无关；编译产物在两策略下输出一致 |
| 阶段衔接 | 为第 18 章后端提供 SSA + 发散标注；第 19 章对照输入 = 本章 IR 快照 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 着色器来源 | 手写 ISA → 源码编译 | —（定案） |
| IR | 内存内结构 → SSA + CFG + 快照 | — |
| 发散 | 无 → uniform/divergent 标注 | —（后端消费） |
| 输入绑定 | 固定 `RenderVertex` | 22（`VertexInputLayout` 动态绑定） |

---

## 10. 待定项

- 内建函数子集范围是否随 S3 场景扩展（如 `texture` 采样内建）——纹理编译入口在第 21 章定案时统一注册，本章留空原语位。
- IO 签名校验对"未使用的额外输出属性"采取宽容还是严格：安全选严格（缺一不可），若 S3 出现 false negative 再放松，需记录决策。
- IR 是否会演进出"块间桶序"以加快后端线性扫描，由第 18 章寄存器分配结果回馈；不冻结 IR 块序语义之外的形状。
