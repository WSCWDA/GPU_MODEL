# GPU 全栈软模型：设计思路、章节与子模块功能总览

> 版本基准：WSCWDA/GPU_MODEL 的 main 分支，读取基准提交 `9d92c4e8cbaafb35aa0896bfe8cfec7e3daa4ee8`。  
> 本文是对仓库设计文档的结构化提取与归纳，不表示代码已按文档全部实现或通过测试。章节功能以各章“目标/模块/输入输出”描述为主；章节间出现冲突时，仍需以单独的设计裁决为准。

## 1. 项目定位与总体设计思路

该项目希望在 CPU 上构建一个 GPU 软件栈的行为模型：让应用可以通过用户态接口提交图形或计算任务，命令经过驱动、固件和命令处理器，最终由 CPU 实现的 GPU 硬件模型执行，并通过内存写回、显示输出、查询或 fence 呈现结果。

它模拟的是 GPU 的**架构语义与软件栈流程**，不是用 CPU 伪装出一块真实 GPU。实际 CPU 的 L1/L2 缓存只影响模拟器自身运行速度；若要研究 GPU 缓存行为，L1/L2 必须作为模拟器中的显式模块建模，包括容量、line、替换策略、写策略和统计。

设计的递进路线是：

1. **Stage 1（第 1–9 章）：** 建立最小可运行模型，先形成一次 draw 的闭环：契约、设备/Ring、平坦内存、SIMT wave、ISA、最小图形管线、纹理、帧输出和端到端回归。
2. **Stage 2（第 10–37 章）：** 将最小模型扩展为更完整的 GPU 系统：KMD/Firmware、虚拟内存、缓存、驻留、调度、编译器、资源绑定、图形固定功能、多队列、查询、错误恢复和计算执行。
3. **Stage 3（第 38–42 章）：** 在已有 HLI 和 Compute 引擎上增加计算驱动对象、kernel 装载与启动、stream/event、协作功能接入以及双入口验收。

核心依赖方向为：

`common → hw_sim → firmware → kmd → hli → umd → apps`

其中 `common` 定义跨层类型和 ABI；`hw_sim` 是执行 GPU 行为的主体；Firmware/KMD 管理启动、提交和内存映射；HLI 提供统一对象接口；UMD 将外部 API 或计算运行时映射到 HLI；应用场景负责构造输入与验收。

## 2. 分层与主要子模块职责

| 层/子模块 | 主要职责 | 代表功能 |
|---|---|---|
| `common` | 跨模块契约与值类型 | 地址/句柄、Result、packet、ISA/Shader ABI、像素格式、日志和配置 |
| `hw_sim/core` | 模拟设备和命令执行控制 | Device、MMIO、Ring、CP、Interrupt、Fence；解析包并派发给硬件单元 |
| `hw_sim/memory` | GPU 地址与存储层次 | 平坦内存、页表/TLB、VRAM/GART/System heaps、L1/L2、写缓冲、驻留与逐出 |
| `hw_sim/shader` / `isa` | SIMT 与指令执行 | Wave、VGPR/SGPR、lane mask、分歧栈、调度器、ISA 解释器、subgroup、LDS/原子/barrier |
| `hw_sim/frontend` / `raster` / `om` | 图形管线 | IA/VGT、裁剪与 PA、光栅和插值、MSAA/可见性、深度/模板/混合/MRT |
| `hw_sim/texture` / `geometry` | 采样与几何阶段 | 纹理格式、tiling、mip/LOD、filter/address；HS/Tessellator/DS/GS |
| `hw_sim/periph` / `query` / `se` | 外设与观测/并行 | Copy Engine、Display/scanout、Query、SE 分区与 tile/波任务调度 |
| `firmware` | 设备启动与固件侧控制 | boot、ring 编程、doorbell、上下文加载、watchdog |
| `kmd` | 内核态驱动职责模型 | DDI、设备初始化、页表/GART/TLB、上下文/调度、提交、中断、复位 |
| `hli` | 统一硬件无关对象接口 | Device、Queue、Resource、View、DescriptorHeap、Pipeline、CommandList、Capability |
| `umd/graphics` | 图形 API 与 HLI 适配 | Vulkan/D3D12/GL 前端、PAL 资源与绑定、命令记录、队列 |
| `umd/shader` | Shader 编译链 | 前端解析/语义/IR，优化、寄存器分配、指令发射，LLVM 对照后端 |
| `umd/compute` | 计算驱动对象模型 | Context、Module、Function、LaunchKernel、Stream、Event |
| `apps` | 可重复的场景入口 | E0–E5 初级图形场景；S0–S19 高级系统/图形/计算场景 |
| `tools` | 教学、调试和判定工具 | 汇编/反汇编、CPU ref model、image diff、scene/spec runner、日志和内存审计 |

### 2.1 主要数据通路

**图形 draw 通路：** 应用创建资源/视图/管线并记录 DRAW → HLI 生成命令 → KMD/Firmware 写 Ring 并敲 doorbell → CP 取指派发 → IA 装配 → VS → 裁剪/图元装配 → 光栅/插值 → PS → OM 深度/模板/混合写回 → Display/Download/fence。

**计算 dispatch 通路：** 应用创建 Context/Module/Function 并准备参数 → Compute UMD 打包参数并调用 HLI Dispatch → 命令进入 Ring → CP 派发到 Compute engine → workgroup 分配给 CU/SIMD、拆成 wave 执行 → 访问寄存器/LDS/全局内存 → 写回 → fence/event/query 通知完成。

## 3. Stage 1：最小可运行模型（第 1–9 章）

| 章 | 章节主题 | 主要功能 | 对应子模块/产物 |
|---:|---|---|---|
| 1 | 契约最小集 | 定义基础类型、错误返回、命令包头/包类型、颜色字节序、日志/断言和配置；用自检固定公共契约 | `common/{types,result,packet,format,log,config}`；E0 自检 |
| 2 | 设备 + 单环 | Device/MMIO、单 Ring、doorbell、最小 CP 状态机、STREAM_HEADER/NOP/FENCE 和 Wait 闭环 | `hw_sim/core/{device,mmio,ring,cp,fence}`；薄提交门面；E1 提交子集 |
| 3 | 平坦内存 | GPU VA 平坦映射、内存子系统接口、bump 分配、Upload/Download；SET_STATE_BASE、CLEAR、COPY_MEM | `hw_sim/memory/flat`、CP 内存包处理 |
| 4 | SIMT 单波 | 固定 32-lane wave、lane mask、VGPR/SGPR、无分歧线性执行、内建 ID 和寄存器终态 | `hw_sim/shader/wave`、初版 ISA 执行器 |
| 5 | 最小 ISA | 64-bit 指令编码、约 20 条指令、指令语义表、汇编/反汇编往返 | `common/isa`、`hw_sim/shader/isa`、`tools/isa_asm`、`isa_disasm` |
| 6 | 固定功能最小集 | 一条非索引 DRAW 完成 IA→VS→裁剪/视口→光栅→PS→OM，输出首个三角形像素 | `frontend/{ia,clip,viewport}`、`raster/{rasterizer,interp}`、`om` |
| 7 | 纹理最小集 | RGBA8 上传、nearest 采样、UV 地址计算、clamp 和 CPU 参考模型逐字节比对 | `hw_sim/texture/nearest`、SAMPLE 指令、`ref_model/image_diff` |
| 8 | 帧输出 | back/front 双缓冲、Clear、flip、Download、raw/PNG 导出 | `periph/display`、提交门面、E1 帧输出 |
| 9 | 端到端验收 | 汇总 E0–E5 场景，验证多 draw 状态切换、重复运行确定性和 headless 回归 | `apps/scene_e5_end_to_end`、spec runner/基线 |

**Stage 1 的学习结果：** 从空项目走到 CPU 上执行一条简单图形命令并得到可比对的像素。阶段内应尽量避免引入多线程、页表、复杂 API 兼容和性能模型。

## 4. Stage 2：系统与图形能力扩展（第 10–37 章）

| 章 | 章节主题 | 主要功能 | 对应子模块/产物 |
|---:|---|---|---|
| 10 | 设备 + 环 + CP | KMD/FW 启动、MMIO/Ring/CP 完整提交、包解析、Engine 握手、fence/中断 | `firmware/{boot,scheduler}`、`kmd/{ddi,submission,interrupt}`、`hw_sim/core` |
| 11 | 地址翻译 + 堆 | 两级页表、VA→PA、ASID/TLB、VRAM/GART/System heap 和内存池 | `hw_sim/memory/{translation,heaps}`、`kmd/memory` |
| 12 | 缓存 + 写缓冲 | 模拟每 CU L1、全局 L2、缓存行/命中统计、写回/合并与失效 | `hw_sim/memory/cache`、缓存旁路和统计 |
| 13 | Copy Engine + Display | 独立 Copy 引擎、显示 scanout/flip/vsync，扩展第 3/8 章同步内存路径 | `hw_sim/periph/{copy,display}`、CP engine dispatch |
| 14 | 驻留 + 上下文 | 资源驻留、页错误、LRU 逐出、多上下文/ASID、状态保存恢复 | `hw_sim/memory/residency`、`kmd/context,scheduler`、Firmware context |
| 15 | 控制流 + 波调度 | lane 分歧、控制流栈、重收敛、多波就绪队列、CU/SIMD 调度 | `hw_sim/shader/{divergence,scheduler,wave}` |
| 16 | ISA + 汇编器 | 扩展完整 opcode 区段、编码/解码、ISA 自检和汇编工具 | `hw_sim/isa`、`tools/isa_asm/isa_disasm` |
| 17 | 编译器前端 | GLSL-like 子集词法/语法/语义分析，AST→SSA IR/CFG | `umd/shader/{frontend,ir}` |
| 18 | 编译器后端 | 优化、uniform/varying 分析、寄存器分配、scratch spill 与 ISA 发射 | `umd/shader/{opt,regalloc,emit}` |
| 19 | LLVM 对照后端 | 将同一 IR 映射到 LLVM，比较代码生成并对运行语义与参考执行结果 | `umd/shader/llvm`、`tools/llvm_compare` |
| 20 | 资源与绑定 | Resource/View/DescriptorHeap、图形/计算 Pipeline、Root Signature、资源状态和参数区 | `umd/graphics/pal/{resource,binding,pipeline}`、HLI 资源对象 |
| 21 | 纹理单元 | 多格式解码、线性/tile 布局、mip/LOD、filter、sampler 与寻址 | `hw_sim/texture`、sampler 对象 |
| 22 | 输入装配器 IA | 多流/多属性顶点布局、索引格式、primitive restart、step rate/实例化 | `hw_sim/frontend/ia` |
| 23 | VGT / SPI | 顶点流生成与切片、VS 波派发、顶点流缓冲 | `hw_sim/frontend/vgt_spi` |
| 24 | 裁剪 + PA | 六平面裁剪、guard band、视口变换、图元装配及 clip/cull distance | `hw_sim/frontend/clip_pa` |
| 25 | 光栅器 | top-left fill rule、2×2 quad、插值/scissor、点线和派生属性 | `hw_sim/raster/{rasterizer,interp}` |
| 26 | 输出合并 OM | 深度/模板、blend、MRT、格式转换、像素输出掩码 | `hw_sim/om` |
| 27 | MSAA + 可见性 | 多采样覆盖、每样本深度/混合、resolve、early-Z/Hi-Z/fast clear | `hw_sim/raster/msaa`、`visibility` |
| 28 | Tess + GS | Patch 输入、HS、细分器、DS、GS 和几何阶段数据通路 | `hw_sim/geometry/{hs,tessellator,ds,gs}` |
| 29 | 波内协作 | ballot、shuffle、broadcast、reduce 和波内协作语义 | `hw_sim/shader/subgroup` |
| 30 | 组内协作/LDS | LDS/STS、bank、原子操作、跨 wave barrier、内存序/作用域 | `hw_sim/shader/shared`、同步原语 |
| 31 | SE + CU 阵列 | SE/tile 分区、CPU worker 执行、CU/SIMD/wave 映射、occupancy 和并行一致性 | `hw_sim/se`、并行调度器、S11/S18 |
| 32 | 队列 + 同步 | Graphics/Compute/Copy 队列、timeline semaphore、SIGNAL/WAIT、barrier 与所有权转移 | `hw_sim/core/multiqueue`、KMD/HLI queue |
| 33 | 命令链 + 多引擎 | IB 链、间接 DRAW/DISPATCH、GPU predicate 和 DMA 切片拷贝 | CP 扩展、Copy Engine、命令生成 |
| 34 | 查询 | occlusion、timestamp、pipeline stats、query pool 与 GPU predicate 输入 | `hw_sim/query`、HLI QueryPool |
| 35 | API 后端/HLI | 冻结 HLI 对象面；Vulkan/D3D12/GL 适配同一 HLI；计算驱动已迁至 Stage 3 | `hli`、`umd/graphics/api`、`umd/graphics/pal` |
| 36 | 错误路径 | 错误定位、故障注入、watchdog、引擎复位/重建、日志与内存审计 | `kmd/reset`、`firmware/watchdog`、审计工具 |
| 37 | 计算管线 | Compute 引擎派发、参数/dispatch 语义、workgroup-wave-LDS 协作，并以 HLI 直连验证 S18/S19 | CP/Compute engine、29–34 共用单元、计算场景 |

**Stage 2 的学习结果：** 学习 GPU 系统真实的“周边机制”：内存映射和缓存、调度、编译器、完整图形管线、资源和同步。它覆盖面最大，也是章节拆分与依赖治理最需要加强的阶段。

## 5. Stage 3：计算驱动开发线（第 38–42 章）

| 章 | 章节主题 | 主要功能 | 对应子模块/产物 |
|---:|---|---|---|
| 38 | 计算驱动骨架与 Module 装载 | Context/Module/Function 生命周期、MUSA_MODULE_HDR 解析、符号表和 kernel 入口 | `umd/compute`、计算模块格式 |
| 39 | LaunchKernel | ArgPack/KernelArgsLayout、参数打包和边界校验、设置 kernel 并提交 dispatch | `umd/compute` 参数与 launch 路径 |
| 40 | Stream/Event 与异步计算 | stream 到 queue 的映射、event 的 Record/Wait/Query、跨图形/计算队列依赖 | `umd/compute` stream/event、HLI/KMD queue |
| 41 | 组内协作与间接触发 | 计算侧三维 ID/边界、协作原语包装、间接 dispatch、UAV 写回与查询接入 | Compute UMD + 第 29–34 章硬件能力 |
| 42 | 双入口与端到端验收 | 比较 HLI 直连和计算驱动入口，覆盖 S18/S19、错误映射及 serial/parallel 一致性 | `apps` demos、spec runner、A42 |

## 6. 章节串联与关键依赖

### 图形主线

`1 契约 → 2 提交 → 3 内存 → 4/5 执行与 ISA → 6 图形管线 → 7 纹理 → 8 帧输出 → 9 端到端`

高级图形扩展主要沿：

`10 KMD/CP → 11/12 内存映射与缓存 → 20 资源绑定 → 21–28 图形阶段 → 32/33 队列与命令扩展 → 34 查询 → 35 HLI/API`

### 计算主线

`4/5 wave 与 ISA → 15/16 控制流与完整 ISA → 17/18 编译器 → 20 资源/参数 → 29/30 subgroup 与 workgroup 协作 → 31 调度 → 32/33 队列与间接命令 → 37 HLI 直连计算 → 38–42 计算驱动与验收`

### 横向模块

- **内存与缓存：** 3 平坦内存 → 11 页表/堆 → 12 L1/L2/写缓冲 → 14 驻留 → 20 资源对象。
- **命令与同步：** 2 单环 → 10 KMD/FW/CP → 32 多队列和 timeline → 33 IB/indirect/predicate。
- **执行与并行：** 4 单 wave → 15 控制流/多 wave → 29 subgroup → 30 LDS/barrier → 31 SE/CU/workers。
- **验证与观测：** 每章场景/E-S case → CPU reference model 和探针 → spec runner 汇总；raw 数据用于机器比对，日志/trace 用于诊断。

## 7. 阅读说明与设计边界

1. 这是“文档功能地图”，不是实现完成度清单。模块名和功能来源于设计文档，尚未对每个代码目录进行实现审计。
2. “模拟器使用 CPU L1/L2”与“模拟 GPU L1/L2”是两种不同目标：前者是 host 优化，后者是 GPU 行为建模；项目设计里的第 12 章对应显式 GPU cache model。
3. 一些章节跨越多个学习层级，例如第 20、31、35、37、41 章，目录中的功能清单并不自动证明任务规模适合教学。若用于课程/自学，应将它们拆成可运行的里程碑和选修扩展。
4. 模块边界的最终裁决，应以 `architecture-design.md` 的分层职责、`roadmap.md` 的依赖/门禁，以及各 `docs/design/*.md` 的接口契约一起阅读。

## 8. 来源文档

- [架构设计](https://github.com/WSCWDA/GPU_MODEL/blob/main/docs/architecture-design.md)
- [总开发计划](https://github.com/WSCWDA/GPU_MODEL/blob/main/docs/roadmap.md)
- [Stage 1 文档目录](https://github.com/WSCWDA/GPU_MODEL/tree/main/docs/stage1)
- [Stage 2 文档目录](https://github.com/WSCWDA/GPU_MODEL/tree/main/docs/stage2)
- [Stage 3 文档目录](https://github.com/WSCWDA/GPU_MODEL/tree/main/docs/stage3)
- [design 模块说明目录](https://github.com/WSCWDA/GPU_MODEL/tree/main/docs/design)

