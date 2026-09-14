# 学习区索引

本区是系统学习本仓库（pi / pi-mono）的工作区，结构与约定仿照 deepseek-harness 仓库的 learning 区。读什么、按什么顺序由 [learning-path.zh.md](learning-path.zh.md) 回答；学习过程如何记录、笔记如何组织由 [method.zh.md](method.zh.md) 回答；本页只做两件事——列出本区文件的入口，登记学习进度。两条通道可选：快速通道（2-4 小时上手扩展开发，聚焦桌面端）见 [quick/](quick/README.zh.md)；深度方案（七阶段源码级理解）见 [learning-path.zh.md](learning-path.zh.md)。

## 文件索引

| 位置 | 内容 |
|---|---|
| [learning-path.zh.md](learning-path.zh.md) | 深度方案：读什么、按什么顺序（七阶段 + 入门锚点实验） |
| [quick/](quick/README.zh.md) | 快速通道：2-4 小时上手扩展开发（五步速通，聚焦桌面端，忽略远程会话轴） |
| `plan/` | 各阶段执行路线与逐步勾选进度，见 [plan/stage-1.zh.md](plan/stage-1.zh.md)、[plan/stage-2.zh.md](plan/stage-2.zh.md) |
| [method.zh.md](method.zh.md) | 学习方法：宪法、目录规则、记录方式（journal 两级 / questions 三态 / map 两级）、笔记模板 |
| [questions.zh.md](questions.zh.md) | 开放问题池（三态流转；初始六问来自建区勘察） |
| [map.zh.md](map.zh.md) | 认知地图（整体心智模型快照；初始版含已验证的依赖拓扑图） |
| [AGENTS.zh.md](AGENTS.zh.md) | 根 AGENTS.md 的全文中文对照翻译（10 节；第 1 步精读产出） |
| [CONTRIBUTING.zh.md](CONTRIBUTING.zh.md) | 根 CONTRIBUTING.md 的全文中文对照翻译（9 节；第 1 步精读产出） |
| `experiments/` | 动手实验（编号文件，完成后登记在此） |
| `journal/` | 认知事件原始记录（`YYYY-MM-DD-NN-slug.zh.md`） |
| `notes/` | 认知单元（每篇一个可独立复述的理解）：architecture / mechanisms / modules |
| `guide/` | 指导性手册（长期反复照做的操作手册） |
| `design/` | 设计方案 |
| `scripts/` | 可跑脚本 |

以上按需生长的目录中，guide / design / scripts 暂无文件，规则见 [method.zh.md](method.zh.md)「目录结构」；experiments / journal / notes 的现有记录见下三节。

### experiments 现有记录

| 实验 | 一句话 |
|---|---|
| [experiments/001-session-anchor.zh.md](experiments/001-session-anchor.zh.md) | 会话锚点：一次对话 = 一棵 JSONL 事件树；三类基本 entry、id/parentId 链、toolCall ↔ toolCallId 关联、多轮 append 均实测 |

### journal 现有记录

- [2026-09-08-01-readme-harness](journal/2026-09-08-01-readme-harness.zh.md)：README 精读（阶段 1 第 1 步）途中辨「仓库是 agent 还是 harness」——等式成立但论域不同：产品是 agent，仓库是 harness。

### notes 现有记录

| 笔记 | 一句话 |
|---|---|
| [notes/architecture/agent-vs-harness.zh.md](notes/architecture/agent-vs-harness.zh.md) | 产品是 agent、仓库是 harness：构成等式（LLM + 工具 + loop）只占全仓约 0.5–17%，其余是 harness |
| [notes/architecture/pi-architecture-overview.zh.md](notes/architecture/pi-architecture-overview.zh.md) | 架构级认知总览：设计理念五信条、11 包分层（ASCII / mermaid / 邻接表三投影）、运行时双栈、消息生命周期、AgentSession 组装、会话树、扩展体系 |
| [notes/architecture/classic-stack-overview.zh.md](notes/architecture/classic-stack-overview.zh.md) | 经典内存栈（生产路径）聚焦视图：6 包依赖、静态结构、数据流与事件流、AgentSession 组装、会话树、工具与 edit 安全设计、三模式、扩展体系、关键文件速查——学习主线文档 |
| [notes/mechanisms/session-message-flow.zh.md](notes/mechanisms/session-message-flow.zh.md) | 一次「一问两调一答」对话在会话 JSONL 里的完整 entry 链与工具调用关联（含基础 / 全量两图） |
| [notes/mechanisms/test-isolation.zh.md](notes/mechanisms/test-isolation.zh.md) | 测试隔离：e2e 激活开关是环境变量与 auth.json 两条钥匙入口，test.sh 双层堵死；check 验「形」、test.sh 验「行」 |
| [notes/mechanisms/contribution-gate.zh.md](notes/mechanisms/contribution-gate.zh.md) | 贡献门槛：auto-close 是三个 workflow + 白名单文件的机器执行；lgtmi/lgtm 能力不对称由代码强制、审批状态进 git |
| [notes/mechanisms/dual-runtime-semantics.zh.md](notes/mechanisms/dual-runtime-semantics.zh.md) | 双栈语义：经典内存栈「先做事后记账」（内存即真相，JSONL 是旁路日志）vs durable 持久化栈「先记账再做事」（accept/drive 操作状态机）；durable 已发布、按场景分工、无替换承诺 |
| [notes/mechanisms/test-sh-on-windows.zh.md](notes/mechanisms/test-sh-on-windows.zh.md) | test.sh 在 Windows：Git Bash 是唯一入口（勿用 WSL bash）；MSYS 对 HOME/USERPROFILE 转换不对称破坏 ~/ 缩写；rolldown 截断诊断案例；套件失败四分类 |

## 进度看板

阶段划分与过关标准以 [learning-path.zh.md](learning-path.zh.md) 为准，本表只登记执行状态与本区产出。状态取值：未开始 / 进行中 / 完成 / 暂缓；完成一行时同步登记产出链接。

| 阶段 | 主题 | 状态 | 本区产出 |
|---|---|---|---|
| 0 | 会话锚点（入门第一步，先于阶段 1） | 完成 | [experiments/001-session-anchor.zh.md](experiments/001-session-anchor.zh.md)（实验 2026-09-07，补录 2026-09-08；工具调用关联用历史会话补验） |
| 1 | 仓库结构与工具链 | 完成（2026-09-12） | 执行路线见 [plan/stage-1.zh.md](plan/stage-1.zh.md)，六步与过关检验全过；三份根文档精读产出（[journal 2026-09-08-01](journal/2026-09-08-01-readme-harness.zh.md)、[agent-vs-harness](notes/architecture/agent-vs-harness.zh.md)、[test-isolation](notes/mechanisms/test-isolation.zh.md)、[contribution-gate](notes/mechanisms/contribution-gate.zh.md)、[AGENTS.zh.md](AGENTS.zh.md) / [CONTRIBUTING.zh.md](CONTRIBUTING.zh.md) 两份对照）；第 2 步文档地图、第 3 步工具链（[test-sh-on-windows](notes/mechanisms/test-sh-on-windows.zh.md)）、第 4 步依赖拓扑（[map.zh.md](map.zh.md)「依赖拓扑」分层修正版）；第 5 步 .pi/ 盘点由用户自行完成，无落盘产出 |
| 2 | ai 包：统一 LLM API | 进行中 | 执行路线见 [plan/stage-2.zh.md](plan/stage-2.zh.md)（2026-09-12 建路：六步，观察先行） |
| 3 | agent 包：运行时核心 | 未开始 | — |
| 4 | coding-agent：产品装配 | 未开始 | — |
| 5 | 扩展体系 | 未开始 | — |
| 6 | 扩展实践 | 未开始 | — |
| 7 | 专项深入（按需） | 未开始 | — |

## 重点学习清单

学习过程中标记出的、需要优先抓重点深入的主题。与进度看板的区别：看板登记「阶段执行状态」，本清单登记「哪些主题值得重点深入」，两者正交。

| 主题 | 为什么重要 | 状态 | 相关产出 |
|---|---|---|---|
| agent-loop 与 AgentHarness 的关系（Q1） | 决定对 agent 包的整体读法：两个入口谁是主路径、谁是未来 | open，阶段 3 首要任务 | [questions.zh.md](questions.zh.md) Q1 |
| 会话 JSONL 树模型 | pi 全部持久能力（分支 / resume / compaction / 导出 / 分享）的地基 | 锚点已建立（[experiments/001](experiments/001-session-anchor.zh.md)，2026-09-08 补录） | [map.zh.md](map.zh.md)「会话模型」；[experiments/001-session-anchor.zh.md](experiments/001-session-anchor.zh.md) |
| 扩展加载机制 | 「最小核心 + 自扩展」是 pi 的立身信条，扩展系统是信条的载体 | 阶段 5 | — |
| 与 dsh 的架构对照 | 已有 dsh 心智模型可迁移：「一切皆插件」（配置式组合） vs「最小核心 + 外挂资源」（资源加载）；对照能凸显两者的真实取舍 | 按需 | [map.zh.md](map.zh.md)「分层心智模型」 |

## 常用命令备忘

只记「跑什么命令看什么」，不存输出全文。Windows 环境注意：`pi-test.sh` 有等价的 `pi-test.ps1` / `pi-test.bat`。

| 想查什么 | 命令 |
|---|---|
| 从源码跑 pi（可在任意目录） | `./pi-test.sh`（Windows：`.\pi-test.ps1`）；传参如 `--list-models`、`-p "..."` |
| 代码检查门禁（改代码后必跑） | `npm run check`（biome + 依赖钉版 + tsgo 等，见根 package.json） |
| 非 e2e 测试（隔离环境，不碰用户配置/凭据） | 仓库根 `./test.sh`；Windows：PowerShell 里 `& "C:\Program Files\Git\bin\bash.exe" ./test.sh`（勿用裸 `bash`，PATH 上是 WSL） |
| 包内单测（vitest 包） | 包根：`node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/<file>.test.ts` |
| 包内单测（tui，node:test） | 包根：`node --test test/<file>.test.ts` |
| 我的会话文件在哪 | `~/.pi/agent/sessions/--<路径转义>--/<timestamp>_<uuid>.jsonl`，格式见 [session-format.md](../packages/coding-agent/docs/session-format.md) |
| tmux 里测交互模式 | 见根 [AGENTS.md](../AGENTS.md)「Testing pi Interactive Mode with tmux」一节的完整流程 |
| 已安装的扩展/技能/模板 | `pi list`（装了什么包）；会话内 `/help` 看 slash 命令 |
| TUI 渲染行或最后发给 LLM 的消息 | 会话内 `/debug`（隐藏 slash 命令），日志 `~/.pi/agent/pi-debug.log` |

## 本区与 dsh learning 区的关系

本区的结构、方法与文件命名仿照 deepseek-harness 的 learning 区（`d:\Tech\Github\deepseek\deepseek-harness\fork\deepseek-harness\learning`），但内容全部针对 pi 重新勘察与制定；两边的通用方法论（journal 两级 / questions 三态 / map 两级 / 版本对齐三问）同源，工程事实互不通用。跨仓库的架构对照见 [map.zh.md](map.zh.md)「分层心智模型」。
