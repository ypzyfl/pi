# Windows 快速命令速查

**用法**：按场景查命令，不存输出全文。本表面向 Windows 11 + PowerShell。`pi-test.ps1` 是 `pi-test.sh` 的 Windows 等价物，经 jiti 直接加载 TS 源码，**无需 `npm run build`**。

## Step 1：5 分钟跑起来

| 想干什么 | 命令 | 说明 |
|---|---|---|
| 首次装依赖（仓库根） | `npm install --ignore-scripts` | 不跑生命周期脚本 |
| 从源码跑 pi（交互模式） | `.\pi-test.ps1` | 看到 pi 输入框即成功 |
| 列出可用模型 | `.\pi-test.ps1 --list-models` | 验证模型目录 |
| 一次性 prompt（需 key） | `.\pi-test.ps1 -p "Say exactly: ok"` | 回复 ok 即通过 |
| 管道输入 | `Get-Content README.md \| .\pi-test.ps1 -p "Summarize this"` | 文本喂给模型 |
| 带图片 | `.\pi-test.ps1 -p @screenshot.png "What's in this image?"` | |

**无 key 也能学**：跳过真实交互，改读 [../experiments/001-session-anchor.zh.md](../experiments/001-session-anchor.zh.md) 的会话 JSONL 做对照，或跑测试套件看 faux provider 会话。

## 日常开发命令

| 场景 | 命令 |
|---|---|
| 从源码跑 pi（任意目录可用） | `.\pi-test.ps1` |
| 临时加载扩展（本次运行） | `.\pi-test.ps1 -e .\my-ext.ts` |
| 继续最近会话 | `.\pi-test.ps1 -c` |
| 浏览历史会话 | `.\pi-test.ps1 -r` |
| 指定会话名 | `.\pi-test.ps1 --name "my task"` |
| 打开特定会话 | `.\pi-test.ps1 --session <path\|id>` |
| JSON 事件输出（程序集成） | `.\pi-test.ps1 -p "..." --mode json` |
| RPC 模式（进程集成） | `.\pi-test.ps1 --mode rpc` |
| 指定 skill | `.\pi-test.ps1 --skill <path>`（可重复） |
| 禁用 skills 发现 | `.\pi-test.ps1 --no-skills` |
| 指定 prompt template | `.\pi-test.ps1 --prompt-template <path>` |

## 代码检查与测试

| 场景 | 命令 | 说明 |
|---|---|---|
| 代码门禁（改代码后必跑） | `npm run check` | biome + 依赖钉版 + tsgo 等，全绿才可提交 |
| 非 e2e 测试（隔离环境） | 仓库根 `.\test.sh`（需 Git Bash） | 见下「Windows 跑 test.sh」 |
| 包内单测（vitest 包） | 包根 `node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/<file>.test.ts` | |
| 包内单测（tui，node:test） | 包根 `node --test test/<file>.test.ts` | |
| 仅在用户要求时 | `npm run build` / `npm test` | AGENTS.md 规定 |

### Windows 跑 test.sh

`test.sh` 需要 Git Bash，不能用 WSL bash（PATH 上是 WSL）：

```powershell
& "C:\Program Files\Git\bin\bash.exe" ./test.sh
```

详见 [../notes/mechanisms/test-sh-on-windows.zh.md](../notes/mechanisms/test-sh-on-windows.zh.md)。

## 会话内 Slash 命令（交互模式）

| 命令 | 作用 |
|---|---|
| `/help` | 看所有 slash 命令 |
| `/reload` | 热重载 `.pi/extensions/` 与全局 extensions |
| `/login` | 选 provider 登录（subscription 或 API key） |
| `/model` 或 Ctrl+L | 选模型；Ctrl+S 存为启动默认 |
| `/thinking` | 选思考级别；Shift+Tab 循环 |
| `/resume`、`/new`、`/tree`、`/fork`、`/clone` | 会话管理 |
| `/settings` | 改设置（含 skill 命令开关） |
| `/skill:name` | 强制加载并执行 skill |
| `/debug`（隐藏） | 写渲染行与发给 LLM 的消息到日志 |

## 包管理

| 场景 | 命令 |
|---|---|
| 装 npm 包 | `.\pi-test.ps1 install npm:@scope/pkg@1.0.0` |
| 装 git 包 | `.\pi-test.ps1 install git:github.com/user/repo@v1` |
| 装本地包 | `.\pi-test.ps1 install .\path\to\pkg` |
| 临时试包（不持久） | `.\pi-test.ps1 -e npm:@scope/pkg` |
| 卸包 | `.\pi-test.ps1 remove npm:@scope/pkg` |
| 看装了什么 | `.\pi-test.ps1 list` |
| 更新 pi 自身 | `.\pi-test.ps1 update --self` |
| 更新 pi + 所有包 | `.\pi-test.ps1 update --all` |

## 关键路径

| 找什么 | 在哪 |
|---|---|
| 会话 JSONL 文件 | `~/.pi/agent/sessions/--<路径转义>--/<timestamp>_<uuid>.jsonl` |
| 全局扩展 | `~/.pi/agent/extensions/*.ts` |
| 全局 skills | `~/.pi/agent/skills/`、`~/.agents/skills/` |
| 全局 prompt templates | `~/.pi/agent/prompts/*.md` |
| 全局设置 | `~/.pi/agent/settings.json` |
| 全局 AGENTS.md | `~/.pi/agent/AGENTS.md` |
| 项目扩展 | `.pi/extensions/*.ts` |
| 项目 skills | `.pi/skills/` |
| 项目 prompts | `.pi/prompts/*.md` |
| 项目设置 | `.pi/settings.json` |
| 项目 AGENTS.md | `AGENTS.md` 或 `CLAUDE.md` |
| 调试日志 | `~/.pi/agent/pi-debug.log` |
| 认证文件 | `~/.pi/agent/auth.json` |

## TUI 交互速查

| 按键 | 作用 |
|---|---|
| Enter | 提交 prompt |
| `@` | fuzzy 搜文件引用 |
| `!command` | 跑 shell 命令，输出送模型 |
| `!!command` | 跑 shell 命令，输出不送模型 |
| Ctrl+V（Win: Alt+V） | 粘贴图片/文本 |
| Ctrl+L | 选模型 |
| Ctrl+S（picker 内） | 存为默认 |
| Shift+Tab | 循环思考级别 |
| Ctrl+P / Shift+Ctrl+P | 循环 scoped 模型 |

## tmux 测交互模式（Windows 用 WSL/Git Bash 时）

```bash
tmux new-session -d -s pi-test -x 80 -y 24
tmux send-keys -t pi-test "./pi-test.sh" Enter
sleep 3 && tmux capture-pane -t pi-test -p
tmux send-keys -t pi-test "your prompt here" Enter
tmux kill-session -t pi-test
```

Windows 原生 PowerShell 不支持 tmux；要跑 tmux 流程需 Git Bash 或 WSL。

## 红线提醒

| 红线 | 原因 | 正确做法 |
|---|---|---|
| 永不手改 `packages/ai/src/models.generated.ts` | 生成物 | 改 `packages/ai/scripts/generate-models.ts` 再生成 |
| 永不 `git add -A` / `git add .` | 多会话会踩别人工作 | `git add <具体路径>` |
| 永不 `git reset --hard` / `git clean -fd` | 破坏性 | 用 `git status` 谨慎操作 |
| 改代码后不跑 `npm run check` 就提交 | 门禁 | 全绿（error/warning/info 清）才提交 |
| 不擅自跑 `npm run build` / `npm test` | AGENTS.md 规定 | 仅用户要求时执行 |
