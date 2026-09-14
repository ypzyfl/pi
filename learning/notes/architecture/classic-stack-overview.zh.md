# 经典内存栈（生产路径）整体架构

状态：初稿（2026-09-12。聚焦视图：本文是 [pi-architecture-overview.zh.md](pi-architecture-overview.zh.md) 中生产路径相关内容的独立重组，供学习主线使用。事实源同总览：源码 + 官方文档，冲突以源码为准；行号为勘察时参考，版本迭代后以源码为准）。

范围声明：本文只覆盖经典内存栈（当前生产路径）。`pi` 命令的三种模式（interactive / rpc / print）全部走这条路径。仓库中不属于本路径的内容（durable 持久化栈、远程会话轴、evals 等）不在本文范围，需要时见 [dual-runtime-semantics.zh.md](../mechanisms/dual-runtime-semantics.zh.md) 与总览 §3——它们不影响对本文任何内容的理解。

## 1. 一句话定位

`pi` = AgentSession（coding-agent 包，组装中枢）把 Agent 运行时（agent 包）+ 统一 LLM API（ai 包）+ 终端 UI（tui 包，仅 interactive 模式）装配成编码 agent CLI；一次对话 = 进程内 while 循环驱动「组请求 → 调模型 → 执行工具 → 判断继续」，内存即真相，会话 JSONL 是旁路日志（事实源，事后 append）。

## 2. 涉及的包（6 包依赖图）

生产路径只涉及 6 个包。箭头 = 「依赖」（读自各包 package.json 的已验证边）：

```
+----------------------------------------------------------------------+
|                          coding-agent (产品层)                        |
|     AgentSession 组装 | 内置工具 x8 | 会话/设置/信任 | 扩展加载         |
|     出口: interactive(TUI) / rpc / print / sdk                        |
+-----+---------------+-----------------+----------------+-------------+
      |               |                 |                |
      v               v                 v                v
+----------+  +-------------+  +-------------+  +------------+
|  chord   |  |    agent    |  |     ai      |  |    tui     |
| 应用组合  |  |  运行时核心  |  |  统一 LLM   |  |  差分渲染  |
| 运行时    |<--| loop+Agent |-->| 多provider  |  |   UI 库   |
| (无依赖) |  |  (双栈§3)   |  |    API      |  |  (无依赖)  |
+----------+  +------+------+  +------+------+  +------------+
                     |                 |
                     | 依赖            | 依赖
                     v                 v
                  +--------------------------+
                  |         telemetry        |
                  |      (无依赖, 地基)        |
                  +--------------------------+
```

各包一句话：coding-agent 是产品（AgentSession、内置工具、会话/设置/信任、扩展加载、三模式）；agent 提供 Agent + agentLoop 循环原语；ai 把多 provider 抹平成统一 API；tui 是差分渲染 UI 库（只被 interactive 模式用）；chord 是应用组合运行时（地基）；telemetry 是遥测契约（地基）。

## 3. 静态结构：谁组装谁

箭头 = 创建 / 持有 / 调用（静态关系，不是数据流）：

```
+---------------------------------------------------------------+
| coding-agent (产品包)                                         |
|                                                              |
|  modes (入口): interactive / rpc / print                     |
|    |  interactive 模式经 tui 包渲染 (差分渲染 UI 库)          |
|    v                                                         |
|  AgentSession (组装中枢, agent-session.ts)                   |
|    组装: [工具注册表] [SessionManager] [ExtensionRunner]      |
|          [ModelRuntime] [SettingsManager] [ResourceLoader]   |
|    工具来源: 内置 x8 + 扩展注册                               |
+--------------+-----------------------------------------------+
               | 创建并持有
               v
+---------------------------------------------------------------+
| agent (运行时包) -- 经典栈本体                                 |
|                                                              |
|  Agent (有状态包装, agent.ts)                                 |
|    持有: _state.messages (内存 transcript, 即真相)            |
|          steering / followUp 队列                             |
|          beforeToolCall / afterToolCall 钩子                   |
|    调用: agentLoop (无状态 turn 循环原语, agent-loop.ts)      |
+--------------+-----------------------------------------------+
               | 每次 LLM 调用经 streamFn
               v
+---------------------------------------------------------------+
| ai (模型包)                                                  |
|                                                              |
|   streamSimple (统一流式 API) | 模型目录 (生成式, 不可手改)   |
+--------------+-----------------------------------------------+
               v
      provider (OpenAI / Anthropic / Google / ...)

外部落点 (静态关系, 不在调用链上):
  SessionManager ----> 会话 JSONL 文件 (append-only 树)
  ExtensionRunner <---- 扩展 (.pi/ 发现: 注册工具/命令/事件钩子)
```

读法：AgentSession 是组装中枢，创建并持有 Agent 与五个服务；Agent 调用 agentLoop；每次 LLM 调用经 ai 的 streamSimple 到 provider。会话 JSONL 与扩展是两个外部落点：前者由 SessionManager 追加，后者经 ExtensionRunner 注入。

## 4. 一次对话：数据流与事件流

### 4.1 精简数据流

```
  用户
   |  interactive(TUI) / rpc / print
   v
+--+------------------------------------------------------------+
| coding-agent (产品层)                                          |
|                                                               |
|   AgentSession [组装中枢]                                     |
|   装配: 工具注册表(内置 x8 + 扩展注册) | SessionManager        |
|         | ExtensionRunner | SettingsManager                   |
|   入口职责: 校验模型与鉴权; 展开 slash 命令 / skill /           |
|             prompt template                                   |
+---------------+-----------------------------------------------+
                | Agent.prompt()
                v
+---------------------------------------------------------------+
| agent (经典内存栈: 内存即真相)                                  |
|                                                               |
|   Agent (agent.ts, 有状态包装)                                 |
|     内存态: _state.messages | steering / followUp 队列         |
|         |                                                     |
|         | runAgentLoop                                        |
|         v                                                     |
|   runLoop (agent-loop.ts, turn 循环)                          |
|     组请求 -> 流式响应 -> 提取 toolCalls                      |
|     -> executeToolCalls (工具: 内置 x8 + 扩展注册)            |
|     -> toolResult 推回上下文 -> 有工具调用则下一 turn          |
+---------------+-----------------------------------------------+
                | streamFn (每次 LLM 调用)
                v
+---------------------------------------------------------------+
| ai (模型层): streamSimple 统一 API -> provider                 |
|               (OpenAI / Anthropic / Google / ...)              |
+---------------------------------------------------------------+

旁路 (不在主链上, 但每 turn 都发生):
  message_end 事件 --> SessionManager append --> 会话 JSONL (树)
  agent 事件 --> ExtensionRunner (30+ 事件钩子) / 模式层渲染
```

### 4.2 详细事件流（turn 生命周期）

```
用户
 |  输入 (interactive TUI / rpc stdin / print -p)
 v
AgentSession.prompt()                                 [coding-agent]
 |  1. 校验模型/鉴权 (ModelRuntime)
 |  2. 展开 slash 命令 / skill / prompt template
 v
Agent.prompt() -> 消息入队                              [agent]
 |
 v
runAgentLoop (agent-loop.ts, 双层 while)
 |
 |  agent_start
 |    |
 |    |  +---------- turn 循环 (while 有工具调用或排队消息) ----------+
 |    |  |                                                          |
 |    |  |  prepareNextTurn      <-- 压缩 (compaction) 的注入点       |
 |    |  |  turn_start                                               |
 |    |  |  [注入 steering 消息, 如有]                                |
 |    |  |       |                                                  |
 |    |  |       v                                                  |
 |    |  |  transformContext -> convertToLlm                         |
 |    |  |       |  (系统提示词 + 历史 + 工具 schema -> Message[])     |
 |    |  |       v                                                  |
 |    |  |  streamFn -----> pi-ai streamSimple -----> provider       |
 |    |  |       |             (统一 API, 词汇转换)     (OpenAI/      |
 |    |  |       |                                    Anthropic/...)  |
 |    |  |  message_start / update / end  (assistant 流式)           |
 |    |  |       |                                                  |
 |    |  |       v                                                  |
 |    |  |  提取 toolCalls; stopReason=error/aborted -> 直接终止      |
 |    |  |  stopReason=length -> 工具调用标记为截断错误 (不执行)       |
 |    |  |       |                                                  |
 |    |  |       v                                                  |
 |    |  |  executeToolCalls  (顺序 / 并行 批执行)                    |
 |    |  |  tool_execution_start / update / end                      |
 |    |  |       |     内置工具或扩展注册的工具                         |
 |    |  |       v                                                  |
 |    |  |  toolResult 消息以 message_start/end 推回上下文           |
 |    |  |       |                                                  |
 |    |  |  turn_end -> shouldStopAfterTurn? -> 可提前终止            |
 |    |  +----------------------------------------------------------+
 |    |
 |    |  (无 follow-up 排队消息 -> 退出外层循环)
 |  agent_end
 |
 v  事件流同时分三路:
 +---> SessionManager: message_end 时 append 进会话 JSONL (事实源)
 +---> ExtensionRunner: 30+ 事件钩子 (tool_call / context 改写 / ...)
 +---> 模式层监听者: TUI 渲染 / rpc stdout / print 输出
```

停止条件全集：error/aborted、`shouldStopAfterTurn`、工具批全部 `terminate === true`、无 follow-up 消息。

## 5. AgentSession 组装详解

```
        createAgentSessionServices (cwd 绑定的基础设施)
        +------------------------------------------------------+
        | ModelRuntime     SettingsManager     ResourceLoader   |
        | (模型目录/鉴权)    (设置)    (extensions/skills/prompts|
        |                            /themes, 含项目信任门控)    |
        +-------------------------+----------------------------+
                                  |  createAgentSession (sdk.ts)
                                  v
        +------------------------------------------------------+
        |                     AgentSession                      |
        |                                                       |
        |  Agent (agent 包)  <--- prompt() / steer() / followUp() |
        |      beforeToolCall/afterToolCall --------------------+
        |      (经 _installAgentToolHooks 桥接到扩展的 tool 事件)  |
        |                                                       |
        |  工具注册表: createAllToolDefinitions (read/bash/edit/   |
        |    write/grep/find/ls/powershell) + 扩展注册工具        |
        |    + allowlist / denylist                              |
        |  SessionManager (JSONL 会话树, 见 §6)                  |
        |  ExtensionRunner (事件分发)                             |
        +---------------------^---------------------------------+
                              | 事件 / prompt
        AgentSessionRuntime 持有它, 负责 /new / resume /fork /import
        的整会话替换 (teardown -> 重建 -> session_start)
                              |
            +-----------------+-----------------+
            v                 v                 v
     interactive (TUI)    rpc (stdin/stdout   print (单发, -p;
     全功能界面/斜杠命令/    JSONL 无头协议;      --mode json 变体
     会话树导航/主题        扩展 UI 请求-响应桥)  输出事件流)
```

生命周期六步：建服务 → 建会话（`new Agent` + 恢复历史）→ 绑定扩展（`bindExtensions`，发 `session_start`）→ 运行回合（`prompt` → `_runAgentPrompt` → `_handlePostAgentRun`：自动重试 / 溢出压缩 / 排队消息 → `agent_settled`）→ 会话替换（switchSession / newSession / fork / import）→ 销毁（dispose）。

## 6. 会话树：持久化旁路（事实源）

```
~/.pi/agent/sessions/--<cwd编码>--/<timestamp>_<uuid>.jsonl

SessionHeader (首行: id / cwd / parentSession / version)
   |
   v
 [e1 user] <-- [e2 assistant] <-- [e3 toolResult] <-- [e5 user] ...   主线
                  ^
                  | 分支: 新 entry 的 parentId 指向 e2
                  |
              [e6 user] <-- [e7 assistant] ...                      分支 B
```

不变量（全部上层能力的地基）：

- **append-only**：日志只增不改；仅在出现首条 assistant 消息后才真正落盘（避免空会话文件）。
- **分支 = 改 leaf 指针**：`branch(id)` 把 leaf 指到历史条目，后续 append 从那里长出新枝，原路径仍在文件中。
- **resume = 路径投影**：从 leaf 沿 `parentId` 回溯到根，生成恢复上下文 `{messages, thinkingLevel, model}`——这就是「崩溃/重启后从日志重建内存态」的机制。
- **compaction = 追加条目**：追加 `compaction` 条目 + `firstKeptEntryId`，旧条目不删除；压缩在 `prepareNextTurn` 注入点触发，实现落在 `coding-agent/src/core/compaction/`。
- **会话级 fork = 导出新文件**：只写 root→leaf 路径，新 header 的 `parentSession` 指回原文件，形成会话间父子链。

权威定义：[session-format.md](../../../packages/coding-agent/docs/session-format.md)；实物锚点：[experiments/001-session-anchor.zh.md](../../experiments/001-session-anchor.zh.md)。

## 7. 内置工具与 edit 的安全设计

工具全集（8 个）：`read / bash / powershell / edit / write / grep / find / ls`；默认激活 `["read", "bash", "edit", "write"]`；工厂函数 `createAllToolDefinitions` / `createCodingTools` / `createReadOnlyTools`。

edit 的安全设计要点：

- **精确文本替换语义**：schema 要求 `edits[].oldText` 必须唯一且不与其他编辑重叠。
- **强校验链**：找不到匹配报错；出现多次直接报错（防止改错位置）；编辑区间重叠检测；替换后无变化也报错。
- **写前检查 + 互斥**：整个读-改-写包在 `withFileMutationQueue` 里（同文件写互斥，避免并发竞态）。
- **编码/换行安全**：BOM 剥离再匹配、写回还原；CRLF/LF 归一化匹配后按原文件换行风格还原。
- **容错而不失安全**：对模型输出怪癖兼容（JSON 字符串形式的 edits 等）；匹配失败先走模糊归一化（智能引号 / Unicode 破折号），但唯一性与重叠校验依旧生效。

## 8. 三种运行模式

模式判定在 `main.ts`（`--mode rpc` → rpc；`--mode json` → json 事件流；`-p` 或非 TTY → print；否则 interactive），三模式共享 AgentSession，只加各自的 I/O 层：

| 模式 | 入口 | 职责 |
|---|---|---|
| interactive | `modes/interactive/interactive-mode.ts` | TUI 全功能界面（tui 包渲染）：斜杠命令、模型/主题/会话选择器、会话树导航；业务逻辑委托 AgentSession |
| rpc | `modes/rpc/rpc-mode.ts` | 无头 JSON 协议：stdin 收 JSON 命令、stdout 吐事件；含扩展 UI 请求-响应桥接 |
| print | `modes/print-mode.ts` | 单发：prompt → 输出 → 退出码；`--mode json` 变体逐行输出事件流 |

## 9. 扩展体系（外挂资源）

```
五种外挂资源 (都不在核心内):
+----------------+--------------------+---------------------------------+
| 资源            | 形态               | 加载时机                         |
+----------------+--------------------+---------------------------------+
| extension      | TS/JS 模块 (jiti)  | 会话启动时全部加载               |
| skill          | SKILL.md           | 系统提示词列目录, 模型按需读      |
| prompt template| md 文件            | /slash 命令展开时                |
| theme          | 主题定义            | TUI 启动/切换时                 |
| pi package     | npm/git/local 包   | pi install 后由 manifest 过滤    |
+----------------+--------------------+---------------------------------+
```

extension 发现顺序（去重后串行加载）：`<cwd>/.pi/extensions/`（项目本地）→ `~/.pi/agent/extensions/`（全局）→ settings/CLI 显式路径。加载细节：jiti 直接加载 TS（无需预编译），扩展永远用宿主打包的 API 版本；未信任项目先加载 pre-trust 扩展集。

extension 的挂载点全集：工具（`registerTool`）、斜杠命令（`registerCommand`）、事件钩子（`pi.on`，30+ 事件：会话生命周期 / turn / 消息流式 / `tool_call`/`tool_result` 可拦截改写 / `context` 改写发给 LLM 的消息 / provider 请求响应前后）、UI（setWidget / 对话框 / 选择器）、provider（`registerProvider`）、资源发现（`resources_discover`）、会话操作（newSession/switchSession/fork）。

## 10. 关键文件速查

| 文件 | 角色 |
|---|---|
| `packages/coding-agent/src/core/agent-session.ts` | AgentSession 组装中枢 |
| `packages/coding-agent/src/core/sdk.ts` | `createAgentSession` 工厂（`new Agent` 在此） |
| `packages/coding-agent/src/core/agent-session-services.ts` | cwd 绑定的基础设施 |
| `packages/coding-agent/src/core/session-manager.ts` | 会话树管理（分支/resume/fork） |
| `packages/coding-agent/src/core/extensions/` | 扩展加载（loader.ts）与分发（runner.ts） |
| `packages/coding-agent/src/core/tools/` | 内置工具族（edit.ts / edit-diff.ts 重点） |
| `packages/coding-agent/src/core/compaction/` | 压缩纯函数落地 |
| `packages/coding-agent/src/modes/` | 三模式入口 |
| `packages/agent/src/agent.ts` | Agent 有状态包装 |
| `packages/agent/src/agent-loop.ts` | turn 循环原语（`runLoop`） |
| `packages/ai/src/` | 统一 LLM API（`models.generated.ts` 红线：改走 `generate-models.ts`） |

## 11. 学习顺序建议

对应 [learning-path.zh.md](../../learning-path.zh.md)：阶段 2（ai 包：统一 LLM API 与模型目录生成链路）→ 阶段 3（agent 包：agent-loop.ts → agent.ts → types 等；harness/ 子树只读 `compaction/`，见 [dual-runtime-semantics.zh.md](../mechanisms/dual-runtime-semantics.zh.md)「学习优先级裁决」）→ 阶段 4（coding-agent：AgentSession / 内置工具 / 会话管理 / 三模式）→ 阶段 5（扩展体系，对照 `.pi/` dogfooding 实例）。

阅读每个部分时以会话 JSONL 为对照物：读 agent loop 时问「这些事件落在 JSONL 哪一行」，读 extension 时问「它改了树的哪部分」。
