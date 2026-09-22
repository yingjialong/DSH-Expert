---
title: 1.5-rc.2 Chat 跟随与分页边界
description: 定位滚动归属、采样与动态高度路径，区分公开展示动作和内部 helper。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-21
updated: 2026-09-22
asked_by: agent
anchors:
  - packages/client/ui-tool/package.json
  - packages/client/ui-tool/src/client/apply.ts
  - packages/client/ui-tool/src/client/tool/ToolCallTree.tsx
  - packages/client/ui-conversation/src/client/conversation/assembler.ts
  - packages/client/ui-conversation/src/client/conversation/event-registry.ts
  - packages/client/ui-conversation/src/client/conversation/view-registry.ts
  - packages/client/ui-chat/src/client/conversation-nodes/chat-snapshot-builder.ts
  - packages/client/ui-chat/src/client/conversation-nodes/assistant.ts
  - packages/client/ui-chat/src/client/chat/TurnTailNodeView.tsx
  - packages/client/ui-chat/src/client/conversation-nodes/turn-tail.ts
  - packages/client/ui-primitives/src/markdown/CodeBlock.tsx
  - packages/client/ui-primitives/src/markdown/MarkdownText.tsx
  - packages/client/ui-primitives/src/markdown/render.tsx
  - packages/client/ui-chat/src/client/chat/ChatView.tsx
  - packages/client/ui-chat/src/client/chat/ChatView.module.css
  - packages/client/ui-chat/src/client/chat/ChatNodeSeat.tsx
  - packages/client/ui-chat/src/client/chat/ReasoningRow.tsx
  - packages/client/ui-chat/src/client/chat/ReasoningRow.module.css
  - packages/client/ui-chat/src/client/chat/searchable-hidden.ts
  - packages/client/ui-chat/src/client/contract/slots.ts
  - packages/client/ui-chat/src/client/apply.ts
  - packages/client/ui-chat/src/client/index.ts
  - packages/client/ui-chat/src/chat-settings.ts
  - packages/client/ui-conversation/src/client/skeleton/ConversationRoot.tsx
  - packages/client/ui-primitives/src/DisclosureRow.tsx
  - packages/api/session-controller/src/client/contract/session.ts
  - packages/client/ui-chat/tests/chat-view.client.spec.tsx
  - apps/web/tests/chat-scroll-contract.e2e.ts
related:
  - "[[wiki/topics/rc15-client-input-layout-contract]]"
  - "[[wiki/topics/rc15-composer-context-react-boundaries]]"
---

固定目标 tag 与本页 SHA 一致。ui-chat/ui-conversation/ui-slots/ui-primitives 四个正式包 sha512/exports/types核验；ui-chat bundle核对500ms、scrollend及<=25。完整答复回传成功后沉淀；未运行fixture或浏览器，不含调用方诊断。

条件：React Client · 默认Chat/Conversation · per-Session视图状态 · 浏览器DOM/ResizeObserver · 官方slots/分页 · 未调用模型 · 浏览器调度未实测 · 固定本页版本。

## 跟随归属

ChatView 内部 scrollerOf 选最近 data-conversation-scroll ancestor，无则 local list。默认ConversationRoot提供该外层及composer seat；CSS使内层overflow visible。TurnNavigator的窄目录滚动不是正文owner。

atBottomRef保存归属，observedTopRef记已处理或程序写入位置。readerMoved判定 abs(top-min(observedTop,floor))>0.5；源码阈值24+1，发布物折叠成<=25。首次open从chatScroll恢复语义位置或贴底。

scroll先pending=true；pinned且没有readerMoved的程序送达/shrink clamp立即sample，其余至多安排一个500ms timer，不在每个事件重置。scrollend提前sample；sample清pending/timer，按当时几何更新归属、保存位置，再触发render。非永久定时轮询。重新进入25px可恢复归属但不立即吸完剩余距离；历史上到过底部不证明以后永远pinned。

ChatView layout effect在pending时跳过；处理首次open、prepend anchor补偿，再依末尾user/steering/submission或followSig变化且pinned跟随。followSig不含每个文本delta。节点独立订阅导致行内更新无需改变ChatView的order；动态高度另由ResizeObserver观察flow column与composer，先followRef再更新active turn。followRef同样跳过pending，只在pinned时写scrollTop。没有ResizeObserver则该动态高度路径不安装。

这些是代码内的偏序，不构成stream/React commit/scroll delivery/resize之间全局固定顺序。listeners与observer在effect([])安装，没有typed live host rebinding入口。

内部toBottom除写DOM还清paging/jump、设置pinned、更新账本、清保存位置。chatScroll.save(null)仅内存保存协议，不是对已挂载组件的滚动命令。tools、业务代码不能从一个DOM位置推断这些内部状态已同步。

## 展开与公开出口

Turn process由ChatNodeSeat按compact模式、closed turn、完整history及answer generation控制；TurnProcessNodeView调用owner.turnProcess.setOpen。公开TurnProcessOwnerProps有setOpen；Chat store的setTurnProcessOpen按turn/answerStep记展开。createChatStore非/client runtime出口。

单条ReasoningRow内部expanded初值false，DisclosureRow受控渲染。running摘要取latestLine，结束取firstLine；data-follow-end是横向摘要尾端CSS，不是独立垂直跟随器。thinkBody自然增长，无专用max-height参数。变化后的正文跟随仍归ChatView。hidden until-found及beforematch/focus reveal保持稳定子树。

公开ui-chat.transcriptView=normal/compact（默认compact）控制完成Turn展示，不是follow开关。公开DisclosureRow提供open/onToggle/children/className，但不能访问默认ReasoningRow内部state。useAnchoredMaxHeight是popup overlay hook，不是过程区设置。

ChatView/ReasoningRow/toBottom/scrollerOf未公开runtime出口；/client公开ChatViewInjected等TYPE，chatScroll.read/save是注册方注入face。没有找到跟随阈值/采样周期/scrollToBottom/typed scrollHost/过程高度专用配置或动作。data-conversation-scroll是实现DOM约定，不能冒充typed provider。替换conversation.view或chat.node是组件组合，不自动继承内部状态机。

## 分页

默认hasMore显示Load earlier按钮，loadingOlder时禁用；点击先捕获语义anchor，再调注入loadOlder。scroll handler无触顶加载逻辑。未加载Turn导航点击则loadThrough(seq)，页到达后由组件补偿/落点，因此也不能说历史只由按钮加载。

公开SessionFace.loadOlder():Promise<void>、loadThrough(seq):Promise<void>及IConversation.loadOlder可组合；后者loadThrough完成包括covered/exhausted/superseded/failed soft，不是目标必存在保证。公开动作不自带默认ChatView的DOM anchor和follow状态。

## 证据与未运行fixture

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-chat/src/client/chat/ChatView.tsx`：scrollerOf/toBottom/layout/scroll/observer/loadOlderAnchored。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-chat/src/client/contract/slots.ts`：TurnProcessOwnerProps/ChatViewInjected。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/client/contract/session.ts:120`：分页结果合同。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-chat/tests/chat-view.client.spec.tsx`：2328暂停恢复；2371 clamp/regrow；2429小手势累积；2482采样；2544resize；2626阈值不吸底；2639外层host；1690 partial history。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/apps/web/tests/chat-scroll-contract.e2e.ts`：509并发prepend/stream；664长scroll-away；755切换/resize；857keyboard；895fling。场景skipIf(record)，本次均未运行。


## 2026-09-22：流内席位与代码块

asked_by: agent；完整回传成功后补充，固定本页SHA，ui-chat/ui-primitives正式tarball重新核对，未运行。

conversation.chat.turnTail是实际data-chat-flow内的session chain，TurnTailNodeView在MessageIconActions之前renderSlotChain；仅turn/end后definition才产生尾节点。owner为turn/seq/openFile，closing=null仍可render tail。它不是任意时刻或执行中占位入口，chain择一不是list累加；ui-deliverables已有注册。assistant-actions是同尾节点的动作条，只有closing durable messageId才render，owner仅messageId，不是fence toolbar。

chat.node是已产生node的keyed renderer；注册新key不自动生成数据节点。commandview/images有实际producer但限定命令/图片。input dock/composer dock属于sticky composer附近，overlay/left/right/header不等于正文流。session slot的inject(sessionId)及公开sessions.binding/scope可取得Client作用域；owner无scrollHost/clipRect/ref，data-conversation-scroll仅实现DOM约定。

assistant text→MarkdownText→内部renderCode→CodeBlock。公开MarkdownText无components/renderCode/actions参数；公开CodeBlock只有code/lang/streaming/className/contentRef/lineNumbers/复制labels，无按钮children/toolbar slot。contentRef仅服务直接创建该primitive的owner，不能访问默认assistant内部实例。内部固定Copy按钮，code来自mdast node.value加合成尾换行；空fence走pre、settled math走TeX。工具结果toolview不是assistant文本fence扩展。未找到满足“保留默认Markdown并加按钮读取代码”的公开细粒度入口。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-chat/src/client/chat/TurnTailNodeView.tsx:26`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-chat/src/client/conversation-nodes/turn-tail.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-primitives/src/markdown/MarkdownText.tsx:167`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-primitives/src/markdown/CodeBlock.tsx:12`。无外部项目诊断或实现方案。


## 2026-09-22：独立节点与 messageId 读取

asked_by: agent；先完整回传，固定源码交叉核验，未运行精确组合。events.register(ConversationNodeDefinition)按kind唯一；assembler遍历所有普通definition，单个真实tool/call可同时命中默认tool及自有kind。context按(kind,id)隔离，每context唯一start，返回node.key须等于context.key且target匹配。自有target=chat节点满足ChatConversationViewNode的anchorSeq/location/visibility/data合同后可由默认ChatSnapshotBuilder接纳，无需再注册chat view builder；views.register按target唯一，重复chat冲突。renderer另经conversation.chat.node新key注册，单有renderer不创建节点。

默认排序为presentation anchor/rank/originalAnchor/key，含Turn process调整，不按注册顺序保证邻接。自有kind不自动拥有内置legacy/navigation专用语义，也可能被compact过程规则折叠。只投影call展示不要求复制官方工具生命周期，但仍须定义自身最小match/start/update/buildViewNode。

公开UiConversation.binding(id).target('chat')提供getSnapshot/subscribe，订阅激活target；ChatSnapshot.nodes.values含已materialize visible/hidden节点。assistant-step的data.finalNode.messageId对应真实assistant/message id，finalNode.blocks由message.content投影，text块取text，reasoning等另分。没有专门getByMessageId，nodes.get的key也不是messageId。SessionBinding.eventSource.getSnapshot().entries另保留当前连续窗口真实assistant/message及content，定位为assembly feed，不是可写或全量数据库。缺窗口节点不证明Host缺消息，partial/中断冻结节点可无durable id。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/conversation/assembler.ts:463`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-chat/src/client/conversation-nodes/chat-snapshot-builder.ts:400`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-chat/src/client/conversation-nodes/assistant.ts:193`。可组合性未作为运行验收。


## 2026-09-22：tool-call renderer 归属

asked_by: agent；先回传成功，ui-tool正式0.1.5-rc.2 tarball sha512/manifest/types/factory核对，未运行。

实际注册conversation.chat.node key=tool-call的是@deepseek-ai/dsh-client-ui-tool/client，factory id为包根名；runtime只export apply/inject。manifest模块inject为api-workspace-controller、client-connection、client-locale、client-ui-conversation；实际factory require react/jsx-runtime、react、ui-primitives。Cordis服务inject为slots/remote，使用remote.$host及connection/reset，locale用conversation。模块加载不等于apply。

ui-chat提供tool数据definition及ChatNodeSeat producer，缺ui-tool且无同key替代时visible工具节点走Unknown surface JsonBlock。ui-tool注册ToolCallTree并声明tool.call.toolview，递归root/subCalls按toolName dispatch，未命中专用row走GenericToolCard。这是两层不同fallback。compact隐藏或renderer崩溃不能直接套“缺模块”结论。

ui-tool apply自行激活bash/read/read_image/write/edit/grep/glob/web_search/web_fetch/todo_write/ask_user_question内建views，无需另导入私有row。ui-attachment图像gallery、ui-deliverables present是额外贡献；Turn process仍由ui-chat拥有。官方web-app配置确有ui-tool。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-tool/src/client/apply.ts:33`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-tool/src/client/tool/ToolCallTree.tsx:39`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/bundle/web-app/cordis.patch.yml:267`。fixture `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-tool/tests/assembly-surfaces.client.spec.tsx:142` 仅读未运行。
