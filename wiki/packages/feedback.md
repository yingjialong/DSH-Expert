---
title: packages/feedback — 人类反馈的两条互不相通的契约
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/feedback/README.md
  - packages/feedback/command-feedback/README.md
  - packages/feedback/message-feedback/README.md
  - packages/feedback/command-feedback/src/index.ts
  - packages/feedback/message-feedback/src/types.ts
  - docs/subsystems/feedback.md
  - docs/capability-seams.md
  - .agents/notes/implemented/architecture/2026-08-10-message-feedback-sidecar.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

记录人类反馈，且**故意拆成两份互不相通的契约**：一份是 canonical session log 里不可变的 `feedback/record` 事件，一份是挂在单条 assistant message 上、可编辑的本地 sidecar。**两者都不进入模型对话。**

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `command-feedback/` | `@deepseek-ai/dsh-command-feedback` | 触发方式无关的 `feedback/record` 事件写入路径 + 人机 `/feedback` 命令 |
| `message-feedback/` | `@deepseek-ai/dsh-message-feedback` | 每条 assistant message 的评分/备注 sidecar + Host `messageFeedback.list/put/delete` Remote 契约 |

## 三件套结构

这一组**不是 capability seam**。`docs/capability-seams.md` 把 `ctx.messageFeedback` 标为 Role `core`（Implementations 与 Consumers 两列都是 `-`），`command-feedback` 根本不注册服务。所以：

- **Service Definition**：无。想换实现只能换整个包。
- **Service Provider**：`message-feedback` 提供 `ctx.messageFeedback`（`MessageFeedbackService`）；`command-feedback` 无服务，只导出函数 `recordFeedback(session, text)` 并向 `ctx.commands` 注册一个 global command。
- **Consumer**：
  - `command-feedback` 的消费者是**可选的** `@deepseek-ai/dsh-session-telemetry-otel`——它 observe `feedback/record` 来释放挂起的 telemetry prefix，或警告 telemetry 关闭时反馈只留在本地。**捕获本身与该策略无关。**
  - `message-feedback` 的 Host Remote 契约随服务发布，但**Client Remote aggregate 挂载与 UI consumer 另有归属且已延后**。

## 扩展点

- **想在 `/feedback` 之外记录反馈**：直接 import 并调用 `command-feedback` 导出的 `recordFeedback(session, text)`——这是 command-independent 的写入路径，UI / hook / host 集成都可以用，不必构造 slash command。
- **想消费反馈事件**：监听 `feedback/record`（log-only session event，不进 ordered surface / `deriveMessages()` / 模型请求）。
- **想读 telemetry 分享状态**：`command-feedback` 用 `ctx.get('telemetry')` 读，**从不声明 injection**——这是「可选服务用 `ctx.get(name)`」纪律的现成范例。
- `message-feedback` 的扩展面是 Remote：`messageFeedback.list` / `put` / `delete`，三个方法都返回判别联合 `{ ok: true, value } | { ok: false, error }`。类型从包根与 `@deepseek-ai/dsh-message-feedback/types` 导出。
- 必填 config：`maxNoteBytes`（正安全整数，单条 note 的 UTF-8 字节上限）。`message-feedback` inject `storageDomain`、`sessionPersistence`、`sessions`，durable domain 名为 `message_feedback`。

## Known Limitations

组 README 无该章节；以下来自两个包的 `## Known Limitations and Deferred Work`。

**command-feedback**
- 没有任何检索 / 聚合 / 分类 / model-facing 工具——OTel 插件只把事件当「分享触发器」用。
- 无结构化字段：一条就是一个自由文本串，无 category / severity / 关联事件。
- 无修改无撤回：log 追加式，本包不加 tombstone，写错只能再写一条覆盖语义。
- **无显式持久化屏障**：ack 跟在 append 后面而非 flush 后面，崩溃前一刻的反馈会随未落盘尾部一起丢；需要保证的消费者自己 `await ctx.sessions.flush(session)`。
- 空白 session 上执行 `/feedback` 事件会记但**没有 ack 行**（Web transcript 只在 session 激活后渲染 command 行）。
- **只有 Web 有 command adapter**：headless / ACP / JSON-RPC 都用不了 `/feedback`。

**message-feedback**
- Client aggregate 与 UI 缺席。
- **compare-and-set 只在单进程内成立**：per-Session promise queue 只串行化一个 service 实例；多 Host 进程写同一 storage root 仍会丢更新（storage-domain 没有跨进程条件写）。
- 无 durable session 删除级联：`session/disposed` / `host/session-removed` 只表示 detach 不表示删除，因此会留空行与孤儿行。
- detach 到 catalog 物化之间的窗口会误返回 `session-not-found`，调用方需重试。
- header identity `{createdAt, cwd}` 不是内容指纹：克隆日志若保留同样 header 无法区分。
- **无认证调用方边界**：`list`/`put`/`delete` 不带 actor 或审计身份，部署必须只在可信/另行认证的边界暴露 Host gateway。
- 冷请求会扫描整个 session snapshot catalog；单 Session 行的条目数与总字节没有上限。

## 陷阱

1. **两条契约别混**：message feedback **不是** session event 也**不是** projection，不发 `feedback/record`，不触发 `FEEDBACK_ONLY` telemetry 释放；fork 因为 session identity 不同，**不会**复制 feedback 行。
2. **`/feedback /plan felt slow` 整串都是反馈内容**，不会被当成另一个命令解析。除首尾空白外不做任何截断 / 大小写折叠 / 控制词处理。
3. **反馈文本只在一个持久化载荷里出现**：`feedback/record`。`dsh-commands` 仍会记 `command/run` / `command/done`，但本命令设了 `recordInput: false`，所以 `command/run` 不带 `args`。
4. **ack 只说明「进了 log」，不说明「落了盘」**，也不承诺投递或留存——披露句只陈述部署当下的 sharing 策略。
5. **首次被接受的反馈可能创建 `$DSH_HOME/.anonymous-user-id`**（来自 `packages/identity/anonymous-user-id`）。
6. `ifVersion: null` 才是「仅创建」；对已存在项的**任何**请求（包括值完全相同的 no-op）都必须带上精确 current version。丢失响应后用旧 token 重试会拿到 `version-conflict.current`，里面带权威条目，不用再 `list` 一次。
7. `put` 只接受**非空、append-origin 的 `assistant/message`**；replacement-origin 副本、空的 usage-only assistant 记录、非 assistant 记录都返回 `target-not-found`。

## 去哪深入（文件路由）

| 问题 | 去哪 |
|---|---|
| 两条契约的分界与 telemetry 关系 | `packages/feedback/README.md` |
| `/feedback` 命令契约、四种 sharing 披露句 | `packages/feedback/command-feedback/README.md` |
| 三个 Remote 方法的请求/成功值/错误码表、CAS 与幂等 | `packages/feedback/message-feedback/README.md` |
| 公开类型定义 | `packages/feedback/message-feedback/src/types.ts` |
| 子系统级参考 | `docs/subsystems/feedback.md` |
| `ctx.messageFeedback` 的角色分类 | `docs/capability-seams.md` |
| sidecar 设计边界的源头 | `.agents/notes/implemented/architecture/2026-08-10-message-feedback-sidecar.md` |
