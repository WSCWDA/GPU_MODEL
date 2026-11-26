# GPU 驱动全栈软模型 — 场景规约（scene-specs）

> 项目范围、阶段与展示目标以[任务规划](../README.md)为主干；本文仍是 E0–E5 / S0–S19 场景输入、探针和判据的唯一技术权威，roadmap 与章节文档只引用并落实这些规约。

> 状态：**正式维护（v1，2026-11-19）**。E0~E5 / S0~S19 的**完整测试规约统一收敛在本单份文件**（每场景一节 `Exx` / `Sxx`）；不建 `docs/scene-specs/` 目录（Q-38）。原 `graphics/` 分支参考文档（章节总览概览表与测试用例）已按 Q-40 将数值按 warp32 重算并入本文件后归档删除，精简概览表由本文件 §2/§3 承接。
> 定位：**场景文档是冻结契约**。修改场景输入、探针集合、阻断判据、KFT 绑定，必须走 `roadmap.md` §7.4 契约变更工作流（新建 ADR）；`apps/scene_*` 严格对齐本文件对应节，不得私自修改输入参数。
> 参照源：**只用 CPU 参考模型 `tools/ref_model`（ADR-027）**，不做真机 / 外部 GPU 参照渲染。探针期望值由 `ref_model` 解析模型**实时生成**（或固定基线），仓库不存高级 S 的完整图像基线；历史手工数值（如 256 基线覆盖 20808）仅作过渡参考，`ref_model` 落地后以 `ref_model` 为准、禁止手改期望（Q-40、Q-78）。
> 本文件是场景输入与判据的唯一技术权威；校验模型全局约定见 `validation-model.md`。项目目标和阶段由任务规划定义，roadmap 和章节文档引用本文件，不复制或自行修改场景规格。

---

## 1. 统一模板与全局约定

本文件不重复每节样板，每个场景一节、按下列**固定十节模板**落内容（§2 起按场景实例化）：

1. **元信息**：场景 ID、程序入口、阶段（初级 / 高级）、关联章节、判定类型（阻断 / 告警 / 观测 / 人工）、KFT 标记、spec 版本、变更 ADR。
2. **前置条件**：硬件配置（`rt_size`、`se_count`、`cache_enable` 等，见 `roadmap.md` §5.1 配置）、功能前置、环境约束（headless / 窗口）。
3. **输入集合**：固定输入 + 子 case 可变参数；初级 E 输入**永久冻结**（修改必须 ADR，只能改探针容差）。
4. **标准化执行步骤**：构建 → 运行（`tools/spec_runner`） → `ref_model` 生成期望 → 比对 → 产物入库。
5. **输出产物集合**：模拟器 `.raw` 帧、探针报告、统计、日志；详见 global 规则。
6. **判定矩阵**（四分类，含容差与失败行为）。
7. **子 case 列表**（`case-0 … case-N`，复用同一探针 / 判据体系）。
8. **退出条件**。
9. **回归矩阵**（`exec_mode` / `cache_enable` / `se_count` / `worker_threads` 等组合）。
10. **引用**（章文档 DoD、ADR、`validation-model.md`、`roadmap.md` §7.3 登记）。

### 1.1 判定矩阵四分类（每项含容差与失败行为）

- **阻断门禁**：不满足即 CI 失败。初级 E：全图逐字节 + L1/L2 断言；高级 S：探针表 L2 为阻断门禁，全图差分仅告警。
- **仅告警**：输出差分图等，不阻断。
- **纯观测统计**：wave 数、缓存命中率等，仅日志。
- **仅人工检视**：窗口可视化画面，不参与自动化 CI。

### 1.2 全局容差口径（唯一）

- 颜色 **±2/通道**、深度 **±1 LSB**。1× 覆盖集合与计数零容差。S9 每像素多样本覆盖率是覆盖样本数除以配置样本数，绝对容差 **1/16**。计算 fp32 **≤1 ULP 通过、>1 ULP 阻断**。
- 所谓"像素零容差"只适用于 **1x 覆盖率与 L1 结构 / 计数断言**，不适用于浮点滤波后的颜色（Q-60）。
- 容差集中管理于 `tools/ref_model/tolerances.toml`，禁止散落在场景代码里。

**计算数值口径（**2026-11-20 定案**，登记 `tolerances.toml`；改值走 §7.4 变更）**：

- 整数 / 位运算 / 索引 / 计数 / 覆盖率：**0 容差**（不进探针表，直接用断言）；
- fp32 `add / mul / fma / div / sqrt`：**≤ 1 ULP（`tolerances.toml` 可配置）**，前提参考模型用同一 op 序 + 同一 FMA 合并规则；
- fp32 超越函数（`sin / exp / log / pow`）：**≤ 2 ULP**，实现不同源须在 `tolerances.toml` 登记；
- 归约 / 原子累加：**归约顺序写死** = `gtid` 升序 + 树形（组内树、组间按 gtid），≤1 ULP + **顺序断言强制**；
- 与 `roadmap.md` §6.7 数值容差表同源，本节为唯一权威。

### 1.3 探针模型（高级 S）

- 探针表每场景一份，坐标用归一化 **Q16**（与 `rt_size` 解耦），固定顺序 `(y, x)` 升序，数量 16~256，必须覆盖前景内部、图元边缘、背景、纹理角点、被遮挡区。
- 探针表是场景输入的一部分，**禁止运行期动态生成**；改动需在章文档登记。
- 探针期望值来自 `tools/ref_model`（解析计算或固定基线），不在代码内硬编码、不由宿主真机渲染得到。
- 任何改变像素输出的实现变更，必须在同一次提交内更新探针表，否则 CI 持续失败。

### 1.4 双轨校验 / 基线存储 / 产物入库

- **高级 S 双轨**：阻断门禁 = L1 契约断言 + L2 探针点比对（期望值 `ref_model` 实时生成）；另加**非阻断全图掩码差分（仅告警）**，阈值 `max_allowed_bad_pixels`，超阈值输出告警与差分 PNG；**严禁把全图差分升级为阻断门禁**。
- **基线存储**：初级 E 校验基线是原始 BGRA `.raw` 二进制，git 入库；PNG 仅人工查看、不作为校验真值；高级 S 不存储任何完整图像基线，仓库只存探针文本表；禁止大尺寸 PNG 进 git。
- **产物入库**：本文件、探针文本表、初级 E 的 `.raw` 基线 git 入库；PNG、差分图、trace 日志为 CI 临时产物、不入库。

### 1.5 KFT 绑定 / 回归矩阵 / CI 诊断

- **KFT**：二期未实现能力可标记整个场景或部分探针点为 Known-Failing-Test，绑定 issue 编号；`kft_allow=on` 时 CI 统计并输出警告、**不阻断门禁**；禁止直接注释 / 删除测试用例，功能实现后移除标记。
- **CI 回归矩阵**：编译器 MSVC / GCC / Clang × `exec_mode ∈ {serial, parallel}` × `cache_enable ∈ {on, off}`；初级 E0~E5 全字节校验**强制关闭 `-ffast-math`**（否则跨编译器字节不一致），高级 S 不依赖全字节、允许浮点优化。
- **校验失败降级行为（`spec_runner` 契约）**：对回归矩阵**逐配置独立判定**——任一组合失败都如实上报并阻断门禁，**禁止自动降级 / 回退**（不得以 serial 掩盖 parallel 差异、不得缩矩阵、不得把失败当 KFT / 告警吞掉）；31 章前 `exec_mode=parallel` 必须显式报"策略 B 未实现"。任一失败产出六件套诊断制品：①输出 `.raw`；②`ref_model` 期望 `.raw`；③差分放大 PNG；④CSV 探针完整报告；⑤场景配置快照 JSON；⑥对应场景 `--trace` 日志。

### 1.6 变更工作流

场景文档是冻结契约：修改输入 / 探针集 / 阻断判据 / KFT 绑定，必须新建 ADR（`roadmap.md` §7.4），更新本文件 → 更新受影响场景 DoD 与判据 → 登记（`roadmap.md` §7.3）→ 回归 E0~E5 + 受影响 S（含 `serial`/`parallel`）→ 评审合并。

### 1.7 尺寸缩放约定

探针坐标与覆盖率随 `rt_size` 推导（`common/config.h` 配置）：坐标按 `x * rt_width / 256`、`y * rt_height / 256` 缩放；期望颜色由参考模型在同一像素中心重新推导，不使用固定字节（Q-35、Q-40）。

---

## 2. 初级场景（E0~E5，输入永久冻结）

> 初级判定以全图逐字节为兜底；探针 / 结构断言为辅助。输入（几何、纹理、配置参数）**永久冻结**：高级阶段只允许改校验口径（探针 / 容差），不得改输入数据（`roadmap.md` §0.4、ADR-011）。

### E0 `scene_e0_selftest` — 契约与工具自检

- 元信息：初级 / 章 1、5 / 阻断；入口 `apps/scene_e0_selftest`；无输入。
- 输入：无。
- 主判据：结构体 / 枚举 / 命令包头断言全绿；汇编 → 二进制 → 反汇编 → 再汇编，两次二进制相同（L1，零容差）。
- 子 case：`case-0` 汇编往返；`case-1` 指令子集全 smoke 无解码失败。
- 回归矩阵：`serial`；全字节校验关 `-ffast-math`。
- 引用：章 1 DoD、章 5 DoD、`roadmap.md` §2.1。

### E1 `scene_e1_clear_present` — 清屏上屏

- 元信息：初级 / 章 2、3、8 / 阻断；入口 `apps/scene_e1_clear_present`。
- 输入：背景色 `0.25` 灰（`0xFF3F3F3F` 即 RGBA `(63,63,63,255)`），`rt_size=256x256`（基线），单次提交。
- 产物：全屏 `(63,63,63,255)`。
- 主判据：全图逐字节等于背景色；读回缓冲 = 显存内容；双缓冲前后帧可区分（L1 + 全字节）。
- 备注：内存序 BGRA `[B,G,R,A]=(63,63,63,255)`；`0.25×255=63.75→63` 截断（`01-contracts-min.md` D1.3 口径）。
- 子 case：`case-0` 256×256；`case-1` 1080×720（期望同口径截断）。
- 回归矩阵：`serial`；关 `-ffast-math`。
- 引用：章 2/3/8 DoD。

### E2 `scene_e2_triangle` — 渐变三角形（初级的核心兜底）

- 元信息：初级 / 章 4、5、6 / 阻断；入口 `apps/scene_e2_triangle`。
- 输入（**永久冻结**）：三顶点 NDC `v0(-0.8,-0.8,0)` 红 `(255,0,0,255)`、`v1(0.8,-0.8,0)` 绿 `(0,255,0,255)`、`v2(0,0.8,0)` 蓝 `(0,0,255,255)`；顶点布局 position `float4`(offset 0) + color `float4`(offset 16)、stride **32**；灰底 `0xFF3F3F3F`。
- 产物：灰底 + RGB 渐变三角。
- 主判据：全图逐字节匹配固定基线（初级核心兜底）；退化三角形不写像素；Gouraud 插值（VS 透传 position+color → VS-out 区重心插值 → PS 输出）。
- 历史参考值（`ref_model` 落地前过渡，Q-78）：256×256 基线覆盖率 **20808**（"与背景色不同的像素数"）；1080×720 实测 **248832**；覆盖/探针由 `PredictedCoverage()` 按当前 RT 推导，不再硬编码基线值。
- 全图不变式：覆盖面内每像素 A ∈ [253,255] 且 R+G+B ∈ [252,255]（仿射插值 + 逐通道截断），可检出任意位置通道错位。
- 回归矩阵：`serial`；双路径（手写 ISA）逐字节一致。
- 引用：章 4/5/6 DoD、`06-fixed-pipeline-min.md` Q-78。

### E3 `scene_e3_textured_quad` — 纹理四边形

- 元信息：初级 / 章 7 / 阻断；入口 `apps/scene_e3_textured_quad`。
- 输入：2×2 纹素（红 / 绿 / 蓝 / 黄）+ uv；RGBA8；nearest；越界 clamp。
- 产物：四象限色块。
- 主判据：nearest 全图逐字节；UV 越界 clamp 行为确定。
- 子 case：`case-0` nearest；`case-1` uv 越界 clamp。
- 回归矩阵：`serial`；关 `-ffast-math`。
- 引用：章 7 DoD、`07-texture-nearest.md`。

### E4 `scene_e4_mesh` — 非索引大网格

- 元信息：初级 / 章 3、6 / 阻断；入口 `apps/scene_e4_mesh`。
- 输入：约 1 千三角形非索引顶点流（`1.5 万 dword` 级，`03-flat-memory.md` D3.8 口径）。
- 产物：模型轮廓。
- 主判据：单条 draw；非背景像素比例、包围盒像素可复算（L2 参考模型）。
- 回归矩阵：`serial`。
- 引用：章 3/6 DoD。

### E5 `scene_e5_end_to_end` — 端到端最小帧

- 元信息：初级 / 章 9 / 阻断；入口 `apps/scene_e5_end_to_end`。
- 输入：复用 E2/E3/E4（同一帧三条独立 draw）。
- 产物：一帧完整画面。
- 主判据：三条独立 draw 共用同一帧；全图逐字节稳定；两次运行一致（`frame_index` 确定性）。
- 回归矩阵：`serial`；两次运行逐字节一致。
- 引用：章 9 DoD、`09-end-to-end.md`。

---

## 3. 高级场景（S0~S19）

> 高级场景复用初级的画面底板（灰底、同一三角形），使"初级 vs 高级"只差一行配置即可对照。判定：L1 契约断言 + L2 探针点（阻断）+ 全图掩码差分（仅告警）；`serial` / `parallel` 下硬判据必须一致。

### S0 `scene_s0_selftest` — 契约与工具自检（高级）

- 元信息：高级 / 章 11、16 / 阻断；入口 `apps/scene_s0_selftest`。
- 输入：无。
- 主判据：结构与地址分区断言全绿；汇编往返两次二进制相同（L1）。
- 引用：章 11/16 DoD。

### S1 `scene_s1_empty_submit` — 空环提交

- 元信息：高级 / 章 10 / 阻断；入口 `apps/scene_s1_empty_submit`。
- 输入：N 条空命令（N > 环容量）。
- 主判据：完成计数 = N、无丢包、fence 严格递增；环回绕正确、包不跨环尾（NOP 填充）；`cmd_packet_version` 不匹配拒绝（L1）。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 10 DoD、`10-device-ring-cp.md`。

### S2 `scene_s2_clear` — 灰底清屏（高级）

- 元信息：高级 / 章 12、13、34 / 阻断；入口 `apps/scene_s2_clear`。
- 输入：0.25 灰 `0xFF3F3F3F`。
- 产物：全屏 `(63,63,63,255)`。
- 主判据：探针全中；出屏 = 显存；翻转链前后帧可区分（L2）。
- 回归矩阵：`serial` / `parallel`；`cache_enable on/off` 前后一致（12 章 DoD）。
- 引用：章 12/13/34 DoD。

### S3 `scene_s3_triangle` — 渐变三角形（多后端基线）

- 元信息：高级 / 章 12、15、17、18、19、24、25 / 阻断；入口 `apps/scene_s3_triangle`。
- 输入：3 顶点 RGB + 退化三角形（输入与 E2 同底，**永久冻结**）。
- 产物：灰底 + 渐变三角。
- 主判据：结构探针（顶点/覆盖/命令语义）按 S3 列表零容差；颜色探针按 §1.2 的 ±2/通道；手写、编译、LLVM 路径均与 `ref_model` 的规范语义输出一致。IR/ABI/ISA 语义必须等价，不要求三后端生成相同 blob 字节。
- 子 case：`case-0` 手写 ISA；`case-1` 编译产物；`case-2` LLVM 后端产物。
- 回归矩阵：`serial` / `parallel`；`cache_enable on/off`。
- 引用：章 15/17/18/19/24/25 DoD、`validation-model.md`。

### S4 `scene_s4_textured_quad` — 纹理四边形（高级）

- 元信息：高级 / 章 20、21 / 阻断；入口 `apps/scene_s4_textured_quad`。
- 输入：2×2 纹素 + uv + 2 层数组 + cubemap + mip/LOD。
- 产物：四象限色块。
- 主判据：nearest ±1、bilinear ±6、repeat/mirror 非边界像素、LOD 正确；层 / 面索引正确；数组 / Cubemap 越界行为确定（L2）。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 20/21 DoD、`21-texture-sampling.md`。

### S5 `scene_s5_model` — 大网格模型

- 元信息：高级 / 章 14、22、23、24 / 阻断；入口 `apps/scene_s5_model`。
- 输入：OBJ（约 1 万三角形）索引网格（`Kozakura_Mari` 级别）+ 贴图。
- 产物：模型图。
- 主判据：命令数 = 1、非背景像素 > 5%、包围盒、顶点流地址可复算；超显存（逐出）结果一致（L1 + L2）。
- 回归矩阵：`serial` / `parallel`；`memory_budget` 溢出场景。
- 引用：章 14/22/23/24 DoD。

### S6 `scene_s6_tess` — 细分三角

- 元信息：高级 / 章 24、28 / 阻断；入口 `apps/scene_s6_tess`。
- 输入：inner 1/2/4 + GS 开关 + 多视口；三种图元模式 triangle / line / point。
- 主判据：顶点数符合公式、GS 图元数正确、三图元模式、多视口定向；patch 载荷无丢失（本章自建 patch 缓冲、C 阶段不依赖 LDS）；细分因子与模式符合公式；输出域属性完整（L2）。
- 子 case：`case-0` A 路径 HS+DS；`case-1` B 路径无 HS 回退；`case-2` C 路径 HS+DS+GS；每路径 × 三图元模式。
- 历史参考值：256 基线覆盖率 A/B/C = **20808/20808/15138**，1080×720 = **248832/248832/179830**（A/B 覆盖集合一致且 maxDelta≤1，过渡值，以 `ref_model` 为准，Q-78）。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 24/28 DoD、`05-tess-gs.md` §7.8。

### S7 `scene_s7_occlusion` — 遮挡对

- 元信息：高级 / 章 26、27、34 / 阻断；入口 `apps/scene_s7_occlusion`。
- 输入：前后两三角 + 深度。
- 主判据：遮挡正确、early-Z 开关 RT 一致、被遮挡 query = 0、可作 GPU 谓词；覆盖 / resolve、Hi-Z 拒绝计数可输出（L2）。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 26/27/34 DoD。

### S8 `scene_s8_blend_mrt` — 混合与 MRT

- 元信息：高级 / 章 26 / 阻断；入口 `apps/scene_s8_blend_mrt`。
- 输入：固定 alpha + 4 RT + stencil。
- 主判据：混合公式、stencil 掩码、各 RT 独立正确；PS 输出到 UNORM/SNORM/UINT/SINT/sRGB 格式转换与舍入确定；双源混合与逻辑 Op（L2）。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 26 DoD。

### S9 `scene_s9_msaa` — 抗锯齿斜边

- 元信息：高级 / 章 25、27 / 阻断；入口 `apps/scene_s9_msaa`。
- 输入：斜边三角 2x/4x + sample 插值。
- 主判据：多样本归一化覆盖率与参考绝对差 ≤1/16，resolve 按颜色探针比较。Hi-Z 拒绝计数用于观测，per-sample 插值可区分（L2）。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 25/27 DoD。

### S10 `scene_s10_reduce` — 协作归约

- 元信息：高级 / 章 29、30 / 阻断；入口 `apps/scene_s10_reduce`。
- 输入：1024 元素数组。
- 主判据：与 CPU 参考一致；ballot / shuffle / 导数与解析值一致；REDUCE identity 按 op 定义（MIN=+∞ / MAX=−∞）且允许部分活跃（L2）。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 29/30 DoD。

### S11 `scene_s11_parallel` — 并行一致性

- 元信息：高级 / 章 31 / 阻断；入口 `apps/scene_s11_parallel`。
- 输入：复用 S3/S5 参数。
- 主判据：`se_count × worker_threads` 全组合逐字节一致；线程本地 TLB + 全局失效无脏读（L1 + 横切，零容差）。
- 回归矩阵：`se_count ∈ {1,2,4,8}` × `worker_threads ∈ {1,4}`，全组合。
- 引用：章 31 DoD、`roadmap.md` §5.4。

### S12 `scene_s12_multiengine` — 多引擎交错

- 元信息：高级 / 章 32、33 / 阻断；入口 `apps/scene_s12_multiengine`。
- 输入：图形、拷贝及 Compute 家族队列三条流。第 32/33 章的 Compute 流使用受控测试提交验证队列依赖，真实 kernel 的多引擎交互在 S19/C3 接入。
- 主判据：依赖顺序、semaphore 不早于信号、IB / 间接 / predication 与直接提交等价（L1 + L2）。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 32/33 DoD。

### S13 `scene_s13_fault` — 故障注入

- 元信息：高级 / 章 36 / 阻断；入口 `apps/scene_s13_fault`。
- 输入：普通未映射或未驻留、可单次中止的非法包、引擎级故障及 watchdog。覆盖预算内退出、恰好耗尽并退出、耗尽仍运行和合法长派发。
- 主判据：普通页错误失败完成且同队列继续运行。复位类故障仅重编程对应 ring，复位前后读取同一旧提交失败，重建后的 S2 通过。Watchdog 边界按 D36.4。
- 回归矩阵：`serial`；注入点隔离。
- 引用：章 36 DoD、`36-error-paths.md`、`design/boot-reset.md`。

### S14 `scene_s14_bindless_uav` — Bindless + 全阶段 UAV

- 元信息：高级 / 章 20、21、26 / 阻断；入口 `apps/scene_s14_bindless_uav`。
- 输入：描述符堆 + 动态索引 + PS/GS 写 UAV。
- 主判据：索引取到正确描述符；写 UAV 的 draw 禁用 Early-Z；越界确定报错（L1）。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 20/21/26 DoD。

### S15 `scene_s15_mesh_task` — Mesh/Task Shader

- 元信息：高级 / 章 35 / 阻断 + 观测；入口 `apps/scene_s15_mesh_task`。
- 输入：一期只做能力位查询。
- 主判据：**一期**返回"不支持"且不崩溃（L1）；二期能力位为真时跑通同一画面（**KFT，绑定二期 issue**）。
- 判定类型：一期 `case-0` 阻断（能力位断言）+ `case-1..N` 二期 KFT 仅告警。
- 引用：章 35 DoD、`8.3 高级二期`。

### S16 `scene_s16_barrier_ownership` — 屏障 + 所有权转移

- 元信息：高级 / 章 32 / 阻断；入口 `apps/scene_s16_barrier_ownership`。
- 输入：记录期子资源范围校验，以及 Copy→Graphics 整资源移交。构造正常、延迟 release、缺 acquire 与目标错配。
- 主判据：记录期非法范围被拒，执行期 release 前 WAIT 停驻，acquire 前访问被拒。配对错误保持原状态，成功移交后整资源读回精确一致。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 32 DoD。

### S17 `scene_s17_api_equivalence` — API 等价一致性

- 元信息：高级 / 章 35 / 阻断；入口 `apps/scene_s17_api_equivalence`。
- 输入：直连 HLI 与项目内最小图形 UMD（`graphics`）使用同一场景必测。启用的 Vulkan/D3D12/GL 运行 capability 声明子集，未支持能力返回 `NotSupported`。
- 主判据：`hli` 与 `graphics` 的清屏/三角形场景探针一致，图形 UMD 经 HLI 提交；启用的外部 API 适配器按能力范围满足同一语义判据。
- 回归矩阵：`hli` 与 `graphics` 为必测。另运行实际启用适配器的 capability=true 用例。
- 引用：章 35 DoD、`35-api-backend.md`、ADR-022。

### S18 `scene_s18_compute_basic` — 计算最小通路

- 元信息：高级 / 章 29、30、31、33、37（A37 主验收）、38~42（A-stage3 双入口）/ 阻断；入口 `apps/scene_s18_compute_basic`。
- 输入：1D/2D/3D grid + block、kernel 参数、越界 work-item。
- 主判据：blockIdx / threadIdx 映射正确；越界 work-item 不写；grid 边界 workgroup 部分有效；wave 归约与 CPU 参考一致（L2）。
- 子 case：`case-0` 1D；`case-1` 2D；`case-2` 3D + 越界；`case-3` 经计算驱动（stage3 双入口主验收，Q-71）；`case-4..12` 绑定业务类 WA-01~WA-09（元素级 / 归约 / scan / 邻域 / GEMM / 直方图 / 排序 / 图像 / 仿真，**2026-11-20 定案**）。子 case 编号以本节 `case-0..12` 为**唯一权威**；WA 业务分类与判据见 `architecture-design.md` §7.9，**输入规模 / 数据类型 / 种子 / 期望产物以本表为唯一权威**（消除循环引用）。

**S18 子 case 输入规格表（2026-11-20 定案，改值走 §7.4）**：数据生成统一 LCG，seed = `0x5EED`（与 `translation.seed` 机制同源可复现）；比对对象为 `tools/ref_model/compute/` 对应参考 kernel（同 op 序 / tiling / 归约序，`validation-model.md` §7）；容差按 §1.2「计算数值口径」（≤1 ULP 通过、>1 ULP 阻断；同序 FMA 下 0 ULP 是诊断信息，不改变门槛）。

| 子 case | 覆盖 | 输入规模 | 数据类型 / 格式 | 生成规则 | 期望产物（元素数 · 布局 · 比对对象） |
|---|---|---|---|---|---|
| case-0 | 1D 基础 | grid 8 × block 32 | int32 ×2 | LCG 0x5EED | 256 int32 元素（逐字节）+ blockIdx/threadIdx 探针 = ref_model case0 |
| case-1 | 2D 基础 | grid (2,2,1)，block (16,16,1)，有效域 (32,32,1) | int32 ×2 | LCG 0x5EED | 恰写 1 024 个 int32，逐字节一致，二维 ID 与参考一致 |
| case-2 | 3D + 越界 | grid (2,2,3)，block (4,4,3)，有效域 (8,8,8) | int32 ×2 | LCG 0x5EED | 576 个 work-item，恰写 512 个 int32。最后一层 Z workgroup 只写其中两层，越界哨兵保持 |
| case-3 | 经计算驱动 | 与 case-1 复用同一 grid、block、有效域和数据 | int32 ×2 | LCG 0x5EED | A39 首验，A42 比较直连 HLI 的关键命令语义与零容差探针 |
| case-4 | WA-01 | N = 65 536 | int32 ×2 + fp32 ×2 | LCG 0x5EED | 65 536 int32（逐字节）+ 65 536 fp32（≤1 ULP）= ref_model wa01 |
| case-5 | WA-02 | N = 262 144 | fp32 | LCG 0x5EED | 组级 2 048 + 全局 1 fp32（≤1 ULP + 归约顺序断言强制）= ref_model wa02 |
| case-6 | WA-03 | N = 16 384 | uint32 | LCG 0x5EED | 16 384 uint32（逐字节，Blelloch 前缀和）= ref_model wa03 |
| case-7 | WA-04 | 512×512 | int32 | LCG 0x5EED | 512×512 int32（逐字节，边界 clamp）= ref_model wa04 |
| case-8 | WA-05 | M=N=K=256 | fp32 | LCG 0x5EED | 256×256 fp32（≤1 ULP，同序 FMA）= ref_model wa05 |
| case-9 | WA-06 | N = 131 072 | uint8（0..255） | LCG 0x5EED | 256 bin uint32 直方图（逐字节）= ref_model wa06 |
| case-10 | WA-07 | N = 4 096 | int32（含重复键） | LCG 0x5EED | 4 096 int32（逐字节 + 稳定 tie-break）= ref_model wa07 |
| case-11 | WA-08 | 256×256 | RGBA8 UNORM | LCG 0x5EED | 256×256 RGBA8（逐字节，固定舍入）= ref_model wa08 |
| case-12 | WA-09 | 65 536 粒子 | fp32（6 DoF） | LCG 0x5EED | 65 536×2 fp32 位置（≤1 ULP + RNG 消费序断言）= ref_model wa09 |

kernel 参数携带各轴有效域。全局坐标按 `groupId * block + localId` 计算。有效坐标才写 `x + X * (y + Y * z)`，`X/Y` 为有效域宽高。身份探针与存储索引分别比较，case-2 检查有效写入数和越界哨兵。旧的未指定有效域的 S18 输入为 **SUPERSEDED**（ADR-058）。

**D-13 静态可调度性负向样例（37 章 T37.1 / A37 必测，不占 case 号）**：

| 负向样例 | 输入 | 期望 |
|---|---|---|
| D13-a | block 513（= 17 waves > 16 waves/CU 驻留上限，B-037） | 记录期 `InvalidArgument`（D-13），**不产生 `SET_KERNEL_ARGS`/`DISPATCH` 包** |
| D13-b | block 64 + `ldsStaticBytes` = 128 KiB（> 64 KiB/CU，B-036） | 记录期 `InvalidArgument`（D-13），不下发 |
| D13-c | grid 总量 = 2^20 + 1（> `kMaxWorkgroups`，B-038） | 记录期 `InvalidArgument`（D-02 上限），不下发 |

**Demo（C1~C5，`roadmap.md` §7.1/§7.3）输入规格**：

| Demo | 对应 | 输入 | 期望产物 |
|---|---|---|---|
| C1 `module_probe` | — | 1 个 kernel module blob（固定 fixture 字节流） | 符号表文本（name/entryOffset/paramBytes/vgprCount/ldsBytes），两次运行逐字节一致（无数值比对） |
| C2 `vector_add` | = case-4（WA-01） | 同上 | 同上 |
| C3 `async_pipeline` | = S19 case-2（WA-11） | 见 S19 表 | 见 S19 表 |
| C4 `reduce` | case-5（WA-02）及局部边界 | case-5 基线加部分活跃 ADD、MIN/MAX 单波样例 | A41 验参考 ≤1 ULP、归约顺序正确及跨策略硬探针精确一致 |
| C5 `indirect_dispatch` | = S19 case-3（WA-12） | 见 S19 表 | 见 S19 表 |

- 回归矩阵：`serial` / `parallel`。
- 引用：章 29/30/31/33/37 DoD、`design/compute-line.md`、`design/compute-driver.md`。

### S19 `scene_s19_compute_memory` — 计算内存与并发

- 元信息：高级 / 章 30、32、33、37（A37 主验收）、38~42（A-stage3 双入口）/ 阻断；入口 `apps/scene_s19_compute_memory`。
- 输入：LDS / 原子 / barrier + 间接 dispatch + 异步计算。
- 主判据：workgroup/device 作用域符合语义，间接派发与直接等价，跨队列完成 release→SIGNAL→WAIT→acquire 后目标读到生产结果（L2）。
- 子 case：`case-0` 作用域与 barrier 基础；`case-1` 非规则访问 gather/scatter（WA-10）；`case-2` 多 kernel 流水 + async compute（WA-11）；`case-3` 间接 dispatch + 谓词（WA-12）；`case-4` 计算 ↔ 图形互操作（P1-9，**2026-11-20 定案纳入**，ADR-049，WA-08 扩展子 case）。子 case 编号以本节为**唯一权威**。

**S19 子 case 输入规格表（2026-11-20 定案，改值走 §7.4）**：生成规则 / 容差口径同 S18 表（LCG seed `0x5EED`，§1.2 数值口径）。

| 子 case | 覆盖 | 输入规模 | 数据类型 / 格式 | 生成规则 | 期望产物（元素数 · 布局 · 比对对象） |
|---|---|---|---|---|---|
| case-0 | 作用域与 barrier 基础 | grid 16 × block 256（组内 8 wave 跨 wave barrier） | int32 | LCG 0x5EED | 4 096 int32 元素（逐字节）+ barrier 放行前无后续写断言（原子计数）+ workgroup/device 作用域可见性矩阵逐格 |
| case-1 | WA-10 | N = 32 768 | int32 值 + 随机索引 | LCG 0x5EED（索引种子化） | 32 768 int32（逐字节；越界可定位中止）= ref_model wa10 |
| case-2 | WA-11 | 3 stage × 65 536 | int32 + fp32 | LCG 0x5EED | 末 stage 65 536 元素（数值等价 + 队列顺序语义）= ref_model wa11 |
| case-3 | WA-12 | N = 10 242（GPU 侧参数 buffer 生成） | uint32 间接参数（grid/block/paramBufVa） | LCG 0x5EED | 与同参数直接 dispatch 输出**逐字节一致**（ref_model wa12） |
| case-4 | WA-08 扩展 | 256×256 | RGBA8 UNORM | LCG 0x5EED | UAV store 写回 256×256 RGBA8，同 `(x,y,format)` 逐字节一致（对照 20 章 image 描述符 + 26 章格式转换表，ADR-049） |
- 并发语义（P1-7，已澄清口径）："计算队列与图形队列并发" = FW 在包粒度 Round-Robin 交错推进；**硬判据对交错顺序不敏感**，种子化交错顺序下结果不变；同刻并行不在本期。
- **计算 ↔ 图形互操作（ADR-049/055）**：case-4 图形写 RT，经整资源移交后计算 image load，计算通过 UAV store 写回。按第 20 章 image 描述符、第 26 章格式转换表比对 `(x,y,format)` 字节布局。
- 回归矩阵：`serial` / `parallel`。
- 引用：章 30/32/33/37 DoD、`design/compute-driver.md`；stage3 仅作双入口等价验收（`api_backend=compute`，第 42 章主验收）。

---

## 4. 章节验收判据（42 章总表）

计算场景按入口成熟度启用。A37 使用直连 HLI 运行 S18 `case-0~2/4~12` 与 S19 `case-0~4`。A39 首验 S18 `case-3` 和 C2。A42 的双入口矩阵覆盖 S18 `case-0~3` 与 S19 `case-0~4`，S18 `case-4~12` 保留 HLI 回归，各 C 系列 demo 按所属章节保留驱动入口验收。

每章的"场景 × 专有判据"以下表为**唯一权威**；`roadmap.md` §8.2 步骤 3 逐"第 N 行"断言通过；行内 sub-case / 探针 / 容差细节以本文件各场景节（§2/§3）与章文档 DoD 为准，两侧必须对齐：

| 章 | 场景 | 专有判据 |
|---|---|---|
| 1 | E0 | 冻结项清单与代码一致；改动有决策记录 |
| 2 | E1 | 命令不跨环尾；完成计数 = 提交数 |
| 3 | E1、E4 | 同一数据两个句柄读回一致；分配区间不重叠；耗尽返回确定错误 |
| 4 | E2 | 寄存器终态全等；无分支；越界访存报出地址；派发参数按 CU 复用，不随 warp 数按栈放大 |
| 5 | E0、E2 | 汇编往返二进制相同；子集全指令 smoke 无解码失败 |
| 6 | E2、E4 | 共享边不重不漏；退化图元丢弃；深度写回正确 |
| 7 | E3 | nearest 全图逐字节；UV 越界 clamp 行为确定 |
| 8 | E1 | 出屏 = 显存；双缓冲前后帧可区分 |
| 9 | E5 | 三条 draw 共用一帧；两次运行逐字节一致 |
| 10 | S1 + S13 | 环回绕正确、包不跨环尾（NOP 填充）；非法包报类型与偏移；`cmd_packet_version` 不匹配拒绝；fence 语义顺序 |
| 11 | S2 + S0 | 同数据两 VA 读回一致；页表项翻转后原 VA 失效；TLB 命中计数递增 |
| 12 | S2 + S3 + S11 | 缓存/写缓冲数据流与 flush 可见性正确；统计仅作软观测，不以 hit/miss 比例作为通过门槛；`cache_enable` 开关前后一致 |
| 13 | S2 | CopyEngine 填充等于扫描结果，翻转前后整帧分别取旧、新 front |
| 14 | S5 + S11 | 超显存结果一致；逐出前经第 12 章 flush 完成；两上下文互不污染；抢占恢复正确 |
| 15 | S3 | LaneMask 轨迹符合预期；多次调度顺序确定 |
| 16 | S0 | 汇编往返二进制相同；全区段编码/解码覆盖无失败（执行语义由 opcode 所属硬件单元负责：ALU/控制流见第 4/5/15 章，Tess/GS 见第 28 章，subgroup 见第 29 章，LDS/原子/barrier 见第 30 章） |
| 17 | S3 + S0 | IR 快照稳定；前端解析/语义/IO 签名错误与源码位置确定；编译后端执行等价由 18/19 章判定 |
| 18 | S3 | 优化开/关一致；寄存器溢出统计可输出 |
| 19 | S3 | ABI/ISA 合法性与规范程序语义一致（允许合法编码差异） |
| 20 | S4/S14 绑定子集，S5 视图与 S18 参数布局单测 | 资源/视图/快照语义与地址范围，校验调用方已准备的参数缓冲，非法输入无派发包 |
| 21 | S4 + S14 | LOD/sRGB/寻址正确；数组/Cubemap 层索引与越界行为确定；采样器并入 stateId 5 描述符堆、比较采样正确；SAMPLE 请求为 float2 uv + 导数 |
| 22 | S5 | 索引与非索引顶点流一致；primitive restart、step rate 正确；`baseVertex`/`baseInstance` 偏移与 `InstanceID` 正确；内建 ID 只读 SGPR/专用寄存器 |
| 23 | S5 | 命令数恒为 1；顶点数不受单次派发容量限制；切片容量归前端派发器、VGT 只产顶点流；切片边界图元完整；顶点流常驻复用不随帧增长 |
| 24 | S3 + S5 + S6 | 出界/跨近平面与参考裁剪器一致；guard band 无裂缝；退化图元丢弃；多视口索引校验正确；Clip/Cull Distance 逐顶点逐图元剔除正确；背面剔除在 PA 判定、OM 只保存状态；距离区写超声明 count 报 L1 错 |
| 25 | S3 + S9 的 1× 光栅子集 | 覆盖集合精确、共享边无重漏、S3 颜色 ±2/通道，scissor、插值与点线规则正确 |
| 26 | S7 + S8 + S14 | 深度/模板/混合/MRT/剔除正确；LoadOp/StoreOp 正确；双源混合与逻辑 Op 正确；PS 输出到 UNORM/SNORM/UINT/SINT/sRGB 的格式转换与舍入确定；写 UAV 时 Early-Z 禁用 |
| 27 | S9 + S7 | coverage/resolve 正确；early-Z 不改结果；fast clear 元数据与 Hi-Z 拒绝计数可输出 |
| 28 | S6 | patch 载荷无丢失（本章自建 patch 缓冲，C 阶段不依赖 LDS）；三图元模式正确；细分因子与模式符合公式；输出域属性完整 |
| 29 | S10，独立 wave 输入 | ballot/shuffle/归约/导数结果确定，部分活跃时按操作使用单位元 |
| 30 | S10，S19 case-0/1 只作单测输入 | LDS 转置、原子、组内 barrier 和作用域按独立 workgroup 用例验证 |
| 31 | S11 和章内独立映射向量 | 并行一致，TLB 失效无脏读，二维/三维映射及边界确定。正式 S18 输入归场景表 |
| 32 | S12 队列子集、S16 | 三队列等待与失败传播，整资源 release→SIGNAL→WAIT→acquire，延迟 release 和缺 acquire 负向用例 |
| 33 | S12 IB/间接绘制子集，S19 case-3 只作参数单测输入 | IB、间接 draw、predicate 与直接路径对照。间接 dispatch 只对照参数描述符，输出等价归 37/42。验证 DMA 切片拷贝 |
| 34 | S7 + S2 | 遮挡 query = 0；timestamp 单调不减；统计不参与正确性；查询可作 GPU 侧谓词 |
| 35 | S17 + S2 + S3 + S4 + S5 | HLI 主路径和最小图形 UMD 同场景通过，外部适配器按能力声明；动态状态取命令缓冲快照，能力位由 HLI 提供 |
| 36 | S13 | 普通页错误与复位分流、旧提交终态、显式重提、其他引擎持续运行和 watchdog 边界 |
| 37 | S18 case-0~2/4~12、S19 case-0~4 | 直连 HLI 跑通参数消费、uniform、调度/协作/写回和完成值。S18 case-3 首验见 A39 |
| 38 | Demo C1 `module_probe` | `MUSA_MODULE_HDR` 字段与 blob 逐字节一致；符号表解析正确（name/entryOffset/paramBytes/vgprCount/ldsBytes）；Function 入口解析 + 4B 对齐；截断/坏 header/越段返回确定错误（`InvalidIsa`/`NotMapped`）且无泄漏 |
| 39 | S18 case-3、Demo C2 `vector_add` | 最小 Stream 可独立提交、同步和销毁；六条参数检查，参数缓冲存活至终态；两入口关键包等价、探针零容差、fp32 CPU 参考 ≤1 ULP |
| 40 | S19 异步子集、Demo C3 `async_pipeline` | 双 UMD 双队列、有序提交、Event Wait 快照，非阻塞区分 pending/成功/失败；图形在同步和所有权获取后读取，失败依赖可观测 |
| 41 | S18/S19 代表用例、Demo C4/C5 | UMD 输入校验与记录正确，复用执行层探针验证三维/协作、UAV 和间接派发。C4 归约 ≤1 ULP 且顺序断言通过，C5 直接/间接逐值一致 |
| 42 | S18 case-0~3、S19 case-0~4（A-stage3） | 同输入关键命令语义等价，双入口/双执行策略探针零容差。错误映射确定，未达成 Query 返回状态；S18 其余 HLI 用例及适用 E/S 回归 |

- E 场景绑定章节：E0→1/5；E1→2/3/8；E2→4/5/6；E3→7；E4→3/6；E5→9。
- S 场景绑定章节：见各场景节"引用"；**多场景章以本表首列场景为主场景**（主判据），其余为专有判据或章文档 §7 明确登记的回归；一 S 号一入口（Q-67），与 `roadmap.md` §8.2 步骤 3 逐行对齐。
- 功能补强项落位（实例化 / Clip-Cull / PS 输出格式转换 / 描述符堆设备内存语义 / 命令包版本等）权威在 `roadmap.md` §2.8。

## 5. 变更记录

| spec 版本 | 日期 | 变更 | 关联 ADR / 决策 |
|---|---|---|---|
| v1 | 2026-11-19 | 自 `graphics/` 分支参考文档（章节总览 §3.5/§4.5、场景规约 §7.4 与测试用例）收敛为本单份文件，并迁入章节验收判据 36 章总表（§4）（后由 v2/v3 更新为 42 章总表）；数值按 warp32 口径并入；原分支文档归档删除 | Q-38、Q-40、Q-78 |
| v2-审查候选 | 2026-11-20 | §4 补第 37 章验收行并移除行 35 的 `35.5` 计算验收表述（归 37）；§1.2 增「计算数值口径」候选；S18 明确子 case 编号唯一权威并挂 WA 绑定；S19 增并发语义（P1-7）与图形↔计算互操作候选 case（P1-9）。**均为审查修订候选，待按 §7.4 新建 ADR 定稿** | 审查评审 / 待定 ADR |
| v3-定案 | 2026-11-20 | v2 全部候选**定案**（D1~D11）：§1.2 数值口径 fp32 ≤1 ULP 可配置 + 归约顺序断言强制；S18 `case-4..12` / S19 `case-1..3` WA 绑定；S19 并发口径定稿。**P1-9 互操作 case 正式纳入为 S19 `case-4`（ADR-049，UAV store 路径，WA-08 扩展）** | ADR-048/049 + 评审定案（D1~D11） |
| v4-二轮评审 | 2026-11-20 | 二轮评审 13 项全部闭环：S18/S19 补**子 case 输入规格表**（规模/类型/种子 `0x5EED`/期望产物，Demo C1~C5 同表）消除与 `architecture` §7.9 的循环引用；§4 行 39/41 与全库 0 ULP 统一为"判据 ≤1 ULP、同序 FMA 期望 0 ULP 告警不阻断"；行 41 互操作判据改 ADR-049 口径（UAV store，不依赖 OM 探针）；case 编号统一**不补零**；S18/S19 元信息章号补主验收/A-stage3 归属；v1 行"36 章总表"加历史括注 | 二轮评审（处置记录已并入各权威文档，原清单文件已删） |
| v5-规格补全 | 2026-11-20 | 规格表补全（讨论定案）：S18 `case-0..3`（1D/2D/3D+越界/经驱动）与 S19 `case-0`（作用域/barrier）基础 case 补输入规格行；新增 **D-13 静态可调度性负向样例 D13-a/b/c**（超驻留波数/超 LDS/超 `kMaxWorkgroups` → `InvalidArgument` 不下发，37 章 T37.1/A37 必测）；WA 规格数值（GEMM 256³、N 规模、seed `0x5EED`）经确认生效 | 评审确认（原清单文件已删） |
| v6 | 2026-11-26 | 对齐 S3/S9 覆盖、S13 故障分流、S16 整资源移交、S18 有效域和 C4 验收归属 | ADR-055/057/058 |
