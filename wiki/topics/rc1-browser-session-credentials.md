---
title: rc.1 官方浏览器 Session 与 CredentialProvider 公共边界
description: 固定版本的公开 apply、Session 行为、交互与凭据写入通知接缝。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-16
updated: 2026-09-16
asked_by: agent
anchors:
  - packages/api/session-controller/src/client/index.ts#apply
  - packages/api/session-controller/src/client/contract/sessions.ts#ISessions
  - packages/api/session-controller/src/client/contract/session.ts#ISession
  - packages/api/session-controller/src/commands.ts#SessionCommandController
  - packages/api/session-controller/src/agent.ts#ApiSessionAgentController
  - packages/client/connection/src/client/index.ts#ConnectionHandle
  - packages/client/ui-user-questions/src/client/index.ts#apply
  - packages/client/ui-approval/src/client/index.ts#apply
  - packages/credentials/credentials/src/index.ts#CredentialProvider
  - packages/credentials/credentials/src/types.ts
---

# rc.1 官方浏览器 Session 与 CredentialProvider 公共边界

条件：TS 浏览器 Client · 完整 CLI/Profile Host · 官方 Remote/Session 模块与注入依赖已装配 · 原生外壳仅监督进程 · 文件系统/工具保持 Host 所有 · 凭据在 Host provider · 本地部署 · 固定 0.1.2-rc.1 与本页 SHA。只读源码核验，未做完整启动、重连或真实模型实测。

## Session 公共面

`dsh-api-session-controller/client` 的运行时 `apply(ctx)` 创建内部 ClientSessions 并提供 `ctx.sessions`。`ISessions`、`ISession`、`SessionFace`、`Session` 是 type exports；ClientSessions/SessionManager 不是该 barrel 的 runtime exports。不能把内部源码中 export class 等同于正式包入口可导入构造函数。

ISessions 提供 `create(opts?)`、`open(id)`、`refresh()`、`scope(id)`、`sessionOf(ctx)`、`binding(id)` 等。create 返回 Promise<SessionId>；open 返回 void，仅选择已知 Session。ISession 提供 `prompt(content, mode, signal?, requestId?)`、`cancel()`、`beginSubmission(input)`、`command(line)` 等；SessionFace 另有 getSnapshot/subscribe，可观察 openState。prompt 的 accepted 是 admission，不是完成。

完整官方 Host/Client 分层组合可直接使用上述 face；没有一体式 Node 函数，不构成要求额外 Node Session adapter 的依据。该判断不认证任意具体 Profile 的启动结果。

`ISession` 没有 resume 方法。打开历史走 SessionEventStream 冷读；需要 Agent 的 prompt 等操作由 Host resolveAgent 去重并调用 ctx.agents.resume。历史打开、恢复 Agent 与跨进程恢复旧 Promise 是不同承诺。

普通 cancel 走 remote.session.cancel，再到 agent.cancel(user, keepInbox:true)；返回仅确认取消请求，非物理 effect drain。子 Agent 地址走 interruptByParent。prompt 的 signal 只约束 admission round-trip，不能代替 turn cancel。

## 问答、审批和重连

官方 ui-user-questions/client 与 ui-approval/client 的 apply 分别监听 remote 的 user-questions/request、approval/request waterfall。实际 pending carrier 的 `answer(QuestionAnswer)` 或 `answer('allowed-once'|'rejected')` 完成返答。PendingQuestion.cancel 是取消问题，不是取消 turn；PendingQuestion/PendingApproval 从各自 /client 仅 type-export，不应外部 new。

Connection /client 的 apply 提供 ConnectionHandle：`reconnect()`、generation/state 的 getSnapshot/subscribe 公开。API Gateway 唯一拥有 connection loop；不要另调 start 创建第二个 owner。Session apply 建 control stream 并监听 connection/reset；历史流另管 resume/baseline。连接重建不承诺恢复旧 Host 内存 pending Promise。

## 凭据公共面

`dsh-credentials` root 运行时导出 abstract CredentialProvider（兼 default）与 credentialRef/credentialKey 等函数；/types 是浏览器安全类型和事件声明。

- reference：`resolve`、`describe`、`set(ref,value)`、`unset(ref)`。每 operation 重解引用；旋转使用 set，没有独立 rotate。只读层覆盖时拒绝写入；空值使用 unset。
- record：`readRecord`、`describeRecord`、`listRecords`、`modifyRecord(key,async current=>replacement|undefined)`、`deleteRecord`。modifyRecord 返回 undefined 表示不改，不是删除；跨进程互斥依 backing store 支持。GrantRecord.payload 归 owner 解释，不自动刷新。
- 事件：`credentials/reference-updated(ref)` 与 `credentials/record-updated(key)`。Provider 子类提交后用 protected notifyUpdated/notifyRecordUpdated；consumer 用 ctx.on，不直接调用 protected 通知方法。环境变量变化不可观察，不发事件。通知不能等同事务回滚或通用跨进程广播。

## 定位证据

锚点全部按本页 SHA 用 git show 读取；本地源仓库为 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness`。关键直接证据：

- [Session Client exports/apply](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/client/index.ts)；固定版 apply 第 89 行。
- [ISession](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/client/contract/session.ts)；固定版 prompt 83、cancel 109。
- [CredentialProvider](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/credentials/credentials/src/index.ts) 与 [事件类型](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/credentials/credentials/src/types.ts)。
