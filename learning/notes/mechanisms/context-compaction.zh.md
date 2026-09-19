# 上下文自动压缩：对历史做结构化摘要，为回复腾出空间

状态：已对照验证（2026-09-19 对照 compaction.md / compaction.ts / agent-session.ts / session-manager.ts / system-prompt.ts；阈值公式、同步阻塞、系统提示词不受影响三点均落回源码）

## 事实源（链接，不复述）

- [compaction.md](../../../packages/coding-agent/docs/compaction.md)：官方文档（触发、五步流程、split turn、配置、扩展 hook）
- [compaction.ts](../../../packages/coding-agent/src/core/compaction/compaction.ts)：压缩纯函数（`shouldCompact` / `findCutPoint` / `generateSummaryWithUsage` / `compact`）
- [branch-summarization.ts](../../../packages/coding-agent/src/core/compaction/branch-summarization.ts)：分支摘要（姊妹机制）
- [agent-session.ts](../../../packages/coding-agent/src/core/agent-session.ts)：`_checkCompaction` / `_runAutoCompaction`（触发与编排）
- [session-manager.ts](../../../packages/coding-agent/src/core/session-manager.ts)：`CompactionEntry` / `sessionEntryToContextMessages`（落盘与重放）
- [system-prompt.ts](../../../packages/coding-agent/src/core/system-prompt.ts)：系统提示词构建（压缩不触碰，只在压缩边界被快照）

## 它是什么（≤5 句）

LLM 上下文窗口有限。会话过长时，pi 把较旧的一段对话历史交给 LLM 压缩成一条结构化摘要，只保留近期消息，为后续回复腾出空间。触发由阈值公式自动完成（也可 `/compact` 手动）。摘要是独立的一次 LLM 调用（独立 routing session、不写 prompt cache），同步阻塞主对话直到完成。压缩只动对话历史，系统提示词与工具声明不受影响，且在压缩边界被完整快照保存。

## 核心机制

### 1. 触发：一条阈值公式

`shouldCompact`（compaction.ts L250）判断 `contextTokens > contextWindow - reserveTokens`。`reserveTokens` 默认 16384，为 LLM 回复预留空间；`keepRecentTokens` 默认 20000，决定保留多少近期消息。两者在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 的 `compaction` 段配置，另支持 `modelOverrides` 按 `provider/modelId` 微调。

### 2. 三个检查时点

自动压缩在三个位置同步触发（agent-session.ts）：

- 工具批执行完、结果追加后、下一个 assistant 回复前（`_checkCompaction` 在 `agent_end` 后调用，L1212）
- 新用户提示发送前（L1337）
- 低层 agent run 结束后（L558）

### 3. 压缩五步

1. **找切点**：从最新消息向前累积 token 估算，直到达到 `keepRecentTokens`（`findCutPoint`，compaction.ts L418）
2. **提取消息**：收集从上次保留边界到切点的消息
3. **生成摘要**：调用 LLM 按固定结构化格式总结，若已有历史摘要则迭代合并
4. **追加条目**：保存 `CompactionEntry`（摘要 + `firstKeptEntryId`）
5. **重建上下文**：后续请求用「摘要 + `firstKeptEntryId` 之后的保留消息」

### 4. 摘要是一次独立调用

`completeSummarization`（compaction.ts L591）是所有摘要调用的统一出口：`sessionId ?? uuidv7()` 生成独立 routing session、`cacheRetention: "none"` 不写 prompt cache。摘要结果不作为 assistant 消息插进对话流，而是存成 `CompactionEntry`，重建上下文时作为 `compactionSummary` 消息注入。

### 5. 同步阻塞

`_runAutoCompaction` 内部 `await this._runDefaultCompaction(...)`，上层 `await this._runAutoCompaction(...)`。主对话必须等摘要生成完毕（或被用户取消——压缩期间持有 `AbortController`）才继续。

### 6. 系统提示词与工具不受影响

`getMessageFromEntryForCompaction`（compaction.ts L93）对 `role === "system"` 的消息返回 `undefined`——系统消息被当作「提示词状态」而非「对话」排除在摘要之外。同时 `CompactionEntry.systemMessage` 字段（session-manager.ts L88）在压缩边界用 `getCurrentSystemMessage` 快照保存「完整提示词 + 工具状态」，重建上下文时随摘要一起重放，确保跨压缩、跨重启都能恢复出完整正确的提示词。

### 7. split turn

单轮超过 `keepRecentTokens` 时切点落在轮次中间，生成两份摘要（history summary + turn prefix summary）再合并（compaction.ts L899）。

### 8. overflow 恢复

请求因上下文溢出失败时，先移除失败/截断的 assistant 消息，压缩后重试被中断的 turn 一次（`_checkCompaction` 的 Case 1，agent-session.ts L2289）。

## 关键取舍

压缩是「用一次额外 LLM 调用，换取后续每轮更短的上下文」。代价是摘要调用本身的 token 与时间（且同步阻塞），收益是上下文不超限、且比「粗暴截断」保留更多语义。摘要 usage 单独计入 session 总量，不混入某条对话消息。

## 与相邻单元的关系

- 依赖 [structured-system-prompt.zh.md](structured-system-prompt.zh.md)：压缩边界的 `systemMessage` 快照正是「system 消息承载提示词与工具」这一重构的直接延伸。
- 落盘进会话树的 `CompactionEntry`，与 [session-message-flow.zh.md](session-message-flow.zh.md) 的 entry 链同源。
- 分支摘要（branch summarization）是 `/tree` 导航时的姊妹机制，共用同一摘要格式与文件跟踪，见官方文档。

## 易误判的边界（本次逐一澄清）

- 「摘要调用不影响主对话」→ 准确说法：摘要调用独立（不污染对话流/缓存），但压缩必然改变主对话后续的上下文——历史被摘要替代，这正是压缩的目的。
- 「压缩是后台/异步进行」→ 实际是同步阻塞，主对话 `await` 等摘要完成才继续。
- 「压缩可能丢系统提示词/工具」→ 实际 system 消息被显式排除，且压缩边界完整快照 `systemMessage`。
- 「压缩 = 简单截断历史」→ 实际是 LLM 生成结构化摘要（Goal / Progress / Decisions / Next Steps 等）而非丢内容。

## 验证方式

- 源码链路：`shouldCompact` → `_checkCompaction` → `_runAutoCompaction` → `compact` → `generateSummaryWithUsage` → `completeSummarization`。
- 阈值与默认值已在 compaction.md「Settings」节与 compaction.ts `DEFAULT_COMPACTION_SETTINGS` 对齐。

## 遗留问题

- 压缩如何改写会话树的 `parentId` 分支结构（追加在主线还是新开分支）未逐一验证——见 [questions.zh.md](../../questions.zh.md) Q5。
- `modelOverrides` 的精确匹配规则（大小写、含斜杠的 modelId、fallback 逐层）只在文档读到，未实操。
