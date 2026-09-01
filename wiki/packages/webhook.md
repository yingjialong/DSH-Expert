---
title: packages/webhook — 已认证外部事件到普通 Session
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/README.md
  - packages/webhook/README.md
  - packages/webhook/webhook/package.json
  - packages/webhook/webhook/src/index.ts
  - packages/webhook/webhook/src/session.ts
  - packages/webhook/webhook/tests/runtime.spec.ts
  - packages/webhook/webhook/tests/session.spec.ts
  - packages/webhook/webhook-github/package.json
  - packages/webhook/webhook-github/src/handler.ts
  - docs/subsystems/webhook.md
commit: cd5ef8148158c3a752a658978873241fdf8e2bbc
verified_at: 2026-08-30
asked_by: agent
---

# webhook/ — 已认证外部事件到普通 Session

## 定位

`Product — stable API`。alpha.1 新增两个 intended public package：

| 包 | 角色 |
|---|---|
| `@deepseek-ai/dsh-webhook` | `ctx.webhookRuntime` Service；注册可信规则并把非空结果创建成普通 Workspace root Session |
| `@deepseek-ai/dsh-webhook-github` | GitHub HMAC HTTP adapter；认证、有界读取并标准化 lossless JSON 后 dispatch |

两者的 tag manifest 均声明 root、`/types`、`/invariant` exports；但 alpha.1 没有 npm tarball，不能把 manifest 意图说成已发布的 npm 闭包。

## 核心语义

- `WebhookRuntime.register(rule)` 是 effect-owned 注册；dispose 先隐藏规则，再 abort 并 drain 活跃 callback。
- `dispatch(delivery)` snapshot/freeze 输入，按 kind 启动当时匹配的规则并立即返回；单个规则异常隔离。
- 规则返回 `null` 或一个 `WebhookSessionRequest`。非空请求必须给绝对 `workspacePath`、title、prompt、agent preset、permission preset，可选 explicit model。
- runtime 在 publication 前校验 presets，解析或创建 Workspace，创建 Agent、mount preset、attach Session，再设置 permission/title 并 admission 普通 user follow-up。
- `Agent.followup()` 成功入 inbox 是 webhook operation 的 commit point；随后完全回到普通 Agent/Session 生命周期，runtime 不等 idle 或回复。
- GitHub adapter 每请求解析 credential、在 JSON parse 前验 untouched body 的 SHA-256 HMAC，验证成功后返回 `202`；这不表示规则命中、Session 创建或 Agent 完成。

## 边界

- 无持久 delivery queue、retry、dedup、crash replay、completion state 或 downstream acknowledgement。
- rule callback 是 same-process trusted code；必须协作响应 AbortSignal，runtime 不能强杀忽略取消的代码。
- provider adapter 拥有认证和 intake；rule 拥有 provider event 字段校验、条件、外部调用与 prompt 的 trust labeling。
- webhook 只是 optional Host plugin，不是 base/web profile 默认挂载能力。CLI closure 安装不等于 mounted，更不等于 published-to-agent。

## 扩展点

- out-of-tree provider 可 merge-extensible `WebhookEventMap`，构造 `VerifiedWebhookDelivery<K>` 后调用 `dispatch()`。
- out-of-tree trusted rule 注入 `webhookRuntime` 并以自己的 effect yield `register()` disposer。
- rule 只能通过公开 Session request 创建普通 root Session；不能把 webhook runtime 当 durable job/workflow owner。

## 陷阱

1. `202` 只证明已认证 payload 进入进程内 dispatch，不证明产生 Session。
2. delivery id 只是 provenance，runtime 不去重。
3. repeated delivery 会重复执行规则，可能创建重复 Session。
4. `model` 省略会 snapshot 当前 deployment selection；explicit route 使用 adapter reasoning default。
5. callback teardown 只对观察 signal 的异步代码有效。
6. 新建 Workspace 在后续 Session rollback 时可能保留，因为并发调用者可能已经使用。
7. adapter 只保证 authenticated generic JSON；GitHub event-specific shape 由 rule 验证。
8. alpha.1 只有 tag source，没有可验证的正式 npm 闭包。

## 未决

- 后续是否加入 durable delivery queue/retry/dedup：未知。
- alpha.1 最终是否会发布同形状 npm tarball：未知。
