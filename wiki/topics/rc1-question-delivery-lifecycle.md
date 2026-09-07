---
title: rc.1 user-questions 的 pending、展示委派与重连边界
description: 固定rc.1下问答请求与展示投递的生命周期证据
type: reference
updated: 2026-09-07
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/interaction/user-questions/src/index.ts
  - packages/api/gateway/src/index.ts
  - packages/api/gateway/src/client/remote-events.ts
  - packages/api/gateway/tests/gateway-stream.host.spec.ts
  - packages/api/remotes/src/index.ts
  - packages/client/ui-user-questions/src/client/index.ts
  - packages/client/ui-user-questions/src/client/contract/slots.ts
  - packages/client/ui-user-questions/src/client/draft-store.ts
  - packages/client/ui-session/src/client/index.ts
  - packages/interaction/tool-ask-user/src/index.ts
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-07
asked_by: agent
---

条件：TS/JS · 官方Host/Remote与Client问答UI · root Agent pending ask · Session存储 · tool-ask-user · 无真实模型调用 · live Host重连与进程重启分开 · 固定rc.1。

固定0.1.2-rc.1 / a66e4702047846cdaa10c66c9d3df3951f5ea70d。本次远端tag与本地对象一致，按完整SHA读取。以下为源码/类型/一方测试源码交叉核验VERIFIED_INFERENCE（本库L1/L2封顶，不标runtime FACT）；未运行完整浏览器/Host重启测试，不检查调用方项目。

结论：必须分三层。单纯Client连接generation丢失通常保留Host live pending并给新Client重投；presentation主动delegate可能让原Host waterfall继续到下游并结束；原request signal或Host Context/Remote source结束则取消原pending，不等待重连。Host重启没有恢复同一个ask Promise或未提交UI草稿的合同。

1. Host original request与wire delivery

UserQuestionService.ask校验exact live root Agent（有Agent时）与问题后，进入agent-scoped user-questions/request waterfall，默认终端是NO_PROVIDER。原request signal原样交给转发层；服务本身没有持久question queue，也没有“所有answerer消失就永远等待新listener”的通用逻辑。
[ask实现](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/interaction/user-questions/src/index.ts#L82-L154)、[Remote事件选取](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/remotes/src/remote-events.ts#L30-L35)。

Gateway把该Host waterfall保存在pendingRemoteEvents内存Map，投影原request并绑定原signal和Host Agent Context effect：
- 原signal abort或Host Context释放→cancelRemoteEvent→删除pending、reject source，并向当前Client发cancel。Remote event source/Gateway关闭也逐个reject。不是再投递的场景。
- 仅某Client $events/物理连接结束→removeRemoteEventClient只移除该client及其delivery，不settle/delete pending。新Client打开$events时，对所有仍pending重新deliver，保留同一个eventId。若当时无任何Client，pending可继续等新Client连接，只要原request/Host owner仍存活。
[original lifetime](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/index.ts#L464-L515)、[新Client重投](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/index.ts#L412-L426)、[disconnect与settlement分离](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/index.ts#L527-L587)。
一方明确测试同eventId重投replacement generation：[gateway-stream.host.spec.ts](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/tests/gateway-stream.host.spec.ts#L809-L841)；Host signal/Context取消测试从[843行](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/tests/gateway-stream.host.spec.ts#L843-L864)开始。本次只阅读测试源码。

Client每代接收时创建delivery signal，融合connection generation与该event cancel controller；原Host signal不会以同一个JS AbortSignal对象跨wire。代结束时abort所有本地delivery并等待listener tasks settle。Client answer若看到delivery signal已abort，直接return，不把这个展示结束错误发回成原request rejected。故“Client PendingQuestion报ASK_ABORTED”不自动证明Host原ask signal已abort，可能仅是wire generation结束。
[Client generation](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/client/remote-events.ts#L121-L182)、[aborted delivery不发回复](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/client/remote-events.ts#L185-L220)。

2. presentation delegate / scope销毁 / 页面重载不是同一分支

官方ui-user-questions.answerQuestion建立PendingQuestion并注册pending interaction domain。domain teardown调用pending.delegate，再await completion；answerQuestion识别delegation sentinel后调用next()，而非当作人类取消。
[answerQuestion](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/ui-user-questions/src/client/index.ts#L53-L79)、[domain teardown](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/ui-session/src/client/index.ts#L299-L322)、[delegate测试](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/ui-user-questions/tests/browser-plugin.client.spec.ts#L188-L199)。

如果该delivery仍有效，Client全部listener delegate则发送outcome next。Host移除此delivery；当pending.deliveries.size为0时，settle为next并继续原Host waterfall下游；下游无人承接则UserQuestionService默认NO_PROVIDER，原ask失败，随后新listener到来不会自动复活该已settled pending。某Client返回rejected也会直接取消Host pending；返回result则完成它。注意“最后一份next恰好在其他delivery断开后到达”也可结束pending，并非所有页面变化都保证重投。
[Host next/result/rejected](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/index.ts#L527-L545)、[转发next回Host下游](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/remotes/src/index.ts#L140-L154)。

Client scope找不到时，ClientRemoteEvents.answer初始outcome为next；若可用Client Context已无法resolve，会走next而不是自动等scope回来。单纯导航切Session不等于Host Agent Context销毁，也不必销毁该Client pending domain；不能仅凭“切Session”推断ask必失败或必重投。真正presentation domain卸载、Client Context无法resolve、整页导致transport先断、或Host Agent Context释放，要分别按上述路径判断。整页重载中如果先是纯连接代丢失且Host仍存活，能够重投；若有效next/rejected已被Host接受，重载不复活它。具体浏览器teardown竞争顺序未运行，UNKNOWN，不能给全场景必然保持的承诺。

PendingQuestion.cancel为ASK_CANCELLED；delivery abort为ASK_ABORTED；delegate是私有sentinel，三者不同。对象完成后重复answer会报already settled，后来的abort/delegate是no-op。
[PendingQuestion](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/ui-user-questions/src/client/contract/slots.ts#L131-L207)。

迟到response：
- Client delivery signal已结束，answer路径不再发送其结果。
- Host收回复按clientId找当前active event stream；旧clientId已经移除时返回RPC failure（no active event stream）。
- active clientId存在但event已settle、或已不属于该pending的有效delivery，receiveRemoteEventResult直接忽略，幂等no-op；不能覆盖新答案。重投用同eventId，不需要/没有独立deliveryId，clientId与delivery membership划分有效投递。
[active client查验](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/index.ts#L354-L368)、[late no-op](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/index.ts#L527-L545)。

3. Host重启、历史与UI草稿

Host pending map、Promise resolve/reject closures、Context effect与Remote client集合全是live内存；没有序列化恢复该pending ask Promise的公开合同。断线重投仅在同一个仍存活Host pending内成立，不能扩为跨Host进程重启恢复。

官方question draft store明文为transient Session store、non-persisted；selected/custom/skipped/index在Client slot-owned instance内。PendingQuestion.key为模块计数生成question:N，新presentation可有新key。导航切换时尚存的同store可保留数据，不代表整页重载/Client重建或Host重启保留草稿；草稿未提交前没有写到Host question结果的合同。
[draft-store完整声明](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/ui-user-questions/src/client/draft-store.ts#L1-L57)、[key生成](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/ui-user-questions/src/client/contract/slots.ts#L109-L154)。

官方tool-ask-user execute await userQuestions.ask，成功后返回结构化answers，render为工具结果文本。标准AgentLoop的assistant tool-call与tool/call可保存已发起问题的参数；成功tool/result保存已提交答案，失败result可保存错误。存储中“问过什么/已提交什么结果”不是“Promise仍pending并可重绑定UI”的证明，更不包括UI未提交草稿。
[tool await与result](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/interaction/tool-ask-user/src/index.ts#L78-L99)。
崩溃恢复遵循标准Session repair：已tool/call但缺result可补TOOL_OUTCOME_UNKNOWN，未started可补TOOL_NOT_STARTED；这是历史平衡/未知结果分类，不是重建旧ask waiter或自动恢复人类答案。不能从raw参数可读推出默认UI会重新打开同一问题。
[repair分类](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/session/src/repair.ts#L105-L124)。

因此已证实的是live Host wire重投、明确的delegate/abort/result生命周期以及提交后标准历史；完整页面重载竞态与Host重启端到端行为未实测，保留UNKNOWN。没有用假设补成跨重启问答持久化，也未设计项目补丁。
依据：0.1.2-rc.1 / a66e4702047846cdaa10c66c9d3df3951f5ea70d，关键类型/实现/一方测试均为以上固定SHA链接。
