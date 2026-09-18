---
title: 1.5-rc.2 Domain 多实例、checkpoint 排除面与 Workspace UI 合同
description: 区分介质原子性、单域写队列、快照新鲜度与真实 Session 导航。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-18
updated: 2026-09-18
asked_by: agent
anchors:
  - packages/storage/storage-domain/src/index.ts#DomainFacility
  - packages/storage/storage-domain/src/domain.ts#KvTableImpl.update
  - packages/storage/storage/src/backend.ts#KvUnit
  - packages/workspace/workspace/src/entity.ts#mutate
  - packages/workspace/workspace/src/index.ts
  - packages/api/workspace-controller/src/index.ts#follow
  - packages/session/session-checkpoint-policy/src/index.ts
  - packages/session/session-checkpoint-policy/tests/session-checkpoint-policy.spec.ts
  - packages/session/session-checkpoint-policy/tests/crash-recovery.e2e.ts
  - packages/bundle/base/cordis.patch.yml
  - packages/client/ui-workspace/src/client/navigation.ts#connectWorkspace
  - packages/client/ui-workspace/src/client/index.ts
  - packages/client/ui-conversation/src/client/skeleton/ConversationRoot.tsx
  - packages/client/ui-conversation/src/client/skeleton/InputBar.tsx
  - packages/client/ui-conversation/src/client/apply.ts
related:
  - "[[wiki/topics/rc15-publication-budget-live-routing]]"
---

固定目标0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203；远端tag一致。7个正式npm包的sha512/exports/types与关键JS已核，测试仅读源码，未运行。完整答案先回传并确认成功后沉淀。

条件（八维坐标）：TS/Client宿主 · 多Context或共享Host控制面 · Session与Workspace所有权分层 · 共享KV介质 · 官方Domain/Workspace/checkpoint服务 · 无模型调用 · 平台实验未做 · 固定目标版本。L2 / verified_inference。

## 多实例KV边界

- KvUnit的putRecord/setGlobal是单次原子耐久full replacement，接口无expectedRevision/CAS/transform，不替caller序列化并发。
- 每个DomainImpl有独立Promise写队列；update在自己的queue slot读自己的Map，运行fn，再putRecord，成功后改本实例memory并发domain/changed。失败保留本实例旧memory。
- DomainFacility.open一次loadAll并构造memory tables/global；get/entries/global.get同步读缓存。single-open/reserved只限制同facility，不跨Context。
- 正式Domain/KvTable/DomainGlobal/DomainFacility没有reload/refresh/invalidate/watch/CAS-replay。domain/changed是本地post-commit通知。close→open可重读，但旧handle关闭，不是透明在线刷新或冲突重放。
- Workspace entity借Domain.update改record，registry另保存global state；不同Session write lock不保护共享Workspace record/global。

条件性推论：两个独立实例基于旧快照构造replacement，单项原子写仍可覆盖对方更新。自定义backend即使CAS拒绝，Domain也没有公开重载后重跑fn的协议；重试旧full value不等于语义合并。是否实际丢失取决于交错，不能由类型替外部实例验收。

官方已有WorkspaceController Remote动作与follow baseline/increments、/client IWorkspaces快照，供Client共享同一Host owner。这个观察/命令通路不自动同步多个Host共享KV，也不是通用remote StorageDomain协议。

## checkpoint-policy的精确覆盖

正式根导出apply/inject/name，依赖llm/sessionPersistence/sessions/tools；base明确挂载，自定义图须实际组合该plugin。AgentLoop与Persistence基类不自动安装等价wrappers，backend live routing是另一职责。

| 边界 | 行为/排除 |
| --- | --- |
| llm/stream | 有sessionId且live Session才flush；缺id或detached直接next |
| tools/execute | 有agent且top-level才flush；无agent/nested直接next；flush后检查abort再body |
| agent/pre-step | 先flush已有prefix再next，不覆盖后续尚未提交的本次request数据 |
| approval/pre-execute | 未安装checkpoint wrapper；这些阶段早于tools/execute，不保证弹窗前audit耐久 |
| turn/end / idle | 没有最终turn/end监听；最后suffix需实际flush/close/write-behind证据 |

正常allow进入execute时，其前缀中的approval audit可随flush持久化；prepare阶段deny不保证经过body前wrapper。flush失败只阻止下游body/adapter，不撤销更早副作用。sessions.flush无人监听返回false；有listener也不独立证明介质实现正确。

官方测试源码证据（未执行）：

- session-checkpoint-policy.spec.ts:53/96：model stream先等flush，reject不dispatch。
- :110/138/176：tool body等待flush，flush中cancel或reject不运行body。
- :76/85/196：无sessionId/detached/nested不checkpoint，nested断言flushes=0。
- :215：pre-step flush。
- :231：dispose policy后再次模型调用，flushes不再增加；证明其他已挂服务不自动补wrapper。
- crash-recovery.e2e.ts:92/105：模型副作用前request已持久、工具效果后崩溃有call且恢复TOOL_OUTCOME_UNKNOWN。使用JSONL且跳过Windows，不认证任意加密后端，不保证exactly-once。

## UiWorkspace与Composer

默认connectWorkspace先查当前清单里blank/cwd相符/属于Workspace且未archive的Session，有则复用；否则sessions.create。inflight Map只属于该UI实例。openWorkspace的导航supersession只阻止选择，不保证此前Session未创建。watchNavigation在ready且无current时会连接最近Workspace；这是UI policy，不是底层storage要求。

/client正式type-export UiWorkspace。connectWorkspace(workspaceId):Promise<SessionId>承诺返回已经可经Session Controller寻址的Session。另一合法provider可兑现该接口是条件性可组合性，但不能返回undefined、虚构id或仅Workspace选择完成信号而仍称合同等价。

默认apply创建内部UiWorkspaceService并注册browser/picker/hooks；正式Client factory只export apply/inject。内部navigation.d.ts存在不代表UiWorkspaceService是/client runtime export。替换同名service不会自动保留默认UI贡献或解决双owner冲突。

默认ConversationRoot在sessionId缺席时inert并给composer.bar disabled；InputBar editable=live&&!locked&&!machineBusy，无Session时保留DOM作为只读Workspace picker入口。公开Conversation Config只有上传并发字段，未找到无Session编辑开关。slots可扩展不等于原组件已有配置开关；其它产品交互不在本页决定。

## 证据与未知

关键绝对路径（均按目标SHA读取）：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/storage/storage-domain/src/domain.ts`：本实例map/queue。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/workspace-controller/src/index.ts`：Remote/follow。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-checkpoint-policy/tests/session-checkpoint-policy.spec.ts`：卸载negative control。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-workspace/src/client/navigation.ts`：真实Session合同。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/skeleton/ConversationRoot.tsx`：无Session inert。

正式包：[storage-domain](https://registry.npmjs.org/@deepseek-ai%2Fdsh-storage-domain/0.1.5-rc.2)、[ui-workspace](https://registry.npmjs.org/@deepseek-ai%2Fdsh-client-ui-workspace/0.1.5-rc.2)。未知/未验证：真实多实例并发、CAS恢复、缺policy硬崩溃对照与替代UI组合；不给外部项目方案验收结论。
