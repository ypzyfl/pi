# 001 会话锚点实验：一次对话 = 一棵 JSONL 事件树

日期：实验执行 2026-09-07；本记录 2026-09-08 补录（补录时用一份历史会话文件增验了工具调用关联与多轮 append，见「操作」第 3 条）。

## 假设

一次 pi 对话落盘为一个 JSONL 文件，UI 里的每个事件对应一行 entry；entry 靠 `id` / `parentId` 成链（分支时成树）；工具调用与结果各有 entry，通过某种字段互相关联。建立这个实物锚点后，pi 的全部持久能力（resume / 分支 / compaction / 导出 / 分享）都能读成「对这棵树的操作」。

## 操作

1. 2026-09-07：按计划从源码启动 pi（Windows `.\pi-test.ps1`，deepseek key 已配置），在仓库根（cwd = `D:\Tech\Github\pi\fork\pi`，可由 SessionHeader 的 `cwd` 字段证实）跑了一轮真实对话「hi → 回答」。该轮未触发工具调用——计划原定「让它读一个文件」未做，缺口由第 3 条补上。
2. 在 `C:\Users\Zhoubin\.pi\agent\sessions\` 下定位会话文件：目录名 `--D--Tech-Github-pi-fork-pi--` 是 cwd 路径转义（`:` 与 `\` 都替换为 `-`）；文件名 `<ISO 时间戳>_<会话 UUID>.jsonl`。锚点文件 1.5KB / 6 行。
3. 2026-09-08（补录本记录时）：锚点会话无工具调用，改用同机历史会话补验——`--D--Tech-Github-pi-pi--\2026-07-30T06-19-23-886Z_019fb1ad-796e-72ad-bc99-d9d4e288e9c7.jsonl`（21KB / 26 entry，5 次工具调用，含两处同 entry 并行双调）。注意：该文件来自安装版 pi（cwd 为 `D:\Tech\Github\pi\pi`），非本 fork 源码运行，但同为 version 3 格式，作关联机制证据有效。
4. 对照 [session-format.md](../../packages/coding-agent/docs/session-format.md) 逐行核读两份文件。

## 观察

锚点会话（6 行，全链）：

```
session(首行，无 parentId) → model_change(v4-pro) → thinking_level_change(high)
→ model_change(v4-flash) → user("hi") → assistant([thinking,text], stopReason:"stop")
```

- 首行 SessionHeader：`type:"session"`，含会话 UUID（= 文件名后缀）与 `cwd`；不参与链——第一个 entry 的 `parentId` 为 `null`，是树根。
- entry 的 `id` 是 8 位短哈希；主线严格线性，每个 entry 的 `parentId` 指向前一行。
- 中途换过模型：两次 `model_change`（v4-pro → v4-flash）都作为链上 entry 记录，`thinking_level_change` 同理——设置变化被持久化为链上节点。
- assistant 消息的 `content` 是块数组（`[thinking, text]`），消息上带 `usage` / `stopReason` / `provider` / `model` / `api`（deepseek 走 `api:"openai-completions"` 适配）等元数据。

补验会话（26 entry，三轮对话节选；另有 `model_change` ×2、`thinking_level_change` ×5 穿插在链上，略）：

```
轮1: user ─ asst(text)
轮2: user ─ asst(toolCall) ─ toolResult ─ asst(toolCall) ─ toolResult ─ asst(text)
轮3: user ─ asst(toolCall×2) ─ toolResult ─ toolResult ─ asst(toolCall×2) ─ toolResult ─ toolResult
         ─ asst(toolCall) ─ toolResult ─ asst(text)
```

- **关联机制实测**：assistant 的 `toolCall` 块内含 `id`（形如 `call_00_xBew…`）；toolResult 是独立 message entry（`role:"toolResult"`），用 `message.toolCallId` 回指。5 组配对全部精确匹配。
- **并行工具调用的链形态**：同一 assistant entry 发两个 toolCall（`call_00` + `call_01`）时，两个 toolResult entry 线性 append——第二个的 `parentId` 指向第一个 toolResult，而非共同的 assistant entry；谁对应谁靠 `toolCallId` 区分，不靠 parentId。
- **多轮 append 实测**：新一轮 user entry 的 `parentId` 指向上一轮最后的 assistant entry——继续对话 = 追加到链尾。

## 结论（实验三问）

① UI 里的每一轮在文件里是一段 entry 链：从 user entry 开始，到 `stopReason:"stop"` 的 assistant entry 结束；中间可穿插任意次「assistant(toolCall) → toolResult」往返；非消息事件（`model_change` / `thinking_level_change`）也在链上占位。
② 工具调用 = assistant entry content 里的 toolCall 块（带 `id`）；结果 = 独立的 `role:"toolResult"` message entry。关联不走 parentId，走 `toolCall.id ↔ toolResult.toolCallId` 互指（实测匹配，含并行双调场景）；parentId 表达的是链上时序位置。
③ 继续对话 = append 到链尾（实测：新 user entry 指向链尾 assistant entry）。树杈只在改写历史时产生（`/tree` 回退、fork），本实验两份文件均为单主线，分支形态未实测——登记为遗留。

## 遗留

- 分支语义未实测：`/tree` / fork 时新 entry 的 `parentId` 如何指向非链尾节点、`branch_summary` entry 如何参与。归 [questions.zh.md](../questions.zh.md) Q5。
- 补验会话来自安装版 pi，非本 fork 源码；如需完全同源证据，可随时用 `.\pi-test.ps1` 跑一轮带工具调用的对话复验（格式同版本，结论不受影响）。
- 版本演进（2026-09-19）：本实验记录的两份会话文件均建于 system 消息出现之前，entry 链从 user 起。新版本会话首条为 system 消息（承载提示词 + 工具声明），文件显著变大；id/parentId 链与 toolCall ↔ toolCallId 关联等核心结论不变。详见 [session-message-flow.zh.md](../notes/mechanisms/session-message-flow.zh.md)「system 消息落盘」。

## 引用

- [session-message-flow.zh.md](../notes/mechanisms/session-message-flow.zh.md)：本实验为其「工具调用关联」与「多轮 append」部分提供了实测证据（该笔记状态行已同步升级）。
