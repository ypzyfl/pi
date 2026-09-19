# agent 包（pi-agent-core）骨架笔记

状态：草稿（2026-09-14 对照 [packages/agent/README.md](../../../packages/agent/README.md) 与 [packages/agent/src/](../../../packages/agent/src/) 顶层文件；阶段 3 第 1 步骨架盘点）

## 事实源（链接，不复述）

- [packages/agent/README.md](../../../packages/agent/README.md)（517 行，嵌入者视角）
- [packages/agent/src/](../../../packages/agent/src/) 顶层七文件 + `search/`
- [packages/agent/src/index.ts](../../../packages/agent/src/index.ts)（出口）
- [packages/agent/src/harness/](../../../packages/agent/src/harness/)（目录结构）

## 它是什么（≤5 句）

pi 的 agent 运行时核心包（npm 名 `@earendil-works/pi-agent-core`），定位一句话：Stateful agent with tool execution and event streaming，built on `@earendil-works/pi-ai`。它提供两层 API——有状态的 `Agent` 类（owns transcript、事件分发、steering/follow-up 队列）与无状态的 `agentLoop` 回合驱动（双层 while）。`AgentMessage` 是包内通用消息词汇，`convertToLlm` 是它到 LLM `Message[]` 的唯一桥接点。同一包内还住着第二条已发布的 durable 栈（`AgentHarness`，语义见 [dual-runtime-semantics.zh.md](../mechanisms/dual-runtime-semantics.zh.md)）。

## 顶层七文件 + search/ 职责

| 文件 | 一句话职责 |
|---|---|
| `index.ts` | 出口：转发 telemetry 类型 + `export *` 导出 agent/loop/harness/proxy/search/types，并公开导出 `AgentHarness` |
| `agent-loop.ts` | 回合驱动：无状态低层 loop，`AgentMessage` 贯穿，只在 LLM 调用边界转 `Message[]` |
| `agent.ts` | 有状态包装：`Agent` 类 owns transcript、事件分发（subscribe）、队列（steering/follow-up） |
| `types.ts` | 词汇：`StreamFn` / `AgentEvent` / `AgentMessage` / `AgentState` / `AgentTool` / `AgentContext` / `AgentLoopConfig` |
| `stream-fn.ts` | 默认流函数：`setDefaultStreamFn` 让宿主注入 model runtime，使本包不依赖 provider catalog |
| `node.ts` | Node 出口：`export { NodeExecutionEnv }` + 转发 index |
| `proxy.ts` | 代理流函数：`streamProxy` 供浏览器应用经后端代理调 LLM，客户端重建 partial |
| `search/index.ts` | 搜索服务契约：`SessionSearchService` / `SearchQuery` / 命中类型（纯接口） |

（2026-09-19 版本对齐：`harness/` 下新增 `pico3/` 实验性子系统，经 package.json 的 `./experimental/pico3` 子路径导出，不并入 `index.ts` 顶层 export。）

## 关键实体（逐个链接到 home）

- `Agent`：有状态包装 → [agent.ts](../../../packages/agent/src/agent.ts)
- `agentLoop` / `runAgentLoop`：无状态回合驱动 → [agent-loop.ts](../../../packages/agent/src/agent-loop.ts)
- `StreamFn`：调 LLM 的统一函数签名 → [types.ts](../../../packages/agent/src/types.ts)
- `AgentMessage` / `AgentEvent` / `AgentState` / `AgentTool` / `AgentContext` / `AgentLoopConfig`：词汇 → [types.ts](../../../packages/agent/src/types.ts)
- `AgentHarness`：durable 栈入口 → [harness/agent-harness.ts](../../../packages/agent/src/harness/agent-harness.ts)
- `streamProxy`：代理流函数 → [proxy.ts](../../../packages/agent/src/proxy.ts)
- `setDefaultStreamFn`：默认流函数注入 → [stream-fn.ts](../../../packages/agent/src/stream-fn.ts)

## 与相邻单元的关系（依赖谁 / 被依赖谁）

- 依赖：`@earendil-works/pi-ai`（模型层，`StreamFn` 的落地）、`pi-telemetry`（遥测类型转发）、chord（facet-service 原语；README 注明 core 不导出 service runtime）
- 被依赖：`coding-agent`（生产路径 `AgentSession → Agent → runAgentLoop`）

## 我曾经的误解（原以为 → 实际是 → 修正来源）

1. 顶层文件数「八文件」→ 实际七 .ts + `search/` 子目录 → `list_dir` 实查
2. README 393 行（阶段 1 勘察值）→ 实际 517 行 → `read_file` 全文
3. 无（第 1 步尚未深读 loop，误解留待第 3 步）

## 本步最大认知增量

1. **observational vs barrier**：README Low-Level API 明说 `agentLoop` 是观察性的——只保序、不等待 async 事件处理 settle；`Agent` 类才在 assistant `message_end` 后设置 barrier，保证 `beforeToolCall` 看到已含该 assistant 消息的状态。两者分工：一个「看」、一个「拦」。
2. **convertToLlm 是唯一桥接**：`AgentMessage[] → transformContext(可选) → convertToLlm(必需) → Message[] → LLM`，UI-only 消息在此过滤。
3. **Q1 证据提前落地**：`index.ts` 公开 `export * from "./harness/agent-harness.ts"`——durable 栈「已发布」的直接证据（第 5 步再补另一条 0 命中证据）。

## 验证方式

- `list_dir` 遍历 packages/agent/src 与 harness/；`read_file` 读 README 全文、index.ts / stream-fn.ts / node.ts / proxy.ts / search/index.ts 全文、agent-loop.ts / agent.ts / types.ts 全文

## 遗留问题

- Q1 agent-loop 与 AgentHarness 关系（第 5 步源码级裁决 + 回填 questions.zh.md）
- Q2 harness/runtime/drive/ 的 drive 概念（延后）
- Q3 chord 在 agent 的参与度（延后）
