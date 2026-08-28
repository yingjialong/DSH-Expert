---
title: packages/sdk — 从另一个进程驱动 Harness runtime
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/sdk/README.md
  - packages/sdk/protocol/README.md
  - packages/sdk/protocol/src/transport.ts
  - packages/sdk/protocol/src/types.ts
  - packages/sdk/client/README.md
  - packages/sdk/client/src/api.ts
  - packages/sdk/server/README.md
  - packages/sdk/server/src/server.ts
  - packages/examples/jsonrpc-demo/README.md
  - python/README.md
  - .agents/notes/implemented/feature/2026-07-27-typescript-sdk-and-sdk-subagent-backend.md
  - .agents/notes/implemented/simplification/2026-08-11-remove-sdk-project-toolchain.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

一条 stdio 上的 newline-delimited JSON-RPC 2.0 协议栈，让外部进程把 Harness runtime 当子进程驱动。调用方自己提供 runtime 可执行文件和它的 `cordis.yml`——**这个组不创建、不配置、不构建、不启动开发者项目**。

## 稳定性

`Product — stable API`（[T1: packages/README.md]：`sdk/` = "Out-of-process runtime SDK: JSON-RPC protocol, TypeScript client, and server plugin"）。但注意协议本身是 pre-release：`serverInfo.version` 为 `0.0.1` 且客户端不校验，没有版本协商，也没有兼容承诺。

## 包清单

| 目录 | npm 名 | 职责 |
|---|---|---|
| `protocol` | `@deepseek-ai/dsh-sdk-protocol` | 纯库：`JsonRpcLineTransport` 传输类 + 两端共用的请求/结果/通知类型。无插件、无 Config、无注册 |
| `client` | `@deepseek-ai/dsh-sdk-client` | 纯库：`DeepSeekHarness`（高层 owned-run API）+ `HarnessClient`（低层协议客户端）。不向 Cordis context 注册任何东西 |
| `server` | `@deepseek-ai/dsh-sdk-jsonrpc-server` | `jsonrpc` 插件：在 stdio 上服务外部 SDK 客户端；`inject: ['agents']` |

**目录名与 npm 名不一致**：`server/` 的包名是 `dsh-sdk-jsonrpc-server`（不是 `dsh-sdk-server`）。

## 三件套结构

这是「协议栈」形态而非 capability seam：

- **Wire 契约（相当于 Service Definition）**：`dsh-sdk-protocol`。两端唯一共享的依赖。
- **Server 侧实现**：`dsh-sdk-jsonrpc-server`（`HarnessSdkJsonRpcServer`，[T1: packages/sdk/server/src/server.ts]）。
- **Client 侧实现**：`dsh-sdk-client`（TypeScript）与 `python/` 下的 Python SDK（包名 `deepseek-harness`）——Python 侧**镜像**这些形状但**不 import** 它们。
- **应用外壳**：`packages/examples/jsonrpc-demo`（提供围绕 server 插件的 `cordis.yml`）。
- **反向消费者**：`@deepseek-ai/dsh-subagent-dsh-sdk`（subagent 后端）就是 `dsh-sdk-client` 的仓内消费者。

## 扩展点

- **要写一个新语言的客户端**：只依赖 wire 契约（`packages/sdk/protocol/src/types.ts`），不要 import server。方法集是封闭的：client→server `initialize` / `session/prompt` / `shutdown`；server→client `session.event` / `session.status` / `subagent.started` / `subagent.finished`。`HarnessSdkRequestMap` 与 `HarnessSdkNotificationMap` 按方法名索引。
- **要在 TS 里驱动 runtime**：用 `DeepSeekHarness`（`launch: { command, args }` 显式指定可执行文件，`await using` 或 `close()` 必须调用），或用 `HarnessClient` 自己管 `start()/initialize()/prompt()/request()/close()` 与订阅。
- **要改 runtime 的能力**：改被服务的那个 `cordis.yml`，不是改本组。server 插件只提供 `maxTokensAsSuccess` 一个产品配置（外加 `input`/`output`/`exit` 三个仅测试用的传输钩子）。
- **要扩展模型侧体验**：本组三个包的 Model Experience 都是 None / 间接——system prompt、tool schema 全部来自周围 `cordis.yml` 组合的插件。
- **过滤与作用域是客户端的事**：runtime 对**上下文里每一个 session** 都发通知；`subscribe(filter?)` / `subscribeSessionTree(id)` 是客户端侧裁剪，TypeScript 与 Python SDK 行为一致。

## Known Limitations

（三个包的 `## Known Limitations and Deferred Work` 合并）

- **无协议版本协商**：握手只带 `serverInfo.version`，客户端不校验。
- **无中途取消、无 per-session close、无 prompt-cancel 方法**：放弃一个 turn 的唯一办法是关掉 runtime 进程；SDK 创建的 agent 一直存活到进程 shutdown。
- **没有 per-prompt 结果**：`session/prompt` 返回的 `MessageId` 只标识 inbox 准入，不标识后续的 assistant 消息、turn 结束或 prompt 结果。需要"自动化区间"语义的客户端必须自己定义并观测它。
- **server→client 请求是死能力**：传输层支持，但 server 从不发；Python SDK 的 responder 面向未来的审批流。client→server 通知同样两端都未实现。
- **stdout 纯净性靠部署保证**：周围 config 仍可能加载一个 stdout logger 从而污染 JSON-RPC 通道，本插件不检查也不否决同级 logger。
- **自动挂 adapter 是 DeepSeek 专属**：`initialize` 能复用任何已注册的 model adapter，但唯一的 fallback 是挂 `dsh-llm-deepseek`；其他无主 provider 直接让初始化失败。
- **无内置 runtime 解析**：TS 侧必须显式命名可执行文件；打包可执行文件的发现仍归 Python 发行版。

## 陷阱

- **stdout 就是协议**：只允许 JSON-RPC 帧，诊断一律走 stderr。
- **`finalResponse` 不是"这条 prompt 的回答"**：`run()` 拥有一个活动区间——排队 prompt → 等自己的 `MessageId` 出现在持久的 `agent/inbox/spliced` 回执 → 收集到下一次整体 `idle`。区间内 steering、注入上下文、其他排队工作都可能贡献内容。
- **`initialize` 是 runtime readiness 边界**：被 Loader 组合挂载时它会等当前插件树 settle 再回复，这样异步的兄弟能力（比如初次 MCP 工具发现）对第一条 prompt 可见。手搭的、不走 Loader 的 context 则立即可用。
- **`session.event` 是全量、不过滤的**：协议直接流式传送完整的 session-log envelope，所以 `SessionEvent`（`dsh-session`）、`ContentBlock`（`dsh-llm`）、`SubagentStopReason`（`dsh-subagent`）的词汇**属于 wire 契约的一部分**——改这些类型就是改协议。
- **`subagent.finished` 只在 in-process 运行时发**：server 只在 service 快照出的 lifecycle `local` 标志为 true 时转发；provider 名、child id、持久 lineage 都不能确立 locality。
- **关闭有梯子**：`close()` 先发协议 `shutdown`（`shutdownTimeoutMs` 默认 1000 ms），再走 stdin-EOF → SIGTERM → SIGKILL（`disposeEofGraceMs` 默认 6000、`disposeGraceMs` 默认 3000）。这个梯子**故意不走 `dsh-subprocess` 服务**——它跑在任何 harness context 之外，是那个 seam 记录在案的例外。
- **`HarnessClientOptions.env` 给了就整体替换子进程环境**（`undefined` 才继承父进程）；凭据策略归调用方，`dsh-subprocess` 的 `scrubbedParentEnv` 是共享的 scrub 基线。
- **`serverInfo.name` 是 wire-stable 的 `deepseek-harness-sdk-runtime`**，别改。
- **两个 SDK 都要同步更新**：agent-loop、session-lifecycle、`SessionEventMap` 的改动必须在同一个 PR 里更新 TypeScript 与 Python SDK 的期望输出，而 `pnpm run test` **两者都不覆盖**（[T1: 仓库根 AGENTS.md]）。

## 去哪深入（文件路由）

| 问题 | 去这里 |
|---|---|
| 组边界（不管项目脚手架）、三包分工 | `packages/sdk/README.md` |
| 传输帧格式、错误码（-32601/-32603）、`JsonRpcResponseError` | `packages/sdk/protocol/src/transport.ts`、`packages/sdk/protocol/README.md` |
| 方法/通知类型表、`maxTokens` 语义、`messageId` 语义 | `packages/sdk/protocol/src/types.ts` |
| `DeepSeekHarness` / `HarnessClient` 用法、错误类型、dispose ladder | `packages/sdk/client/README.md`、`packages/sdk/client/src/{api,client,dispose}.ts` |
| 服务端接线、Config、shutdown/exit 语义、wire notes | `packages/sdk/server/README.md`、`packages/sdk/server/src/server.ts` |
| 可运行的 JSON-RPC 应用外壳 | `packages/examples/jsonrpc-demo/` |
| Python SDK 与打包 runtime | `python/README.md` |
| 客户端契约的设计决策 | `.agents/notes/implemented/feature/2026-07-27-typescript-sdk-and-sdk-subagent-backend.md` |
| 为什么砍掉 SDK 项目工具链（产品边界） | `.agents/notes/implemented/simplification/2026-08-11-remove-sdk-project-toolchain.md` |
| `maxTokens` 上限的决策 | `.agents/notes/implemented/feature/2026-07-28-sdk-max-output-tokens.md` |
