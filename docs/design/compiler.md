# 编译器管线 — 独立模块设计

> 定位：定义着色器源码（GLSL-like 子集）到 ISA blob 的完整编译管线：前端（词法/语法/语义 + SSA IR + 分歧分析）、后端（优化 + 寄存器分配 + 指令选择与发射）、以及 SSA ↔ LLVM IR 对照后端。
> 落地章：第 17 章编译器前端（M17.1~M17.3）、第 18 章后端（M18.1~M18.3）、第 19 章 LLVM 对照后端。
> 权威来源：从 `architecture-design.md` §4.4 拆出，并吸收 Mesa NIR 形状对照（原 §4.7.3，见本文件 §1）、`roadmap.md` §2.8、ADR-012（LLVM 对照后置）、ADR-034（标量/向量与发散分析）与 `isa-abi.md`（编码与 ABI 唯一权威）。

---

## 1. 管线总览

```
源码（GLSL-like 子集）
  → 词法/语法/语义（17）
  → AST → SSA IR + CFG（17）
  → 常量折叠 / DCE / CSE / 复制传播（18）
  → 分歧分析（18）
  → 线性扫描寄存器分配（18）
  → 指令选择 + 64 bit 发射（18）
  → ISA blob（16 编码）
  →（对照）SSA ↔ LLVM IR → JIT → 规范程序语义比对（19）
```

## 2. 阶段与硬约束

| 阶段 | 输入 | 输出 | 硬约束 |
|---|---|---|---|
| 前端 | 源码文本 | AST + 符号表 | 类型检查失败必须带行号 |
| IR | AST | SSA + CFG | 每个值单一赋值；φ 节点显式 |
| 优化 | SSA | SSA | 不做任何循环变换（含 LICM/循环不变量外提）；优化不得改变可观测结果 |
| 分歧 | SSA | 分歧标注 | 只标注，不重写控制流 |
| 寄存器分配 | SSA | 物理寄存器 | 不覆盖 ABI 固定分区（`isa-abi.md` §3） |
| 发射 | 分配后 IR | ISA | 编码与第 16 章逐位一致 |
| LLVM 对照 | SSA | LLVM IR | 只做对照；允许合法编码与寄存器分配差异，以 ABI/ISA 合法性和规范程序语义验收 |

规则：

1. 第 19 章对照相同规范 IR 的 ABI、指令合法性与执行语义；两后端可以有不同的指令选择、寄存器分配和 blob 字节序列，不以逐字节一致作为阻断判据，不阻塞第 20 章。
2. **不分歧优化**：分歧分析只标注、不重写控制流；一致值走 SGPR、发散值走 VGPR（ADR-034）。
3. 编译器与 `hw_sim` 的唯一连接是 `common` 的 ISA 定义与 Shader ABI，不共享内部数据结构；不引入真机 / 外部编译后端作参照。
4. `umd/graphics/api`（35.2~35.4）与 `umd/compute`（第 37 章收口、stage3 38~42）只消费已编译的 ISA/blob，**不重复实现编译器**（`architecture-design.md` §2.3 分层职责表）。

## 3. 子里程碑拆分（任务归属见 `roadmap.md` §2.1）

| 章 | 子里程碑 | 范围 | 验收 |
|---|---|---|---|
| 17 | M17.1 | 词法/语法/语义分析 | S3 + S0（IR 快照稳定；编译产物与手写 ISA 一致） |
| 17 | M17.2 | 中间表示 + 发散值→VGPR 分析 | S3 |
| 17 | M17.3 | 着色器 IO 签名/属性校验 | 签名不匹配返回确定错误 |
| 18 | M18.1 | 优化（常量折叠/死代码/CSE/复制传播；不含 LICM，禁止循环变换） | 优化开/关一致；寄存器溢出统计可输出 |
| 18 | M18.2 | SGPR/VGPR 分配、溢出 | S3 |
| 18 | M18.3 | 指令选择、发射顺序 | 编码逐位一致 |
| 19 | — | SSA ↔ LLVM IR 对照 | ABI/ISA 合法性与规范程序语义一致；不阻塞第 20 章 |

## 4. 与 Mesa NIR 的关系

只借鉴 IR 形状（SSA + 显式 φ、CFG），**不照搬指令集、不引入 Mesa 代码**（本文件 §1）。NIR 是"形状参考"而非实现对照。

## 5. 与一致性验收的关系

- 三路径需满足同一 ABI 与规范程序语义判据；手写 ISA、编译产物、LLVM 路径可使用不同合法编码。
- 初级不引入编译器，全部手写 ISA（初级简化登记，`architecture-design.md` §7.5），高级第 5→16/17~19 替换。
- 计算 kernel 由同一编译器产出（`Stage::Compute` 前端不假设图形语义，`compute-line.md`）。

### 5.1 计算 kernel 编译路径与溢出落点（P1-8，规格补齐）

- `Stage::Compute` 前端不假设图形语义：内建 ID 走计算命名（`GET_THREAD_ID / GET_GROUP_ID` 等，`isa-abi.md` §2 内建区段），`LDS/STS` 走 shared 区，原子与组内 barrier（`BARRIER`）沿用 29/30 章共用实现（不新增计算特化分支，ADR-032）；
- 编译器按 `architecture-design.md` §3.12（M-01~M-08）识别 `tid/gtid/waveId/blockIdx`：`blockIdx` 由驱动经参数区传入，`gtid` 由 block 推导，**不引入图形顶点/像素内建**；
- **溢出落点**：寄存器溢出与 private 数组下沉到 GART scratch（per-thread，`architecture-design.md` §3.13、`memory-system.md` §3），per-thread base 由驱动经参数区传入；每线程最多 4 KiB、4 B 对齐、不清零，因此每 wave（32 lane）最多 128 KiB；每 dispatch 按有效线程数分配 scratch，受 GART 可用容量约束，分配失败在 dispatch 前返回 `OutOfMemory`。该 scratch 不占用 CU 的 LDS/VGPR/SGPR 驻留预算，也不改变 occupancy。

## 6. 契约变更

- `CompileShader` 双返回契约见第 17 章 §3.1：`Result` 只表示调用有效性，解析/语义/IO 编译结果唯一写入 `FrontendOutput.result/error`。
- IR 形态、IO 签名语义、ABI 分区均属冻结契约，变更走 `roadmap.md` §7.4。
- opcode / ABI 分区来自 `isa-abi.md`，此处不重复定义。
