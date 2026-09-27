# pi 的 ACP 支持三路径：extension 死路、mode 改源码、SDK 内嵌

状态：草稿（2026-09-22 三轮推演式讨论落盘；涉及的架构事实均已对照源码（见"事实源"），三路径权衡为分析结论，路径 C 未实测）

背景：另一 workspace 的 pi-acp 项目（`d:\Coding\desktop\pi-acp`，ACP adapter，现架构为 spawn `pi --mode rpc` + stdio 双向翻译）提出的问题——能否不改/少改 pi 让它支持 ACP（Agent Client Protocol）。本文回答：三条路径各自的边界、代价与推荐。

## 事实源（链接，不复述）

- [rpc-types.ts](../../../packages/coding-agent/src/modes/rpc/rpc-types.ts)（`RpcCommand` 封闭 union L20-74，无版本协商字段）
- [rpc-mode.ts](../../../packages/coding-agent/src/modes/rpc/rpc-mode.ts)（stdio 行读取循环归属 mode，L386 起 `handleCommand`）
- [extensions/types.ts](../../../packages/coding-agent/src/core/extensions/types.ts)（`ExtensionMode` 封闭 union L317：tui/rpc/json/print）
- [sdk.ts](../../../packages/coding-agent/src/core/sdk.ts)（`createAgentSession` 公开 SDK 入口）
- [agent-session.ts](../../../packages/coding-agent/src/core/agent-session.ts)（`bindExtensions` 接受自定义 `uiContext` L2919-2922）
- [rpc.md](../../../packages/coding-agent/docs/rpc.md)（协议文档，无兼容性承诺）

## 它是什么（≤5 句）

ACP 的形态要求是刚性的：ACP client（如 Zed）spawn agent 进程，并在其 stdin/stdout 上说 JSON-RPC 2.0——ACP agent 必须拥有进程 stdio 的**解释权**。而 pi 的进程模型里 stdio 解释权属于 **mode**（rpc/interactive/print），extension 只是 mode 内部的租客，无法替换宿主的协议循环。因此"用 extension 让 pi 支持 ACP"是死路；正确路径只有两条：给 pi 加 `--mode acp`（改 pi 源码四处），或让 pi-acp 自己的进程内嵌 pi SDK（不改源码、单进程）。分界线一句话：**要不要让 `pi --mode acp` 这七个字符存在。要，就改源码；不要，SDK 够用。**

## 为什么 extension 是死路：stdio 所有权论证

- **RPC 模式**：stdin/stdout 被 pi 自己的 JSONL 协议独占，ACP client 发的 `{"jsonrpc":"2.0","method":"initialize"}` 与 pi 期待的 `{"type":"prompt"}` 信封（`method` vs `type`）、词汇、会话语义全不同。
- **interactive 模式**：stdin 是 raw mode 终端，被 TUI 占用；**print/json 模式**：stdio 被 prompt/结果占用且单发即退，撑不起 ACP 长连接。
- **绕路均不成立**：extension 抢读 stdin 会破坏宿主协议；进程内起 TCP/named pipe 说 ACP 偏离标准 transport（Zed 只 spawn stdio 进程）；extension spawn ACP 网关子进程则端点位置错（client spawn 的是 pi 本身），退化为 adapter 现状零增益。
- **RPC 命令集封闭**：`RpcCommand` 是静态 union，extension 只能注册 `/slash` 命令（走 prompt 通道），不能向 RPC 协议注册新命令——"ACP over pi-RPC 词汇"的渐进路线也无接口。

## 语义层同构：翻译层是薄的

ACP 与 pi 事件/命令近逐点对应，翻译不难（pi-acp 项目已验证），难的是传输层站位：

| ACP | pi 对应物 |
|---|---|
| session/new / session/prompt / session/cancel | new_session / prompt / abort（AgentSession 方法） |
| agent_message_chunk 通知 | message_update 事件 |
| tool_call / tool_call_update | tool_execution_start/update/end |
| session/request_permission | tool_call 拦截 + ctx.ui.confirm（RPC 模式已有 request/response 子协议） |
| fs/terminal 委托 | pi 本地自理，无需委托（pi-acp MVP 决策） |

## 三路径对比

| | A. 外部 adapter（pi-acp 现状） | B. ACP mode（改 pi 源码） | C. SDK 内嵌 |
|---|---|---|---|
| 改 pi 源码 | 否 | **是**（fork 或上游） | 否 |
| 进程形态 | 双进程（adapter + pi --mode rpc） | 单进程 | 单进程 |
| 依赖契约 | pi-RPC JSONL 协议（最脆，运行时漂移） | 进程内 AgentSession API | SDK API（typed，编译期暴露漂移） |
| extension 支持 | 在子进程里，UI 经两层翻译 | 原生（ctx.mode 正确） | 可加载，ctx.mode 借用 "rpc" |
| 维护方 | pi-acp | pi 上游（理想）或 fork | pi-acp |
| 上手成本 | 已建成 | 需过贡献门槛 / fork 冲突 | 中等重构 |

### 路径 B 的改动清单（改 pi 源码四处）

1. 新增 `src/modes/acp/`：ACP JSON-RPC 循环，骨架与 rpc-mode.ts 同构（行读取 → 分派到 AgentSession 方法 → 事件流转 stdout，换一套协议词汇表）。
2. `args.ts`：`--mode` 取值加 `acp`。
3. `main.ts`：模式分派分支。
4. `ExtensionMode`（types.ts L317）加 `"acp"`，extension 的 ctx.mode 才有正确语义。

归宿：fork 维护（同步上游时冲突）或推动上游（pi 贡献门槛：新贡献者 PR 自动关闭、维护者每日审；此量级需先走 RFC 渠道 rfc.earendil.com）。

### 路径 C 的性质与代价

pi-acp 自己的进程内 `createAgentSession()` + 自己的 stdio ACP 循环，直接调 session 方法、订阅事件翻译成 ACP 通知。性质：SDK 是公开承诺的 API 面（非内部协议）；类型化接口使 pi 升级的漂移在编译期暴露；extension 依然可用（`createAgentSession` 加载扩展，`bindExtensions` 接受自定义 uiContext，可将 ctx.ui.confirm/select 映射到 ACP 的 session/request_permission / user_input）。代价：`ExtensionMode` 无 "acp"，bindExtensions 的 mode 只能借用 "rpc" 或类型断言——extension 里 ctx.mode 的语义是借名，这是不改源码方案的固有税。

## 协议漂移：路径 A 的核心成本

"协议漂移"= pi 上游迭代时 JSONL 协议（命令名、事件类型、字段结构）的任何变化，adapter 无法编译期察觉、只能运行时撞上。三层叠加：

1. **协议无稳定性承诺**：RpcCommand/RpcResponse 是 pi 仓库内部类型非对外契约，lockstep 快迭代下随代码走；且无版本协商字段（无 protocolVersion 握手），漂移完全静默。
2. **依赖方式是运行时字符串匹配**：adapter 按事件 `type` 分派、取字段，pi 侧类型定义不在 adapter 编译单元里（手抄的 interface 只是历史快照）；漂移不产生编译错误，只产生运行时症状（字段 undefined、翻译空洞、事件被静默丢弃），且只在真实事件流过时暴露。实例：2026-09-19 的"system 消息化重构"（systemPrompt 删除、工具声明改由 transcript system 消息承载）直接改变 get_messages 的消息序列与类型集合——假设"消息只有 user/assistant/toolResult"的 adapter 会收到不认识的 system 消息，无声错掉。
3. **对端版本不受控（最痛）**：adapter spawn 的是用户机器上安装的 pi，不是自己锁定的依赖；pi-acp 发布时测过的协议组合，在用户升级 pi 后即失效，三方节奏（pi-acp 发布 / pi 协议变化 / 用户升级）互不同步。

SDK 免疫三机制：编译期暴露（.d.ts 进包，字段删除在 tsc 报错）、版本钉死（package.json 钉精确版本，升级是主动的 npm update + 编译验证）、无跨进程协议（函数调用取代序列化信封，漂移面少一个维度）。本质：**adapter 依赖"pi 恰好此刻在 stdout 上说什么"（观察契约），SDK 依赖"pi 承诺的函数签名"（接口契约）——只有后者会被编译器执行。**

## 结论

- "让 pi 官方原生支持 ACP"：路径 B 唯一，必须改源码，先 RFC 后 PR。
- "pi-acp 项目本身更好"：**路径 C 最务实**——保留单进程形态的类型安全红利，甩掉 pi-RPC 协议依赖，extension 生态以可控方式接入；且是从现状 A 的自然重构（翻译层从"协议对协议"变成"API 对协议"），不动 pi 一行代码。
- extension 的真实角色：**不是承载协议，而是协议之上的行为增强**——在 B/C 任一路径下，把 extension 的 ctx.ui 映射到 ACP 的权限/输入请求，复用权限门、工具包装等扩展生态。

## 我曾经的误解（原以为 → 实际是 → 修正来源）

- 原以为：extension 是任意进程内 JS，能力上或许能实现 ACP 端点。
- 实际是：ACP 端点必须是被 client spawn 的那个进程的 stdio，该资源被 mode 独占；extension 无论抢 stdin、开端口还是 spawn 网关，端点位置或 transport 形态必错其一。"前门钥匙只有 mode 能拿，extension 是客厅里的客人。"
- 修正来源：rpc-mode.ts 的 stdio 循环归属、rpc-types.ts 封闭 union、ExtensionMode 封闭 union 三处源码事实。

## 与相邻单元的关系

- **依据** [command-dispatch.zh.md](../mechanisms/command-dispatch.zh.md)：stdio/命令归属 mode、extension 是 mode 内租客的结论由该篇的命令分派分析支撑；`RpcCommand` 封闭不可扩展也是该篇事实。
- **依据** [extension-hooks.zh.md](../modules/extension-hooks.zh.md)：路径 B/C 中 extension 的 ctx.ui → ACP request_permission 映射，落在该篇 hook 能力分类（拦截型/可修改型）之上。
- **对照** [prompt-change-practices.zh.md](../architecture/prompt-change-practices.zh.md)：system 消息化重构作为协议漂移实例的机制背景。

## 验证方式

- `read_file` 读 [rpc-types.ts](../../../packages/coding-agent/src/modes/rpc/rpc-types.ts) L20-74（确认封闭 union 与无版本字段）
- `read_file` 读 [extensions/types.ts](../../../packages/coding-agent/src/core/extensions/types.ts) L317（ExtensionMode 四值封闭）
- `read_file` 读 [agent-session.ts](../../../packages/coding-agent/src/core/agent-session.ts) L2919-2922（bindExtensions 的 uiContext 注入，路径 C 的 UI 映射支点）
- 路径 C 的实测（未做）：在 pi-acp 里用 createAgentSession 替换 spawn 子进程，验证事件流翻译与 extension 加载——见遗留问题

## 遗留问题

- 路径 C 中 bindExtensions 借用 mode:"rpc" 时 extension 的行为是否有隐性分支依赖（ctx.hasUI、RPC UI 子协议假设）未逐一核对——若做 C 的重构，需先审 ExtensionRunner 里读 _extensionMode 的所有分支。
- ACP 协议自身的多 session 模型（一个连接多个 session/new）与 AgentSessionRuntime 的 newSession/switchSession 生命周期映射未核对——路径 C 重构时的第一个设计点。
- pi 上游对 ACP 的态度未知（issue/RFC 检索未做）——若上游已有意向，路径 B 的协作姿势优于 fork。
