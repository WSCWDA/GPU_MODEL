# 第 28 章 Tess + GS

> 本章补齐几何阶段的曲面细分（HS → Tessellator → DS）与几何着色器（GS）。patch 载荷经专用 stage-local buffer 跨 HS→DS 传递，v1 不并入 workgroup LDS（Q-59）；HS 按 `factor[0..7]` ABI 输出 TessFactor，GS 可 emit/restart 生成多图元并改变图元模式。第 28 章是 C 阶段收口章（S6 接入验收）。
> 与本章相关的设计说明：Q-59（patch payload 使用专用 stage-local buffer，v1 不并入第 30 章 LDS）；ADR-034（发散分析）、ADR-033（距离区）、ADR-050（细分因子与域公式）；`../design/isa-abi.md` §3（HS 因子区布局）；opcode 区段 0x60~0x6F（曲面/几何）。
> 落地模块：`hw_sim/geometry/{hs,tessellator,ds,gs}`、`apps/scene_s6_tess`（含 GS 子 case）。

---

## 1. 目标与非目标

### 1.1 目标

1. patch 化：`DRAW` 支持 `patchTopology` + `patchControlPoints`（如 3）；IA 以 patch 为单位装配。
2. HS：输入 patch 控制点 → 输出 `factor[0..7]`（按 TessDomain 解释）+ 可选 patch 数据（经专用 stage-local patch buffer）。
3. Tessellator：按三图元模式（`triangles/quads/isolines`）与因子计算细分网格；顶点带重心/参数坐标（经过 DS）。
4. DS：输入每个细分域顶点 → 按 patch 与域坐标求值（∟坐标/quad/line 插值）→ 输出位置/属性。
5. GS：输入图元（点/线/三角形/patch 域），可 `EMIT` 顶点、`RESTART` 条带；输出图元可改变原语模式（三图元模式输入/输出声明）。
6. patch 载荷不丢失：HS→DS 经自建 patch 缓冲传全部 patch 属性，C 阶段不依赖 LDS（Q-59），越界不丢失。
7. 输出域属性完整：DS 输出位置 + 属性与后续裁剪/光栅 ABI 匹配。

### 1.2 非目标

- 不做硬件固定的细分内核并行/阶段分离优化（本期同步串行）。
- 不做动态细分因子反馈环（不支持基于前一帧的自动细分）。
- 不做 GS 的多重输入拓扑混合（单拓扑每 dispatch）。
- 不做 Tess 的 quads/isolines 之外的 TessDomain（范围外能力位）。

---

## 2. 单元框图

```
 IA(patch) ──► HS（每 patch 波）──► Tessellator（固定功能）──► DS（每域顶点波）
     patch 载荷（自建 patch 缓冲，Q-59）                 │
       └────────────► (HS patch 属性) ────────────────────┤
                                                        ▼
                                              GS（可选）emit/restart
                                                        │
                                                        ▼
                                   裁剪/光栅（24/25 章）
```

- 自建 patch 缓冲是 patch 载荷的唯一通道（不写全局内存、C 阶段不依赖 LDS）；HS→DS 的域坐标由 Tessellator 产生。
- 三图元模式：Tessellator 输出域的 bary/uv 由 DS 消费。

---

## 3. 输入输出契约

### 3.1 DRAW patch 化

```
DRAW { topology=patch(3) , patchControlPoints=3 }
IA → 每 3 控制点 = 1 patch → 送 HS
```

- `patchControlPoints` ≤ 32；`patchTopology` 决定 Tessellator 模式（tri/quads/iso）。

### 3.2 HS 输入/输出

```cpp
struct HsOutput {
    Vec4 tessInnerOuter[2];  // {inner} + {e0,e1,e2} → v[28..35]
    u32  patchDataSize;      // patch 数据经自建 patch 缓冲传递（Q-59）
};
```

- ABI：`v[28..35]` 固定 store inner/e0e1e2（ABI 预留）。
- HS 每 patch 执行 1 次；patch 控制点经自建 patch 缓冲写入再由 DS 读（C 阶段不依赖 LDS）。

### 3.3 Tessellator 内核

```
输入：TessFactor(inner[1], outer[3])，TessMode(tri/quads/iso)，LOD 相关
输出：域顶点列表 + 每顶点域坐标（bary 或 uv），用于 DS
因子输入必须为有限正数，先 clamp 到 [1,16] 再 ceil 得整数分段数；triangles 使用 N=ceil(max(inner0,outer0,outer1,outer2))，顶点数=(N+1)(N+2)/2；quads 使用 Nu=ceil(max(inner0,outer1,outer3))、Nv=ceil(max(inner1,outer0,outer2))，顶点数=(Nu+1)(Nv+1)；isolines 使用 lineCount=ceil(outer0)、segments=ceil(outer1)，顶点数=lineCount×(segments+1)。非法 NaN/Inf/非正因子返回 InvalidArgument。位置由 DS 求值。
```

- 模式判定与因子到细分步长唯一按上表；factor 先 clamp 再 ceil，`ref_model` 与实现共用同一组整数分段公式。
- 每域顶点一个 DS 波；域坐标确定性由内核次序定。

### 3.4 DS

```
DS 输入：域坐标 (u,v,w) + patch 数据(自建 patch 缓冲) → 输出 位置+属性 → v[16..27]（进入后续）
```

- `TSETUP/EVAL` opcode：DS 内属性求值指令。

### 3.5 GS

```cpp
struct GsEmitImpl {
    void EmitVertex(const Vec4& pos);   // 累加输出
    void RestartStrip();                // 断条带
    u32  PrimitiveModeIn, PrimitiveModeOut;
};
```

- GS 输入图元（来自 Domain 或 VS），输出图元流；每图元一个 GS wave（或每 batch）。
- `PrimitiveModeIn/Out` 使用独立 GS mode 枚举，与 `PrimitiveTopology`（22 章）、`tessDomain`（23 章）分立并登记映射（Q-63）。
- `EMIT/RESTART` opcode（0x60~0x6F 区段，7 bit 重编号）。

---

## 4. 内部数据流

### 4.1 管线次序

```
IA→ patch → HS wave → 记 patchData（自建 patch 缓冲）+ tessFactor
Tessellator：生成域网格（顶点点位/域坐标）
DS wave：每域顶点求值（读自建 patch 缓冲）→ 位置/属性
（若有 GS）GS wave：处理输入的图元 → emit → 输出图元
→ 裁剪/光栅
```

- patch 载荷在自建 patch 缓冲常驻，直到 DS 读取完成（C 阶段不依赖 LDS，Q-59）。

### 4.2 三图元模式的表

| 模式 | `factor[0..7]` 有效字段 | 分段数/域坐标 |
|---|---|---|
| Triangles | `inner0`, `outer0..2` | `N=ceil(clamp(max(valid factors),1,16))`；barycentric grid，顶点数 `(N+1)(N+2)/2` |
| Quads | `inner0..1`, `outer0..3` | `Nu=ceil(clamp(max(inner0,outer1,outer3),1,16))`；`Nv=ceil(clamp(max(inner1,outer0,outer2),1,16))`；`(u,v)` grid，顶点数 `(Nu+1)(Nv+1)` |
| Isolines | `outer0=lineCount`, `outer1=segments` | 两者各自 clamp+ceil；顶点数 `lineCount×(segments+1)` |

- 每模式细分网格顶点数可通过 `factor` 表达式重推（可断言）。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T28.1 | ① 契约 | `HsOutput`/TessMode/本机 `EMIT/RESTART/EVAL`/patch 载荷结构落头文件 |
| T28.2 | ② 主体 | IA patch 化：patch 控制点装配 |
| T28.3 | ② 主体 | HS：因子计算 + patchData 写自建 patch 缓冲 + `v[28..35]` 输出 |
| T28.4 | ② 主体 | Tessellator 内核：tri/quads/iso 域生成 |
| T28.5 | ② 主体 | 因子到细分步长公式与边界（factor=1 → 细分数=1） |
| T28.6 | ② 主体 | DS：域坐标求值（EVAL）→ 位置/属性输出 |
| T28.7 | ② 主体 | patch 载荷经自建 patch 缓冲无丢失（HS 写 / DS 读完整，C 阶段不依赖 LDS） |
| T28.8 | ② 主体 | GS：输入图元 + EMIT 顶点聚合 |
| T28.9 | ② 主体 | GS：RESTART 条带 + 图元模式转换（ModeIn→ModeOut） |
| T28.10 | ② 主体 | GS 输出 ABI：v[16..27] + 距离区兼容 |
| T28.11 | ② 主体 | 与 VGT/SPI 切片集成（GS 输出也切波） |
| T28.12 | ② 主体 | 统计：patch 数、域顶点数、GS 顶点数、图元模式 |
| T28.13 | ② 主体 | 与 `ref_model` 的 tess/gs 模块对照（细分网格坐标） |
| T28.14 | ② 主体 | 切片边界 patch/DS 完整性回归（并入 23 章切片语义） |
| T28.15 | ③ 接入 | `scene_s6_tess` 接入（细分子 case + GS 子 case） |
| T28.16 | ④ 验收 | A28：S6 全绿 + 专有判据 + R28 回归项盘点 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M28.1 | T28.1~T28.7 | 独立验证 patch、HS 因子、细分坐标、DS 输出与载荷完整性 |
| M28.2 | T28.8~T28.10 | 以构造图元验证 GS 的 EMIT、RESTART 与输出 ABI |
| M28.3 | T28.11~T28.16 | 切片集成/统计/ref 对照/接入验收，A28 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D28.1 | 接口独立编译；`patchControlPoints≤32`、因子区 `v[28..35]` 校验 | L1 契约 |
| D28.2 | patch 化：3 控制点=1 patch；非 patch 拓扑不误入 | L1 契约 |
| D28.3 | HS 每 patch 1 次；因子写 ABI；`patchDataSize` 与写一致 | L1 契约 |
| D28.4 | Tessellator：tri/quads/iso 网格顶点与 `ref_model` 公式一致（域坐标逐点） | L2 解析 |
| D28.5 | tri/quad/isoline 的 factor=1、非整数 factor、factor=16、非法 NaN/Inf/非正输入，域顶点数和序列逐项符合 §4.2 公式；非法输入为 `InvalidArgument` | L2 解析 |
| D28.6 | DS：`(u,v,w)` → 位置按 patch 求值 = 参考；属性插值正确 | L2 解析 |
| D28.7 | patch 载荷无丢失：HS 写 200 项 patch 属性，DS 读回全部一致（自建 patch 缓冲，C 阶段不依赖 LDS） | L1 契约 |
| D28.8 | GS 每图元一次；EMIT 输出顶点序正确 | L1 契约 |
| D28.9 | RESTART 断条带；ModeIn=Tris → ModeOut=Line/Point 转换正确 | L1 契约 |
| D28.10 | GS 输出 ABI：位置入 `v[16..27]`、距离区可用 | L1 契约 |
| D28.11 | GS 输出切片与 VGT 切片同序 | L1 契约 |
| D28.12 | patch/域顶点/GS 顶点数统计两次运行一致 | L1 契约 |
| D28.13 | 20 细分 case 与 `ref_model` 对照通过 | L2 解析 |
| D28.14 | patch draw 切片容量为 patchControlPoints 整数倍，patch 不跨片；HS 输出与对应 DS 输入一一配对，无半 patch | L1 契约 |
| D28.15 | S6 出图 = ref_model（细分网格/GS 多点探针） | L2 + A28 |
| D28.16 | A28 全绿：patch 载荷无丢失（自建 patch 缓冲，C 阶段不依赖 LDS）、三图元模式正确、细分因子与模式符合公式、输出域属性完整 + R28 回归项 | L1 + L2 |

---

## 7. 场景与判据（A28）

**绑定场景：S6 `scene_s6_tess`（细分子 case + GS 子 case，见 `scene-specs.md` §3 S6 的 `case-2`）。**

| 项 | 内容 |
|---|---|
| 输入 | S6 `case-0/1`（细分）：某曲面先平细（factor=1）再细分（factor=4/8）；S6 `case-2`（GS）：三角形进、点/线出 + restart |
| 产物 | 域网格坐标、patch 载荷快照、GS 输出图元、RT |
| 主判据 | S6 细分/GS 输出 = ref_model（含细分网格与块点） |
| 本章专有判据 | ① patch 载荷无丢失（自建 patch 缓冲全程，C 阶段不依赖 LDS）；② 三图元模式（tri/quads/iso）正确；③ 细分因子与模式符合公式（factor→细分数可再推）；④ 输出域属性完整；⑤ GS mode 独立枚举与映射正确 |
| 回归项 | R28：E2/E4（非 patch 基线）不退化；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 动态细分范围外；GS 与 31 章 tile 交互由 31 章 §10 复核 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 域顶点生成序、因子公式、GS emit 序固定 |
| 时间无关 | 无时间输入 |
| 可观测 | patch/顶点/GS 计数、自建 patch 缓冲占用、`--trace=hs/ds/gs/tess` |
| 契约影响 | patch 载荷（自建 patch 缓冲，C 阶段不依赖 LDS）、因子 ABI、三图元模式、GS mode 枚举、EMIT/RESTART 语义冻结 |
| 简化登记 | 无 Tess/GS → 完整几何阶段 + 自建 patch 缓冲 |
| 计算兼容 | geometry 阶段图形专用（明确允许）；自建 patch 缓冲不与 LDS 耦合，LDS 技术（第 30 章）独立落地 |
| 配置与回退 | `patchControlPoints` 来自 DRAW 载荷；TessMode 由 PSO 声明；`hs/ds/gs` stage 为 PSO 阶段标志（均非全局配置键） |
| 调试钩子 | `--trace=gs`（emit/restart 一行）；`--trace=tess`（网格一行） |
| trace 契约 | 不进入判据 |
| 契约变更 | patch 载荷/因子/模式/EMIT/RESTART 为冻结项；变更走 ADR 回归 S6 |
| 校验归属 | L1（结构/计数/自建 patch 缓冲完整性）+ L2（细分坐标与 ref_model） |
| 双策略 | 几何阶段与并行策略无关；细分网格在 `parallel` 下输出一致 |
| 阶段衔接 | C 阶段收口：S3~S9、S14 全绿进 D 阶段（寄存器在 29~31） |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 几何 | 无 Tess/GS → HS/Tessellator/DS/GS | —（定案） |
| patch 数据 | 无 → 自建 patch 缓冲传属性（C 阶段不依赖 LDS） | — |
| GS | 只 VS | EMIT/RESTART + 模式转换 | — |
| 三图元模式 | 仅 Triangle | + Quads + Isolines | — |

---

## 10. 待定项

- Tessellator 的细分公式与 `ref_model` 的精确对齐在首次校验时可能遇到 ULP 差异（`ceil` vs `round`）；以 `ref_model` 为唯一期望，若不一致（±1 细分点）先修内核实现。
- GS 每图元一 wave 的容量与切片策略在 `patchControlPoints` 较大（>16）时可能触碰 23 章 slice 上限——以配置调高切片容量，正确性不受影响（软观测）。
- `quads/isolines` 的纵深探索在 S6 先用最小集，若后续场景需要更复杂（如 tess LOD 带导数），登记到能力位，不扩实现面。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A28 和适用的 R28 后提交。分配核对项包括HS 因子位槽、Tess/GS opcode 与暂存容量。变更按已有分配流程登记。

```text
feat(gpu-ch28): tessellate patches and emit GS primitives

- 分开实现 patch 因子、域求值和 GS 输出缓存。
- 用 S6 对照图元计数、顺序、旁路与超预算行为。

Tasks: T28.1–T28.16
Validation: <实际执行的命令、用例和结果>
```
