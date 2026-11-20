# 第 35 章 API 后端（HLI / Vulkan / D3D12 / GL / 计算驱动）

> 本章收口用户态对象面：35.1 冻结 HLI 原生对象接口（`architecture-design.md` §3.4），35.2~35.4 实现 Vulkan/D3D12/OpenGL 三套图形 UMD。**原 35.5 MUSA-like 计算驱动已于 2026-11-19 迁出**：对象契约与验收归第 37 章（管线收口），实现主体归 **stage3 38~42（`docs/stage3/`，ADR-047）**。**35.1 冻结门禁**：场景直连 HLI 全绿后才允许写任何 UMD。全部路径硬判据一致（探针零容差）。
> 与本章相关的设计说明：`../architecture-design.md` §3.4（HLI 对象面）、§3.8（`KernelArgsLayout`）、§3.10（回调链）；`../design/compute-line.md` §2~§3（MUSA 对象模型）；ADR-009/022/024/041（自动引用计数）；`../design/sync-primitives.md`。
> 落地模块：`hli/`、`umd/graphics/api/{vulkan,d3d12,gl}`、`umd/graphics/pal`、`umd/compute`、`apps/scene_s17_*` 等。

---

## 1. 目标与非目标

### 1.1 目标

1. 35.1 HLI 原生对象面：`IHliDevice/Queue/Resource/View/DescriptorHeap/Pipeline/CommandList/QueryPool` + `QueryCapability` 唯一能力入口；提交统一经 KMD DDI。
2. HLI 双维度引用计数（ADR-041）：用户引用 + 硬件提交引用双双归零才释放 GPU 内存。
3. 三套图形 UMD：Vulkan/D3D12/GL 各封装同一 HLI（不重写命令生成）；UMD 用命令缓冲快照动态状态。
4. ~~计算驱动（原 35.5）~~ 已迁出：对象模型与 ABI 见 `docs/design/compute-driver.md`，实现与双入口等价验收归 stage3 38~42（A42，ADR-047）。
5. 能力位只从 HLI 读（唯一入口）；UMD 不许向 HLI 注入 API 语义枚举（`Vk*/D3D12_*/GL_*` 不出现）。
6. 四路径一致性（HLI/Vulkan/D3D12/GL）+ 计算驱动与直连 HLI 等价（探针零容差）。
7. 动态状态是命令缓冲快照：两个命令缓存各自提交互不干扰。
8. `HLI 不做 API 兼容 workaround`（ADR-022）；workaround 只放 UMD。

### 1.2 非目标

- 不做 API 完整兼容（子集；能力四档 ADR-025 之外返回 `NotSupported`）。
- 不做 API 二进制兼容/真实驱动提交。
- 不重实现编译器（消费 ISA；17~19 产出）。
- 不做 RenderPass 对象（HLI 用 LoadOp/StoreOp），不建 D3D12 的 root signature 完整层级（用 HLI 绑定槽映射）。
- stage3 38~42（计算驱动，原 35.5）不做 MUSA API/二进制兼容（只借鉴对象模型）。

---

## 2. 单元框图

```
 apps（api_backend=hli）──► HLI（35.1）──► KMD（DDI）──► FW ──► HW
 apps（api_backend=vulkan|d3d12|gl）──► umd/graphics/api/* ──► PAL ──► HLI
 apps（计算驱动）──► umd/compute ──► HLI
```

- 开放对象：Hli Handles（opaque）+ `IHli*` 纯虚接口；UMD 只使用公开对象。
- HLI 对象表 / 引用计数跨层共享。

---

## 3. 输入输出契约

### 3.1 HLI 对象面（`architecture-design.md` §3.4 冻结）

| 对象 | 关键方法 | 语义 |
|---|---|---|
| `IHliDevice` | `QueryCapability/CreateQueue/CreateResource/CreateView/CreateDescriptorHeap/CreatePipeline/CreateCommandList` | 唯一能力入口 |
| `IHliQueue` | `Submit/Wait/SignalSemaphore/...` | 队列家族 Graphics/Compute/Copy |
| `IHliResource` | `Map/Unmap（仅 host-visible）、GetVa、GetSize` | 只描述存储 |
| `IHliView` | `CreateSrv/Uav/Cbv/Sampler/Rtv/Dsv` | 用途+切片 |
| `IHliDescriptorHeap` | `Write/GetGpuVa` | 动态索引；图形计算共用 |
| `IHliPipeline` | `SetStages/SetFixedState/SetKernel` | 图形 stages+fixed；计算 kernel |
| `IHliCommandList` | `Bind*/Draw/Dispatch/Barrier/Copy/Reset` | 唯一命令生成出口 |
| `IHliQueryPool` | `Begin/End/Resolve` | 34 章 |

- 全部输入输出用 `common` 类型 + 自身句柄；不出现 API 枚举。
- 动态状态在记录时快照：提交用记录值，两命令缓冲互不干扰。

### 3.2 引用计数语义（ADR-041）

```
对象有 refCount{userRef, hwRef}
userRef++：Create/UMD 持有
hwRef++：硬件提交引用（入 ring/IB 前 ++，完成 fence 后 --）
释放：仅当两者都 0 → 真正 FreeGpuMem。
```

- 引用计数不可被外界打开（opaque），只经 HLI 方法操作。

### 3.3 UMD 约束（35.2~35.4）

| 后端 | 约束 |
|---|---|
| Vulkan-UMD | `VkBuffer→IHliResource`,`VkImageView→IHliView`,`VkPipeline→IHliPipeline`,`VkCommandBuffer→IHliCommandList` |
| D3D12-UMD | `ID3D12Resource→IHliResource` 等；命令表 → HLI cmdlist |
| GL-UMD | `glBufferData`→资源；状态 → PSO fixedstate |

- UMD 不拼包（必须经 `IHliCommandList`）；能力只从 `QueryCapability`。
- 各后端"动态状态快照"语义一致。

### 3.4 计算驱动对象（原 35.5，已迁出 stage3 38~42；权威 `docs/design/compute-driver.md`）

| MUSA 概念 | 本项目 |
|---|---|
| `Context` | HLI device + KMD context（ASID），1:1:1 |
| `Module` | `MUSA_MODULE_HDR` + code + 符号表（name/entryOffset/paramBytes/vgprCount/ldsBytes），常驻 GART |
| `Function` | module 入口（`codeBaseVa + entryOffset` + `KernelArgsLayout`） |
| `Stream` | 1:1 映射 HLI 队列 |
| `Event` | 独立对象 `{queueId, timelineValue}`；Record → `SIGNAL`，Wait → `WAIT` |

- kernel 启动：`LaunchKernel(kernel, block, grid, args[])` → 打包参数 GART → `DISPATCH`。
- 计算驱动（stage3 38~42，原 35.5）只消费 HLI，不直接碰硬件单元。

### 3.5 Result → API 错误码映射示例（单一来源 `architecture-design.md` §3.9）

映射表本体与硬规则（禁止吞错、创建期 vs 执行期区分、UMD 一次性映射）见 `../architecture-design.md` §3.9，本章举例帮助实现对照（防止"创建期、执行期映射混淆"，Mesa 真实驱动高频踩该坑）：

1. **创建期**：UMD 查询 `QueryCapability` 发现目标能力位恒 false（如 35.2 之前无 MSAA 位）→ HLI `CreateXxx` 返回 `NotSupported` → Vulkan-UMD 在创建面映射为 `VK_ERROR_FEATURE_NOT_PRESENT`（`E_NOTIMPL` / `GL_INVALID_OPERATION`）。创建期 `InvalidArgument`/`NotSupported` → 参数/特性类错误。
2. **执行期**：提交后的命令流遭遇引擎故障 → 底层 `ResetRequired`/`DeviceLost`（36 章复位入口）→ Vulkan-UMD 在 `vkQueueSubmit`/`vkWaitForFences` 面映射为 `VK_ERROR_DEVICE_LOST`（`DXGI_ERROR_DEVICE_REMOVED` / `GL_CONTEXT_LOST` / `MUSA_ERROR_DEVICE_LOST`），**不得**再重复报创建期参数类错误。
3. **同一底因、两个时期**：`NotMapped`——创建期在映射侧返回 `VK_ERROR_MEMORY_MAP_FAILED`；同一 `NotMapped` 在执行期（着色器访存未映射触发提交中止）→ UMD 上报设备丢失类错误而非 `VK_ERROR_MEMORY_MAP_FAILED`。

边界：`kernel 命中未映射 paramBufVa` 属创建期（校验见 20 章 `KernelArgsLayout`），映射 `NotMapped`→`MUSA_ERROR_INVALID_VALUE`/`VK_ERROR_MEMORY_MAP_FAILED`；加载后执行期再命中同地址 → 设备丢失类。HLI 层自身不出现任何 API 错误码（`architecture-design.md` §3.9 规则 3）。

---

## 4. 内部数据流

### 4.1 直连 HLI（35.1）冻结流程

```
IHliDevice.CreateQueue → CreateCommandList → BindDescriptorHeap
→ BindPipeline（PSO 含 stages+fixed）→ Draw/Dispatch/Barrier/Copy
→ IHliQueue.Submit（写 ring + doorbell，35.2 前仅 HLI 层）
→ Wait/Host wait → 场景校验
```
（场景入口 `api_backend=hli` 与初级薄门面 `main` 兼容，场景数据不变——ADR-028 保持。）

### 4.2 UMD 命令快照

```
UMD 记录命令缓冲：Bind* / Draw ... → 生成 IHliCommandList（记录时快照动态状态）
提交：IHliQueue.Submit(cmdList) → HLI 检查引用计数 → KMD Submit → ring
两 cmdlist 各自提交：快照互不干扰
```

### 4.3 计算驱动启动

```
LaunchKernel(fn, grid, block, args):
  驱动写参数缓冲 GART（按 KernelArgsLayout.paramOffsets）
  HLI.CreatePipeline(kernel) → HLI.command.Dispatch(grid, block)
  stream.Record(SIGNAL(timeline)) → event
```

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T35.01 | ① 契约(35.1) | HLI 对象接口 + 句柄 + 能力查询 + 动态状态快照语义落头文件 |
| T35.02 | ② 主体(35.1) | HLI 对象工厂与资源表、`QueryCapability` 唯一能力入口 |
| T35.03 | ② 主体(35.1) | 双维度引用计数（user/hw ref + 释放条件） |
| T35.04 | ③ 接入(35.1) | 场景直连 HLI 全 E/S 跑通；**冻结门禁：全绿后才 35.2** |
| T35.05 | ④ 验收(35.1) | A35.1：直连 HLI 全 E/S 探针全绿并留档门禁记录（D35.04/D35.05） |
| T35.06 | ① 契约(35.2) | Vulkan-UMD 对象映射 + 命令缓冲快照 + capability 只读 |
| T35.07 | ② 主体(35.2) | Vulkan 资源/视图/管线/命令表 → HLI 映射 |
| T35.08 | ② 主体(35.2) | Vulkan 同步（fence/semaphore）→ HLI Queue |
| T35.09 | ③ 接入(35.2) | `api_backend=vulkan` S 场景四路径一致 |
| T35.10 | ④ 验收(35.2) | A35.2：Vulkan 路径在全部 E/S 上与 `hli` 逐字节一致（D35.09） |
| T35.11 | ① 契约(35.3) | D3D12-UMD 映射 + 命令列表快照 |
| T35.12 | ② 主体(35.3) | D3D12 资源/视图/PSO/列表 → HLI |
| T35.13 | ② 主体(35.3) | D3D12 同步 → HLI 队列/semaphore |
| T35.14 | ③ 接入(35.3) | `api_backend=d3d12` 四路径一致 |
| T35.15 | ④ 验收(35.3) | A35.3：D3D12 路径在全部 E/S 上与 `hli` 逐字节一致（D35.14） |
| T35.16 | ① 契约(35.4) | GL-UMD 映射 + 状态 → PSO fixed |
| T35.17 | ② 主体(35.4) | GL 资源/纹理/着色器 → HLI；GL 同步 → fence |
| T35.18 | ② 主体(35.4) | GL 状态机（状态、绑定）→ HLI 快照 |
| T35.19 | ③ 接入(35.4) | `api_backend=gl` 四路径一致 |
| T35.20 | ④ 验收(35.4) | A35.4：GL 路径在全部 E/S 上与 `hli` 逐字节一致（D35.19） |
| ~~原 35.5 计算驱动任务~~ | —（**2026-11-19 迁出**） | 计算驱动任务归 **stage3 38~42**（T38.x~T42.x，`docs/stage3/`，ADR-047）：对象契约头文件 + `MUSA_MODULE_HDR` → module 加载 + 参数打包 + LaunchKernel → stream/event → 双入口等价验收（A42） |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M35.1 | T35.01~T35.05 | HLI 对象面落地并通过冻结门禁，D35.01~D35.05 全绿 |
| M35.2 | T35.06~T35.10 | Vulkan 路径四路径一致，D35.06~D35.10 全绿 |
| M35.3 | T35.11~T35.15 | D3D12 路径四路径一致，D35.11~D35.15 全绿 |
| M35.4 | T35.16~T35.20 | GL 路径四路径一致，D35.16~D35.20 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D35.01 | HLI 头独立编译；句柄 opaque；能力查询返回能力位（`QueryCapability`） | L1 契约 |
| D35.02 | 工厂对象创建/销毁关闭；能力查询只从 HLI | L1 契约 |
| D35.03 | 引用计数：对象 user=0,hw=0 前不释放内存（测试模拟未完成提交释放被拒） | L1 契约 |
| D35.04 | **冻结门禁**：直连 HLI 全部 E/S 场景探针全绿 | 横切 + 门禁 |
| D35.05 | 门禁记录留档：全绿证据可查；未全绿时 35.2 任何任务不得开工 | L1 契约 |
| D35.06 | Vulkan-UMD 契约：对象映射表落头文件；`Vk*` 不进 HLI；动态状态快照语义 | L1 契约 |
| D35.07 | Vulkan 资源/视图/管线/命令表映射正确；两 cmdlist 各自提交互不干扰（快照） | 横切 |
| D35.08 | Vulkan fence/sem → HLI 队列 timeline | L1 契约 |
| D35.09 | `api_backend=vulkan` 与 `hli` 探针逐字节一致 | 横切零容差 |
| D35.10 | A35.2 全绿：Vulkan 路径在全部 E/S 上逐字节一致，回归记录留档 | L2 + A35 |
| D35.11 | D3D12-UMD 契约：映射表落头文件；`D3D12_*` 不进 HLI；列表快照语义 | L1 契约 |
| D35.12 | D3D12 资源/PSO 映射正确 | L1 契约 |
| D35.13 | D3D12 同步 → HLI 队列 | L1 契约 |
| D35.14 | `api_backend=d3d12` 与 `hli` 逐字节一致 | 横切 |
| D35.15 | A35.3 全绿：D3D12 路径在全部 E/S 上逐字节一致，回归记录留档 | L2 + A35 |
| D35.16 | GL-UMD 契约：状态 → PSO fixed 映射表落头文件；`GL_*` 不进 HLI | L1 契约 |
| D35.17 | GL 资源/纹理 → HLI | L1 契约 |
| D35.18 | GL 绑定/状态机 → HLI 快照一致 | L1 契约 |
| D35.19 | `api_backend=gl` 逐字节一致 | 横切 |
| D35.20 | A35.4 全绿：GL 路径在全部 E/S 上逐字节一致，回归记录留档 | L2 + A35 |

---

## 7. 场景与判据（A35）

**绑定场景：S17（四路径一致）、S2/S3/S4/S5（各 UMD 路径覆盖）。**

| 项 | 内容 |
|---|---|
| 输入 | 同一 E/S 场景分别以 `api_backend=hli/vulkan/d3d12/gl` 跑（计算入口归 37/42，本章不含） |
| 产物 | 四路径探针表、命令缓冲快照 |
| 主判据 | 四路径探针逐字节一致 |
| 本章专有判据 | ① 三路径（Vulkan/D3D12/OpenGL）逐字节一致；② UMD 不改写 HLI 包；③ 动态状态是命令缓冲快照；④ 能力位只从 HLI 读；⑤ 计算驱动 kernel 启动与直连 HLI 等价归 stage3 38~42（原 35.5，A42） |
| 回归项 | 全部 E/S 在不切换后端下均通过；UMD 引入不改变硬件单元契约 |
| 后续补齐 | API 特化兼容（workaround）只在该 UMD 层，不反向进 HLI |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 命令生成序、快照取时点、引用计数增减序确定 |
| 时间无关 | 无墙钟；fence 值为逻辑序 |
| 可观测 | 命令缓冲快照 dump、能力位、引用计数表、`--trace=hli/umd` |
| 契约影响 | HLI 对象面/引用计数/能力入口冻结（35.1 门禁） |
| 简化登记 | 初级薄门面 → HLI → UMD（三层替换，场景入口不变） |
| 计算兼容 | 计算驱动共用 HLI（跳图形 UMD）；硬件单元按包类型派发不按 UMD 类型 |
| 配置与回退 | `api_backend` 键；`hli/vulkan/d3d12/gl` 全部路径；blit 窗口 `BW_WINDOW` 为内部常量（非配置键） |
| 调试钩子 | `--trace=hli/umd/compute`；能力位 dump |
| trace 契约 | 不进入判据 |
| 契约变更 | HLI 对象面、引用计数、能力位为冻结项；变更走 ADR 回归 S17 及受影响 S |
| 校验归属 | L1（结构/引用计数）+ L2（S17 输出）+ 横切（四路径） |
| 双策略 | 四路径在两策略下一致（UMD 层与执行策略无关） |
| 阶段衔接 | E 阶段门禁（S12/S13/S15/S16/S17/S19 + 契约冻结 + 全 E/S 双策略不退化）在此章对齐 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 提交通道 | 薄门面 → HLI（35.1 定案） | — |
| 图形 UMD | 无 → Vulkan/D3D12/GL | —（35.2~35.4） |
| 计算驱动 | 无 → context/module/function/stream/event | stage3 38~42（原 35.5，ADR-047） |
| 引用计数 | 无 → 双维度 | —（41 号 ADR） |

---

## 10. 待定项

- HLI 的 `QueryCapability` 返回能力位集（graphics stages 位、计算位、绑定位、sampler 位、MSAA 位）——具体位枚举在 35.1 实现时按“只用一次查全能力访问”的原则定稿，登记到 `command-mmio` 或 ADR。
- 三套 UMD 的 API 特化 workaround（如 GL 状态机差异）各自放对应 UMD，是否统一管理在 `umd/graphics/os` 层——先分散，35.4 后再评估合并（属工程层面，不进契约）。
- Vulkan 的 `vkCmdDrawIndirect` 与 D3D12 的 ExecuteIndirect 参数布局差异：HLI 层统一抽象（33 章 `DRAW_INDIRECT` 载荷），各 UMD 映射；若发现不可统一语义，以 ADR 记录 API 差异并只在 UMD 兼容。
