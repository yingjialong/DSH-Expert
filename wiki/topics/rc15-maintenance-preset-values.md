---
title: 1.5-rc.2 maintenance、preset 与 values
description: 核对活动收敛、有效预设读取与值工具迁移。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-23
updated: 2026-09-23
asked_by: agent
anchors:
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent-loop/src/index.ts
  - packages/core/tools/src/index.ts
  - packages/preset/agent-presets/src/index.ts
  - packages/preset/agent-presets/src/session.ts
  - packages/preset/agent-presets/src/mount.ts
  - packages/session/session-projection/src/index.ts
  - packages/util/values/src/index.ts
related:
  - "[[wiki/topics/rc15-tool-effect-boundaries]]"
  - "[[wiki/topics/rc15-client-agent-preset-projection]]"
---

固定T如header；对照S=0.1.1-rc.2 / b150a551b8d465e31e418e1b2eaf5e79bbb7d28e。T tag及util-values/presets正式包sha512/root声明核验；完整回传B5-MCP-015RC2-PUBLIC-01后沉淀，未运行。

条件：TS Host · 官方AgentLoop · maintenance/driver互斥 · provider I/O未验 · own工具与继承restriction · 无模型调用 · 同进程scope · 固定T。

maintenance同步检查内部phase idle，不检查inbox为空；public status在maintenance仍idle。waking输入在idle同步进入running，故随后maintenance拒绝；非waking pending可与真idle并存。callback期间wake只排队/latch，finally无论成败都可启动仍pending的wake。返回Promise保留job结果，whenIdle跟随activityDone换代但不作为job成功凭证。cancel默认清inbox/latch并abort；不合作job不被强停。Handle.dispose真实cancel(disposed)→whenIdle→scope.dispose→handle.close→detach。

agent.ctx.tools注册own与restrict继承正式可用；callback结束不自动撤销，不提供多注册事务。same-layer重复/非法schema/unknown restriction可失败，先成功的注册不回滚。whenIdle不追踪任意direct execute/detached I/O或async tools/result listener；tools/change不是execution drain。

有效preset用公开sessionProjections.stateOf(session,'agentPreset')，定义init header值或null，selected更新；cold观察读官方projections，不自行fold。缺definition为undefined，无记录为null。defaultId每调用settings.default??config.default；缺省resolve用它，明确id失踪不静默换默认。live用composedPreset(agent.ctx)或公开standingMountFor读取真实join；standingKeyFor可能挂载插件，不是纯读。header仅创建事实，projection不等于固定代码generation。

S旧resolveSessionPreset({header,events}) root出口在T移除，由projection合同替代。

T util-values root公开snapshotJsonValue/isJsonValue/deepFreeze/deepEqualJson/assertNever，JsonValue为TYPE。snapshot单遍值读取、detach但不freeze；invalid JSON返回undefined，getter/proxy异常传播。拒绝循环、非有限/-0、稀疏/多余key数组、exotic、symbol/nonenumerable及非JSON标量；plain/null-prototype可用，null-prototype输出普通对象。shared子图非祖先cycle。deepFreeze就地返回同对象，跳AbortSignal、防cycle，仅递归Object.keys子对象；非clone/JSON validator/全内部槽不可变，抛错可部分已freeze。

与S core/session/json.ts的walker及S llm/call-config.ts的deepFreeze主体比较一致，主要搬迁出口，未发现本题函数语义变化。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/agent.ts:157`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/preset/agent-presets/src/session.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/util/values/src/index.ts`。fixture `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/tests/loop.spec.ts:336` 等仅读未运行。
