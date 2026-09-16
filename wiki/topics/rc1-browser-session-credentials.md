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

### Host occurrence 与 awaited hook 边界

固定rc.1公开事件为agent/inbox/claimed({agent,message,turn})而非session/input-claimed；它是contained emit观察。Agent.cancel(cause,{keepInbox?})无expected-turn/input/seq CAS，whenIdle跟随whole-agent activity而非某个MessageId。旧观察跨await后再cancel不能自动绑定旧turn。

agent/pre-step是awaited waterfall，可reject拟议step；agent/request修改调用配置；agent/request-error处理恢复；agent/turn-stopping是awaited serial停止边界。pre-step已经过claim和assembly，不能替代入箱授权。同步Host检查与立即决策仅在相应live上下文/无异步间隙条件下成立，不升级成跨进程原子取消合同。loop hook内await自己的whenIdle可能自等待。

steer/inject/followup/Inbox存在多个入口；Controller/Remote检查不覆盖in-process。没有核验到统一pre-steer awaited veto；inbox观察/Session commit事件不能veto既定入箱。问题PendingQuestion有sessionId与本地question:N key，type-only公开；draft非持久，连接代重投不等于reload/Host重启恢复草稿。

证据：本页固定SHA的 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent/src/runtime-types.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/agent.ts`；问答详见 [rc.1生命周期专题](rc1-question-delivery-lifecycle.md)。未跑并发或浏览器故障实测。

### 零 provider 的默认模型标签

base composition独立声明agent-default-model为deepseek-official/deepseek-v4-flash；它只是选择配置。Host buildModelCatalog在providers为空时仍返回default，routableProviders/groups为空；Client current取projection.next或default，routable=false使composer blocked，ModelSelect没有catalog匹配时显示provider/model字符串。标签不注册provider、不加载adapter、不读模型credential、不发上游生成；但Client确有session.modelCatalog的Host读RPC，不能表述为零网络。历史选择或配置改变可改变标签。

证据：固定本页SHA的 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/catalog.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-model-selection/src/client/directory.ts` 与 `ModelSelect.tsx`；base/cordis.patch.yml的agent-default-model条目。只读核验，不由标签判定整个Profile的其他插件行为。

`beginSubmission({mode,text,images,onRetire?})` 同步返回 `{requestId, abandon()}`，不是 RemoteResult。它登记本地 echo，observed 退场对应 durable user/message 或 queue occurrence，不是执行完成；abandon 不是取消 Host turn。prompt/cancel 成功形状为 `{ok:true,value:{accepted:true}}`，失败为 `{ok:false,error}`。

Connection 的 apply 内 `generationId=0`，onConnected 使用 `++generationId`（固定 SHA 的 connection/src/client/index.ts:192、268）。因此 generation.id 是 Client 实例局部计数，不是持久 Host epoch、跨 Client 身份或授权 CAS；采样不锁住后续 RPC 连接代。

`dsh-api-session-controller/client` 的运行时 `apply(ctx)` 创建内部 ClientSessions 并提供 `ctx.sessions`。`ISessions`、`ISession`、`SessionFace`、`Session` 是 type exports；ClientSessions/SessionManager 不是该 barrel 的 runtime exports。不能把内部源码中 export class 等同于正式包入口可导入构造函数。

ISessions 提供 `create(opts?)`、`open(id)`、`refresh()`、`scope(id)`、`sessionOf(ctx)`、`binding(id)` 等。create 返回 Promise<SessionId>；open 返回 void，仅选择已知 Session。ISession 提供 `prompt(content, mode, signal?, requestId?)`、`cancel()`、`beginSubmission(input)`、`command(line)` 等；SessionFace 另有 getSnapshot/subscribe，可观察 openState。prompt 的 accepted 是 admission，不是完成。

完整官方 Host/Client 分层组合可直接使用上述 face；没有一体式 Node 函数，不构成要求额外 Node Session adapter 的依据。该判断不认证任意具体 Profile 的启动结果。

`ISession` 没有 resume 方法。打开历史走 SessionEventStream 冷读；需要 Agent 的 prompt 等操作由 Host resolveAgent 去重并调用 ctx.agents.resume。历史打开、恢复 Agent 与跨进程恢复旧 Promise 是不同承诺。

普通 cancel 走 remote.session.cancel，再到 agent.cancel(user, keepInbox:true)；返回仅确认取消请求，非物理 effect drain。子 Agent 地址走 interruptByParent。prompt 的 signal 只约束 admission round-trip，不能代替 turn cancel。

## 问答、审批和重连

### Session 可调用性与保留 binding（同 SHA 补充核验）

`OpenState = cold | loading | open | error` 仅表示历史窗口。Session.prompt/cancel 不检查 openState、removed、running、list.current 或 connection generation；prompt 明确支持导航前 first-send。因此不存在以 open/running/current 组合定义的统一 API 前置条件。Host 普通 cancel 仅查 live Agent，未 attached 返回 session/not-found；prompt 则可按需 resume。

仅切换 current、原 Session 仍列出且 Client 生命周期有效时，public binding(id).session 可继续 cancel 原 Session：方法使用自身 sessionId/address，不取 current。旧引用不是有效性 lease；scope prune/dispose 或 removed 后不能由引用仍存在推断可调用成功。普通 removed 来自 removal 事件，与断线或磁盘历史删除不同；durable subagent removal 特判为 running=false，并可保留 parent address。

Connection generation ready 只说明 source 已挂增量 listeners，不代表所有 Session 历史/control baseline 完成，更不锁定后续 RPC 的 Host epoch。断线时 generation 缺失，Session snapshot 可保留；冷历史、移除、断线三者不能互推。cancel ok 仅为请求接受，非 running=false 或物理停止；pending inbox 保留，后续仍可运行。

证据：固定 SHA 下 session-controller 的 `src/client/contract/snapshot.ts:55`、`src/client/sessions/session.ts:225/312/551`、`src/client/sessions/service.ts:503`、`src/client/sessions/manager.ts:732`、`src/commands.ts:434`；Connection `src/client/connection.ts:56` 和 `src/client/index.ts`。均位于上文确认的本地上游仓库路径；本次只读源码，未执行 transport 竞态实测。

官方 ui-user-questions/client 与 ui-approval/client 的 apply 分别监听 remote 的 user-questions/request、approval/request waterfall。实际 pending carrier 的 `answer(QuestionAnswer)` 或 `answer('allowed-once'|'rejected')` 完成返答。PendingQuestion.cancel 是取消问题，不是取消 turn；PendingQuestion/PendingApproval 从各自 /client 仅 type-export，不应外部 new。

Connection /client 的 apply 提供 ConnectionHandle：`reconnect()`、generation/state 的 getSnapshot/subscribe 公开。API Gateway 唯一拥有 connection loop；不要另调 start 创建第二个 owner。Session apply 建 control stream 并监听 connection/reset；历史流另管 resume/baseline。连接重建不承诺恢复旧 Host 内存 pending Promise。

## 凭据公共面

### 手工 pi-ai route 的显式 key 优先级

固定 rc.1 lock 的 pi-ai 为 0.84.2。真正未命中 catalog 的 provider id 使用 DSH harnessApiKeyAuth，仅消费传入 credential.key。streamWithSnapshot 先 await resolveApiKey，再调用 models.streamSimple；显式 ref miss/空值在 SDK auth 前 MISSING_CREDENTIAL，格式校验可 INVALID_CREDENTIAL。

pi-ai resolveProviderAuthWithSignal 在 overrides.apiKey 非 undefined 且 provider.auth.apiKey 存在时直接走该 resolver，跳过 CredentialStore.read 和 ambient 分支；手工 resolver 不读取 AuthContext.env/fileExists。Models 的构造、setProvider/getModel 不触发 auth。三协议 formatter 使用显式 key，不另找认证。

结论仅限这次生成认证：catalog 同名 route 的 resolver、checkAuth/getAvailable/login/refresh、缺省而非缺失的 apiKeyEnv 是其他路径；非认证环境读取（如 cache retention）仍可能存在。自定义 CredentialProvider.resolve 的内部读取也不属于 pi-ai fallback。

证据：本页 SHA 的 llm-pi-ai/src/provider.ts#harnessApiKeyAuth/routeAuth、src/index.ts#apply、src/adapter.ts#streamWithSnapshot；[官方 pi-ai 0.84.2 tarball](https://registry.npmjs.org/@earendil-works/pi-ai/-/pi-ai-0.84.2.tgz) 内 dist/models.js、dist/auth/resolve.js 及三协议 dist/api 文件。内存包 sha512 与固定 pnpm-lock 完全一致，未执行模型请求，verified_inference。

ApiKeyRecord 的 key-only/env-only/key+env/二者缺席均合法；二者缺席是 owner 确认 ambient auth，不等于无 record。env 为 provider 环境配置，不自动写 process.env。官方 credentials-local 校验存在的 key 非空、env 名 POSIX且值非空；空 env map 允许，HTTP key 合法性由 adapter 另验。GrantRecord.payload 必须可存 JSON，但内容归 owner 解释，不通用限定 OAuth token 或 BrowserAuth 格式。

llm-pi-ai 的公开 providers[route].apiKeyEnv 经 credentialRef 验证，Host 每 stream 调 ctx.credentials.resolve，显式引用 miss 不回退 ambient；无 credentials 服务时才读 launch environment，省略 ref 才允许 provider 自身发现。recordKeyFor(provider) 属于另一 record key 空间，不能当 apiKeyEnv。Host 消费位置不证明秘密永不经过单独的浏览器凭据写入 API，也不是 Host plugin 的隔离沙箱。

本段按同 SHA 的 credentials/src/types.ts、credentials-local/src/index.ts#parseRecord、llm-pi-ai/src/config.ts 与 src/index.ts#apply 核验；未调用真实模型或读取用户秘密。通知是提交后 void fan-out，普通 listener 异常被记录、同步 INVARIANT 可重抛，不构成所有订阅者完成屏障。

`dsh-credentials` root 运行时导出 abstract CredentialProvider（兼 default）与 credentialRef/credentialKey 等函数；/types 是浏览器安全类型和事件声明。

- reference：`resolve`、`describe`、`set(ref,value)`、`unset(ref)`。每 operation 重解引用；旋转使用 set，没有独立 rotate。只读层覆盖时拒绝写入；空值使用 unset。
- record：`readRecord`、`describeRecord`、`listRecords`、`modifyRecord(key,async current=>replacement|undefined)`、`deleteRecord`。modifyRecord 返回 undefined 表示不改，不是删除；跨进程互斥依 backing store 支持。GrantRecord.payload 归 owner 解释，不自动刷新。
- 事件：`credentials/reference-updated(ref)` 与 `credentials/record-updated(key)`。Provider 子类提交后用 protected notifyUpdated/notifyRecordUpdated；consumer 用 ctx.on，不直接调用 protected 通知方法。环境变量变化不可观察，不发事件。通知不能等同事务回滚或通用跨进程广播。

## 定位证据

锚点全部按本页 SHA 用 git show 读取；本地源仓库为 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness`。关键直接证据：

- [Session Client exports/apply](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/client/index.ts)；固定版 apply 第 89 行。
- [ISession](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/client/contract/session.ts)；固定版 prompt 83、cancel 109。
- [CredentialProvider](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/credentials/credentials/src/index.ts) 与 [事件类型](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/credentials/credentials/src/types.ts)。
