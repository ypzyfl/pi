# 阶段 1（仓库结构与工具链）执行路线与进度

本文是阶段 1（含阶段 0 会话锚点）的**执行路线 + 逐步勾选进度**：把 [learning-path.zh.md](../learning-path.zh.md) 阶段 1 的「精读材料 + 动手任务 + 过关检验」拆成可逐步推进的小步骤，并标出每步的验证点与学习区落盘动作。事实源仍是 learning-path.zh.md，本文不重复其内容、只做执行拆解；冲突以 learning-path.zh.md 为准。

过关标准（来自 learning-path.zh.md 完成标志表阶段 1 行）：① 工具链三命令本机通过；② 能画 11 包依赖拓扑；③ 锚点实验完成，能说出三类基本 entry；④ 能复述五个核心术语；⑤ 能复述一条消息穿过各层的路径。

## 路线总览（六步，锚点先行）

```
第 0 步  会话锚点实验 ── 一次真实会话 + 会话 JSONL 逐行对照（阶段 0）
第 1 步  精读三份根文档 ── README / AGENTS / CONTRIBUTING
第 2 步  文档地图 ── docs/index.md + development.md，建立文档分区认知
第 3 步  工具链跑通 ── install / check / test.sh 三命令
第 4 步  依赖拓扑亲手画 ── build 脚本顺序 × package.json 依赖边
第 5 步  .pi/ 目录盘点 ── dogfooding 实例逐文件过一遍
```

## 第 0 步：会话锚点实验（阶段 0）

目标：建立「一次对话 = 一棵 JSONL 事件树」的实物锚点。假设 → 操作 → 观察 → 结论四段式，完成后落盘 `experiments/001-session-anchor.zh.md`。

- [x] 用 `./pi-test.sh`（Windows：`.\pi-test.ps1`）从源码启动 pi，在一个小目录里跑一轮带工具调用的对话（例如让它读一个文件）；无 key 时改跑 `./test.sh` 后在测试产物/fixture 里找一个会话文件替代（实际跑的「hi」轮未触发工具调用；关联机制于 2026-09-08 用 2026-07-30 历史会话补验，见 experiments/001）
- [x] 在 `~/.pi/agent/sessions/` 下找到本次会话的 JSONL 文件（目录名是 cwd 路径转义）
- [x] 对照 [session-format.md](../../packages/coding-agent/docs/session-format.md) 逐行读：SessionHeader、消息 entry（text / thinking / tool call）、toolResult entry；标出 `id` / `parentId` 如何成链
- [x] 回答实验三问：① UI 里的每一轮在文件里是哪些行？② 工具调用与结果分别是哪个 entry、怎么关联？③ 这棵树怎么表达「继续对话」（追问是 append 到链尾？）
- [x] 落盘 `experiments/001-session-anchor.zh.md`（2026-09-08 补录）

## 第 1 步：精读三份根文档

- [x] [README.md](../../README.md)：产品定位、包清单表、开发命令、供应链加固一节扫读（补读了 Permissions & Containerization / standalone binaries / session sharing 三段；途中辨析「agent 还是 harness」，见落盘产出）
- [x] [AGENTS.md](../../AGENTS.md)：精读 Conversational Style、Code Quality、Commands、Git 四节（Commands 与 Git 是后续每次动手的在场规则）（其余六节通读完毕；产出 test-isolation 笔记与全文中文对照 [AGENTS.zh.md](../AGENTS.zh.md)）
- [x] [CONTRIBUTING.md](../../CONTRIBUTING.md)：最小核心哲学、auto-close 政策、`npm run check` + `./test.sh` 门禁（auto-close 经三个 workflow + 白名单文件逐条机制验证；产出 contribution-gate 笔记与全文中文对照 [CONTRIBUTING.zh.md](../CONTRIBUTING.zh.md)）

## 第 2 步：文档地图

- [x] [packages/coding-agent/docs/index.md](../../packages/coding-agent/docs/index.md)：记下六分区各自收什么，后续阶段的入口都从这里走（原记「五分区」漏了 Platform setup——windows / termux / tmux / terminal-setup / shell-aliases，对 Windows 用户恰是重点；learning-path 已同步修正）
- [x] [packages/coding-agent/docs/development.md](../../packages/coding-agent/docs/development.md)：本地开发流程、项目结构、调试手段（remote harness 解释 client/protocol/server 三包宿命；fork piConfig 本 fork 实况已核；/debug 隐藏命令、check:package-install 对应 check 的 runtime-deps / entry-graphs 门禁）
- [x] 扫读 [quickstart.md](../../packages/coding-agent/docs/quickstart.md) 与 [usage.md](../../packages/coding-agent/docs/usage.md)：产品使用面全貌（slash 命令、CLI 参数；steering / follow-up 队列、System Prompt Files、project trust 留待阶段 3 / 5 深挖）

## 第 3 步：工具链跑通

- [x] `npm install --ignore-scripts`（AGENTS.md：不跑生命周期脚本）
- [x] `npm run check` 通过（子门禁组成与 check / test.sh 职责边界见 [test-isolation](../notes/mechanisms/test-isolation.zh.md) 笔记）
- [x] `./test.sh` 经 Git Bash 跑通，隔离机制实证（ai 包 847 个 e2e 测试无 key 自动跳过）；套件在 Windows 存在环境性失败（POSIX 路径假设、symlink EPERM、POSIX 独有机制，另有执行沙箱污染待本机复核），详见 [test-sh-on-windows](../notes/mechanisms/test-sh-on-windows.zh.md) 笔记；途中修复一处磁盘级 rolldown 绑定截断损坏

## 第 4 步：依赖拓扑亲手画

- [x] 读根 [package.json](../../package.json) 的 `build` 脚本，抄下构建顺序（chord → tui → telemetry → ai → agent → sqlite-node → protocol → client → server → coding-agent；evals 不在链上——private 无 build 脚本）
- [x] 逐包打开 package.json 核对 workspace 依赖边，亲手画一张拓扑图（含 dependencies 实线与 devDependencies 虚线、L0–L4 分层，见 [map.zh.md](../map.zh.md)「依赖拓扑」修正版）
- [x] 与 [map.zh.md](../map.zh.md)「依赖拓扑」的已验证版对照：发现三处偏差并已按 package.json 修正——map 原图漏 client → chord、server → chord 两条边；「evals 独立无 workspace 依赖」有误（实有 dev 边 ai + coding-agent）
- [x] 用一句话回答：为什么 build 顺序恰好是依赖拓扑序？——包内 `tsconfig.build.json` 的 paths 把 workspace 依赖解析到**上游 dist 产物**（agent 的 paths 指向 `../ai/dist/index.d.ts`），下游构建前上游产物必须在场；根 build 脚本即此 DAG 的手工线性化（check 与 build 解析面分离：根 tsconfig paths 指向 src，故 check 不依赖 dist）

## 第 5 步：.pi/ 目录盘点（为阶段 5 预热，只盘点不深入）

- [x] 列出 [.pi/](../../.pi/) 下 extensions / prompts / skills / git / npm 五个子目录的文件
- [x] 每个 prompt template（`cl.md`、`deslop.md`、`is.md`、`pr.md`、`sa.md`、`wr.md`）读开头，记一句话用途
- [x] `skills/add-llm-provider.md` 与 `extensions/` 四个文件只登记文件名与一句话猜测，不精读（阶段 5 的活）

## 已完成的落盘产出

（完成时登记：experiments / journal / notes / map 更新 / questions 状态流转。）

- 2026-09-08（第 1 步：README.md 精读完成，含 Permissions / binaries / session sharing 补遗与「agent 还是 harness」之辨）：
  - journal：[journal/2026-09-08-01-readme-harness.zh.md](../journal/2026-09-08-01-readme-harness.zh.md)
  - notes：[notes/architecture/agent-vs-harness.zh.md](../notes/architecture/agent-vs-harness.zh.md)
  - map：提纲新增「总体定性」；依赖拓扑补「README 只列 6/11」；横切补「session-backends 实为目录组」；分层心智模型补总定性段
- 2026-09-08（补录第 0 步：锚点实验落盘，第 0 步五项 checkbox 全部完成）：
  - experiments：[experiments/001-session-anchor.zh.md](../experiments/001-session-anchor.zh.md)（实验执行 2026-09-07；补录时用 2026-07-30 历史会话补验工具调用关联与多轮 append）
  - notes：[session-message-flow.zh.md](../notes/mechanisms/session-message-flow.zh.md) 状态升级——工具调用关联与多轮 append 已实测
  - index：阶段 0 置为完成
- 2026-09-09（第 1 步：根 AGENTS.md 精读完成——四节精读 + 其余六节通读）：
  - notes：[notes/mechanisms/test-isolation.zh.md](../notes/mechanisms/test-isolation.zh.md)（vitest 全量禁令的机制、check / test.sh 职责边界）
  - 对照翻译：[AGENTS.zh.md](../AGENTS.zh.md)（根 AGENTS.md 全文中文对照，10 节）
  - 主线认知：AGENTS.md 本质是副作用控制手册——对真钱、他人未提交工作、装依赖执行的代码、错误发布信号四类不可控副作用系统性设防
- 2026-09-09（第 1 步：CONTRIBUTING.md 精读完成，三份根文档齐，第 1 步收官）：
  - notes：[notes/mechanisms/contribution-gate.zh.md](../notes/mechanisms/contribution-gate.zh.md)（auto-close 的机器执行：三层防线、lgtmi/lgtm 能力不对称、审批即写白名单文件）
  - 对照翻译：[CONTRIBUTING.zh.md](../CONTRIBUTING.zh.md)（根 CONTRIBUTING.md 全文中文对照，9 节）
  - 主线认知：CONTRIBUTING.md 本质是注意力保护手册——四道防线各保一种稀缺资源：最小核心保复杂度预算、auto-close + Quality Bar + Blocking 保维护者时间、The One Rule 保审查成本、check + test.sh 双门禁保合并质量
- 2026-09-10（第 2 步：文档地图完成）：
  - 修正：五分区 → 六分区（index.md 实有六区，learning-path 原漏 Platform setup，两处已改）
  - map：包总数口径提前核销（原挂第 4 步）——根 workspaces = packages/*（10 包）+ session-backends/*（1 子包）= 11 包，另有示例 workspace（coding-agent/examples/extensions/with-deps）；提纲新增「文档布局」
  - 主线认知：文档跟着交付单元走——coding-agent/docs 的 30 篇写给「装 pi 的人」（产品使用面），ai / agent / tui 的 README 写给嵌入者与维护者（库 API 面：ai 1339 行、agent 393 行、tui 704 行）；dsh 对照：框架型仓库 docs/ 在仓库根，产品型仓库文档跟着产品包走
- 2026-09-10（第 3 步：工具链三命令跑通，含一次环境诊断）：
  - notes：[notes/mechanisms/test-sh-on-windows.zh.md](../notes/mechanisms/test-sh-on-windows.zh.md)（Git Bash 执行方式、MSYS 环境变量转换不对称、rolldown 截断损坏诊断案例、Windows 套件失败四分类）
  - 环境修复：node_modules/@rolldown/binding-win32-x64-msvc 磁盘截断损坏，经 registry integrity 校验后用原版覆盖（不触碰 lockfile）
  - 主线认知：test.sh 原生支持 Windows（显式继承 SystemRoot 等变量）；Windows 失败集中于测试的 POSIX 假设而非产品代码；tui imageFallback 分隔符问题是真跨平台 bug（fork 修复候选）
- 2026-09-10（第 4 步：依赖拓扑亲手画完成）：
  - map：依赖拓扑图重画为 L0–L4 分层版（实线 dependencies / 虚线 devDependencies），修正三处——补 client → chord、server → chord 漏边；evals「独立无依赖」改为「private 包有 dev 边」；新增 build 顺序与两套解析面的机制段
  - 主线认知：build 顺序 = DAG 手工线性化，强制力来自包内 tsconfig.build.json 的 paths 指向上游 dist（build 走产物面）；check 的根 tsconfig paths 指向 src（源码面）——两套解析面分离解释了 clean && build && check 的顺序无关性
- 2026-09-12（第 5 步：.pi/ 目录盘点，用户自行完成，无落盘产出；阶段 1 六步齐，过关检验五项全过，正式收官）：
  - index：阶段 1 置为完成（2026-09-12）

## 过关检验自测（完成时逐条打勾）

- [x] ① `npm install --ignore-scripts` / `npm run check` / `./test.sh` 本机通过
- [x] ② 能不看资料画出 11 包依赖拓扑（至少六个关键包的位置与依赖边）
- [x] ③ 锚点实验完成，能说出 SessionHeader / 消息 entry / toolResult entry 三类基本结构
- [x] ④ 能用自己的话复述五个核心术语（AgentSession / agent loop / harness / extension / skill）
- [x] ⑤ 能凭记忆复述一条消息穿过各层的路径
