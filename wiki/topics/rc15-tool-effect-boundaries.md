---
title: 1.5-rc.2 工具执行与外部效果边界
description: 区分 execution、审批、Shell 与文件操作结果以及日志耐久。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-21
updated: 2026-09-21
asked_by: agent
anchors:
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
