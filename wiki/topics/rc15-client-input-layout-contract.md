---
title: 1.5-rc.2 Client 输入交接与 Workspace/Layout 合同
description: 区分公开导航、输入状态接纳、默认 apply 贡献与 Renderer 挂载。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-21
updated: 2026-09-30
asked_by: agent
anchors:
  - packages/client/ui-conversation/src/client/input/editor/keymap.ts
  - packages/client/ui-conversation/src/client/input/submission-policy.ts
  - packages/client/ui-conversation/src/submission-settings.ts
  - packages/client/ui-conversation/src/client/skeleton/InputBar.tsx
  - packages/client/ui-conversation/src/client/skeleton/TodoPanel.tsx
  - packages/core/agent-loop/src/agent.ts
  - packages/client/ui-workspace/src/client/navigation.ts
  - packages/client/ui-workspace/src/client/index.ts
  - packages/api/session-controller/src/client/contract/sessions.ts
  - packages/client/ui-session/src/client/index.ts
  - packages/client/ui-conversation/src/client/contract/input.ts
  - packages/client/ui-conversation/src/client/input/facade.ts
  - packages/client/ui-conversation/src/client/input/machine.ts
  - packages/client/ui-conversation/src/client/skeleton/ConversationSession.tsx
  - packages/client/ui-renderer/src/client/index.ts
  - packages/client/ui-renderer/src/client/app.tsx
  - packages/client/ui-layout/src/client/index.ts
  - packages/client/ui-layout/src/client/service.ts
related:
  - "[[wiki/topics/rc15-domain-checkpoint-workspace-contract]]"
---

固定0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203。6个正式UI包sha512/类型/实际bundle核验，tag一致；未运行。完整回传咨询DSH-015-B4-CLIENT-INPUT-05后沉淀，L2 / verified_inference。

## Workspace

connectWorkspace必须返回真实可由Session Controller寻址的SessionId；默认blank查找/复用算法不是外部provider必须照抄的实现。openWorkspace默认beginNavigation并检查lifetime/superseded，在仍current时调用同步beforeOpen(id)，再检查才open；beforeOpen不await。superseded不回滚已创建Session。startSession返回void非完成回执；openSession只导航；archive不终止Agent，UI清选择由观察状态处理。

默认apply另安装navigation watcher、root workspaces hook、workspace locale、sidebar.workspaces/hero.workspace slots及各自directoryFlow子声明、store和回调。只提供uiWorkspace服务不会自动保留这些。公开slots/locale注册可消费；默认Browser/Picker/store工厂和UiWorkspaceService非/client runtime出口。

## 输入scope与setDraft

公开Client sessions.scope(id)/binding(id).ctx提供Session scope；scopeOf/sessionOf检查归属。未知且未已有scope的id可无binding；不要拿Host Agent.ctx或root替代。UiSession.provide的resolve(binding)也提供同一scope。Conversation用其提供input hooks/actions，并由resident shell管理输入；不需私有InputHub.shell。

SessionInput.setDraft(text):void，全量替换、去REFERENCE_PLACEHOLDER、按换行建paragraph、caret到末尾、Lexical discrete:true/HISTORY_MERGE_TAG；相同clean文本no-op。公开state.getSnapshot().draft为clipboard投影，内容变化draftRev递增；可subscribe确认，不把void当提交回执。

setDraft本身无忙期/disposed guard，UI read-only不代表程序调用拒绝；可覆盖文本，但不改已捕获提交attempt。claimed状态破坏token前缀会释放claim。生命周期结束后旧facade不再有效，返回不能证明状态接纳。无公开input.ready或awaitSetDraft票据。

默认ConversationSession负责seed并绑定draft mirror，editor更新回写store；mirror不是Session持久日志。离开该渲染路径不能假定mirror已绑定。submit是void，default optimistic清空并detached发送；phase plain/draft空不代表Host接受或模型完成。实际admission归Session.prompt/Conversation发送链，运行完成归Session事件。

## Renderer/Layout

uiRenderer.mount(container)渲染slots.root，返回React unmount disposer，不dispose全部Cordis，也不代表异步业务ready。Renderer.apply创建SlotRegistry并安装renderer。

默认Layout.apply额外创建store、panelInfo root hook、layout service、root AppFrame及sidebar/main/rightbar/shell.overlay子声明、main变更retain策略、theme DOM呈现。跳过apply这些不会自动存在。

LayoutController公开constructor(actions,hasMainPanel)、导航/几何方法，selectPanel校验main key，beginNavigation/dispose取消导航；不自动创建store/root/hook/theme。PanelActions引用内部store类型不等于store工厂公开。

官方renderer组合须真实root registration、子slot声明/render授权、所消费standard hooks和Session scope adapter。UiSession.apply提供root sessions/pendingInteraction及session adapter；不能以SlotMap类型声明替运行注册。不在本页选择外部root组件或roster。

## 证据

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/input/facade.ts`：setDraft/publish。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-workspace/src/client/navigation.ts`：导航。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-layout/src/client/index.ts`：默认副作用。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/tests/input-bar.client.spec.tsx:1083`：programmatic draft同步state/DOM断言，仅读未运行。

未验证外部provider或真实Layout/导航/提交组合；不据静态类型宣称运行验收。

## 2026-09-30：默认 Composer 与 Todo 展示边界

asked_by: agent；固定本页 SHA，远端 tag 一致，完整回传后沉淀。静态核验 L2 / verified_inference，fixtures 仅读未运行。

Composer 普通 Enter 先仲裁输入菜单，再判断重复键与 canSubmit；Shift+Enter 先于 IME guard 放行原生换行。普通 Enter 的 isComposing/keyCode229/composition watch（compositionend 后 10ms）阻止误提交，不 preventDefault。pasteText 接收 text/plain，多行作为文本插入而不提交；混合文件/文字分别 intakeFiles/pasteText。

Host user-settings 的 ui-conversation.busyEnter 为 queue|steer，默认 queue。Agent 运行且支持 steering 时，Enter/主 Send 采用偏好，Ctrl/Cmd+Enter 采用相反模式；其他情况 queue。提交状态机 adjudicating/submitting 的 machineBusy 不等于 Agent running。普通会话运行且草稿空/blocked 时主按钮为 Stop；continuable child 可有独立 Stop。上传未结束、locked 等约束仍影响发送。该设置不承诺公开任意 keymap 重映射。

TodoDock 在 conversation.input.dock 注册 todo/order=0；todos 为 null/空数组时不显示，TodoPanel 初始折叠，展开遍历全部条目，无最多 3 条限制。Question/Todo 的默认实现不能推导永久固定面板或固定像素布局合同。

ISessions.open(id):void 是导航接口，不能误写为 SessionFace.open。Client 历史窗口与 Host 模型历史分离；Agent 的最终内部 request.messages 在 agent.ts:603 来自 session.deriveMessages()，历史行可见不能单独证明 provider wire 中有哪些消息。

新增关键证据（均按固定 SHA 读取）：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/input/editor/keymap.ts`：registerComposerKeymap。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/input/submission-policy.ts`：resolveSubmitMode。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/submission-settings.ts`：ConversationSettingsSchema。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/skeleton/InputBar.tsx`：machineBusy/primaryStops。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/skeleton/TodoPanel.tsx:88`：collapsed/empty；:123 为 TodoDock。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/agent.ts:603`：模型请求 messages 边界。
