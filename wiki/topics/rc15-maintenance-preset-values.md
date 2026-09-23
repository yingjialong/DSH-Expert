---
title: 1.5-rc.2 maintenance、preset 与 values
description: 核对活动收敛、有效预设读取与值工具迁移。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-23
updated: 2026-09-24
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


## 非hash preset ID 的当前实现

2026-09-23，asked_by: agent；先完整回传DSH-015-B6-PRESET-IDENTITY-01，未运行。ID是preset目录名，发现正则^[a-z0-9][a-z0-9-]*$，不是内容hash要求。公开roots可声明system信任并在原合法id目录提供当前agent.cordis.yml；shipped默认最先、configured其次、user最后，同id先root胜出，includeShippedRoot可显式关闭。Profile/bundle只是当前runtime装配，不替Session重命名。

默认Controller resume按effective projection resolve/mount；mount通过setup join不append selected。显式id缺失/损坏拒绝，不等于未记录值采用default。裸AgentFactory.resume的setup由调用者负责，不宣称核心自动mount所有preset。低层restore/query/export不验证roster可挂载，但普通follow会后台promotion而可能失败。

无通用preset历史ref重写API；format V2→V3有明确code→ptc特例，不能泛称所有字符串永远原样，也不提供宿主任意hash映射。其它合法旧id由当前root提供实现是公开可组合性，不证明历史行为等价或代码generation冻结。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/preset/agent-presets/src/preset.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/boot/app-boot/src/profile.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-format-v2-to-v3/src/migration.ts:18`。discovery/authoring fixtures仅读未运行。


## 2026-09-24：inactive查询与取消批次

asked_by: agent；先完整回传DSH-015-B6-DISPOSE-03。Cordis ctx.get(name)跳过inject要求，仅按realm查ACTIVE provider，调用者inactive无专门拒绝；provider已unloading也返回undefined，不是未注销就返回。直接ctx.sessions可因required注入已inactive抛错。get(false)不复活服务，非生命周期lease。

SessionStore.get/list实现无assertActive，只读store；保留对象不必然throw，但无卸载后可用/完整最终snapshot承诺。服务注销与业务记录detach非同一原子步骤，不据旧对象推断writer drain。

AgentLoop公开默认maxParallelToolCalls=10。正常取消且无scheduler/log失败时N项计划仍写N call/N result，未started为synthetic ABORTED_BEFORE_DISPATCH，不等body已执行。started先drain再按序结果；scheduler failure不保证配对，tools/result异步observer不await。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/cordis/src/reflect.ts:233`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/session/src/index.ts:1170`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/constants.ts`。fixture `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/tests/tool-calls.spec.ts:466`仅读未运行。


## 2026-09-24：Session title 后台生命周期

asked_by: agent；DSH-015-B6-TITLE-01已完整回传，固定T，session-title正式包sha512/root声明核验，未运行。SessionTitleService需sessions/sessionProjections与三个必需正整数config；register唯一provider后服务自身即自动consumer，AgentLoop图不自动装它。

first-prompt只针对无parentSession、第一条eligible source=user文本且无title；预留pending后等request/header或marked主llm/stream匹配route边界才generate。不是挂载即补跑历史。provider failure显式refresh重试；请求messages按watermark截断。

service提交session/title，投影latest-wins，持久化依赖已有writer路由与后续flush/close。后台inFlight不属于Agent.whenIdle；provider disposer/service teardown会abort并await跟踪Promise，Session disposed通知本身不await所有任务。不合作generate可阻塞drain。自动signal不继承主turn取消；purpose=session-title不自动进入AgentLoop agent/request-error retry。

rename supersede后source=user pin，迟到provider受signal/revision/exact-session检查不能覆盖；refresh是明确unpin。projection本身不强制任意直接append的手动优先。官方title-llm另加timeout与session/title-llm-request，自有provider不自动继承。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-title/src/index.ts`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-title-llm/src/index.ts`。fixture `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-title/tests/provider.spec.ts:119`与rename.spec.ts:131仅读未运行。
