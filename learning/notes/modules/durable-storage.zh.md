# durable 包（Pico runtime）存储与事务体系笔记

状态：已对照验证（2026-09-27 对照 [packages/durable/src/](../../../packages/durable/src/) 的 types.ts、ids.ts、documents.ts、session/transaction.ts、session/forks.ts、README.md；阶段 7 按需预读，非主线）

## 事实源（链接，不复述）

- [index.ts](../../../packages/durable/src/index.ts)：公开导出（`createSession` / `defineDoc` / `defineDocFamily` / `MemoryStorage` / 大量品牌类型）
- [types.ts](../../../packages/durable/src/types.ts)：record 契约、品牌 ID、document 语义
- [ids.ts](../../../packages/durable/src/ids.ts)：品牌 ID 的安全构造入口
- [session/transaction.ts](../../../packages/durable/src/session/transaction.ts)（880 行）：事务性 session 的心脏
- [session/forks.ts](../../../packages/durable/src/session/forks.ts)：fork 文档复制
- [README.md](../../../packages/durable/README.md)：三存储后端 + conformance/benchmark 用法

## 它是什么（≤5 句）

`@earendil-works/pi-durable` 是「conversation / task / document 三类事实」的 durable 运行时（Pico5 落地），2026-09-19 时只有 record 契约 + `MemoryStorage`，现已扩张为完整存储体系：三存储后端（memory / jsonl / sqlite）+ 品牌类型 ID + 事务性 session + migrate/checkpoint + 惰性 fork + conformance/benchmark 套件。document 的可变 JSON 状态由 chord 的 immutable delta 承载（`track`/`prepare`/`adopt`）。仍无其他 workspace 包依赖它，是独立发布、未取代 AgentHarness 的新一代探索分支。

## 关键实体

- **Storage 契约**：`mintId` / `scan*` / 原子 commit；三个实现 MemoryStorage / JsonlStorage(node) / SqliteStorage(node)，经 `./storage/*` 子路径导出。
- **品牌类型 ID**：`Id<Kind, Type> = number & { [idBrand]: ... }`，把裸 number 变成 `ConversationId` / `EntryId` / `TaskId` / `DocumentId` 等**编译期不可互换**的类型；`ROOT_CONVERSATION_ID = 1`；`ids.ts` 的 `idFromNumber`/`seqFromNumber` 是打品牌的唯一可信入口。
- **document 语义**：`DocumentSemantics` 四种 scope（session / conversation-Latest / conversation-Rewindable / task），`CommonDocDefinition` 带 `version` + `migrate`（旧版本迁移）+ `checkpointWhen`（何时把 delta 折叠成 base）。
- **Transaction**（`session/transaction.ts`）：commit callback 里读写先内存 staging，`settleSuccess` → prepare（转 ops）→ assemble（原子 `StorageWrite[]` 批）→ Storage 采纳；`#read` 在写后 throw `ReadAfterWrite`；`settleFailure`/`discard` 回滚。
- **fork**：`forkConversation` → `prepareForkDocumentCopies` → `document.copy`（惰性，记 source 引用、消费者 hydrate），`#rejectForkSourceWrites` 禁止在 fork 事务里改源文档。

## 与相邻单元的关系

- 依赖 chord（`Context`/`copyJson`/`Draft`）+ chord/delta（`track`/`beginChange`/`prepare`/`adopt`）+ pi-ai（`Message`）。
- 是 [dual-runtime-semantics.zh.md](../mechanisms/dual-runtime-semantics.zh.md)「版本演进」的延续：AgentHarness 是第一代 durable 栈，Pico3（agent harness/pico3）是参考，Pico5（本包）是规范 + 实现中。
- 与本次 chord 的「immutable delta tracking 收敛」配套演进：document 用 chord delta，chord 优化 delta 引擎。

## 我曾经的误解（原以为 → 实际是 → 修正来源）

- 原以为（2026-09-19）Pico5 只有「第 1 步（record 契约 + MemoryStorage）」→ 实际已落地三后端 + 事务 + 迁移/checkpoint + fork + 品牌 ID → 对照 `durable/src/` 目录与 README。
- 原以为 document fork 是深度复制 → 实际是 `document.copy` 惰性物化（记 source 引用，hydrate 时才读）→ `forks.ts` + `transaction.ts` 的 `fork-copy` 分支。
- 原以为 ID 只是 number 别名 → 实际是 unique symbol 品牌，编译期强制「conversation 的 ID 不能当 document 用」→ `types.ts` L8-22。

## 验证方式

- `read_file` 读 `types.ts`（品牌 ID + 三类 record + document 语义）、`transaction.ts`（staging/原子提交/写后读禁止/fork-copy）。
- `git grep -l "pi-durable"` 确认全仓仅它自己的 package.json 命中（0 消费）。

## 遗留问题

- 三个后端的取舍与 fsync/WAL 语义（JSONL 的 `fsync` 默认 false、SQLite 的 WAL checkpoint）未深读。
- `migrate` / `checkpointWhen` 在真实会话里的触发频率与性能影响未实测。
- Pico5 与经典栈的集成路径（何时真正被 coding-agent 采用）仍待观察。
