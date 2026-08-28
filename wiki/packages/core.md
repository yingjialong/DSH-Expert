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
  - packages/core/session/README.md
  - packages/core/scope/README.md
  - packages/core/agent-default-model/README.md
  - packages/core/agent-tool-presentation/README.md
  - packages/core/agent/src/index.ts
  - packages/core/agent-loop/src/agent.ts
  - packages/core/tools/src/index.ts
  - packages/core/session/src/surface.ts
  - docs/subsystems/core.md
  - docs/subsystems/tools.md
  - docs/tool-execution-pipeline.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
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
- **scope 是路由不是沙箱**：`restrict()` 是「实时可见性组合」，不是权限边界；`agent-scope` 的 Agent Note 明确把安全与授权列为非目标。
- **`agent.status === 'running'` 不等于「某个 turn 还开着」**：它描述 driver 级的 drain 区间，可能横跨 turn 关闭、持久化 checkpoint 和连续排队的多个 turn。
- **config agent 默认每次启动都是新会话**：省略 `sessionId` 会每次生成 `${label}-session-<uuid>`；要「有则续、无则建」必须给稳定的 `sessionId`，`resumeSessionId` 则要求已有持久化历史且与 `sessionId` 互斥。
- **`SessionStartSource` 预留了 `'clear'` / `'compact'` 但至今没有发射方**（`dsh-agent` README 标 `TODO(compaction)`），尽管 compaction 组已是 product 级——别按这两个值写消费逻辑。
- **`renderPrompt` 是严格模式**：未知变量、注册了但没值的变量、畸形的 `{{…}}` 组、`{{{model}}}` 这种都直接抛。

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
