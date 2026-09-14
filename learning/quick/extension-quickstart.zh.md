# 插件开发快速上手

**目标**：45 分钟内亲手写出 extension + skill + prompt template 三件套，全部在 `.pi/` 或临时目录，不动 `packages/`。

**核心认知**：pi 扩展无需编译。扩展是默认导出的 TS 工厂函数，经 [jiti](https://github.com/unjs/jiti) 运行时加载，改完 `/reload` 即生效。

## 三件套总览

| 形态 | 文件 | 放哪（项目级 / 全局） | 生效方式 |
|---|---|---|---|
| extension | `.ts`，默认导出工厂函数 | `<cwd>/.pi/extensions/hello.ts` / `~/.pi/agent/extensions/hello.ts` | 启动自动发现，`/reload` 热重载 |
| skill | `SKILL.md` + frontmatter | `<cwd>/.pi/skills/` / `~/.pi/agent/skills/` | 描述进系统提示词，正文按需加载 |
| prompt template | `.md` + frontmatter | `<cwd>/.pi/prompts/hi.md` / `~/.pi/agent/prompts/hi.md` | 输入 `/hi` 展开 |

## 三件套一：最小 extension

照 [../../.pi/extensions/tps.ts](../../.pi/extensions/tps.ts) 抄一个最小可用扩展。

> **路径别搞错（常见坑）**：本文的 `.pi/` 指**当前项目目录**下的 `.pi`。全局扩展必须放 **`~/.pi/agent/extensions/`**——中间有 `agent` 一层；写成 `~/.pi/extensions/` **不会被扫描**（Windows：`C:\Users\<你>\.pi\agent\extensions\`）。pi 只扫描两处（源码 `loader.ts`）：
>
> | 位置 | 路径 | 生效条件 |
> |---|---|---|
> | 项目级 | `<cwd>/.pi/extensions/` | 项目被信任后加载 |
> | 全局 | `~/.pi/agent/extensions/` | 所有项目，无需信任 |
>
> 二选一，创建 `<项目>/.pi/extensions/hello.ts` 或 `~/.pi/agent/extensions/hello.ts`：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // 1. 事件订阅：会话启动时通知
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Hello extension loaded!", "info");
  });

  // 2. 注册自定义工具：LLM 可调用
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "Greet someone by name",
    parameters: Type.Object({
      name: Type.String({ description: "Name to greet" }),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `Hello, ${params.name}!` }],
        details: {},
      };
    },
  });

  // 3. 注册 slash 命令
  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => {
      ctx.ui.notify(`Hello ${args || "world"}!`, "info");
    },
  });
}
```

**验证**：

```powershell
# 仓库根目录跑起 pi
.\pi-test.ps1
# 在会话里输入
/hello              # 看到 notify
# 或让模型调用工具
请用 greet 工具向 pi 打招呼
```

**临时测试（不写文件）**：

```powershell
.\pi-test.ps1 -e .\my-temp-extension.ts
```

`-e` / `--extension` 只对本次运行生效，不持久化。

## 三件套二：最小 skill

skill 是 `SKILL.md` + frontmatter，目录结构自由。创建 `.pi/skills/word-count/SKILL.md`：

````markdown
---
name: word-count
description: Count words in a file. Use when the user asks to count words or analyze text length.
---

# Word Count

## Usage

Count words in a file using PowerShell:

```bash
(Get-Content <file> -Raw).Split(' ').Count
```

## Notes

- Empty strings count as zero
- Use `-Raw` to read the whole file at once
````

**验证**：

```powershell
.\pi-test.ps1
# 在会话里
/skill:word-count             # 强制加载
# 或让模型按需加载
帮我数一下 README.md 有多少词
```

**关键点**：`description` 决定模型何时加载这个 skill——要具体，"Helps with PDFs" 是差的，"Extracts text and tables from PDF files. Use when working with PDF documents." 是好的。

## 三件套三：最小 prompt template

prompt template 是带 frontmatter 的 `.md`，文件名即命令名。创建 `.pi/prompts/wc.md`：

```markdown
---
description: Count words in a file
argument-hint: "<file-path>"
---
Count the words in $1 and report the total. Use the word-count skill if needed.
```

**验证**：

```powershell
.\pi-test.ps1
# 在会话里
/wc README.md
```

**参数语法**：

| 写法 | 含义 |
|---|---|
| `$1`, `$2` | 位置参数 |
| `$@` 或 `$ARGUMENTS` | 所有参数拼接 |
| `${1:-default}` | 有则用，无则用 default |
| `${@:N}` | 从第 N 个起的所有参数 |

## 三件套进阶：事件钩子拦截

拦截危险命令（参考 [../../packages/coding-agent/docs/extensions.md](../../packages/coding-agent/docs/extensions.md) Quick Start 示例）：

```typescript
pi.on("tool_call", async (event, ctx) => {
  if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
    const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
    if (!ok) return { block: true, reason: "Blocked by user" };
  }
});
```

返回 `{ block: true, reason }` 阻止调用；返回正常对象修改输入。

## 三件套进阶：打包成 pi package

**为什么要打包**：直接放 `.pi/extensions/*.ts` 只适合自用与单项目；一旦要**复用、共享、版本化**，package 才是分发单元。

| 好处 | 说明 |
|---|---|
| 一个单元带四类资源 | 一次打包 extension + skill + prompt template + theme，安装一次全部就位，不必逐个复制文件 |
| 多渠道分发 | npm（版本可 pin）/ git（tag 或 commit 固定）/ local（原地引用、不复制） |
| 统一生命周期 | `pi install` / `remove` / `list` / `update --all` 一套命令管理整组资源 |
| 依赖自动就位 | npm/git 包安装时自动 `npm install`，`dependencies` 里的三方运行时依赖自动装 |
| 团队共享 | 项目级安装（`-l` → `.pi/settings.json`）落入仓库，项目信任后启动自动安装 |
| 精确过滤与开关 | `pi` 对象形式按 glob 选 / 排除资源（`!` 排除、`+` / `-` 强制）；`pi config` 交互式启用或禁用单个资源 |
| 可发现 | 加 `pi-package` keyword 进 pi.dev/packages 画廊，可配 video / image 预览 |
| 模块隔离 | 每个包独立 module root，多包共存不冲突、不共享模块 |

**什么时候不必打包**：自用、单项目、频繁改动 → 直接放 `.pi/extensions/*.ts`，`/reload` 即生效，比打包快得多。打包的收益只在"要给别人用 / 要跨项目复用 / 要版本化"时才体现。

打包必看的两条规则（详见 [packages.md](../../packages/coding-agent/docs/packages.md)）：

- **核心包不要打包**：import 这些要写进 `peerDependencies` 且范围 `"*"`——`@earendil-works/pi-ai`、`@earendil-works/pi-agent-core`、`@earendil-works/pi-coding-agent`、`@earendil-works/pi-tui`、`typebox`（pi 自带）。
- **三方运行时依赖**放 `dependencies`；要打包的其他 pi 包放 `dependencies` + `bundledDependencies` 并通过 `node_modules/` 路径引用。

把三件套打成一个目录，加 `package.json` 声明 `pi` 字段：

```json
{
  "name": "my-pi-pack",
  "version": "1.0.0",
  "pi": {
    "extensions": ["./extensions/hello.ts"],
    "skills": ["./skills/word-count"],
    "prompts": ["./prompts/wc.md"]
  }
}
```

安装与卸载：

```powershell
.\pi-test.ps1 install .\my-pi-pack      # 本地路径
.\pi-test.ps1 install npm:@scope/pkg@1.0.0  # npm
.\pi-test.ps1 install git:github.com/user/repo@v1  # git
.\pi-test.ps1 remove npm:@scope/pkg
.\pi-test.ps1 list                       # 看装了什么
```

三种来源的安装落点：

| 来源 | 命令前缀 | 落点 |
|---|---|---|
| npm | `npm:` | `~/.pi/agent/npm/` |
| git | `git:` | `~/.pi/agent/git/` |
| local | 路径 | 原地引用，不复制 |

## 可用导入

| 包 | 用途 |
|---|---|
| `@earendil-works/pi-coding-agent` | `ExtensionAPI`、`ExtensionContext`、事件类型 |
| `typebox` | 工具参数 schema（`Type.Object`、`Type.String`） |
| `@earendil-works/pi-ai` | `StringEnum`（Google 兼容枚举）、AI 工具 |
| `@earendil-works/pi-tui` | 自定义 TUI 组件 |
| `node:fs`、`node:path` 等 | Node 内置 |

npm 依赖也行：在扩展目录旁放 `package.json`，`npm install` 后从 `node_modules/` 导入。

## 扩展位置与发现

| 位置 | 作用域 |
|---|---|
| `~/.pi/agent/extensions/*.ts` | 全局（所有项目） |
| `~/.pi/agent/extensions/*/index.ts` | 全局（子目录） |
| `.pi/extensions/*.ts` | 项目级（信任后加载） |
| `.pi/extensions/*/index.ts` | 项目级（子目录） |
| `pi -e ./path.ts` | 临时（本次运行） |

**项目信任**：`.pi/` 下的资源只在项目被信任后才加载。首次启动会提示信任。

## 调试技巧

- `/debug`（隐藏命令）：写渲染行与发给 LLM 的消息到 `~/.pi/agent/pi-debug.log`
- `/reload`：热重载 `.pi/extensions/` 与 `~/.pi/agent/extensions/` 下的扩展
- `ctx.ui.notify(msg, "info"|"warn"|"error")`：在状态栏显示消息
- `ctx.ui.setStatus("my-ext", "Processing...")`：footer 状态
- `ctx.ui.setWidget("my-ext", ["Line 1", "Line 2"])`：编辑器上方 widget
- `ctx.ui.custom(component)`：完整自定义 TUI 组件

## 常见坑

| 坑 | 原因 | 解法 |
|---|---|---|
| 改了 extension 没生效 | 未 reload | 会话内 `/reload` |
| `.pi/` 下的扩展不加载 | 项目未信任 | 首次启动确认信任 |
| 工具参数 schema 报错 | typebox 用错 | 参考 `tps.ts` 与 `extensions.md` |
| `models.generated.ts` 改了出问题 | 红线文件不可手改 | 走 `packages/ai/scripts/generate-models.ts` 再生成 |
| git 依赖安装慢 | `min-release-age=2` 限制 | 用 `npm_config_min_release_age=0` 仅在必要时 |

## 完整 API 参考

本文只覆盖快速上手。完整 API（全部事件、ExtensionContext、ExtensionAPI 方法、Custom UI、State Management、Mode Behavior）见 [../../packages/coding-agent/docs/extensions.md](../../packages/coding-agent/docs/extensions.md)（3024 行，按需查章节）。
