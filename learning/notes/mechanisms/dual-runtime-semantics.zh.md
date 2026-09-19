# 运行时双栈的语义：经典内存栈与 durable 持久化栈

状态：已对照验证（2026-09-12，基于两组源码勘察（code-explorer 子代理，只读）+ 当日两轮讨论修正；行号为勘察时参考，版本迭代后以源码为准）。

## 事实源（链接，不复述）

- [agent-loop.ts](../../../packages/agent/src/agent-loop.ts)、[agent.ts](../../../packages/agent/src/agent.ts)：经典栈本体（`runLoop` 双层 while；`Agent` 的 `_state.messages`）
- [agent-harness.ts](../../../packages/agent/src/harness/agent-harness.ts)、[harness/runtime/lane.ts](../../../packages/agent/src/harness/runtime/lane.ts)：durable 栈本体（接口与 accept/drive 操作状态机）
- [harness.md](../../../packages/agent/docs/harness.md)：durable 运行时规格（自述 "A durable runtime for agent conversations"）
- coding-agent CHANGELOG："inherited v2 session and `AgentHarness` API"
- [pi-architecture-overview.zh.md](../architecture/pi-architecture-overview.zh.md) §3：双栈的架构定位与源码证据链

## 它是什么（≤5 句）

「栈」不是数据结构，指从入口到存储的一整条执行链路：谁驱动循环、状态放在哪、怎么调 LLM、结果落到哪。agent 包里有两条各自完整、互不调用的链路。经典内存栈把持久化当日志用（执行后 append，内存即真相）；durable 持久化栈把持久化当账本用（执行前登记、每步提交，持久层即真相）。两条栈都已正式发布，按场景分工并存，是否收敛是开放问题。

## 「栈」指什么

不是数据结构的栈，是**执行链路**。agent 包里存在两条完整的链路，每条都独立回答四个问题：循环怎么驱动、状态的事实源在哪、LLM 怎么调、结果怎么落。两条链路互不 import（`harness/` 全目录对 `agent-loop` 引用 0 命中），共享的只有类型底座（types.ts、pi-ai、chord Context）。

## 两条栈逐词拆解

**经典内存栈（路径 A）**：

- **「内存」**：状态事实源在进程内存——`Agent` 持有 `_state.messages`（内存数组），`runLoop` 是进程内 while 循环，turn 的全部状态就是这个循环的局部变量。
- **「经典」**：教科书式的 agent loop 写法——一个进程、一个循环、内存即真相，人人熟悉。
- **「生产路径」**：pi 当前真正跑的链路——`AgentSession → Agent → runAgentLoop`，敲 `pi` 命令走的就是它。

**durable 持久化栈（路径 B）**：

- **「durable」（耐久）**：对比内存的易失。每一步**先持久化、再执行**：操作（run / compaction / navigation）先经 `accept` 在 session 事务中登记（纯准入，不启动任何效果），再由宿主反复调 `drive` 分阶段推进，每个阶段结束原子提交完整进度。
- **「持久化」**：状态事实源在持久层（`pi.op.state` 等），内存只是缓存。
- **设计目标**：崩溃可恢复、多 lane（一个 harness 管多个并行会话）、可被远程宿主（server / session-worker）驱动。

## 核心差异：先做事后记账 vs 先记账再做事

```
经典内存栈 (生产路径):

  内存 while 循环 ----------执行----------> 副作用产生
       |                                    |
       |          事件完成后才 append         |
       +--------- JSONL (旁路日志, 恢复材料) --+

  持久化角色: 事后记账。执行不依赖日志; 崩溃后用日志重建。

durable 持久化栈 (已发布的第二条栈):

  accept -----> 持久层登记操作 (字据)
                     |
  drive 阶段N ---> 提交进度 ---> 执行副作用 ---> 提交结算
                     |
  崩溃重启: 读 pi.op.state, 看到推进到哪, 从那里继续
           (已结算的效果不重放, 未开始的继续或明确放弃)

  持久化角色: 事前立据。执行本身依赖持久层做推进凭证。
```

## 一个崩溃场景

Turn 进行中：assistant 消息已流完，发了 3 个工具调用，执行完 2 个，此时进程被 kill。

- **经典栈**：内存 transcript 全丢；JSONL 里只落了已完成的条目。重启 resume 时从 JSONL 重建上下文——已完成的 2 个工具结果还在，但第 3 个工具可能已经产生了副作用（比如文件已写）而结果丢失，恢复后的上下文里这个 tool_call 没有配对结果，只能靠上层逻辑兜底。
- **durable 栈**：kill 前最后一次提交记录了「工具 2 已结算」。重启后 harness 读 `pi.op.state`，从工具 3 处继续——已结算的不重放（避免重复写文件），未开始的接着做。

注意 durable 不是「回滚副作用」（文件写了就是写了），它的价值是：**精确知道进行到哪一步 + 不重复执行已完成的操作 + 未完成的可以继续**。

## 最易误解的一点：经典栈不等于没有持久化

「经典栈」不等于「没有持久化」——生产路径明明有会话 JSONL（见 [experiments/001-session-anchor.zh.md](../../experiments/001-session-anchor.zh.md) 的实物锚点）。区别在**持久化的时机与角色**：

| | 经典内存栈 | durable 持久化栈 |
|---|---|---|
| 状态事实源 | 内存（循环变量） | 持久层（session 存储） |
| 循环模型 | 进程内 while | 操作状态机（accept / drive） |
| 持久化时机 | 事件完成后 append（事后） | 操作开始前登记 + 每步提交（事前/事中） |
| 崩溃语义 | 进行中的 turn 作废，从最后完整消息恢复 | 任意两次提交之间可恢复，继续推进 |
| 谁驱动循环 | 进程自己 | 宿主（server / worker）反复调 drive |

一句话：**经典栈把持久化当日志用，durable 栈把持久化当账本用**。日志丢了可以重建（有损），账本丢了业务就断了（所以它必须是执行的前置条件）。

## 状态澄清：durable 栈已发布，无替换承诺（认知修正）

我曾经的误解：以为 durable 栈「还在开发中、还没有正式发布、将来作为下一代方案替换经典栈」。

实际是：

- **已正式发布**：`AgentHarness` 随 `pi-agent-core` 公开导出（`packages/agent/src/index.ts`），有规格文档（`docs/harness.md`）、有存储一致性测试（sqlite-node 的 conformance 套件），并已在真实场景服役（coding-agent experimental 的 session-worker / mini / services、packages/server、evals）。不是藏起来的原型或实验分支。
- **无替换承诺**：仓库没有公开承诺「将替换经典栈」或给出时间表。CHANGELOG 的 "v2 session and AgentHarness API" 是方向依据，不是替换声明。
- **准确表述**：经典栈是当前产品主路径；durable 栈是已正式发布的第二条栈，已在服务端场景服役，设计意图指向下一代，但「未来是否/何时替换主路径」仓库没有承诺，属于开放问题。

```
两条已发布的栈, 按场景分工并存:

  经典内存栈   -> 单进程交互式使用 (pi 命令的产品主路径)
  durable 栈   -> 崩溃可恢复 / 多 lane / 远程驱动的宿主场景
                 (server, session-worker, evals -- 已在服役)

  是否收敛 / 替换: 仓库无承诺, 属开放问题
```

误读的代价：记成「未发布的实验品」会低估 experimental / server 代码的成熟度（那里是有 conformance 测试和规格文档的正式子系统）；记成「已确定要替换」会误读它的当前定位。修正来源：2026-09-12 两轮讨论。

### 版本演进（2026-09-19，合并 main commit 99d9144）：AgentHarness 与 Pico 系列

durable 方向出现两个新落地，读源码后裁决如下（对照 `packages/durable/docs/pico-v5*.md` 与 `agent/src/harness/pico3/`）。

**「Pico」是什么**：pi 内部给「durable、可扩展的 agent harness」这条研发线起的代号（codename），非通用术语。`pico-v5.md` 首句即 "Pico5 is a durable, extensible agent harness"。

**版本脉络**（代号迭代，非线性的数字序列）：

| 代号 | 位置 | 状态 |
|---|---|---|
| pico、pico4 | （已删） | 早期原型，已废弃删除 |
| Pico3 | `agent/src/harness/pico3/` | 实验性 kernel，仅作参考（`./experimental/pico3` 导出） |
| Pico5 | `packages/durable` | 规范（normative），仅完成第 1 步（record 契约 + `MemoryStorage`） |

**AgentHarness 与 Pico 的关系（已裁决）**：AgentHarness（本篇所述的第一代 durable 栈）**不属于 Pico 系列**——它比 Pico 更早，是第一代；Pico 是后来起的「重新设计」代号。时间线是 AgentHarness →（pico/pico4 已删）→ Pico3（参考）→ Pico5（规范 + 实现中）。

**为什么要有 Pico5**：把 durable 从「只覆盖 conversation」扩展到「conversation + task + document」三类事实——`TaskRecord`（durable 状态机，checkpoint / after / abort / background）与 `DocumentRecord`（可变 JSON 状态，rewindable / fork）是 AgentHarness 的 conversation-only 模型表达不了的。抽成独立包 `pi-durable`，定义独立 `Storage` 契约（原子 commit / mintId / scan），与 agent 解耦。**关键**：document 的可变状态用 chord 的 replicated state 承载（`applyImmutable` + `Op`）——这回答了 Q3（chord 在 agent 里的运行时参与度）：chord 的 replicated state 正在成为 durable runtime 的 document 底座。

**当前状态（2026-09-19）**：AgentHarness **仍在使用、没有过时**——`index.ts` 仍导出、`harness/` 子树只增不减（仅新增 `pico3/`），且仍在 server / evals / sqlite-node 服役；Pico5 只有第 1 步、`pi-durable` 依赖全仓 0 命中、Pico3 仅作参考。Pico 系列是「更远的探索分支」，未取代现有服役的那一代。

**结论不变**：本篇「双栈并存、无替换承诺」的核心结论依然成立，AgentHarness 仍是当前真正的 durable 实现。

## 学习优先级裁决（2026-09-12）

主流使用（`pi` 命令的 interactive / rpc / print 三模式）全部走经典栈；durable 栈在服务端场景（server / session-worker），一般用户不接触。因此**学习主线是经典栈**，harness/ 子树按需选读：

```
经典栈 (学习主线, 阶段 3 重点):
  agent-loop.ts / agent.ts / types.ts ......... 必读, 全部
  coding-agent 的 AgentSession / SessionManager  阶段 4 必读

harness/ 子树 (选择性读):
  harness/compaction/ ......................... 必读——生产路径在复用
                                                (coding-agent/src/core/compaction
                                                是它的落地, 阶段 3/4 绕不开)
  docs/harness.md (规格) ..................... 浏览——理解设计方向即可
  runtime/lane.ts, drive/* .................... 推迟——只服务 durable 栈,
                                                主线学完或触碰 server 时再读
```

结论：把 harness/runtime/ 从「当前最大的洞」降级为「已知位置、暂不深读」——洞已填（双栈关系已裁决），剩下的是按需深入。

## 与相邻单元的关系

- 上游：[pi-architecture-overview.zh.md](../architecture/pi-architecture-overview.zh.md) §3 给出双栈的架构定位与源码证据（生产路径的 `sdk.ts` `new Agent`、主路径 0 命中验证），本篇展开两条栈的语义差异与状态澄清。
- 关联问题：Q2（drive 的分阶段推进）、Q3（chord 参与度）见 [questions.zh.md](../../questions.zh.md)。
- 「会话 JSONL 树模型」是经典栈持久化旁路的实物锚点：resume = 从 JSONL 树投影重建内存态。

## 验证方式

- 生产路径：`packages/coding-agent/src/core/sdk.ts` 的 `new Agent({...})`；`packages/agent/src/harness/` 全目录对 `agent-loop` 引用 0 命中。
- durable 栈发布状态：`packages/agent/src/index.ts` 导出 `AgentHarness`；`packages/agent/docs/harness.md` 存在规格；sqlite-node conformance 测试存在。
- 消费方分布：`Agent` 仅 coding-agent core 生产代码与测试；`AgentHarness` 在 coding-agent experimental、packages/server、evals、sqlite-node。

## 遗留问题

- 双栈未来是否收敛 / 替换（观察 experimental 与 server 的演进）——同架构总览 §10 第一条。
- durable 方向的新一代落地：`packages/durable`（Pico5 规范）与 `harness/pico3/`（参考）相对 AgentHarness 的定位已裁决（见「版本演进」节）——它们是更远的探索分支，未取代 AgentHarness；Pico5 的 task/document 语义（checkpoint/after、rewindable/fork）仍待深入。
