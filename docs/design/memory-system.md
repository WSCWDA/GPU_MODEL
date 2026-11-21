# 内存系统 — 独立模块设计

> 项目目标、范围、阶段与交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

> 定位：定义 GPU 内存模型的所有权归属：三类内存地址与唯一访问通道、两级页表 / TLB、三堆（VRAM/GART/System）与分配器、缓存与写缓冲、驻留与上下文，以及主机-GPU 缓存同步、分配确定性与内存 Sanitizer。
> 落地章：第 3 章平坦内存（初级）、第 11 章地址翻译 + 堆、第 12 章缓存 + 写缓冲、第 14 章驻留 + 上下文；Copy 引擎与 Display 统一走本模型。
> 权威来源：从 `architecture-design.md` §4.2 拆出，并吸收内存域 / 地址模型与堆（原 §4.8，已并入本文件）、`roadmap.md` §5.1（`mem_sanitizer` 配置）与 §2.8（统一内存域、I-Cache）、ADR-015 / ADR-031 / ADR-042。
> 职责关系：项目范围与阶段见任务规划；本文件负责内存模块内部机制，架构文档负责 IMemorySubsystem 等跨层接口，roadmap 负责章级实施顺序与验收。

---

## 1. 三类内存地址与唯一访问通道

| 地址类型 | 含义 | 使用范围 |
|---|---|---|
| `HostVa` | CPU 主机虚拟地址 | 仅 KMD/UMD/上层工具使用 |
| `GpuVAddr` | GPU 虚拟地址 | **`hw_sim` 硬件单元全部只使用此类型** |
| `PhysAddr` | GPU 物理地址 | 页表内部使用 |

硬规则（ADR-042）：**硬件单元内部永远禁止直接解引用主机 CPU 指针**，全部经 `IMemorySubsystem::Translate` 翻译（接口见 `architecture-design.md` §3.6）；初级 flat 内存中 Translate 是恒等映射，但接口必须保留，高级第 11 章直接替换实现、上层代码无需改动。

## 2. 地址与页表

- 软件类型 `GpuVAddr`/`PhysAddr` 为 u64；**硬件有效 VA = 36 bit**（B-07 裁定，ADR-043）：两级页表（PDE → PTE）各 4096 项 × 4 KiB 页 = 64 GiB/AS，覆盖 KMD 4 GiB/8 GiB 堆基址（`common` 页表常量 12+12+12）。历史"64 bit VA / 每级 512 项（30 bit / 1 GiB）"表述已废止，见 `bit-budget.md` §4。
- 每个上下文一份页表根，`SET_STATE_BASE(stateId=1)` 在初级由驱动写入，第 10 章迁 FW（Q-36）。
- TLB：每 CU 64 组 × 4 路 = 256 项，索引 `(va>>12)&0x3F`（Q-45）；失效必须全局可见；策略 B 下线程本地 TLB 依赖全局失效序列（见 `../docs/design/execution-parallel.md`）。
- 页错误：记录 VA 与上下文后**中止当前提交、不复位**；复位仅第 36 章按引擎级接入（Q-30、Q-69；细粒度页错误恢复在项目范围外，见 `roadmap.md` §0.5 能力边界）。

## 3. 堆与分配

| 堆 | 用途 | 分配器 | 可见性 |
|---|---|---|---|
| VRAM | RT、纹理、深度、IB、常量 | 空闲链表，4 KiB 对齐 | GPU 可读写；host 不可直接映射 |
| GART | 顶点/索引/上传/系统缓冲、描述符堆（Q-51）、计算 scratch（per-thread 寄存器溢出，`architecture-design.md` §3.13） | 无回收 bump，有上限 | host 可写，GPU 可读 |
| System | 驱动元数据 | host 分配器 | 仅 host |

堆管理器行为约定（本文件 §3）：

| 堆 | 分配策略 | 失败行为 |
|---|---|---|
| `VRAM` | 空闲链表分配 | `OutOfMemory` |
| `GART` | bump 分配 + 容量上限，**不做碎片整理**（对齐真实硬件行为） | `OutOfMemory` |
| `System` | KMD 主机侧内存 | — |

约束：

- 描述符堆登记 GART（GPU 可读），不落 `System` 堆（Q-51）。
- GART 不做碎片整理；跨帧复用缓冲必须按上下文常驻，不每帧分配（工程红线 §7.4 #4）。
- 分配失败返回 `OutOfMemory`，不允许静默回退。
- 计算线的 unified / pinned 内存映射到 GART/System 域的 `HOST_VISIBLE` / `HOST_COHERENT` 组合，不新增第四类堆。
- **计算 scratch**（P1-8，**2026-11-20 定案**）：per-thread 栈 / 寄存器溢出落 GART，per-thread base 由驱动经参数区传入；每线程 ≤4 KiB、4 B 对齐、不清零；每 wave 上限 128 KiB，每 dispatch 按有效线程数计算需求并受 GART 可用容量约束，分配失败在提交前返回 `OutOfMemory`。scratch 不占 CU 的 LDS/VGPR/SGPR 驻留预算；分配器按 `translation.seed` 保持地址布局可复现。
- **统一内存域**（`roadmap.md` §2.8）：图形与计算共用同一虚拟地址空间，资源零拷贝共享；ASID 为上下文粒度；GART/VRAM/System 对计算同样可见。
- 全部堆支持输出快照 dump。

### 3.1 KMD 内存池（补充）

`kmd` 层引入**内存池**概念（类似操作系统段页式管理与 STL 分配器的分层）：

- KMD 维护 GPU 地址空间预算与页粒度管理（段 → 页 → 小对象）；
- 小内存分配走对象池 / 缓存池，减少对大堆空闲链表的压力；大块分配仍走堆管理器；
- 分配元数据（标签 / 逻辑时间 / 大小 / VA）由 KMD 记录，供内存快照与泄漏检测复用。

## 4. 缓存与写缓冲（第 12 章）

- L1：每 CU 16 KiB，64 B 行，写穿；L2：全局 256 KiB，写回；写缓冲按 64 B 合并。
- I-Cache（指令缓存）：取指走 I-Cache，只做命中统计，不影响正确性；软观测项，初级可无、高级补上（`roadmap.md` §2.8）。
- 缓存一致性由屏障与 fence 维护，不依赖时间或线程顺序。
- `cache_enable=off` 只关闭命中统计与缓存路径，不改变结果；任何开关前后结果不同视为正确性缺陷。
- 缓存统计（命中/失效/写回）进入校验模型的软观测（`../docs/design/validation-model.md`）。

缓存/写缓冲的教学目标是解释数据如何经过 L1/L2、何时留存或排空、flush/invalidate 如何影响可见性，以及消费者何时复用数据。hit/miss、失效数和写回数只是诊断统计，不是核心学习目标或正式场景通过条件。GS/Tessellation 的 patch/control-point、顶点流和生成数据缓冲属于阶段局部暂存/输出队列，不能与通用缓存层混为一谈。

## 5. 驻留与上下文（第 14 章）

- 驻留表记录 `{资源, 页范围, 优先级}`；逐出按 LRU，逐出前必须 flush 写回。
- 上下文切换保存：通用寄存器、控制流栈、谓词、PC、绑定基址、SE 状态；shared memory 按上下文隔离。
- 第 14 章要求两个上下文交替提交时互不污染；页错误与逐出路径必须可重复。

## 6. 主机-GPU 缓存同步语义

区分 `HOST_VISIBLE` / `HOST_COHERENT`，以及 `Flush`（CPU 缓存写回 GPU）、`Invalidate`（GPU 修改后 CPU 缓存失效）。初级 flat 内存简化为全部 coherent；高级第 11 章完整实现内存域与 flush/invalidate。

## 7. 分配确定性与内存 Sanitizer

- **分配确定性**：所有随机行为绑定可复现伪随机种子，禁止直接调用主机全局 `rand()`；分配器 VA 序列必须可复现——初级 bump 天然可复现，高级第 11 章堆管理器支持配置种子，多次运行 GPU VA 分配序列完全一致。
- **GPU 内存 Sanitizer**（`mem_sanitizer=on/off`，默认关，高级 A）：不是操作系统 ASAN，而是内存子系统内部检查，捕获 GPU 侧越界读写、GPU 侧 use-after-free（资源已释放仍被访问）、读取未初始化 GPU 内存；开启显著降低性能，CI 部分测试打开、日常默认关闭。属于调试工具，不混入硬件核心执行路径。
- **分配元数据与泄漏检测**：每块分配记录标签 / 逻辑时间 / 大小 / GPU VA，可导出内存快照 dump；`tools/mem_audit` 离线解析，检测 GPU 内存泄漏与存活分配统计。

## 8. 接口与登记

- 内存访问统一走 `IMemorySubsystem`（`architecture-design.md` §3.6：`Read/Write/Translate/Map/Unmap/InvalidateTlb/IsResident`）；所有单元（含 Copy、Display）统一 `VA → TLB → L2 → 存储`，不得绕过翻译直连堆。
- 未映射返回 `NotMapped` 并带 VA；未驻留触发页错误路径，不得静默读 0。
- RT 字节序统一 BGRA（ADR-015，`R8G8B8A8_UNORM`、内存 `[B,G,R,A]`，Clear/光栅/OM/Present/校验全栈一致）。
- 登记与变更：地址模型 / 堆 / 页错误语义等契约变更走 `roadmap.md` §7.4；`stateId` 与 MMIO 段登记见 `../docs/design/command-mmio.md`。
