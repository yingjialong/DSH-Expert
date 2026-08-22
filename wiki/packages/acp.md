---
title: packages/acp — Agent Client Protocol 自动化服务端
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/acp/README.md
  - packages/acp/acp/README.md
  - packages/acp/acp/src/index.ts
  - packages/acp/acp/package.json
  - packages/subagent/subagent-acp/README.md
  - docs/postmortem/0001-acp-default-export-drops-inject.md
  - packages/README.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

把 harness agent 通过 stdio JSON-RPC 暴露给程序化客户端的 **Agent Client Protocol server**；README 明确它是 interoperability transport，**不是 UI 集成、也不是 capability seam** [T1: packages/acp/acp/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

| 包名 | npm 名 | 一句话职责 |
|---|---|---|
| `acp` | `@deepseek-ai/dsh-acp` | Automation-only ACP server：在 stdin/stdout 上开 `AgentSideConnection`，驱动 `ctx.agents` |

组内只有 1 个包，组 README 的全部内容就是这张表 + 一句"对应的 ACP **客户端**在别处"。

## 三件套结构

**本组没有三件套**，这是要点而非遗漏：

- **Service Definition**：无。`dsh-acp` 不注册任何 ctx key，它是**函数插件**（`export const name = 'acp'` / `inject = ['agents']` / `Config` / `apply`）[T1: packages/acp/acp/src/index.ts]。
- **Provider / Consumer**：协议对端不是 Cordis 插件，而是外部进程。
- 跨组综合：ACP 的**客户端半边**在 `subagent/subagent-acp`，之所以不在本组，是因为它实现的是 subagent provider 接口——即"按角色归组，不按协议归组"。

它消费的能力（peerDependencies，可视为隐式依赖清单）：`dsh-agent`(`ctx.agents`)、`dsh-attachment`、`dsh-llm`、`dsh-session`、`dsh-user-approval` [T1: packages/acp/acp/package.json]。

## 扩展点

想在这条路上扩展，**不要改 acp 包**：

- **换模型路由** → 插件 `config.provider` / `config.model`（两者都可选，留空则由另一个 agent/request 监听器提供；但 runnable ACP composition 要求都填）。
- **加协议能力**（editor / terminal / fs / MCP / mode / elicitation）→ 目前一律不 advertise，需要改 `initialize` 协商本身，无扩展点。
- **改权限应答** → 走 `dsh-user-approval` 的 approval seam；`session/request_permission` 只是把 bridge-owned 请求转成 one-shot allow/reject。
- **改图像准入** → 走 `dsh-attachment` seam；ACP 只在"挂了 durable attachment store **且** 配置的 exact provider/model 解析出显式 image input"时才 advertise image prompt。
- **要富 UI / 实时进度** → 这不是本组的目标，去 Web host + client 那条路。

## Known Limitations

摘自 `packages/acp/acp/README.md#Known Limitations and Deferred Work`：

- **Fresh sessions only** —— load / list / resume / delete / fork 全部不支持。
- **Raster images and one workspace only** —— 仅 PNG/JPEG/WebP/GIF；audio、embedded resource、非空 `additionalDirectories`、`mcpServers` 一律 reject；resource link 被压平成文本引用而非抓取内容。
- **Committed answers only** —— live progress、reasoning、tool activity、plan、title、usage 都不上线。
- **Connection-owned lifetime** —— 一条连接释放它的全部 session，没有 per-session close。

## 陷阱

1. **stdout 是协议帧专用**。任何往 stdout 打印的调试输出都会污染 JSON-RPC 帧 [T1: packages/acp/acp/README.md#Plugin]。
2. **函数插件禁止有 default export**——本仓 0001 号 postmortem 的主角就是这个包：混用 default export 会让 Loader 丢掉函数插件的 namespace，`inject` 静默失效（见 `docs/postmortem/0001-acp-default-export-drops-inject.md`）。
3. **只输出 committed 文本/图像**，不做逐 token 流式。这是刻意用延迟换"干净的自动化结果"：未提交的 provider chunk 和重试尝试不会泄漏半截文本。
4. **`stopReason` 不是 turn 级结论**。bridge 不声称 prompt 专属的 turn outcome：interval 从 prompt 进入 Agent inbox 起算，到 admission + whole-Agent idle + ordered output delivery 全部静默才结束；settlement 优先级是 explicit cancellation → output-delivery failure → interval-wide Agent failure → correlated turn ending。**token-limit 结束会报成 `end_turn`**，容易被误判为正常完成。
5. **`session/cancel` 有两段语义**：prompt 尚未进 Agent inbox 时只中止 admission，不打扰无关 Agent work；已进入则取消该 Agent 并等待 quiesce。无 in-flight prompt 时它取消 autonomous work，未知 id 是 no-op。
6. **一次 prompt 一个 session**：每 session 只允许一个 in-flight request，delivery 按 session 串行（因为 attachment 读是异步的）；损坏或缺失的 committed image 会让整个 prompt 失败，而不是降级成占位图。
7. **teardown 只清自己那片森林**：ACP 插件 reload 只 drain 本连接 exact owned Agents 之下的 continuable descendants，共享同一 Context 的其他前端不受影响。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 组定位、与 subagent-acp 的分工 | `packages/acp/README.md` |
| 逐方法协议契约表（initialize / session-* 的确切行为） | `packages/acp/acp/README.md#Protocol contract` |
| lifecycle、teardown 顺序、`stopReason` 判定 | `packages/acp/acp/README.md#Lifecycle` |
| Config 字段与插件形态 | `packages/acp/acp/src/index.ts` |
| 内容块编解码（text/image/resource_link） | `packages/acp/acp/src/codec.ts`、`packages/acp/acp/src/content.ts` |
| 反向的 ACP 客户端（把外部 agent 当 subagent 用） | `packages/subagent/subagent-acp/README.md` |
| default-export 吞掉 inject 的事故 | `docs/postmortem/0001-acp-default-export-drops-inject.md` |
| 跑起来 | 仓库根 `pnpm run demo:acp`（需要 `DEEPSEEK_API_KEY`） |
