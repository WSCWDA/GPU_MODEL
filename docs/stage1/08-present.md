# 第 8 章 帧输出

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章收口 E1 的完整验收口径：清屏、单 RT、双缓冲、提交后读回、PNG 导出，并让"出屏 = 显存""双缓冲前后帧可区分"成为机器判据。第 2、3 章的提交/fence 与内存通道在这里被验证为一条可用通路。
> 与本章相关的设计说明：`../design/validation-model.md` §1~§2（初级基线 raw RGBA8 + 场景元数据、全图逐字节）；`../design/memory-system.md` §7（BUMP 确定性）；第 3 章（`Download` 读回）；第 2 章（fence/提交）。
> 落地模块：`hw_sim/periph/display`（scanout 简化版）、`hw_sim/submit/facade`（`Download`/PNG 导出）、`tools/image_diff`（raw 比对）、`apps/scene_e1_clear_present`。

---

## 1. 目标与非目标

### 1.1 目标

1. 单 RT + 双缓冲：back buffer 渲染，提交完成后 flip 到 front，帧序由 `frame_index` 显式驱动。
2. 清屏语义：`CLEAR` 写背缓冲（0.25 灰 `[63,63,63,255]`），front 内容不被 Clear 覆盖（前后帧可区分）。
3. 读回：`Download` 读出前/后缓冲字节；对比通道证明"显存内容 = 透传上屏内容"。
4. PNG 导出：raw/PNG 双形态产物，PNG 仅供人工查看，raw 供机器比对。
5. E1 全绿：全图逐字节等于背景色、读回缓冲 = 显存、双缓冲前后帧可区分——本收口 E1（Q-27 的 A 方案的收口章）。

### 1.2 非目标

- 不做 vsync / 翻转链 / 显示器扫描时序 / 分辨率刷新率配置（第 13 章）。
- 不做窗口与 headless 解耦之外的任何显示后端（窗口是可选调试工具，见 `../design/validation-model.md` §8）。
- 不做多 RT / MRT / 交换链格式（第 26/13 章）。
- 不做 PNG 的严格字节基线（PNG 只做人工检视；机器判据一律用 raw）。

---

## 2. 单元框图

```
 scene_e1
   │ 提交（CLEAR + 双缓冲 flip）
   ▼
┌────────────────── hw_sim/periph/display（最小版） ──────────────────┐
│ Framebuffer  front / back 双缓冲（各 w*h*4 字节，[B,G,R,A]）         │
│ Flip          back→front（提交完成后由门面触发，frame_index++）      │
│ Scanout(简化) 出屏 = 直接透传 front 内容（无扫描时序，无 vsync）      │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ Download / raw 导出
                                ▼
                     tools/image_diff（机器比对） + PNG（人工）
```

---

## 3. 输入输出契约

### 3.1 双缓冲描述

```cpp
struct FramebufferDesc {
    GpuVAddr backVa;      // CLEAR 目标（渲染目标，stateId 0）
    GpuVAddr frontVa;     // 当前显示内容（创建时清为背景色，由 Flip 更新）
    u32      width, height;
    u8       format;      // 0 = R8G8B8A8_UNORM（本章仅此值）
};
```

| 操作 | 接口 | 语义 |
|---|---|---|
| 提交渲染 | 门面 `Submit(desc)` | 命令执行于 back 缓冲 |
| Flip | 门面内部 `Flip()` | `front = back` 内容拷贝（整块），`frame_index++` |
| 读回 | `Download(GpuVAddr, host, bytes)` | 任意缓冲全量读回 |
| 导出 | `ExportRaw(FramebufferDesc, path)` | raw RGBA8（[B,G,R,A] 内存序字节流） |
| 比对 | `tools/image_diff E1_raw E1_expected` | 逐字节 0 容差或按探针掩码 |

- `CLEAR` 只写 back；front 上残留上一帧内容，用于"前后帧可区分"断言。
- 两次连续 `Submit`（不 flip 的中间态可省略）后前后缓冲必须不同：设计上规定 `frame_index` 每 Flip 一次 +1，且清屏只在 back 上执行。

### 3.2 门面追加接口

```cpp
// 接续第 2 章 hw::submit
Result SubmitFrame(const DrawDesc&);   // = Submit + Flip（E1 专用便捷封装，内部两步可拆）
Result Readback(u64* frameIndex);      // 当前 front 快照指针描述（校验用）
```

- `SubmitFrame` 可被拆成 `Submit` + `Wait` + `Flip`，供测试分别断言。

### 3.3 E1 验收口径（本收口锚定）

| 断言 | 判据 |
|---|---|
| 全图等于背景色 | 256×256 全图逐字节都为 `[63,63,63,255]` |
| 读回 = 显存 | `Download(backVa)` 与 `Download(frontVa)` 内容在 Flip 后一致（front == back 快照） |
| 前后帧可区分 | flip 前后 `frame_index` 不同；用一个"第二帧不同清屏色"的变体用例断言两帧字节不同 |

---

## 4. 内部数据流

```
提交帧 n：
  CLEAR(back, 0xFF3F3F3F)   → 写 back 全图
  命令完成 fence
  Flip()：front ← back（整块拷贝）→ frame_index++
当前上屏（简化 scanout）：读 front 缓冲输出 = 逐字节透传，无时序状态机
读回/导出：Download(front|back) → ExportRaw → image_diff 比对
```

无独立显示状态机；"扫描输出"被压缩为"front 内容直接等于显示内容"这一恒等式，避免在第 13 章前引入无消费者时序。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T8.1 | ① 契约 | `FramebufferDesc`/`SubmitFrame`/`Readback`/导出接口冻结；E1 三条机器判据写进断言头 |
| T8.2 | ② 主体 | 双缓冲：back 渲染/front 显示、`Flip` 整块拷贝、`frame_index` 递增、清屏只作用于 back |
| T8.3 | ② 主体 | 读回与导出：`Download`、`ExportRaw`（[B,G,R,A] 内存序）、前置到 `image_diff` 的比对通道、PNG 人工导出（可选开关） |
| T8.4 | ③ 接入 | 门面 `SubmitFrame` 集成；`scene_e1` 双缓冲清屏上屏，含"第二帧不同清屏色"变体用例 |
| T8.5 | ④ 验收 | A8：E1 全绿收口（三条判据 + R08 回归项） |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D8.1 | 头文件独立编译；`FramebufferDesc` 偏移 `static_assert`；E1 三条判据常量与 §3.3 表一致 | L1 契约 |
| D8.2 | 创建时 `front` 与 `back` 均初始化为确定背景色；两次 `SubmitFrame` 后 `frame_index∈{1,2}`；flip 前 `Download(frontVa)` 保留上一帧快照，`CLEAR` 只改 back，翻转后 front 与 back 内容一致；第二色帧能区分前后帧 | L1 契约 |
| D8.3 | `ExportRaw` 输出与内存字节逐字节一致；`image_diff` 对 0 差异返回成功、对构造的 1 字节差异文件返回失败并指出坐标 | L1 契约 |
| D8.4 | `scene_e1`：E1 全图逐字节 = 背景色；读回缓冲 = 显存；双缓冲前后帧可区分用例通过；连续两次运行 raw 基线逐字节一致 | L1 契约 + A8 |
| D8.5 | A8 全绿：R08 回归项；初级 `--trace` 样例每章一份齐备（对照 `roadmap.md` §2.3 初级门禁） | L1 契约 |

---

## 7. 场景与判据（A8）

**绑定场景：E1 `scene_e1_clear_present`（清屏上屏）——本收口。**

| 项 | 内容 |
|---|---|
| 输入 | `0.25` 灰、`256x256`、1 帧（+第二帧变色变体） |
| 产物 | front/back raw、PNG、image_diff 报告 |
| 主判据 | 全图逐字节等于背景色；读回缓冲 = 显存；双缓冲前后帧可区分 |
| 本章专有判据 | ① 出屏 = 显存（front 透传恒等）；② 双缓冲前后帧可区分 |
| 回归项 | R08：E0–E4 已交付基线不退化；E0 单独执行契约自检；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 真扫描时序/vsync/翻转链在第 13 章；E1 输入冻结，不做改动 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | `frame_index` 驱动帧序；Flip 整块拷贝无半帧态；无墙钟/vsync |
| 时间无关 | 帧推进只与 `frame_index` 相关 |
| 可观测 | `Readback(frameIndex)`；raw 快照；`--trace=display` 打印 flip 前后 front/back 首字节 ......（人机两用，仅供人工） |
| 契约影响 | 双缓冲语义、E1 判据为冻结契约；变更需 ADR 并回归 E1 |
| 简化登记 | 单 RT、双缓冲、提交后读回、无 vsync/翻转链（§9） |
| 计算兼容 | 显示通路只吃 `frame_index` 与 front 缓冲，无图形/计算分支 |
| 配置与回退 | 无独立开关；PNG 导出可关 |
| 调试钩子 | `--trace=display` 仅人工 |
| trace 契约 | 不进入判据 |
| 契约变更 | 见"契约影响" |
| 校验归属 | 全部 L1 契约断言（逐字节） |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 显示 | 单 RT、双缓冲、提交后读回、无 vsync/翻转链 | 13（scanout/flip/vsync） |
| 交换链 | 无交换链对象 | 13 |
| 窗口 | 可选 PNG 人工导出，无窗口 blit | 35（`umd/os` 窗口工具） |
| 帧调度 | `frame_index` 手动递增 | 13（显示驱动） |

---

## 10. 待定项

- **Q-81（已定案）** 双缓冲的 `Flip` 保持"整块拷贝"，与真实硬件的指针交换不同：本章用拷贝是为了让"读回缓冲 = 显存"断言直观；指针交换差异在第 13 章引入交换链时以 ADR 记录，不影响 E1 判据。
- **Q-82（已定案）** 半帧态（Flip 与渲染交叠）在本章不存在（提交同步完成）；异步提交与 display 并发的"帧完整性"判据归第 13 章明确定义。
