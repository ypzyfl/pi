# 版本对齐 2b0a123de 与重新学习（2026-09-27）

背景：pi 源码从 36b60d2e（0.85.1 末期）更新到 2b0a123de（0.87.1 之后），跨 116 commit、547 文件、+54k / -22k 行，历经 v0.86.0 / v0.86.1 / v0.87.0 / v0.87.1 四个版本。按 method.zh.md「版本对齐三问」核对，发现三处结论被推翻，做了一轮四部分的重新学习。

## 三个认知翻转

**1. `shouldStopAfterTurn` → `finishTurn` 不是改名，是语义重构**

原以为：布尔回调改成决策对象，等价替换。实际是：`finishTurn` 返回三态 `AgentTurnDecision`（`{action:"end"}` / `{action:"continue"}` / `undefined`），把「不停」拆成「正常调度」和「强制再来一次」两种旧 API 表达不了的意图；时序从「先 turn_end 后判停」反转为「先 finishTurn 后 turn_end、决策在 turn_end 后应用」；且 `finishTurn` 现在也对 error/aborted 触发（决策被忽略，迁移时若带副作用必须 guard）。另新增 `prepareRequest`（每次 provider 请求前，含第一次）与 `explicitContinuation` 记账位（`continue` 是「保证最少一次」，能复用 tool-result/steering/follow-up 就不新增请求）。详见 [agent-loop.zh.md](../notes/modules/agent-loop.zh.md)、[agent-loop-runloop.zh.md](../notes/modules/agent-loop-runloop.zh.md)。

**2. `tsgo` → `tsc` 不是换命令，是 native 编译器转正**

原以为：命令名替换。实际是：TS7（native 编译器，Go 重写）从 `@typescript/native-preview` 预览包转正为 `typescript@7.0.2`，CLI 从 `tsgo` 回归 `tsc`；同时移除 `tsx`，改用「Node 内置 type stripping + 自定义 source resolver hook」直跑源码。resolver hook 的动机是「Node 不应用 tsconfig paths，会静默落到过期 dist」——宁可 throw 也不 fallthrough。详见 [typescript-7-toolchain.zh.md](../notes/mechanisms/typescript-7-toolchain.zh.md)。

**3. Pico5 不是「仅第 1 步」，已是完整存储体系**

原以为（2026-09-19 记录）：`packages/durable` 只有 record 契约 + `MemoryStorage`。实际是：已扩张为 memory / jsonl / sqlite 三后端 + 品牌类型 ID + 事务性 session（staging/原子提交/写后读禁止）+ migrate/checkpoint + 惰性 fork + conformance/benchmark 套件；document 的可变状态由 chord 的 immutable delta 承载（印证旧结论）。但 `pi-durable` 仍无其他 workspace 包依赖、未取代 AgentHarness。详见 [durable-storage.zh.md](../notes/modules/durable-storage.zh.md)、[dual-runtime-semantics.zh.md](../notes/mechanisms/dual-runtime-semantics.zh.md)「版本演进」。

## 落盘去向（版本对齐的产出 = 打补丁）

- A 级（结论性错误，已修正）：`agent-loop` / `agent-loop-runloop` / `classic-stack-overview` / `pi-architecture-overview` / `stage-3` / `dual-runtime-semantics` / `map` / `learning-path` / `index`（9 文件）。
- B 级（tsgo→tsc、版本号，已批量替换）：`quick/commands` / `test-isolation` / `method` / `quick/README` / `stage-1`。
- C 级（内容性补充）：`streamfn-production-path`（新增「沿链 request/payload 钩子分层对照」+ `prepareRequest` 命名碰撞提醒）、`agent-package-overview`（词汇表补 `FinishTurn`/`PrepareRequest`/`AgentTurnDecision`）。
- 新笔记：`typescript-7-toolchain.zh.md`（机制）、`durable-storage.zh.md`（模块）。
- 次要增量（chord delta / tui OKLab / 模型目录协议）登记进 [map.zh.md](../map.zh.md)。

## 附带收获

- 「沿链 request/payload 钩子」至少三层、跨三包：agent 包 `AgentLoopConfig.prepareRequest`（AgentMessage 层）、`ModelRuntime.prepareRequest`（请求准备层）、扩展 `before_provider_request`（经 `SimpleStreamOptions.onPayload`，provider 协议层）。同名不同物，读码先问「在哪个包、操作什么对象」。
- 「两套解析面」（check 走 src、build 走 dist）的运行时延伸：直跑源码也要走 src，靠 resolver hook 把 `@earendil-works/*` 扳回 tsconfig paths 指向的 src，避免落到过期 dist。
- C 级纯行号（约 13 篇笔记指向 coding-agent 包的行号）未逐个核对：结论仍对、靠各笔记「行号以源码为准」免责声明兜底，属低价值高成本项，留待后续。
