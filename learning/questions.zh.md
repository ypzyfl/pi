# 开放问题池

问题挂着不动会烂掉：每一条要么被解答（填答案摘要与出处链接），要么被蒸馏进对应笔记后从本表删除。三态流转与拆分规则见 [method.zh.md](method.zh.md)：超过约 50 行或出现 3 个以上域主题时，按域拆分为 `questions/<domain>.zh.md`，本文件退化为索引。

初始六问均为建立学习区时勘察产生的真实问题，多数预期在阶段 3–5 解答。

| 问题 | 状态 | 答案摘要 | 出处 |
|---|---|---|---|
| Q1 `agent-loop.ts` 与 `harness/`（AgentHarness）的关系：新旧两条演进路径、分层协作，还是同一层两半？ | open | — | [map.zh.md](map.zh.md)「agent 运行时」；[learning-path.zh.md](learning-path.zh.md) 阶段 3 |
| Q2 `harness/runtime/drive/` 的「drive」是什么概念？驱动器与 loop 的关系？ | open | — | [packages/agent/src/harness/runtime/](../packages/agent/src/harness/runtime/) |
| Q3 chord 在 agent 中的使用范围：仅类型（`harness/context.ts` 的 `Context` / `ContextKey` / `JsonRepresentation`）还是有运行时参与？ | open | — | [map.zh.md](map.zh.md)「依赖拓扑」；[packages/chord](../packages/chord) |
| Q4 coding-agent 的 extension 系统（`core/extensions/`）与 agent 包的 hooks（`harness/hooks.ts`）是什么关系：extension API 是 hooks 的封装，还是独立机制？ | open | — | [packages/coding-agent/src/core/extensions/](../packages/coding-agent/src/core/extensions/)；[packages/agent/src/harness/hooks.ts](../packages/agent/src/harness/hooks.ts) |
| Q5 会话分支（`parentId`）被 tree navigation / resume / compaction 三者分别如何使用？resume 的精确语义（从任意节点重放？） | open | — | [sessions.md](../packages/coding-agent/docs/sessions.md)；[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) |
| Q6 print mode / json event mode / rpc mode / sdk 四个出口的边界与重叠，各自面向什么集成场景？ | open | — | [usage.md](../packages/coding-agent/docs/usage.md)、[json.md](../packages/coding-agent/docs/json.md)、[rpc.md](../packages/coding-agent/docs/rpc.md)、[sdk.md](../packages/coding-agent/docs/sdk.md) |
