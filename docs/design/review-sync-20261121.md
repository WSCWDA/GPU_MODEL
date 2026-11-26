# 设计评审待验证事项

本清单记录实施待验证事项。原评审编号可查阅 Git 历史中的[问题评审原稿](https://github.com/WSCWDA/GPU_MODEL/blob/6a33e1234080e2860e9ece008dc360a1cb66f43a/docs/%E8%AE%BE%E8%AE%A1%E6%96%87%E6%A1%A3%E9%97%AE%E9%A2%98%E8%AF%84%E5%AE%A1-20261120.md)和[改进建议原稿](https://github.com/WSCWDA/GPU_MODEL/blob/6a33e1234080e2860e9ece008dc360a1cb66f43a/docs/GPU%20%E8%BD%AF%E6%A8%A1%E5%9E%8B%E8%AE%BE%E8%AE%A1%E8%AF%84%E5%AE%A1%E6%94%B9%E8%BF%9B%E5%BB%BA%E8%AE%AE%E6%96%87%E6%A1%A3.txt)。 本轮文档裁决见 [ADR-054～058](../roadmap.md#03-adr-决策清单)。旧待修项已写入责任章节及场景判据。

## 实施待验证

| 项目 | 关闭证据 |
|---|---|
| P0-03 · 契约快照与回归影响集 | [路线图 §7.4](../roadmap.md#74-契约变更工作流)的快照与 CI 检查落地。变更需要 ADR，有效变更触发相应 `R_n`，退役判据退出门禁。 |
| B-042 · MMIO 审计 | [公共 MMIO 表](command-mmio.md)与实现的偏移、权限和位宽一致。D2.2、D3.6、D10.1 和 D11.12 验初级子集、CLEAR、公共表与页表根读回。 |
| P1-11 / P1-45 / BACK-03 / BACK-04 · 完成与移交 | [同步原语 §2.1–§2.2](sync-primitives.md)落地后，S16/C3 验延迟和失败移交。S13 验普通页错误后队列继续，以及复位前等待与复位后查询得到同一旧失败。 |
| P0-10 / T-03 · Watchdog 边界 | [D36.4](../stage2/36-error-paths.md)覆盖预算内、边界正常退出、耗尽仍运行及合法长派发，验证根因诊断与引擎隔离。 |

S3、S18、C4 和显示翻帧的结果由现行场景规约与各章 DoD 验收。
