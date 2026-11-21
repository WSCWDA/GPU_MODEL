# 纹理单元 — 独立模块设计

> 项目目标、范围、阶段与交付以[任务规划](../CPU模拟GPU教学项目_任务规划与实施规格.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

> 定位：定义纹理采样子系统的格式解码、存储布局、mip/LOD、过滤、寻址、比较采样与采样器对象。初级只做最小集（RGBA8 / nearest / clamp），高级补齐完整单元并接入异步延迟隐藏。
> 落地章：第 7 章纹理最小集（初级，M）、第 21 章纹理单元（高级，M21.1~M21.3）。
> 权威来源：从 `architecture-design.md` §4.10 拆出，吸收章节总览与初级简化登记（原 §4.2 / §3.3，权威在 `roadmap.md` §2.1 / `architecture-design.md` §7.5）、Mesa NIR/llvmpipe 对照与 ADR-036（纹理延迟隐藏）、`roadmap.md` §2.8（数组/Cubemap/独立采样器/各向异性）。

---

## 1. 环节约定（第 21 章完整版）

| 环节 | 约定 |
|---|---|
| 格式解码 | 一期支持 RGBA8/BGRA8、R8、RG8、D24S8；sRGB 采样时解码到线性空间 |
| 存储布局 | 线性与 tile 两种；tile 尺寸由格式与 mip 决定，寻址对上层透明 |
| mip / LOD | LOD 由 `float2 uv` 与 `dPdx/dPdy` 及采样器状态计算；相邻 mip 三线性过滤；越界 clamp 到有效层 |
| 过滤 | nearest、bilinear、trilinear；各向异性只保留 capability bit，一期拒参（返回不支持） |
| 寻址 | wrap / clamp / mirror / border；数组层与 Cubemap 面选择显式 |
| 比较采样 | `SAMPLE_CMP` 返回比较结果；shadow 路径不改变格式解码 |
| SAMPLE 请求 | `float2 uv + dPdx/dPdy`；LOD 由纹理单元计算，不由着色器预计算 |
| 采样器对象 | 第 21 章引入，写入 `stateId 5` 描述符堆、复用 `BIND_DESCRIPTOR_TABLE`；初级只做固定 nearest，不建采样器对象 |
| 只读边界 | 纹理单元只读；绑定与视图归第 20 章，采样器写入 `stateId 5` 描述符堆并经 `BIND_DESCRIPTOR_TABLE` 下发（Q-52） |

## 2. 初级最小集（第 7 章，初级简化登记，高级由第 21 章替换）

| 项 | 初级实现 | 高级替换章 |
|---|---|---|
| 格式 | RGBA8 | 21 |
| 过滤 | nearest | 21 |
| 层 | 单层、无 mip、无 sRGB | 21 |
| 采样器 | 无采样器对象 | 21 |
| 寻址 | 越界 clamp | 21 |

初级活验收：E3 nearest 全图逐字节；UV 越界 clamp 行为确定。

## 3. 高级完整单元（第 21 章）

- M21.1 格式解码 / sRGB / tiling；M21.2 mipmap / LOD / 过滤；M21.3 数组 / Cubemap / 独立采样器 + 异步延迟隐藏。
- 错误语义：纹理错误（越界、非法格式、非法 LOD）返回 `Result` 并带 `{资源 VA, 坐标, 格式}`，**不做静默 clamp**（采样器 clamp 属于合法寻址，不算错误）。
- 场景：S4（纹理四边形 + 2 层数组 + Cubemap）、S14（bindless + 全阶段 UAV 中的纹理采样路径）。
- 补强项：独立采样器对象、数组纹理 / Cubemap、sRGB、比较采样（`roadmap.md` §2.8）。

## 4. 延迟隐藏接入（ADR-036）

纹理采样异步化是第 21 章重要部分：波发出纹理请求即挂起、让出 CU 给就绪波，结果返回后唤醒（与 `../docs/design/execution-parallel.md` §5 同一机制；波调度器维护"就绪波队列"）；软观测：波切换次数。只影响性能与调度观测，不影响采样结果。

## 5. 与内存 / 校验的关系

- 采样请求统一走 `IMemorySubsystem` 读入（`../docs/design/memory-system.md` §8），不绕过翻译；L1/L2 缓存对纹理同样生效。
- 探针表必须覆盖纹理角点与 LOD 边界（`../docs/design/validation-model.md` §3）。

## 6. 契约变更

- 格式列表、tiling 布局、寻址模式、采样器语义变更走 `roadmap.md` §7.4（新建 ADR）并回归 S4/S14。
