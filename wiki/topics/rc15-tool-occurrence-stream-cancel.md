---
title: 1.5-rc.2 工具 occurrence 归属与逻辑流取消
description: 区分 prompt 身份、真实工具 producer、已接纳输入与 carrier 取消。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-21
updated: 2026-09-21
asked_by: agent
anchors:
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent-loop/src/tool-calls.ts
  - packages/core/tools/src/index.ts
  - packages/core/session/src/index.ts
  - packages/api/session-controller/src/index.ts
  - packages/api/session-controller/src/commands.ts
  - packages/api/gateway/src/index.ts
  - packages/api/gateway/src/types.ts
  - packages/api/gateway/src/client/index.ts
  - packages/client/connection/src/client/rpc.ts
  - packages/client/connection/src/http-bridge.ts
  - packages/client/connection/tests/client-apply.client.spec.ts
related:
  - "[[wiki/topics/rc15-publication-budget-live-routing]]"
---

固定0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203。远端tag一致；6个指定正式npm包sha512/exports/types核对。主答复及同咨询补充各完整回传成功后沉淀。未运行模型、Host或取消实验，L2 / verified_inference。

条件：TS宿主 · 正式SessionController/AgentLoop · per-Agent turn/step · 公开Session事件读取 · registry-owned execution · 外部准入不在本页证明 · carrier实现未实测 · 固定目标。

## producer与关联

prompt入口检查signal后不继续传给commands。commands准入生成UserMessage，source.kind=user且source.rpcId=request.requestId；message.id独立。先inbox/spliced与inserted，再driver turn/start、claim、assembly/pre-step。reject不开step不提交claimed user/message。

enter→step/start→agent/request/prepareCall→system/message→首attempt user/message→按需request/header/context→模型stream→assistant settlement→scheduler startCall写tool/call→ToolRuntime创建token/pre-execute→approval/guard→tools/execute wrappers→body→post/result通知→日志tool/result(sourceEventSeqs=[call.seq])。

pre-execute和execute wrapper入口都不证明body已完成；正常取消未started项可有synthetic call/result而无真实ToolRuntime路径。

Session root公开eventAt/snapshotEvents及seq类型；ToolExecution公开agent/token/callId/rootCallId/parent，但无promptRequestId/turn/step/callSeq。顶层通常parent缺席且rootCallId=callId，nested沿用root并有parent；这些单独不证明标准模型producer，程序直接execute也可带agent。

条件性关联须依exact Agent/Session与真实事件先后，匹配唯一call occurrence及相应turn/step内实际admitted user source.rpcId。token只标live execution、不在日志；provider callId可复用。并行/await期间最后日志项可已变化；arguments原字符串与执行对象也不同。多输入/steer合批、后续无新输入step、retry等无法被一条requestId自动一对一归属；歧义时不能声称唯一证明。result的精确sourceEventSeqs是事后链接，不是execution已有callSeq。禁止手工emit无producer事件补证据。

## 取消

Connection unary每call独立生成外层rpcId，与payload内requestId不同；fetch signal、Host连接中断abort只是transport/handler信号，原生carrier是否兑现另验。

Gateway.invoke只向声明cancellation的方法追加signal，await业务Promise，不通用race或回滚；业务拒绝且已abort映射cancelled，不保证忽略signal的业务不再成功。

stream用next/abort竞速，finally await iterator.return；底层不合作仍可能等待，关闭订阅不取消Agent。prompt仅入口throwIfAborted，之后async准入仍可入箱；已accepted后取消旧RPC不能撤回。

SessionController.cancel调用agent.cancel(user,{keepInbox:true})并立即accepted；保留pending inbox，不await idle/物理drain，不resume冷Agent。queue remove是不同操作且可能已claimed；不能用cancel当清队列。

## openStream补充

ClientTransportHooks.openStream：(endpoint,payload,signal)=>AsyncIterable<unknown>；Host Gateway.wireStream.open返回Promise<AsyncIterable<unknown>>。Client hook交已解码逻辑item，不是Response/SSE/NDJSON或WebSocket mux envelope；rpc.open只接受/api并转交同一signal。

普通item属各Remote域（如session/follow）；Gateway $events由真实producer发ready(clientId,host)、emit、waterfall、cancel。可原样运输wireStream产生的item，不手工伪造；内部stream-protocol完整union/parser不等于root公开导入面。

Connection不替任意hook实现signal race/drain，carrier须传Host取消并释放迭代。Gateway Client用normalizeConnectionStream重建带dshRemoteStreamFailure标记的Error；marker接口内部，不能称公共factory。公开Host wireStream.failure映射失败字段，Client另公开RemoteStreamCarrierError。

目标无公开Gateway HTTP streaming handler；默认网络流为内部WebSocket mux。createSharedFetchHandler只做exact Fetch/unary分派，invoke拒绝stream方法；可复用公开stream/wireStream逻辑边界，不导入私有mux/parser。

## 证据与限制

固定目标关键路径：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/tool-calls.ts`：startCall/appendToolResult。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/index.ts`：prompt入口signal。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/commands.ts`：cancel keepInbox。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/gateway/src/index.ts`：invoke/stream/cancellableStream。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/connection/tests/client-apply.client.spec.ts:486`：decoded openStream与signal转交fixture，仅读。

未认证外部Main identity映射、多输入预算归属或carrier物理排空；正式包核验不构成运行验收。
