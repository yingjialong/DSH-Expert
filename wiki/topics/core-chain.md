---
title: 核心链 core / session / preset / llm（源码级）
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent-loop/src/index.ts
  - packages/core/agent-loop/src/tool-calls.ts
  - packages/core/agent-loop/src/runtime-context.ts
  - packages/core/agent/src/index.ts
  - packages/core/agent/src/inbox.ts
  - packages/core/agent/src/dispatch.ts
  - packages/core/session/src/index.ts
  - packages/core/system-prompt/src/index.ts
  - packages/core/scope/README.md
  - packages/session/session-persistence/README.md
  - packages/session/session-projection/README.md
  - packages/preset/agent-presets/README.md
  - packages/llm/llm/src/index.ts
  - docs/subsystems/core.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

# 核心链：从输入到结果，代码上到底走了什么

路由：事件签名 → `docs/subsystems/core.md#cordis-surface`（生成区）；事件变体目录 → `docs/subsystems/session.md` + `docs/persistence-catalog.md`；StreamChunk 协议 → `docs/subsystems/llm-streaming.md`；时序图 → `docs/agent-lifecycle.md`。本页只写函数级调用链、替换/集成的硬约束、跨文档才拼得出的结论。

## agent loop 的真实结构（收到输入 → 产出结果）

唯一实现在 `packages/core/agent-loop/src/agent.ts#ReactLoopAgent`。调用链（行号为该文件）：

```
send(message, target, wakeup)                 :113  ← followup/steer/inject 都是它的固定预设别名
  └ inbox.splice(...)                                 先写 durable splice，再改 live projection
  └ wakeDriver(wakingAfterAbort)              :172
      └ ctx.agents.withInitiator(this, kick)  :192  ← 整个驱动跑在 initiator ALS 边界内
kick()                                        :210  while (await this.turn()) {}
 turn()                                       :246  append 'turn/start' → 步骤循环 → finally append 'turn/end'
  ├ preStep(target, {turn, step})             :225
  │   ├ inbox.claim(target, turn)                    next-step 全部 + turn 边界上 1 条 next-turn
  │   ├ ctx.systemPrompt.assemble(assembleContextFor(this, signal))
  │   ├ runtimeContext.project(...)                  动态 context 快照 → 候选 user/message
  │   └ dispatch.waterfall('agent/pre-step')         默认 enter = claimed + context
  ├ append 'step/start' → 每条 decision.messages append 'user/message'
  ├ step(assembly)                            :332
  │   ├ buildRequest(...)                     :426  agent/request waterfall → ctx.llm.prepareCall → request/header(+request/context)
  │   ├ preparedCall.stream(request) ?? ctx.llm.stream(request)
  │   ├ 每个 chunk append 'assistant/chunk' + BlockAssembler.push
  │   ├ finish=error|aborted → waterfall 'agent/request-error' → {kind:'retry'} 则 continue 重走 buildRequest
  │   ├ append 'assistant/message'（sourceEventSeqs 列出 chunk seq）
  │   └ executeToolCalls(...)                 tool-calls.ts:59
  └ 无 tool 续接且 next-step 空 → serial 'agent/turn-stopping' → 再查一次 inbox → break
```

关键事实（不在任何单篇文档里）：

- **phase 是三态联合** `idle | maintenance | running`（agent.ts:38），但 `status` 把 `maintenance` 映射成 `'idle'`（:99）。所以 `runMaintenance()` 运行期间外部看到的仍是 idle。
- **system prompt 组装发生在 `agent/pre-step` waterfall 之前**（preStep 里先 assemble 再 waterfall）。pre-step listener 改不了本 step 的 system 文本和 tool schema。
- **`agent/request-error` 的 retry 是 `step()` 内的 `while(true)`**，会重跑 `buildRequest`，因此 `agent/request` waterfall 和 header 折叠会再来一遍。
- **`turn/end` 的 reason 在 finally 里落**；`max-tokens` 粘性，后续正常完成的 step 不会把它降级（:290）。
- **tool 调度不接受 Session 参数**，而是 `ctx.agents.requireInitiator()` 取回 agent 再 derive session（tool-calls.ts:67）；每组开始时重读 `ctx.agentLoop.config.maxParallelToolCalls`（:131），settings 改动只影响下一组。

## agent-loop 为什么被设计成可替换

上游明确：`dsh-agent-loop` 是唯一含具体 loop 逻辑的包，扩展插件依赖 `dsh-agent` 而**从不**依赖 `dsh-agent-loop`（`docs/subsystems/core.md:20`）。机制上的三道保证：

1. `AgentLoop` 只是把自己注册进注册表：`ctx.effect(() => ctx.agents.setFactory(this))`（index.ts:350）。消费方永远调 `ctx.agents.create/resume`，拿到的是 `dsh-agent` 里的 `Agent` 接口。
2. `ReactLoopAgent` 是包内私有：包 exports map 不开 `./src/*`，外部无法构造或启动驱动内部件。
3. 一切可观测行为只走两条通道：durable session 事件 + `agent/*` 事件分类。

**换 loop 的实际约束**（拼出来的）：替代实现必须自己 (a) 走 `ctx.agents.enter/announce` 的发布顺序并保证 detach 顺序；(b) 复刻 `agent/*` 全套语义（含 `pre-step` 的 reject/enter 权威性、`turn-stopping` 的"数据决定"规则）；(c) 提供 `maxParallelToolCalls` 等价物——`tool-calls.ts` 直接读 `ctx.agentLoop.config`。`agent-loop/*` 命名空间只有一个事件 `agent-loop/config-start-failed`，是 loop 私有的启动失败信号。

## session 数据平面：持久化 seam、JSONL/SQLite、projection seam

三层"derive"必须分清（这是集成方最常混的）：

| 层 | 入口 | 产出 | 谁消费 |
|---|---|---|---|
| 模型历史 | `session.deriveMessages()` | `Message[]`，按 surface 节点投影，按节点缓存，`replaceGeneration` 变化才重建 | agent-loop 组请求 |
| 客户端读模型 | `ctx.sessionProjections`（`init/apply/view` 三个纯同步函数 + `stateVersion`） | wire-JSON 整值 | apiproxy / UI |
| 原始事件流 | `session/event`（同步通知） | `SessionEvent` | 持久化、telemetry、UI token 流 |

- **Session 本体**：`packages/core/session/src/index.ts#Session`，`seq === log.length` 契约，事件与 data 在 accept 时 deep-freeze，header 走单独通道不进日志。
- **发布三段式**：`prepare()` → `enter()`（返回 detach，尚未 announce）→ `announce()`。`create()` 只是把三段包进一个 effect 的便利函数；agent factory 故意不用它，因为它要把 session 生命周期折进 agent 的同一个复合 effect 里，保证 teardown 有序（否则驱动的收尾事件会在 publication hook 被拆掉之后才 commit，直接丢）。
- **`session/flush` 只有一个入口**：`ctx.sessions.flush(session)`，store 持有 carrier，不能自己 `ctx.parallel('session/flush')`。
- **持久化 seam**：`ctx.sessionPersistence`（abstract）+ 两个后端。JSONL = 每 session 一个 artifact（`locate()` 返路径、`supportsRawArtifacts=true`，默认 zstd 帧）；SQLite = 共享单库（`locate()` 返 `undefined`、不支持 raw artifact、schema 17，拒绝旧 schema 而非迁移）。两个一方后端共用 `PersistenceCoordinator`，只实现 `PersistenceBackend` 钩子。
- **checkpoint 是另一个插件**：`dsh-session-checkpoint-policy` 才决定"模型请求前 / 顶层 tool body 前 / 每个 pre-step"落盘。**projection cache** 存 `(sessionId, key, ver, seq, val)` 行；改了 fold 语义必须 bump `stateVersion`，否则旧行会被 forward-apply 成垃圾。

## 会话的并发与隔离边界（条件 → 结论）

| 条件 | 结论 |
|---|---|
| 同进程两个操作用同一个 `SessionId` | 两边都能 `prepare`，但 `SessionStore.enter()` 是唯一仲裁点，输者抛 `session "…" already exists` 并回滚私有资源。id 的全局唯一性由**调用方**保证，UUID 碰撞明确在支持模型之外。 |
| 多进程 / 多副本共享同一存储目录或 db | **没有写者互斥**。coordinator 的 revision 新鲜度检查上游原文"does not add cross-process writer exclusion"；持续外部写者会让 `load`/`inspect`/`prepare` 迟迟不收敛。要多副本必须在存储之外自己加锁。 |
| 把 `ctx.sessions` / `ctx.agents` 当集群注册表 | 都是进程内 Map，`get`/`list`/`roots` 只看本进程。 |
| 跨 worker / HTTP / 队列 / 重启后依赖 `ctx.agents.currentInitiator()` | 拿不到。initiator 是 `AsyncLocalStorage`（agent/src/index.ts:259），进程本地；边界处必须显式带 identity。且 ambient 存在既不是存活证明也不是授权。 |
| 在 preset 的 standing mount 里放模块级可变状态 | 会被所有 join 该 preset 的 session 共享（每进程只 mount 一次）。插件必须按 Session/Agent 键控自己的状态。 |
| preset 的 row 直接 publish 一个 service | 落进 root realm = 进程全局，第二个 preset 同名即撞；必须包在带 `isolate` realm 的 group 里，否则 `mount()` 直接拒绝，且 invariant 会在每次 service 通知时复查。 |
| 想让一个 agent 并行跑两个 turn | 不可能。phase 单一，`runMaintenance()` 在非 idle 时**同步 throw**。并发只存在于单个 step 内的 tool 池（`maxParallelToolCalls`，默认 10；exclusive 调用形成 barrier）。 |
| cancel 之后立刻再 `followup()` | `send()` 在 abort 已发生时把 target 强制改成 `next-turn` 并 latch `wakeRequested`，驱动收敛到 idle 时自动重放；但 `{kind:'disposed'}` 的 cancel 永不 latch。 |
| 想用 `whenIdle()` 判断"我这条消息处理完了" | 不行。它观察整个 agent 的静止，会跟随在观察到的驱动退休前启动的替换性工作；`followup()` 也不返回任何句柄，`MessageId` 只标识 inbox 的插入/claim/discard 事实。 |
| 长驻服务里需要精确回收某个 agent | `AgentHandle.dispose()` 是 capability，只有 create/resume 的 owner 持有；`ctx.agents.get(id)` 只给裸 `Agent`。provider unload 是独立的结构性 teardown 边界，会停并 drain 它造出的每个 handle。 |
| 把 `Scope.ctx` 交给第三方 | 同时交出了 minter 插件的 service 解析面，事后无法收窄。scope **不是**沙箱、不是授权边界（上游明确列为 non-goal）。 |
| 用 projection key 的存在与否做能力探测 | 错。unit 表是进程级的，任何 preset 注册的 key 会出现在**每个** session 的 snapshot 里；必须读 value。 |

## preset：cordis.yml 如何在每个 session 组装出一个 agent

- 一个 preset = 目录 + `agent.cordis.yml`（顶层 plugin row 列表）+ 可选 `preset.yml`（**只**放 `name`/`description`，`id` 来自目录名、`trust` 来自 root）。roster **每进程 mount 一次**（standing scope），session 通过 `dsh-scope` 的 key 父链 join：解析 `agent → preset → global` 近的遮蔽远的，事件准入反向沿链向上。
- **唯一支持的调用点是 agent factory 的 `setup(agentCtx)`**。生产调用点见 `packages/host/apiproxy/src/api-proxy.ts` 的 `composeAgent()`（`installSelection` 后 `await presets.mount(agentCtx, resolvedId)`）。setup 里 reject 会把整个 agent 创建事务回滚。
- **子 agent 用 `composeFrom(childCtx, parentCtx)`**，同步 bind、不重读 roster——这是为了让子 agent 拿到父 agent 历史所在的**同一 generation**（重新按 id mount 会拿到被编辑过的新 generation，或因 preset 被删而直接失败）。见 `packages/subagent/subagent/src/child-agent.ts#applyChildComposition`。
- **generation stamp 只看组合文件的 mtime+size**：改 skill 文件或旁边的资源不会触发新 generation；已 join 的 session 永远留在旧 generation 上，而旧 generation 永不回收。
- **"这个 session 跑的是哪个 preset"用 `resolveSessionPreset(session)`**，不是读 header。header 是创建事实且冻结；切换写 `agent-preset/selected` 会话事件，并再以非 scoped cordis 事件 `agent-preset/selected(sessionId, agentPreset)` 广播。
- `recompose()` 只对"什么都没产出"的 session 合法，**且这个检查由 caller 负责**。rosterless 部署里 `composeFrom` 返回 `undefined` 而不是报错，模型可见的 row 落在 host composition 的 global 层。
- preset 文件是**输入不是持久化目标**：mount 出的子树把 Loader 的 `write()` 覆写成 no-op，否则一个 row 自行 dispose 就会把 `disabled` 写回所有 session 共享的文件。

## llm：抽象服务 + provider adapters

- `ctx.llm` = `packages/llm/llm/src/index.ts#LlmRuntime`：adapter registry + `stream()` + `prepareCall()` + `resolveModelInfo()`，拦截点是 `llm/stream` waterfall。
- **接自定义模型**：继承 `LlmAdapter`（同文件 :180），**只有 `stream()` 是 abstract**；`providerInfo` / `providerRetryPolicy` / `listModels` / `resolveModel` 都有默认实现。注册用 `ctx.llm.registerAdapter(['my-provider'], adapter)`，effect-scoped、全有或全无，返回的 handle 带 `replace(providers)` 做无缝换路由。协议义务清单在 `docs/cookbook/adding-an-llm-adapter.md`。
- provider route 独占；**model id 由 adapter 解释**，`listModels()` 只是 advisory，消费方不得因为 model 不在列表里就拒绝请求。两条且只有两条错误路径：`stream()` throw（transport/protocol，带稳定 code 的 `LlmError`），或以 `finish {kind:'error'|'aborted'}` 结束流（provider in-band）。
- **`prepareCall()` 捕获的是"那一次"的 adapter registration**，跨越异步解析、`request/header` 落盘和终端 dispatch，所以 HMR 不会把 A adapter 的能力结果配到 B adapter 的请求上；它同时用 `adapterDefaults` 标记哪些字段是 adapter 填的，loop 在下一轮 waterfall 前用 `requestProposal()`（agent.ts:55）把这些字段剥掉，让当前 route 自己重新物化默认值。
- **route 没有 adapter 时**：`prepareCall` 抛 `NO_ADAPTER`，loop 捕获并保留 `proposedConfig`，好让 `llm/stream` middleware 接管；真的走到终端 dispatch 仍然 `NO_ADAPTER` 失败（agent.ts:470-474）。
- **重试不在这个包里**：`LlmRuntime` 只存 policy，执行者是 `dsh-llm-retry`（监听 `agent/request-error`）；在 `llm/stream` 里自己重试没有 durable attempt 边界。部署默认模型是另一个服务 `ctx.agentDefaultModel`（`packages/core/agent-default-model`），不校验 catalog 成员资格。

## system-prompt 与 context 注入的时机

两条完全不同的通道，别混：

- `ctx.systemPrompt.section({name, order, text})` → 进 **system 文本**。order 约定：`-100` harness identity，`0` = `deployment:persona`，`100–199` tool 指南；同名近 scope 遮蔽远 scope，同 order 靠注册顺序 tie-break（上游自称"插件加载产物"）。
- `ctx.systemPrompt.context({name, order, text})` → 进 **message history**，不是 system。

时机链（都在 `preStep` 内、每个 step 各跑一次）：`inbox.claim` → `assemble(assembleContextFor(agent, signal))` → `renderContextSections` → `RuntimeContextProjection.project` → `agent/pre-step` waterfall（默认 `enter` 的 messages = claimed + 这条 context 快照）。

- `system-prompt/assemble` waterfall 是最终权威，**但**带 `complete: true` 的 section 会在 waterfall 之后被恢复成唯一 section（`minimal` preset 就靠这个把整个 prompt 钉死）；同时有两个 complete section 会抛。
- `{{variable}}` 严格插值：未知或 `undefined` 直接抛，无转义语法。agent-loop 注册了 `provider`/`model`/`cwd` 三个变量（index.ts:351-353）。运行时 context 快照被 `RuntimeContextProjection` 变成一条 `source.kind='plugin'` 的 `user/message`，**只有内容变化才写**；被 surface replacement 遮蔽后 retained 置空，下次会写一条 `Current runtime context: none…` 的 CLEARED 文案。任一 scope 注册了 suppressor 就整体关闭（`includeRuntimeContext: false`）。
- 想在会话开始前塞模型可见上下文：用 `agent/session-start` 里的 `agent.inject()`。`inject` 走 next-step inbox 且**不唤醒**驱动，idle 时会一直挂着直到 followup/steer 唤醒；也可能错过一个 pre-step 已经 claim 完的批次。

## 陷阱

- `agent/pre-step` 里改 tool 注册/prompt section **对本 step 无效**——assemble 已经跑完了。
- `cancel(cause)` 默认清空 inbox（`{ keepInbox: true }` 才保留）；没有"只中止当前 step、保留 turn"的 API。
- 配置式 agent 不写 `sessionId` 时，每次启动都是全新的 `${id}-session-<uuid>`；要 resume-or-create 必须显式给稳定 `sessionId`，`resumeSessionId` 则要求历史已存在，两者互斥。
- agent-loop 的 settings section **只有** `maxParallelToolCalls`；`agents` 故意不在里面（启动时消费一次，存进去只会看起来生效）。
- `ctx.agents.resume()` 需要 persistence 后端，缺了会明确报错而不是降级。
- 装了 persistence 却不装 `dsh-session-checkpoint-policy`，崩溃会丢批写窗口内的事件。
- preset 的 superseded generation 永不回收（`dsh-skill-filesystem` 默认还带 watcher），"编辑组合文件 → 建新 session"循环会累积。
- `docs/subsystems/core.md:248` 的事件清单已过期：`steering/message` 和 `TurnTriggerMap` 在 `packages/` 里都已不存在（前者被合并成带 identity 的 `user/message`），以 `packages/core/session/src/known-event-types.ts` 的 47 项为准。
