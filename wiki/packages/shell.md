---
title: packages/shell — bash / pwsh capability family
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/shell/README.md
  - packages/shell/shell/README.md
  - packages/shell/shell/src/index.ts
  - packages/shell/shell/src/types.ts
  - packages/shell/shell/src/render.ts
  - packages/shell/bash-local/src/index.ts
  - packages/shell/tool-bash/src/index.ts
  - packages/shell/shell-env/src/index.ts
  - docs/subsystems/shell.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

`ShellExecutor`（`ctx.shell`）定义「跑一条前台命令 / 起一个后台进程」这件事**是什么**，不管**怎么做**；本地 bash、沙箱 bash、pwsh 是可互换的 Provider，模型侧的 `bash` / `pwsh` 工具是 Consumer。名字叫 shell，但组里同时住着 PowerShell 全家桶和两个持久化 PTY 工具。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。组 README 也自称「All are **product** packages」。

## 包清单

目录实测 10 个包（`packages/shell/` 下），比组 README 表格多 3 个：

| 包名 | npm | 一句话职责 |
|---|---|---|
| `shell` | `@deepseek-ai/dsh-shell` | Service Definition：`ShellExecutor` 抽象服务 + 词汇类型 + `parseExitStatus` 渲染契约 |
| `bash-local` | `@deepseek-ai/dsh-bash-local` | Provider：经 `ctx.subprocess` 每次起一个 `bash -c <command>` 进程组 |
| `bash-sandbox` | `@deepseek-ai/dsh-bash-sandbox` | Provider：bash-local 的机制 + 每次 spawn 都过 `ctx.sandbox` 约束，拒绝事实盖在结果上 |
| `pwsh-local` | `@deepseek-ai/dsh-pwsh-local` | Provider：`pwsh -NoLogo -NoProfile -NonInteractive -Command <command>` |
| `pwsh-sandbox` | `@deepseek-ai/dsh-pwsh-sandbox` | Provider：pwsh-local 的 pwsh 孪生沙箱版（组 README 表格**未列出**） |
| `shell-env` | `@deepseek-ai/dsh-shell-env` | 支撑服务：`ctx.shellEnv`，可信的每次执行 `DSH_*` 变量注册表 |
| `tool-bash` | `@deepseek-ai/dsh-tool-bash` | Consumer：模型侧 `bash` 工具 + 后台进程接入 `ctx.jobs` |
| `tool-pwsh` | `@deepseek-ai/dsh-tool-pwsh` | Consumer：模型侧 `pwsh` 工具，与 tool-bash 逐调用对齐 |
| `tool-bash-persistent` | `@deepseek-ai/dsh-tool-bash-persistent` | Consumer：`bash(command)` 但走 `ctx.terminals` 的 owner-scoped 常驻 shell（组 README 表格**未列出**） |
| `tool-pwsh-persistent` | `@deepseek-ai/dsh-tool-pwsh-persistent` | Consumer：上者的 pwsh 版（组 README 表格**未列出**） |

## 三件套结构

- **Service Definition**：`dsh-shell`，ctx key `ctx.shell`，`abstract class ShellExecutor extends Service` [T1: packages/shell/shell/src/index.ts#ShellExecutor]。
- **Service Provider**：`bash-local` / `bash-sandbox` / `pwsh-local` / `pwsh-sandbox`。**一个 host 只能挂一个** `ctx.shell` provider，挂两个会因重复服务注册 fail loud。
- **Consumer**：`tool-bash` / `tool-pwsh`（走 `ctx.shell`）与 `tool-bash-persistent` / `tool-pwsh-persistent`（**不走 `ctx.shell`，走 `ctx.terminals`**，见陷阱）。
- **旁支**：`shell-env` 既非 Definition 也非 Provider，是 `ctx.shellEnv` 这个独立注册表，被两个非持久化工具消费。

跨包综合：`bash-sandbox` 的关键设计是**不需要另写工具插件**——`tool-bash` 在注册时读 executor 的 `sandboxMode` capability，有值才把 escalation 字段加进 schema，Consumer 因此永远不 import Provider。

## 扩展点

- 想换执行世界（容器、远端、另一种 shell）：依赖 **`@deepseek-ai/dsh-shell`**，继承 `ShellExecutor` 实现抽象方法，disposal 必须杀掉所有运行中进程并等其退出。声明 `sandboxMode` 就能让 `tool-bash` 自动加出 escalation 字段。
- `SHELL_SETTINGS_NAMESPACE`（值 `bash`）由 **Service Definition 而非 Provider 导出**，因为它命名的是能力而非实现。这样 POSIX 与 win32 各自的 provider 都能用自己的 schema 注册同一个 namespace 而永不冲突，同一份 `settings.yaml` 跨平台都能解析。
- 想给 shell 调用注入受信环境变量：依赖 `ctx.shellEnv`，用 effect-scoped disposal 注册可枚举事实。内置 `DSH_HOME`、`DSH_SHELL=1`、`DSH_SESSION_ID` 归注册表自己所有，重复 ownership 或未声明的运行期键 fail loud。
- 渲染契约共享点：`parseExitStatus` / `ParsedExitStatus` 放在 Definition 里，是 `tool-bash` 的 `renderResult` 与 `tool-pwsh` 的 `renderPwshResult` 追加的 `[exit code: N]` / `[killed by signal: X]` 标记的**逆函数**，两个工具的 `presentResult` 用它切分终端卡片正文与 exit pill——放在这里就是为了两个工具永不漂移。
- 请求 → 规格两段式：`ShellExecRequest`（command, workdir?, timeoutMs?, stdoutMaxBytes?, signal?, stdin?, env?, dshEnv?, sandboxPolicy?）在执行前解析成 `ShellExecSpec`。`stdin`/`env` 给进程内可信插件（hooks 桥、原生插件）用，`dshEnv` 是受类型限制的托管键覆盖层；**模型侧工具一个都不暴露**。

## Known Limitations

- `dsh-shell`：无交互输入词汇（`stdin` 只在 spawn 时写一次就关闭，没有 PTY session 概念）；前台超时永远归 executor 所有，caller-owned deadline 模式被 timeout-policy Agent Note 明确推迟。
- `dsh-bash-local`：自身不做约束（跑在 harness 进程权限下）；无持久 shell / PTY，每次新起非登录 `bash -c`；`bash` 二进制硬编码，**POSIX only**。
- `dsh-bash-sandbox`：约束**只覆盖文件效果**，网络与进程可见性不变，因此不是通用安全沙箱；拒绝是从失败命令的 stderr 推断的，可能误判也可能漏判；`danger-full-access` 故意绕过 `ctx.sandbox`。
- `dsh-tool-bash`：replay 的 exit pill 从结果文本解析，正文最后一行恰好是 `[exit code: N]` 时会显示错误 pill 并吞掉那行（display-only 残留）；`bash` 工具**主动退出 `timeout-policy` 预算**，保留 executor-owned 的 `BASH_TIMEOUT`；后台进程无 executor 超时。
- `dsh-shell-env`：`list()` 只枚举 contributor 声明的变量，**不含**注册表自有的 `DSH_HOME` / `DSH_SHELL` / `DSH_SESSION_ID`，不能当作完整环境目录。
- `dsh-pwsh-local`：编码 preamble 排在命令之前，因此首语句是 `param(...)` / `#requires` / `using` 的命令跑不了（`param` 可用 `& { … }` 包住绕开，`using`/`#requires` 无解，只能落文件执行）；Windows 强杀报 exit 1 且 `signal: null`。
- `dsh-tool-pwsh` / `pwsh-sandbox`：Windows ACL 沙箱 read-only 下 pwsh 进入 ConstrainedLanguage（`Add-Type`、`[math]::` 等失败且**无法从内部解除**）；两种受限模式都拒绝 named-pipe 打开。

## 陷阱

- **两个 tool-*-persistent 不属于 `ctx.shell` 家族**：源码 `inject = ['tools', 'terminals']`，它们消费的是 terminal 组的 `ctx.terminals`，只是恰好放在 shell 目录里、注册同名 `bash` / `pwsh` 工具。选型时不要以为换 `ctx.shell` provider 能影响它们。
- 持久化工具的状态会被丢弃：显式 `exit`、超时、**以及取消**都会重置 shell 并丢结果，即便完整状态标记已经可见，下一次调用是全新 shell。
- 后台 spawn 失败提示是**单次投递**：subprocess 服务不为「从未跑起来的进程」缓冲输出，executor 把 `spawn failed: …` 塞进恰好一个 `readOutput()` delta，读者丢了就再也拿不回来（bash-local 与 pwsh-local 都是）。
- `ShellProcess.readOutput()` 是**增量**读，连续读永不重投；丢数据时会标 `lossy` 并指向全流 spill 文件。
- per-session 的 sandbox mode 覆盖词汇（`'sandbox/mode'` 事件、`effectiveSandboxMode(events)`、`setSandboxMode`）**不在本组**，在 `@deepseek-ai/dsh-sandbox-policy`。
- `run()` 只在基础设施失败（workdir 不可用、shell 缺失、signal 已 abort）时 reject；非零退出、超时杀、abort 杀都是**正常 resolve** 的 `ShellRunResult`。

## 去哪深入

- 组装配方（哪些包一起装、沙箱组合）→ `packages/shell/README.md` 与 `examples/acp-agent/`
- Service API / 词汇 / `dshEnv` 语义 → `packages/shell/shell/README.md`、`packages/shell/shell/src/types.ts`
- 子系统参考（request/spec、结果、后台进程、事件）→ `docs/subsystems/shell.md`
- 沙箱执行器与策略的分工 → `packages/shell/bash-sandbox/README.md`、`packages/sandbox/sandbox-policy/`
- 超时归属决策 → `.agents/notes/implemented/architecture/2026-07-07-tool-call-timeout-policy.md`
- capability seam 三角形的原始决策 → `.agents/notes/implemented/architecture/2026-06-13-capability-seams.md`
