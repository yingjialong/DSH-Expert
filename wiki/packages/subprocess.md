---
title: packages/subprocess — subprocess capability family
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/subprocess/README.md
  - packages/subprocess/subprocess/README.md
  - packages/subprocess/subprocess/src/index.ts
  - packages/subprocess/subprocess/src/types.ts
  - packages/subprocess/subprocess-local/README.md
  - packages/subprocess/subprocess-local/src/spawn.ts
  - packages/subprocess/subprocess-local/src/terminal.ts
  - docs/subsystems/subprocess.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

**一个执行世界的共享进程基座**：可执行文件查找、完全显式指定的托管子进程树（raw 或 collected stdio），以及唯一一个「深度」终端进程原语（拥有 PTY 分配、前台进程组、provider 可观测的会话清理）。命令默认值、shell 语义、deadline、协议分帧、readiness、呈现**全部不在这里**，归各消费方。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

只有两个包，是这批里最小的一组：

| 包名 | npm | 一句话职责 |
|---|---|---|
| `subprocess` | `@deepseek-ai/dsh-subprocess` | Service Definition：可执行查找、普通托管 spawn、terminal 原语、句柄生命周期、共享环境/输出词汇 |
| `subprocess-local` | `@deepseek-ai/dsh-subprocess-local` | 本地 Provider：detached 进程树、有界收集/spill、`node-pty`、前台/会话检查、树级信号、terminate-and-join disposal |

## 三件套结构

- **Service Definition**：`dsh-subprocess`，ctx key `ctx.subprocess`，`abstract class SubprocessRuntime extends Service` [T1: packages/subprocess/subprocess/src/index.ts#SubprocessRuntime]。
- **Service Provider**：`dsh-subprocess-local`，`class LocalSubprocessRuntime extends SubprocessRuntime` [T1: packages/subprocess/subprocess-local/src/index.ts#LocalSubprocessRuntime]。它**没有配置**——每个 disposition、limit、终端尺寸、grace、目录都来自调用它的能力 seam。
- **Consumer**：**不在本组**。消费方分散在四个别的组：`shell/bash-local`（bash 执行器）、`lsp/lsp-stdio`（LSP host）、`terminal/terminal-bash`（PTY shell 后端）、`subagent/subagent-acp`（ACP 子代后端）。

这条分工线是本组的核心设计：**服务拥有跨消费方重载的进程生命期；消费方拥有「一个进程意味着什么」以及塑造它的每一个默认值**。

## 扩展点

想接一个新的执行世界（容器、远端机器、VM）：依赖 **`@deepseek-ai/dsh-subprocess`**，继承 `SubprocessRuntime`。契约要点：

- `spawn(spec)` **立即**返回活句柄；`done` 在进程 close 时带退出事实 resolve（`SubprocessOutcome` **不带输出、不带原因分类**），**只在 spawn 级失败时 reject**。
- spec 必须**完全显式**——argv、cwd、每流 stdio disposition、grace，因为随部署变化的默认值属于调用方配置，不属于隐藏的 subprocess 默认值（`dsh-shell` 的 request/spec 拆分是范本）。`argv` **永不做 shell 解释**，想要 shell 的消费方自己传 `['bash', '-c', command]`。
- `resolveExecutable(command, env?, signal?)`：绝对路径做校验，裸名在该世界的**已 scrub 的 PATH** 加显式覆盖里解析。spawn 的工作目录与可执行路径都属于 provider 的执行世界。
- stdio 按 Node 形状逐流配置：`'pipe'` 把裸流交给调用方做自己的协议分帧（LSP JSON-RPC、ACP ndjson）；`'inherit'` 透传父描述符；collect 模式 `{ maxBytes, spill? }` 缓冲有界尾部并可选全流 spill 文件。
- **collect 读者取的是全流字节偏移且永不消费**，因此多个独立读者不会互相偷 delta；偏移滑出内存尾部的读标 `lossy` 并在有 spill 时指向该文件。结算后收集的输出仍可读。
- **终止一律树级**（POSIX detached 进程组 + 直接子进程回退；Windows `taskkill /T`）。`terminate()` 是**唯一**的终止动词：SIGTERM→grace→SIGKILL 逐级升级，幂等，也由 spec 的 abort signal 驱动，树已消失时是 no-op。`waitForExit(signal?)` 观测整树存活，让消费方自己的 teardown 阶梯每一级都停在真实静默上——**管理器只反应，从不分类原因**。
- `spawnTerminal(spec)` 是**唯一**的非 pipe 原语：句柄拥有真实 PTY、UTF-8 文本 I/O、前台进程组检查/信号，以及一个 awaited 的 `terminate()`（覆盖 provider 仍能观测到的每个会话成员，并结算 in-flight 句柄调用）。spec signal **只取消分配**，已发布的句柄自己拥有生命期。
- `scrubbedParentEnv()` / `SENSITIVE_ENV_PATTERN` 是**唯一一份** scrub 定义：环境里凭证形状的名字与 `DSH_*` 名字被丢弃，显式 `env` 在 scrub **之后**合并。自带 spawn 的 SDK transport 可以直接 import 它，让环境策略保持单一来源。
- 服务 disposal 会终止所有仍在运行的托管进程并等待其退出。

## Known Limitations

- `dsh-subprocess`：**SDK 托管的 spawn 在外面**——自己拥有内部 spawn 的 SDK transport 无法把那次调用路由进本服务（但仍可 import `scrubbedParentEnv`）；**teardown 阶梯归消费方所有**，seam 只出信号动词与树存活等待，不出成套的 quiesce 序列（ACP 后端的 stdin-EOF-first 阶梯是仓库内范本）。
- `dsh-subprocess-local`：Windows 树支持是 **best-effort**（走 `taskkill /PID <pid> /T /F`，存活判断退回直接子进程边界）；**Windows 终端信号是 console 级**（SIGINT 是写入 `\x03`，被 conhost 变成 console-wide CTRL_C 事件；SIGTSTP 与 SIGHUP 直接拒绝；不带 `/F` 的 `taskkill` 杀不掉 console 进程，所以 TERM 层只是 `/F` 升级前的 grace 等待）；**守护化的终端后代仍可能逃出可观测边界**（macOS 上在任何前台快照之前 reparent 的子进程、Linux 上调用 `setsid` 的子进程），本地 provider **不加**持续的进程表监控；**进程内清理需要 JavaScript 可观测的退出**；**凭证 scrub 是名字启发式**，只认 `*KEY*`/`*PASSWORD*`/`*SECRET*`/`*TOKEN*`，`*PASSPHRASE*` 之类漏网；**已完成的 spill 文件不删**，在 OS tmpdir 下累积直到外部清理。

## 陷阱

- 别指望 `done` 告诉你「为什么」结束：`SubprocessOutcome` 只有退出事实，**没有输出也没有原因分类**。deadline、teardown 阶梯、原因分类全是调用方的活。
- `spawn` 只在 **spawn 级失败**时 reject；非零退出是正常 resolve。
- 「从未跑起来」的进程**不缓冲输出**——这正是 `bash-local` / `pwsh-local` 要把 `spawn failed: …` 塞进恰好一个 `readOutput()` delta 的原因，丢了就没了（见 `wiki/packages/shell.md`）。
- 进程内清理的覆盖面有明确边界：`process.exit()`、默认未捕获异常、默认未处理 rejection 会触发 Node 同步 `exit` 事件；但**未处理的 `SIGTERM`/`SIGINT`/`SIGHUP` 的 OS 默认处置会绕过该事件**——应用必须自己装 handler 做正常 disposal 或调 `process.exit()`。`SIGKILL`、致命 OOM、`process.abort()`、原生崩溃、断电则**必须**靠外部 supervisor / 容器 init。
- 想在受限环境里控制输出体积，参数不在这个包的配置里——`subprocess-local` **没有配置**，`maxBytes`/`spill`/grace/终端尺寸都由调用的 seam 传。
- 环境 scrub 会**丢掉所有 `DSH_*`**，托管的 `DSH_*` 事实要靠 `dsh-shell` 的 `dshEnv` 覆盖层在 scrub 之后重新合并。

## 去哪深入

- 组定位与「服务拥有什么 / 消费方拥有什么」的分界 → `packages/subprocess/README.md`
- 完整契约（spawn spec、stdio 模式、terminate/waitForExit、`spawnTerminal`、`scrubbedParentEnv`）→ `packages/subprocess/subprocess/README.md`、`packages/subprocess/subprocess/src/types.ts`
- 裸进程处理实现 → `packages/subprocess/subprocess-local/src/spawn.ts`；服务接线在同包 `src/index.ts`（README 原文点名）
- PTY / 终端句柄实现 → `packages/subprocess/subprocess-local/src/terminal.ts`
- 子系统参考（spawn specs、output readers、outcomes、`DSH_*` 环境）→ `docs/subsystems/subprocess.md`
- seam 决策 → `.agents/notes/implemented/architecture/2026-07-26-subprocess-seam.md`
