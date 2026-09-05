---
title: rc.2 SessionFace 寻址、创建耐久性与 blank preset 切换边界
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/client/runtime/src/client/index.ts#SessionFace
  - packages/client/runtime/src/client/contract/session.ts#ISession
  - packages/client/runtime/src/client/contract/sessions.ts#ISessions
  - packages/client/runtime/src/client/sessions/service.ts#SessionRuntime
  - packages/client/runtime/src/client/sessions/manager.ts#SessionManager.get
  - packages/client/runtime/src/client/sessions/session.ts#Session.prompt
  - packages/host/apiproxy/src/api/sessions.ts#SessionsApi
  - packages/host/apiproxy/src/api/agent-presets.ts#AgentPresetsApi.select
  - packages/host/apiproxy/src/api-proxy.ts#ensureSession
  - packages/core/agent/src/index.ts#AgentHandle
  - packages/core/agent-loop/src/index.ts#setupAndPublish
  - packages/session/session-persistence/src/index.ts#SessionPersistence
  - packages/session/session-persistence/src/coordinator.ts#installWritePath
  - packages/session/session-persistence-jsonl/src/index.ts#appendBatch
  - packages/preset/agent-presets/src/index.ts#AgentPresets.recompose
  - packages/preset/agent-presets/src/session.ts#resolveSessionPreset
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-05
asked_by: agent
---

## 条件与版本

八维坐标：TS/JS 宿主 · 官方 Web Client/Host · 普通 Session，异步操作期间可切换导航 · Host 可见 cwd/Workspace · 工具随 preset standing composition 提供 · 不依赖真实模型调用 · 同一 live Host，耐久性另受 backend 与事件写入条件限制 · 固定 `dsh-v0.1.1-rc.2` / `b150a551`。

2026-09-05 只读核对远端 tag 与本地对象一致。旧通用页为 stale，仅用作路由；本页重新读取目标 tag 的公开类型、实现与一方测试源码，不代表 master。认知状态为 `verified_inference`，未运行测试，不提升为实测事实。

## 固定寻址与导航选择

`ISessions.binding(id)` 返回的 `SessionBinding.session` 是 `SessionFace`；`ISession` 公开 readonly sessionId、prompt、cancel。`SessionRuntime.resolve(id)` 按 id 取 `manager.get(id)` 并缓存 binding；Session 构造时保存该 id。普通 Session 的 prompt/cancel 使用 `this.sessionId`，不读取 `list.current`；open 进入 manager.select，只改变导航与 staging。[公开类型](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/client/runtime/src/client/contract/session.ts#L29-L93) · [binding 实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/client/runtime/src/client/sessions/service.ts#L570-L650) · [命令实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/client/runtime/src/client/sessions/session.ts#L184-L333)。

条件性推论：await 前取得 A 的 face，await 后调用同一 face，即使 current 为 B，仍寻址 A；await 后重新按 current 取 binding 是另一次目标选择。这只保证寻址，不保证 A 的存活、可写性、Workspace 验证或 runtime generation 不变。addressed subagent 另有 parent/child 路由与 one-shot 限制。

## create 的 live 与 durable 边界

- 对新 id 的首次成功创建，Host 等待 preset setup/mount 后发布 live Session/Agent，header 记录 cwd 与解析后的 preset id；带 workspaceId 时还要在 publication 后完成 attach 才返回成功。显式已有 id 的 create 可采用 live Agent 或 resume，不能都视为 fresh blank。[ensureSession](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L1558-L1647) · [create](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L2079-L2152)。
- `SessionPersistence.create` 允许 lazy materialization；官方 coordinator 只登记 materialized:false。session/created 启动初始化，session/event 进入 write-behind，session/flush 是立即 durability barrier，Host create 未等待此 flush。因此 create 成功不是 Session log/header 已落介质的回执。[Persistence 合同](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/session/session-persistence/src/index.ts#L126-L143) · [lazy createCore](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/session/session-persistence/src/coordinator.ts#L629-L658) · [写入 hooks](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/session/session-persistence/src/coordinator.ts#L1117-L1132)。
- Host attached blank 的判据是没有 turn/start，不是零事件；标题、命令、goal、preset selection 等事件仍可落盘。只有 lazy backend 且无 seed/任何事件被持久化等条件成立，才可推断 fresh Session 尚无 artifact。不能由“首 prompt 未接受”推出“无 durable 副作用”。[Host blank](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L441-L450)。
- 明确未调用/在 admission 前拒绝，与超时、断线、响应丢失不同。Host 在 followup/steer 后返回 accepted；没有收到回执不能证明未接受或仍 blank。独立 prompt 拒绝不撤销成功 create。[prompt admission](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L2361-L2416)。
- header/selection event 记录 preset id，不保存定义、插件实例或完整 generation；live 装配成功与重启恢复的条件不同。[preset fold](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/preset/agent-presets/src/session.ts#L18-L54)。

## 无原子 abandon/delete，但有同 id blank switch

`SessionFace`、`SessionsApi`、`SessionPersistence` 无 session abandon/delete，也无覆盖 create、在途 prompt、日志与 standing composition 的整体撤销操作。Client clear 只清导航；cancel 取消当前 turn 且保留 inbox。AgentHandle.dispose 是本地创建者持有的生命周期能力，Host create 只返回 id/preset，不返回 handle，dispose 也不等于 durable log delete。[SessionsApi](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api/sessions.ts#L235-L377) · [AgentHandle](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent/src/index.ts#L158-L174)。

**存在** `api.agentPresets.select({sessionId, agentPreset})`（wire：agentPreset.select）。Host 在 per-session select queue 内检查真实 log 的 blank，非 blank 返回 agent-preset-locked，无 roster 也拒绝。它保留原 Agent/Session，rebind 到新 standing，再追加 agent-preset/selected；header 保留最初选择，resolveSessionPreset 取最后一次 selection。[select 类型](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api/agent-presets.ts#L63-L72) · [select 实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L2984-L3033)。

跨源码控制流推论：presetSwitches 只串行 select，prompt 不加入同一队列；blank 检查在 await recompose 前，不是与首 prompt 共享的 reservation。recompose 先确保新 standing 再 rebind，未知/不可装配 preset 在 rebind 前失败会保留旧关联；但 rebind、后续 Session.append 与 persistence 不在同一事务，不能承诺任一步失败都 all-or-old。切换不卸载旧 standing。[recompose](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/preset/agent-presets/src/index.ts#L437-L471)。

同 id 重试 create 并传不同 preset 不是 switch：ensureSession 对已有 effective preset 做冲突检查。带 agentPreset 的 create 属于 Host SessionsApi / IApiClient.sessions.create；concrete SessionRuntime.create 的 opts 只有 workspaceId/cwd/sessionId，不透传 preset。[Client create](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/client/runtime/src/client/sessions/service.ts#L474-L489)。

## 验证边界

阅读一方 binding identity、blank preset select、JSONL lazy materialization 测试源码交叉核验，未执行测试或调用模型。具体部署的 backend、事件序列、落盘结果与故障后存活状态不在已验证范围，没有用未验证假设补齐条件。
