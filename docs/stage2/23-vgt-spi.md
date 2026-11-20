# 第 23 章 VGT / SPI

> 本章把"VS 逐波分派"拆为 VGT（几何装配）+ 前端派发器（切片）+ SPI（着色器分派器）三级：命令数恒为 1（不因顶点数拆分命令包）、顶点流自动切波、切片边界图元完整、顶点流缓冲常驻复用不随帧增长（ADR-017`IEngine::Execute` 末参）。切片容量与切片决策归本章前端派发器，VGT 只产顶点流（Q-58）。`IEngine` 握手中 `DRAW` 载荷在 VGT/SPI 处理的语义、切片与 `worker_threads` 解耦在这里定型。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（`IEngine` 末参 `IShaderArray*`）；ADR-017/019（几何前端切片；SE 分区）；`../design/execution-parallel.md` §4（功能级流水线/背压）。
> 落地模块：`hw_sim/frontend/vgt_spi`、`apps/scene_s5_model`（大顶点流）。

---

## 1. 目标与非目标

### 1.1 目标

1. VGT：接收 `DRAW`（1 条命令），只生成顶点流；切片容量与切片决策归本章前端派发器（每切片 ≤ 容量 N 顶点），切片数与 `worker_threads` 解耦（Q-58）。
2. SPI：把每切片派给 VS 波；`Dispatch(active)` 与 IA 输入协作；VS 波在 CU/SIMD 上执行。
3. 命令数恒为 1：`stats.commandCount==1` 即使顶点 100 万仍是一条 DRAW（应用不拆批）。
4. 顶点流缓冲常驻复用：跨帧不重复分配（GART 周期缓冲常驻，红线上 #4）。
5. 切片边界图元完整：不产生"半图元"（块在三角形边界处完整跨越）。
6. 顶点流统计：`{切片数, VS 波数, 顶点总数}` 可观测。

### 1.2 非目标

- 不做 Tess/GS 输入（28 章）；不做 `ES(streamout)`。
- 不做多引擎仲裁（32 章）。
- 不做像素/SE 分区（31 章）；`se_count=1` 逻辑前置。
- 不做波调度的策略 B（31 章）；切片是独立于线程数的正交维度。

---

## 2. 单元框图

```
 DRAW（1 条）──► VGT：图元装配 → 顶点流（只产流）
                    │  前端派发器：切片（boundary 完整，≤Cap verts）→ SPI
                    ▼
                  SPI：Dispatch 到 VS 波（IShaderArray）
                    │  每波执行 ISIMT 波
                    ▼
              VS 输出 v[16..27]/距离 v[68..75] → 下一单元（24 裁剪）
```

- VGT 与 SPI 间以功能级 FIFO + valid/ready 握手（ADR-035）；切片 FIFO 背压反压 VGT 生成。

---

## 3. 输入输出契约

### 3.1 VGT 输入

| 项 | 说明 |
|---|---|
| 包 | `DRAW`（0x03），1 条命令（命令数恒 1） |
| 载荷 | `{primTopology, firstVertex, vertexCount, instanceCount}`（索引/布局来自 22 章绑定） |
| 切片参数（前端派发器） | `vgtSliceCapacity`（默认 256 顶点），每切片 ≈ 整数个 wave；容量归前端派发器，VGT 不参与切片（Q-58） |

- `tessDomain` 为独立枚举（`triangles/quads/isolines`），与 `PrimitiveTopology`（第 22 章）、GS mode（第 28 章）分立并登记映射（Q-63）；本章只透传声明，不改变拓扑族。

### 3.2 切片输出（SPI 输入）

```cpp
struct DrawSlice {
    u32 sliceId;
    u32 firstVertexGlobal;   // 全局偏移
    u32 vtxCount;            // 含边界完整图元
    u32 instance;            // 本切片实例
    Topology topo;
};
// 边界保证：slice 的 vtx 集合涵盖若干完整三角形（不产生半个三角形跨片）
```

- SPI 收到切片后对每 32 顶点（或 slice 剩余）构造波 `Dispatch(mask)`。

### 3.3 SPI 出波

```
Dispatch(activeMask) → IShaderArray 波执行入口（warp32, v[0..15] 已装配, 内建就位）
VS 波数统计 = Σ ceil(slice 顶点/32)
```

- 顶点流缓冲：第一个切片分配一次 GART 周期缓冲；后续切片复用（常驻）。

---

## 4. 内部数据流

### 4.1 前端派发器切片（边界完整；VGT 只产顶点流）

```
DRAW → 展开顶点索引流（on-demand）：
  slice n: 收集顶点使包含完整图元：
    对 triangle：每 3 顶点成图元；片尾若剩余 <1 个完整图元则并入下一片
    （条带/fan 在前端派发器内按拓扑切）
  切片起点与全局 firstVertex 对齐；实例化：每实例视为独立 draw 段（切片在实例边界可断）
```

- 片尾不足一个完整图元的顶点并入下一片（本片只含完整图元，图元不在切片间拆分）；切片由前端派发器执行，VGT 只负责顶点流。

### 4.2 背压

```
VGT 生成切片 → PushVGT(slice)（FIFO 满 → 等待/Busy）
SPI 每波出队 → 分发到 CU/SIMD 波执行
下游（24 裁剪）也通过 FIFO 背压 VGT 不让其一次产出过多
```

- 软观测：切片 FIFO 占用、反压周期计数。

### 4.3 顶点流缓冲复用

```
第一个 DRAW：AllocGpuMem(GART, vertexStreamPool)（常驻）
后续 frame：复用（`streamFrame` 递增标记），不清空重分配
```

- 复用前提是内容被进程内先写完；跨 context 用 fence（14 章）排序。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T23.1 | ① 契约 | `DrawSlice`/`vgtSliceCapacity`/切片边界规则/SPI Dispatch 结构落头文件 |
| T23.2 | ② 主体 | VGT 装配 + 顶点整流化（on-demand 索引流） |
| T23.3 | ② 主体 | 前端派发器切片生成：按容量切、完整图元、实例边界（VGT 只产顶点流） |
| T23.4 | ② 主体 | SPI 分发：切片 → per-wave `Dispatch(mask)`、VS 波数统计 |
| T23.5 | ② 主体 | VGT↔SPI 功能级 FIFO + 背压（ADR-035 软观测） |
| T23.6 | ② 主体 | 顶点流缓冲池：首次分配 + 帧间复用 + `streamFrame` 标记 |
| T23.7 | ② 主体 | 命令数恒 1 断言与超量顶点完整执行 |
| T23.8 | ③ 接入 | `scene_s5_model`（大顶点流：100K 三角形级）接入全链路 |
| T23.9 | ④ 验收 | A23：S5 大顶点流全绿 + 专有判据 + A22 回归 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D23.1 | 接口独立编译；`vgtSliceCapacity` 默认 256 | L1 契约 |
| D23.2 | 顶点流展开：非索引/索引顶点的 on-demand 流正确 | L1 契约 |
| D23.3 | 边界完整：每 slice 顶点集含整数个三角形；跨片的图元不出现 "半三角形"（脚本遍历原语列表验证） | L1 契约 |
| D23.4 | 100K 三角形（300K 顶点）：`stats.commandCount==1`；VS 波数 = Σ ceil(...) 精确 | L1 契约 |
| D23.5 | FIFO 满时 PushVGT 返回 Busy 且切片不丢；排空后全部完成 | L1 契约 |
| D23.6 | 跨 3 帧提交：GART 顶点流缓冲地址不变（未重分配）；`streamFrame` 递增 | L1 契约 |
| D23.7 | 单条 DRAW 全程无额外命令包；切片数与 `worker_threads` 无关 | L1 契约 |
| D23.8 | S5 大顶点流出图 = ref_model；探针覆盖网格中心与边缘 | L2 + A23 |
| D23.9 | A23 全绿：命令数恒 1、顶点数不受单次派发容量限制、切片边界图元完整、顶点流常驻复用不随帧增长 + 回归 | L1 + L2 |

---

## 7. 场景与判据（A23）

**绑定场景：S5 `scene_s5_model`（大顶点流压测）。**

| 项 | 内容 |
|---|---|
| 输入 | S5：100K~1M 顶点非索引/索引大网格；`vgtSliceCapacity` 固定 |
| 产物 | 切片列表、VS 波数、命令包计数、GART 顶点流缓冲 VA 轨迹 |
| 主判据 | S5 大网格出图 = ref_model（中心/边缘探针零容差） |
| 本章专有判据 | ① 命令数恒为 1；② 顶点数不受单次派发容量限制（无 overflow）；③ 切片容量归前端派发器、VGT 只产顶点流；④ 切片边界图元完整；⑤ 顶点流常驻复用不随帧增长 |
| 回归项 | A1~A22 全绿；E4（千三角形基线）不退化 |
| 后续补齐 | SE 分区/tile（31 章）在切片之上再做并行；GS 输入流（28 章）走 patch |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 切片序固定（升序）、边界算法确定、[]内分配时序确定 |
| 时间无关 | 无墙钟；流复用受 fence 逻辑时序 |
| 可观测 | 切片数/VS 波数/命令数、FIFO 占用、`--trace=vgt/spi` |
| 契约影响 | 切片规则/`DrawSlice`/流复用为冻结项（登 `IEngine` 语义） |
| 简化登记 | 单批 VS → VGT 切片 + SPI 分发 |
| 计算兼容 | SPI 分发与计算 dispatch 共用 `IEngine` 握手（`IShaderArray` 不特化） |
| 配置与回退 | `vgtSliceCapacity` 配置；`worker_threads` 只影响谁执行（31 章用） |
| 调试钩子 | `--trace=vgt`（切片一行）/`--trace=spi`（波一行） |
| trace 契约 | 不进入判据 |
| 契约变更 | 切片边界/容量/命令数恒 1 为冻结项；变更走 ADR 回归 S5 |
| 校验归属 | L1（结构/计数）+ L2（S5 输出与 ref_model） |
| 双策略 | 切片粒度与线程数解耦（31 章正式用）；两策略输出一致 |
| 阶段衔接 | 为 24（裁剪消费顶点流）、31（SE 分区）提供明确切片序与边界保证 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 分派 | 单批 → VGT 切波 | — |
| 命令数 | 多命令 → 恒 1 | — |
| 顶点流 | 每帧分配 → 常驻复用 | — |
| 并行 | 串行执行切片 | 31（SE/tile 并行） |

---

## 10. 待定项

- "半图元"处理策略：前端派发器在片尾丢弃不完整图元还是并入下片，取决于 24 章裁剪是否要求图元级连续——本期采用"并入下片直到完整图元"，若有性能副作用延后评估索引流缓冲。
- `vgtSliceCapacity` 数值（256）是否与 31 章 CU occupancy 有耦合，待 ADR-038 落地时统一调优，作为软观测参数并存。
- GS（28 章）输入流的切片语义会在 TS 中复用；先记录互操作需求，不改 slices 结构。
