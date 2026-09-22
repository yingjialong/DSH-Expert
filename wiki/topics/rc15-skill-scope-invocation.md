---
title: 1.5-rc.2 Skill scope、缓存与双入口
description: 区分 Skill Provider 读取、用户指令注入和工具前置 gate。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-23
updated: 2026-09-23
asked_by: agent
anchors:
  - packages/skill/skill/src/index.ts
  - packages/skill/tool-skill/src/index.ts
  - packages/preset/agent-presets/src/index.ts
  - packages/core/scope/src/store.ts
  - packages/core/tools/src/index.ts
  - packages/client/ui-skill/src/client/index.ts
  - packages/api/session-controller/src/skill-catalog.ts
related:
  - "[[wiki/topics/rc15-tool-effect-boundaries]]"
  - "[[wiki/topics/rc1-skill-read-authority]]"
---

固定T=0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203；对照R=0.1.1-rc.2 / b150a551b8d465e31e418e1b2eaf5e79bbb7d28e。两tag核验，两Skill正式包sha512/exports/types核对。完整回传后沉淀，未运行。

条件：TS Host · 官方SkillRegistry/tool-skill · standing/Agent scope · provider自有资源 · ToolRuntime与pre-step · 未调用模型 · provider物理取消未验 · 固定T。

## Registry

registerProvider同步factory接control.signal/invalidate，返回effect disposer。依访问Context scope归层；读取省scope只global，Agent scope按global→远祖→近祖→own合并。近层覆盖同名skill，不过滤其它名字。同层rank低优先，再provider注册order、list localOrder；同层provider同名拒绝，跨层分别存在。runtime同名first-wins，rank250；bundled标准600。

缓存仅collect目录：cwd/scope chain identities/revision，默认128，超限删最早key。注册/撤销/有效invalidate清缓存并无参skills/change；迟到旧control无效。并发失效最多尝试2次，仍变化incomplete；provider失败warn/跳过，不给完整失败明细。incomplete候选可加载但不cache。get每次调用provider.get，借用candidate/definition，非body缓存；name漂移invalidate后undefined，不自动选次优。

list/snapshot/get均invocation-neutral。content是provider给出的正文，locator/resourceBase不提供digest/immutable/auth保证。resourceBase只渲染directory/url/opaque资源指引，不自动读取或授权执行。provider list也可能做I/O，过滤summary不是来源访问屏障。

## 双入口

apply贡献skill工具+2个agent/pre-step listeners。模型tool：list→summary.modelInvocable→get→loaded.modelInvocable，lookup包含cwd/signal/scope:exec.agent，经真实ToolRuntime pre/execute。

用户：pre-step先await next，reject则返回；扫描claimed source.kind=user消息所有text块中有空白边界的/name并去重，get后查userInvocable，再追加source skill-invocation/form instructions。不是独立input-transform事件，不经过工具pre/execute，无该路径tool token。源码较早first-line注释不如invokedSkillNames及mid-sentence测试精确。双false不禁止直接get。

目录listener以tools.get('skill',agent)===本apply工具对象检查，然后snapshot/filter modelInvocable；incomplete不发布。这个exact检查不涵盖用户gesture listener。catalog作为持久context公布，不等于资源不可变或后续操作权限。后续工具仍独立经过各自边界。

## gate 与对照

tools/pre-execute与tools/execute都可await，guard只能同步。execute wrapper在next前的工作位于body前，但不覆盖pre-step/direct skills.get。核心不抛弃gate Promise；prebody取消ABORTED_BEFORE_DISPATCH，body已启动则drain后成功可改ABORTED，已有错误可保留。SkillRegistry waitWithAbort提前停止等待provider不证明物理I/O已停；registration signal与lookup signal不同，不自动融合。dispatch再解析definition，无generation lease。

R→T的skill核心文件仅导入变化，scope/cache/factory/flags非新增。tool-skill目标pre-step改为保留...decision，catalogHistory从session.events改seq/eventAt(SessionSeq)。CallId→ToolCallId、code→ptc是ToolRuntime相关词汇变化；不据此改变Skill授权结论。

## 证据

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/skill/skill/src/index.ts`：registerProvider/collect/get/renderSkillContent。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/skill/tool-skill/src/index.ts`：apply/execute/两个pre-step/invokedSkillNames。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/preset/agent-presets/src/index.ts:462`：scope parent。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/skill/skill/tests/skill.spec.ts`：163 policy-neutral、302取消、407借用、707旧invalidate、825有限重试。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/skill/tool-skill/tests/tool-skill.spec.ts:1034`：mid-sentence gesture。

Fixture仅读未运行，不认证外部Provider实现。
