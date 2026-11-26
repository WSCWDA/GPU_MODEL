# 启动、复位与关闭 — 独立模块设计

> 项目目标、范围、阶段与交付以[任务规划](../README.md)为主干；roadmap展开章级任务，architecture-design定义模块树与层间契约。本文负责本模块的内部技术设计。

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

正常 `Shutdown` 先停止接收新提交，保持 CP 可推进。所有已接受提交成功或明确失败后停 CP，依次释放 UMD、HLI、KMD、FW、HW。若剩余队列在关闭期间无法推进，依 [同步原语 §2.1](sync-primitives.md#21-提交完成与失败传播)结束等待并释放引用。创建失败只逆序回滚已创建的对象。

## 2. 复位路径（第 36 章）

复位为**引擎级复位**（Q-69）：只重置故障引擎并仅重编程该 ring，其余引擎的队列/数据/fence 保持完好，不做设备级重启。初级只记录错误并中止当前提交、**不产生 `ResetRequired`**；复位路径仅第 36 章接入（Q-30）。

1. 检测到该引擎的 `ResetRequired` 或按引擎 watchdog 超时；
2. 停驻该引擎 CP 与 worker 并确认其在飞资源访问已结束。将丢弃提交和未执行信号登记失败，唤醒等待者并释放对应引用。旧终态继续可查，独立引擎继续推进。
3. 复位期间新提交返回 `Busy`；复位完成后旧 queue/stream generation 句柄返回 `DeviceLost`；不隐式重放；保留错误记录且不触碰其他引擎数据；
4. 仅重编程该引擎的 ring（不重新 boot 全设备 FW、不重建其他上下文）；
5. 应用显式重建受影响的 queue/stream/资源并重新提交固定输入；结果与首次运行一致。

丢弃提交的受影响产物需重新初始化。保留用户引用和其他有效提交引用的原生命周期。

## 3. 故障分层（三类，禁止混淆，ADR-039）

| 故障类型 | 处理位置 | 行为 |
|---|---|---|
| 硬件故障 | `hw_sim` | 可定位非法包、非法 ISA 和普通页错误返回对应错误并结束单次提交。引擎级 `ResetRequired` 或 watchdog 超时按 §2 复位 |
| 驱动层错误：KMD/HLI 传参非法 | `kmd` / `hli` | 返回业务 `Result` 错误码，**不触发设备复位** |
| 应用层输入错误：非法句柄、错误参数 | `umd` 层拦截校验 | 转换上层返回码，向下不传递非法参数给 KMD |

红线（`architecture-design.md` §2 分层与故障模型）：**应用逻辑错误不允许穿透直达硬件单元、触发 GPU 设备复位**；`hw_sim`/`firmware` 不修复非法输入、不做兼容修补；API workaround 只在 UMD。

## 4. Watchdog 与故障注入

- **Watchdog 按引擎监控**：每 wave 独立累计退休指令，生产预算 `kMaxInstructionsPerWave=2^24`。预算内或最后一条恰好耗尽并正常退出均成功。耗尽且仍运行返回 `Timeout` 并保留 wave/PC/dispatch，只复位该引擎。测试可使用缩小的同语义预算。
- **故障注入范围（ADR-031）**：仅高级硬件单元；初级与 UMD 层不注入。
- 调用方在状态恢复后显式重提有效命令。
- S13 分开验证普通页错误后同队列继续运行、引擎复位后显式重提及 watchdog 边界。

## 5. 逻辑特权分层（高级 A，ADR-040）

`hw_sim` 硬件引擎接收提交时携带特权标记；UMD（用户态）不允许直接下发复位、修改页表根、看门狗等特权 MMIO 操作；特权操作必须经 KMD-DDI 封装，由 Firmware 执行。复现真实驱动安全模型，为未来虚拟化扩展预留。

## 6. 与错误契约的联动

全栈跨层接口统一返回 `Result`（`architecture-design.md` 层间接口契约 §3.9）；错误日志必须包含 `{unit, packetType, dwordOffset, va, code}`（`../docs/design/logging.md`）；初级不产生 `ResetRequired`，第 36 章按引擎级复位接入（ADR-031、Q-30、Q-69）。

## 7. 契约变更

- 复位语义、watchdog 行为、特权边界属冻结契约，变更走 `roadmap.md` §7.4（新建 ADR）并回归 S1/S13/S2。
