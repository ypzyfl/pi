# 贡献门槛：auto-close 政策的机器执行

状态: 已对照验证（2026-09-09 对照 CONTRIBUTING.md 与 .github/ 下 issue-gate / pr-gate / approve-contributor 三个 workflow、APPROVED_CONTRIBUTORS、三个 issue 模板）

## 事实源（链接，不复述）

- [CONTRIBUTING.md](../../../CONTRIBUTING.md) L21–34（Contribution Gate 节）、L36–54（Quality Bar / Blocking）、L56–71（PR 前置与双门禁）
- [issue-gate.yml](../../../.github/workflows/issue-gate.yml)（issue 侧 auto-close）
- [pr-gate.yml](../../../.github/workflows/pr-gate.yml)（PR 侧 auto-close）
- [approve-contributor.yml](../../../.github/workflows/approve-contributor.yml)（lgtm / lgtmi 的审批执行）
- [APPROVED_CONTRIBUTORS](../../../.github/APPROVED_CONTRIBUTORS)（白名单文件本体）

## 它是什么（用自己的话，≤5 句）

「新贡献者的 issue 和 PR 默认自动关闭」不是口头政策，是三个 GitHub Actions workflow 加一个进 git 的白名单文件在机器执行。issue 打开时 issue-gate 依次查三层——受信 bot、协作者权限、白名单——未命中则评论说明、打 `untriaged` 标签、以 `not_planned` 关闭；PR 打开时 pr-gate 只认白名单里的 `pr` 能力。维护者在任意 issue 评论 `lgtm` / `lgtmi`（命令位置由正则强制：开头或结尾），approve-contributor 验证评论者有写权限后更新白名单、由 github-actions[bot] 提交推送。`lgtmi` → `issue` 能力（今后的 issue 存活），`lgtm` → `pr` 能力（issue 与 PR 都存活），且 `pr` 不因后续 `lgtmi` 降级。整个准入系统零基础设施：没有数据库、没有管理界面，状态就是一个可 diff、可审计、可 revert 的纯文本文件。

## 三层防线（issue 与 PR 同构）

| 层 | issue-gate | pr-gate | 放行条件 |
|---|---|---|---|
| 受信 bot | L20 | L21 | `dependabot[bot]` / `sentry[bot]` / `claude[bot]` 直接跳过 |
| 协作者 | L84–88 | L101–105 | GitHub 权限 admin / maintain / write |
| 白名单 | L90–97 | L107–114 | APPROVED_CONTRIBUTORS 条目（见下节能力不对称） |

未命中后果：issue-gate L99–129 评论 + `untriaged` 标签 + close(not_planned)；pr-gate L116–126 评论 + close。两侧自动关闭评论的措辞与 CONTRIBUTING.md 逐句对应——workflow 是文档的机器镜像。

## lgtmi / lgtm 的能力不对称是代码强制的

- issue 存活：issue-gate L94 `capability === 'issue' || capability === 'pr'`——两种能力皆可
- PR 存活：pr-gate L111 `capability === 'pr'`——仅 lgtm

文档「`lgtmi` 不授予提交 PR 的权利。只有 `lgtm` 授予」不是提示，是两个 if 的区别。命令解析在 approve-contributor L43 直接映射：`lgtmi` → `issue`，`lgtm` → `pr`。

## 审批的机械执行（approve-contributor.yml）

1. 命令位置正则（L33–35）：`lgtm` / `lgtmi` 必须在评论开头（前面允许 `@mention`，逗号 / 冒号 / 空格分隔）或结尾，大小写不敏感——文档说的 command position 是正则强制的
2. 评论者权限检查（L46–61）：必须 admin / maintain / write——普通人刷 `lgtm` 无效
3. 目标解析（L135–136）：评论里 @ 了谁就批准谁，一条评论可批量批准多人；没有 @ 则批准 issue 作者
4. 写文件 + 提交（L173、L176–183）：更新 APPROVED_CONTRIBUTORS，`github-actions[bot]` 提交，提交信息 `chore: approve contributors from issue #N`
5. `pr` 粘性（L145）：已有 `pr` 能力的人再收到 `lgtmi` 不降级（alreadyTargets 分支）——权限只升不降

## 设计选择

- **Default-deny + 人工捞回**：与安全 allowlist 同构。默认关闭不是惩罚是缓冲——FAQ 第一问自答：issue 量超过维护者实时审查能力，auto-close 让维护者按自己的节奏捞回有价值的
- **状态是文件，不是数据库**：审批状态是进 git 历史的纯文本（约 260 个条目，几乎全是 `pr`，仅 crisog、npupko 两个 `issue`）。可 diff、可审计、可 revert，机器人直接提交
- **pr-gate 用 `pull_request_target`**（[pr-gate.yml](../../../.github/workflows/pr-gate.yml) L4，非 `pull_request`）：在 base 仓库上下文运行、拿得到写权限，fork 来的 PR 也被门禁覆盖
- **`claude[bot]` 在受信名单**：Anthropic GitHub 集成发的 PR 免检

## 附带观察

- [bug.yml](../../../.github/ISSUE_TEMPLATE/bug.yml) 要求报 core bug 前先用 `pi -ne` 排除扩展嫌疑——最小核心哲学落到 triage 第一问就是「先剥掉扩展」
- 白名单上半段（L7–149）连续排列、下半段每条空一行分组——文件形态保留了不同时期审批批次的痕迹

## 文档漂移（精读时发现，只记录不修）

| CONTRIBUTING.md 说 | 实际 |
|---|---|
| 「两个 GitHub issue 模板」（L38） | 三个：bug / package-report / contribution（package-report 为 pi.dev 包举报后加） |
| Discord 链接 | 两个不同邀请码：L25（与 issue 模板 config.yml 同款）与 L75，疑似一新一旧 |

元观察：机制演化后文档不会自动跟随——模板从两个长到三个，正文没有同步。

## 容易产生的误解

原以为：auto-close 是维护者的社交政策，靠人手动执行。
实际是：纯机器执行——三层白名单 + 正则识别命令 + 机器人提交；维护者的自由裁量只出现在「每日捞回有价值 issue」和「发 lgtm」两个动作上。
修正来源：2026-09-09 CONTRIBUTING.md 精读 + workflow 源码验证。

## 验证方式

- 三个 workflow 源码：`.github/workflows/` 下 issue-gate.yml、pr-gate.yml、approve-contributor.yml
- 白名单文件：`.github/APPROVED_CONTRIBUTORS`（每行 `<username> <capability>`，capability ∈ {issue, pr}）
- 实际触发路径：任何新贡献者开 issue，Action 日志可见 Issue Gate 的评论与关闭记录

## 遗留问题

无新增。
