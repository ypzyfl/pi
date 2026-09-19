# 结构化系统提示词：system 消息承载 prompt 与工具声明

状态：已对照验证（2026-09-19 对照 agent-loop.ts / agent & ai types.ts / ai utils/transcript.ts / coding-agent system-prompt.ts & agent-session.ts & session-manager.ts / session-format.md；system 消息落盘由真实会话文件观测证实）

## 事实源（链接，不复述）

- [packages/ai/src/types.ts](../../../packages/ai/src/types.ts)：`SystemMessage`（L484-500）、`Context`（L610-614）、`TranscriptContext`
- [packages/ai/src/utils/transcript.ts](../../../packages/ai/src/utils/transcript.ts)：`normalizeContext` / `getCurrentSystemMessage` / `getCurrentTools` / `getToolStateChanges` / `resolveTranscript` / `collapseSystemMessages` / `toToolDeclaration`
- [packages/agent/src/agent-loop.ts](../../../packages/agent/src/agent-loop.ts)：`declareToolChanges`（L291）、`streamAssistantResponse`（L339）
- [packages/agent/src/types.ts](../../../packages/agent/src/types.ts)：`AgentContext`（L434）、`AgentState.systemPrompt`（L348）、`StreamFn`（L33）
- [packages/coding-agent/src/core/system-prompt.ts](../../../packages/coding-agent/src/core/system-prompt.ts)：`buildSystemPromptSections`（L121）、`diffSystemPromptSections`（L204）
- [packages/coding-agent/src/core/agent-session.ts](../../../packages/coding-agent/src/core/agent-session.ts)：`_preparePromptAndToolLoadout`（L1115）、`_installAgentNextTurnRefresh`（L565）
- [packages/coding-agent/src/core/session-manager.ts](../../../packages/coding-agent/src/core/session-manager.ts)：`appendMessage`（L1082）、`_persist`（L1040）
- [session-format.md](../../../packages/coding-agent/docs/session-format.md)：`SessionMessageEntry` 的 system 消息形态（L226-236）

## 它是什么（≤5 句）

pi 把「系统提示词」和「工具声明」从运行时内存字段（旧 `AgentContext.systemPrompt`、`AgentToolResult.addedToolNames`）改成了 transcript 里的 `system` 消息。系统提示词被结构化成「命名段」（sections），工具声明用 `toolsAdded`/`toolsRemoved` 表达；变化时只 diff 出变化了的段、追加一条 patch 消息落盘；发给模型前重放所有 system 消息得到「当前提示词 + 当前工具集」。持久层是 append-only（A 和 B 都在文件里），投影层是替换（重放后只有 B）。

## 核心机制

### 1. 提示词结构化 + 工具声明消息化

系统提示词不是一个整块文本，而是 `buildSystemPromptSections` 产出的命名段集合：`preamble`（角色定义）、`tools`、`rules`、`docs`、`skills`、`cwd`、`project_context` 等。每段渲染成 `<name>...</name>`。工具声明是 system 消息上的 `toolsAdded` / `toolsRemoved` 两个字段。

### 2. 变化 = diff + 追加 patch，而非整体替换

切换智能体等导致提示词变化时，`_preparePromptAndToolLoadout` 对比「重放当前 transcript 得到的 sections」与「目标 sections」，`diffSystemPromptSections` 只产出变化了的段（同名段新值、消失段 `null`）：

```1124:1128:packages/coding-agent/src/core/agent-session.ts
	const sections = diffSystemPromptSections(
		getCurrentSystemMessage(messages)?.sections ?? {},
		buildSystemPromptSections(options),
	);
	return sections ? { role: "system", content: "", sections, timestamp: Date.now() } : undefined;
```

产物是一条 **patch system 消息**：`content` 空、`sections` 只含变化段。注入时机两处：`before_agent_start`（`unshift` 到消息前）与每个 turn 之间（`prepareNextTurn`）。工具集差异由 agent 层的 `declareToolChanges` 在每次请求前折算成 system 消息的 `toolsAdded`/`toolsRemoved`。

### 3. 落盘：append-only，但首条 assistant 才真正 flush

system 消息作为 `message` entry（`role:"system"`）落盘（`agent-session._handleAgentEvent` 把 `system/user/assistant/toolResult` 都走 `appendMessage`）。文件写入有闸门：`session-manager._persist` 检查「是否已有 assistant 消息」，没有就只进内存不写文件，出现首条 assistant 时才 `"wx"` 排他创建文件并全量写入——所以「避免空会话文件」机制在 system 消息出现后依然成立。

### 4. 投影给模型：重放，按段覆盖

发给模型前，`getCurrentSystemMessage` 按序重放所有 system 消息：

- **sections**：同名段**覆盖**（`Map.set`），`null` 删除
- **content**：**追加**（`join("\n\n")`）
- **tools**：重放增删

`resolveTranscript` 决定形态：支持 mid-conversation system 的 provider 就地保留；否则 `collapseSystemMessages` 重放成一条 leading system message。

### 5. prompt caching 断点

对 `cacheControlFormat: "anthropic"` 的模型，`applyAnthropicCacheControl` 打三个 `cache_control: ephemeral` 断点——system 消息、最后一个 tool、最后一条对话消息——把 prompt 分成「系统提示词段 / 工具段 / 历史段」三段独立缓存。

## 关键取舍：持久层「追加」vs 投影层「替换」

这是本篇最重要的认知，直接回答「A 到底还在不在」：

| 层面 | 系统提示词 A（切换后被 B 取代） |
|---|---|
| JSONL 文件 | **还在**（append-only，A 和 patch 都在，可追溯） |
| 发给模型的上下文 | **没了**（重放按段覆盖，只剩 B） |

设计意图：文件里保存完整历史（能回放「切换前模型看到的是 A」），但上下文里永远只呈现「当前态」（B 盖住 A）——既不会丢历史，也不会让模型同时看到两套矛盾指令。

## 代价与收益

- **收益**：可追溯（resume 精确还原任意时刻的提示词）、可重放（会话文件自洽）、工具增删对称（`toolsAdded`/`toolsRemoved`，旧 `addedToolNames` 只能加不能删）。
- **代价**：落盘体积变大（提示词 + 每个工具的完整定义都写进 JSONL）、实现复杂度高（diff + 重放 + 段命名管理）、切提示词导致前缀缓存失效（靠 cache_control 断点把失效范围压到「系统提示词段」）。

## 与相邻单元的关系

- 上游语义见 [dual-runtime-semantics.zh.md](dual-runtime-semantics.zh.md)（system 消息化重构的动机——把「模型看到什么」纳入 append-only 事实源）。
- 落盘形态与 entry 链见 [session-message-flow.zh.md](session-message-flow.zh.md)「system 消息落盘」。
- 对业界主流做法（内存态整体替换）的对照见 [prompt-change-practices.zh.md](../architecture/prompt-change-practices.zh.md)。

## 验证方式

- 真实会话文件：新会话首条为 `{"type":"message","message":{"role":"system","sections":{...},"toolsAdded":[...]}}`，文件体积较旧版显著增大。
- 源码链路：`buildSystemPromptSections` → `diffSystemPromptSections` → `_preparePromptAndToolLoadout` → `declareToolChanges` → `getCurrentSystemMessage` 重放 → `resolveTranscript` 投影。

## 遗留问题

- 支持 mid-conversation system 的 provider 就地把多条 system 消息发送给模型时，token 与语义的确切表现（对比 collapse 成一条）未逐一验证。
- `cache_control` 断点缓存与 sections 内部结构（preamble 在最前）叠加时，preamble 变化导致的「系统提示词段整体失效」是否可进一步细分，未深究。
