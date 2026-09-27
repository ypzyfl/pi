# TypeScript 7 工具链迁移笔记

状态：已对照验证（2026-09-27 对照 [package.json](../../../package.json)、[tsconfig.base.json](../../../tsconfig.base.json)、[source-resolver.ts](../../../packages/coding-agent/src/experimental/source-resolver.ts)、commit ca7460d16）

## 事实源（链接，不复述）

- [package.json](../../../package.json)：`check` 脚本（`tsc --noEmit`）、devDependencies（`typescript@7.0.2`，移除 `@typescript/native-preview` 与 `tsx`）
- [tsconfig.base.json](../../../tsconfig.base.json)：`target` ES2024、`verbatimModuleSyntax`、移除装饰器三选项
- [source-resolver.ts](../../../packages/coding-agent/src/experimental/source-resolver.ts)（87 行）：运行时源码解析 hook
- [agent/package.json](../../../packages/agent/package.json)：`node --import ../coding-agent/src/experimental/source-resolver.ts ...` 用法

## 它是什么（≤5 句）

TypeScript 7（native 编译器，Go 重写）从预览包转正，CLI 从 `tsgo` 回归 `tsc`；同时移除 `tsx`，改用「Node 内置 type stripping + 自定义 source resolver hook」直跑 `.ts` 源码。配套收紧 tsconfig 到「只留可擦除语法」：`target` 升 ES2024、启用 `verbatimModuleSyntax`、删除装饰器选项。本质是「检查/构建/运行」三套解析面里，「运行」这一面也统一到 src，并清掉所有需要 emit 的 TS 语法。

## 关键实体

- **`tsgo` → `tsc`**：native 编译器转正，不是换命令。`@typescript/native-preview@7.0.0-dev...`（CLI `tsgo`）被 `typescript@7.0.2`（CLI `tsc`）取代。
- **type stripping**：Node 22.19+ 原生剥离类型注解直接运行 `.ts`。
- **source resolver hook**：`source-resolver.ts` 用 `registerHooks({ resolve })` 把 `@earendil-works/*` 的 import 解析到 tsconfig `paths` 指向的 src，`shortCircuit: true` 掐断默认 dist 解析。
- **`tsx`（移除）**：之前用 esbuild 转译运行 TS，现在职责被上面两者分担。

## 机制：为什么直跑源码需要 resolver hook

Node 能剥类型，但**不应用 tsconfig `paths` 别名**。运行 `import { X } from "@earendil-works/pi-ai"` 时，Node 按 package.json `exports["."].import` 找到 `./dist/index.js`（编译产物）——src 改了没 build 就是旧的。resolver hook 三步：① 启动读 `paths`、过滤 `@earendil-works/` 开头、按 pattern 长度降序；② `resolve` 里匹配 specifier（精确 / 通配）；③ 扩展名换挡（`.js`→`.ts` 等）后 `existsSync` 确认。两个关键设计：`shortCircuit: true`（不回退默认解析）；匹配到 `@earendil-works/` 却找不到文件就 **throw**（快速失败，不静默落到 dist）。

## 与相邻单元的关系

- 是 `map.zh.md`「check 走 src、build 走 dist」两套解析面的**运行时延伸**——直跑源码也走 src。
- 与根 `AGENTS.md`「只用可擦除 TS 语法（Node strip-only 模式）」一体两面：`verbatimModuleSyntax` 保证 import 可被直接 strip。
- 工具链门禁 `check-ts-relative-imports.mjs` / `check-runtime-deps.mjs` 被 port 到 TS7 API，说明 TS7 编译 API 与 5.x 不完全兼容。

## 我曾经的误解（原以为 → 实际是 → 修正来源）

- 原以为 `tsgo`→`tsc` 是换命令名 → 实际是 native 编译器从预览转正，CLI 回归 `tsc` → commit ca7460d16 message。
- 原以为「run sources with plain node」= Node 内置 type stripping 就够了 → 实际还需 resolver hook 处理 `paths` 别名，否则落到过期 dist → `source-resolver.ts` 头注释。

## 验证方式

- `git show ca7460d16 -- package.json tsconfig.base.json` 看 diff。
- `read_file` 读 `source-resolver.ts` 全文，对照 `tsconfig.json` 的 `paths` 与 `packages/ai/package.json` 的 `exports`（无 `"source"` 条件）。

## 遗留问题

- `--conditions=source`（durable bench 用）与 resolver hook 的分工：前者依赖包暴露 `"source"` 条件（ai 包没有），后者通用；两者并存是否只是历史遗留，待观察。
- `check-ts-relative-imports.mjs` / `check-runtime-deps.mjs` 的 TS7 API 移植细节未展开。
