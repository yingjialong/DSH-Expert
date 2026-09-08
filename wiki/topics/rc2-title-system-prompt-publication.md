---
title: rc.2 自动标题、system 映射与 prompt 投影时序
description: 区分异步标题、模型系统提示映射、输入接收回执与官方 Client 可见性。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/session/session-title/src/index.ts#SessionTitleService
  - packages/session/session-title/src/normalize.ts#fallbackSessionTitle
  - packages/session/session-title/tests/provider.spec.ts
  - packages/bundle/base/cordis.patch.yml
  - packages/llm/llm-deepseek/src/serialize.ts#serializeRequest
  - packages/llm/llm-deepseek/tests/serialize.spec.ts
  - packages/llm/llm-pi-ai/src/context.ts
  - packages/llm/llm-pi-ai/package.json
  - pnpm-lock.yaml
  - packages/host/apiproxy/src/api-proxy.ts
  - packages/client/runtime/src/client/contract/session.ts
  - packages/client/runtime/src/client/sessions/session.ts
  - packages/client/runtime/src/client/sessions/manager.ts
  - packages/client/runtime/src/client/sessions/notifier.ts
  - packages/client/runtime/tests/session.client.spec.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/packages/session]]"
  - "[[wiki/packages/llm]]"
  - "[[wiki/packages/client]]"
---

条件：TS/JS · 官方 Host 与普通 Client SessionFace · 自动标题与主请求并行生命周期 · Session 日志和 Client baseline/mux · 不改变工具 · 官方 DeepSeek/pi-ai · 本地部署 · 固定 `0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`。

## 自动标题不是 admission 等待门

- `SessionTitleService.register` 注册唯一 provider，返回异步 disposer（撤销会 abort/drain）。automatic 为 first-prompt/all-prompts，不在 register 时等待生成。
- user/message 已提交后，onUserMessage 只安排 pending 与 deferred fallback。eligible 消息要求 source.kind=user 且有清理后非空文本；first-prompt 还要求非 fork、第一条 eligible message、没有 title。用户 rename pin 住 title。
- provider 等来源消息之后的 request/header route，或符合检查条件的 marked AgentLoop llm/stream，再通过 defer/Promise.then 启动。runProvider 先确保 fallback 再调用 generate；普通 prompt 不等待这一 Promise。显式 refresh 是另一个会等待刷新的入口。
- 不 await 不等于线程隔离；provider 自身同步 CPU 工作仍占所在 JS event loop，本页不据此诊断任何部署。
- fallback 取 Session.events 第一条 eligible human message 的全部 text blocks，清理控制/转义/方向字符、折叠空白，再按单词数与 UTF-8 字节数取前缀。它不来自 assistant/system/id/path；无 eligible text 则无 fallback。
- fallback 写 session/title，source=fallback，messageSeqs=[first.seq]。shipped base 配置为 5 words / 40 bytes，maxTitleBytes=80；组件 Config 要求显式 limits，不能把 shipped 值当所有 composition 固定值。

## system 到 adapter wire

- DeepSeek serializeRequest 与 image 路径在 system!==undefined 时先插 `{role:'system',content:options.system}`，再序列化历史；有直接测试。
- pi-ai adapter 将它放入 PiContext.systemPrompt；最终 wire 由协议 formatter 决定，不能统一称为 role=system。
- rc.2 tag lock 的 pi-ai 为 0.82.1（manifest 范围 ^0.82.1）。只读该精确 tarball：Completions 依 reasoning/compat 选择 developer 或 system；Responses shared converter 也依 reasoning/compat 选择；Anthropic 则放顶层 system text blocks。空字符串在下游 truthy 判断中可被省略；不保证模型服从。
- 此 pi-ai 核验不证明任意只锁 DSH 根包的部署实际解析同一递归版本。

## prompt 回执与 Client 可见性

- 普通 SessionFace.prompt 返回 Promise<RpcResult<{accepted:true}>>，Host 在 admission、Agent.steer/followup 后应答，不等待整轮、自动标题或 Client 渲染。fulfilled 仍需检查 result.ok。
- Client 入口同步改 promptAttempted/firstPromptPendingTurn 并 markDirty，composerPhase 可先变 engaging；成功降低 blankBit，但不凭该回执插入真实 user/message 或设置 running=true。
- 用户聊天内容来自 session/event mux 或 history baseline，经 Conversation 形成 chat/views；loading 缓冲、cold/error 等 open/backfill。队列中的输入或 pre-step 未进入/拒绝，不因 accepted 就已有 transcript message。
- running 来自 Host agent/status→host/session-status→Client handleRunning，与 RPC 回包、user/message 不共享统一渲染顺序。它可先于用户消息发布。
- title 是独立 Host projection，不 gate 聊天/running；Notifier 将结构更新合并到 microtask，流式可按 animation frame。await prompt 不是屏幕绘制 barrier。

## 证据与验证

[标题调度](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/session/session-title/src/index.ts#L462)、[fallback](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/session/session-title/src/index.ts#L755)、[provider 测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/session/session-title/tests/provider.spec.ts#L115)、[DeepSeek system](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm-deepseek/src/serialize.ts#L378)、[pi-ai Context](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm-pi-ai/src/context.ts#L126)、[prompt 合同](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/client/runtime/src/client/contract/session.ts#L35)、[Client phase 测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/client/runtime/tests/session.client.spec.ts#L520)。

远端 rc.2 tag 已核对；全部 DSH 证据按固定 SHA 读取。下游 pi-ai 0.82.1 只在内存读取 dist/api/openai-completions.js、openai-responses-shared.js、anthropic-messages.js，包内逻辑路径不是本机安装定位。未运行真实模型/标题 provider/Host-Client E2E，不评价外部项目。
