# 阶段 3（agent 包：运行时核心）执行路线与进度

本文是阶段 3 的**执行路线 + 逐步勾选进度**：把 [learning-path.zh.md](../learning-path.zh.md) 阶段 3 的「精读材料 + 动手任务 + 过关检验」拆成可逐步推进的小步骤，并标出每步的验证点与学习区落盘动作。事实源仍是 learning-path.zh.md，本文不重复其内容、只做执行拆解；冲突以 learning-path.zh.md 为准。

过关标准（来自 learning-path.zh.md 完成标志表阶段 3 行）：① 通过三项前置检查（事实源 / append-only + 分支 / 改动落点）；② 能对照源码复述一次 turn 的完整生命周期与继续条件；③ 能说出 `harness/` 各子目录职责一句话；④ 能对「agent-loop 与 AgentHarness 的关系」给出源码级裁决（并回填 questions.zh.md）。

已备认知（阶段 1 产出，本阶段复用而非重读）：
- [dual-runtime-semantics.zh.md](../notes/mechanisms/dual-runtime-semantics.zh.md)——经典内存栈 vs durable 持久化栈的裁决已落盘；本阶段主任务是**到源码里亲手验证它**，而非重新裁决。
- [experiments/001-session-anchor.zh.md](../experiments/001-session-anchor.zh.md)——会话 JSONL 锚点，第 6 步的事件对照物。

## 路线总览（六步，loop 先行）

```
第 1 步  布局盘点 + README 精读 ── src/ 顶层八文件 + harness/ 子目录 + README（嵌入者视角）
第 2 步  词汇与出口 ── types.ts + stream-fn.ts + node.ts + proxy.ts + index.ts
第 3 步  agent-loop.ts 精读 ── 主循环 + 事件序列 + 工具执行 + 停止/继续条件（过关检验②主场）
第 4 步  agent.ts 精读 ── 有状态包装 + 与 loop 的对接 + 队列/生命周期（过关检验②补全）
第 5 步  harness/ 按优先级读 ── compaction/ 必读 + 顶层文件 + Q1 源码级裁决（过关检验③④主场）
第 6 步  动手任务 ── 锚点 JSONL 事件对照 + agent-loop 单测（过关检验①主场）
```

## 第 1 步：布局盘点 + README 精读

目标：先拿到 agent 包的骨架——顶层文件各自职责、harness/ 子目录一撇、README 的嵌入者叙事——再进逐文件精读。

- [x] 盘点 [packages/agent/src](../../packages/agent/src) 顶层七文件 + `search/` 目录（计划原写「八文件」有误，实际 7 个 .ts + 1 个子目录），各记一句话职责：`index.ts`（出口）、`agent-loop.ts`（回合驱动）、`agent.ts`（有状态包装）、`types.ts`（词汇）、`stream-fn.ts`（默认流函数）、`node.ts` / `proxy.ts`（运行环境出口）、`search/`（搜索服务契约）
- [x] 盘点 [packages/agent/src/harness](../../packages/agent/src/harness) 结构：顶层 12 个文件（`agent-harness.ts` / `context.ts` / `events.ts` / `hooks.ts` / `messages.ts` / `system-prompt.ts` / `skills.ts` / `prompt-templates.ts` + `config.ts` / `result.ts` / `telemetry.ts` / `types.ts`）与八个子目录（`session/` / `execution/` / `runtime/`（含 `drive/`）/ `compaction/` / `tools/` / `env/` / `utils/` / `pico3/`（2026-09-19 合并新增，实验性））；对照 [dual-runtime-semantics.zh.md](../notes/mechanisms/dual-runtime-semantics.zh.md) 确认：`compaction/` 必读，`runtime/`（含 `drive/`）/ `session/` / `execution/` 属 durable 栈延后
- [x] [packages/agent/README.md](../../packages/agent/README.md) 通读一遍（517 行，嵌入者视角），重点：Quick Start、Core Concepts（AgentMessage vs LLM Message、Message Flow）、Event Flow（prompt()/continue() 事件序列）、Agent Options / State / Methods、Steering and Follow-up、Low-Level API
- [x] 用一句话回答：README 的「嵌入者」是谁（写 coding-agent 的人），它怎么描述「谁驱动循环、状态放哪、怎么调 LLM」？

## 第 2 步：词汇与出口（过关检验②的前置）

目标：读透 agent 包自有的类型词汇，为精读 loop 备料。核心检查点（均在 [types.ts](../../packages/agent/src/types.ts)，符号已核实在场）：

- [x] `StreamFn`：签名与契约（不抛错、失败编码进流事件 + 最终 stopReason "error"/"aborted"）
- [x] `AgentMessage` / `AgentState`：自定义消息扩展（`CustomAgentMessages` 声明合并）、`AgentState` 的 accessor 复制语义（`tools` / `messages` 赋值即拷贝）
- [x] `AgentContext`（messages + tools；`systemPrompt` 字段已于 2026-09-19 删除，改由 transcript 的 system 消息承载）与 `AgentLoopConfig`（convertToLlm / transformContext / getApiKey / shouldStopAfterTurn / prepareNextTurn / getSteeringMessages / getFollowUpMessages / beforeToolCall / afterToolCall / toolExecution）
- [x] `AgentTool`（label / prepareArguments / execute / replay / executionMode）与 `AgentToolResult`（content / details / usage / terminate；`addedToolNames` 字段已于 2026-09-19 删除，工具声明改为经 transcript 的 system 消息动态声明）
- [x] `AgentEvent` 全集：三类生命周期——agent（`agent_start` / `agent_end`）、turn（`turn_start` / `turn_end`）、message（`message_start` / `message_update` / `message_end`）、tool（`tool_execution_start` / `tool_execution_update` / `tool_execution_end`）
- [x] `ToolExecutionMode`（sequential / parallel）与 `QueueMode`（all / one-at-a-time）两种枚举的语义
- [x] 浏览 [stream-fn.ts](../../packages/agent/src/stream-fn.ts)、[node.ts](../../packages/agent/src/node.ts)、[proxy.ts](../../packages/agent/src/proxy.ts)、[index.ts](../../packages/agent/src/index.ts)：默认流函数从哪来、node/proxy 两个出口差异、`index.ts` 公开导出哪些符号（确认 `AgentHarness` 是否在列，为第 5 步备料）

## 第 3 步：agent-loop.ts 精读（过关检验②主场）

目标：读通 pi 的心脏——双层循环、事件序列、工具执行管线、停止/继续条件。全部检查点均在 [agent-loop.ts](../../packages/agent/src/agent-loop.ts)：

- [x] 四个入口：`agentLoop` / `agentLoopContinue` / `runAgentLoop` / `runAgentLoopContinue` 的分工（EventStream 包装 vs 直接 Promise；带 prompt vs 从现有 context 续跑）
- [x] `runLoop` 双层 while 结构：外层 `while(true)` 处理 follow-up 消息；内层 `while(hasMoreToolCalls || pendingMessages.length>0)` 处理工具调用与 steering 消息——用一句话说清两层各管什么
- [x] 事件序列：从 `agent_start` 到 `agent_end` 完整走一遍，逐事件标出由哪个函数哪一行 `emit`（配合 [session-message-flow.zh.md](../notes/mechanisms/session-message-flow.zh.md) 已有的 entry 链）
- [x] `streamAssistantResponse`：`transformContext`（AgentMessage[]→AgentMessage[]）→ `convertToLlm`（AgentMessage[]→Message[]，**LLM 调用边界**）→ 组装 `Context` → `getApiKey` 解析 → `streamFunction` → 流式事件折叠成 `partialMessage` 并持续 `message_update`，最终 `done`/`error` 结算
- [x] 工具执行三段式：`prepareToolCall`（找 tool → `prepareArguments` → `validateToolArguments` → `beforeToolCall`，可 `block` 或 `immediate`）→ `executePreparedToolCall`（`execute` + `onUpdate` 回调）→ `finalizeExecutedToolCall`（`afterToolCall` 字段级合并）；`sequential` 与 `parallel` 两条路径的执行差异
- [x] 停止/继续条件全集：`stopReason === "error"/"aborted"` 提前返回、`shouldStopAfterTurn`（turn 后优雅停）、`getSteeringMessages`（turn 间隙注入）、`getFollowUpMessages`（本应停止后继续）、`prepareNextTurn`（下一 turn 前替换 context/model/thinking）
- [x] 一个边界保护：`failToolCallsFromTruncatedMessage` 为什么存在（`stopReason === "length"` 时工具参数可能被截断，宁可全部标错也不执行）
- [x] 用一句话回答：一次 turn 的完整生命周期是什么、loop 凭什么决定「继续还是结束」？

## 第 4 步：agent.ts 精读（过关检验②补全）

目标：看清 `Agent` 如何把无状态的 loop 包成有状态对象，并和 loop 的 config/事件对接。全部检查点均在 [agent.ts](../../packages/agent/src/agent.ts)：

- [ ] `Agent` 的职责定位（类注释原话）：owns transcript、emits lifecycle events、executes tools、queueing APIs for steering/follow-up
- [ ] 状态：`_state`（`MutableAgentState`，accessor 复制语义）+ `steeringQueue` / `followUpQueue`（`PendingMessageQueue`，`QueueMode` 决定 drain 方式）+ `listeners`（`subscribe` 订阅集）
- [ ] 队列 API：`steer` / `followUp` / `clearSteeringQueue` / `clearFollowUpQueue` / `hasQueuedMessages` 与 loop 里 `getSteeringMessages` / `getFollowUpMessages` 的对应关系
- [ ] 生命周期：`prompt` / `continue`（`continue` 遇到 assistant 末消息时的 drain 顺序）→ `runPromptMessages` / `runContinuation` → `runWithLifecycle`（`activeRun` 互斥 + `AbortController` + `finishRun`）
- [ ] 上下文与配置：`createContextSnapshot`（从 `_state` 快照）与 `createLoopConfig`（把 `Agent` 的字段/hook 翻译成 `AgentLoopConfig`）——注意 `shouldStopAfterTurn` / `prepareNextTurn` / `getSteeringMessages` 在这里怎么被包一层
- [ ] `processEvents`：事件如何回写到 `_state`（`message_end` 才 `push` 进 `messages`；`tool_execution_start/end` 维护 `pendingToolCalls`；`turn_end` 记 `errorMessage`），再 `await` 所有 listener
- [ ] 失败路径：`handleRunFailure` 手工发出 message_start/end + turn_end + agent_end 一条完整序列
- [ ] 用一句话回答：`Agent`（有状态）与 `runAgentLoop`（无状态）谁拥有 transcript、谁只是执行器？

## 第 5 步：harness/ 按优先级读（过关检验③④主场）

目标：按 [dual-runtime-semantics.zh.md](../notes/mechanisms/dual-runtime-semantics.zh.md)「学习优先级裁决」执行——`compaction/` 必读，其余 durable 栈子树延后；并完成 Q1 的源码级裁决。

- [ ] [harness/compaction/](../../packages/agent/src/harness/compaction/) 必读三文件：`compaction.ts` / `branch-summarization.ts` / `utils.ts`——记一句话职责，并想清楚「生产路径在哪落地」（阶段 4 将到 `coding-agent/src/core/compaction/` 验证，本步只读这层纯函数）
- [ ] harness/ 顶层文件逐个记一句话职责：`agent-harness.ts`（durable 栈入口）、`context.ts`（chord `Context` 用法）、`events.ts` / `hooks.ts` / `messages.ts` / `system-prompt.ts` / `skills.ts` / `prompt-templates.ts`——只求职责定位，不深入实现
- [ ] 延后确认（不深读，只登记位置）：`runtime/`（含 `drive/`，durable 栈）、`session/`、`execution/`、`tools/`、`env/`、`utils/`——各记一句话职责即可（过关检验③）
- [ ] **Q1 源码级裁决**：亲手验证 [dual-runtime-semantics.zh.md](../notes/mechanisms/dual-runtime-semantics.zh.md) 的两条证据——① `harness/` 全目录对 `agent-loop` 引用 0 命中（两栈互不 import）；② `index.ts` 公开导出 `AgentHarness`（durable 栈已发布）——核实后回填 [questions.zh.md](../questions.zh.md) Q1（状态 → answered，摘要指向 dual-runtime-semantics 笔记）
- [ ] 可选浏览 [docs/harness.md](../../packages/agent/docs/harness.md) 规格开头，理解 durable 栈的设计方向即可

## 第 6 步：动手任务（过关检验①主场）

目标：把 loop 读进实物——用锚点 JSONL 对照事件落点，用单测验证理解。

- [ ] 对照 [experiments/001-session-anchor.zh.md](../experiments/001-session-anchor.zh.md) 的会话 JSONL，在 [agent-loop.ts](../../packages/agent/src/agent-loop.ts) 里标出每类 entry（SessionHeader / 用户消息 / assistant 消息 / toolResult）由哪段代码 emit 的哪个事件触发（注意：loop 本身不写 JSONL，写盘在 coding-agent 消费事件时发生——本步只需对上「事件 → 谁消费 → 落哪类 entry」这条链）
- [ ] 三项前置检查自测（过关检验①）：
  - ① 事实源：用自己的话说「会话 JSONL 是对话状态的唯一事实源；模型历史 / UI / resume 都从它投影」
  - ② append-only + 分支：解释「日志只增不改；分支靠 `parentId`」，并在锚点 JSONL 里指出一处分支点
  - ③ 改动落点：新增一个模型可见的输入/工具结果形态，必须先成为 JSONL 里的一种 entry
- [ ] 从 [packages/agent](../../packages/agent) 包根跑单测（AGENTS.md 规定的包内 vitest 方式）：
      `node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/agent-loop.test.ts`
      跑通并浏览用例，看 faux provider 怎么替身、事件断言怎么断言序列

## 已完成的落盘产出

（完成时登记：experiments / journal / notes / map 更新 / questions 状态流转。）

- 2026-09-14（第 1 步：布局盘点 + README 精读完成）：
  - notes：[notes/modules/agent-package-overview.zh.md](../notes/modules/agent-package-overview.zh.md)（顶层七文件 + search 职责、README 嵌入者叙事、observational vs barrier、convertToLlm 唯一桥接）
  - index：阶段 3 置为进行中，plan 文件索引补 stage-3
- 2026-09-14（第 2/3 步：词汇精读 + agent-loop 精读完成）：
  - notes：[notes/modules/agent-loop.zh.md](../notes/modules/agent-loop.zh.md)（四入口 2×2 组合、双层 while 两种「继续」、事件序列、工具三段式、停止/继续条件、错误通道辨析、runAgentLoop vs agentLoop 搜证）

## 过关检验自测（完成时逐条打勾）

- [ ] ① 通过三项前置检查（事实源 / append-only + 分支 / 改动落点）
- [ ] ② 能对照源码复述一次 turn 的完整生命周期与继续条件
- [ ] ③ 能说出 `harness/` 各子目录职责一句话
- [ ] ④ 能对「agent-loop 与 AgentHarness 的关系」给出源码级裁决（并回填 questions.zh.md）
