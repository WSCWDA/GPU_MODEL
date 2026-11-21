# 计算线（含计算驱动）— 独立模块设计

> 项目目标、范围、阶段与交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

> 定位：定义高级阶段引入的第二条纵向功能线：计算最小通路、workgroup/wave 映射语义、内存作用域与 barrier、MUSA 对象模型的计算驱动、以及图形线内的预留点与启动条件。计算线通路与共用单元以独立第 37 章收口（ADR-013 修订，2026-11-19）；**计算驱动对象模型与 ABI 的驱动专属细节迁出到 `docs/design/compute-driver.md`**，计算驱动开发任务的 **stage3 新章 38~42**（`docs/roadmap.md` §0.2/§2.1 登记、`docs/stage3/`）作为 MUSA 风格计算驱动开发线承载。
> 落地章：共用单元在第 20（kernel 参数与绑定）、29（波内协作）、30（组内协作）、31（三维 workgroup 映射）、32（队列与同步）、33（间接派发）、34（查询）、37（计算管线收口）章；计算驱动开发线在 stage3 新章 38~42（`docs/stage3/`）。
> 权威来源：从 `architecture-design.md` §4.9 拆出，吸收 `roadmap.md` §2.1（两条纵向线与共用单元）、计算最小通路 / MUSA 借鉴 / 场景判据 / 门禁（原 §4.7，已并入本文件）、`roadmap.md` §6.1 预留点与 ADR-009 / ADR-013 / ADR-032；驱动对象模型与 ABI 详见 `docs/design/compute-driver.md`。

---

## 1. 计算最小通路

```
kernel（编译器产物或手写 ISA） → module / function 对象 → kernel 参数缓冲
→ dispatch(grid, block) 命令包 → CP 派发 → workgroup 分配到 CU / SIMD
→ wave 调度 → LDS / 原子 / barrier → 全局内存或图像写回
→ fence / 完成通知 → 查询与统计
```

与图形线的三点本质差异：

1. 没有 IA / 光栅 / OM，输入是 grid 与参数缓冲，输出是内存而不是像素；
2. **workgroup 是调度单位、wave 是执行单位**，二者映射必须显式建模（block 维度、越界 work-item、grid 边界的部分 workgroup）；**barrier 是组内跨 wave 同步**（组内全部 wave 到达才放行）；
3. 内存作用域（workgroup / device）与 barrier 是正确性核心，不是可选优化（详见 `../docs/design/sync-primitives.md` §3）。

## 2. MUSA 计算驱动分层（stage3 38~42 借鉴，非 API/二进制兼容）

| MUSA 概念 | 本项目对应 | 落位 | 一期 |
|---|---|---|---|
| Runtime API（alloc / memcpy / launch） | 计算驱动的用户态入口 | 38~42 | 做 |
| Driver API（context / module / function / stream / event） | 计算驱动对象模型 | 38~42 | 做 |
| kernel 参数与 launch config | HLI dispatch 包 + 参数缓冲 | 20、33、35.1 | 做 |
| stream / event | 队列 + timeline semaphore / fence | 32 | 复用 |
| unified / pinned memory | 内存域模型（HOST_VISIBLE / HOST_COHERENT） | 11 | 做 |
| occupancy / 性能调优 | 只输出 wave 数与占用统计，不建模占用率 | 31、34 | 不做性能 |
| MUSA 二进制 / API 兼容 | 不做 | — | 不做 |

**对象粒度（38 章实现前冻结；实现细节与 ABI 见 `compute-driver.md`）**：

- 一个 HLI device 可创建多个计算 `Context`；每个 Context 绑定唯一 KMD context/ASID（Context:ASID=1:1）。销毁不隐式同步；有子对象或硬件提交引用时返回 `Busy`，stream drain 归计算驱动第 40 章。
- `module` = 自有头 `MUSA_MODULE_HDR` + 代码段 + 符号表（`name / entryOffset / paramBytes / vgprCount / ldsBytes`），常驻 GART；不引入 ELF/COFF，不做重定位。
- `function` = module 内入口（`codeBaseVa + entryOffset` + 参数布局）；分支用 pc 相对偏移，因此加载不需要地址修补。
- `stream` = 1:1 映射一个 HLI 队列，stream 内顺序即队列顺序；Graphics/Compute 等不同队列可各自拥有 stream，单个 stream 不跨队列；优先级创建时声明、不可变。
- `event` 拥有独立 timeline semaphore 与单调 value；Record 在调用 stream 中插 `SIGNAL`，Wait 可跨队列在其他 stream 插 `WAIT`，Query 非阻塞读取完成值；不复用队列 fence 值域。
- kernel 参数由驱动打包：应用按签名提交 `ArgPack{ptr, bytes}[]`，驱动写入 GART `HOST_VISIBLE | HOST_COHERENT` 参数缓冲并生成 `KernelArgsLayout`（`architecture-design.md` 层间接口契约 §3.8）；应用直接给参数 VA 的 `LaunchWithParamBuffer` 留二期。
- 计算驱动（38~42）**不新增 HLI 接口**：dispatch 走已有 `DISPATCH` 包与 compute pipeline，队列与同步走第 32 章原语。

## 3. 对象模型 ↔ HLI 映射

| 对象 | 职责 | 映射到 HLI |
|---|---|---|
| Runtime API（alloc / memcpy / launch） | 用户态入口 | `IHliDevice` 资源接口 + `IHliQueue` 提交 |
| `Context` | 设备、内存域、模块与流的容器 | `IHliDevice` + 资源表 |
| `Module` | 一份 ISA blob + 符号表 | `IHliPipeline`（compute） |
| `Function` | 模块内入口 + 参数布局 | `IHliPipeline` + `KernelArgsLayout` |
| `Stream` | 顺序提交队列 | `IHliQueue` |
| `Event` | 完成标记 | timeline fence 值 |

## 4. 共用单元约束

`IShaderArray`、内存、缓存、队列、描述符、查询均为 workload 无关；硬件单元按包类型派发，不按上层是图形 UMD 还是计算驱动分支（ADR-032）。计算专属能力（subgroup、shared、atomic、三维 workgroup）只加能力，不改图形路径语义。

## 5. 图形线内的预留点（`roadmap.md` §6.1 权威表）

第 1、10、3/11、12、14、4/15、16、17/18、20、29/30、31、32/33、34、35.1 章在图形线推进时均为计算线预留（`Stage::Compute`、多 Ring 仲裁、统一 VA/TLB、subgroup/atomic/barrier opcode 区段、kernel 参数缓冲等），不实现在计算驱动之前。

## 6. 场景判据与门禁

- 专属验收入口：S18（计算最小通路）、S19（计算内存与并发）；S10 担组内归约与原子，S12 担多引擎交错，S14 担全阶段 UAV。
- 计算线门禁：
  1. S18/S19 在 `serial` / `parallel` 下硬判据一致；
  2. 共用单元不得出现"图形专用"或"计算专用"分支；HLI 侧按命令包类型区分，硬件单元按包类型派发；
  3. 计算驱动只消费 HLI，不得直接调用硬件单元；
  4. 计算线接入后不得回退图形线 E0~E5 与 S0~S17。

## 7. 启动条件与验收（`roadmap.md` §6.3、§2.1 章 37~42）

- 启动条件：第 20 章绑定接口、第 29~31 章波内/组内/并行稳定、第 37 章计算通路收口、35.1 HLI 冻结后；计算驱动（stage3 38~42）才可落地，且不反向修改 HLI 接口（第 37 章 DoD 全绿纪律）。
- 验收：S18、S19 双策略硬判据一致（第 37 章 A37 主验收）；计算驱动入口（`umd/compute`）与直连 HLI 入口**双入口等价**（stage3 第 42 章 A-stage3 主验收，探针零容差）。
- 编号与落地：计算驱动开发线为 **stage3 新章 38~42**（ADR-047、`roadmap.md` §0.2/§2.1）；共用单元仍在 20、29~31、32~34、37 落地，且不新增 `hw_sim` 目录（ADR-009/032）。
