# 启动、复位与关闭 — 独立模块设计

> 项目目标、范围、阶段与交付以[任务规划](../CPU模拟GPU教学项目_任务规划与实施规格.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

> 定位：定义设备上电/初始化顺序、拒绝创建路径、复位路径（定位 → reset → 重跑一致）、按引擎 watchdog、以及故障分层与逻辑特权分层的硬件侧语义。
> 落地章：第 10 章设备/KMD-FW 建立（启动序列）、第 36 章错误路径（复位 / watchdog / 故障注入）；`ResetRequired` 语义贯穿 11/14/36。
> 权威来源：从 `architecture-design.md` §4.11 拆出，吸收分层 / 故障分层 / 逻辑特权模型（原 §5.1/§5.2/§5.4，权威在 `architecture-design.md` §2）、能力边界（原 §8.4 → `roadmap.md` §0.5）与 ADR-031 / ADR-039 / ADR-040、`roadmap.md` §2.8（Watchdog 按引擎监控）。

---

## 1. 启动序列

```
上电 → MMIO 复位 → FW boot 握手 → ring 编程 → CP stop
→ KMD 建页表根/ASID → HLI 建设备/队列 → UMD 建默认对象
→ 正常提交
```

回调节点（层级生命周期，详细见 `architecture-design.md` 层间接口契约 §3.10）：

| 阶段 | 必须完成 | 失败处理 |
|---|---|---|
| HW 设备 | MMIO 复位、环清零、fence 归零 | 返回 `DeviceLost` |
| FW boot | 握手、ring 编程、CP 停止 | 中止创建，释放 HW |
| KMD | 页表根、ASID、提交队列 | 释放 FW/HW |
| HLI | 对象表、描述符堆默认状态 | 释放 KMD |
| UMD | API 对象、默认管线 | 释放 HLI |

`Shutdown` 时必须：停止 CP → 等待队列空闲 → 释放 UMD → 释放 HLI → 释放 KMD → FW halt → HW 关闭。任何一步跳过都视为资源泄漏。

## 2. 复位路径（第 36 章）

复位为**引擎级复位**（Q-69）：只重置故障引擎并仅重编程该 ring，其余引擎的队列/数据/fence 保持完好，不做设备级重启。初级只记录错误并中止当前提交、**不产生 `ResetRequired`**；复位路径仅第 36 章接入（Q-30）。

1. 检测到该引擎的 `ResetRequired` 或按引擎 watchdog 超时；
2. 停止该引擎 CP，丢弃其未完成包（其余引擎继续运行）；
3. 清空该引擎队列与 fence 监视，保留错误记录；不触碰其余引擎数据；
4. 仅重编程该引擎的 ring（不重新 boot 全设备 FW、不重建其他上下文）；
5. 场景可从固定输入重跑，结果与首次运行一致。

复位不承诺保留该引擎的 GPU 内存内容；不承诺恢复 wave 级中间状态（细粒度页错误恢复在项目范围外，`roadmap.md` §0.5）。

## 3. 故障分层（三类，禁止混淆，ADR-039）

| 故障类型 | 处理位置 | 行为 |
|---|---|---|
| 硬件故障：非法命令包、非法 ISA、GPU 内存访问违例 | `hw_sim` | 记录错误并中止当前提交（初级不产生 `ResetRequired`）；第 36 章按引擎级复位接入 |
| 驱动层错误：KMD/HLI 传参非法 | `kmd` / `hli` | 返回业务 `Result` 错误码，**不触发设备复位** |
| 应用层输入错误：非法句柄、错误参数 | `umd` 层拦截校验 | 转换上层返回码，向下不传递非法参数给 KMD |

红线（`architecture-design.md` §2 分层与故障模型）：**应用逻辑错误不允许穿透直达硬件单元、触发 GPU 设备复位**；`hw_sim`/`firmware` 不修复非法输入、不做兼容修补；API workaround 只在 UMD。

## 4. Watchdog 与故障注入

- **Watchdog 按引擎监控**（`roadmap.md` §2.8）：每引擎独立活跃标记，watchdog 按引擎判定是否推进并只复位对应引擎（贴近 TDR），不做全设备一刀切。**计算引擎推进单位（2026-11-20 定案）= workgroup 完成**（每个 workgroup 出队算一次推进，packet / wave 粒度不算）；watchdog 阈值按 dispatch 规模折算（`kMaxWorkgroups` = 2^20 上限 × 单组预算），防止长 kernel（如 2^20 组的 GEMM）被误判超时复位。
- **故障注入范围（ADR-031）**：仅高级硬件单元；初级与 UMD 层不注入。
- 错误只做"记录 + 中止当前提交"，不做重试与自动恢复；第 36 章只保证"可定位 + reset 后可重跑"。
- 场景 S13：非法 VA / 非法包 / 死循环 → 错误码、日志、复位后画面；每类故障定位到单元与位置；reset 后重跑 S2 成功。

## 5. 逻辑特权分层（高级 A，ADR-040）

`hw_sim` 硬件引擎接收提交时携带特权标记；UMD（用户态）不允许直接下发复位、修改页表根、看门狗等特权 MMIO 操作；特权操作必须经 KMD-DDI 封装，由 Firmware 执行。复现真实驱动安全模型，为未来虚拟化扩展预留。

## 6. 与错误契约的联动

全栈跨层接口统一返回 `Result`（`architecture-design.md` 层间接口契约 §3.9）；错误日志必须包含 `{unit, packetType, dwordOffset, va, code}`（`../docs/design/logging.md`）；初级不产生 `ResetRequired`，第 36 章按引擎级复位接入（ADR-031、Q-30、Q-69）。

## 7. 契约变更

- 复位语义、watchdog 行为、特权边界属冻结契约，变更走 `roadmap.md` §7.4（新建 ADR）并回归 S1/S13/S2。
