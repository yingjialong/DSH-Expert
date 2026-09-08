---
title: rc.1 初始模型选择与 Preset 分离
description: CLI/Profile Host 的默认模型、Session 待消费选择与 model/selection 记录边界。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/core/agent-default-model/src/index.ts#AgentDefaultModelConfig
  - packages/core/agent-default-model/package.json
  - packages/bundle/base/cordis.patch.yml
  - packages/api/session-controller/src/index.ts#SessionController
  - packages/api/session-controller/src/types.ts#SessionSelectModelRequest
  - packages/api/session-controller/src/commands.ts
  - packages/api/session-controller/src/agent.ts
  - packages/api/session-controller/src/model-selection-projection.ts
  - packages/api/session-controller/tests/session-models.host.spec.ts
  - packages/core/agent/src/model-selection.ts#installModelSelection
  - packages/core/agent/src/runtime-types.ts#AgentOptions
  - packages/core/agent-loop/src/index.ts#Config
  - packages/preset/agent-presets/src/preset.ts#Config
  - packages/preset/agent-presets/src/index.ts#AgentPresets
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/topics/rc1-input-authority-retry]]"
  - "[[wiki/packages/llm]]"
---

条件：TypeScript Host · 完整 CLI/Profile 与官方 SessionController · blank/非 blank 分开 · SessionStore 持久化 · 不改变工具组合 · 已注册 LlmAdapter route · 本地运行 · 固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。

## 初始默认与显式选择

- `@deepseek-ai/dsh-agent-default-model@0.1.2-rc.1` 提供 `ctx.agentDefaultModel`，公开 class/default export 为 `AgentDefaultModelConfig`。Loader Config 仅必填 `provider`、`model`；base 已有 id `agent-default-model` 的 row。修改已有 row 的 config，不需增加第二 service。
- `agent-default-model` settings namespace 另允许 `reasoningEffort?`。`currentSelection()` 读合成值，用户 settings 可覆盖 composition；`saveSelection()` 保存完整选择，无 settings 时 no-op。默认服务自身不检查 adapter availability。
- `SessionController.create` 只有 workspaceId?/cwd?/sessionId?/agentPreset?，无 provider/model。Host 创建 Agent 的基础 options 来自 default-model。
- Host `selectionFor.current` 优先 pending selection，否则 latest requestHeader（剔除 adapter-defaulted effort），否则 live default。没有 pending/header 的 blank Session 可继续随默认值变化，不是 create 时永久冻结。
- 单 Session 的最小公开顺序：route 已注册 → `await ctx.sessionController.create(...)`（或已有 Session）→ `await ctx.sessionController.selectModel({sessionId,provider,model,reasoningEffort?})` → prompt。返回 `{selected}` 是规范化后接受的值。
- selectModel 调用 `llm.resolveCallConfig`；目录未列出的 model 不必然拒绝，exact adapter 合同决定。后置 `agent/request` 改 route 不等于更新 Host admission 提前读取的 current selection。

## rc.1 记录事实

`selectForNextRequest` 先 append `model/selection`，再修改 live picked selection。Controller 随后尝试保存默认值；settings 保存失败只 warn，Session 选择仍接受。不要沿用旧版“selectModel 只改内存、直到 request/header 才留痕”的结论。

modelSelection projection 保存 lastUsed/pending，匹配的 request/header 消费 pending；返回视图为 lastUsed/next。append 是 live commit，selectModel 没有显式 SessionStore.flush，不把它写成已落盘回执。Controller 的正常副作用还包括尝试保存新默认，并非严格只影响当前 Session。

## 与 Preset 的区别

- selectModel 对 blank 和非 blank 都可用，无 blank 检查，不调用 recompose。assembly 捕获 selection 并交给该 step 的 request，之后更改不回溯修改已捕获请求。
- cold Session 的 selectModel 会先 resolve/resume，恢复自身可按 recorded preset 装配；这不同于为切换 model 而调用 Preset recompose。
- `AgentPresets.select(agent,id)` 才检查 turnBoundary，open turn 或 lastTurn>0 时 `agent-preset/locked`；blank 则 recompose 后记 `agent-preset/selected`。
- Preset roster Config 是 default/roots/includeShippedRoot/includeUserRoot，不含 provider/model。Preset metadata 也没有这两个字段。
- AgentOptions 有 provider?/model?/reasoningEffort?/maxTokens?；AgentLoop Config.agents[] 加 id/sessionId?/cwd?/resumeSessionId?，用于声明启动 Agent。它不替代 Controller 创建任意 Session 所读取的 default-model。

## 证据及验证

[默认服务](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-default-model/src/index.ts#L21)、[选择优先级与立即 append](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/agent.ts#L276)、[selectModel](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/commands.ts#L119)、[modelSelection projection](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/model-selection-projection.ts#L38)、[Preset 切换](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/preset/agent-presets/src/index.ts#L693)。

本次核对远端固定 tag。default-model 正式 rc.1 tarball 的 manifest/root `.d.ts` 在内存检查，确认 class/Config/settings schema/currentSelection/saveSelection 可达；Controller/Agent/Loop/LLM 正式公开面复用本系列同版本核验。内部 selectionFor/projection/commands 仅是实现证据，不作私有导入建议。上游 session-models 的保存失败、unroutable prompt 与 unlisted model 测试只读断言，未运行；未验证具体部署的并发 prompt、adapter 接受性或存储结果。
