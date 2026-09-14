# 测试隔离：为什么禁止直接跑全量 vitest，check 与 test.sh 的边界

状态: 已对照验证（2026-09-09 对照 test.sh、根 package.json、AGENTS.md Commands 节、packages/coding-agent/test/utilities.ts 与多个测试文件）

## 事实源（链接，不复述）

- [AGENTS.md](../../../AGENTS.md) L31–35（Commands 节：check 门禁、vitest 全量禁令、单测跑法）
- [test.sh](../../../test.sh)（隔离环境全量测试的唯一入口）
- [package.json](../../../package.json) L21（`check` 的组成）、L39–40（`test` 的组成）
- [utilities.ts](../../../packages/coding-agent/test/utilities.ts) L32（`API_KEY` 定义）、L81–104（`resolveApiKey` 读 `~/.pi/agent/auth.json`）
- [rpc.test.ts](../../../packages/coding-agent/test/rpc.test.ts) L14（`describe.skipIf` 激活模式示例）

## 它是什么（用自己的话，≤5 句）

「禁止直接跑全量 vitest」的真实原因是环境不可控，不是 vitest 本身有问题：e2e 测试的激活开关就是「钥匙存在与否」。开关定义在 [utilities.ts L32](../../../packages/coding-agent/test/utilities.ts)：`API_KEY = process.env.ANTHROPIC_OAUTH_TOKEN || process.env.ANTHROPIC_API_KEY`，测试块用 `describe.skipIf(!API_KEY)` 自选激活——无 key 自动跳过，有 key 对真实 API 跑 e2e（烧真钱、结果不可复现）。钥匙有两条入口：shell 环境变量，以及 `~/.pi/agent/auth.json`（`resolveApiKey` 会读真实凭证文件，本机确有 deepseek key 存于此）。`./test.sh` 用 `env -i` 白名单 + HOME 重定向把两条入口同时堵死，才在隔离环境里跑 `npm test`；`npm run check` 与测试执行零重叠——它是静态门禁（lint / 格式 / 类型 / 依赖契约），回答「代码形式合规吗」，test.sh 回答「代码行为正确吗」。

## 激活开关与钥匙入口

开关（[utilities.ts L32](../../../packages/coding-agent/test/utilities.ts)）：

```ts
export const API_KEY = process.env.ANTHROPIC_OAUTH_TOKEN || process.env.ANTHROPIC_API_KEY;
```

7 个 key 激活测试块（`describe.skipIf` 模式）：agent-session-compaction（L24）、agent-session-branching（L27）、agent-session-tree-navigation（L15、L279）、compaction.test（L544，仅看 `ANTHROPIC_OAUTH_TOKEN`）、compaction-extensions（L29）、rpc（L14）。

钥匙的两条入口：

1. shell 环境变量——直接跑 vitest 时进程继承你的环境，任何 `ANTHROPIC_*` 变量都会激活全部 e2e 块。
2. `~/.pi/agent/auth.json`——[resolveApiKey](../../../packages/coding-agent/test/utilities.ts) 从 `homedir()` 拼出路径读取真实凭证；即使环境变量干净，测试也可能从 HOME 里找到真钥匙（OAuth 凭证还会自动刷新并写回）。

## test.sh 的防线

| 防线 | 行 | 挡住什么 |
|---|---|---|
| `env -i` + 白名单 | [L40–64](../../../test.sh)、L79 | 从空环境开始，shell 里的一切变量（含 API key）全部剥掉 |
| `HOME` / `USERPROFILE` 指向临时目录 | L43–44 | `homedir()` 被重定向，`~/.pi/agent/auth.json` 不存在——auth.json 入口被切断 |
| `PI_NO_LOCAL_LLM=1` | L62 | 不探测 / 使用本机 LLM |
| `AWS_EC2_METADATA_DISABLED=true` | L63 | 不探测云元数据端点（可能挂起或泄漏） |
| `GIT_ASKPASS=false`、`GIT_TERMINAL_PROMPT=0` | L55–56 | git 凭证交互直接失败而不是挂死测试 |
| git / npm 全部指向隔离配置 | L53–61 | 测试不读也不污染真实 git 配置与 npm 凭证 |

## check 与 test.sh 的职责边界

一句话：**check 验代码的「形」，test.sh 验代码的「行」**，互不重叠。

| | `npm run check` | `./test.sh` |
|---|---|---|
| 性质 | 静态门禁 | 行为验证（隔离环境全量测试） |
| 组成（[package.json L21](../../../package.json)、L39–40） | biome、pinned-deps、runtime-deps、ts-imports、entry-graphs、shrinkwrap、install-lock、`tsgo --noEmit`、browser-smoke | `npm test` = scripts 测试 + 全部 workspace 测试 |
| 回答的问题 | 代码形式合规吗？类型对吗？依赖钉死了吗？ | 代码行为正确吗？ |
| 执行任何测试吗 | 否（AGENTS.md L31 明说 "Does not run tests"） | 只做这个 |
| 时机 | 每次代码改动后（docs 除外），提交前修完所有 error / warning / info | 需要验证行为时；CI 全量 |

第三条路：开发中只跑单个测试文件，从包根直接跑（AGENTS.md L34–35）——vitest 用根 node_modules 二进制 `--run test/specific.test.ts`；tui 用 `node --test`。单文件通常不碰 e2e 块，目标小、无环境问题。

```
日常开发循环：  改代码 → npm run check（形） → 跑相关单测（点）
提交前 / CI：  ./test.sh（行，全量，隔离）
永远不直接跑：  全量 vitest（环境不可控）
```

## 附带观察

- skipIf 还有平台维度：测试按 `process.platform` 自选（win32 的 powershell-tool、bash-close-hang；非 win32 的 auth-storage 权限位测试）——测试套件天然按平台分层。
- 「全量 vitest」≠ 全量测试：根 `npm test` = `scripts/*.test.mjs`（node --test）+ 各 workspace 自己的 test 脚本；tui 用 `node:test`，scripts 测试也在 vitest 之外。全量只有一个定义：`./test.sh`。

## 容易产生的误解

原以为：禁令针对 vitest 工具本身（不稳定或太慢）。
实际是：针对「你的环境」——钥匙从环境变量与 auth.json 两个入口漏进测试进程；环境隔离之后跑什么都安全。
修正来源：2026-09-09 AGENTS.md Commands 节精读问答。

## 验证方式

- 找激活开关：在 packages/coding-agent/test 下搜 `skipIf`（7 个 key 类 describe 块 + 若干平台类）。
- 开关定义：packages/coding-agent/test/utilities.ts L32。
- 隔离实现：test.sh L40–64（白名单）、L79（`env -i ... npm test`）。

## 遗留问题

无新增。
