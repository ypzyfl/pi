# test.sh 在 Windows 上：Git Bash 执行与三层坑

状态: 已验证（2026-09-10：两次完整实跑 + env -i 探针实证；沙箱污染部分待本机自由终端复核后定论）

## 事实源（链接，不复述）

- [test.sh](../../../test.sh) L39-L78（env -i 白名单）、L66-L70（Native Windows 变量继承）
- [packages/tui/src/terminal-image.ts](../../../packages/tui/src/terminal-image.ts) L669-L676（shortenImagePath 的 startsWith 比较）
- package-lock.json L1679-L1682（rolldown 绑定的 integrity 基准）

## 怎么跑（≤5 句）

Windows 没有 test.ps1 等价物，唯一入口是 Git Bash。PowerShell 里必须全路径调用 `& "C:\Program Files\Git\bin\bash.exe" ./test.sh`——PATH 上的裸 `bash` 解析到 WSL（`C:\windows\system32\bash.exe`），会进错误环境。test.sh 专门为 Native Windows 继承 `SystemRoot` / `SYSTEMROOT` / `WINDIR` / `COMSPEC` / `PATHEXT`（L66-L70 注释原话 "Native Windows needs these inherited values to launch child processes"），`env -i` / `mktemp` / `type -P` 在 MSYS2 里都有——这是被设计支持的路径，不是将就。

## 坑一：MSYS 环境变量转换不对称（机制层）

test.sh 用 `env -i` 把 HOME 和 USERPROFILE 指向同一隔离目录。MSYS 向原生 Windows 进程传环境变量时做路径转换，但两个变量得到**不同形式**（env -i 探针实证，2026-09-10）：

| 变量 | node 实际收到 |
|---|---|
| HOME | `C:\Users\...\Temp\probe-home`（反斜杠） |
| USERPROFILE | `C:/Users/.../Temp/probe-home`（**正斜杠**） |

后果链：`os.homedir()` 读 USERPROFILE 得正斜杠形式 → `path.win32.join(homedir(), ...)` 归一成反斜杠 → [shortenImagePath](../../../packages/tui/src/terminal-image.ts) 的 `filename.startsWith(home + "/" 或 "\\")` 因混合分隔符永不匹配 → `~/` 缩写失效。同族失败：agent 的 jsonl 测试（`/workspace` 被解析成 `D:\workspace`）、coding-agent 的 footer / context path 测试。

## 坑二：symlink EPERM（权限层）

Windows 未开开发者模式（且非管理员）时 `fs.symlink()` 被系统拒绝（EPERM）。约 12 个测试（nodejs-env / prompt-templates / skills / tools-edit / file-mutation-queue 等）在 setup 阶段创建 symlink 即失败。Linux/macOS 无此限制；这不是代码缺陷，是系统策略。

## 坑三：rolldown 绑定磁盘损坏（诊断案例，方法论可复用）

首跑 10 个 vitest 包全部秒挂（`ERR_DLOPEN_FAILED`），根因是 `node_modules/@rolldown/binding-win32-x64-msvc/rolldown-binding.win32-x64-msvc.node` 在磁盘上被截断。诊断路径：

1. `ERR_DLOPEN_FAILED: ... is not a valid Win32 application` → 文件存在但 dlopen 拒绝
2. 读文件头：MZ 魔数 + PE Machine=0x8664（AMD64）均合法 → 不是架构错配
3. 下载 registry tarball，sha512 与 package-lock.json 的 integrity 精确匹配 → 原版可信
4. 对比哈希：本地 20,764,625 字节 ≠ 原版 23,471,616 字节（截断约 2.7MB）→ 磁盘级损坏（npm 安装时有 integrity 校验，损坏发生在装完之后）
5. 用验证过的原版覆盖 → rolldown 恢复，全部 vitest 包正常启动

## Windows 套件现状（2026-09-10 修复 rolldown 后实跑）

| 包 | 结果 |
|---|---|
| agent | 613 过 / 52 败 |
| ai | 1011 过 / 847 skip（e2e 无 key 自动跳过——test.sh 隔离机制的直接证据） |
| coding-agent | 2067 过 / 83 败 / 52 skip |
| client / server / chord / evals / tui | 8 / 7 / 1 / 1 / 5 败 |
| sqlite-node / protocol / telemetry | 全过 |

失败四分类：① 测试硬编码 POSIX 路径（约 45 个，含 tui imageFallback——**真跨平台 bug**，即使原生 Windows 无 Git Bash 也必败）；② symlink EPERM（约 12 个）；③ POSIX 独有机制（SIGCONT 挂起恢复等）；④ 执行沙箱污染（unix.test.ts 的 `D:\tmp` EPERM、experimental-remote-runtime 系列——经 agent 工具沙箱跑出的失败，本机自由终端复核后才可定论）。

## 容易产生的误解

原以为：test.sh 不支持 Windows；测试大面积失败说明产品在 Windows 上是坏的。
实际是：test.sh 显式支持 Native Windows；失败集中在测试代码的 POSIX 假设（测试面向 POSIX 环境编写），产品代码的 Windows 支持（powershell 工具、`pi-test.ps1` 等）是另一回事，两者不可互相推断。
修正来源：2026-09-10 两次实跑 + 逐类失败核对。

## 验证方式

- 复跑：PowerShell `& "C:\Program Files\Git\bin\bash.exe" ./test.sh`（仓库根）
- MSYS 转换探针：`env -i` 下设 HOME/USERPROFILE 同值，node 打印 `process.env.HOME` / `USERPROFILE` / `os.homedir()` 对比
- rolldown 修复验证：仓库根 `node -e "require('rolldown')"`

## 遗留问题

- ④ 类（沙箱污染）待在自有 Git Bash 终端复跑定论
- tui imageFallback 分隔符 bug：fork 修复候选——`shortenImagePath` 需归一分隔符，测试期望需跨平台化
