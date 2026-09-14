<p align="center">
  <a href="https://pi.dev">
    <img alt="pi logo" src="https://pi.dev/logo-auto.svg" width="128">
  </a>
</p>
<p align="center">
  <a href="https://discord.com/invite/3cU7Bz4UPx"><img alt="Discord" src="https://img.shields.io/badge/discord-community-5865F2?style=flat-square&logo=discord&logoColor=white" /></a>
  <a href="https://www.npmjs.com/package/@earendil-works/pi-coding-agent"><img alt="npm" src="https://img.shields.io/npm/v/@earendil-works/pi-coding-agent?style=flat-square" /></a>
</p>

> 来自新贡献者的新 issue 和 PR 默认会被自动关闭。维护者每天审查被自动关闭的 issue。见 [CONTRIBUTING.md](CONTRIBUTING.md)。

# Pi Agent Harness

这是 Pi agent harness 项目的所在地，包括我们的自扩展（self extensible）编码 agent。

* **[@earendil-works/pi-coding-agent](packages/coding-agent)**：交互式编码 agent CLI
* **[@earendil-works/pi-agent-core](packages/agent)**：带工具调用和状态管理的 agent 运行时
* **[@earendil-works/pi-ai](packages/ai)**：统一的多 provider LLM API（OpenAI、Anthropic、Google 等）

了解更多关于 Pi 的信息：

* [访问 pi.dev](https://pi.dev)，项目网站，包含演示
* [阅读文档](https://pi.dev/docs/latest)，你也可以让 agent 解释它自己

## 所有包

| 包 | 描述 |
|---------|-------------|
| **[@earendil-works/chord](packages/chord)** | 独立的应用组合运行时，用于服务、复制状态、RPC 和插件 |
| **[@earendil-works/pi-telemetry](packages/telemetry)** | 供应商中立的遥测契约、参考适配器、一致性测试和类型化 schema |
| **[@earendil-works/pi-ai](packages/ai)** | 统一的多 provider LLM API（OpenAI、Anthropic、Google 等） |
| **[@earendil-works/pi-agent-core](packages/agent)** | 带工具调用和状态管理的 agent 运行时 |
| **[@earendil-works/pi-coding-agent](packages/coding-agent)** | 交互式编码 agent CLI |
| **[@earendil-works/pi-tui](packages/tui)** | 带差分渲染的终端 UI 库 |

Slack/聊天自动化和工作流请见 [earendil-works/pi-chat](https://github.com/earendil-works/pi-chat)。

## 权限与容器化

Pi 不内置用于限制文件系统、进程、网络或凭据访问的权限系统。默认情况下，它以启动它的用户和进程的权限运行。

如果你需要更强的边界，请将 Pi 容器化或沙箱化。三种模式见 [packages/coding-agent/docs/containerization.md](packages/coding-agent/docs/containerization.md)：

- **Gondolin 扩展**：将 `pi` 和 provider 认证保留在主机上，同时将内置工具和 `!` 命令路由到本地 Linux micro-VM。
- **纯 Docker**：将整个 `pi` 进程运行在本地容器中，实现简单隔离。
- **OpenShell**：将整个 `pi` 进程运行在策略控制的沙箱中。

## 贡献

贡献指南见 [CONTRIBUTING.md](CONTRIBUTING.md)，项目特定规则（对人类和 agent 均适用）见 [AGENTS.md](AGENTS.md)。Pi 的长期规划也可以在 [RFCs](https://rfc.earendil.com/keyword/pi/) 中找到。

## 开发

```bash
npm install --ignore-scripts  # 安装所有依赖，不运行生命周期脚本
npm run build         # 刷新模型数据，然后构建所有包
npm run build:offline # 使用现有模型数据重新构建，无需网络访问
npm run check         # Lint、格式化和类型检查
./test.sh            # 运行测试（没有 API key 时跳过依赖 LLM 的测试）
./pi-test.sh         # 从源码运行 pi（可以从任何目录运行）
```

## 从发布源码构建独立二进制文件

GitHub releases 包含带版本的源码归档，由该 release 的 `SHA256SUMS` 文件覆盖。解压并运行与官方独立二进制文件相同的构建脚本：

```bash
VERSION="<release-version>"
tar -xzf "pi-${VERSION}-source.tar.gz"
cd "pi-${VERSION}"
./scripts/build-binaries.sh --offline-model-data --platform linux-x64 --out "$PWD/out"
```

该归档包含发布模型数据和原生预构建（prebuild）。`--offline-model-data` 使用该模型数据而不刷新 provider 目录。脚本会安装依赖并构建可执行文件及其运行时资产；如果依赖已提供，传入 `--skip-install`。

## 供应链加固

我们将 npm 依赖变更视为经过审查的代码变更。

- 直接外部依赖固定到精确版本。内部 workspace 包保持版本范围。
- `.npmrc` 设置 `save-exact=true` 和 `min-release-age=2`，避免 npm 解析时引入当日发布的依赖。
- `package-lock.json` 是依赖的基准事实（ground truth）。除非设置 `PI_ALLOW_LOCKFILE_CHANGE=1`，pre-commit 会阻止意外的 lockfile 提交。
- `npm run check` 验证固定的直接依赖、原生 TypeScript 导入兼容性以及生成的 coding-agent shrinkwrap。
- 发布的 CLI 包包含 `packages/coding-agent/npm-shrinkwrap.json`，从根 lockfile 生成，为 npm 用户固定传递依赖。
- 发布冒烟测试使用 `npm run release:local` 构建、打包，并在打 release 标签之前在仓库外创建隔离的 npm 和 Bun 安装。
- 本地 release 安装、文档中的 npm 安装以及 `pi update --self` 在支持的地方使用 `--ignore-scripts`。
- CI 使用 `npm ci --ignore-scripts` 安装，并且一个定时 GitHub workflow 运行 `npm audit --omit=dev` 和 `npm audit signatures --omit=dev`。
- Shrinkwrap 生成对依赖生命周期脚本有显式白名单；新的带生命周期脚本的依赖在审查前会失败检查。

## 分享你的 OSS 编码 agent 会话

如果你将 Pi 或其他编码 agent 用于开源工作，请分享你的会话。

公开的 OSS 会话数据帮助改进编码 agent，使用真实世界的任务、工具使用、失败与修复，而不是玩具基准。

完整说明见[这篇 X 上的帖子](https://x.com/badlogicgames/status/2037811643774652911)。

要发布会话，使用 [`badlogic/pi-share-hf`](https://github.com/badlogic/pi-share-hf)。阅读其 README.md 了解设置说明。你只需要一个 Hugging Face 账户、Hugging Face CLI 和 `pi-share-hf`。

你也可以观看[这个视频](https://x.com/badlogicgames/status/2041151967695634619)，我展示了如何发布我的 `pi-mono` 会话。

我定期在这里发布我自己的 `pi-mono` 工作会话：

- [badlogicgames/pi-mono on Hugging Face](https://huggingface.co/datasets/badlogicgames/pi-mono)

## 许可证

MIT

<p align="center">
  <a href="https://pi.dev">pi.dev</a> 域名慷慨捐赠自
  <br /><br />
  <a href="https://exe.dev"><img src="packages/coding-agent/docs/images/exy.png" alt="Exy mascot" width="48" /><br />exe.dev</a>
</p>
