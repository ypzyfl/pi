# 扩展能力全景：1 个被动补丁层 + 5 条主动轴

状态：草稿（2026-09-17 对照 [extensions/types.ts](../../../packages/coding-agent/src/core/extensions/types.ts) 的 `ExtensionAPI` / `ExtensionContext` / `ExtensionCommandContext` / `ExtensionUIContext`、[event-bus.ts](../../../packages/coding-agent/src/core/event-bus.ts)、[examples/extensions/subagent/index.ts](../../../packages/coding-agent/examples/extensions/subagent/index.ts)；能力分类、权限分层均逐点对照源码）

本文是 [extension-hooks.zh.md](extension-hooks.zh.md)（hook 点梳理）的姊妹篇与总纲：前者只展开「被动补丁层」这一层，本文给出扩展系统**完整的能力全景**。

## 事实源（链接，不复述）

- [extensions/types.ts](../../../packages/coding-agent/src/core/extensions/types.ts)（`ExtensionAPI` L1252-1506、`ExtensionContext` L309-349、`ExtensionCommandContext` L355-389、`ExtensionUIContext` L133-284）
- [extensions/runner.ts](../../../packages/coding-agent/src/core/extensions/runner.ts)（`ExtensionRunner`）
- [event-bus.ts](../../../packages/coding-agent/src/core/event-bus.ts)（`EventBus`，基于 `EventEmitter` 的 pub/sub）
- [examples/extensions/subagent/index.ts](../../../packages/coding-agent/examples/extensions/subagent/index.ts)（子智能体：`registerTool` + spawn 独立 `pi` 进程）
- [sdk.ts](../../../packages/coding-agent/src/core/sdk.ts)（hook 接线）

## 一句话总结

**pi 扩展的能力 = 1 个被动补丁层（hook）+ 5 条主动轴（注入 / 驱动 / 重配 / 生命周期 / UI 协作）。**

- 被动补丁层：hook（事件订阅）——观察/修改/拦截**既有数据流**，不创造新东西。
- 主动轴：注入 / 驱动 / 重配 / 生命周期 / UI 协作——主动改变状态、增加能力、操纵会话。

## 五条主动轴

### 轴 1：能力注入（给 pi 增加新能力/入口/资源）

| API | 作用 |
|---|---|
| `registerTool` | 给**模型**增加新动作（任意 Node.js 代码） |
| `registerProvider` | 给**模型**增加/替换来源，可自定义 `streamSimple` 接管「调 LLM」 |
| `registerCommand` | 给**用户**增加 slash 命令（`/xxx`） |
| `registerShortcut` | 注册键盘快捷键 |
| `registerFlag` / `getFlag` | 注册/读取 CLI flag |
| `registerMessageRenderer` / `registerEntryRenderer` | 自定义消息/entry 渲染 |
| `registerMarkdownTransformer` | Markdown 渲染前转换 |

`registerTool` 是「深度影响」的**核心杠杆**：工具的 `execute` 里可写任意 Node.js 代码。子智能体（[subagent 示例](../../../packages/coding-agent/examples/extensions/subagent/README.md)）正是靠它——`registerTool` 注册一个 `subagent` 工具，内部 `spawn` 一个独立的 `pi` 进程（`--mode json -p --no-session`），靠**进程边界**天然获得上下文隔离、工具隔离、权限隔离，扩展只需做「进程编排」。

### 轴 2：会话驱动（扩展主动当「第二个用户」）

| API | 作用 |
|---|---|
| `sendMessage(msg, { triggerTurn, deliverAs })` | 发自定义消息，可选触发 turn、指定 steer/followUp/nextTurn |
| `sendUserMessage(text, { deliverAs })` | 发用户消息，**总是触发 turn** |
| `appendEntry(customType, data)` | 追加持久化 entry（状态存储，不进 LLM） |
| `exec(cmd, args)` | 执行 shell 命令 |

扩展不仅能「被动响应事件」，还能**主动向 agent 发消息**——`triggerTurn: true` 时等于扩展自己驱动新一轮对话。

### 轴 3：运行时重配（动态改 agent 配置）

| API | 作用 |
|---|---|
| `setActiveTools` / `getActiveTools` / `getAllTools` | 运行时增删工具 |
| `setModel` | 运行时换模型（不改默认） |
| `setThinkingLevel` / `getThinkingLevel` | 运行时调思考等级 |

允许扩展在会话中途动态调整 agent 的「器官」（工具）和「大脑」（模型）。

### 轴 4：会话生命周期（最深的一层，命令上下文专属）

命令 handler 拿到的 `ExtensionCommandContext` 比普通 `ExtensionContext` 多几个**高危、高权限**操作：

| API | 作用 |
|---|---|
| `ctx.newSession()` | 新建会话 |
| `ctx.fork(entryId)` | 分叉会话树 |
| `ctx.navigateTree(targetId)` | 导航到会话树任意节点 |
| `ctx.switchSession(path)` | 切换会话文件 |
| `ctx.reload()` | 重载扩展/技能/提示词/主题 |
| `ctx.waitForIdle()` / `abort()` / `shutdown()` / `compact()` | 等待空闲 / 中止 / 退出 / 触发压缩 |

**权限分层是有意的**：`newSession`/`fork`/`navigateTree`/`switchSession`/`reload` 只出现在**命令上下文**（用户主动调 `/xxx` 时），而不在工具的 `execute` 里——「新建/分叉/切换会话」是用户级别的高危操作，不该由模型的一句工具调用就触发。

### 轴 5：UI 控制 + 扩展间协作

`ctx.ui`（`ExtensionUIContext`）提供完整界面控制：

| 类别 | API |
|---|---|
| 对话框 | `select` / `confirm` / `input` / `editor` / `notify` / `custom` |
| 终端输入 | `onTerminalInput`（截获原始按键） |
| 状态/指示 | `setStatus` / `setWorkingMessage` / `setWorkingIndicator` / `setHiddenThinkingLabel` |
| 组件 | `setWidget` / `setFooter` / `setHeader` / `setTitle` |
| 编辑器 | `setEditorText` / `getEditorText` / `pasteToEditor` / `setEditorComponent` / `addAutocompleteProvider` |
| 主题 | `getAllThemes` / `getTheme` / `setTheme` |

以及 `pi.events`（`EventBus`）——基于 `EventEmitter` 的 pub/sub 通道，用于**扩展之间通信**。

## 被动补丁层：hook（独立于五轴）

`on(event, handler)` 订阅 30+ 事件，覆盖「一问一答」的 4 个边界层（输入 / agent / 工具 / provider），详见 [extension-hooks.zh.md](extension-hooks.zh.md)。它与五轴的性质差异是：

- **五轴 = 主动**：主动注入能力、主动驱动会话、主动重配、主动操纵会话树；
- **hook = 被动**：不在流程里凭空创造任何东西，只在既有数据流经过时**观察、修改、拦截**。

## 完整能力地图

| # | 类别 | 代表 API | 性质 |
|---|---|---|---|
| 1 | 能力注入 | `registerTool` / `registerProvider` / `registerCommand` / `registerShortcut` / `registerFlag` / 渲染器 | 主动 |
| 2 | 会话驱动 | `sendMessage` / `sendUserMessage` / `appendEntry` / `exec` | 主动 |
| 3 | 运行时重配 | `setActiveTools` / `setModel` / `setThinkingLevel` | 主动 |
| 4 | 会话生命周期 | `ctx.newSession` / `fork` / `navigateTree` / `switchSession` / `reload` | 主动 |
| 5 | UI + 协作 | `ctx.ui.*` / `pi.events` | 主动 |
| — | **被动补丁层（hook）** | `on("input" / "context" / "before_provider_*" / …)` | **被动** |

## 与扩展能力相关的关键洞察

1. **`registerTool` / `registerProvider` 归入轴 1（能力注入）**：它们与 `registerCommand` 一样，本质都是「给 pi 注入新东西」——tool 给模型新动作、provider 给模型新来源、command 给用户新命令。
2. **hook 单独成层**：其余五类都是「主动改变状态/注入能力」，hook 是「被动响应」。二者不是同一维度，故 hook 不夹在五轴之间，而是独立为「补丁层」。
3. **「深度影响」的杠杆不是 hook，而是 `registerTool`**：子智能体靠的是「注册工具 + spawn 独立进程」，不是「拦截事件」。hook 只负责在既有流程上做细粒度、可插拔的补丁。

## 验证方式

- `read_file` 读 `extensions/types.ts` 的 `ExtensionAPI` / `ExtensionContext` / `ExtensionCommandContext` / `ExtensionUIContext` 四个接口
- `read_file` 读 `event-bus.ts` 全文、`examples/extensions/subagent/index.ts` 核心段（`registerTool` + `spawn`）
- `search_content` 搜 `registerCommand` / `sendMessage` / `setActiveTools` 定位 API 实现

## 遗留问题

- `ExtensionContext` 的只读信息面（`sessionManager` / `modelRegistry` / `scopedModels` / `getContextUsage` 等）本篇只归入「上下文能力」未展开，留待阶段 5。
- `registerProvider` 的 `streamSimple` 自定义（完全接管「调 LLM」）与 `custom-provider` 示例的深入分析，留待阶段 5。
