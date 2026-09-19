# agent 与 harness 之辨

状态: 已对照验证（2026-09-08 对照根 README.md、各包 package.json、packages/agent 源码结构与行数统计）

## 事实源（链接，不复述）

- [README.md](../../../README.md)：项目自称 "Pi agent harness project"；产品自称 "self extensible coding agent"——双词并用
- [agent-loop.ts](../../../packages/agent/src/agent-loop.ts)：loop 本体，725 行
- [packages/agent/src/harness/](../../../packages/agent/src/harness/)：仓库自己命名的 harness 子系统（agent-harness.ts + compaction / storage / session-repo / lane 等）
- [containerization.md](../../../packages/coding-agent/docs/containerization.md)：安全边界放在 harness 之外的佐证

## 它是什么（用自己的话，≤5 句）

agent 是运行时涌现的行为，构成等式：LLM + 工具 + loop（通行定义，如 Anthropic《Building Effective Agents》：预编排路径 = workflow，LLM 自主决定路径 = agent）。harness 是工程实体：等式的生产级实现 + 等式之外的一切支撑设施。同一个系统两个论域：产品是 agent（用户敲 pi 看到的行为），仓库是 harness（工程师维护的代码主体）。harness 一词从 test harness 借来（跑测试的脚手架 → 跑 agent loop 的脚手架），本义「马具」——马（智能）在外部，马具让它能干活。

## 数字证据（2026-09-08 统计）

| 层 | 行数 | 占比 |
|---|---|---|
| 全仓 src（packages/ 12 个目录合计，行数为 2026-09-08 统计、未含 durable） | ~134,700 | 100% |
| agent-loop.ts（等式本体） | 725 | 0.5% |
| 等式三项生产实现（loop + agent 包工具设施 + pi-ai） | ~23,500 | ~17% |
| coding-agent（会话 / 扩展 / TUI / 工具实现） | 62,923 | 47% |

等式内部同样有 harness 现象：「接大模型」最小实现约 500 行（单 provider），生产实现 22.1k 行（pi-ai：多提供商统一 / 流式 / 缓存 / 用量核算）——差值不增加 agent 度，只增加可用性。

## 我曾经的误解

原以为：pi 是完整可运行的最小 agent 核心，所以仓库叫 agent 也说得通。
实际是：「构成了 agent」与「是一个 agent 仓库」是两个命题。用等式在仓库里画圈，圈出的部分越小，圈外（harness）越大——等式三项约 1–17%，其余 83% 不在等式里。README 双词并用：产品叫 agent，项目叫 harness，各说各的层面，不冲突。
修正来源：2026-09-08 讨论，[journal/2026-09-08-01-readme-harness.zh.md](../../journal/2026-09-08-01-readme-harness.zh.md)。

## 与相邻单元的关系

- 解释力：为什么 `.pi/` 扩展、compaction、TUI 都不在等式里却占 83% 代码——它们是 harness，不是 agent。
- 学习路线映射：等式三项（loop + 工具接口 + LLM 适配）= 阶段 2 精读范围；等式外的 83% = 阶段 3–6 主体。
- 命名辨析：本篇的 harness 是泛指（仓库自我定位）；[AgentHarness](../../../packages/agent/src/harness/agent-harness.ts) 是 agent 包 harness/ 子目录的具体类，二者关系见 [questions.zh.md](../../questions.zh.md) Q1。

## 验证方式

行数统计可复跑（PowerShell，仓库根）：

```powershell
(Get-Content packages/agent/src/agent-loop.ts | Measure-Object -Line).Lines
Get-ChildItem packages -Directory | ForEach-Object { $n = 0; if (Test-Path "$($_.FullName)\src") { $n = (Get-ChildItem "$($_.FullName)\src" -Recurse -Filter *.ts | Get-Content | Measure-Object -Line).Lines }; "$($_.Name): $n" }
```

## 遗留问题

无新增；AgentHarness 类与 agent-loop.ts 的关系仍归 [questions.zh.md](../../questions.zh.md) Q1（open）。
