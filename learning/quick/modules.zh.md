# 六个桌面端核心包速查

**用法**：这是索引，不是全读源码的清单。遇到问题回来查"去哪个文件"，然后只读那一个文件。

## 速查总表

| 包 | 一句话定位 | 关键文件 | 改它去哪 |
|---|---|---|---|
| **chord** | 应用组合运行时（services、复制状态、RPC、plugins），几乎被所有上层包依赖 | `packages/chord/src/` | 桌面端开发一般不改 |
| **tui** | 差分渲染终端 UI 库，只被 interactive 模式消费 | `packages/tui/src/` | 想自定义 TUI 组件时读 `packages/coding-agent/docs/tui.md` |
| **telemetry** | 厂商中立遥测契约，无依赖 | `packages/telemetry/src/` | 接遥测时读 `packages/telemetry/README.md` |
| **ai** | 把多 provider 抹平成一个 API | `packages/ai/src/stream.ts`、`packages/ai/src/providers/`、`packages/ai/src/models.generated.ts`（生成物，**红线不可手改**） | 改模型目录走 `packages/ai/scripts/generate-models.ts` |
| **agent** | 通用 agent runtime | `packages/agent/src/agent-loop.ts`（循环）、`packages/agent/src/agent.ts`（有状态包装）、`packages/agent/src/harness/`（大子系统） | 想改运行时行为读 `agent-loop.ts` |
| **coding-agent** | 产品层 CLI，AgentSession 组装中枢 | `packages/coding-agent/src/core/agent-session.ts` | 见下表 |

## coding-agent 包关键文件速查

coding-agent 是产品层，文件最多、最常碰。按职责分：

| 职责 | 关键文件 | 干什么 |
|---|---|---|
| 组装中枢 | `src/core/agent-session.ts` | AgentSession：装配工具/会话/扩展/模型/设置 |
| 模型运行时 | `src/core/model-runtime.ts` | ModelRuntime：`implements ai.Models`，叠加认证 / 自定义模型（`~/.pi/agent/models.json`）/ 扩展 provider / 缓存 |
| 内置工具 | `src/core/tools/` | read / bash / powershell / edit / write / grep / find / ls（含 renderers） |
| 编辑安全 | `src/core/tools/edit.ts`、`edit-diff.ts` | edit 工具的安全设计 |
| 系统提示词 | `src/core/system-prompt.ts` | 系统提示词装配 |
| 会话管理 | `src/core/session-manager.ts` | 会话 JSONL 读写、分支、树导航 |
| 设置 | `src/core/settings-manager.ts` | 全局/项目设置 |
| 信任 | `src/core/trust-manager.ts` | 项目信任（影响 `.pi/` 加载） |
| Slash 命令 | `src/core/slash-commands.ts` | 内置 slash 命令 |
| Skills 加载 | `src/core/skills.ts` | skill 发现与按需加载 |
| 扩展加载 | `src/core/extensions/` | extension 发现与加载机制 |
| 压缩 | `src/core/compaction/` + `docs/compaction.md` | 会话压缩 |
| 运行模式 | `src/modes/` | interactive / rpc / print 三模式入口 |

## 内置工具一览

| 工具 | 作用 | 默认开启 |
|---|---|---|
| `read` | 读文件 | 是 |
| `write` | 创建或覆盖文件 | 是 |
| `edit` | 给文件打补丁 | 是 |
| `bash` | 跑 shell 命令 | 是 |
| `grep` | 正则搜内容 | 否（tool options 开启） |
| `find` | 按名找文件 | 否 |
| `ls` | 列目录 | 否 |
| `powershell` | Windows 下跑 PowerShell 命令 | 平台相关 |

## 扩展资源五种形态

| 形态 | 是什么 | 放哪 | 加载时机 |
|---|---|---|---|
| **extension** | TS 模块，注册工具/命令/事件钩子 | `.pi/extensions/*.ts` 或 `~/.pi/agent/extensions/*.ts` | 启动时 + `/reload` 热重载 |
| **skill** | `SKILL.md` 能力包，模型按需读 | `.pi/skills/*/SKILL.md` 或 `~/.pi/agent/skills/` | 描述进系统提示词，正文按需加载 |
| **prompt template** | `.md` 片段，slash 命令展开 | `.pi/prompts/*.md` 或 `~/.pi/agent/prompts/*.md` | 输入 `/name` 时展开 |
| **theme** | 主题 | `.pi/themes/` 或 settings | 启动时 |
| **pi package** | 把上述资源打包经 npm/git 分发 | `pi install npm:...` 或 `git:...` | 安装到 `~/.pi/agent/npm/` 或 `git/` |

## dogfooding 实例（.pi/ 目录）

本仓库用 pi 开发 pi，`.pi/` 目录是现成实例：

| 文件 | 形态 | 作用 |
|---|---|---|
| [.pi/extensions/tps.ts](../../.pi/extensions/tps.ts) | extension | 统计每回合 token/s，`agent_end` 时 notify |
| [.pi/extensions/prompt-url-widget.ts](../../.pi/extensions/prompt-url-widget.ts) | extension | 自定义 widget |
| [.pi/extensions/redraws.ts](../../.pi/extensions/redraws.ts) | extension | 重绘相关 |
| [.pi/extensions/import-repro.ts](../../.pi/extensions/import-repro.ts) | extension | 导入复现 |
| [.pi/skills/add-llm-provider.md](../../.pi/skills/add-llm-provider.md) | skill | 添加 LLM provider 的工作流 |
| [.pi/skills/interactive-testing.md](../../.pi/skills/interactive-testing.md) | skill | 在 tmux 受控终端测试 pi 交互模式（0.85.1 新增） |
| [.pi/skills/release.md](../../.pi/skills/release.md) | skill | 发布 pi 的完整流程（0.85.1 新增） |
| [.pi/prompts/cl.md](../../.pi/prompts/cl.md) | prompt template | 发布前审计 changelog |
| [.pi/prompts/wr.md](../../.pi/prompts/wr.md) | prompt template | 端到端完成任务 |
| [.pi/prompts/is.md](../../.pi/prompts/is.md) | prompt template | 分析 GitHub issue |
| [.pi/prompts/pr.md](../../.pi/prompts/pr.md) | prompt template | 从 URL review PR |
| [.pi/prompts/deslop.md](../../.pi/prompts/deslop.md) | prompt template | 去除冗余文风 |
| [.pi/prompts/sa.md](../../.pi/prompts/sa.md) | prompt template | shell 别名 |

**最佳学习姿势**：精读 `tps.ts`（最小有用 extension，47 行）+ `cl.md`（最小 prompt template），照着改。

## 官方文档地图（按需查）

文档全在 `packages/coding-agent/docs/`，按场景查：

| 想干什么 | 读哪篇 |
|---|---|
| 装与用 | `quickstart.md`、`usage.md` |
| 写扩展 | `extensions.md`（3024 行，完整 API） |
| 写 skill | `skills.md` |
| 写 prompt template | `prompt-templates.md` |
| 打包分发 | `packages.md` |
| 自定义模型 | `models.md`、`providers.md`、`custom-provider.md` |
| 会话格式 | `session-format.md`、`sessions.md` |
| 设置 | `settings.md` |
| 安全 | `security.md`、`containerization.md` |
| 压缩 | `compaction.md` |
| TUI 自定义 | `tui.md`、`themes.md`、`keybindings.md` |
| 程序化嵌入 | `sdk.md`、`rpc.md`、`json.md` |
| Windows 平台 | `windows.md`、`terminal-setup.md`、`shell-aliases.md` |
| 开发贡献 | `development.md` |
| 环境变量 | `environment-variables.md` |
