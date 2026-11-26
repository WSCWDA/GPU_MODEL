# 第 13 章 Copy Engine + Display

> 项目范围、阶段位置和交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design规定模块树与层间契约。本章只细化分配给本章的实现和验收，不单方面改变整体范围、正式顺序或跨层契约。

> 本章把 `COPY_MEM` 从"CP 同步执行"迁到独立 Copy Engine 引擎实例，并把初级"立即读 RT 上屏"替换为真实显示控制器：逐行扫描、翻转（flip）、vsync。两条线共享同一 `IMemorySubsystem`，都走 `VA → TLB → L2`。
> 与本章相关的设计说明：`../architecture-design.md` §3.3（`IEngine` 统一签名，Copy 引擎 `sa=nullptr`）；`../design/memory-system.md` §8（Copy/Display 统一走翻译）；`../design/sync-primitives.md` §1（fence）；ADR-015（BGRA 字节序）延续。
> 落地模块：`hw_sim/periph/copy`、`hw_sim/periph/display`、`hw_sim/core/cp` 追加 `COPY_MEM` 引擎派发、`apps/scene_s2_clear`、`apps/scene_e1`（迁移到新 Display）。

---

## 1. 目标与非目标

### 1.1 目标

1. 独立 Copy Engine：`COPY_MEM` 不再由 CP 同步执行，改由 CP `Push` 到引擎（`sa=nullptr`），`Execute`/握手排空。
2. Display 控制器：逐行从显存（RT VA）读像素 → 行消隐 → 数显输出 → 场消隐 → vsync；分辨率/刷新率/消隐期可配置。
3. Flip 双缓冲：将写完的缓冲登记为候选 front，每帧扫描使用同一锁存缓冲。
4. vsync 由扫描位置触发；`frame_index` 由提交驱动、禁 vsync/墙钟。
5. 扫描输出可配置分辨率（`rt_size`）且与 RT 尺寸一致；显示帧从 `front` 缓冲读。
6. CopyEngine 与 Display 均走 `IMemorySubsystem::Translate`，不得绕过翻译直连堆。

### 1.2 非目标

- 不做多显示头/模式切换/缩放；单 Display 输出。
- 不做引擎优先级仲裁（第 32 章）；Copy 与图形异步由 fence 排序。
- 不做 `COPY_MEM` 的 stride/深度相关拷贝（第 33 章 DMA 切片）；本章块拷贝。
- 不做窗口 blit（`umd/os` 可选，headless 可完整校验）。

---

## 2. 单元框图

```
 CP：COPY_MEM 包 ──► Copy Engine（Push/Pump/Pop，sa=nullptr）
                       │ 块拷贝经 IMemorySubsystem
 RT(back) ── 提交完成 ──► flip ──► Display(front)
 Display：扫描行计数器 → 行消隐 → 读像素 → 输出缓冲 → 场消隐 → vsync
                                  │
                       frame_buffer[front]（BGRA）
```

- Copy Engine 是独立 `IEngine` 实例，`IShaderArray*` 恒 `nullptr`（非着色引擎）。
- Display 与引擎并发（软模拟单线程内分时），只在提交完成后读 front。

---

## 3. 输入输出契约

### 3.1 Copy Engine

| 项 | 说明 |
|---|---|
| 包 | `COPY_MEM`(0x05) `{u64 dstVa; u64 srcVa; u64 bytes;}` |
| 输入 | `Push(pkt, ..., sa=nullptr)`；返回 `Busy` 表示 FIFO 满 |
| 输出 | 拷贝完成后向 CP 出队/写 fence |
| 错误 | 仅 `dst==src` 允许重叠；其余重叠 → `InvalidArgument`；未映射 → `NotMapped`+VA |

块拷贝语义：仅 `dst==src` 允许重叠，其余重叠返回 `InvalidArgument`；按 4 KiB 块升序；跨页自动走 `Translate`；`bytes=0` 直接成功。

### 3.2 Display 控制器（`hw_sim/periph/display/display.h`）

```cpp
struct DisplayDesc {
    u32 width;   // 例如 256
    u32 height;  // 例如 256
    u32 refreshHz;      // 默认 60
    u32 hBlankLines;    // 行消隐
    u32 vBlankLines;    // 场消隐
    GpuVAddr rtVa;      // front 缓冲 VA（flip 切换）
};
class Display {
public:
    Result Initialize(const DisplayDesc&);
    Result FlipTo(GpuVAddr newFrontVa);      // 只读指针切换
    Result ScanNextLine();                    // 推进行扫描
    const std::vector<u8>& FrontBufferSnapshot() const; // 已经由 IMemorySubsystem 读回的 CPU 副本，只读
    bool     VBlank();                        // 当前在场消隐？
    u32      ScanY() const;                   // 当前扫描行
};
```

- `FlipTo` 登记下一帧的 front VA，可见区的请求在 VBlank 接收，VBlank 内请求更新候选。下一帧首条可见扫描线锁存候选 VA。
- 输出缓冲从 `front` VA 读像素，字节序 `[B,G,R,A]`（ADR-015）。

### 3.3 帧时序

- 一帧 = 可见行(高) + 场消隐；行消隐在每行末端。
- `vsync` 为进入 VBlank 的逻辑扫描事件，首条可见扫描线锁存本帧缓冲。
- `frame_index` 由提交驱动（每帧 +1），不由 vsync 推进。

---

## 4. 内部数据流

### 4.1 CopyEngine 拷贝

```
Pull COPY_MEM → 校验区间 → 分块(4KiB)升序 → 每块 Translate src/dst → 拷贝
→ FIFO 出队 → 写 fence（第 10 章机制）
```

### 4.2 显示扫描

```
ScanNextLine: y=ScanY()
  可见区 → 从本帧锁存的 front VA 读第 y 行
  行消隐 / 场消隐 → 推进 ScanY，VBlank=场消隐 → 触发 vsync 回调
  场消隐结束 → y 回 0（下一帧）
```

### 4.3 Flip 与撕裂（ADR-058）

可见区请求延至 VBlank 更新候选，VBlank 内以最后接受的候选为准。下一帧首条可见线锁存该 VA，保持至本帧扫描完成。场景在扫描期间保留该缓冲并避免复用其存储。旧的扫描线与地址比较判据为 **SUPERSEDED**。

---

## 5. 任务清单

| 任务 | 段 | 内容 |
|---|---|---|
| T13.1 | ① 契约 | `DisplayDesc`/`Display`/Copy 引擎接口、扫描时序常量落头文件 |
| T13.2 | ② 主体 | CopyEngine 独立实例：`Push`/FIFO/块拷贝/`sa=nullptr` 容忍 |
| T13.3 | ② 主体 | CopyEngine 与 CP 解耦：`COPY_MEM` 派发改走引擎、fence 出队 |
| T13.4 | ② 主体 | Display 扫描：逐行读取、行消隐、场消隐、vsync 位置 |
| T13.5 | ② 主体 | Display 双缓冲 flip：front/back 指针 + 无撕裂时机 |
| T13.6 | ② 主体 | Display 与 `IMemorySubsystem`：读像素走 `Translate`；RT VA 对齐校验 |
| T13.7 | ② 主体 | Display 输出导出与前端帧一致；`rt_size` 分辨率配置 |
| T13.8 | ③ 接入 | `scene_s2_clear`：CopyEngine 填充 RT → flip → Display 扫描读回；`scene_e1` 迁到新 Display |
| T13.9 | ④ 验收 | A13：S2 CopyEngine + Display 全绿 + 专有判据 + R13 回归项 |

**章内里程碑（Q-49）**：

| 里程碑 | 任务范围 | 收口判据 |
|---|---|---|
| M13.1 | T13.1~T13.5 | 接口/CopyEngine/Display 扫描与 flip，D13.1~D13.5 全绿 |
| M13.2 | T13.6~T13.9 | 内存接入/输出导出/场景接入验收，A13 全绿 |

---

## 6. DoD

| DoD | 判据（机器可判） | 级别 |
|---|---|---|
| D13.1 | 接口独立编译；`DisplayDesc` 默认值冻结（宽=高、refreshHz、消隐行） | L1 契约 |
| D13.2 | `COPY_MEM(256B)` 后 `Download(dstVa)` 与 `srcVa` 逐字节一致；`sa` 传 `nullptr` 不炸 | L1 契约 |
| D13.3 | 当前帧中途提出 flip，两缓冲使用可按行区分的图案。当前帧全部匹配旧 front，下一帧全部匹配新 front | L1 契约 |
| D13.4 | 分别在可见区和 VBlank 请求 flip。前者在进入 VBlank 更新候选，后者直接更新候选。下一帧首可见行锁存有效 VA | L1 契约 |
| D13.5 | CopyEngine 在扫描进行中仍可执行（软模拟单线程分片），完成 fence 值单调 | L1 契约 |
| D13.6 | Display 读像素走翻译：`rtVa` 位于未映射区 → `NotMapped` 报错 | L1 契约 |
| D13.7 | `rt_size=256x256` 与标准 16:9 `1280x720` 两种输入下，Display 经 `Download` 生成的 CPU 快照尺寸/字节序与对应 GPU front buffer 逐字节一致；不得暴露可绕过翻译的显存指针 | L1 契约 + L2 |
| D13.8 | `scene_s2_clear`：CopyEngine 填充值 = 出屏扫描结果（首行到末行逐行一致）；`scene_e1` 结果与 8 章一致 | L1 契约 + A13 |
| D13.9 | A13 全绿：S2 主判据 + 专有判据（CopyEngine 填充=出屏结果、双缓冲无撕裂） + R13 回归项 | L1 契约 |

---

## 7. 场景与判据（A13）

**绑定场景：S2 `scene_s2_clear`（+CopyEngine/Display 组合）。未映射 RT 作为本章负向单测，不绑定 S13 主判据。**

| 项 | 内容 |
|---|---|
| 输入 | S2：CopyEngine 把一段灰度源拷贝到 `back`，flip 后 Display 扫描；负向：`rtVa` 指向未映射区 |
| 产物 | CopyEngine 填充前后源/目的、Display 扫描帧、flip 快照 |
| 主判据 | `CopyEngine 填充 = 出屏结果`（逐像素）；S2 全流程输出与 read-back 一致 |
| 本章专有判据 | CopyEngine 填充与扫描读回一致。可见区中途翻转前后，两整帧分别对应各自锁存的 front |
| 回归项 | R13：`scene_e1` 与第 8 章基线一致（Display 迁移不破基线）；未列明历史 A 门禁不自动阻断 |
| 后续补齐 | 多引擎 Copy 并发与 slice 拷贝在第 33 章；display 多 head/缩放范围外 |

---

## 8. 横向关注点

| 关注点 | 本章回答 |
|---|---|
| 确定性 | 拷贝块序升序；扫描行序固定；flip 时机受 vsync 窗口（非墙钟） |
| 时间无关 | vsync 由扫描位置触发，非墙钟；`frame_index` 由提交驱动 |
| 可观测 | `ScanY/VBlank`、flip 计数、输出首行快照、`--trace=display/copy` |
| 契约影响 | `Display` 接口与 CopyEngine `IEngine` 实例冻结 |
| 简化登记 | 同步 `COPY_MEM` → 独立 CopyEngine；立即上屏 → 逐行扫描 + flip |
| 计算兼容 | CopyEngine 与图形/计算共用内存与 Fence；Display 只读 front |
| 配置与回退 | `rt_size`、refreshHz、消隐行可配置；Display 不影响渲染计算 |
| 调试钩子 | `--trace=display` 每行一行；`--trace=copy` 每次拷贝一行 |
| trace 契约 | 不进入判据 |
| 契约变更 | 扫描时序、flip 语义、CopyEngine 派发为冻结项；变更走 ADR 回归 S2/S13 |
| 校验归属 | L1（结构/fence）+ L2（扫描帧与 ref_model 计算的 front 期望逐像素比） |
| 双策略 | Display 只读 front，不与执行并发写；CopyEngine 在多引擎场景由 fence 排序（第 32/33 章） |
| 阶段衔接 | 为第 26 章 OM 写 RT、第 33 章 DMA 切片提供 Copy/Display 通路 |

---

## 9. 简化登记

| 项 | 本章实现 | 替换章 |
|---|---|---|
| 拷贝 | CP 同步 → 独立 CopyEngine | —（定案） |
| 显示 | 立即读出 → 扫描 + flip + vsync | —（定案） |
| 撕裂 | vsync 窗口 flip（无混合行） | — |
| 引擎并发 | 单模拟线程分片 | 32（真异步） |
| DMA · slice | 块拷贝 | 33（切片/间接） |

---

## 10. 待定项

- CPU 读回只通过 `Download(frontVa)` 生成独立 CPU 快照；`FrontBufferSnapshot()` 返回该快照，不暴露 GPU 显存指针。窗口 blit 消费 CPU 快照，不直接读取 GPU VA；GPU 侧访问仍统一走 `IMemorySubsystem` 的翻译路径。
- flip 根据逻辑扫描位置推进，执行 D13.3/D13.4 的整帧判据。
- 双缓冲的 front/back 在 `wave` 未排空时 flip 是否允许，受限语义留到第 32 章 fence 真异步后统一。

## 执行切分与 Git 提交模板

先用章内最小输入验证本章新增模块和失败路径，再接入 §7 登记的场景子例，完成 A13 和适用的 R13 后提交。分配核对项为拷贝偏移单位、RT 锁存，新增或改动按 [编码与位段校验表](../design/allocation-audit.md)登记。

```text
feat(gpu-ch13): copy resources and export latched frames

- 接通 Copy Engine 与 Display，保持整帧锁存。
- 用 S2 比对跨引擎数据/帧与未映射负例。

Tasks: T13.1–T13.9
Validation: <实际执行的命令、用例和结果>
```
