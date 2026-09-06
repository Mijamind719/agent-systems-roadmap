# 补充研究：Agent 安全与 DeepSeek Harness

核验日期：2026-09-06。初步问题地图；未进行源码审计、攻击复现或性能测量。

## Agent 安全

暂按“保护 agent 系统及它访问的数据和外部系统”理解。用 agent 做漏洞研究、检测和响应属于另一个应用方向，是否纳入待用户确认。

公开依据：Anthropic 将可信 agent 的实践放在模型、工具、权限和环境等多个层次；MINJA 研究说明记忆污染能够影响后续会话。这支持跨层研究，但不能证明某个项目存在相同漏洞。[Trustworthy agents](https://www.anthropic.com/research/trustworthy-agents)、[MINJA](https://arxiv.org/abs/2503.03704)

建议研究矩阵（助手提出，尚未批准）：

| 层次 | 核心问题 | 与现有投入的连接 |
|---|---|---|
| 输入与意图 | 外部数据如何被误当作指令；如何识别目标偏移？ | Harness 的输入来源与执行策略 |
| 身份与权限 | 工具调用以谁的身份执行，委派能否扩大权限，撤权何时生效？ | Harness、multi-agent |
| 运行环境 | 隔离、出网、凭证、宿主与租户边界如何闭合？ | Sandbox |
| 记忆与上下文 | 污染、跨租户泄漏、来源丢失、删除后的派生内容如何处理？ | OpenViking、共享记忆 |
| 插件与工具 | 不可信插件/MCP/skill 的代码和返回内容分别能触及哪些资源？ | DeerFlow、DSH 生态 |
| 协作与恢复 | 错误是否跨 agent 传播；重试、回滚是否重复外部副作用？ | 四方向接口 |
| 验证与处置 | 如何测攻击成功、业务损失、误阻断、人工负担及撤销恢复？ | 横向评估 |

规划推断：安全既应有独立问题清单，也应成为每个方向的设计和验收条件。是否成立独立投入主线取决于用户、威胁模型和可验证差异。沙箱隔离不等同于语义授权正确，插件可替换也不等同于安全隔离。

## DeepSeek Harness

已确认是 deepseek-ai/deepseek-harness（dsh）。官方当前标注 developer preview，核心能力和 API 仍将演进。[官网](https://deepseek.com/harness/en/)、[仓库](https://github.com/deepseek-ai/deepseek-harness)

官方声明：基于 Cordis 的插件组合；模型、工具、session、sandbox、storage、loop、调度和 UI 都可替换；append-only session log 记录模型上下文和执行事件，支持轨迹检查以及恢复、分叉、搜索和回放；提供 Standard、Code、Minimal、Creator 多种模式。上述是设计与能力声明，尚未证明恢复正确性或跨任务效果。[官方介绍](https://deepseek.com/harness/en/)

对照研究待办（助手建议）：

- 插件边界与故障边界是否一致；权限和资源隔离如何实现。
- session 日志是否足以支撑恢复，哪些外部状态无法被重放。
- OpenViking 类长期上下文与 session 原始记录如何划分职责。
- 换 sandbox 或 agent loop 后，验收、安全和恢复语义是否保持。
- 不同运行模式与多 agent 协作的收益，在同模型和预算下是否稳定。

不因 DSH 的插件理念预设应迁移 DeerFlow；优先明确哪些边界值得借鉴、哪些问题需要共同实验。
