---
title: packages/core — 产品 API 主干
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/core/README.md
  - packages/core/agent/README.md
  - packages/core/agent-loop/README.md
  - packages/core/tools/README.md
  - packages/core/system-prompt/README.md
  - packages/core/system-prompt/src/index.ts
  - packages/core/system-prompt/tests/system-prompt.spec.ts
  - packages/core/session/README.md
  - packages/core/scope/README.md
  - packages/core/agent-default-model/README.md
  - packages/core/agent-tool-presentation/README.md
  - packages/core/agent/src/index.ts
  - packages/core/agent/src/runtime-types.ts
  - packages/core/agent/src/dispatch.ts
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent-loop/src/tool-calls.ts
  - packages/core/agent-loop/src/index.ts
  - packages/core/agent-loop/tests/tool-calls.spec.ts
  - packages/core/agent-loop/tests/cancel.spec.ts
  - packages/core/agent-loop/tests/request-error.spec.ts
  - packages/core/agent-loop/tests/request-reconstruction.spec.ts
  - packages/core/tools/src/index.ts
  - packages/core/tools/src/invariant.ts
  - packages/core/tools/tests/tools.spec.ts
  - packages/core/tools/tests/code-mode.spec.ts
  - packages/core/tools/tests/ts-types.spec.ts
  - packages/core/tools/tests/py-types.spec.ts
  - packages/core/session/src/surface.ts
  - packages/core/session/src/repair.ts
  - packages/session/session-persistence/tests/contract.ts
  - docs/subsystems/core.md
  - docs/subsystems/tools.md
  - docs/tool-execution-pipeline.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-31
asked_by: self
---

## 一句话定位

harness 默认控制主干：会话日志、system prompt 组装、工具注册与执行流水线、Agent 词汇表与注册表、部署默认模型、以及唯一那份具体 loop——插件与消费者要编程对着的稳定面。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包目录 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `scope/` | `@deepseek-ai/dsh-scope` | 无（库，不是服务） | 作用域注册原语：`createScope` / `scopeOf` / `scopeTarget` / 父链继承 |
| `session/` | `@deepseek-ai/dsh-session` | `ctx.sessions` | 事件溯源的 session 日志 + surface 投影 + 内存 store |
| `system-prompt/` | `@deepseek-ai/dsh-system-prompt` | `ctx.systemPrompt` | prompt section / tool schema / 变量的组装注册表 |
| `tools/` | `@deepseek-ai/dsh-tools` | `ctx.tools` | 作用域化工具注册表 + 执行流水线 + 模型呈现形态（native / Code Mode） |
| `agent/` | `@deepseek-ai/dsh-agent` | `ctx.agents` | `Agent` 接口、注册表、initiator scope、`agent/*` 事件词汇表 |
| `agent-default-model/` | `@deepseek-ai/dsh-agent-default-model` | `ctx.agentDefaultModel` | 入口点共用的部署级默认 provider/model |
| `agent-loop/` | `@deepseek-ai/dsh-agent-loop` | `ctx.agentLoop` | **唯一**含具体 loop 逻辑的包 |
| `agent-tool-presentation/` | `@deepseek-ai/dsh-agent-tool-presentation` | 无（preset row） | agent preset 用来声明本 agent 看到 `native` / `code` / `both` 哪种工具形态 |

## 三件套结构

core 是「主干」而非单一 capability family，但同样按 seam 纪律拆分：

- **Service Definition**：`dsh-agent`（`Agent` 接口 + `ctx.agents` + 全部 `agent/*` 事件，零 loop 依赖）、`dsh-tools`、`dsh-system-prompt`、`dsh-session`。
- **Service Provider**：`dsh-agent-loop`——它实现 `AgentFactory` 并在构造时 `ctx.agents.setFactory(this)`，所以插件永远通过 `ctx.agents.create()/resume()` 建 agent，不需要 import loop 包。
- **Consumer**：本组不放模型工具。可跑的组合在 `packages/examples/agent-spine-demo`。
- `scope` 是被上面几个注册表共用的底层库（`ScopeLayer` / `ScopedLayers` / `NamedEntries`），key-agnostic，不依赖 agent。

## 扩展点

- **换掉 loop**：依赖 `dsh-agent`（seam），实现 `Agent` 并 `ctx.agents.register()`；或实现 `AgentFactory` 走 `setFactory`。不要改 `agent-loop`（改它必须同步更新 `docs/architecture.md`）。
- **工具**：`ctx.tools.register(definition)`（schema 自动流入 prompt 组装）；`ctx.tools.guard()` 单调同步拒绝；`ctx.tools.restrict(filter)` agent 作用域可见性掩码；`ctx.tools.presentAs(mode)` 单 agent 呈现形态。
- **执行流水线四段可插**：`tools/pre-execute`（可重排的 allow/deny/ask 门）→ 注册的单调 guards → `tools/execute`（环绕包装，用于 timeout/retry/metrics）→ `tools/post-execute`（换 content 或 value、block、挂 contexts）→ 定义自己的 `finalizeContent` → 只读的 `tools/result` 通知。
- **prompt**：`ctx.systemPrompt.section()` / `.context()` / `.tools()` / `.variable()` / `.suppressRuntimeContext()`；变换点是 `system-prompt/assemble` waterfall。
- **agent 生命周期**：`agent/pre-step`（可否决，返回 `PreStepDecision`）、`agent/request-error`（失败重试 waterfall，返回 `{ kind: 'retry' }` 且不调 `next()`）、`agent/turn-stopping`；`agent/session-start` 是首个受支持的启动注入点。
- **单 agent 作用域**：一切通过 `agent.ctx` 注册的东西只对该 agent 生效并随其销毁而回收（工具影子、prompt section 影子、监听器）。
- **持久化不在这里**：`dsh-session` 不实现持久化，插件订阅 `session/event`、在 `session/flush` 落盘。

## Known Limitations

- `agent`：initiator scope 是进程内的（worker/子进程/HTTP/重启都要显式带身份）；`agent/session-start` 是无否决权的同步通知，异步组合要放进 factory 的 `setup(agentCtx)`；`cancel()` 默认清空 inbox，没有「只中止当前 step 保留 turn」的 API；一条附加 `UserMessage` 只能带一个 `MessageSource`。
- `agent-loop`：并发安全分类是一元的（要比较兄弟调用才能判定安全的必须串行）；config agent 没有 per-agent persona 或 setup 钩子；**没有内置 turn 预算**。
- `tools`：`timeoutMs` 只是声明，注册表不强制执行（需要 `@deepseek-ai/dsh-tool-call-timeout-policy` 包装器）；`tools/pre-execute` 刻意**不能改写 `exec.arguments`**；Code Mode 中间值是执行本地的、无字节上界、无法从日志重放；`run_code` 每次运行状态全新。
- `system-prompt`：`{{…}}` 没有转义语法；`toolOrder` 配错在**首个 turn 的 prompt 组装**才暴露（只有形状违规在 load 抛）；同 `order` 的 section 按注册顺序 tie-break。
- `session`：无 session 分支树；`fork()` 只在活会话的稳定边界切；`SESSION_FORMAT_VERSION` 钉死在 `0`，无兼容承诺。
- `scope`：只有 scope-aware 的 API 才隔离状态；一个 context 只带最近的一个 scope key；把 `Scope.ctx` 交出去等于把铸造插件的注入服务一起交出去。
- `agent-default-model`：一个进程一个默认值；没有 settings provider 时 `saveSelection()` 是 no-op。
- `agent-tool-presentation`：preset 能选 Code Mode 但供不了 TypeScript runtime——没组合 runtime 的部署就组合不出 code-mode preset。

## 陷阱

- **`tools/result`（live 事件）和 `tool/result`（durable session 事件）名字只差一个 `s`**，是两回事：前者是流水线末尾的只读观察点，后者是 loop 随后追加的会话事件。
- **可选服务必须用 `ctx.get(name)`**，`ctx.<name>` 属性代理对拓扑敏感，只留给声明过 inject 的服务（`packages/CLAUDE.md` 明文规则，有 postmortem 背书）。`dsh-tools` 自己就用 `ctx.get('approval')` 机会性消费审批 seam。
- **`code` 模式不只是展示**：注册表会把模型直呼其他工具的调用在 policy 之前解析成 `UNKNOWN_TOOL`，所以宣称面和可执行面才一致；`run_code` 这个名字**在任何模式下都被保留**，不可注册/影子/限制/移除。
- **ToolRuntime 不是 provider tool-name compatibility validator**：`ToolSchema.name` 只有 `string` 类型，`register()` 只拥有同层唯一性和保留名 `run_code`，不会按 provider 的 regex 或最大长度预检；`schemas()` 又原样投影 name。companion invariant 会拒绝最终 execution 的空 name，而 Code Mode renderer 明确用 quoted/subscript access 支持 `my-mcp.tool` 这类 exotic name。注册成功只能证明 DSH registry 接受，不能证明目标 provider wire 接受。
- **`maxParallelToolCalls`不是max tool catalog size**：它只限制模型已经返回调用后，parallel-safe bodies的执行池并发。`SystemPrompt.assemble()`会收集全部provider schemas并完整进入`PromptAssembly.tools`，没有统一count cap；同step `agent/request-error` retry复用原assembly/tools，只重建request并重跑`agent/request`。
- **scope 是路由不是沙箱**：`restrict()` 是「实时可见性组合」，不是权限边界；`agent-scope` 的 Agent Note 明确把安全与授权列为非目标。
- **`agent.status === 'running'` 不等于「某个 turn 还开着」**：它描述 driver 级的 drain 区间，可能横跨 turn 关闭、持久化 checkpoint 和连续排队的多个 turn。
- **config agent 默认每次启动都是新会话**：省略 `sessionId` 会每次生成 `${label}-session-<uuid>`；要「有则续、无则建」必须给稳定的 `sessionId`，`resumeSessionId` 则要求已有持久化历史且与 `sessionId` 互斥。
- **`SessionStartSource` 预留了 `'clear'` / `'compact'` 但至今没有发射方**（`dsh-agent` README 标 `TODO(compaction)`），尽管 compaction 组已是 product 级——别按这两个值写消费逻辑。
- **`renderPrompt` 是严格模式**：未知变量、注册了但没值的变量、畸形的 `{{…}}` 组、`{{{model}}}` 这种都直接抛。

## 2026-08-30 · rc.2 Tool identity、cancel ack与crash repair

- `ToolRunContext`把`callId/rootCallId`、Agent、frozen arguments与signal交给provider；Registry另mint process-local symbol `token`。`callId`来自模型block/直接caller，不是Host全局operation id，公共契约未保证跨Session/history唯一；`tool/call`event seq虽在该Session内耐久唯一，却不传给provider。
- AgentLoop在prepare/body前先追加`tool/call`，settle后才追加`tool/result`。同live`ToolExecution`可穿过around-dispatch wrapper，但没有跨Host restart的receipt/query或exactly-once seam。
- `session.cancel`只同步abort Agent后立即返回accepted，不await`whenIdle()`。未启动body形成`ABORTED_BEFORE_DISPATCH`；已启动body必须合作观察signal并drain，成功才改为`ABORTED`。这不是physical effect已静止的ack，也不列出in-flight calls。
- Crash load对assistant call但无`tool/call`补`TOOL_NOT_STARTED`；对已有`tool/call`无result补`TOOL_OUTCOME_UNKNOWN`，明确要求side-effect调用先核验外部状态。它防止误判“没执行”，但不替physical owner保存receipt或去重。
- **认知状态**：verified_inference（固定tag ToolRuntime/AgentLoop/Session repair与Persistence contract/e2e tests；未执行native effect）。

## 2026-08-31 · rc.2 ToolDefinition replacement与borrow缺口

- **没有统一definition-generation handle**：`ToolExecutionToken`只做process-local call/parent correlation；`ToolExecution`不含definition、schema、registration或generation，`register()`只返回同步`()=>void` disposer。root虽暴露`TOOL_RUNTIME_SCHEDULER`类型，但明确标`@internal`、非plugin extension point，且prepared result也无definition handle。
- **各阶段读取definition的时点不同**：model assembly克隆current schema；`createExecution()`只快照当时definition的`finalizeContent`；`executionMode()`和body dispatch都按name重查current registry。body成功首次规范化用dispatch时局部`tool`的output contract；wrapper-authored success与post value replacement又按name重查current definition；content-only replacement不重查。
- **一方tests把replacement可见性钉成行为**：exclusive barrier替换工具后pending calls重新分类并执行replacement；Code binding枚举后注销工具，dispatch得到`UNKNOWN_TOOL`；post value replacement或wrapper success期间owner消失也得到`UNKNOWN_TOOL`。相反，execution开始时快照的finalizer即使工具已注销仍会执行。
- **局部引用不是owner lease**：body拿到局部definition对象后会用同一output schema/render投影该次返回值；finalizer也是JS函数引用。但ToolRuntime不保留registration refcount、不阻止global/HMR disposer关闭外部资源，也不在durable`tool/result`记录generation。
- **cancel/drain只覆盖execution**：AgentLoop停止补位、等待started dispatch settle并按model顺序commit；Agent owner dispose先`cancel→whenIdle→scope.dispose`，可保护该Agent scope的普通顺序。它不使任意global tool registration或外部Client自动延迟dispose。
- **未来架构条件**：中心ToolRuntime、opaque token、per-execution WeakMap、Agent in-flight drain、request/header与tool call/result日志、checkpoint pre-dispatch flush，均可承载未来borrow机制；未发现结构性不可能。但现合同仍缺registration/generation owner、schema+dispatch同handle绑定、retire/refcount/release与durable generation observation。
- **认知状态**：verified_inference（固定rc.2正式root `.d.ts`、ToolRuntime/AgentLoop控制流与replacement一方tests；未构造外部carrier竞态）。

### static no-swap并不要求generation borrow

- generation borrow解决的是G2替换后仍需让schema G对应的未完成调用继续持有G。若registration/closure在Session活动期绝不dispose、shadow或replace，call-time按name重查仍得到同一对象，因而该静态语义本身不需要不存在的borrow合同。
- 公开`agent/pre-step`与`tools/pre-execute`/guard/executor可在外部资源失效后拒绝后续step或call；这只建立fail-closed路径，不会生成DSH-owned catalog generation或“必须新建Session”的通用状态。
- 该结论的硬前提包含HMR/owner teardown不得早于in-flight execution settle。只要允许replacement、并发teardown或需要DSH证明同代，上一节列出的borrow/retire/refcount/drain缺口仍完整存在。
- **认知状态**：verified_inference（固定rc.2 ToolRuntime current-registry解析与AgentLoop drain控制流；无运行期换代实测）。

### async catalog check的真实ordering与teardown边界

- 每个step先append`turn/start`，再由`preStep()`调用`systemPrompt.assemble()`；assemble同步调用全部tool-schema providers、clone/order schemas，随后才await scope-filtered`system-prompt/assemble`waterfall，返回后才进入`agent/pre-step`。因此两条async event都能在model adapter前fail closed，但都不是“schema collection之前”。`SystemPrompt.tools()`的public provider签名是同步`ToolProviderResult`，不能直接await remote check。
- `system-prompt/assemble`或`agent/pre-step`throw会把turn收成`UNKNOWN` error且不调用model；pre-step显式reject则收成blocked。两者都带turn signal，但不合作的listener Promise不会被DSH抛弃。assembly每step只算一次，同step`agent/request-error`retry复用原assembly，不会重跑check。
- tool侧顺序是durable`tool/call`→async`tools/pre-execute`→可选Approval→同步guards→`tools/execute`wrappers→body。`guard()`不能async；pre-execute距离body中间可能隔着人类Approval。adapter若要求最后一次远端校验，应把它放在`ToolDefinition.execute`的第一步、物理effect之前；任意check后的外部TOCTOU仍需远端digest/version条件调用解决。
- `AgentHandle.dispose()`公开语义包含cancel→whenIdle→agent-scope dispose，factory unload也drain其handles；Preset standing scope却归`AgentPresets.selfCtx`并只在whole-tree teardown回收。DSH没有Host-wide admission-close→all Agents drain→standing/plugin dispose协调器。Host可保留public handles/fibers顺序组合；无法证明该顺序或存在agentless calls时，plugin仍需自管in-flight drain。
- **认知状态**：verified_inference（固定rc.2正式system-prompt/agent/tools types、AgentLoop/ToolRuntime控制流与一方tests；未连接远端catalog）。

### `agent/request`是loop-level retry的逐attempt fence

- `step(assembly)`只收一次assembly，但其`while(true)`每轮都重新`buildRequest(...)`；buildRequest每次通过fused Agent dispatcher运行scope-filtered`agent/request`waterfall，再prepare adapter与发起stream。`agent/request-error`返回retry后continue回同一while顶部，因此下一次loop-level adapter attempt会重跑`agent/request`，但复用原`assembly.tools`与rendered system。
- listener throw/reject使buildRequest失败、adapter不调用，且该middleware failure不再进入`agent/request-error`；一方test断言adapter requests与recovery次数均为0。listener返回后还有`signal.throwIfAborted()`，cancel可阻断；不合作Promise仍会卡住turn。
- 此保证只覆盖AgentLoop因`agent/request-error`产生的retry。adapter/HTTP SDK在单次`stream()`内部自行重试时不会重新buildRequest，也不会重跑该fence。
- **认知状态**：verified_inference（固定rc.2 Agent scoped event types/dispatcher、AgentLoop step/buildRequest控制流与request-error/reconstruction tests）。

### 同一batch pending/started calls与注销顺序

- Scheduler在started calls按model order commit后会继续`fillPool()`；只等待当前executor settle而不先cancel Agent，pending calls仍可补位。官方replacement test明确让pending calls在前一barrier换tool后重新classification并执行replacement。
- unregister不取消已进入body的call；局部definition引用与Promise继续settle。尚未进入body的call会在`dispatchToolBody()`按name重查：无definition时`UNKNOWN_TOOL`且不调用旧executor，同名replacement存在时执行replacement。已进入async pre-execute/Approval但尚未body的call也属于后者。
- Agent cancel先abort共享turn signal，阻止pool补充；已started dispatch全部drain，未启动calls被写成balanced synthetic`tool/call`+`tool/result`，错误`ABORTED_BEFORE_DISPATCH`。`AgentHandle.dispose()`随后await`whenIdle`再dispose Agent scope；ApiProxy`session.cancel`只回accepted，不是quiescence ack。
- 因此公开可组合顺序是先关闭外部admission，再cancel/dispose所有相关Agents并await idle，补drain任何agentless/detached调用，最后unregister tool与close carrier。若全部调用确由这些Agents拥有，pending已被cancel收口；否则plugin仍需自己的accepting/in-flight计数。
- **认知状态**：verified_inference（固定rc.2`executeToolCalls`/`ToolRuntime`控制流与tool-calls/cancel一方tests；未关闭真实carrier）。

## 去哪深入（文件路由）

| 想知道什么 | 去哪 |
|---|---|
| 逐包的 loop 地图、`Agent` 句柄与投递/拦截契约 | `docs/subsystems/core.md` |
| `agent/*` 事件精确签名、dispatch 模式、scope 过滤规则 | `docs/subsystems/core.md#cordis-surface` |
| 工具流水线完整顺序图 | `docs/tool-execution-pipeline.md`、`docs/subsystems/tools.md#cordis-surface` |
| 已发布工具的 schema 目录 | `docs/tool-catalog.md` |
| 创建/恢复的回滚事务、co-owner 与 quiescence | `packages/core/agent-loop/README.md`、`packages/core/agent/README.md` |
| section order band（-100 身份、0 persona、100–199 工具指导）、`toolOrder` 语义 | `packages/core/system-prompt/README.md` |
| surface / 事件溯源 / `fork()` / 有序拆卸原语 | `packages/core/session/README.md`、`docs/subsystems/session.md` |
| scope 父链与 `ScopedLayers` 存储 | `packages/core/scope/README.md`、`docs/subsystems/scope.md` |
| 默认可跑组合 | `packages/examples/agent-spine-demo/README.md` |
