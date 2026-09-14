# 会话消息流：从 Session 创建到最终回答

状态: 草稿（流程框架已对照 session-format.md 与真实会话文件；工具调用关联与多轮 append 已于 2026-09-08 对照真实会话实测——含并行双调，见 [experiments/001](../../experiments/001-session-anchor.zh.md)；压缩、分支部分仍来自文档推导；session/run/turn 层级来自 agent 包源码阅读，未经运行时事件验证）

## 事实源（链接，不复述）

- [session-format.md](../../../packages/coding-agent/docs/session-format.md)：文件位置、entry 类型、消息类型与 content 块定义、id/parentId 链
- [packages/ai/src/types.ts](../../../packages/ai/src/types.ts)：`AssistantMessage.content` 的块类型联合（L430 附近）
- [packages/agent/src/types.ts](../../../packages/agent/src/types.ts)：`AgentEvent`（agent / turn / message / tool_execution 生命周期，L431 起）
- [packages/agent/src/agent-loop.ts](../../../packages/agent/src/agent-loop.ts)：run 的两层循环、turn 事件发射、steering / follow-up 队列
- [packages/agent/src/agent.ts](../../../packages/agent/src/agent.ts)：`Agent.prompt()` / `Agent.continue()` 两个 run 入口
- 真实会话文件：见「验证方式」

## 它是什么（用自己的话）

一次「用户提问 → 模型两次工具调用 → 最终回答」的完整对话，在会话 JSONL 里是从 user entry 到最终 assistant entry 的一段 entry 链；每轮模型请求都携带完整消息数组，历史随轮次增长。assistant 消息的 content 是 text / thinking / toolCall 三种块的任意组合；模型输出 `stopReason: "toolUse"` 时 agent loop 在端侧本地执行工具，把结果作为独立 toolResult entry 落盘并随下一轮请求回传，见到 `stopReason: "stop"` 才结束本轮。工具调用与结果的关联不走 parentId，而是 toolCall 块的 `id` 与 toolResult entry 的 `toolCallId` 互相回指。

## 基础流程图（单轮：一问两调一答）

```
     端侧 (pi / agent loop)                              模型 (LLM API)
     ════════════════════════                          ═════════════════
          │
   ① Session 创建
      写首行 SessionHeader
      {"type":"session",version:3,
       cwd:...}              (首行，无 id/parentId)
      + model_change（记录选定模型）
          │
   ② 用户输入"问题"
      append e1 {role:"user",
        content:[{type:"text",text:"问题"}]}
          │
          ├─────────── 请求 #1 ───────────►       收到消息数组 [user]
          │       (携带完整消息数组)                  │ 思考
          ◄═════════ 流式响应 ═══════════════        │ 决定调用工具A
          │                                          ▼
   ③ append e2 {role:"assistant",
        content:[{thinking},
                 {toolCall id:"tc1",
                  name:"read_file"}],
        stopReason:"toolUse"}
          │
   ④ 本地执行工具 A（端侧进程内，不出网）
      append e3 {role:"toolResult",
        toolCallId:"tc1",
        content:[{type:"text",text:...}],
        isError:false}
          │
          ├─────────── 请求 #2 ───────────►       收到 [user, assistant,
          │                                          toolResult#1]
          ◄═════════ 流式响应 ═══════════════          │ 决定调用工具B
          │                                          ▼
   ⑤ append e4 {role:"assistant",
        content:[{thinking},
                 {toolCall id:"tc2",name:"bash"}],
        stopReason:"toolUse"}
          │
   ⑥ 本地执行工具 B
      append e5 {role:"toolResult",
        toolCallId:"tc2", content:[text]}
          │
          ├─────────── 请求 #3 ───────────►       收到全部历史
          ◄═════════ 流式响应 ═══════════════          │ 生成最终回答
          │                                          ▼
   ⑦ append e6 {role:"assistant",
        content:[{thinking},{type:"text",text:"回答"}],
        stopReason:"stop"}
          │
   (会话挂起：等待下一问，或 /resume 继续)
```

图例与要点：

- **方向**：`├──►` = 端→模型（HTTP 请求，流式）；`◄═══` = 模型→端（流式响应）；②④⑥ 的 JSONL 落盘和工具执行都在**端侧本地**完成。
- **entry 链**：`e1 → e2 → e3 → e4 → e5 → e6`，每个 entry 的 `parentId` 指向前一个的 `id`（本例是线性链，无分支；`e1` 的 `parentId` 为 `null`）。实际 `id` 是 8 位十六进制，`e1`…`e6` 是示意缩写。
- **关联方式**：工具调用与结果不靠 `parentId` 关联，而是 assistant 里的 `toolCall` 块带 `id`，toolResult entry 用 `toolCallId` 回指。
- **stopReason** 区分轮次性质：`"toolUse"` = 模型还要继续（带工具调用），`"stop"` = 最终回答，agent loop 见到 `stop` 才结束本轮。
- 每次请求都携带**完整消息数组**（历史随轮次增长）；真实文件里可能还夹着 `model_change` / `thinking_level_change` 等条目，图中省略（全景图覆盖它们）。

## 全景流程图（覆盖全部 entry 类型与消息 role）

场景是构造的复合会话，覆盖 [session-format.md](../../../packages/coding-agent/docs/session-format.md) 全部 entry 类型与 AgentMessage 全部 7 个 role；真实会话通常只出现其中一小部分。编号 entry（e1…e13、mc1 等）均为示意缩写。

```
        端侧 (pi / agent loop)                              模型 (LLM API)
        ════════════════════════                          ═════════════════
             │
    ① Session 创建
       首行 SessionHeader {"type":"session",version:3,
         id:<uuid>,cwd:...}             (元数据，不在树中，无 id/parentId)
             │
    ② model_change {provider, modelId}            用户选定模型
    ③ thinking_level_change {thinkingLevel:"high"} 用户设置思考等级
       ※ 两者不作为消息进上下文；请求时从路径提取最新值作为设置
             │
    ④ 用户输入"问题"
       append e1 {role:"user", content:[text]}
             │
             ├────────── 请求 #1 ──────────►        [e1]
             ◄═════════ 流式响应 ═══════════
    ⑤ append e2 {role:"assistant",
         content:[thinking, toolCall#1(read_file)],
         stopReason:"toolUse"}
             │
    ⑥ 本地执行工具 A
       append e3 {role:"toolResult", toolCallId:"tc1", ...}
             │
             ├────────── 请求 #2 ──────────►        [e1,e2,e3]
             ◄═════════ 流式响应 ═══════════
    ⑦ append e4 {role:"assistant",
         content:[thinking, toolCall#2(bash)],
         stopReason:"toolUse"}
             │
    ⑧ 本地执行工具 B
       append e5 {role:"toolResult", toolCallId:"tc2", ...}
             │
             ├────────── 请求 #3 ──────────►        [e1..e5 完整历史]
             ◄═════════ 流式响应 ═══════════
    ⑨ append e6 {role:"assistant",
         content:[thinking, text], stopReason:"stop"}   ← 第 1 轮结束
             │
    ⑩ custom entry {customType:"todo", data:{...}}
       ※ 扩展状态持久化，不进上下文（重载时恢复扩展状态）
    ⑪ custom_message entry {customType:"lint",
         content:"发现 3 个警告...", display:true}
       ※ 扩展注入的上下文，进 LLM；display 只控制 TUI 是否显示
    ⑫ 用户 shell 命令（TUI 中 "! 前缀"）
       append {role:"bashExecution", command:"npm test",
         output:..., exitCode:0, cancelled:false}
       ※ 进上下文；"!!" 前缀则 excludeFromContext:true
             │
    ⑬ 用户追问
       append e7 {role:"user", content:[text]}
             │
             ├────────── 请求 #4 ──────────►        [e1..e6 + CustomMessage(⑪)
             ◄═════════ 流式响应 ═══════════           + bashExecution(⑫) + e7]
    ⑭ append e8 {role:"assistant",
         content:[thinking, text], stopReason:"stop"}   ← 第 2 轮结束
             │
    ⑮ 上下文过长（或 /compact）触发压缩
             ├──── 摘要生成调用 ─────►            "请总结以上对话..."
             ◄═════ 摘要文本 ═════════
       append cp1 {type:"compaction",
         summary:"此前讨论了 X、Y、Z...",
         retainedTail:[...保留的近期消息],
         tokensBefore:50000, usage:{...}}
       ※ 自包含检查点：之后的请求从
         [compactionSummary + retainedTail] + 之后 entries 重建
             │
    ⑯ 用户新问题
       append e9 {role:"user", content:[text]}
             │
             ├────────── 请求 #5 ──────────►        [cp1(摘要+保留尾), e9]
             ◄═════════ 流式响应 ═══════════
    ⑰ append e10 {role:"assistant",
         content:[thinking, toolCall#3], stopReason:"toolUse"}
    ⑱ 本地执行工具 C
       append e11 {role:"toolResult", toolCallId:"tc3", ...}
             │
             ├────────── 请求 #6 ──────────►        [.., e10, e11]
             ◄═════════ 流式响应 ═══════════
    ⑲ append e12 {role:"assistant",
         content:[thinking, text], stopReason:"stop"}   ← 第 3 轮结束
             │
    ⑳ /tree：用户回到 e8 另起分支
             ├──── 摘要生成调用 ─────►            "总结被放弃的路径..."
             ◄═════ 摘要文本 ═════════
       append bs1 {type:"branch_summary",
         parentId:e8, fromId:e12,               (新分支的第一个 entry)
         summary:"该分支探索了方案 A，因 X 放弃..."}
       append e13 {role:"user", "换个思路：..."}
       ※ 被放弃的旧路径(cp1..e12)压缩成 branchSummary 注入新分支；
         压缩 cp1 只作用于它所在的旧路径，新分支不再受它影响
             │
             ├────────── 请求 #7 ──────────►        [root→e8 路径,
             ◄═════════ 流式响应 ═══════════           branchSummary, e13]
    ㉑ session_info {name:"重构 auth 模块"}
       (/name 设置；/resume 列表显示用)  ※ 不进上下文
    ㉒ label {targetId:e6, label:"checkpoint-1"}
       (给任意 entry 打书签)          ※ 不进上下文
```

结束时的树结构（SessionHeader 在树外；mc1 的 `parentId` 为 `null`，是树的根）：

```
mc1 ─ tl1 ─ e1 ─ e2 ─ e3 ─ e4 ─ e5 ─ e6 ─ c1 ─ cm1 ─ be1 ─ e7 ─ e8 ─┬─ cp1 ─ e9 ─ e10 ─ e11 ─ e12   ← 旧叶子（被放弃）
                                                                      │
                                                                      └─ bs1 ─ e13 ─ si1 ─ lb1        ← 当前叶子
```

### 各类型是否进入 LLM 上下文

| 类型（entry / role） | 进上下文 | 请求时如何参与 |
|---|---|---|
| `session`（header） | 否 | 元数据，不在树中 |
| `message`: user / assistant / toolResult | 是 | 对话主体，原样进消息数组 |
| `message`: bashExecution | 是（`!`）/ 否（`!!`） | 用户 shell 命令与输出 |
| `custom` | 否 | 扩展状态，重载时恢复 |
| `custom_message` | 是 | 转为 `CustomMessage`（role: `custom`） |
| `compaction` | 是 | 转为 `compactionSummary` + `retainedTail` 检查点 |
| `branch_summary` | 是 | 转为 `branchSummary`，保留被放弃分支的上下文 |
| `model_change` / `thinking_level_change` | 否 | 从路径提取最新值作为请求设置 |
| `session_info` / `label` | 否 | 显示名 / 书签，纯交互元数据 |

AgentMessage 全部 7 个 role 在图中都有出现：user、assistant、toolResult（④—⑲）、bashExecution（⑫）、custom（⑪ 经转换）、branchSummary（⑳ 经转换）、compactionSummary（⑮ 经转换）。

两个值得记住的细节：**compaction 和 branch_summary 的摘要本身各是一次 LLM 调用**（entry 上的 `usage` 字段记录，计入会话成本）；**压缩是路径局部的**——分支到 e8 之后，旧路径上的 cp1 不再参与新分支的上下文重建。

## 概念层级：session / run / turn（持久层 vs 运行时）

JSONL 里所有消息平等地挂在 session 树上，没有 turn 分组——这个观察对持久层成立，但 pi 在运行时有完整的 turn 概念，且定义比「一个用户问题到处理完毕」更细：**一个 turn = 一次 assistant 响应 + 它的工具调用/结果**（[agent/types.ts](../../../packages/agent/src/types.ts) L435）。「一个问题到处理完毕」（含 N 次工具调用）= N+1 个 turn，外面套一个 **run**（一次 agent loop 调用，`agent_start` → `agent_end`）。turn / run 都是内存中的事件，只有产出的 message entry 落盘。

| 概念 | 定义 | 生命周期 | 持久化 | 代码权威 |
|---|---|---|---|---|
| session | 一棵 entry 树（JSONL 文件） | 跨进程，创建到删除 | 是 | [session-manager.ts](../../../packages/coding-agent/src/core/session-manager.ts) |
| run | 一次 agent loop 调用（`agent_start` → `agent_end`） | 一次 `prompt()` / `continue()` | 否（产物落盘） | [agent-loop.ts](../../../packages/agent/src/agent-loop.ts) |
| turn | 一次 assistant 响应 + 其工具调用/结果 | 一次 LLM 请求 | 否 | [agent/types.ts](../../../packages/agent/src/types.ts) |
| message | AgentMessage（user/assistant/toolResult/…） | 原子单位 | 是（`message` entry） | [agent/types.ts](../../../packages/agent/src/types.ts) |
| tool execution | 单个工具调用的执行 | turn 内 | 产物（toolResult）落盘 | [agent/types.ts](../../../packages/agent/src/types.ts) |

```
Session (JSONL 文件，唯一持久容器)
════════════════════════════════════════════════════════════════════
 entry 树:  [model_change]──[user]──[asst+tool]──[toolRes]──[asst]──┐
                                （run 1 的落盘产物）                    │
                                                                      │
 Run 1  (agent_start ──────────────────────────────── agent_end)    │
 ├── Turn 1: assistant(stopReason=toolUse) + toolResult ─────────────┤
 ├── Turn 2: assistant(stopReason=toolUse) + toolResult ─────────────┤
 └── Turn 3: assistant(stopReason=stop) ─────────────────────────────┘
                                                                      │
 entry 树续:                                              ──[user]──[asst]
 Run 2  (agent_start ──── agent_end)                     （run 2 的落盘产物）
 └── Turn 1: assistant(stopReason=stop)
════════════════════════════════════════════════════════════════════
```

运行时事件嵌套（一个 run 内部，`message_update` 只在 assistant 流式期间出现）：

```
agent_start
 ├── turn_start
 │    ├── message_start ─ message_update* ─ message_end   ← assistant（流式）
 │    ├── tool_execution_start ─ tool_execution_end      ← 每个工具调用
 │    ├── message_start ─ message_end                     ← toolResult
 │    └── turn_end (message, toolResults)
 ├── turn_start ─ … ─ turn_end                            ← 0..N 个 turn
 └── agent_end (newMessages)
```

关系规则：

1. **session : run = 1 : N**。session 跨进程存活；run 是内存中的事件流，结束后只有产出的 message entry 追加进 session 树，run/turn 本身不落盘。
2. **run : turn = 1 : N**。turn 与 assistant message 严格 1:1——每个 turn 恰好一次 LLM 请求、一条 assistant message；run 内 turn 数 = LLM 请求数。
3. **turn 内**：assistant 的 `toolCall` 块与 `toolResult` message 用 `toolCallId` 配对；无工具调用的 turn 直接以 `stopReason: "stop"` 结束。
4. **run 有两个入口**：`Agent.prompt()`（新问题）与 `Agent.continue()`（从既有 transcript 继续，末条须为 user 或 toolResult）。
5. **两个例外使 run 边界 ≠ 问题边界**：
   - **steering**：agent 运行中途用户插话，作为 user message 注入下一个 turn 之前（内层循环），不新开 run；
   - **follow-up**：agent 本该停止时队列里还有消息，外层循环吞掉后继续同一个 run。

为什么持久层可以不做 turn：边界可从数据推导——新 run 起点是 `role: "user"` 的 message entry；turn 内边界看 assistant 的 `stopReason`（`"toolUse"` = 还有后续 turn，`"stop"` = run 结束）；工具配对靠 `toolCallId`。持久层因此保持一棵最小消息树，turn/run 是运行时派生视图，与 `buildSessionContext()` 只走树、不依赖 turn 结构一致。注意推导是近似的：steering 消息也以 `role: "user"` 落盘，形态上和新问题无法区分，「用户消息 = 新 run 起点」在 steering 场景会切错——不影响上下文构建，只影响事后按 turn 统计。

一句话：**session 是树，run 是树上一次生长事件，turn 是生长中的一节，message 是落下的叶子——只有叶子持久化。**

## content 块的组合规则（thinking / toolCall 澄清）

thinking 和 toolCall 都是 assistant 消息 `content` 数组里的**内容块**，不是独立消息。定义在 [types.ts](../../../packages/ai/src/types.ts)（L430 附近）：

```typescript
interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  ...
}
```

三种块平级，出现在同一条 assistant 消息里，顺序和组合由模型流式输出决定；content 数组没有「必须含 toolCall」的约束，任意组合都合法。常见组合：

```
[thinking]                        只有思考（模型只输出了推理）
[thinking, text]                  思考 + 文字回答（最常见，无工具轮）
[thinking, toolCall]              思考 + 工具调用（流程图 ③ 的情况）
[toolCall]                        直接调用，无思考块
[thinking, text, toolCall]         边说边调（少见但合法）
```

补充两点：

- thinking 是否出现取决于模型能力和 thinking level 设置（会话文件里的 `thinking_level_change` entry 就是在记录这个）。
- 反过来 toolResult 永远是**独立 entry**（role: `toolResult`），不会和 assistant 混在一条消息里——工具结果由端侧生成，不是模型输出的。

## 我曾经的误解

原以为 thinking 可能是独立的消息类型、或必须与 toolCall 绑定出现 → 实际是 `AssistantMessage.content` 数组里与 `TextContent` / `ToolCall` 平级的块，任意组合合法；真实会话里 `assistant: [thinking, text]`（无工具调用）是常态（2026-09-07 对照 types.ts L430 与真实会话文件）。

原以为「一个用户问题到处理完毕」是一个 turn、持久层缺了这层结构 → 实际 pi 的 turn 定义更细（一次 assistant 响应 + 工具结果；问题到完毕是 N+1 个 turn 外套一个 run）；turn/run 是运行时事件不落盘，持久层只存 message 树，边界可靠 `stopReason` 推导（2026-09-07 对照 packages/agent/src/{types.ts, agent-loop.ts, agent.ts}）。

## 验证方式

真实会话文件（无 key、无工具调用的最简轮）：`~/.pi/agent/sessions/--D--Tech-Github-pi-fork-pi--/2026-09-07T08-04-50-926Z_01a07ae6-086e-7060-b4fc-c651af95bcce.jsonl`，结构为 `session → model_change → thinking_level_change → message(user,text) → message(assistant,[thinking,text])`。逐行摘要命令（PowerShell）：

```powershell
Get-Content <会话文件> | ForEach-Object { $o = $_ | ConvertFrom-Json; if ($o.type -eq 'message') { "{0}: {1}" -f $o.message.role, (($o.message.content | ForEach-Object { $_.type }) -join ',') } else { $o.type } }
```

会话文件位置规则见 session-format.md「File Location」；Windows 实测补充：cwd 的 `:` 与 `\` 都替换为 `-`（如 `D:\Tech\...` → `--D--Tech-...`）。带工具调用的真实会话已补验（2026-09-08，[experiments/001](../../experiments/001-session-anchor.zh.md)）：`~/.pi/agent/sessions/--D--Tech-Github-pi-pi--/2026-07-30T06-19-23-886Z_019fb1ad-796e-72ad-bc99-d9d4e288e9c7.jsonl`（安装版 pi，5 次调用含两处并行双调），图中 ③—⑥ 的 toolCall ↔ toolCallId 关联与多轮 append 均实测通过；并行双调时两个 toolResult 线性 append（第二个的 `parentId` 指向第一个），配对只靠 `toolCallId`。

## 遗留问题

- 树形分支语义：追问 append 到链尾已实测（[experiments/001](../../experiments/001-session-anchor.zh.md)）；仍待验证：`/fork` / `/clone` 时 `parentId` 如何指向非链尾 entry、`branchSummary` entry 的参与方式。
