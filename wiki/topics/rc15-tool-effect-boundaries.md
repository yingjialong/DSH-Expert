---
title: 1.5-rc.2 工具执行与外部效果边界
description: 区分 execution、审批、Shell 与文件操作结果以及日志耐久。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-21
updated: 2026-09-22
asked_by: agent
anchors:
  - packages/interaction/user-questions/src/index.ts
  - packages/interaction/user-questions/src/types.ts
  - packages/interaction/tool-ask-user/src/index.ts
  - packages/client/ui-user-questions/src/client/index.ts
  - packages/client/ui-user-questions/src/client/contract/slots.ts
  - packages/api/gateway/src/stream-protocol.ts
  - packages/api/remotes/src/index.ts
  - packages/api/remotes/src/remote-events.ts
  - packages/api/gateway/src/index.ts
  - packages/fs/tool-fs/src/index.ts
  - vendor/cordis/src/context.ts
  - vendor/cordis/src/fiber.ts
  - packages/core/scope/src/store.ts
  - packages/core/agent/src/index.ts#AgentRegistry
  - packages/core/agent-loop/src/index.ts
  - packages/core/agent-loop/src/tool-calls.ts
  - packages/core/tools/src/index.ts#ToolRuntime
  - packages/core/tools/src/ptc.ts
  - packages/core/scope/src/index.ts#scopeOf
  - packages/interaction/user-approval/src/index.ts#ApprovalService
  - packages/shell/tool-bash/src/index.ts
  - packages/shell/bash-local/src/index.ts
  - packages/shell/shell/src/types.ts
  - packages/fs/tool-fs/src/read.ts
  - packages/fs/tool-fs/src/write.ts
  - packages/fs/tool-fs/src/edit.ts
  - packages/fs/tool-fs/src/error.ts
  - packages/fs/fs-observation-policy/src/index.ts
  - packages/fs/fs-local/src/fsio.ts#writeFileAtomic
  - packages/fs/fs-local/src/index.ts
  - packages/fs/fs/src/types.ts
  - vendor/cordis/src/events.ts
  - vendor/cordis/src/reflect.ts#notify
related:
  - "[[wiki/topics/rc15-publication-budget-live-routing]]"
  - "[[wiki/topics/rc15-tool-occurrence-stream-cancel]]"
---

固定 0.1.5-rc.2；远端 tag 与上述 SHA 一致。静态源码与 fixture 核验；八个相关 npm 包 sha512/exports 核对，不代表安装或运行通过。完整答复先回传成功，再沉淀。

条件：TS 同进程宿主 · 官方 AgentLoop/ToolRuntime · Agent scope 与并行工具 · provider 管理的文件/进程 · 标准工具或 PTC · 未调用模型 · 平台具体持久性未实测 · 固定本页 SHA。

## 身份与执行顺序

agents.create 返回 AgentHandle，其 agent 是实际实例；agentLoop.create 返回 Agent。官方 publish 先进入两个 registry，再 session/created、agent/created、agent/session-start；created 同步异常可 veto，异步 rejection 不成为准入屏障。scopeOf(agent.ctx) 是官方 Agent 对象，非字符串 id/token。

标准模型路径：tool/call → execution/token → pre → 按需 approval → 同步 guards → execute wrappers/body → post → finalizeContent/materialization → tools/result → Session tool/result。提前拒绝、materialization 失败、pipeline throw 分支可跳过阶段；synthetic skipped result 可没有真实 execution。direct execute 不自动写模型日志对；tools/result 不 await listener Promise。

每个 execution 新 token；rootCallId 默认 callId，PTC 子调用带 parent=外层 token、独立 subCallId、沿用 rootCallId。wrapper 正常只改 signal；进入 body 前融合原 caller signal，结束恢复 wrapper signal。身份是 readonly 合同，execution 直到 result 通知才整体 Object.freeze；不能称全过程具备不可篡改的 JS 安全隔离。

ABORTED_BEFORE_DISPATCH 表示核心未调用 body；started Promise 仍需 drain，取消覆盖成功时为 ABORTED，已有工具错误可保留。二者都不证明 hook 副作用被撤销。TOOL_OUTCOME_UNKNOWN 是恢复时已记录 call 而缺 durable result，不是 provider 已失败回执。

await next 的调用链包含 body 与其 await 的 provider；异步动态上下文的覆盖是以其正确实现为条件的结论，不是 DSH 内建 token-local 存储。不会自动覆盖外层 pre/approval、后续 post/log、worker/RPC 或 detached job 的全部寿命。scope/request 未在该 SHA 的 packages/vendor 检出；不能自造 producer。

## 审批与 provider 关联

ApprovalService.request 要求 open turn；asked → decide → decided。缺答者、答者抛错或词汇外结果为 unavailable；never 为 rejected；只有 allowed-once 授权。多个答者是 waterfall 委派/短路，不是并行投票。abort 可先返回 cancelled，晚答不再增加 decided，但不强制终止答者自身工作。审计 append 失败仍拒绝，append 不等于 flush。

ToolRuntime pre ask 在 execute wrapper 之外；bash/fs body 内的 sandbox escalation 在其中。ShellExecRequest/Spec、FileSystem provider 参数和 ApprovalRequest 均无 execution token；approval 有 agent/callId，fs intent/observed 工具层事件则显式带 exec actor。不存在必须用某一种动态作用域关联的上游强制合同。

## Shell 与文件结果

官方 bash foreground：参数/policy/按需审批 → cwd/env → resolve → await run → aborted 转工具错误，否则返回 DTO。非零 exit 仍可成功取得工具值；isError 不是 exitCode 判定。Local run 等 subprocess.done，timeout/abort 分类不证明外部副作用回滚。

background：预检及取消检查 → jobs.start starter 中 resolve/start → 立即 jobId。官方 starter 不传 exec.signal，接纳后由 jobs 拥有 detached work；cancel 调 kill，done 单独等进程。Local kill 先标 killed 并 terminate，boolean 不代表已退出；done 不 reject，provider failure 可呈 killed/stderr。后台 ack、任务 completed 或进程结束均不证明任意业务外部效果耐久。

read：resolve/stat → readText 或 streamText → 窗口 → observed(stat.version)。没有读取后复查，不能称内容/版本来自单一原子快照。

write/edit：policy/审批 → resolve → intent waterfall → writeText/editText → observed(outcome.version)。工具层不先 stat。observation-policy 按 Session 对象/targetKey 存观察：write present→replaceIfVersion，否则 createIfAbsent；edit unseen→FS_NOT_OBSERVED，absent→FS_NOT_FOUND，present→version。无策略时 intent undefined 是无条件操作。

Local provider 的 per-target lock 不约束所有外部进程。createIfAbsent 有最终 no-replace publication；版本检查到 rename 之间仍有工作，不扩张成通用跨进程 CAS。临时文件写入/sync/close 后检查取消，再 link/rename/Windows replacement；未由此证明目录掉电耐久。后续 probe 失败不必然表示未写入；写后被删除有 missing sentinel。FsWriteOutcome/FsEditOutcome 没有 durability/unknown 字段，before=null 也不只表示新建。

FsError extends HarnessError，ToolRuntime 保留 error.info.name/code；普通错误不自动结构化。remediateFsError 只重写 NOT_OBSERVED/STALE_VERSION 提示，sandbox mapping 保留 FS_SANDBOX_DENIED；失败结果不能通用推断物理状态未改变。

## producer 注释陷阱与证据

tools/change 有 ScopedLayers producer；tools.guard 是 API 而非 tools/guard 事件。Cordis internal/service 声明注释称 no core producer，但 reflect.notify 实际 emit；按源码裁决，见 E049。该事件是 binding 通知，不是每次 service 方法调用。

本地根路径已核对为 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness`。关键证据：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/tools/src/index.ts`：createExecution、dispatchToolBody、notifyResult。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/tool-calls.ts`：startCall/commitReady。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/interaction/user-approval/src/index.ts`：request/decide。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/shell/bash-local/src/index.ts`：runArgv/startArgv。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/fs/fs-local/src/fsio.ts`：writeFileAtomic。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/cordis/src/reflect.ts:333`：internal/service producer。

Fixture 仅读未运行：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/tools/tests/tools.spec.ts`：1072 顺序、1480 signal、1585 pre-abort。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/interaction/user-approval/tests/approval.spec.ts`：180 多答者、289 late答。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/shell/tool-bash/tests/tools.spec.ts`：680 审批后取消。

未对任何外部宿主作运行验收。


## 2026-09-22：ToolFS 的 attachments 隔离

固定本页 SHA；asked_by: agent；完整回传成功后补充。ToolFS/Cordis 两正式包 sha512/exports 核验，未运行组合测试。

ToolFS root 必需 tools/fs/systemPrompt；read_image 仅在内部 inject(['attachments']) callback 注册，read/write/edit在外层注册。Config只有四个read限额，无禁用image开关；独立applyReadTool等不在root公开出口，正式包无src。

在新 attachments isolation label 没有provider、其他依赖满足且没有同layer重名冲突时，isolate('attachments')后挂完整ToolFS模块使conditional子fiber不激活，而其他工具仍进入继承ToolRuntime。这是公开语义+源码推论，非精确组合实测。isolate只覆盖一个service label，_getImpl不回退父label；以后该label提供attachments可激活read_image。它不卸载原provider、不删除别处已有read_image，不是全局安全限制。

Cordis realm隔离不改变原DSH scope tag；global/Agent/preset注册层沿用原scope。服务trace视图不必===，审计应看tools label/implementation与目标fiber状态；registry.has只证明runtime有fiber，不证明目标ACTIVE。公开schemas(scope)看可见集，省略scope是global。confining fs仍要求sandboxPolicy。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/fs/tool-fs/src/index.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/cordis/src/context.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/cordis/src/fiber.ts:597`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/scope/src/store.ts:226`。一方 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/fs/tool-fs/tests/read-image.spec.ts` registration surface覆盖attachments卸载/重挂时仅conditional工具变化；未覆盖本次精确隔离组合，本次仅读。


## 2026-09-22：审批 remote forwarding 与先行答者

固定本页SHA及Cordis4.0.2，asked_by: agent。api-remotes/Cordis正式tarball sha512/exports/声明核验，先回传成功，未运行组合。

api-remotes apply注册Gateway唯一source时立即调用source并安装固定allowlist listeners，不等Client连接。approval/request普通push listener先forward，result直接返回，next才委派后续Host；无Client或Client不答时pending没有默认超时。断线移除delivery不等于next；request abort/Agent Context释放/source终止可拒绝pending。多个Client复用同一Host listener，首result/rejected结算，next须剩余delivery为空；同Gateway重复source注册拒绝。

Cordis on公开prepend/global。后注册prepend用unshift，进入此前api-remotes之前，但只影响新dispatch；已经等待中的waterfall已有callback快照。多个prepend后装者更前；不同fiber不另建顺序，隔离events则另当别论。global跳过Context.filter（含Agent/base过滤），不是只提高优先级；原carrier及req.agent不变，必须明确处理归属。普通untagged root listener本身已可接收DSH scope允许的请求。

官方ApprovalService仍拥有open-turn检查、asked→decide→decided。never与预先abort在waterfall之前，prepend不绕过。合法答者直接return outcome短路，其它next；allowed-once仅本请求，非持久许可/执行完成证明。未找到api-remotes单独exclude approval配置；导出allowlist常量不是可变配置合同。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/remotes/src/index.ts`（remoteEventSource/forwardWaterfall）、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/gateway/src/index.ts:238`（source注册）、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/cordis/src/events.ts`（dispatch/register）。fixture `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/gateway/tests/gateway-stream.host.spec.ts:705` 与 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/interaction/user-approval/tests/approval.spec.ts:422` 仅读未运行。


## 2026-09-22：Question schema 与 wire 公开边界

asked_by: agent；主答与缩小范围增量均完整回传成功后沉淀。固定本页SHA，三个Question正式包sha512/exports核验；未运行。

ask_user_question输入questions项为id/question/header?/options[{label,description?}]?/multi_select?，映射service multiSelect。无optionsOnly/allowCustom:false；generic UI允许custom和skip，答案selected/custom。Service另有detail与plan-review intent，默认工具不转发。service未做唯一id/选项成员/一题一答关系校验，不能把输出结构schema当这类约束。

Service.ask校验预abort、非空、指定Agent exact live root及intent；随后waterfall返回answer或next，无deny union。scope listener可throw/reject阻止，next是委派。工具pre/guard只涵盖工具路径，不能涵盖任意direct ask。ask本身没有通用abort race，官方Gateway/UI配合signal取消，忽略signal的自定义永不结算Promise不被强制结束。

官方Client包为dsh-client-ui-user-questions/client，runtime仅apply/inject，PendingQuestion为TYPE。apply真实接remote.$on并发布uiSession pending，conversation.composer selector渲染组件。carrier.answer/cancel是本地结算，不是Host收到/耐久确认；取消整批、多题一次answers。无专门持久question asked/answered producer；tool-call/result是另一层。exec.agent传入request，token/callId/turn/prompt requestId不传；各层id不得混同。

逻辑stream准确名$events而非$events/stream，payload={args:{}}；ready有clientId/host，waterfall有event/eventId/agentId/request，cancel有eventId。结果$events/result的args含clientId/eventId/outcome。api-remotes核对Agent并提供context，Gateway私有startRemoteEvent分配eventId，openRemoteEvents分配clientId；request.agent/signal由wire投影移出，Client重建scope。

这些wire frame/ID/parser在内部stream-protocol文件，root仅明确公开RemoteEventHostInfo；公开TypertRemoteEventInvocation有原request/context/resolve/reject但没有eventId。未找到分配后关联eventId与原Host request的公开hook/query；wire同时含agentId和eventId不等于原对象关联API。Client remote.$on也不交eventId。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/interaction/user-questions/src/index.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/interaction/tool-ask-user/src/index.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-user-questions/src/client/contract/slots.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/gateway/src/stream-protocol.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/gateway/src/types.ts`。browser-plugin.client.spec.ts与user-questions-composer.client.spec.tsx位于上述ui-user-questions包tests，仅读未运行。
