# README 精读与「agent 还是 harness」之辨（2026-09-08）

背景：阶段 1 第 1 步执行中。先按计划的四个板块走 README（产品定位 / 包清单 / 开发命令 / 供应链加固），后补 Permissions & Containerization / standalone binaries / session sharing 三段；途中对「仓库为什么叫 harness 而不是 agent」产生分歧，讨论两轮。

## 卡点：仓库为什么叫 harness 而不是 agent

我的初始主张：pi 是一个完整可运行的最小 agent 核心（loop + 基本工具支持 + 接大模型 = agent），所以仓库叫 agent 也说得通。

## 辩论与修正过程

第一轮（三论据）：

1. 仓库不含智能——agent 的决策来自外部 LLM（pi-ai 只是统一适配器）；仓库静态时是纯 harness，agent 只在运行时涌现。
2. 代码质量分布——agent-loop.ts 仅 725 行，全仓 src 约 134.7k 行（0.5%）；coding-agent 62.9k 行（47%）；且 packages/agent/src 自己就分成 agent-loop.ts 与 harness/ 子目录（agent-harness.ts 581 行 + compaction / storage / session-repo / lane 等）。
3. 范围——仓库 = 11 个目录，agent CLI 只是其中之一；pi-tui / pi-ai / chord 都不是「the agent」。

第二轮（我坚持等式，等式反而反证）：等式（LLM + 工具 + loop = agent）本身成立，是通行定义（Anthropic《Building Effective Agents》：预编排路径 = workflow，LLM 自主决定路径 = agent）。但代入仓库后结论反转：等式三项按最小实现只占约 1–2%；即便把 pi-ai 全部 22.1k 行算进「接大模型」也只占约 17%；其余 83%（会话持久化 / compaction / lane / 分支 / TUI / 扩展 / skills）没有一项出现在等式里。等式内部还有 harness 现象：「接大模型」最小实现约 500 行 → 生产实现 22.1k 行，差值不增加 agent 度、只增加可用性。

## 结论

- 「构成了 agent」（等式成立）与「是一个 agent 仓库」（工程主体描述）是两个命题。
- README 双词并用、分工清晰：产品叫 "self extensible coding agent"（pi-coding-agent 包），项目叫 "Pi agent harness"（仓库整体）。产品是 agent，仓库是 harness，各说各的层面。
- 学习路线映射：等式三项（loop + 工具接口 + LLM 适配）= 阶段 2 精读范围；等式之外的 83% = 阶段 3–6 主体。这场辨析的副产品是整条学习路径的粗地图。
- 稳定理解蒸馏进 [notes/architecture/agent-vs-harness.zh.md](../notes/architecture/agent-vs-harness.zh.md)。

## 附带收获（README 精读事实）

- README 包清单只列 6/11 个目录；未列的 5 个：client / protocol / server（「远程会话」轴）、evals（private 评估包）、session-backends（实为目录组而非包——无根 package.json，内含 sqlite-node 子包）。第 4 步画拓扑须以根 package.json 的 workspaces 字段为准。
- pi 刻意不做内置权限系统，安全边界放 harness 之外：Gondolin（内置工具进 micro-VM、key 留宿主机）/ Plain Docker（整个进程）/ OpenShell（策略沙箱）；Gondolin 只路由内置工具，其他扩展的工具默认仍在宿主机执行。
- min-release-age=2 防新鲜投毒（恶意新版本在发现 / 下架前被依赖解析拉取）；save-exact=true 锁已审查版本。
- 会话 JSONL 是产品级数据资产：作者公开自己的 pi-mono 会话到 HuggingFace，锚点实验逐行读的格式即此。
