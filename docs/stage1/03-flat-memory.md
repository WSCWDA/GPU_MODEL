# 第 3 章 平坦内存

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章建立全项目唯一的内存访问通道：单一虚拟地址空间、恒等映射、bump 分配器、资源上传与读回，以及 `SET_STATE_BASE`/`CLEAR`/`COPY_MEM` 三个与内存直接相关的命令包。所有硬件单元此后只能经 `IMemorySubsystem` 访问存储，不得直接解引用主机指针。
> 与本章相关的设计说明：`../architecture-design.md` §3.6（内存接口）；`../design/memory-system.md` §1~§3、§6、§7（三类地址、堆、主机同步、分配确定性）；`../design/command-mmio.md` §2~§4（`SET_STATE_BASE`/`CLEAR`/`COPY_MEM`、`stateId`、MMIO 段）。
> 落地模块：`hw_sim/memory/flat`、`hw_sim/core/{cp,mmio}` 追加分支、`hw_sim/submit/facade`、`apps/scene_e1_clear_present`、`apps/scene_e4_mesh`（本章只跑资源上传/读回子集）。

---

## 1. 目标与非目标

### 1.1 目标

1. 单一 VA 空间：`GpuVAddr` 64 bit，页 4 KiB 对齐；本章 `Translate` 为恒等映射，但接口与错误语义按最终形态定义。
2. `IMemorySubsystem` 七个方法全部落地：`Read/Write/Translate/Map/Unmap/InvalidateTlb/IsResident`。
3. bump 分配器：顺序分配、不回收、不重叠；耗尽返回 `OutOfMemory`；分配序列可复现。
4. 资源生命周期：上传（host→GPU）、读回（GPU→host）、同一数据多句柄读取一致。
5. `SET_STATE_BASE` 写入 `stateId 0`（RT 基址）与 `stateId 1`（页表基址，本章恒等）；
6. `CLEAR` 包：锁存 RT 尺寸到 MMIO `0x30` 段并按 BGRA 填充；
7. `COPY_MEM` 包：块拷贝，经由内存子系统执行（第 13 章迁到独立 Copy 引擎，包格式不变）。

### 1.2 非目标

- 不做页表/TLB/页错误（第 11 章）；不做 L1/L2 缓存与写缓冲（第 12 章）。
- 不做 VRAM/GART/System 三堆与驻留（第 11、14 章）。
- 不做独立 Copy 引擎与 DMA 调度（第 13 章）；本章 `COPY_MEM` 由 CP 同步调用内存子系统完成。
- 不做计算内存域与 HOST_VISIBLE/HOST_COHERENT 组合（第 11 章），本章全部视为 coherent。

---

## 2. 单元框图

```
 门面 / CP 包处理
      │ Map / Read / Write / Translate
      ▼
┌──────────────────── hw_sim/memory/flat ────────────────────┐
│ AddressSpace   VA [0, kFlatSize) 恒等映射；页 4 KiB 对齐     │
│ BumpAllocator  nextVa 单调递增；4 KiB 对齐；OOM 时不变更状态  │
│ Storage        std::vector<u8>（按页扩展）；越界即错误        │
│ 资源表          {va, bytes, tag}；供 dump 与泄漏检查         │
└───────────────────────────┬────────────────────────────────┘
                            │
   CP 分支：SET_STATE_BASE → 基址表；CLEAR → 填充；COPY_MEM → 块拷贝
```

---

## 3. 输入输出契约

### 3.1 内存接口（`hw_sim/memory/flat/memory.h`）

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

| 方法 | 本章语义 | 错误 |
|---|---|---|
| `Read/Write` | 恒等映射后直接读写 `Storage`；`bytes` 可为 0 | 越界/未映射 → `NotMapped` + VA |
| `Translate` | `*outPa = va` | 越界 → `NotMapped` |
| `Map` | 标记 `[va, va+bytes)` 为已映射；已映射区间重复映射 → `InvalidArgument` | — |
| `Unmap` | 清除映射；未映射区间 → `InvalidArgument` | — |
| `InvalidateTlb` | 空实现（无 TLB） | — |
| `IsResident` | 已映射即 true | — |

硬规则：硬件单元内部禁止直接解引用主机指针；`Read/Write` 是唯一通道（ADR-042）。

### 3.2 分配器契约

```cpp
struct AllocDesc {
    u64      bytes;      // 向上取整到 4 KiB
    u32      alignment;  // 2 的幂，>= 4096
    const char* tag;     // 资源标签，仅诊断
};
struct AllocResult { GpuVAddr va; u64 bytes; };
Result Alloc(const AllocDesc&, AllocResult* out);
```

- 分配序列只由调用顺序决定，不含随机；同一输入多次运行 VA 完全相同。
- 失败时 `nextVa` 与已分配集合不变（无部分分配）。
- 不回收；释放仅标记 `tag` 结束（用于快照），VA 不复用。

### 3.3 包契约（本章新增分支）

| 包 | 载荷 | CP 行为 |
|---|---|---|
| `SET_STATE_BASE` (0x02) | `{u32 stateId; u32 reserved; u64 baseVa;}` | 写 `stateId` 基址表（MMIO `0x40~0x7F` 对应项） |
| `CLEAR` (0x08) | `{u32 rtWidth; u32 rtHeight; u32 colorArgb;}` | 锁存 `RT_SIZE`（0x30），按 BGRA 填充 `stateId 0` 指向的 RT |
| `COPY_MEM` (0x05) | `{u64 dstVa; u64 srcVa; u64 bytes;}` | 经 `IMemorySubsystem` 逐段拷贝；区间重叠且非恒等 → `InvalidArgument` |

`stateId` 取值见 `../design/command-mmio.md` §3；本章只使用 0（RT 基址）与 1（页表基址）。

**初级提前落地**：本章写 stateId 0/1（第 10 章迁到 FW）；stateId 基址步长 2 dword（`stateId n → 0x40+2n`，`0x50~0x7F` 保留）（Q-36、Q-31）。

> ⚠️ stateId 编号纪律（三档，`../design/bit-budget.md` §3.1）：`0~7` 直接 MMIO 映射（`0x40+2n`）冻结；`0x50~0x7F`（stateId 8~31）在**本章范围按"保留且拒绝"处理**（读写拒绝、未实现，见 D3.5）；该段仅作为未来选配的临时过渡缓冲设计（`bit-budget.md` §3.1），不得作为长期设计，一旦启用必须尽快切换为 GPU 内存间接状态表（`SET_STATE_TABLE_INDEX`）；**禁止继续在 MMIO 硬编码新增 stateId**。本章只用已登记的 stateId 0/1。

### 3.4 资源上传 / 读回

| 操作 | 接口 | 语义 |
|---|---|---|
| 上传 | `Upload(hostPtr, bytes) -> GpuVAddr` | 分配 + `Write`；返回 VA |
| 读回 | `Download(GpuVAddr, hostPtr, bytes)` | `Read` 到主机缓冲 |
| 快照 | `DumpAllocs()` | 输出 `{va, bytes, tag}` 列表（诊断） |

---

## 4. 内部数据流

### 4.1 分配 → 写入 → 读回

```
Alloc(bytes, align) → 返回 va（不触碰 Storage 内容）
Write(va, src, bytes) → 边界检查 → 复制 → 记录脏页（本章无缓存，仅记录）
Read(va, dst, bytes)  → 边界检查 → 复制
DumpAllocs() → 按 va 升序输出
```

### 4.2 `CLEAR` 包处理

```
解析载荷 → 校验 rtWidth/rtHeight ≤ 8192 → 写 MMIO RT_SIZE（宽低16|高高16）
→ 读取 stateId 0 基址 → 计算 bytes = w*h*4
→ 对每个像素写 [B,G,R,A]（由 0xAARRGGBB 拆解）
→ 区间越界 → NotMapped + 错误记录（不部分填充）
```

### 4.3 `COPY_MEM` 包处理

```
读 dstVa/srcVa/bytes → 校验区间均映射 → bytes==0 直接成功
→ 若区间重叠：仅允许 dstVa == srcVa（直接成功），否则 InvalidArgument
→ 分块拷贝（块大小 4 KiB，与页对齐无关）→ 返回 Ok
```

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T3.1 | ① 契约 | `IMemorySubsystem`、`AllocDesc/AllocResult`、`MapFlags`、资源快照结构、地址空间上限与页粒度常量落头文件 |
| T3.2 | ② 主体 | `AddressSpace` + `Translate` + `Map/Unmap` + `IsResident`：恒等映射、区间校验、重复映射拒绝 |
| T3.3 | ② 主体 | `Read/Write`：边界检查、`NotMapped` 错误带 VA、`bytes==0` 语义 |
| T3.4 | ② 主体 | bump 分配器：对齐、耗尽 `OutOfMemory`、失败不变更状态、分配序列可复现 |
| T3.5 | ② 主体 | `SET_STATE_BASE`：`stateId` 基址表 + MMIO `0x40` 段 LO/HI 写入与回读 |
| T3.6 | ② 主体 | `CLEAR`：尺寸锁存、BGRA 填充、越界拒绝 |
| T3.7 | ② 主体 | `COPY_MEM`：块拷贝、重叠判定、零长度 |
| T3.8 | ③ 接入 | 门面接入：为 RT/顶点/索引/纹理分配并上传，提供 `Download` 读回；`scene_e1` 走 `CLEAR` 全流程 |
| T3.9 | ④ 验收 | A3：E1 读回 = 显存子集 + E4 资源上传子集 + 本章专有判据 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D3.1 | 头文件独立编译；`AllocDesc` 字段偏移 `static_assert`；地址空间上限、页粒度常量与 §3.2 一致 | L1 契约 |
| D3.2 | 同 VA 两次 `Map` 返回 `InvalidArgument`；`Unmap` 后 `Translate` 返回 `NotMapped`；`Map` 区间重叠检测覆盖 4 种边界（起点内/终点内/包含/相交） | L1 契约 |
| D3.3 | 写入 `[va, va+N)` 后读回逐字节相等（N=0、1、4096、4097）；跨越未映射空洞的读返回 `NotMapped` 且 `ErrorRecord.va` 等于空洞起点。**新增 L1 契约单测（正常+非法边界，部分失败语义）**：多页范围（≥4097 B）其中一页未映射 → 整体返回 `NotMapped`，且**无半写副作用**（目标区间任何已映射字节不得被部分修改） | L1 契约 |
| D3.4 | 连续 1000 次分配区间两两不重叠（脚本校验 `[va, va+bytes)`）；把容量耗到 0 后下一次分配返回 `OutOfMemory` 且 `nextVa` 不变；同一调用序列两次运行 VA 序列完全相同 | L1 契约 |
| D3.5 | `SET_STATE_BASE(stateId=0, va)` 后 `0x40/0x41` 回读等于 va 的低/高 32 bit；`stateId=8` 被拒 | L1 契约 |
| D3.6 | `CLEAR(256,256,0xFF3F3F3F)` 后 RT 区间每个字节序列为 `[63,63,63,255]`；`RT_SIZE` 寄存器为 `0x0100_0100`；宽 0 或 >8192 返回 `InvalidArgument` 且目标区间零字节变更 | L1 契约 |
| D3.7 | `COPY_MEM` 源→目标逐字节相等；`dst==src` 返回 `Ok`；部分重叠返回 `InvalidArgument` 且目标不变；`bytes=0` 返回 `Ok` | L1 契约 |
| D3.8 | `scene_e1` 提交后 `Download` RT 全图等于 `(63,63,63,255)`；门面公开头中不含 `IMemorySubsystem` 之外的内存类型；`scene_e4_mesh` 可上传约 1 千三角形顶点（`1.5 万 dword` 级）并原样读回 | L1 契约 + A3 |
| D3.9 | A3 全绿：E1 的"读回缓冲 = 显存内容"子项通过；E4 的"资源上传/读回"子项通过（渲染判据待第 6 章）；同一数据两个句柄（不同 VA 同内容）读回一致 | L1 契约 |

---

## 7. 场景与判据（A3）

**绑定场景：E1 `scene_e1_clear_present`（清屏）、E4 `scene_e4_mesh`（非索引大网格）——本章只跑内存相关子集。**

| 项 | 内容 |
|---|---|
| 输入 | E1：`0.25` 灰、`256x256`；E4：约 1 千三角形非索引顶点流 |
| 本章产物 | RT 内存字节、顶点缓冲内容、资源快照 `{va,bytes,tag}` |
| 主判据 | E1：RT 每像素 `[63,63,63,255]`；E4：上传后读回与源逐字节一致 |
| 本章专有判据 | ① 同一数据两个句柄读回一致；② 分配区间不重叠；③ 耗尽返回确定错误 |
| 回归项 | R03：E0 契约自检，以及 E1 已交付的 fence/完成计数子判据；不把 A2 整章作为门禁 |
| 后续补齐 | E1 的"双缓冲前后帧可区分"依赖第 8 章；E4 的"包围盒像素可复算"依赖第 6 章 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 分配序列由调用顺序决定，无随机；拷贝按地址升序分块，结果与块大小无关 |
| 时间无关 | 无超时、无墙钟；`IsResident` 只依赖映射表 |
| 可观测 | `DumpAllocs()` 资源快照；`ERROR_*` MMIO；`--trace=memory` 打印每次 `Read/Write` 的 `{op, va, bytes}` |
| 契约影响 | `IMemorySubsystem` 签名冻结；第 11 章只替换实现，不改签名 |
| 简化登记 | 恒等映射、无 TLB/缓存/页错误、单一 coherent 域（§9） |
| 计算兼容 | 地址空间不区分图形/计算；`MapFlags` 预留 `HOST_VISIBLE/HOST_COHERENT` 组合位，本章全部视为 coherent |
| 配置与回退 | 无独立开关；`mem_sanitizer` 已登记但本章不实现（第 11 章起） |
| 调试钩子 | `--trace=memory` 仅人工；不进入判据 |
| trace 契约 | 字段不承诺稳定 |
| 契约变更 | 地址语义、分配失败行为、包载荷为冻结项；变更需 ADR 并回归 E1/E4 |
| 校验归属 | 全部 L1 契约断言；RT 字节比对按 BGRA 取字节再与 RGBA 期望比对 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 地址 | 单一连续 VA，恒等映射，无 TLB、无页错误 | 11 |
| 存储 | 扁平字节数组，无 L1/L2、无写缓冲 | 12 |
| 堆 | 单一 bump 分配器，无 VRAM/GART/System 区分 | 11 |
| 主机同步 | 全部 coherent，无 flush/invalidate | 11 |
| 拷贝 | 由 CP 同步调用内存子系统，无独立 DMA 引擎 | 13、33 |
| 驻留 | 全部常驻，无逐出 | 14 |

---

## 10. 待定项

无。GART 容量、页表层级与 `MapFlags` 的完整语义在第 11 章定案；本章只保证签名与失败语义不变。
