# 第 11 章 地址翻译 + 堆

> 项目范围、阶段位置和交付以[任务规划](../CPU模拟GPU教学项目_任务规划与实施规格.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把初级"单 VA 恒等映射"替换为真实 GPU 地址模型：两级页表/TLB、VRAM/GART/System 三堆、KMD 内存池、分配种子确定性、以及 flush/invalidate 主机同步。所有硬件单元统一走 `VA → TLB → L2 → 存储`，不得绕过翻译直连堆（ADR-042）。
> 与本章相关的设计说明：`../architecture-design.md` §3.6（`IMemorySubsystem` 接口）；`../design/memory-system.md` §1~§3、§6~§7（三类地址、页表/TLB、堆/分配器、主机同步、分配确定性）；`../design/command-mmio.md` §3（`stateId 1` 页表根）。
> 落地模块：`hw_sim/memory/{translation,heaps}`、`kmd/memory`、`umd/os`、`firmware/scheduler`（doorbell 写页表根）、`apps/scene_s0_selftest`、`apps/scene_s2_clear`。

---

## 1. 目标与非目标

### 1.1 目标

1. 两级页表（PDE→PTE，各 4096 项 × 4 KiB 页 = 64 GiB/AS，硬件有效 VA = 36 bit，B-07/ADR-043，`../design/memory-system.md` §2）与 64 组 × 4 路 = 256 项的组相联 TLB；`Translate` 在未命中时查表填 TLB。
2. 三堆语义：VRAM 空闲链表（4 KiB 对齐）、GART bump + 容量上限不碎片整理、System 仅 host；分配失败 `OutOfMemory` 不改状态（`../design/memory-system.md` §3）。
3. KMD 内存池：段 → 页 → 小对象分层；分配元数据 `{tag, logicalTime, size, va}` 记录，供快照与泄漏检测（§3.1）。
4. 分配确定性：分配器支持配置种子，多次运行 VA 分配序列完全一致（§7）。
5. flush/invalidate：`HOST_VISIBLE`/`HOST_COHERENT` 内存域的 flush（CPU 缓存→GPU）与 invalidate（GPU→CPU）语义落地。
6. 页错误入口：未映射返回 `NotMapped` 带 VA；未驻留返回 `NotResident`；均记录 `{unit,packetType,dwordOffset,va}` 后中止当前提交，**不复位**（ADR-031；细粒度页错误恢复在范围外）。
7. 页错误只做"记录 + 中止提交"入口，不做 wave 级恢复。

### 1.2 非目标

- 不做缓存/写缓冲（第 12 章）；`L2` 仍直通存储；`cache_enable` 统计占位。
- 不做驻留/逐出/launch（第 14 章）；全部内存视为驻留，`IsResident` 查映射表即可。
- 不做 Copy/Display 特化（第 13 章）；拷贝仍走同步内存子系统。
- 不做多上下文 ASID 隔离（第 14 章）；单上下文页表根。
- 不做 compute 专用缓存/带宽；unified 域映射到 GART `HOST_VISIBLE`，不新增第四类堆。

---

## 2. 单元框图

```
 IMemorySubsystem（接口不变，实现替换）
        │ Read/Write/Translate/Map/Unmap/InvalidateTlb/IsResident
        ▼
┌────────────────── hw_sim/memory/translation ──────────────────┐
│ PageTable(PDE→PTE, 4KiB 页)  TLB(64 组 × 4 路 = 256 项)  页错误入口        │
│            │ VA→PA                                          │
┌────────────────── hw_sim/memory/heaps ────────────────────────┐
│ VRAM(空闲链表)  GART(bump, 上限)  System(host)  分配元数据表    │
└───────────────────────────────┬───────────────────────────────┘
                                │
   FW.Doorbell(stateId=1) ──► 页表根 MMIO(0x40+1*2)   KMD.Map/Unmap
                                │
   UMD/App：AllocGpuMem → Map 到给定 VA → Read/Write
```

所有单元（Copy/Display/纹理/着色器）只走 `VA → TLB → L2 → 存储`（见 §3.5）。

---

## 3. 输入输出契约

### 3.1 内存接口（不变，与 §3.6 一致）

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

本章实现语义：

| 方法 | 实现 | 错误 |
|---|---|---|
| `Read/Write` | `Translate(va)→pa` → 查堆 → 读写；`bytes=0` 直接 `Ok` | 未映射 → `NotMapped`+VA；页表缺失 → 页错误入口 |
| `Translate` | TLB 命中直接返回；否则查页表回填 | 缺项 → `NotMapped` |
| `Map` | 分配 PTE、把 `[va,va+bytes)` 映射到 `pa`（跨页自动多 PTE） | 重叠/对已映射区 → `InvalidArgument` |
| `Unmap` | 释放 PTE，标记 tlb 需要失效 | 未映射 → `InvalidArgument` |
| `InvalidateTlb` | 全清 TLB（本章无需按 VA 精确失效） | — |
| `IsResident` | 只查映射表（驻留面在第 14 章） | — |

### 3.2 页表与 TLB

- 页 4 KiB；两级各 4096 项（硬件有效 VA = 36 bit，`../design/memory-system.md` §2，B-07）：PDE 索引 = `(va >> 24) & 0xFFF`，PTE 索引 = `(va >> 12) & 0xFFF`。
- TLB：64 组 × 4 路 = 256 项，4 路组相联；组索引 = `(va >> 12) & 0x3F`（tag = va>>12 高位）；LRU 替换。
- TLB 命中计数、缺失计数经 MMIO `0x28` 段软观测（不阻断开销）。

### 3.3 三堆分配器（`../design/memory-system.md` §3）

| 堆 | `AllocDesc.heap` | 分配策略 | 失败 | 对齐 |
|---|---|---|---|---|
| VRAM | `Heap::Vram` | 空闲链表，返回首个满足块 | `OutOfMemory`（不带状态变更） | 4 KiB |
| GART | `Heap::Gart` | bump，容量上限常量；**不做碎片整理** | `OutOfMemory` | 4 KiB |
| System | `Heap::System` | KMD 主机侧分配 | `OutOfMemory` | 自然对齐 |

- `AllocDesc{ heap, bytes, alignment, tag, seed? }`；`AllocResult{ va, bytes }`。
- GART 周期性缓冲按上下文常驻复用，不每帧分配（工程红线 #4）。

### 3.4 KMD 内存池（`../design/memory-system.md` §3.1）

- `kmd/memory` 维护 GPU 地址空间预算与页粒度管理（段 → 页 → 小对象）。
- 大分配走堆管理器；小对象走对象池；分配元数据 `{tag, logicalTime, size, va}` 记录。
- 接口：`AllocGpuMem/FreeGpuMem`（DDI 已有）；新增 `SetMemoryPoolSeed(u64)` 供确定性测试。

### 3.5 内存域与主机同步（`../design/memory-system.md` §6）

| 域 | `MapFlags` | 语义 |
|---|---|---|
| `HOST_VISIBLE` | `MapFlags::HostVisible` | host 可映射、可写 |
| `HOST_COHERENT` | `MapFlags::HostCoherent` | 无显式 flush 也可读（本章模拟即刻一致） |
| 非 coherent | 缺省 | 需第 12 章 `CacheSubsystem` 的 flush（CPU→GPU）、invalidate（GPU→CPU）显式同步 |

- flush/invalidate 归第 12 章 `CacheSubsystem`，不在 `IMemorySubsystem` 上新增 `Flush/Invalidate`；主接口 7 方法冻结（Q-44）。
- `Unmap` 前默认不自动 flush（调用方责任）。

### 3.6 包 / MMIO 触点

| 触点 | 说明 |
|---|---|
| `SET_STATE_BASE(stateId=1, rootVa)` | KMD 在 `MemoryInit` 时设置自身页表根；FW doorbell 时写入硬件页表根寄存器（`0x40+1*2` 段） |
| TLB 强制失效 | `InvalidateTlb`：`Map/Unmap` 后、每次上下文切换前（第 14 章） |
| 页错误 | 缺失 PTE → `NotMapped` 记录 VA；未驻留 → `NotResident`；记录 `{unit,packetType,dwordOffset,va}` 后中止当前提交（ADR-031：不复位） |

> ⚠️ stateId 编号纪律（三档，`../design/bit-budget.md` §3.1）：①`0~7` 直接 MMIO 映射（`0x40+2n`）冻结、兼容 E0~E5；②`0x50~0x7F`（stateId 8~31）只是**临时过渡缓冲**，不得作为长期设计；一旦耗尽必须切换为③GPU 内存间接状态表（`SET_STATE_TABLE_INDEX` 索引内存数组，MMIO 只存根指针）。**禁止继续在 MMIO 硬编码新增 stateId**；本章只用 `stateId=1`，不占临时槽。

---

## 4. 内部数据流

### 4.1 Map → Translate → Read/Write

```
Map(va, pa, bytes): 对齐检查 → 逐页建 PDE/PTE → TLB 标记 invalid → 记录元数据
Translate(va): TLB 命中 → 返回 pa（命中计数++）
              TLB 缺失 → 逐级查页表 → 命中孤缺? → 填 TLB（缺失计数++）→ 返回 pa
                           → 缺项 → NotMapped（页错误入口）
Read(va): Translate → pa → 定位 heap（按 pa 区间归属）→ 读存储
Write(va): 同 Read 路径 + 写存储
```

### 4.2 Unmap 与 TLB 一致性

```
Unmap(va, bytes): 校验已映射 → 释放 PTE → InvalidateTlb（全清，本章粒度）
验证：Unmap 后 Translate(va) 必须返回 NotMapped（原文访存不得读到旧物理页内容）。
```

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T11.1 | ① 契约 | 页表/TLB/三堆/`MapFlags`/`Heap` 枚举/`AllocDesc` 扩展落头文件；flush/invalidate 归第 12 章 `CacheSubsystem`，`IMemorySubsystem` 7 方法冻结 |
| T11.2 | ② 主体 | 两级页表 + `Translate`：建表、缺项、页级映射 |
| T11.3 | ② 主体 | TLB：64 组 × 4 路 = 256 项、组索引 `(va>>12)&0x3F`、LRU、命中/缺失计数、`InvalidateTlb` |
| T11.4 | ② 主体 | VRAM 堆空闲链表分配/释放；碎片拒绝策略 |
| T11.5 | ② 主体 | GART bump + 容量上限；bump 不回退 |
| T11.6 | ② 主体 | KMD 内存池：页粒度分层、小对象池、元数据记录 |
| T11.7 | ② 主体 | `Map/Unmap` 跨页建表/释放、重叠拒绝、TLB 失效联动 |
| T11.8 | ② 主体 | 页错误入口：`NotMapped`/`NotResident` + VA、记录 `{unit,packetType,dwordOffset,va}` 后中止提交（不复位） |
| T11.9 | ② 主体 | 内存域语义：`HOST_VISIBLE/HOST_COHERENT`；flush/invalidate 由第 12 章 `CacheSubsystem` 接入 |
| T11.10 | ② 主体 | 分配种子确定性：`SetMemoryPoolSeed` 后 VA 序列可复现 |
| T11.11 | ② 主体 | 分配元数据快照 dump + 泄漏检测接口（供 `mem_audit`） |
| T11.12 | ② 主体 | `SET_STATE_BASE(stateId=1)` + FW doorbell 页表根与 MMIO 回读 |
| T11.13 | ② 主体 | `IMemorySubsystem` 替换实现：全部方法走翻译、不直连堆；单元调用点不改 |
| T11.14 | ③ 接入 | `scene_s2_clear` 全流程：上传→映射→读回；`scene_s0` 自检翻译/TLB |
| T11.15 | ④ 验收 | A11：S2/S0 全绿 + 本章专有判据（同数据两 VA 一致、翻页失效、TLB 命中递增）+ 回归 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M11.1 | T11.1~T11.3 | 页表/TLB/Translate 契约与主体，D11.1~D11.3 全绿 |
| M11.2 | T11.4~T11.10 | 三堆/内存池/Map/页错误/种子，D11.4~D11.10 全绿 |
| M11.3 | T11.11~T11.15 | 元数据/state base/实现替换/接入验收，A11 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D11.1 | 头文件独立编译；页粒度/PDE/PTE 层数常量 `static_assert`；`MapFlags/Heap` 取值冻结 | L1 契约 |
| D11.2 | 随机 1000 组 `Map(随机 va)` 后 `Translate` 全部命中且 pa 与输入一致；未映射 `va` → `NotMapped` | L1 契约 |
| D11.3 | TLB 容量 256 项（64 组 × 4 路）；命中计数仅增不减；`InvalidateTlb` 后首次访问缺失计数 +1 | L1 契约 |
| D11.4 | 连续 500 次 VRAM 分配区间两两不重叠；`Free` 后复用首块；OOM 不做部分分配 | L1 契约 |
| D11.5 | GART 分配单调；达到 `gartCapacity` 后 `OutOfMemory`；bump 地址不复用已释放段 | L1 契约 |
| D11.6 | 内存池元数据 `{size, tag}` 与分配一致；小对象复用不产生泄漏计数净增长 | L1 契约 |
| D11.7 | `Map` 重叠区 → `InvalidArgument` 且两页不被破坏；`Unmap` 后 `Translate` 返回 `NotMapped` | L1 契约 |
| D11.8 | 访问缺项 VA → `NotMapped` 且 `ErrorRecord.va` 精确；未驻留 → `NotResident`；当前提交中止且**不复位**，修正映射后重跑结果一致 | L1 契约 |
| D11.9 | 第 12 章 `CacheSubsystem` 的 flush 后 GPU 读到 CPU 最新值；invalidate 后 CPU 读到 GPU 新值；两者为空操作时语义仍成立（模拟即刻一致） | L1 契约 |
| D11.10 | 相同输入 + 相同 seed：两次运行 100 次分配 VA 序列完全一致；不同 seed 生成不同序列 | L1 契约 |
| D11.11 | 分配元数据快照可导出；模拟"分配后释放再引用"被 `mem_audit` 标记 UAF | L1 契约 |
| D11.12 | `SET_STATE_BASE(stateId=1, root)` 后 MMIO `0x42/0x43` 回读等于 root 高/低 32 bit；doorbell 后 `CTX_STATE` 更新 | L1 契约 |
| D11.13 | 新旧实现下所有 `Read/Write` 调用点（着色器/Copy/Display/纹理）结果一致；代码搜索无直接 `Storage` 访问。**新增 L1 契约单测（正常+非法边界，部分失败语义）**：多页 `Write` 中一页未映射 → 整体 `NotMapped` 且无半写副作用（已映射页不被部分修改） | L1 契约 |
| D11.14 | `scene_s2` 上传 N 顶点、映射读回逐字节相等；`scene_s0` 自检 `Translate`/TLB 命中无误 | L1 契约 |
| D11.15 | A11 全绿：S2 主判据 + 专有判据（两 VA 一致、翻页失效、TLB 递增）+ S0 自检 + E0~E9 A1~A9 回归 | L1 契约 + L2 |

---

## 7. 场景与判据（A11）

**绑定场景：S2 `scene_s2_clear`（地址翻译为主）、S0 `scene_s0_selftest`（翻译/TLB 自检）。**

| 项 | 内容 |
|---|---|
| 输入 | S2：两个不同 VA 映射到同一内容的两块物理内存、页表项翻转（`va→pa1` 改 `va→pa2`）、随机 `Map` 序列；S0：自检指令 `Translate`/TLB |
| 产物 | 两张读回缓冲、TLB 命中/缺失计数、页表快照 |
| 主判据 | S2 全流程 `Download` 与上传数据逐字节一致；S0 自检全绿 |
| 本章专有判据 | ① 同数据两 VA（va1/va2 指向同一内容）读回一致；② 页表项翻转（va→pa1 改 va→pa2）后原 pa1 读回不再可见、新 pa2 完全可见；③ TLB 命中计数严格递增（重复访问同页） |
| 回归项 | A1~A10 全绿；E 场景不进新映射仍以旧输出一致 |
| 后续补齐 | 缓存命中/失效统计在第 12 章；驻留逐出在第 14 章；多上下文 ASID 页表切换在第 14 章 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 分配 VA 序列由 seed 与调用顺序唯一决定；页表遍历顺序固定（PDE→PTE 升序） |
| 时间无关 | 无墙钟/随机；页错误是逻辑事件，记录后中止提交，不产生 `ResetRequired`，不依赖时钟 |
| 可观测 | TLB 命中/缺失计数、页表快照 dump、分配元数据、`--trace=memory` |
| 契约影响 | `IMemorySubsystem` 签名不变（7 方法冻结），实现替换；新增 `Heap`、`AllocDesc.seed`；flush/invalidate 归第 12 章 |
| 简化登记 | 恒等映射 → 两级页表；bump → 三堆 + 内存池；coherent → flush/invalidate |
| 计算兼容 | VA/TLB 不区分图形/计算堆；unified 域 → GART `HOST_VISIBLE`；`DISPATCH` 参数区走 GART（30/37，原 35.5） |
| 配置与回退 | `mem_sanitizer=on` 起用（越界/UAF 检查关闭默认）；`translation.seed` 配置键 |
| 调试钩子 | `--trace=translation`：每次 `Translate{va,pa,hit}` 一行；`--dump-page-table` |
| trace 契约 | 不进入判据 |
| 契约变更 | 页粒度/层数/堆语义/页错误行为为冻结项；变更走 ADR 并回归 S2/S0 |
| 校验归属 | L1（结构/状态机）+ L2（`ref_model` 对随机 Map/读回做解析期望） |
| 双策略 | `Translate` 并发安全（读不变量），`InvalidateTlb` 在并行下原子；第 31 章线程本地 TLB 依赖本章计数语义 |
| 阶段衔接 | A 阶段门禁 2（地址模型/KMD-FW 接口冻结）在此章达成；S1 空环继续可跑 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 地址模型 | 恒等 → 两级页表 + TLB + 页错误入口 | —（定案） |
| 堆 | 单一 bump → VRAM/GART/System + KMD 内存池 | —（定案；容量可配置） |
| 主机同步 | coherent → flush/invalidate 语义 | —（定案） |
| Coherence | TLB 失效粒度全局（全清） | 31（线程本地 TLB + 全局失效序列） |
| 驻留 | `IsResident` 只查映射表 | 14（真驻留/逐出） |

---

## 10. 待定项

- flush/invalidate 由第 12 章 `CacheSubsystem` 提供，本章只声明内存域语义；若引入写缓冲，第 12 章 flush 必须为真实屏障工具。登记到待定项，不阻塞 S2。
- GART 容量上限的默认值（本章取测试友好值）与真实硬件对齐留到第 14 章驻留模型定案时统一。
- `HOST_COHERENT` 的"即刻一致"是模拟近似；若后期要摸真实 GPU 的 flush 语义差异（页错误恢复），以新 ADR 记录。
