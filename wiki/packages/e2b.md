---
title: packages/e2b — E2B 远程运行时族（POC）
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/e2b/README.md
  - packages/e2b/e2b/README.md
  - packages/e2b/fs-e2b/README.md
  - packages/e2b/subprocess-e2b/README.md
  - packages/e2b/e2b/src/index.ts
  - packages/e2b/e2b/package.json
  - examples/headless-agent/e2b.cordis.yml
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

一个 provider 组合的 POC：把**一个**文件系统 / 进程执行世界搬进 E2B 的 Linux sandbox。E2B 只提供 sandbox 生命周期 + 两个最基础的 OS adapter，更高层能力由 provider-neutral 的消费者在其上搭建。

## 稳定性

`POC`（`packages/README.md` 表格原文）。全组唯一被标 POC 的组——不是 `Product — stable API`，也不是 `Unreleased`。

## 包清单

| 包目录 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `e2b/` | `@deepseek-ai/dsh-e2b` | `ctx.e2b` | 单个 sandbox 的共享生命周期所有者：创建、准备工作/运行时目录、暴露 SDK 句柄、超时或销毁时删除 |
| `fs-e2b/` | `@deepseek-ai/dsh-fs-e2b` | `ctx.fs` | 用 E2B Filesystem API 实现文件系统 seam |
| `subprocess-e2b/` | `@deepseek-ai/dsh-subprocess-e2b` | `ctx.subprocess` | 用 E2B Commands / PTY API 实现可执行查找、托管进程组与 stdio、远程 spill 文件、terminal session |

## 三件套结构

本组**只出 Provider**，不出 Service Definition，也不出 Consumer：

- **Service Definition**（在别的组）：`@deepseek-ai/dsh-fs`（`fs/` 组）、`@deepseek-ai/dsh-subprocess`（`subprocess/` 组）。
- **Service Provider**（本组）：`fs-e2b`、`subprocess-e2b`，分别顶替 `dsh-fs-local` / `dsh-subprocess-local`。
- **Consumer**（在别的组、且**无需为 E2B 分叉**）：`dsh-bash-local`、`dsh-terminal-bash`、`dsh-lsp-stdio`。它们把每一个执行世界操作都委托给 `ctx.fs` 和 `ctx.subprocess`，所以挂上这两个 adapter 就把它们的可变工作整体搬进同一个 sandbox。
- `ctx.e2b` 本身不是能力 seam，是**共享资源所有者**，另外两个包 inject 它、await 它唯一的 SDK 句柄，从而保证同处一个远程工作树与进程世界。

## 扩展点

- **想换远程运行时（非 E2B）**：这组本身就是范例——依赖 `dsh-fs` 与 `dsh-subprocess` 两个 Service Definition，实现 `ctx.fs` / `ctx.subprocess`，再加一个共享生命周期所有者服务即可，不需要碰任何 bash/PTY/LSP Consumer。这套「可移植执行世界」的通用组合规则由 `.agents/notes/implemented/architecture/2026-07-28-portable-execution-world-consumers.md` 拥有。
- **加载顺序是硬约束**：provider 插件必须在 `@deepseek-ai/dsh-e2b` **之后**加载、在它**之前**销毁。
- `fs-e2b` **没有任何 config**；`subprocess-e2b` 只有一个 `pollMs`（默认 20ms，每 tick 一次控制面请求）。

## Known Limitations

- **这不是整机运行时**：Cordis 服务、agent/session 状态、session 日志、LLM 请求、skill、更高层协议状态、E2B SDK 缓冲区全部留在宿主进程。
- **sandbox 状态是易失的**：销毁与超时都会删除 sandbox；reconnect、pause/leave 保留、template、volume、snapshot 都在 POC 之外。
- 没有配置任何部署平台：网络策略、宿主 workspace 同步、sandbox 发现都在 POC 之外。
- `fs-e2b`：不做宿主同步（空的远程 cwd 就一直是空的）；变更协调只在宿主进程内；读操作按路径重新打开规范目标（无稳定文件句柄围栏）；整文件变更成本仍在（overwrite diff 与字面 edit 会把整文件读进宿主内存）；只针对 E2B 默认 Linux 镜像（依赖 GNU `realpath`/`base64`/`chmod`、同文件系统 rename、流式读、metadata 扩展属性）。
- `subprocess-e2b`：SDK 仍在宿主内存里累积完整命令输出；同步 PID 消费者不受支持；私有状态活到 sandbox 结束；控制状态与 sandbox 用户同 UID；数字进程标识没有 reuse 围栏；E2B 不暴露信号事实；无法精确探测 terminal 是否在等 stdin；假定 Linux 工具与 E2B 传输语义（无 Windows / 逃逸会话恢复 / 网络分区保真）。

## 陷阱

- **`pid` 在远程启动期间是 `-1`**。同步 seam 立即返回句柄而命令在远端后台启动，stdin 与普通观察都要等 wrapper 发布并校验其进程组 id。**需要立刻拿到正 PID 的消费者（含 ACP child backend）无法直接用这个 provider**。
- **`cwd` 是解析约定，不是 containment**：adapter 和命令都能寻址 sandbox 里的其他路径；E2B 网络访问保留基础镜像的策略。
- **给它一个宿主路径当 `cwd` 只会在远端创建一个同名目录**，不会上传也不会回写本地文件。
- **`.dsh-e2b` 的 `0700`/`0600` 隔离不了同 sandbox 内的并发进程**——E2B 把每条命令都以同一个默认用户跑。后台进程理论上能改写 `pid`/`exit-code` 或读走尚未消费的 `environment` 文件；adapter 只做了值校验并拒绝负形式不安全（`<= 1`）的组 id。
- **不要把密钥放进 sandbox 默认环境变量**：初始环境探测会继承 sandbox 默认值，无法在枚举前把未知的凭据形状名清空——README 明确「this POC therefore does not support secrets in sandbox-default environment variables」。wrapper 会剔除 ambient 的 `DSH_*` 与 `*KEY*`/`*SECRET*`/`*TOKEN*` 名字，并只按 `spec.env` 显式恢复。
- **`e2b` npm 依赖被钉死在 `2.29.1`** [T1: packages/e2b/e2b/package.json]，README 与 package.json 一致。
- **`apiKey` 配置或 `E2B_API_KEY` 只配置宿主 SDK 连接，绝不安装进 sandbox**；`cwd` 默认 `/home/user/workspace`（必须是绝对 POSIX 路径），`timeoutMs` 默认 5 分钟且到期即删 sandbox。
- **退出码可能是伪装的信号**：E2B 不暴露信号事实，任何非 adapter 请求的 SDK 退出都以退出码形式呈现，**包括等于 `128 + signal` 的值**。
- `SandboxNotFoundError` 在探活/终止/回滚/断连时被当作 quiescence 接受（证明远端执行世界无法保留工作），不是错误。

## 去哪深入（文件路由）

| 想知道什么 | 去哪 |
|---|---|
| 组的边界声明（什么被搬走、什么没被搬走）、为何 bash/terminal/lsp 无需分叉 | `packages/e2b/README.md` |
| sandbox 生命周期、目录准备、`.dsh-e2b` 保留路径校验、销毁顺序、配置三项 | `packages/e2b/e2b/README.md` |
| 远程身份与 metadata、原子写（staging + rename / `ln -T`）、有界字节读、失败映射 | `packages/e2b/fs-e2b/README.md` |
| `exec setsid --wait` 进程组、环境边界与清洗、stdio 三模投影、PTY 会话 | `packages/e2b/subprocess-e2b/README.md` |
| 被顶替的两个 Service Definition | `packages/fs/fs/README.md`、`packages/subprocess/subprocess/README.md` |
| 通用「可移植执行世界」决策 | `.agents/notes/implemented/architecture/2026-07-28-portable-execution-world-consumers.md` |
| 可跑的组合 | `examples/headless-agent/e2b.cordis.yml` |
