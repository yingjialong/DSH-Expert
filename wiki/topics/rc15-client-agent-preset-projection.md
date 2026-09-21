---
title: 1.5-rc.2 Client agentPreset 投影与缺值资格
description: 区分 effective preset、Browser 类型入口及 partial list/follow 状态。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-21
updated: 2026-09-21
asked_by: agent
anchors:
  - packages/preset/agent-presets/src/session.ts
  - packages/preset/agent-presets/src/types.ts
  - packages/api/session-controller/src/list.ts
  - packages/api/session-controller/src/control.ts
  - packages/api/session-controller/src/history.ts
  - packages/api/session-controller/src/client/sessions/projection-store.ts
  - packages/api/session-controller/src/client/sessions/manager.ts
  - packages/preset/agent-presets/tests/session.spec.ts
  - packages/api/session-controller/tests/projection-store.client.spec.ts
related:
  - "[[wiki/topics/rc15-public-embedding-migration]]"
---

固定0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203；tag一致。presets/controller/projection三正式包sha512/exports/types核验；presets/types实际JS为空export。完整回传后沉淀，未跑runtime，L2 / verified_inference。

## 读取与值

公开SessionFace.projections.faceOf('agentPreset').getSnapshot()返回当前Client已收到的unknown值，稳定face可subscribe。typed useProjection通过SessionProjectionMap得string|null|undefined。Browser用type-only agent-presets/types扩充及session-projection/types表，不加载Host root runtime。

官方projection init=header.agentPreset??null，selected事件的string覆盖，其余不变。header只是创建事实，不能替代effective投影。null为已计算的无preset记录；undefined可能未收到key/能力缺席，不能映射default或null。selected无null清空合同，空串没有特殊unset语义。roster默认不等于历史Session事实。

## list与follow

Host wire summary带可选projections，Client summary.projectionValues来自与SessionFace共享的ProjectionValueStore。

list live只cachedSnapshot已有cells；cold unseeded只取匹配cache或predecessor title，seeded不给该hint；不在list补fold完整日志。缺cache/key或异常可以省略block。refresh成功不证明preset已完整/最新。

Client list按key apply而非完整seed，缺key不清旧值，higher seq wins；list快照通知经microtask，短暂不与face同步。值存在也是最后收到的状态，不是跨断线实时证明。

control baseline对attached Session正式snapshot；history follow通过官方query observation/all projection给完整cut。page/loadOlder不保证另给preset。完整baseline可清cut上遗漏key，较新frame保留；replacement baseline可truncate遗失尾部rows。face无独立ready/epoch/per-key watermark，不能单独证明Host新鲜确认。

Client正式路径是face/useProjection；未找到专用getEffectivePreset Remote。Host已有Session可用官方projection，冷读可用sessionQuery.observeSession与已注册definition；不要求Client自行fold。roster list/read查当前定义，select有修改，standingKeyFor可能挂载，均非替代历史投影查询。

## 证据

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/preset/agent-presets/src/session.ts`：header/selected。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/list.ts`：partial缓存提示。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/client/sessions/projection-store.ts`：face/seed语义。

一方session.spec.ts验证null与覆盖；projection-store.client.spec.ts验证baseline/更高seq，仅读。未认证具体Client已完成baseline或缓存新鲜度。
