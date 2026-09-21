---
title: 1.5-rc.2 Controller Session 终局处置与 receipt 生命周期
description: 区分单 Session handle 能力、结构 owner 卸载及失败后的耐久与清理证据。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-21
updated: 2026-09-21
asked_by: agent
anchors:
  - packages/core/agent/src/index.ts
  - packages/api/session-controller/src/agent.ts
  - packages/core/agent-loop/src/index.ts
  - packages/core/scope/src/index.ts
  - packages/core/session/src/index.ts
  - packages/client/file-upload/src/index.ts
  - vendor/cordis/src/fiber.ts
  - packages/core/agent-loop/tests/scope-lifecycle.spec.ts
related:
  - "[[wiki/topics/rc15-model-defaults-file-retention]]"
---

固定0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203；tag一致。agent/controller/file-upload三正式包sha512及root声明核验，固定源码/fixture仅读。完整回传成功后沉淀，未运行生命周期实验，L2 / verified_inference。

条件：TS宿主 · 官方Controller创建普通Agent · 多owner生命周期 · SessionHandle及内存receipt · 仅公开操作 · 无模型调用 · 实际排空未验 · 固定目标。

## 无公开按id取回终局handle入口

AgentRegistry.create/resume返回AgentHandle{agent,dispose}，dispose是仅owner拥有的capability；get(id)只返裸Agent。Controller内部只取(await create/resume).agent，公开SessionController没有disposeSession/delete/close或handle lookup。

重新create/resume同id不是取回handle；不建议读取私有effect/map或手造disposed事件。Workspace.detach只改membership，archive只改可见性；cancel保留inbox且不等idle/close；whenIdle也不卸载。

agent.ctx.fiber属于注册scope。createScope.dispose只拆scope-owned注册，完整生命周期effect在创建caller ownerCtx上且被factory追踪；拆Agent scope不能等同完整终局dispose。

## 有效结构路径

持有AgentHandle时，标准dispose为cancel(disposed)→whenIdle→scope.dispose→handle.close→detachAgent→detachSession，Promise memoized。真实创建owner fiber卸载触发相同生命周期；factory/root卸载涵盖所拥有Agent，不是单Session最小范围API。兄弟Context卸载不等于owner卸载。

标准detachSession实际生产session/disposed；FileUploads监听仍在时同步删除该Session staged map。它没有独立agent/disposed清理，不能提前把Agent事件当receipt清除。

完整root卸载可并行移除FileUploads listener与Agent生命周期；不能保证Map.delete回调一定执行，但旧service生命周期终止、无合法活跃receipt消费面，新service用新Map。不是旧内存全部清零或附件对象回收证明。

## 失败边界

AgentHandle.dispose收集错误后仍尽量detach；handle.close失败也可发生Session脱离/receipt失效，不证明持久化成功。never-settling driver/disposer会阻塞，没有硬超时保证。

Cordis Fiber._unload并行独立disposers，异常catch并logger.error；根dispose settled不证明每个子系统耐久/撤权成功。正常服务生命周期结束与确切disposed事件观察应分别表达。进程崩溃可能不执行disposer；新进程不能恢复旧receipt，但持久附件可留存。dispose也不删除Session历史，后来可冷resume。

## 证据

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/agent.ts`：只取create/resume返回值.agent。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/index.ts`：memoized dispose、owner effect及实际顺序。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/file-upload/src/index.ts`：session/disposed清map。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/cordis/src/fiber.ts`：异常contain及并行卸载。

[正式agent包](https://registry.npmjs.org/@deepseek-ai%2Fdsh-agent/0.1.5-rc.2)。未对外部owner、物理存储或具体Context卸载做运行验收，不扩展GC。
