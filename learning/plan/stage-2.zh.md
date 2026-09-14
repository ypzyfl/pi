# 阶段 2（ai 包：统一 LLM API）执行路线与进度

本文是阶段 2 的**执行路线 + 逐步勾选进度**：把 [learning-path.zh.md](../learning-path.zh.md) 阶段 2 的「精读材料 + 动手任务 + 过关检验」拆成可逐步推进的小步骤，并标出每步的验证点与学习区落盘动作。事实源仍是 learning-path.zh.md，本文不重复其内容、只做执行拆解；冲突以 learning-path.zh.md 为准。

过关标准（来自 learning-path.zh.md 完成标志表阶段 2 行）：① 能复述一次模型调用的统一数据流（request → 流式事件 → 聚合 message）；② 能解释 `models.generated.ts` 的生成链路与「不可手改」规则，说出改模型目录的正确入口（`generate-models.ts`）；③ 能指出新增自定义模型 / 自定义 provider 的文档入口。

## 路线总览（六步，观察先行）

```
第 1 步  观察先行 + 布局盘点 ── --list-models 输出 + src/ 顶层文件职责
第 2 步  README 精读 ── packages/ai/README.md（嵌入者视角全景）
第 3 步  统一词汇与流式类型 ── types.ts + api/（服务过关检验①）
第 4 步  provider 适配层 ── providers/ + auth/ + models.ts 门面
第 5 步  模型目录生成链路 ── scripts/generate-models.ts + data 源（服务过关检验②）
第 6 步  产品配置面 + 一次真实调用 ── 文档三篇 + 最小调用 / faux 替身（服务过关检验③）
```

## 第 1 步：观察先行 + 布局盘点

目标：先拿到两个实物印象——模型目录长什么样、ai 包的源码骨架长什么样——再进精读。

- [ ] `.\pi-test.ps1 --list-models`（或 `./pi-test.sh --list-models`）跑一遍，观察输出的组织方式（provider 分组、模型条目字段）
- [ ] 盘点 [packages/ai/src](../../packages/ai/src) 顶层：`index.ts`（出口）、`types.ts`（词汇）、`models.ts` / `model-catalog.ts` / `models-store.ts`（模型目录三件套）、`auth/`（认证）、`api/`（API 协议适配）、`providers/`（provider 适配）、`compat/` + `compat.ts`、`oauth.ts`、`cli.ts`（bin：`pi-ai`）
- [ ] 对照 [package.json](../../packages/ai/package.json) 的 `exports` 面与 scripts（`generate-models` / `check:model-data` / `build` 先 generate 再 offline）记一句话：这个包对外暴露什么、构建时先做什么

## 第 2 步：README 精读

- [ ] [packages/ai/README.md](../../packages/ai/README.md)（1339 行，阶段 1 勘察结论：写给嵌入者与维护者的库 API 面文档）通读一遍，重点：统一 API 的使用示例、streaming 词汇、provider 注册方式、认证机制

## 第 3 步：统一词汇与流式类型（过关检验① 的主场）

目标：读透「一次模型调用的统一数据流」的事实源。核心检查点（均在 [types.ts](../../packages/ai/src/types.ts)，符号已核实在场）：

- [ ] 消息三形态：`UserMessage` / `AssistantMessage` / `ToolResultMessage` 与 `Message` 联合；content 块（`TextContent` / `ThinkingContent` / `ImageContent`）与 `ToolCall`
- [ ] 请求侧：`Context`（systemPrompt + messages + tools）、`StreamOptions`、`Tool` schema、`StopReason`、`Usage`
- [ ] 流式侧：`StreamFunction` 签名、`AssistantMessageEvent`（流式事件的统一词汇）、`ProviderStreams`（stream / streamSimple / fetchDeferred / cancelDeferred 四出口）
- [ ] API 词汇：`KnownApi` / `ApiOptionsMap` 列出的 API 种类（openai-completions / openai-responses / anthropic-messages / google-generative-ai 等），及各 `*Compat` 开关族（`OpenAICompletionsCompat` / `AnthropicMessagesCompat`…）——provider 差异被收敛成开关位
- [ ] 浏览 [api/](../../packages/ai/src/api/) 目录，对照 `ApiOptionsMap` 确认每种 API 的适配文件，弄清「统一事件 ← 各家协议流」的翻译发生在哪层
- [ ] 用一句话回答：request → 流式事件 → 聚合 message，各段分别由哪个类型/函数承载？

## 第 4 步：provider 适配层

- [ ] [providers/](../../packages/ai/src/providers/) 的双文件模式：每 provider 一对 `xxx.ts`（适配器）+ `xxx.models.ts`（静态目录）；数一遍有多少对、`all.ts` 怎么聚合
- [ ] `data/` 下 39 个 JSON 与 `data-json.d.ts`：静态模型数据的存放方式
- [ ] [faux.ts](../../packages/ai/src/providers/faux.ts)：无 key 时的替身 provider（第 6 步无 key 路线的事实源）
- [ ] [models.ts](../../packages/ai/src/models.ts) 的 `Models` 门面：`getProviders` / `getModel` / `stream` / `complete` / `login` / `logout` / `refresh` / `getAvailable`——认证（`auth/`、`oauth.ts`）与模型发现（`refreshModels` / `ModelsPublication`）挂在哪里
- [ ] `createProvider`：自定义 provider 怎么注册（`models.ts` 底部，为过关检验③备料）

## 第 5 步：模型目录生成链路（过关检验② 的主场）

- [ ] 精读 [scripts/generate-models.ts](../../packages/ai/scripts/generate-models.ts)，连带 `model-data.ts` / `check-model-data.ts`：数据源 → 生成 → 校验三段
- [ ] 对照 [models.generated.ts](../../packages/ai/src/models.generated.ts) 文件头与任选一个 provider 的段落（动手任务②）
- [ ] 回答：AGENTS.md 红线「永不手改 models.generated.ts」的机制闭环是什么——改目录的正确入口在哪、`npm run check` 里哪道子门禁会拦住漂移（对照阶段 1 的 test-isolation 笔记）

## 第 6 步：产品配置面 + 一次真实调用（过关检验③ 的主场）

- [ ] [models.md](../../packages/coding-agent/docs/models.md)：自定义模型条目怎么加
- [ ] [providers.md](../../packages/coding-agent/docs/providers.md)：内置 provider 与认证
- [ ] [custom-provider.md](../../packages/coding-agent/docs/custom-provider.md)：自定义 provider 与 OAuth 流程
- [ ] `.\pi-test.ps1 -p "Say exactly: ok"` 跑一次最小真实调用（需 key），观察流式输出；无 key 则精读 [packages/agent/test/agent-loop.test.ts](../../packages/agent/test/agent-loop.test.ts) 的 faux provider 用法，对照 `providers/faux.ts` 说清替身怎么工作

## 已完成的落盘产出

（完成时登记：experiments / journal / notes / map 更新 / questions 状态流转。）

## 过关检验自测（完成时逐条打勾）

- [ ] ① 能复述一次模型调用的统一数据流（request → 流式事件 → 聚合 message）
- [ ] ② 能解释 `models.generated.ts` 的生成链路与「不可手改」规则，说出改模型目录的正确入口（`generate-models.ts`）
- [ ] ③ 能指出新增自定义模型 / 自定义 provider 的文档入口
