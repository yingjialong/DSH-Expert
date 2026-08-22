---
title: DSH 整体骨架：一次输入的完整旅程
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - docs/architecture.md
  - docs/agent-lifecycle.md
  - docs/tool-execution-pipeline.md
  - docs/event-producer-consumer.md
  - docs/capability-seams.md
  - docs/glossary.md
  - docs/persistence-catalog.md
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent-loop/src/tool-calls.ts
  - packages/core/agent-loop/src/constants.ts
  - packages/core/tools/src/index.ts
  - packages/core/session/src/index.ts
  - packages/core/agent/src/types.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

# DSH 整体骨架

跨 `architecture` / `capability-seams` / `agent-lifecycle` / `tool-execution-pipeline` / `event-producer-consumer` 五篇 + `agent-loop` 源码交叉验证。路径相对 `upstream/deepseek-harness/`。

## 0. 分层：没有特权内核

```
profile（Harness home 里的命名组合）
  └─ bundle 有序叠加（dsh-base 永远第一层）→ profile cordis.patch.yml → home patch → --patch
      └─ Cordis 插件树（每一行 = 一个可被 patch 替换的 config row）
          ├─ core spine: session / system-prompt / tools / agent / agent-loop / scope
          ├─ capability seams: ctx.llm / ctx.fs / ctx.shell / ctx.subprocess / ...
          └─ consumers: tool-* 包、UI、SDK
```

`dsh --profile web --dump-config` 打印实际启动的树；打印出的**任何一行**都能被自己的 patch 替换 `[T1: docs/architecture.md]`。model adapter、tool registry、session log、agent loop 自身全是插件。

## 1. 一次用户输入的完整路径

以下每一步标注：**层 / seam / 事件（durable=进 session log，live=Cordis 事件）**。

| # | 发生什么 | 层 / seam | 事件 |
|---|---|---|---|
| 1 | `agent.followup(content)` 进入唯一 inbox | `ctx.agents` | `agent/inbox/spliced`（**durable**）+ `agent/inbox/inserted`（live） |
| 2 | 有可唤醒消息 → driver 醒 | `ctx.agentLoop` | `agent/status` running（live） |
| 3 | 开 turn | agent-loop | `turn/start`（durable） |
| 4 | claim 待处理 next-step 输入 + 一条排队 prompt | inbox | `agent/inbox/spliced`（纯删除）+ `agent/inbox/claimed`（live，逐条） |
| 5 | **组装 prompt sections + tool schemas** | `ctx.systemPrompt` | `system-prompt/assemble`（waterfall，live） |
| 6 | 决定模型看什么：可改写、可整体拒绝 | agent-loop | `agent/pre-step`（waterfall，live） |
| 7 | 若 reject / 首次 claim 被改写成空 → 关掉一个**没花任何 step 的 durable turn** | agent-loop | 直接跳到 `turn/end` |
| 8 | 开 step，逐条落库 entered messages | session | `step/start` + `user/message`（durable，`surfaceOp: 'append'`） |
| 9 | 从日志投影模型历史 | `ctx.sessions` | `Session.deriveMessages()`（非事件） |
| 10 | 协商本次请求的 provider/model/effort/maxTokens | agent-loop | `agent/request`（waterfall，live） |
| 11 | 发起流式请求 | `ctx.llm` seam | `llm/stream`（waterfall，live） |
| 12 | 每个 chunk 落库 | session | `assistant/chunk`*（durable，log-only） |
| 13 | 请求失败 → 决定重试还是保留原错 | agent-loop | `agent/request-error`（waterfall，live） |
| 14 | 成功 → 落一条助手消息（含 usage、`sourceEventSeqs`） | session | `assistant/message`（durable，**surface**） |
| 15 | 按 `executionMode` 分类待执行调用，barrier + 有界 rolling pool | `ctx.tools` | `tool/call`*（durable） |
| 16 | 工具管线（见 §2） | `ctx.tools` | `tools/pre-execute` → guards → `tools/execute` → `tools/post-execute` → `tools/result`（全 live） |
| 17 | 单一模型可见结果 | session | `tool/result`*（durable，**surface**） |
| 18 | 关 step；还欠请求或来了新输入 → 回到 4/6 继续下一 step | agent-loop | `step/end`（durable） |
| 19 | 自然停止且 next-step inbox 为空 → 终局检查点 | agent-loop | `agent/turn-stopping`（**serial，无 `next()`**，live） |
| 20 | 关 turn | session | `turn/end`（durable，带 `TurnEndReason`） |
| 21 | driver 回到空闲 | agent-loop | `agent/status` idle（live） |

**turn = 0 个或多个 step**；step = 一次模型请求 + 它引发的工具执行 `[T1: docs/glossary.md]`。

## 2. 工具管线内部（第 16 步展开）

```
tool/call（执行前先落库）
  → tools/pre-execute waterfall（hooks / permission / sandbox）
      allow ↓        deny ↓        ask ↓
      │              │             ctx.approval 一次性询问；缺席或无法作答 = deny（fail closed）
      ↓
  → 已注册 monotonic guards（deny 或弃权；identity 受保护）
  → tools/execute waterfall（around dispatch：timeout / retry / metrics）
      → ToolDefinition.execute()
          ↔ fs/write-intent | fs/edit-intent（仅 tool-fs 类改写）
          → 工具自有 durable 事件：todo/write、fs/observed、hook/invoked、hook/result、tool/code-dispatch
  → tools/post-execute waterfall（accept / block / replace / add context）
  → registry 外层归一化（快照抛错 → isError）
  → ToolDefinition.finalizeContent（同步、仅 content 的最后不变量）
  → tools/result（同步通知，冻结的权威结果）
  → tool/result（durable）→ 批次结算后，additionalContexts 以 FIFO 注入 user/message
```

任何一环 throw 都汇入 normalized → finalize，不会绕过 `tools/result`。

## 3. 三个事件域（选错域是最常见的设计错误）

| 域 | 判据 | 例子 |
|---|---|---|
| **Session events** | 事实必须在 reload 后仍然存在 → 追加进日志并经 `session/event` 广播 | `turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*`、`agent/inbox/spliced` |
| **Agent events**（`agent/*`） | 携带活的 `Agent`，用于观察/拦截在途工作 | `agent/pre-step`、`agent/request`、`agent/status`、`agent/turn-stopping` |
| **Capability events** | 给某个 seam 挂策略/适配器，不 import loop | `fs/*`、`tools/*`、`telemetry/*` |

Dispatch mode 是事件公开契约的一部分（`@mode` 标注）：`emit` / `waterfall` / `parallel` / `serial`。**waterfall 监听器必须调 `next()`**，不调即短路——单决策事件的短路是设计，注解型监听器必须委托。

**Model-visible ⟺ logged**：凡是能到达模型请求的东西必须能从日志重建，运行时不变量会断言这一点；新增模型可见输入 = 新增一个 session event。

## 4. seam：为什么换一个 provider 能改变整个产品

seam = **Service Definition + Service Provider + Consumer** 三角色齐全的可换能力，缺一不成 seam。`packages/shell` 是范例：`dsh-shell`（Definition）/ `dsh-bash-local`、`dsh-bash-sandbox`、`dsh-pwsh-local`（Providers）/ `dsh-tool-bash`、`dsh-tool-pwsh`（Consumers）。

跨文档才能得出的关键结论：**filesystem 与 subprocess provider 共享同一个 execution world**，所以把它们指向远程沙箱，Bash / PTY / LSP 会**一起搬走**，不需要 fork 任何 provider `[T1: docs/architecture.md]`。同理 `ctx.sandboxPolicy` 被 `bash-sandbox`、`fs-sandbox`、`terminal-bash` 三家共读，保证 bash 和 fs 不会围到不同的 root 上 `[T1: docs/capability-seams.md]`。

## 5. 最易被误解的术语（保留英文原文）

- **seam** — 完整能力，**不是**单个角色、不是接口。别用它指某个类或扩展点。
- **Service Definition** — 拥有 `ctx.<key>` 的 Cordis `Service`（抽象类如 `ShellExecutor`，或具体 registry 如 `WebRuntime`），**永远不是 TypeScript `interface`**。
- **turn / step / round** — turn 是一次「排空已接纳输入」，含 0..n 个 step；**round 是外层策略迭代**（goal round、Ralph round），不等于 turn，round 计数属于那个策略而不是 session。
- **scope / scope key** — 每 agent 注册单位，**两级扁平**：scoped 注册**不向 subagent 继承**；子树行为用 lineage 数据表达，绝不用 scope 结构。
- **lineage** — 父子事实是**数据**（`parentSession`、durable `delegationDepth`、runtime `subagentDepth`），永不影响可见性。
- **shadowing** — 最具体者胜：同名 scoped tool/section/variable 只在该 scope 内替换 global 双胞胎。
- **restriction** — `tools.restrict` 过滤的是 **GLOBAL 工具集**（按交集组合）；被滤掉的 global 工具在 prompt 中缺席**且**拒绝执行，与「不存在」不可区分。
- **setup window** — `CreateAgentOptions.setup`：scope 与 agent 对象已存在、但 agent/session 尚未发布之前的创建槽。**setup 只注册，绝不驱动 agent**。
- **human command** — 经 `ctx.commands` 的斜杠指令，**不变成模型消息**；既不是模型工具，也不是 `ctx.shell` 的 shell 命令。命令输出是 UI 状态，除非 handler 另行改了 durable 域。
- **goal activation** — 进程内的 `armed`/`disarmed` 许可，**故意不进 durable replay**，所以 resume 和 fork 必须先有一次人类授权的 resume 变更才能自动继续。
- **profile vs bundle** — profile 是 Harness home 里的**命名组合**（列出它叠的 bundle、装的外部插件、自己的 `cordis.patch.yml`）；bundle 是 config rows + 代码的**分发格式**。二者都在自己 `package.json` 的 `dsh` 字段里自述（`dsh.profile` / `dsh.bundle`）。
- **surface** — 只有 `user/message`、`assistant/message`、`tool/result` 三个 `SessionEventType` 能进 surface（48 个 durable 事件类型中仅此 3 个），其余是 log-only。

## 6. 源码交叉验证得到的非文档事实

- `system-prompt/assemble` 在 `preStep()` **内部、`agent/pre-step` 之前**被调用，且早于 `step/start`；组装出的 `assembly` 直接传给 `step()` 使用 `[T1: packages/core/agent-loop/src/agent.ts]`。
- `agent/request` waterfall 的 payload 只有 `{ turn, step, signal }`，返回的是 **`LlmCallConfig` 提案**（provider/model/reasoningEffort/maxTokens），**不是**完整请求；消息历史由 `session.deriveMessages()` 独立投影后再合成 `[T1: packages/core/agent-loop/src/agent.ts]`。缺 provider/model 会直接抛错。
- `agent/request-error` 返回 `{ kind: 'retry' }` 时，在**同一个 step 内**重发（`continue` 回到 while 顶），**不产生新的 `step/start`** `[T1: packages/core/agent-loop/src/agent.ts]`。
- `max-tokens` 结束是**粘性**的：某个 step 撞顶后，后续正常完成的 step 不会把 turn 结果降级；且 max-tokens 那一步直接返回，不执行工具调用。
- 工具批次调度：exclusive 调用形成 barrier，parallel 调用走有界 rolling pool，**每次启动前重新分类**（注册表变化可以临时造出 barrier）；上限是 agent-loop config 的 `maxParallelToolCalls`，默认常量 `DEFAULT_MAX_PARALLEL_TOOL_CALLS = 10` `[T1: packages/core/agent-loop/src/constants.ts]`。
- `ctx.agentLoop` 在 capability-seams 表中的 Role 是 **`bundle`**（不是 `seam` 也不是 `core`）：它是唯一的具体 loop 插件，扩展包应依赖 `dsh-agent` 的事件与服务，**不要依赖这个包**。
- Code Mode：`run_code` 是 tool registry 拥有的**保留传输**，位于可过滤能力层之外；其序列化子调用会**重新进入完整 guarded pipeline**，携带 parent token、记录 `tool/code-dispatch`、把 denial 作为 binding rejection 返回，并**省略 `additionalContexts`** 以保持 call/result 相邻 `[T1: docs/tool-execution-pipeline.md]`。

## 7. 新行为该挂哪里

`docs/architecture.md` 末尾的 "Where new behavior goes" 表是权威索引（模型 provider→`ctx.llm`；模型能力→`ctx.tools`；shell→`ctx.shell`；后台工作→`ctx.jobs`；人类命令→`ctx.commands`；模型可见上下文→`agent.inject()`；durable 状态→扩展 `SessionEventMap`；只作用于一个 agent→用该 agent 的 `agent.ctx`）。改 loop 本身必须同步更新那张表。落地步骤见 `docs/cookbook/extension-cookbook.md`。
