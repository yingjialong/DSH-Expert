---
title: packages/terminal — 持久 PTY 能力族
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/terminal/README.md
  - packages/terminal/terminal/README.md
  - packages/terminal/terminal/src/index.ts
  - packages/terminal/terminal-bash/README.md
  - packages/terminal/terminal-bash/src/index.ts
  - packages/terminal/terminal-bash/src/config.ts
  - packages/terminal/tool-terminal/README.md
  - docs/subsystems/terminal.md
  - .agents/notes/implemented/feature/2026-07-16-persistent-pty-sessions.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

跨工具调用保持状态、可交互 stdin 的持久伪终端（PTY）能力族；与一次性的 shell / fs 工具互补而非替代，后者的单次操作契约更强。

## 稳定性

`Product — stable API`（`packages/README.md` 表格中 `terminal/` 行原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `terminal/terminal/` | `@deepseek-ai/dsh-terminal` | Service Definition：backend 注册表、branded session id、精确 Agent 归属围栏、会话操作、等待式清理；`ctx.terminals` |
| `terminal/terminal-bash/` | `@deepseek-ai/dsh-terminal-bash` | Provider：基于 `ctx.subprocess.spawnTerminal` 的 shell backend（就绪检测、有界终端状态、sandbox 策略） |
| `terminal/tool-terminal/` | `@deepseek-ai/dsh-tool-terminal` | Consumer：六个 model-facing 工具 + 后台发送的 `ctx.jobs` 集成 |

## 三件套结构

- **Service Definition**：`@deepseek-ai/dsh-terminal`，导出 `TerminalSessionService`（默认导出）挂在 `ctx.terminals`，另有 `TerminalSessionId`、`TerminalError` / `TerminalErrorCode`、`TerminalBackendCleanupError` [T1: packages/terminal/terminal/src/index.ts]。seam 内**不含** `node-pty`、sandbox、tool schema、prompt、task、终端渲染策略。
- **Service Provider**：`@deepseek-ai/dsh-terminal-bash`，通过 `ctx.terminals.registerBackend(new BashTerminalBackend(...))` 注册，backend `type` 由 config `backendType` 决定，默认 `'shell'` [T1: packages/terminal/terminal-bash/src/config.ts:85]。
- **Consumer**：`@deepseek-ai/dsh-tool-terminal`，注册 `terminal_open` / `terminal_send` / `terminal_read` / `terminal_signal` / `terminal_close` / `terminal_list` 六个工具，并贡献一段固定的 Terminal guidance 系统提示。

## 扩展点

- 想换一套终端实现（远端执行世界、容器 PTY、非 shell 后端）：依赖 **`@deepseek-ai/dsh-terminal`**，实现 backend 契约并 `registerBackend`。契约要点：注册一个稳定 `type`；返回**未发布的** `TerminalBackendSession`；setup 失败或被取消必须清理部分资源；清理失败要以 `TerminalBackendCleanupError` reject，registry 才能跨取消保留它。
- 想换一套 model-facing 表达（不同工具名 / 不同渲染）：写自己的 Consumer 依赖 `ctx.terminals`，不要依赖 `terminal-bash`。
- `terminal-bash` 自身的可调点是 `shellDialect`（`bash` 默认 / `pwsh`）、`backendType`、`maxReadBytes`、`handoffGraceMs` 等 Config 字段。
- `tool-terminal` Config 只有两项：`enableRunInBackground`（默认 `true`）、`maxResultBytes`（默认 `262144`，最小 `64`）。

## Known Limitations

（三个包各自 README 的 `## Known Limitations and Deferred Work`，合并要点）

- 会话是进程本地的，**harness 重启后不恢复**；跨 agent 共享被刻意省略，将来要做需要独立的 authority 契约。
- 输出按行归一化，**不支持 alternate-buffer 全屏交互**（TUI）。
- 精确的 stdin-wait 检测取决于挂载的 subprocess provider；证明不了的 provider 退化到 prompt-marker + 静默/超时判定。**Windows 就是这类 provider**：shell pid 充当伪前台组，没有精确 stdin-wait 层级。
- pwsh 引导通过 `[Console]::` 写入（UTF-8 编码锁定 + prompt 函数），Windows ACL sandbox 只读模式（ConstrainedLanguage）可能拒绝；此时 marker readiness 不可用，非 ASCII 输出可能走宿主代码页。
- 清理保证等同于 `SubprocessTerminalHandle`，provider 特有缺口属于那一层的契约。
- schema 未暴露命名按键序列、TUI、BEL、resize、自动启动、跨 agent 共享；后台模式**同时**需要 `@deepseek-ai/dsh-jobs` 及其 model-facing 控制器。

## 陷阱

- **`inferred_idle` / `timeout` 结果不证明前台命令已退出**——这是写进系统提示的原话，也是最容易被上层误读的一点。
- `TerminalSendResult.waitReason` 与 `sessionStatus` 是**独立**的；`session_exit` 描述顶层 PTY 进程，不是任意前台命令。
- 一个 session 同时只允许**一个 live send**；读和信号可以并发观察，另一个 send 会失败直到当前操作 settle。
- 取消 send 时**不会**写 `\x03` 模拟中断（保证 raw-mode 程序仍可取消），而是向前台进程组发真实 `SIGINT`；若 provider 的 write 或 signal 永不 settle，slot 会被无限期占用，**唯一恢复手段是 `terminal_close`**。
- sandbox 模式切换有围栏：该 owner 有开着的 PTY 或进行中的 spawn 时，切换到不同 effective mode 会在 `sandbox/mode` 事件提交前被拒绝。改模式前先关会话。
- `danger-full-access` 直接起 shell，不要求挂 sandbox provider；受限模式必须有同 world 的 `ctx.sandbox`，否则 spawn 前就失败。

## 去哪深入（文件路由）

| 想查 | 去 |
|---|---|
| id、backend/session 契约、send 就绪、有界读的子系统参考 | `docs/subsystems/terminal.md` |
| seam 契约条款（owner 围栏、rollback、quiescence） | `packages/terminal/terminal/README.md` §Contract |
| 服务源码与 ctx 声明 | `packages/terminal/terminal/src/index.ts#TerminalSessionService` |
| bash/pwsh 就绪检测、prompt marker、取消语义的完整叙述 | `packages/terminal/terminal-bash/README.md` §Plugin |
| 六个工具的 schema | `docs/tool-catalog.md` 锚点 `#deepseek-aidsh-tool-terminal` |
| 设计与被推迟的边界 | `.agents/notes/implemented/feature/2026-07-16-persistent-pty-sessions.md` |
