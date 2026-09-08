---
title: rc.2 todo 与问答生命周期归属
description: todo 整表日志和 turn reset、Host pending、Client 应答与 AgentLoop 续步分层。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/todo/tool-todo/src/index.ts
  - packages/todo/tool-todo/tests/projection.spec.ts
  - packages/interaction/user-questions/src/index.ts
  - packages/interaction/tool-ask-user/src/index.ts
  - packages/host/apiproxy/src/api-proxy.ts
  - packages/host/apiproxy/tests/api-proxy-question.spec.ts
  - packages/client/runtime/src/client/sessions/pending.ts
  - packages/client/runtime/src/client/sessions/session.ts
  - packages/client/runtime/src/client/contract/session.ts
  - packages/core/agent-loop/src/tool-calls.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/packages/todo]]"
  - "[[wiki/packages/interaction]]"
---

条件：官方 rc.2 Host/Client · 普通 root Agent · todo projection 与 question pending · Session 持久化已组合 · 官方工具调用 · UI 只保留临时草稿 · live reconnect 与进程恢复分开 · 固定 `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`。

## todo

- todo_write 校验并写该 Agent Session 的整表 todo/write，状态为 pending/in_progress/completed；content trim 后非空且唯一。allowParallelInProgress 必填，false 限最多一个进行中，不强制最低一个。
- tool-todo 拥有 todos projection unit，SessionProjectionRegistry 驱动：init=null，todo/write 取整表，turn/start 清 null，turn/end 保留，stateVersion=2。下一 turn 清显示不删除旧日志；未装 unit 与值为 null 不同。
- SessionPersistence 保存/恢复事件，projection 重放恢复视图；工具 append 不另给 flush receipt。todo 没有独立 Provider/调度器，不创建 Goal/workflow/Job，不自动判断物理完成。

## 问答

- rc.2 UserQuestionService 是 registerProvider/ask seam，一个 context 一个 provider；不是新版 Remote waterfall 结构。要求有 Agent 时为 exact live root，缺 provider 拒绝。
- tool-ask-user 传 questions/Agent/signal 并 await answer，返回 answers canonical value、JSON text 工具结果。正式 Web ApiProxy 持有 pending Map/Promise，发 requested/resolved frames，并校验回应匹配。
- signal abort 或 Host provider dispose 撤回 pending；用户显式取消产生 ASK_CANCELLED，不自动等同整个 turn abort。live Client 重连可重投仍 pending 的同 rpcId；Host 重启不恢复同一 Promise。
- Client 从 requested 创建 PendingWait，Session observable snapshot.pending 供展示；respond(result) 自动填 rpcId，返回 carrier receipt，不代表 Agent 整轮完成。resolved frame 才清列表/markSettled；已 settled wait 再应答拒绝。
- 正常 answer 经 Host settle→service→tool→ToolRuntime→AgentLoop 标准 tool/result 与续步。UI 不需要另提交 prompt 续步；pending 与未提交草稿并非已持久工具历史。
- **条件性结论**：UI 只显示官方 todos projection/pending，并向当前官方 wait respond，草稿只作临时选择时，DSH 继续拥有上述生命周期。实际重连、receipt rejection、stale wait 与草稿处理未验证，不以接口存在宣称 UI 合规。

证据：[todo projection](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/todo/tool-todo/src/index.ts#L134)、[turn reset 测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/todo/tool-todo/tests/projection.spec.ts#L94)、[ask service](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/interaction/user-questions/src/index.ts#L52)、[Host pending](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L1299)、[回应结算](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L3609)、[PendingWait](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/client/runtime/src/client/sessions/pending.ts#L29)。

远端固定 tag 已核对；本次源码/类型/测试断言核验，未运行 UI 或问答 E2E，不检查外部项目。
