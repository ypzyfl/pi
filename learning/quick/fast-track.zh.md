# 五步速通路线图

**目标**：2-4 小时内独立写出一个可生效的 pi 扩展。每步有过关标志，不达标不进入下一步。

## 速通全景

```mermaid
gantt
    title 快速通道时间线（理想节奏）
    dateFormat X
    axisPlacement none
    section 跑起来
    Step 1 跑通 pi-test.ps1           :done, s1, 0, 5min
    section 看懂
    Step 2 架构一图一表               :s2, after s1, 30min
    Step 3 核心模块速查               :s3, after s2, 15min
    section 动手
    Step 4 写扩展三件套               :s4, after s3, 45min
    Step 5 数据流对照                 :s5, after s4, 25min
```

## 五步详解

### Step 1 — 5 分钟跑起来（5 min）

**做什么**：在 Windows 上从源码跑起 pi，看到交互界面。

**怎么做**：见 [commands.zh.md](commands.zh.md) 「Step 1」一节。核心一条：

```powershell
# 仓库根目录
.\pi-test.ps1
# 或传参
.\pi-test.ps1 --list-models
.\pi-test.ps1 -p "Say exactly: ok"
```

**关键认知**：`pi-test.ps1` 经 jiti 直接加载 TS 源码，**无需 `npm run build`** 即可运行——这是 pi 扩展开发"快速迭代"的基石。

| 过关标志 | 达成方式 |
|---|---|
| ① `pi-test.ps1` 能启动交互界面 | 看到 pi 的输入框 |
| ② `--list-models` 列出模型目录 | 命令行有输出 |
| ③ `-p "Say exactly: ok"` 回复 ok | 有 key 时；无 key 改读 [../experiments/001-session-anchor.zh.md](../experiments/001-session-anchor.zh.md) 的会话 JSONL |

### Step 2 — 30 分钟看懂架构（30 min）

**做什么**：建立 pi 的整体心智模型——它是什么、分几层、一次消息怎么穿层。

**怎么做**：精读 [architecture.zh.md](architecture.zh.md)。只需记住三张图：

1. **分层图**：6 个桌面端包的依赖拓扑
2. **组装图**：AgentSession 是组装中枢，谁注入谁
3. **数据流图**：一次消息从用户输入到工具执行的全路径

| 过关标志 | 达成方式 |
|---|---|
| ① 能不看资料画出 6 包分层 | chord/tui/telemetry 是地基，agent 叠三者，coding-agent 是产品 |
| ② 能说出 AgentSession 的四项组装 | 工具注册表 / SessionManager / ExtensionRunner / ModelRuntime |
| ③ 能复述一次消息穿过 5 层的路径 | 用户→TUI→AgentSession→agent loop→ai→provider |

### Step 3 — 核心模块速查（15 min）

**做什么**：给 6 个桌面端包各建一句话定位 + 关键文件指针，供后续随时查。

**怎么做**：读 [modules.zh.md](modules.zh.md)。重点是建立"遇到问题去哪个文件"的索引，不是全读源码。

| 过关标志 | 达成方式 |
|---|---|
| ① 每个包能说一句话定位 | 见 modules.zh.md 表格 |
| ② 知道改扩展去哪、改工具去哪、改系统提示词去哪 | 关键文件指针表 |
| ③ 知道 `.pi/` 目录的角色 | dogfooding 实例：本仓库用 pi 开发 pi |

### Step 4 — 插件开发上手（45 min）

**做什么**：亲手写 extension + skill + prompt template 三件套，全部在 `.pi/` 或临时目录，不动 `packages/`。

**怎么做**：照 [extension-quickstart.zh.md](extension-quickstart.zh.md) 的三个可跑示例抄一遍并改。参考实例：[../../.pi/extensions/tps.ts](../../.pi/extensions/tps.ts)、[../../.pi/prompts/cl.md](../../.pi/prompts/cl.md)。

| 过关标志 | 达成方式 |
|---|---|
| ① extension 在 pi 会话中生效 | 自定义工具可调或命令可见 |
| ② skill 被模型按需加载 | `/skill:name` 能加载 |
| ③ prompt template 经 slash 命令展开 | `/myname` 展开正确 |

### Step 5 — 数据流对照（25 min）

**做什么**：把 Step 2 的数据流图与 Step 4 写的扩展对照——你写的扩展挂在数据流的哪一环。

**怎么做**：见 [architecture.zh.md](architecture.zh.md) §4「扩展挂载点全集」。用 `/debug` 命令查看实际渲染行与发给 LLM 的消息。

| 过关标志 | 达成方式 |
|---|---|
| ① 能说出自己写的扩展挂在哪个事件 | tool_call / session_start / agent_end 等 |
| ② 能用 `/debug` 看到扩展的副作用 | `~/.pi/agent/pi-debug.log` 有记录 |
| ③ 能解释为何扩展无需重新构建 pi | jiti 运行时加载 TS，`/reload` 热重载 |

## 完成标志总表

| Step | 主题 | 必达标志数 | 预计耗时 |
|---|---|---|---|
| 1 | 跑起来 | 3 | 5 min |
| 2 | 看懂架构 | 3 | 30 min |
| 3 | 核心模块速查 | 3 | 15 min |
| 4 | 插件开发上手 | 3 | 45 min |
| 5 | 数据流对照 | 3 | 25 min |
| **合计** | | **15** | **~2 小时** |

## 风险提示

- **无 key 的能力边界**：Step 1 的真实交互需 key；无 key 只能读历史会话或测试套件会话，体验打折但不阻塞后续步骤。
- **`models.generated.ts` 是红线**：永不手改；改模型目录走 [../../packages/ai/scripts/generate-models.ts](../../packages/ai/scripts/generate-models.ts) 再生成（AGENTS.md 明文）。
- **改代码后必跑 `npm run check`**：但本通道产出都在 `.pi/` 或临时目录，不在 check 范围。
- **不跑全量测试与构建**：`npm run build` / `npm test` 仅在用户要求时执行（AGENTS.md）。
- **并行会话纪律**：多个 pi 会话可能同时改本仓库；git 操作只碰自己改的文件，禁用 `git add -A`、`git reset --hard`。
