# GPU 驱动全栈软模型 — 架构设计

> 定位：定义分层、模块边界、数据流、接口契约与 ABI，回答"系统由哪些层和单元组成、它们如何协作、哪些契约不能破"。
> 文档职责：项目目标、范围、阶段与交付以 [任务规划](CPU模拟GPU教学项目_任务规划与实施规格.md) 为主干；`roadmap.md` 展开章级任务与门禁；本文件负责模块树、数据流、层间契约与 ABI；`docs/design/` 细化模块内部机制；`scene-specs.md` 冻结场景输入与判据；stage 文档描述章节实现。本文件不新增章号与场景号。
> 决策记录见 `roadmap.md` §0.3 ADR 清单与 `docs/adr/`；原 `open-questions.md` 的 Q-01~Q-83 已全部定案关闭，本文件只写已定案内容。
> 模块设计：本文件 §4 为独立模块设计文档（`docs/design/`）索引；子系统的内部设计细节不在此内联，见各模块文档。

---

## 1. 定位、目标与边界

### 1.1 设计目标

1. **可解释**：每条 draw/dispatch 从提交到结果，都能在代码中定位到具体单元、具体包、具体指令与具体像素/内存位置。
2. **可校验**：每个硬件单元都有机器可判的输入输出契约；正确性由参考模型与探针判定，不靠肉眼。
3. **可替换**：初级用最简实现打通链路，高级逐章替换为硬件级实现；替换不改变场景入口与外部契约。
4. **双线兼容**：图形与计算共用内存、缓存、队列、ISA、调度与错误模型；共用单元不得为任一方向特化。
5. **可移植**：核心层不依赖操作系统；Windows 与 Linux 构建同一套代码，差异只在窗口与文件系统适配。

### 1.2 设计原则

| 原则 | 含义 | 反例 |
|---|---|---|
| 契约先行 | 先写头文件/枚举/断言，再写实现 | 先写执行逻辑，后补接口导致上下游反复改 |
| 单向依赖 | 严格 `common → hw_sim → firmware → kmd → hli → umd → apps` | UMD 直接 include 硬件单元内部头 |
| 单元自治 | 每个硬件单元只认命令包与自身状态 | CP 里写光栅逻辑、光栅里判断 UMD 类型 |
| 确定性优先 | 所有影响输出的顺序都显式定义 | 用线程完成顺序决定像素写入顺序 |
| 观测分离 | 硬判据只来自探针与参考模型 | 用日志文本或耗时判断正确性 |
| 简化可定位 | 每条初级简化在代码中有唯一落点 | 简化散落在多个单元，高级无法替换 |
| 能力显式 | 不支持的能力返回明确错误，不静默降级 | 把 MSAA 当 1x 跑完还报成功 |

### 1.3 非目标

不做 API/二进制兼容、不做真实 GPU 驱动、不做周期级时序、不做电源/热/ECC/虚拟化、不做稀疏资源与压缩纹理、不做可变 subgroup、不做细粒度页错误恢复、不做 TDR 级容错。二期能力只保留 capability bit，一期返回"不支持"。

### 1.4 正式阶段与能力档

| 阶段 | 章 | 分层形态 | 验收 |
|---|---|---|---|
| 初级 | 1~9 | 场景 → 薄提交门面 → 硬件单元 | E0~E5，全图逐字节 |
| 高级图形与共用底座 | 10~36 | 场景 → 图形 UMD/HLI → KMD → FW → 硬件单元 | S0~S17，探针 + 差分 |
| 计算管线 | 37 | HLI DISPATCH → 共用 KMD/FW/Compute 引擎 | S18/S19 的直连 HLI 路径 |
| 计算驱动扩展 | stage3 38~42 | 计算 UMD → HLI → KMD → FW → Compute 引擎 | S18/S19 双入口等价 |

能力四档见 `roadmap.md` §0.5：初级实现 / 高级一期 / 高级二期（只留 bit）/ 范围外。

---

## 2. 分层架构

### 2.1 初级分层（1~9 章）

```
apps/scene_e0 .. scene_e5
        │  include 只有 hw_sim/submit/facade.h
        ▼
hw_sim::submit（薄提交门面）
        │  submit(draw_desc) / wait()
        ▼
┌──────────────────────────────────────────────────────────┐
│ hw_sim                                                    │
│  core: device / mmio / ring / cp / fence                  │
│  memory/flat: 单 VA 空间 / 直接映射 / bump 分配            │
│  shader/wave: warp32 / VGPR / 谓词 / 单波执行              │
│  isa: 内存内指令结构 + 解释                                          │
│  frontend: IA / VS 分派 / 裁剪 / PA                        │
│  raster: 三角形填充 / 插值                                 │
│  om: 深度 / 颜色写回                                       │
│  periph/display: 扫描输出                                  │
└──────────────────────────────────────────────────────────┘
```

初级不建立 `firmware`、`kmd`、`hli`、`umd` 四层。薄门面直接组装命令包并驱动 CP；`Submit` 同步执行完命令，`Wait` 退化为 fence 值断言。目的只有一个：让 E0~E5 的链路最短、每一步都能单独解释。

### 2.2 高级分层（10~37 章；stage3 计算驱动 38~42）

```
apps/scene_s0 .. scene_s19
   │                    │                         │
   │ api_backend=vulkan │ api_backend=d3d12/gl    │ 计算驱动入口（stage3 38~42）
   ▼                    ▼                         ▼
umd/graphics/api/*   umd/graphics/api/*        umd/compute
   │                    │                         │
   ▼                    ▼                         │
umd/graphics/pal（resource / binding / pipeline / cmd_gen / queue）
   │                    │                         │
   └──────────┬─────────┘                         │
              ▼                                   ▼
            hli（35.1：设备 / 队列 / 资源 / 视图 / 描述符 / 管线 / 命令 / 能力）
              │
              ▼
            kmd（DDI / 内存 / 上下文 / 提交 / 调度 / 中断 / 复位）
              │
              ▼
            firmware（boot / ring 管理 / doorbell / 上下文加载 / watchdog）
              │
              ▼
            hw_sim（core / memory / shader / frontend / raster / om / geometry / query / se）
```

说明：

- **入口**：`api_backend=hli` 时场景直连 HLI，用于 10~34 的硬件单元验收；`api_backend=vulkan|d3d12|gl` 时经对应 UMD；计算场景在 stage3（38~42）之后经计算驱动入口（第 42 章双入口等价为主验收）。各路径的硬判据必须一致。
- **HLI 位置**：HLI 是"类 KMD 原生接口"，位于 KMD 之上、UMD 之下。它负责对象、资源/视图、描述符、管线与命令组装；提交统一走 KMD DDI，不复制 KMD 的页表/上下文接口。
- **KMD/FW 引入**：第 10 章起建立最小 KMD/FW（DDI、ring 写入、doorbell、fence 中断），后续章只在其上追加能力。
- **UMD 与计算驱动并列**：二者互不依赖，都只依赖 HLI 与 `umd/os`；编译器（17~19）只产出 ISA，35.x 只消费 ISA。

### 2.3 分层职责表

| 层 | 负责 | 不负责 | 允许依赖 |
|---|---|---|---|
| `common` | 类型、`Result`、包格式、ISA、Shader ABI、格式、日志、配置 | 任何执行逻辑 | 无 |
| `hw_sim` | 设备、环、CP、内存、缓存、SIMT、纹理、固定功能、光栅、OM、几何、查询、SE | 不认识 UMD/KMD 概念，不写驱动策略 | `common` |
| `firmware` | 上电、ring 管理、doorbell 响应、上下文加载、watchdog | 不解析 draw 语义 | `common`、`hw_sim`（只经 MMIO/环） |
| `kmd` | DDI、物理页与页表、GART、TLB 失效、上下文、提交、调度、中断、复位 | 不做 API 语义翻译 | `common`、`hw_sim`、`firmware` |
| `hli` | 设备/队列、资源/视图、描述符、管线、命令组装、能力查询 | 不做 API 兼容 workaround，不直接驱动单元 | `common`、`hw_sim`、`firmware`、`kmd` |
| `umd/graphics` | API 翻译、状态修补、PAL、命令记录、同步 | 不改写 HLI 输出包，不判断硬件实现 | `common`、`hli`、`umd/os` |
| `umd/compute` | context/module/function/stream/event、kernel 启动 | 不做 API/二进制兼容 | `common`、`hli`、`umd/os` |
| `apps` | 场景构造、提交、校验入口 | 不 include 硬件单元内部头 | 对应入口层 |

### 2.4 模块树（括号内为落地章；模块内部设计见 §4 对应的 `docs/design/` 文档）

```
common/                          (1)   types / result / packet / isa / shader_abi / format / log / config

hw_sim/
  submit/                        (2)   初级薄提交门面
  core/                          (2,10) device / mmio / ring / cp / interrupt / fence
  memory/
    flat                         (3)   单 VA / 直接映射 / bump
    translation                  (11)  两级页表 / TLB / VA → PA
    heaps                        (11)  VRAM / GART / System
    cache                        (12)  L1 / L2 / 写缓冲
    residency                    (14)  驻留 / 逐出 / 页错误入口
  periph/
    copy                         (13)  Copy Engine
    display                      (13)  scanout / flip / vsync
  shader/
    wave                         (4)   warp32 / VGPR / SGPR / 谓词 / 单波
    divergence                   (15)  分歧 / 控制流栈 / 重收敛
    scheduler                    (15)  波调度 / CU 多 SIMD
    subgroup                     (29)  ballot / shuffle / reduction / 导数
    shared                       (30)  shared memory / atomic / barrier / 内存序
  isa/                           (5,16) 编码 / 解释器 / 汇编 / 反汇编
  texture/                       (21)  格式解码 / tiling / mip-LOD / 过滤 / 寻址
  frontend/
    ia                           (22)  索引 / 拓扑 / restart / step rate / 实例化
    vgt_spi                      (23)  VS 分派 / 切片 / 顶点流缓冲
    clip_pa                      (24)  裁剪 / guard band / 视口 / 图元装配
  raster/
    rasterizer                   (25)  fill rule / 2x2 quad / 插值 / scissor / 点线
    msaa                         (27)  coverage / resolve
  om/                            (26)  depth / stencil / blend / MRT / cull / 格式转换
  visibility/                    (27)  early-Z / Hi-Z / fast clear
  geometry/                      (28)  HS / Tessellator / DS / GS
  query/                         (34)  occlusion / timestamp / pipeline stats
  se/                            (31)  SE 分区 / 线程池 / tile 调度

firmware/
  boot                           (10)  上电 / 握手
  scheduler                      (10,14) ring 管理 / doorbell / 上下文加载
  watchdog                       (36)  超时 / 复位

kmd/
  ddi, init                      (10)  DDI 入口 / 设备初始化
  memory                         (11)  物理页 / 页表 / GART / TLB 失效
  context                        (14)  ASID / 上下文切换
  submission                     (10,32) ring 写入 / doorbell / 提交队列
  scheduler                      (14)  上下文选择 / 抢占
  interrupt                      (10)  fence / 错误中断
  reset                          (36)  引擎复位 / 队列重建

hli/                             (35.1) device / queue / resource / view / descriptor / pipeline / command / capability

umd/
  graphics/api/{vulkan,d3d12,gl} (35.2~35.4)
  graphics/pal/
    resource                     (20)  Heap / Resource / Placement / 驻留
    binding                      (20)  描述符堆 / root signature / binding table
    pipeline                     (20,22~26) PSO / 着色阶段 / 固定功能状态
    cmd_gen                      (33)  命令记录 / IB / 间接命令
    queue                        (32)  队列 / fence / timeline semaphore
  shader/
    frontend, ir                 (17)  词法/语法/语义 / SSA + CFG
    opt, regalloc, emit          (18)  优化 / 线性扫描 / 指令选择与发射
    llvm                         (19)  SSA ↔ LLVM IR 对照后端
  compute/                       (38~42) context / module / function / stream / event
  os/                            (10)  主机内存 / 线程 / 时间源隔离

apps/
  scene_e0_selftest .. scene_e5_end_to_end          (1~9)
  scene_s0_selftest .. scene_s19_compute_memory     (stage2 10~37; stage3 38~42 for driver entry)

tools/
  isa_asm, isa_disasm            (5,16)
  ref_model                      (7)   CPU 参考模型 / 探针期望生成
  image_diff                     (7)   探针比对
  pick_probe                     (7)   离线探针拾取（headless 等价窗口点击）
  scene_snapshot                 (7)   场景 JSON 快照生成 / 读取
  spec_runner                    (7,36) 读 scene-specs 跑回归矩阵，HTML 汇总
  log_analyze                    (36)  批量日志分析（Error/Warn 统计、去重）
  mem_audit                      (36)  内存快照解析 / 泄漏与 UAF 检测
  state_inspector                (36)  状态与统计查看
```

### 2.5 依赖规则

1. 严格单向：`common → hw_sim → firmware → kmd → hli → umd → apps`；任何反向 include 视为架构违规。
2. `hw_sim` 内部方向固定：`frontend → shader`、`raster → shader`、`raster → om`、`geometry → shader`；反向通过回调或返回值。
3. 驱动层不得直接调用硬件单元：HLI/KMD/FW 访问硬件的唯一路径是"写命令到环 → 敲门铃 → CP 取指"。
4. `hw_sim` 不得出现 UMD/KMD 概念；`umd` 不得出现硬件内部结构；`hli` 不得出现 API 枚举与错误码。
5. 编译器与 `hw_sim` 的唯一连接是 `common` 的 ISA 定义与 Shader ABI，不共享内部数据结构。
6. `apps` 的依赖随阶段抬升：初级只认薄门面，直连阶段只认 HLI，UMD 阶段才认 `umd/graphics` 或 `umd/compute`；同一场景源码不改。
7. 禁止两个模块互相持有对方具体类型；跨层只用接口与 `common` 类型。

### 2.6 一次 draw 的旅程（高级直连 HLI）

| 阶段 | 位置 | 动作 | 可见证据 |
|---|---|---|---|
| P0 应用 | `apps` | 构造顶点/索引/纹理/常量，调用 HLI 命令列表 | 场景输入快照 |
| P1 绑定 | HLI 20 | 创建资源与视图，写描述符堆，设置根参数 | 描述符堆内容 |
| P2 管线 | HLI 20/22~26 | 选择 PSO（着色入口 + 固定功能状态） | `SHADER_PSO` 载荷 |
| P3 记录 | HLI 命令 | 绑定 RT/深度/描述符/常量，发 `DRAW` | 命令包序列 |
| P4 提交 | KMD/FW 10 | 写入环，更新 tail，敲门铃，等 fence | ring 快照 + fence 值 |
| P5 前端 | IA 22 → VS 23 → 裁剪/PA 24 | 装配顶点、着色、切片、裁剪、组装图元 | 顶点流缓冲、图元列表 |
| P6 光栅 | 25 → 27 | 三角形转像素，覆盖/early-Z，PS 着色，插值 | 像素候选列表、覆盖掩码 |
| P7 合并输出 | 26 → 13 | 深度/模板/混合/MRT，翻转扫描输出 | RT 字节、显示帧 |

任何一步失败都必须能定位到"包类型 + dword 偏移 + 单元 + 资源 VA"。

---

## 3. 层间接口契约

> 本文件只持有**层间接口**（C++ 接口与跨层数据契约）；子系统内部设计见 §4 对应的 `docs/design/` 文档。冻结接口的登记位置见 `roadmap.md` §7.3。

### 3.1 契约总表

| 上游 → 下游 | 接口 | 方向 | 冻结章 | 载体 |
|---|---|---|---|---|
| 场景 → 薄门面 | `hw::submit::Submit/Wait` | 向下 | 2 | C++ 值类型 |
| 场景 → HLI | `IHliDevice/Queue/CommandList` | 向下 | 35.1 | C++ 纯虚 |
| UMD → HLI | 同上（只允许使用公开对象） | 向下 | 35.1 | C++ 纯虚 |
| 计算驱动 → HLI | 同上 + `IHliPipeline`（compute） | 向下 | 35.1/stage3 38~42 | C++ 纯虚 |
| HLI → KMD | `IKmDdi`（设备、内存、上下文、提交） | 向下 | 10 | C++ 纯虚 |
| KMD → FW | `IFirmware`（ring、doorbell、上下文、复位） | 向下 | 10 | C++ 纯虚 |
| FW → HW | MMIO 写 + ring 内存 | 向下 | 10 | 寄存器/内存 |
| CP → 引擎 | `IEngine`（握手式，§3.3） | 向下 | 10 | 命令包 |
| 引擎 → 内存 | `IMemorySubsystem::Read/Write/Translate` | 双向 | 3/11 | C++ 纯虚 |
| 引擎 → 着色核 | `IShaderArray::Dispatch(...)` | 向下 | 4 | C++ 纯虚 |
| IA → VS | `VertexInputLayout`（§3.7） | 数据契约 | 22 | 结构体 |
| 计算驱动 → kernel 参数 | `KernelArgsLayout`（§3.8） | 数据契约 | 20、38~42 | 结构体 |
| 计算驱动 → HLI | `IHliPipeline::SetKernel`、`IHliCommandList::Dispatch/IndirectDispatch` | 向下 | 35.1/stage3 38~42 | C++ 纯虚（§3.4 计算侧行） |
| 场景 / 驱动 → 计算包 | `DispatchDesc` / `DispatchArgsIndirect`（§3.11） | 数据契约 | 20、35.1 | 结构体 |
| 计算 dispatch → 线程身份 | workgroup / wave / lane 映射（§3.12） | 数据契约 | 31 | 公式 + 契约 |
| 计算访存 → 内存子系统 | 地址空间与 `BARRIER` 作用域（§3.13） | 数据契约 | 30、31 | 枚举 + 包载荷 |
| 全层 → 日志 | `GD_LOG(level, module, fmt, ...)` | 旁路 | 1 | 宏 |
| 校验 → 场景 | 探针表 / 基线 | 旁路 | 7 | 文本/二进制 |

### 3.2 初级薄提交门面（第 2 章，ADR-028）

```cpp
namespace hw::submit {

struct DrawDesc {
    const void* vertexData;   // CPU 侧顶点数组
    u32         vertexCount;
    u32         stride;
    const void* indexData;    // 可为空
    u32         indexCount;
    const void* textureData;  // RGBA8，可为空
    u32         texWidth;
    u32         texHeight;
    u32         rtWidth;
    u32         rtHeight;
    u32         clearColor;   // 0xAARRGGBB，内存写 BGRA
};

Result Submit(const DrawDesc& desc);  // 同步执行完命令
Result Wait();                        // fence 值断言
}
```

约束：

- 场景只 include 该头文件；`hw_sim` 其余头文件对场景不可见。
- 门面内部允许调用 CP 与引擎，但不得把内部类型暴露到签名。
- 35.1 用 HLI 替换门面时，`apps/scene_*` 的 `main` 与场景数据保持不变。

### 3.3 IEngine 统一签名（第 10 章冻结；接口按功能级握手定义，ADR-035）

```cpp
namespace hw {
class IEngine {
public:
    virtual ~IEngine() = default;
    virtual EngineType Type() const = 0;
    // 功能级握手（类似 AXI-Stream valid/ready）：
    //   Push 请求入队（pkt 指向包载荷，不含头；sa 仅着色引擎非空）；
    //        输入 FIFO 满 → 返回 Busy，触发上游背压停发；
    //   Pump 推进流水线一级（功能级"周期"，不做真实周期精确）；
    //   Pop  结果出队，投递给下游输入 FIFO。
    // 初级内部可以仍是同步实现，但接口一律按握手定义，物理接口不与实现绑定。
    virtual Result Push(const u32* pkt, u64 dwordCount,
                        ContextId ctx, IMemorySubsystem* mem,
                        shader::IShaderArray* sa) = 0;
    virtual Result Pump() = 0;
    virtual Result Pop() = 0;
    virtual bool   IsIdle() const = 0;   // 全部输入 / 输出 FIFO 为空
    virtual CycleStats Stats() const = 0;// FIFO 占用率、反压周期计数等软观测
    // 同步便捷封装 = Push → Pump×N → Pop 至排空；初级场景与策略 A 使用
    virtual Result Execute(const u32* pkt, u64 dwordCount,
                           ContextId ctx, IMemorySubsystem* mem,
                           shader::IShaderArray* sa) = 0;
};
}
```

`FENCE` / `SET_STATE_BASE` / `CLEAR` / `NOP` 由 CP 内部处理，不经过引擎；`COPY_MEM` 走 Copy 引擎且 `sa = nullptr`；`DRAW` / `DISPATCH` 必填。非着色引擎必须容忍 `sa == nullptr`。

握手化的目的不是追求周期精确，而是给"GPU 为什么做深流水线""为什么有气泡""tile 大小要匹配 FIFO 深度"提供可观测的载体。初级（1~9）保持同步调用即可；第 10 章起 `IEngine` 及各级 FIFO 按握手实现，`Pump` 推进时更新 FIFO 占用率与反压计数（ADR-035）。执行与并行机制详见 `docs/design/execution-parallel.md`。

**错误流契约（第 10 章冻结，ADR-035 附属）**——`Push` / `Pump` / `Pop` 的返回码必须反映错误而非吞掉错误：

- **错误即中止、不重试**：引擎在 `Pump` 推进中发现硬错误（解码 `InvalidPacket`、执行中 `NotMapped`/`NotResident`、`InvalidIsa` 等）时，**立即返回该错误码**，抛弃本引擎当前提交的剩余未完成包，进入"已中止"状态；上层收到错误码后停止本提交（`Execute` 同步路径在首个错误返回点中止），不做重试与自动恢复。
- **错误后状态可观测**：引擎进入"已中止"状态后，`IsIdle()` 必须返回 `true`（输入/输出 FIFO 已排空），后续 `Push` 返回 `Ok` 但**不执行**（空转），直到该引擎经第 36 章引擎级复位；不得在错误后返回虚假成功。
- **单次错误码语义**：`Pump` 一次只上报一个错误（首个故障点），同一中止周期内的后续 `Pump` 保持 `Ok`+空转，错误现场以日志 `{unit, packetType, dwordOffset, va, code}` 记录（§3.9），不重复上报。
- **背压优先**：仅 FIFO 满返回 `Busy`（触发背压），`Busy` 不是错误、不改变引擎状态；硬件类错误一律走返回码 + 中止，两者不得混用。
- **多引擎隔离**：错误仅中止故障引擎，其余引擎照常推进；`Copy` 引擎把结果投递给下游或返错都由其自身握手返回，不吞错、不以引擎间转发掩盖。

### 3.4 HLI 对象面（35.1）

| 对象 | 关键方法 | 语义 |
|---|---|---|
| `IHliDevice` | `QueryCapability`、`CreateQueue`、`CreateResource`、`CreateView`、`CreateDescriptorHeap`、`CreatePipeline`、`CreateCommandList` | 设备与对象工厂；能力查询是唯一能力入口 |
| `IHliQueue` | `Submit(cmdList, fenceValue)`、`Wait(fenceValue)`、`Signal/SignalSemaphore` | 提交单位；队列家族 Graphics/Compute/Copy |
| `IHliResource` | `Map/Unmap`（仅 host-visible）、`GetVa`、`GetSize` | 只描述存储，不描述用途 |
| `IHliView` | `CreateSrv/Uav/Cbv/Sampler/Rtv/Dsv` | 用途与切片；不改变资源 |
| `IHliDescriptorHeap` | `Write(index, view)`、`GetGpuVa(index)` | 动态索引；图形与计算共用 |
| `IHliPipeline` | `SetStages`、`SetFixedState`、`SetKernel` | 图形用 stages+fixed（含 LoadOp/StoreOp、插值、混合、多视口），计算用 kernel |
| `IHliCommandList` | `Bind*`、`Draw`、`Dispatch`、`Barrier`、`Copy`、`Reset` | 唯一允许生成命令包的出口 |
| `IHliQueryPool` | `Begin/End/Resolve` | 34 章 |

硬规则：

1. HLI 不出现 `Vk*`、`D3D12_*`、`GL_*` 枚举或错误码。
2. HLI 输入输出只使用 `common` 类型与自身句柄。
3. UMD（含 PAL）不允许直接拼包或分配包号；所有命令必须经 `IHliCommandList`。
4. HLI 的 `CreatePipeline` 对图形与计算同构，不按上层类型分支。
5. 描述符堆按设备内存语义读取：提交后修改堆，下一次 draw 生效；命令包不拷贝描述符。
6. 动态状态在命令列表记录时快照，提交时按记录值执行；两个命令缓冲各自提交时动态状态互不干扰。
7. 35.1 DoD 全部以"场景直连 HLI"跑通后，才允许开始 35.2。

### 3.5 KMD / FW 契约（第 10 章）

```cpp
namespace kmd {
class IKmDdi {
public:
    virtual Result CreateDevice(const DeviceDesc&, IDevice** out) = 0;
    virtual Result AllocGpuMem(const AllocDesc&, GpuVAddr* outVa, void** outHost) = 0;
    virtual Result FreeGpuMem(GpuVAddr va) = 0;
    virtual Result CreateContext(u32 asid, IContext** out) = 0;
    virtual Result Submit(u32 queueId, const SubmitDesc&) = 0;  // 写环 + doorbell
    virtual Result WaitFence(u32 queueId, u64 value) = 0;
    virtual Result ResetEngine(EngineType) = 0;
};
}

namespace firmware {
class IFirmware {
public:
    virtual Result Boot(const BootDesc&) = 0;
    virtual Result ProgramRing(u32 ringId, GpuVAddr base, u32 sizeDwords) = 0;
    virtual Result OnDoorbell(u32 ringId) = 0;   // 加载页表基址，启动 CP
    virtual Result LoadContext(ContextId, GpuVAddr pageTableBase) = 0;
    virtual Result Halt() = 0;
};
}
```

规则：

- `Submit` 只负责写入 ring、更新 tail、写 doorbell；真正的取指由 CP 完成，KMD 不得直接调用引擎。
- doorbell 处理中，FW 只做"加载页表基址 + 启动 CP + 记录上下文"，不解析 draw。
- 中断只做通知；KMD 从 MMIO 读回完成值，不依赖中断载荷。
- 第 10 章冻结 DDI 形状；后续章只追加方法，不改变既有签名。

### 3.6 内存接口（第 3 / 11 章）

```cpp
class IMemorySubsystem {
public:
    virtual Result Read(GpuVAddr va, void* dst, u64 bytes) = 0;
    virtual Result Write(GpuVAddr va, const void* src, u64 bytes) = 0;
    virtual Result Translate(GpuVAddr va, PhysAddr* outPa) = 0;
    virtual Result Map(GpuVAddr va, PhysAddr pa, u64 bytes, MapFlags) = 0;
    virtual Result Unmap(GpuVAddr va, u64 bytes) = 0;
    virtual void   InvalidateTlb() = 0;
    virtual bool   IsResident(GpuVAddr va) const = 0;
};
```

规则：

- 所有单元（含 Copy、Display）统一走 `VA → TLB → L2 → 存储`，不得绕过翻译直连堆（`docs/design/memory-system.md`）。
- TLB 为**每 CU** 64 组 × 4 路 = **256 项**，组索引 `(va>>12)&0x3F`（容量口径权威 `docs/design/memory-system.md` §2）；失效必须全局可见；策略 B 下线程本地 TLB 依赖全局失效序列（`docs/design/execution-parallel.md` §1）；L1 写穿入 L2，仅脏行替换才进写缓冲。
- **有效 VA 位宽 = 36 bit（契约，B-07）**：两级页表各 4096 项 × 4 KiB = 64 GiB/AS，与 `common` 页表常量及 4 GiB/8 GiB 堆基址一致；`GpuVAddr`/`PhysAddr` 软件类型仍为 u64，"硬件有效位宽 ≤ 软件类型位宽"。见 `memory-system.md` §2 与 `bit-budget.md` §4。
- 未映射返回 `NotMapped` 并带 VA；未驻留触发页错误路径，不得静默读 0。
- 页错误处理为"记录 + 中止当前提交"，**不复位**；复位统一由第 36 章按引擎级接入（ADR-011、Q-30、Q-69），不做 wave 暂停/恢复。
- `Flush` / `Invalidate` 归第 12 章 `CacheSubsystem`，**不加入本接口**；本接口 7 个方法冻结（Q-44）。
- **部分成功语义（第 11 章冻结）**：`Read`/`Write` 按**调用级原子性**处理——请求范围 `[va, va+bytes)` 内任一 4 KiB 页翻译失败（未映射 / 未驻留）时，**整次调用不产生任何写副作用并返回 `NotMapped`**（附首个故障 VA 及其页偏移），禁止"先成功写前几页、遇故障页才报错"的部分写入；翻译按页完成、全部成功后才一次提交。`Translate` 一次只翻译单个 VA（返回该 VA 所在页的 `PhysAddr`），不承诺相邻 VA 的翻译。`Map`/`Unmap` 要求 `va` 与该范围均为 4 KiB 对齐、范围合法；部分覆盖同一页的重叠 `Map` 为未定义行为（上层责任，调用前应保证 PTE 区域互斥）。
- 分配器：VRAM 空闲链表 + GART bump；GART 有容量上限，周期性缓冲按上下文常驻复用，不每帧分配；描述符堆登记 GART（GPU 可读，Q-51）。

### 3.7 层间数据契约：顶点布局（第 22 章冻结）

```cpp
struct VertexInputLayout {
    struct Attrib {
        GpuVAddr bufferBase;
        u32      bufferStride;
        u32      offsetInVertex;
        u32      format;
        u32      stepRate;        // 0 = per-vertex，非 0 = per-instance
    };
    std::vector<Attrib> attribs;
    u32 indexFormat;
    u32 baseVertex;
    u32 baseInstance;
};
// 取第 n 个顶点的属性 i：addr = bufferBase + (n / max(1, stepRate)) * stride + offsetInVertex
```

IA 只消费该布局，不感知 API 顶点格式；Shader ABI 输入属性区 `v[0..15]` 由 IA 按本布局装配（`docs/design/isa-abi.md` §3）。初级固定 `RenderVertex`（stride 32：`float4 position` + `float4 color`），第 22 章切换为 `VertexInputLayout` 动态装配。

### 3.8 计算 kernel 参数（第 20 章冻结；计算驱动落地 stage3 38~42，ABI 见 `compute-driver.md`）

```cpp
struct KernelArgsLayout {
    GpuVAddr paramBufVa;          // 参数区 VA（host 预写或 H2D 拷贝后）
    u32      paramBufByteSize;
    u32      paramVgprBase;       // 默认 56（= uniform 区起点 v[56]，isa-abi.md §3 权威分区）
    std::vector<u32> paramOffsets; // 每个参数在参数区内的字节偏移
};
```

计算流程：应用预写参数区 → 计算驱动 `LaunchKernel` → HLI 组 `DISPATCH` 包 → Compute 引擎把 `paramBufVa`、grid/block 写入 VGPR uniform 区 → kernel 按偏移读取。对象模型与打包语义见 `docs/design/compute-line.md` §2。

**输入校验规则（第 20 章冻结，计算驱动 / HLI 创建侧执行）**——违反任何一条返回 `InvalidArgument`，**不下发硬件**；硬件侧对越界访问只做确定性中止（§3.3 错误流），不做语义修复：

1. `paramBufVa` 必须已映射且驻留（`HOST_VISIBLE | HOST_COHERENT` 参数缓冲）；未映射 / 未驻留返回 `NotMapped`/`NotResident`。
2. `paramBufByteSize > 0` 且不超过参数区容量上限（独立契约常量 `kMaxParamBufBytes`，**2026-11-20 定案 `4 KiB`**，登记 `docs/design/bit-budget.md` §7.1 **B-041**；**与 VGPR uniform 分区容量解耦**——`v[56..67]` 只承载 `paramBufVa/grid/block` 等描述符，不承载参数本体，见 `docs/design/isa-abi.md` §3）。
3. `paramOffsets` 必须**严格递增且互不重叠**；每条参数满足 `offset + size ≤ paramBufByteSize`（size 由驱动按签名推导，KernelArgsLayout 只携带偏移）。
4. 参数偏移按类型自然对齐：标量 4 B、`float2/vec2/int2` 8 B、`float4/vec4` 16 B；违反对齐返回 `InvalidArgument`。
5. `paramVgprBase` 默认 56（`v[56]`，uniform 区起点），合法值必须在 uniform 分区 `v[56..67]` 内且不与其他 ABI 分区冲突（`isa-abi.md` §3 权威分区）；越界（如落在 RT1~3 `v[44..55]` 或临时量区 `v[76..]`）一律 `InvalidArgument`。
6. 校验只做**静态结构校验**（映射 / 对齐 / 边界 / 区段），不断言参数语义（值域由 kernel 自证）；`DISPATCH` 包生成前完成，失败即 `InvalidArgument` 且不产生包。

### 3.9 错误契约

```cpp
enum class Result {
    Ok = 0,
    InvalidArgument,
    NotSupported,
    NotMapped,
    NotResident,
    InvalidPacket,
    InvalidIsa,
    OutOfMemory,
    DeviceLost,
    ResetRequired,
    Internal,
};
```

- 所有跨层接口返回 `Result`；禁止用异常跨层传播。
- 错误日志必须包含 `{unit, packetType, dwordOffset, va, code}`；能定位才允许返回。
- 错误只做"记录 + 中止当前提交"，不做重试与自动恢复；第 36 章只保证"可定位 + reset 后可重跑"（`docs/design/boot-reset.md`）。
- `ResetRequired` 仅保留枚举值：**初级与高级一期均不产生**，复位只在第 36 章以**引擎级**接入（仅重编程该 ring，其余引擎数据完好）。**例外（2026-11-20 定案）**：第 10 章显式 `ResetEngine` 诊断入口与非法包错误路径可将 `ResetRequired` 置位，仅用于人工诊断与错误记录，不触发自动复位状态机、不进入阶段门禁；其余路径仍为"记录 + 中止提交"。
- `NotSupported` 用于能力二期/范围外；不得用成功返回值掩盖未实现能力。

**`Result` → 上层 API 错误码映射（35.2~35.4 与 stage3 38~42 冻结，单一来源）**——UMD（Vulkan/D3D12/GL）与计算驱动的 API 参数是上层入口，不得向应用泄露 `Result` 枚举本身；映射只做一次、方向单向（底层 `Result` → 上层错误码），禁止一方把上层错误反向当输入。HLI 自身（§3.4 硬规则 1）**不出现** `Vk*`/`D3D12_*`/`GL_*` 错误码，本表是唯一的 API 错误码登记位置。计算驱动列的名称为 MUSA 风格示意，权威的是**映射方向与禁止项**（底层 `Result` 单向映射为对等语义的非成功码），具体枚举名以计算驱动头文件为准：

| 底层 `Result` | 语义 | Vulkan | D3D12 | OpenGL | 计算驱动（MUSA 风格） |
|---|---|---|---|---|---|
| `InvalidArgument` | UMD/驱动侧参数、句柄或数据契约非法（§3.8） | `VK_ERROR_INITIALIZATION_FAILED`（创建期） | `E_INVALIDARG` | `GL_INVALID_VALUE` / `GL_INVALID_OPERATION` | `MUSA_ERROR_INVALID_VALUE` |
| `NotSupported` | 能力二期 / 范围外（capability bit 恒 false） | `VK_ERROR_FEATURE_NOT_PRESENT` | `E_NOTIMPL` | `GL_INVALID_OPERATION`（能力特性缺失） | `MUSA_ERROR_NOT_SUPPORTED` |
| `NotMapped` / `NotResident` | GPU 地址未映射 / 未驻留（§3.6） | `VK_ERROR_MEMORY_MAP_FAILED`（映射侧）；执行期走设备丢失类 | `E_FAIL` / `DXGI_ERROR_DEVICE_REMOVED`（执行期） | `GL_OUT_OF_MEMORY` | `MUSA_ERROR_INVALID_VALUE`（执行期走设备丢失类） |
| `InvalidPacket` / `InvalidIsa` | 命令包或 ISA 非法（§3.3 错误流） | `VK_ERROR_INVALID_SHADER_NV`（shader 侧） | `E_INVALIDARG` | `GL_INVALID_SHADER` | `MUSA_ERROR_INVALID_DEVICE_FUNCTION` |
| `OutOfMemory` | GPU 内存不足（11、14 章） | `VK_ERROR_OUT_OF_DEVICE_MEMORY` | `E_OUTOFMEMORY` | `GL_OUT_OF_MEMORY` | `MUSA_ERROR_OUT_OF_MEMORY` |
| `DeviceLost` / `ResetRequired` | 设备 / 引擎复位，需重跑（36 章） | `VK_ERROR_DEVICE_LOST` | `DXGI_ERROR_DEVICE_REMOVED` | `GL_CONTEXT_LOST` | `MUSA_ERROR_DEVICE_LOST` |
| `Internal` | 未归类内部错误 | `VK_ERROR_UNKNOWN` | `E_FAIL` | `GL_INVALID_OPERATION` | `MUSA_ERROR_UNKNOWN` |

映射硬规则：

1. **禁止吞错 / 降级为成功**：任何非 `Ok` 的底层 `Result` 都必须映射为上层非成功错误码（见 §3.9 规则 3）；`DeviceLost`/`ResetRequired` 类硬件错误必须映射为上层"设备丢失/移除/上下文丢失"类错误，禁止映射成参数类错误掩盖复位事实。
2. **创建期与执行期区分**：`InvalidArgument`/`NotSupported` 主要在创建期映射为参数/特性错误；执行期（提交后）同一类故障按 §3.3 已中止提交，UMD 应优先上报设备类错误而非重复报参数错误。
3. **映射在 UMD 层一次性完成**：Vulkan/D3D12/GL 与计算驱动各自维护本表映射，不跨后端共享错误码对象；HLI 返回值在进入不同 API 后端前必须已完成转换，禁止 API 对象携带另一后端的错误码。

### 3.10 回调链与生命周期

创建顺序：`HW 设备 → FW boot → KMD 设备/上下文 → HLI 设备/队列 → UMD 设备`；销毁严格逆序。

| 阶段 | 必须完成 | 失败处理 |
|---|---|---|
| HW 设备 | MMIO 复位、环清零、fence 归零 | 返回 `DeviceLost` |
| FW boot | 握手、ring 编程、CP 停止 | 中止创建，释放 HW |
| KMD | 页表根、ASID、提交队列 | 释放 FW/HW |
| HLI | 对象表、描述符堆默认状态 | 释放 KMD |
| UMD | API 对象、默认管线 | 释放 HLI |

`Shutdown` 时必须：停止 CP → 等待队列空闲 → 释放 UMD → 释放 HLI → 释放 KMD → FW halt → HW 关闭。任何一步跳过都视为资源泄漏。启动/复位/关闭的硬件侧细节见 `docs/design/boot-reset.md`。

**对象销毁契约（35.1，问题：销毁异常 / 复位时序）**：

1. **正常销毁 = 逆创建顺序**，任一对象的销毁不等待除自身以外的提交：`IHliResource/View/Pipeline/DescriptorHeap` 可随时销毁，但底层 GPU 内存仅在**用户引用 AND 硬件提交引用双归零**后释放（见 §3.10、ADR-041）；HLI 保证"销毁后同一句柄再使用返回 `InvalidArgument`"，不产生悬空访问。
2. **销毁不做隐式同步**：应用要求对象仍被 in-flight 命令引用时，销毁只减用户引用、不动硬件提交引用（硬件引用由 fence 完成回调释放）；是否阻塞由上层（UMD）决定，HLI 只维护引用计数与释放点。
3. **异常路径（设备丢失 / 引擎复位期间）**：复位发生时（下节"引擎复位后的对象 / 命令 / 引用计数语义" + `boot-reset.md` §2）被丢弃命令的硬件提交引用**即刻释放**，不再等待永远不会完成的 fence；用户引用仍归应用持有（除非上层因 `DeviceLost` 选择销毁）。**销毁在复位期间必须安全且幂等**——底层内存已由复位路径回收的，`Destroy` 不得二次 free、不得崩溃。

**引擎复位后的对象 / 命令 / 引用计数语义（第 36 章，问题 13）**——引擎级复位（仅重编程故障引擎的 ring），逐条明确：

| 维度 | 复位后语义 |
|---|---|
| 未完成命令 | 故障引擎已提交未完成命令全部丢弃；被丢弃命令**不再执行**、其产物（写出的像素/内存）为未定义，场景按固定输入重跑才与首次一致（`boot-reset.md` §2） |
| fence | 被丢弃命令关联的 fence 不承诺到达目标值；本引擎 fence 监视清空并保留错误记录；其余引擎 fence 语义完好、继续推进 |
| 对象 / 引用计数 | 被丢弃命令持的硬件提交引用**即刻归零**（不等 fence）；用户引用不变，应用仍可持有句柄直至自己销毁；其余引擎引用的对象与数据完好 |
| 命令缓冲（HLI） | Reset 只作用于故障引擎的 ring 与 in-flight 命令，**不使任何 HLI 命令缓冲失效**；重跑用同一命令缓冲、同样输入，结果与首次一致 |
| 责任归属 | 复位是 36 章引擎级行为，不级联到其他引擎 / 设备，UMD 依 §3.9 把 `ResetRequired` 映射为设备丢失类错误上报 |

初级与高级一期不产生 `ResetRequired`（§3.9），故本节约束对初级薄门面不生效，仅第 36 章接入后启用。

### 3.11 计算 dispatch 载荷与计算 ABI（第 20、35.1 章冻结，ADR-046；计算驱动落地 stage3 38~42）

计算通路与图形通路共用 HLI、KMD、FW、内存、缓存、队列与 ISA；计算侧独有的**只有一处**：把 `grid / block / kernel 入口 / 参数区` 打包成一个 `DISPATCH` 包。`DISPATCH` 包类型 `0x04` 已在第 1 章预留（本文件不新增包号），本节冻结其载荷语义。

```cpp
namespace compute {

struct Dim3 { u32 x, y, z; };              // z 仅三维 dispatch 使用，缺省 1

struct DispatchDesc {
    GpuVAddr kernelVa;                     // module 符号解析后的 ISA 入口
    GpuVAddr paramBufVa;                   // 参数区（HOST_VISIBLE|HOST_COHERENT，§3.8）
    u32      paramBufByteSize;
    u32      paramVgprBase;                 // 默认 56，合法域见 §3.8 规则 5
    std::vector<u32> paramOffsets;
    Dim3     grid;                         // workgroup 数
    Dim3     block;                        // 每 workgroup 线程数
    u32      ldsStaticBytes;               // 静态 LDS 需求；动态 LDS 属范围外
    u32      sgprCount;                    // 资源声明，只进 occupancy 预算
    u32      vgprCount;
    u32      subgroupSize;                 // 固定 32
    GpuVAddr indirectArgsVa;               // 0 = 直接 dispatch
    u32      flags;                        // bit0 = 组内使用 barrier；bit1 = 需 device 作用域；其余必须 0
};

struct DispatchArgsIndirect {              // 6 dword，16 B 对齐
    Dim3 grid;
    Dim3 block;
    u32  reserved0;                        // 必须 0
};

}
```

**输入校验规则（第 20、35.1 章冻结；HLI 与计算驱动在创建 / 记录期执行）**——违反任一条返回 `InvalidArgument`，**不产生包、不下发硬件**；执行期只做确定性中止（§3.3），不做语义修复：

| 编号 | 规则 |
|---|---|
| D-01 | `block` 三维各 ≥ 1，`block.x*block.y*block.z ≤ 1024`；**允许非 32 倍数**，末波按 `LaneMask` 处理（§3.12 M-02），不得为凑整波而静默补齐线程 |
| D-02 | `grid` 三维各 ≥ 1，总 workgroup 数 ≤ `kMaxWorkgroups`（ADR-038 固定容量，**`2^20`**，登记 `bit-budget.md` §7.1 **B-038**；2026-11-20 定案，ADR-043/045） |
| D-03 | `ldsStaticBytes ≤` CU LDS 容量（永久 **64 KiB/CU**，B-036）且 16 B 对齐；超限直接拒绝，不做"缩 LDS 硬跑" |
| D-04 | `sgprCount ≤ s0..s31` 可用量、`vgprCount ≤ v0..v127` 扣除 ABI 保留区（`isa-abi.md` §3 `kVgprBase*` 分区）；声明值只影响调度预算，不改变 ISA 语义 |
| D-05 | `subgroupSize` 必须 32（ADR-018）；可变 subgroup 属范围外（ADR-025），拒绝而非按 32 静默执行 |
| D-06 | `kernelVa` 必须落在已装载 module 的代码段内且 4 B 对齐；未映射 `NotMapped`，越段 `InvalidIsa` |
| D-07 | 参数区校验**完全复用 §3.8 规则 1~6**，本节不重复定义，禁止两处各写一套口径 |
| D-08 | `DISPATCH` 包**不得携带** IA / 光栅 / OM / RT / DS 任一固定功能状态；携带即 `InvalidPacket`（ADR-032：共用单元按包类型派发，不得为计算开特化字段） |
| D-09 | `flags` 未知位必须为 0；置未知位返回 `InvalidArgument`，不得静默忽略（ADR-039：不做兼容修补） |
| D-10 | 间接 dispatch：`indirectArgsVa` 必须已映射、16 B 对齐；参数在**展开 workgroup 前读一次**，作为本次 dispatch 唯一真值，执行中不重读；取值非法（grid/block 为 0 或越限）由 GPU 侧确定性中止，不修复 |
| D-11 | 一个 workgroup 的波组不得跨 CU、不得跨 SE（§3.12 M-03）；调度器不得拆分组 |
| D-12 | 校验只做静态结构校验（映射 / 对齐 / 边界 / 区段）；参数**取值语义**由 kernel 自证，校验器不断言（与 §3.8 规则 6 同口径） |
| D-13 | **静态可调度性**（HLI/计算驱动记录期判定，ADR-038）：`wavesPerGroup ≤` CU 最大驻留波数 且 `ldsStaticBytes ≤` CU LDS 容量（上限见 `bit-budget.md` §7.1）；违反直接 `InvalidArgument`，**不进入排队**。M-06 的"排队"仅适用"组级需求本身可满足、只是暂被占用"，防止 D-01 × M-03 × M-06 组合出永久挂起 |

字段位段（`DISPATCH` = `0x04`，新增字段走载荷位段 / `SubOp`）必须先审计再登记，禁止新增顶层包号；本条与 §7.4 红线 11 同源。

### 3.12 workgroup / wave / CU 映射与计算确定性（第 4、15、29、30、31 章冻结，ADR-046）

图形线把 32 个 lane 合成一个 quad 处理像素；计算线的并行单位是 **workgroup（调度单位）→ wave（执行单位）→ lane（线程）**。以下公式是唯一映射，任何单元不得另立一套：

```
threadsPerGroup = block.x * block.y * block.z
wavesPerGroup   = ceil(threadsPerGroup / 32)
gtid            = tid.x + tid.y*block.x + tid.z*block.x*block.y      // z 最外层
waveIdInGroup   = gtid / 32                 lane = gtid % 32
workgroupId     = gx + gy*grid.x + gz*grid.x*grid.y
globalThreadId  = workgroupId * threadsPerGroup + gtid
```

**契约规则**

- **M-01 身份唯一**：`tid / gtid / waveId / workgroupId` 只由 `grid` 与 `block` 决定，与线程完成顺序、CU 数量、SE 数量无关；`serial` 与 `parallel` 下同一线程的 ID 必须逐值相同。
- **M-02 末波部分有效**：`LaneMask` 为有效 lane 集合（组线程数非 32 倍数时末波部分有效）；**无效 lane 不执行访存 / 原子 / barrier**。此点与第 25 章"2×2 quad 内任一 lane 有效则 4 lane 全执行"**不同源**：quad 规则只服务导数与像素着色一致性，**禁止移植到计算通路**（否则 warps 执行数判据失真）。
- **M-03 组驻留整体性（防死锁的唯一手段）**：`wavesPerGroup` 个波必须同时驻留同一 CU；预算 `wavesPerGroup × (VGPR + SGPR 占用) + ldsStaticBytes` 超限时**整个 workgroup 排队**，不得部分驻留、不得跨 CU 拼装。因此组内 barrier 永远可达，不存在"等不到对端波"的死锁。
- **M-04 barrier 一致到达**：组内全部活跃线程必须在一致控制流到达同一 `BARRIER` PC；非一致控制流由 ISA 层检测并返回 `InvalidIsa` 中止（给可定位错误，不写成未定义行为）。
- **M-05 推进顺序**：组内波按 `waveIdInGroup` 升序推进；跨组顺序策略 A 按 `workgroupId` 升序、策略 B 由 CU 原子领取，两者均不得改变硬判据（§5.4）。
- **M-06 occupancy 上限固定**（ADR-038）：CU 驻留波数、每 CU 的 VGPR/SGPR/LDS 总量、输入/输出 FIFO 深度固定且可配置（2026-11-20 定案：驻留 **16 waves/CU**、LDS **64 KiB/CU**、FIFO **8 项**，登记 `bit-budget.md` §7.1 **B-036/B-037/B-039**）；超限**排队等释放**，不报错、不降级。排队**仅限**"组级需求本身可满足（`wavesPerGroup ≤` 驻留波数、`ldsStaticBytes ≤` CU LDS）、仅当前被其它组暂占"的情形；需求本身超上限在 HLI/驱动记录期被 §3.11 D-13 直接 `InvalidArgument` 拒绝，避免"永远排不上且无进展"的挂起。
- **M-07 组边界不隐式同步**：workgroup 结束不等于 device 可见；dispatch 间、kernel 间可见性一律走 §3.13 的显式 barrier / fence，禁止把"提交完成"当同步。
- **M-08 确定性**：计算通路无时间输入；需随机数的 kernel 用固定种子、按 `gtid` 升序消费（§7.1 规则 5）。

**Compute 引擎的职责边界**：它是 `IEngine` 实例中的**薄壳**——只做间接参数读取、kernel 入口装载、uniform 区写入（`paramBufVa` + `grid/block` + 组基址）、workgroup 展开与派发到 `se`；线程执行、分歧、subgroup、LDS、原子、barrier 全部由 `shader/*` 与 `se` 承担。引擎不解析 kernel 语义，不参与 IA / 光栅 / OM（§7.8），也不新增 `hw_sim` 目录。

### 3.13 计算地址空间与内存作用域（第 3、11、12、14、30、31 章冻结，ADR-046）

计算把"一片地址空间"拆成五类访问，每类只走一条路径；**共用内存子系统，不为计算开旁路**（§3.6 规则 1）：

| 地址空间 | 承载 | 访问路径 | 作用域 | 复用模块 | 一致性保证 |
|---|---|---|---|---|---|
| Global（UAV / SRV buffer） | VRAM / GART / System 三堆 | `VA → TLB → L2 → 堆` | Device | `memory/*`、`cache`(12) | L1 写穿 L2；跨 CU 可见需 device barrier |
| Constant / Param | HOST_VISIBLE 参数缓冲、CBV | `VA → TLB → L2`（只读） | 单次 dispatch | `memory`、`binding`(20) | 提交前写入、dispatch 内只读，不跨 dispatch 缓存 |
| Local / Shared（LDS） | CU 片上 LDS | **不经 VA/PA**（LDS 索引 + bank） | Workgroup | `shader/shared`(30) | 组内 barrier；组退出即失效 |
| Private / Spill | GART scratch（per-thread） | `VA → TLB → L2` | Thread | `memory/heaps`(11) | 线程私有，无跨线程别名 |
| Image（read / write） | tiled 资源 | `VA → TLB → L2 → tiling decode` | Device | `texture`(21) + `common/format` | 只读靠 barrier 后失效 L1；写回复用 21 章编码 |

**一致性矩阵**（写者 → 读者，唯一来源；"不满足"列即定位线索）

| 写 → 读 | 需要的机制 | 不满足时的现象 |
|---|---|---|
| 同线程 private | 程序序 | — |
| 同组 LDS | `BARRIER(scope=Workgroup)` | 读到旧值 / 竞态 |
| 同组 global（同 CU） | `BARRIER(scope=Workgroup)`（排空该 CU 写队列） | 读到 L1 旧值 |
| 跨组 / 跨 CU global | `BARRIER(scope=Device)`（排空全部写队列 + L1 invalidate） | 部分 CU 读到旧值 |
| 跨 dispatch / 跨队列 | fence / timeline + `BARRIER(scope=Device)` | 顺序不可解释 |
| 图形 ↔ 计算 | 队列家族所有权转移（32）+ fence + device barrier | RT / UAV 互不可见 |

**规则**

- **S-01 LDS 永不进 VA/PA**：LDS 只经 `shader/shared` 的索引与 bank 访问；任何以 VA 触达 LDS 的指令判 `InvalidIsa`（与 ADR-016 的"片上存储不走 VA"同源）。
- **S-02 无隐式全局刷新**：dispatch 边界不自动 flush / invalidate；可见性只由 `BARRIER` + fence 建立，保证每条读写都能定位到机制（§1.1）。
- **S-03 作用域在载荷内表达**：`0x0E` 载荷携带 `scope ∈ {Workgroup, Device}`、`access ∈ {READ, WRITE, RW}`（粗粒度集合，不做逐访问位掩码，ADR-031）；**禁止为此新增顶层包号**（ADR-043）。
- **S-04 计算写回**：buffer store 由 memory 子系统直写（不经 OM）；image store 复用第 21 章 tiling 编码与 `common/format` 转换；RT 写回与计算写回**共用同一格式转换与写掩码实现**，不得复制两份（ADR-032 的实现级推论）。
- **S-05 缓存口径**：L1 写穿 L2、L2 为 coherence point（与 §3.6、第 12 章一致）；`cache_enable=off` 只影响统计，不改变结果（§5.1）。
- **S-06 故障同路**：未映射 / 未驻留与图形一致（`NotMapped` / `NotResident` + 中止当前提交），禁止静默读 0，禁止为计算单独开"宽松读"路径。

计算 scratch（per-thread 栈 / 寄存器溢出）放 GART，per-thread base 由驱动经参数区传入；分配走第 11 章确定性分配器，同 `translation.seed` 下布局逐字节可复现。

---

## 4. 模块设计文档（`docs/design/`）

> 子系统因工程量增长拆分为独立设计文档；本文件只保留分层、模块树与层间接口。项目范围与阶段由任务规划确定；本文件是模块树和层间契约的权威来源；各 `docs/design/<模块>.md` 负责模块内部机制；各章文档负责实现落地。
> 子模块文档自含：模块内部结构与状态、落地章、验收要点与契约变更规则；同时承载原架构 §4 全部内容与 roadmap 中相关权威条目。

| 模块 | 文档（`docs/design/`） | 落地章 | 职责概览 | 相关层间接口 |
|---|---|---|---|---|
| ISA 与 Shader ABI | `isa-abi.md` | 5、16 | 64 bit 定长编码、opcode `[62:56]` + `[63]=SFLAG`、VGPR/SGPR 固定分区（`kVgprBase*`） | `common/isa.h`、`SymbolBindings` |
| 内存系统 | `memory-system.md` | 3、11、12、14 | 三类地址、页表/TLB、三堆与分配器、缓存/驻留、主机同步与确定性 | `IMemorySubsystem` |
| 同步原语 | `sync-primitives.md` | 2、32、33 | fence / timeline semaphore / event / barrier / 内存序作用域；`PREDICATE`=0x0D、`BARRIER`=0x0E、0x0F 预留 | `SIGNAL`/`WAIT`/`BARRIER`/`PREDICATE` 包 |
| 编译器管线 | `compiler.md` | 17~19 | 前端(SSA+分歧)、后端(优化/寄存器分配/发射)、LLVM 对照 | `common` ISA/ABI |
| 命令包、stateId 与 MMIO | `command-mmio.md` | 1、2、10 | 包头与版本（高级 `version=2` 且拒绝 v1）、包类型登记、stateId（dword 索引，`0x40+2n`）、MMIO 分段 | `common/packet.h`、MMIO |
| 轻量日志、硬件统计与断言 | `logging.md` | 1、高级 A | `GD_LOG`、MMIO 统计计数器、三套断言 | `GD_LOG` 宏 |
| 校验模型 | `validation-model.md` | 7 起 | L1/L2/横切分层、探针与基线（初级不引入探针）、单份场景文档、CI 诊断 | `tools/ref_model` 等 |
| 执行与并行模型 | `execution-parallel.md` | 4、15、31 | 双策略、波调度、SE/CU 分区、occupancy、时间无关确定性 | `IEngine`（握手） |
| 计算线 | `compute-line.md` | 20、29~31、33、37（通路收口）；stage3 38~42（计算驱动，见下行） | 计算通路、workgroup/wave 映射、内存作用域 | `DISPATCH` 包、`KernelArgsLayout` |
| 计算驱动对象模型与 ABI | `compute-driver.md` | 38~42 | MUSA_MODULE_HDR、符号表/Function 入口、参数打包 ABI（六条校验）、Stream/Event、生命周期、错误码映射 | umd/compute、§3.8/§3.9/§3.11 |
| 纹理单元 | `texture-unit.md` | 7、21 | 格式解码、tiling、mip/LOD、过滤、采样器、比较采样 | `SAMPLE`/`SAMPLE_CMP` |
| 启动、复位与关闭 | `boot-reset.md` | 10、36 | 启动序列、复位路径、watchdog、故障分层、逻辑特权 | `Result`、`ResetRequired` |

**计算线文档归属**（同属 `compute-line.md`，子节分工固定；模块内部细节不进本文件，本文件只持层间契约）：

| `compute-line.md` 子节 | 内容 | 本文件对应契约 |
|---|---|---|
| §1 计算最小通路 | kernel → module/function → 参数区 → dispatch → workgroup/CU → wave → LDS/atomic/barrier → 写回 → fence | §2.2、§3.3 |
| §2 对象模型 | MUSA-like context/module/function/stream/event 与 HLI 映射 | §3.4、§3.11 |
| §3 地址空间与作用域 | 五类访问的模块化实现、L1/L2 口径、scratch 分配 | §3.13 |
| §4 dispatch 打包与间接参数 | `DispatchDesc` 落地、间接参数读取时机、校验实现 | §3.11 |
| §5 workgroup/wave/CU 映射与 occupancy 预算 | 映射公式实现、组驻留账本、FIFO 与背压 | §3.12 |
| §6 计算业务目录 WA | 12 类 workload 的 kernel 规格与硬件行为预期 | §7.9 |
| §7 驱动 API 面与 HLI 映射 | 计算驱动函数面、错误码映射、生命周期 | §3.4、§3.9 |
| §8 验收与差分口径 | 数值容差、结构断言、双入口等价 | `roadmap.md` §6.7 |

项目级范围和阶段见任务规划；模块设计文档的登记位：包类型 / `stateId` / MMIO 段 / opcode 区段的权威登记表见 `roadmap.md` §7.3，增量在 `command-mmio.md`、`isa-abi.md` 对应节登记。位宽与地址空间余量（包类型、MMIO、stateId、GPU VA、ISA opcode、环/堆容量）的集中审计见 `docs/design/bit-budget.md`（**唯一审计基准**：任一涉及包 / MMIO / ISA / 页表分配的章节开工前强制对照，见 `bit-budget.md` §1 与 `roadmap.md` §7.3/§7.4）。

---

## 5. 目录结构

```
gpu-driver-model/
├── CMakeLists.txt
├── common/
│   ├── include/common/{types,result,packet,isa,shader_abi,format,log,config}.h
│   └── src/
├── hw_sim/
│   ├── include/hw_sim/{submit,device,mmio,ring,cp,engines,memory,shader,isa,display,query}.h
│   └── src/
├── firmware/
├── kmd/
├── hli/
├── umd/
│   ├── graphics/{api/{vulkan,d3d12,gl},pal,shader,os}
│   └── compute/
├── apps/
│   ├── scene_e0_selftest/ … scene_e5_end_to_end/
│   ├── scene_s0_selftest/ … scene_s19_compute_memory/
│   └── demo_c1_module_probe/ … demo_c5_indirect_dispatch/（stage3 38~42，`C<n>` 登记见 `roadmap.md` §7.1）
├── tools/
│   ├── isa_asm/ isa_disasm/
│   ├── ref_model/
│   ├── image_diff/
│   ├── pick_probe/
│   ├── scene_snapshot/
│   ├── spec_runner/
│   ├── log_analyze/
│   ├── mem_audit/
│   └── state_inspector/
├── third_party/
│   └── stb_image_write.h
├── docs/
│   ├── roadmap.md
│   ├── architecture-design.md
│   ├── design/isa-abi.md … compute-driver.md（§4 的 12 个独立模块设计文档）
│   ├── adr/ADR-<三位序号>-<kebab-title>.md
│   ├── design/scene-specs.md（场景规约单份：每场景一节）
│   ├── stage1/01-contracts-min.md … 09-end-to-end.md
│   ├── stage2/10-device-ring-cp.md … 37-compute-pipeline.md
│   ├── stage3/38-compute-driver-skeleton.md … 42-end-to-end-acceptance.md
│   └── notes/
└── tests/
    ├── contract/        L1
    ├── unit/            单元
    ├── scenes/          E/S 场景驱动
    └── golden/          基线
```

规则：`hw_sim` 不 include 任何 `kmd/umd/apps` 头；`common` 不 include 任何上层头；章文档路径与文件名严格遵循 `roadmap.md` §7.3 与 §0.1（`stage1/NN-<slug>.md`、`stage2/NN-<slug>.md`）；独立模块设计文档统一放 `docs/design/` 下，文件名为 §4 索引所列 `NN-<slug>.md`（如 `memory-system.md`）。

---

## 6. 构建与跨平台

| 项 | 约定 |
|---|---|
| 语言 | C++20 |
| 构建 | CMake ≥ 3.20；目标：`common`、`hw_sim`、`firmware`、`kmd`、`hli`、`umd_graphics`、`umd_compute`、`tools`、`apps`、`tests` |
| 平台 | Windows 10+（MSVC）/ Linux（GCC 11+、Clang 14+） |
| 警告 | `/W4` 或 `-Wall -Wextra`；警告即错误由 CI 配置决定 |
| 异常 | 跨层禁用；层内允许但必须转 `Result` |
| RTTI | 关闭；接口用纯虚 + `Type()` 区分 |
| 线程 | `std::thread` + 显式同步；不使用平台线程 API |
| 时间 | 仅统计用 `steady_clock`；着色器与位姿禁止时间输入 |
| 窗口 | 上层调试工具，不属于硬件单元；构建开关 `BUILD_WITH_WINDOW`（默认 OFF），关闭时 headless 可完整跑全部校验；开窗时只做 CPU 帧 → 宿主 GPU 纹理 blit，放 `umd/os` 适配层，不参与任何渲染计算 |
| 配置 | 运行期键见 `roadmap.md` §5.1（权威全表）；构建期只保留 `GD_WARP_SIZE=32`、`GD_LOG_LEVEL`、`BUILD_WITH_WINDOW` 等 |
| 测试 | `ctest` 分组：`contract`、`unit`、`scene_e`、`scene_s`、`golden` |

跨平台红线：核心层不得出现 `<windows.h>`、`<unistd.h>`、路径分隔符假设；文件路径统一用 `/` 或 `std::filesystem`；字节序统一小端（如遇大端平台，显式转换）。

---

## 7. 横向机制与工程红线

### 7.1 确定性

1. 无时间输入：着色器与位姿不用 `uTime`、`clock()`、墙钟差值。
2. 位姿可复算：只由 `(scene_id, frame_index, camera_params, model_matrix)` 决定；`frame_index=n` 与直接以 n 初始化一致。
3. 遍历顺序固定：图元、像素、描述符、资源顺序显式定义。
4. 归约顺序固定：`ballot/shuffle/atomic add` 按 lane 升序或原子累加，不依赖完成顺序。
5. 随机固定种子且消费顺序固定。
6. 平局可判：深度相等、时间戳相等、稳定排序给出 tie-break。
7. 内存序显式：`Relaxed/Acquire/Release/AcqRel/SeqCst` + `Workgroup/Device` 作用域；shared 栅栏与全局屏障分开。

双执行策略与时间无关的完整职责见 `docs/design/execution-parallel.md`、`docs/design/sync-primitives.md`。

### 7.2 观测

- 软观测：耗时、加速比、缓存/TLB 命中率、每 SE 波分布、每线程负载、非探针像素。
- 统计输出：`state_inspector` 只读 MMIO、队列与统计，不反向写状态。
- 日志与统计不得进入硬判据。

### 7.3 错误

- 所有错误可定位到 `{unit, packetType, dwordOffset, va, code}`；无位置的错误视为日志缺陷。
- 错误路径不分配大对象、不递归、不抛异常。
- 第 36 章只做"故障注入 → 定位 → reset → 重跑一致"。

### 7.4 工程红线

| # | 红线 | 检查方式 |
|---|---|---|
| 1 | `LaneMask` 必须是 `u32`，取反必须 `& 0xFFFFFFFF` | 编译期类型 + 单测 |
| 2 | 控制流栈必须是 `{pc, mask}` 结构体，不做位域 | 代码审查 |
| 3 | 热循环内不得动态分配 | 代码审查 + 计数断言 |
| 4 | GART 只 bump，周期性缓冲跨帧复用 | 分配计数 |
| 5 | 所有访存统一 VA → TLB → L2 | 绕过翻译的直连检查 |
| 6 | 命令包不得跨环尾；不足用 `NOP` 填充 | 环边界测试 |
| 7 | 程序结束必须 flush 写回后再 `EXIT` | 终态检查 |
| 8 | 不得引入时间输入与未定义顺序 | 差分测试 |
| 9 | 硬件单元不得判断上层是 UMD 还是计算驱动 | 依赖/分支审查 |
| 10 | 不支持的能力必须返回 `NotSupported`，不得静默降级 | capability 测试 |
| 11 | 新增编号（包类型 / `stateId` / MMIO 偏移 / opcode / 页表位宽）必须先对照 `bit-budget.md` 审计并登记（`roadmap.md` §7.3）；**禁止在代码或章文档私自硬编码新编号**；新增功能优先载荷 `SubOp` / 间接状态表 / `INDIRECT` IB，禁止新增顶层包号与直接 MMIO 槽 | 登记表 diff 审查（ADR-043/044/045） |
| 12 | 硬件命令包 / MMIO 寄存器 / 页表项**禁止用 C/C++ 位域结构体直接映射二进制**；一律显式 mask+shift 序列化 / 反序列化，避免编译器位域布局差异 | 代码审查 + ABI 单测 |
| 13 | 大块状态结构（完整 PSO、绑定表、查询结果）**禁止整体放入 MMIO**；MMIO 只放栅极/控制/根指针/计数，实体放 GPU 内存 | MMIO 段占用审查（`bit-budget.md` §3.2） |

### 7.5 简化登记（初级）

| 简化 | 落点 | 替换章 |
|---|---|---|
| 单 VA / 直接映射 / bump | `hw_sim/memory/flat` | 11 |
| 无缓存 | 内存直通 | 12 |
| 无驻留/上下文切换 | 单上下文 | 14 |
| 单波、WARP32、无分支 | `hw_sim/shader/wave` | 15、16 |
| 手写 ISA，无编译器 | `hw_sim/isa` | 17~19 |
| 固定管线顺序、单 RT | `hw_sim/frontend` \| `raster` \| `om` | 22~28 |
| nearest 采样、无采样器对象 | 纹理单元 | 21 |
| 同步提交、无多队列 | 薄门面 | 32、33 |
| 无查询/统计 | — | 34 |
| 无错误恢复（记录 + 中止提交，不复位） | 错误即中止 | 36 |
| 无探针表/`spec_runner`（全图逐字节 + 结构断言） | `tools/image_diff` | 高级（第 21 章起） |
| 无 PAL（场景直连薄门面） | `hw_sim/submit` | 35 |

每条简化必须能在代码中定位到唯一实现点；高级替换时不得改变场景入口与探针判据。

### 7.6 编码资源设计权衡（直接编码 vs 间接编码）

> 顶层硬件编码（`PKT_TYPE`、`stateId`、opcode、MMIO 偏移）是**稀缺资源**；扩容手段分"直接编码"与"间接编码"两类，决策约束见 `roadmap.md` §7.4、审计见 `bit-budget.md`。本节为通用设计原则，任一新增编号必须先审计（§7.4 红线 11）。

| 手段 | 说明 | 代价 | 适用 |
|---|---|---|---|
| 直接编码（原地加宽） | 扩大位域 / 新增顶层包号 / 新增直接 MMIO 槽 | 破坏冻结的二进制包头与 MMIO 布局，E0~E5 基线全失效；位域此消彼长 | **尽量避免**（ADR-043/044/045 禁止首选取号） |
| 载荷 `SubOp` 子操作码 | 复用已有顶层包，载荷内 8~16 bit 子操作码 | 改包载荷布局（契约变更，走 ADR）；老包二进制兼容 | 首选（B-03 低成本位） |
| 间接缓冲 `INDIRECT` / IB | 顶层只消耗 1 个包号，细分命令放 GPU 内存 | 内存访问开销（模拟器无周期代价，可忽略） | 大命令流（`bit-budget.md` §2.3 方案 A） |
| 内存间接状态表 | MMIO 只存根指针，`SET_STATE_TABLE_INDEX(index)` 索引内存数组 | CP 多一次读内存；改动集中、不与包布局冲突 | `stateId 0~7` 用尽后（长期方案） |
| 页表级数扩展 | 保持 `GpuVAddr` u64 与 `IMemorySubsystem` 接口不变，只改内存子系统内部级数 / 索引位 | 页表布局属 ABI、需 ADR；上层包与场景透明 | VA 上限不足时（当前裁定 36 bit 已覆盖 64 GiB） |

**工程约束**：MMIO 寄存器稀缺，仅放根指针 / 门控 / 控制 / 计数，大块状态结构一律放 GPU 内存（§7.4 红线 13）；对外 ABI（`GpuVAddr` u64、`IMemorySubsystem::Translate` 签名）与内部实现解耦，页表级数是内存子系统内部细节；老二进制路径保留、新能力走新路径，不改老二进制的行为（基线兼容优先）。

---

### 7.7 横向关注点（每章都要逐条交代）

> 每章文档 DoD 第 8 节必须逐条回答下表（`roadmap.md` §7.2），无空缺、无未登记待定项。

| 关注点 | 要求 |
|---|---|
| 确定性 | 该单元引入的遍历 / 分配 / 并发必须确定序或确定归约；两种策略下硬判据一致 |
| 时间无关 | 不得引入时间输入；帧序由 `frame_index` 显式驱动 |
| 可观测 | 该单元暴露的最小统计量、错误路径与 `--trace` 字段 |
| 契约影响 | 是否修改契约冻结项；修改需登记决策记录并走 `roadmap.md` §7.4 |
| 简化登记 | 初级章必须在简化登记节登记简化点，并写明高级替换章 |
| 计算兼容 | 该单元在计算通路上是否有对应语义；共用单元不得为图形或计算任一方向特化 |
| 配置与回退 | 该单元能否关闭以回到简化基线 |
| 调试钩子 | shader printf / wave trace 只在显式开关下输出，不进入硬判据 |
| trace 契约 | `--trace` 仅供人工阅读，不保证字段稳定、不做机器解析契约、不进入硬判据 |
| 契约变更 | 是否改动冻结契约；改动必须走 `roadmap.md` §7.4 并回归 E0~E5 与受影响 S 场景 |
| 校验归属 | 该单元在哪些探针 / 断言上、按什么容差验收；探针表改动需登记（`docs/design/scene-specs.md` §1.3） |

### 7.8 共用单元的计算语义映射（"用图形模块实现计算"的落地表）

计算线**不新增任何 `hw_sim` 单元、不新增任何目录**；唯一计算专属目录是 `umd/compute`（ADR-009 已登记）。下表逐单元回答"图形做什么、计算做什么、怎么复用、计算侧不做什么"，凡"不参与"者构成计算通路的**旁路清单**，CP 不向其派发。

| 模块（落地章） | 图形职责 | 计算职责 | 复用方式 | 计算侧边界（不做） |
|---|---|---|---|---|
| `common`（1） | 包 / ISA / ABI / 格式转换 | 同一套 | 唯一连接点，只经 `common` 类型（§2.5 规则 5） | 不加计算专用格式或 opcode 语义 |
| `core/cp/ring`（2、10） | `DRAW` 派发 | `DISPATCH` 派发、多 Ring | CP 按包类型派发（`0x04` → Compute 引擎） | 不做计算专用队列优先级策略 |
| `memory/flat`（3） | 单 VA / bump | 参数区、scratch、buffer 同空间 | 同一 VA，零拷贝共享 | — |
| `memory/translation`（11） | 两段页表 / TLB | 同 | unified VA，ASID 为上下文粒度 | 不做计算专用分配器 |
| `memory/heaps`（11） | VRAM / GART / System | scratch 走 GART、结果走 VRAM | 三堆共用同一分配器 | 不设独立计算堆 |
| `memory/cache`（12） | L1 写穿 L2 | 同 + barrier 时 flush/invalidate | L1/L2 不按 workload 分叉 | 不做 CS 专属缓存 |
| `memory/residency`（14） | 驻留 / 逐出 | 同 | 同构复用 | 不做计算抢占（二期） |
| `shader/wave`（4） | warp32 / VGPR / 谓词 | 同 + `THREAD/GROUP` 内建 ID | 同一波模型与寄存器分区 | 不做动态并行 |
| `shader/divergence`（15） | 分支、控制流栈 | if / loop 内核分歧 | 同一重收敛机制 | barrier 一致到达由 ISA 校验（M-04） |
| `shader/scheduler`（15） | 波调度、纹理挂起唤醒 | 同 + 长延迟 global 访问挂起 | 同一就绪波队列（ADR-036） | 不做计算专用调度 |
| `shader/subgroup`（29） | 导数、少量 ballot | ballot / shuffle / reduce 主用 | 同一 lane 集合与 reduce identity | 导数只服务图形，计算不提供 |
| `shader/shared`（30） | 极少 | LDS / atomic / barrier 主用 | 同一 LDS 布局与 bank 规则 | 不为 LDS 新增专用包 |
| `se`（31） | SE tile 分区、CU 领取波 | workgroup 分块、共用线程池 | 同一 SE/CU 阵列与 occupancy 账本 | 不做计算专用分区策略（组不跨 SE 是约束） |
| `frontend/ia`（22） | 索引 / 拓扑 / 实例化 | **不参与** | — | 携带 IA 状态即 `InvalidPacket`（D-08） |
| `frontend/vgt_spi`（23） | VS 分派 / 切片 | **不参与** | — | 同上 |
| `frontend/clip_pa`（24） | 裁剪 / PA / 剔除 | **不参与** | — | 同上 |
| `raster/rasterizer`（25） | 三角形填充、插值 | **不参与** | — | quad 全执行规则不移植（M-02） |
| `raster/msaa`（27） | 覆盖 / resolve | **不参与** | — | — |
| `visibility`（27） | early-Z / Hi-Z | **不参与** | — | — |
| `om`（26） | depth / blend / MRT | 不参与状态机，**复用格式转换与写掩码**（S-04） | `common/format` 单一实现 | 不把 UAV 写回塞进 OM 状态机 |
| `geometry`（28） | HS / Tess / DS / GS | **不参与** | — | — |
| `texture`（21） | 采样 / LOD / 过滤 | image read（显式或无 LOD）、image store | 复用 tiling 解码 / 编码 | 隐式 LOD 导数只服务图形 |
| `query`（34） | occlusion / timestamp | dispatch timestamp、波数 / 线程数统计 | 同一 query pool | 不做计算专项计数器 |
| `periph/copy`（13） | H2D / D2H、blit | 参数与结果搬运 | 同一 Copy 引擎（`sa = nullptr`） | 不以 Copy 代替 kernel 内拷贝 |
| `periph/display`（13） | 扫描输出 | **不参与**（计算无 RT） | — | — |
| `firmware`（10） | ring / doorbell / 上下文加载 | 同 | 同一 FW，不解析 dispatch 语义 | — |
| `kmd`（10~14） | DDI / 内存 / 上下文 / 提交 | 同（**无独立计算 KMD**） | 同一 DDI 形状 | 不新增计算专用 ioctl |
| `hli`（35.1） | 对象面 + 图形管线 | `SetKernel` + `Dispatch`（同构） | 同一对象面（§3.4） | 不新增 `CreateComputePipeline` 平行入口 |
| `umd/compute`（38~42） | — | MUSA-like context/module/function/stream/event | 只依赖 HLI 与 `umd/os` | 不做 API / 二进制兼容 |

**计算通路旁路清单**（CP 不派发、包内不得携带其状态）：`ia`(22)、`vgt_spi`(23)、`clip_pa`(24)、`rasterizer`(25)、`msaa`(27)、`visibility`(27)、`geometry`(28)、`display`(13)。

**结论**：一条 dispatch 的旅程与一条 draw 共用 P0~P4（构造 → 绑定 → 管线 → 记录 → 提交），在 P5 分叉：draw 进 `ia→vgt_spi→clip_pa`，dispatch 直接进 `se` 展开 workgroup；尾部 draw 走 `om→display`，dispatch 走 `memory`（buffer）或 `texture`（image）写回 + `query`。分叉点唯一，且由包类型决定，不由上层驱动类型决定（ADR-032）。

### 7.9 计算能力档与计算业务目录

**能力分档**（沿用 ADR-025 四档；计算线共用单元不新增章号，计算驱动开发线为 stage3 新章 38~42，ADR-047）：

| 档 | 内容 |
|---|---|
| 高级一期（必须实现） | 一 / 二 / 三维 dispatch；`workgroup barrier`（含跨 wave）；subgroup `ballot/shuffle/reduce`；LDS + 原子；global UAV/SRV buffer；image read + image store（整元素）；间接 dispatch；GPU 侧谓词；compute 队列异步并行；timeline / event；timestamp 与 pipeline stats；格式转换写回；per-thread scratch；确定性归约顺序 |
| 高级二期候选（**先领 ADR 扩 `Capability` 位，未获前一律 `NotSupported`**） | grid-wide barrier（cooperative kernel）；cluster / distributed shared；计算抢占与 TDR 恢复；async copy（直通 LDS）；image 原子；fp16/bf16 打包运算 |
| 范围外（永不实现） | MUSA / CUDA API 或二进制兼容；动态并行（child kernel）；tensor / matrix 指令；可变 subgroup；fp8；稀疏计算 |

> 二期候选**不得**直接加进 35.1 的 `Capability` 枚举（§0.5 已冻结二期四位），必须先走 §7.4 契约变更并新建 ADR；在此之前调用返回 `NotSupported`，禁止静默降级。

**计算业务目录（WA：Workload Archetype）**——业务视角的 12 类计算，是 S18/S19 子 case 的来源；每类都只落在既有模块上：

| 编号 | 业务类 | 代表 kernel | 访存 / 同步特征 | 主要复用模块 | 判据类型 | 场景子 case | 落位章 |
|---|---|---|---|---|---|---|---|
| WA-01 | 元素级流式 | vector add、relu、cast、色彩空间转换 | 1 读 1 写，无同步，纯带宽 | wave / isa / memory / format | 整型逐字节；fp32 ≤1 ULP | S18 case-4 | 4、16、20 |
| WA-02 | 归约 | sum / max / min（两级） | subgroup reduce + LDS 树 + 跨组二次归约 | subgroup(29) / shared(30) / se(31) | ≤1 ULP + 归约顺序断言（强制） | S18 case-5 | 29、30、31 |
| WA-03 | 前缀和 | Blelloch scan（upsweep / downsweep） | barrier 密集，组内两趟 | shared(30) | 整型逐字节 | S18 case-6 | 30 |
| WA-04 | 2D 邻域 | 5 点 stencil、3×3 卷积 | 邻域读、边界 clamp | memory / shared / isa | 整型逐字节；fp32 ≤1 ULP | S18 case-7 | 20、30 |
| WA-05 | 矩阵乘 | tiled GEMM / GEMV | LDS tile 复用 + 寄存器分块 + FMA 序 | shared(30) / subgroup / se | ≤1 ULP（同序 FMA） | S18 case-8 | 29、30、31 |
| WA-06 | 直方图 / 原子竞争 | histogram、scatter-add | 高冲突原子 | shared(30) 原子 / cache | 计数逐字节；冲突数仅软观测 | S18 case-9 | 30 |
| WA-07 | 排序 | bitonic sort 多阶段 | barrier 密集 + LDS 交换 | shared / se | 结果逐字节 + 稳定 tie-break | S18 case-10 | 30、31 |
| WA-08 | 图像处理 | blur、sobel、warp + 计算↔图形互操作（扩展子 case，ADR-049） | 采样器读 + image store | texture(21) / format / om 转换 | UNORM 逐字节（固定舍入）；互操作同 `(x,y,format)` 逐字节 | S18 case-11、S19 case-4 | 20、21、26 |
| WA-09 | 仿真 / 采样 | N-body、Monte-Carlo | fp 归约 + 固定种子 RNG | subgroup / wave / isa | ≤1 ULP + RNG 消费序 | S18 case-12 | 29、31 |
| WA-10 | 非规则访问 | gather / scatter | 随机地址，TLB 压力 | translation(11) / cache(12) | 逐字节 + 越界可定位中止 | S19 case-1 | 11、12 |
| WA-11 | 多 kernel 流水 | 多 stage 图像流水 + async compute | 队列 / event / fence 交叉 | queue(32) / cmd_gen(33) | 顺序语义 + 数值等价 | S19 case-2 | 32、33 |
| WA-12 | 间接 / 谓词 | 由 GPU 侧参数决定的 dispatch | 参数在 GPU 读、谓词跳过 | cmd_gen / query(34) | 与直接 dispatch 逐值等价 | S19 case-3 | 33、34 |

> 场景子 case 编号唯一权威为 `docs/design/scene-specs.md` 的 S18/S19 节（编号统一**不补零**：`case-4..12` / `case-1..3`）；本表即 WA-01~WA-12 与子 case 的绑定（WA-01~09 → S18 `case-4..12`、WA-10~12 → S19 `case-1..3`）。**"2026-11-20 定案" = 文档已就地改完即生效**，`roadmap.md` §7.4 仅为流程留痕（同一次提交补登），不构成"待变更"。

**代表 kernel 规格（block / 静态 LDS / 关键约束）**：输入规模、数据类型、种子、期望产物以 `scene-specs.md` S18/S19「子 case 输入规格表」为**唯一权威**（本表与规格表不得各写一套数值，改值走 §7.4）：

| WA | block | 静态 LDS | 关键约束 | 必测断言 |
|---|---|---|---|---|
| WA-02 | 256 | 256 × 4 B | 两阶段：组内树归约 + 第二 dispatch 单组汇总 | 部分 lane 参与时 identity 正确（MIN=+∞ / MAX=−∞） |
| WA-03 | 256 | 2 × 256 × 4 B | 双缓冲，避免 upsweep/downsweep 竞争 | 每阶段 barrier 前后中间值快照 |
| WA-05 | 16 × 16 | 2 × 16 × 16 × 4 B | WPT 4 × 4，K 步长 16，FMA 顺序固定 | 与 CPU 参考模型逐元素 ≤1 ULP |
| WA-07 | 256 | 256 × 4 B | `log2(N)` 阶段，每阶段组内 barrier | 阶段数与 barrier 计数断言 |
| WA-09 | 128 | 0 | 固定种子、按 `gtid` 升序消费 | 同种子两次运行逐字节一致 |

---

## 8. 知识笔记（`docs/notes/`）

| 主题 | 内容 | 关联章 |
|---|---|---|
| API 差异 | Vulkan / D3D12 / OpenGL 在资源状态、描述符、同步、裁剪空间上的差异与统一方式 | 35.2~35.4 |
| Mesa 分层 | state tracker / gallium / NIR / winsys / llvmpipe 与本项目的对应关系 | 17、20、25、35.1 |
| MUSA 对象模型 | context/module/function/stream/event 与 HLI 的映射 | 38~42 |
| 光栅规则 | fill rule、top-left、2x2 quad、重心插值、guard band | 24、25 |
| 深度与 MSAA | early-Z、Hi-Z、coverage、resolve、深度平局 | 26、27 |
| 曲面管线 | HS/DS 载荷、tess factor、patch 存储、GS emit/restart | 28 |
| 内存序 | acquire/release、workgroup/device 作用域、原子与屏障 | 30、31 |
| 校验方法 | ref_model、探针、容差、差分矩阵 | 7、9 |
| 编译器 | SSA、φ、线性扫描、分歧分析、LLVM 对照 | 17~19 |

笔记只记录"为什么这样设计"与对照结论，不复制代码；项目范围和阶段以任务规划为准，模块树与层间契约以本文件为准，模块内机制和场景判据分别以对应 design 文档为准。
