---
title: rc.1 程序工具的 open-turn 生命周期与记录归属
description: 公开输入关联、turn-stopping、plugin-source 日志、flush 及程序化工具审批的边界。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/api/session-controller/src/index.ts#SessionController
  - packages/api/session-controller/src/types.ts#SessionPromptRequest
  - packages/api/session-controller/src/commands.ts
  - packages/core/agent/src/runtime-types.ts
  - packages/core/agent/src/dispatch.ts#agentEvents
  - packages/core/agent/src/inbox.ts#Inbox
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent-loop/src/tool-calls.ts
  - packages/core/agent-loop/tests/contract-regressions.spec.ts
  - packages/core/agent-loop/tests/cancel.spec.ts
  - packages/core/session/src/index.ts#Session
  - packages/core/session/src/types.ts#SessionEventMap
  - packages/core/tools/src/index.ts#ToolRuntime
  - packages/interaction/user-approval/src/index.ts#ApprovalService
  - packages/interaction/user-approval/tests/approval.spec.ts
  - packages/client/ui-approval/src/client/index.ts
  - packages/client/ui-approval/src/client/ApprovalPanel.tsx
  - packages/llm/llm/src/message.ts#createUserMessage
  - packages/llm/llm/src/brand.ts#ToolCallId
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/topics/rc1-input-authority-retry]]"
  - "[[wiki/topics/rc1-question-delivery-lifecycle]]"
---

条件：TypeScript Host 插件 · 官方 CLI/Profile 与 Client · 同 Agent open turn · 公开 SessionStore checkpoint · 程序调用已注册工具 · 无模型 tool-call 的回复 · 本地部署 · 固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。

## 输入到实际 turn

`SessionController.prompt(request, signal)` 接收 caller-minted `requestId`，返回 `{accepted:true}`；无 caller source/MessageId/turn 参数。Host 创建 `source.kind=user`、`source.rpcId=requestId` 的新 UserMessage。signal 仅在进入 commands.prompt 前检查，不成为 Agent turn signal。详见[已有输入专题](rc1-input-authority-retry.md)。

输入持久 splice → `agent/inbox/inserted({agent,message})` → waking driver → `turn/start` → claim → `agent/inbox/claimed({agent,message,turn})` → assembly/pre-step → `step/start` → admitted `user/message` → `agent/request({agent,turn,step,signal},next)`。claimed 是直接来源关联，但 pre-step 仍可 reject/替换，不能推出原消息必被模型消费。

Inbox 通知和 post-commit `session/event(session,event)` 是观察，不等待异步回调，也不能 veto。`agent/request` 是 awaited call-config waterfall，不是接收通知；保持配置需要委派 `next()`。关联观察须先于提交输入建立，不能依赖 accepted 回包才开始订阅。

## 收尾尚未完成

- `ctx.on('agent/turn-stopping', async ({agent,turn,signal}) => {...})` 是公开 serial、awaited hook，无 step/finalMessage/endReason 参数。
- 它位于 `step/end` 后、`turn/end` 前；既有 stopping outcome 且 nextStep 为空时才进入。返回后检查 signal、重读 Inbox，steer 可继续同 turn 并再次抵达 hook；不是每 turn 恰好一次。
- `max-tokens` 可进入；blocked/早期空输入/异常等路径不保证调用。throw 会走 error 收尾；取消需要 callback/工具合作响应 signal，不能强杀未收敛 Promise。
- 通过 `agent.session.ownEvents()/snapshotEvents()/eventAt()` 读取该 turn 的 `assistant/message`，保留 event.seq/time、data.turn/step、message.id/content/source 与 interrupted 标记。这些是公开原始事件，不需要私有 reducer。
- stopping 时只能称“候选最终回复”。确认该 turn 正常完成要观察后来的同 turn `turn/end.reason.kind=completed`。`agent.whenIdle()` 是 whole-agent quiescence，不是特定输入/turn 回执。

## plugin-source 记录与耐久

```ts
import { createUserMessage } from '@deepseek-ai/dsh-llm'

const event = agent.session.append('user/message', createUserMessage({
  content: [{ type: 'text', text: JSON.stringify({ phase: 'intent', operationId: 'example' }) }],
  source: { kind: 'plugin', plugin: 'example-plugin', form: 'notice', summary: '记录操作意图' },
}), { surfaceOp: 'append' })
const participated = await ctx.sessions.flush(agent.session)
```

代码只示意公开调用形状，未执行。outcome 可用同一插件自有 operationId 写另一消息；这些 text 字段不是 DSH 内建事务。`append` 同步返回 frozen live event，不等磁盘；user/message 必须明确 surfaceOp，会成为模型历史，不是仅内部审计。它自身不写 Inbox、不唤醒或续步。`agent.inject/steer/followup` 才是不同的调度入口；running 下 inject 新增 nextStep 也可能使当前 turn 继续。

`ctx.sessions.flush(session): Promise<boolean>` 等全部 checkpoint listener 成功；无 listener 为 false，有 listener 为 true，失败等全部 settled 后抛出。没有 `session.flush()`。此 checkpoint 不原子提交外部效果，不覆盖之后才出现的 turn/end。loop 不在 turn/end 自动 await flush。不得在 session/event 同步观察回调里重入 append。

## 程序工具与审批 owner

- `tools.execute({callId,name,arguments,agent?,signal,rootCallId?,parent?})` 返回 `ToolExecutionResult`。调用者可用公开 `ToolCallId(crypto.randomUUID())` 建相关 ID；brand 不建立模型调用或幂等。token 由 runtime 生成，不能伪造 parent。
- 成功含 `isError:false,value,content,meta?,additionalContexts?,concludesTurn?`；失败含 `isError:true,error,content,...`，无成功 value。fulfilled 不等执行成功。
- `register(ToolDefinition)` 返回 exact disposer，按调用 Context scope 决定全局或 Agent 层；Agent scoped 可 shadow global，同层重名/保留 run_code 拒绝。定义必须含 output schema/render 和返回 canonical JSON 的 execute；输入验证由工具/公开 defineTool 拥有。注册撤销不强杀已运行效果。
- PTC-only 对无 parent 的原生 name 可在 policy 前拒绝；程序调用不自动绕过 scope/presentation。工具需转发 signal 并等自有工作 quiescent 才 settle。
- 审批不是每次 execute 必有：`tools/pre-execute` 返回 ask 后，runtime 才以 agent/name/callId/signal 调 `approval.request`；缺 agent/approval 拒绝。ApprovalService 要求 open turn，拥有 `approval/asked` 与 `approval/decided`；只有 allowed-once 放行，取消/无回答方/never policy 各自 fail closed。
- 官方 ui-approval 通过 Remote waterfall 和 Session scope 展示 reason/toolName；callId 仅供查已有调用 detail。没有模型 tool/call 时不能保证完整参数卡片或标准结果卡，不应伪造模型 tool-call 补齐。
- ToolRuntime 拥有执行管线、结果与同步 `tools/result` 观察；AgentLoop 的模型调用 dispatcher 才拥有标准 tool/call/tool/result、additionalContexts 入 Inbox 和 concludesTurn 消费。direct execute 不自动完成这些记录/调度。插件负责自己的 user/message intent/outcome 与 checkpoint；value 不是自动耐久字段。

## 证据与验证界限

[收尾控制流](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts#L298)、[append/flush](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/session/src/index.ts#L635)、[工具执行](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/tools/src/index.ts#L1320)、[模型工具日志 owner](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/tool-calls.ts#L262)。上游 contract-regressions/cancel/approval 测试只读断言，未运行。

精确 rc.1 正式 tarball 在内存读取 exports 与 `lib/types/index.d.ts`：agent、agent-loop、session、tools、user-approval、llm、api-session-controller 的上述公共面可达。内部 loop 文件仅作实现证据，不作为调用入口。

尚未验证完整“无模型 tool-call 回复 → stopping 程序工具 → 官方审批 UI → 合作取消 → 最终 turn checkpoint/恢复”组合；不能从接口存在推出 E2E 通过。实际 competing steering、输入归属、无调用卡时展示内容、物理效果结果与取消窗口均须具体组合测试，不替调用方做设计裁决。
