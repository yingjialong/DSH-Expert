---
title: packages/guard — loop 卫生守卫（不是能力，是消费者）
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/guard/README.md
  - packages/guard/repeat-tool-reminder/README.md
  - packages/guard/repeat-tool-reminder/src/index.ts
  - packages/guard/timeout-policy/README.md
  - packages/guard/timeout-policy/src/index.ts
  - packages/guard/timeout-policy/package.json
  - packages/core/tools/src/index.ts
  - packages/core/agent-loop/src/tool-calls.ts
  - packages/core/tools/src/code-mode.ts
  - packages/core/session/src/types.ts
  - packages/session/session-checkpoint-policy/src/index.ts
  - docs/subsystems/tools.md
  - .agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.md
  - .agents/notes/archived/feature/2026-07-08-repeat-tool-guard.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-31
asked_by: self
---

## 一句话定位

两个盯着 agent loop 的行为守卫插件：一个数「连续重复的同参数工具调用」并注入劝告，一个给声明了 `timeoutMs` 的工具 arm 每次调用的协作式 deadline。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `repeat-tool-reminder/` | `@deepseek-ai/dsh-repeat-tool-reminder` | 连续重复工具调用的**劝告式**提醒，挂在 tool 与 agent 事件上 |
| `timeout-policy/` | **`@deepseek-ai/dsh-tool-call-timeout-policy`** | 单个 `tools/execute` around-dispatch listener，arm per-call 协作式 deadline |

## 三件套结构

**这一组没有三件套。** 组 README 原文定性：

> A guard is a self-contained consumer of core services and extension points, **not a swappable capability**.

- **Service Definition**：无。两个包都不注册服务，`docs/capability-seams.md` 里也没有它们的 ctx key 行。
- **Service Provider**：无。
- **Consumer**：两个包本身就是 core 扩展点的纯消费者。`timeout-policy` 是 function/namespace plugin（`name` / `inject` / `apply`，无 default export），消费 `ctx.tools` 提供的 `tools/execute` waterfall；`repeat-tool-reminder` 挂在 `tools/post-execute` 与 `agent/pre-step` 上。

## 扩展点

**想写自己的 guard**：不要向本组「注册」什么，直接做一个消费扩展点的普通 Cordis 插件。本组的两个包就是两种范式的参考实现：

- **`tools/execute` around-dispatch wrapper 的参考实现是 `timeout-policy`**。组 README 与包 README 都称它「the reference `tools/execute` wrapper」。它的关键技巧：cordis 的 `next()` **忽略传入参数**，所以 wrapper 必须**原地 mutate 共享的 `exec`**——换上 derived signal → dispatch → 恢复调用方原 signal（好让 `tools/post-execute` 看到的是调用方自己的 signal）。
- **多个 `tools/execute` listener 按 cordis 注册顺序组合**，顺序即语义：timeout 注册在外 = 「超时覆盖整个 retry 操作」，注册在内 = 「超时覆盖每次尝试」。
- **提醒/上下文注入的参考路径是 `additionalContexts`**：reminder 走 post-execute decision 的 `additionalContexts`（source `{kind:'plugin', plugin:'repeat-tool-reminder'}`），**绝不替换 `content`**——`tool/result` 事件必须保持工具自己的输出以便审计。loop 会把它作为 injected `user/message` 追加在该 step 的工具结果之后。**这样就不需要新增 session event**，仍满足「model-visible ⟺ logged」。
- **超时预算的归属**：budget 由**工具插件自己**在 `ToolDefinition.timeoutMs` 上声明（例如 `dsh-tool-web` 的 `fetchTimeoutMs`/`searchTimeoutMs`、`dsh-tool-fs-search` 的 `timeoutMs`），guard 只负责执行。所以这个插件是**零 config** 的，也就不可能写错工具名。

## Known Limitations

组 README 无该章节；以下来自两个包。

**repeat-tool-reminder**
- **只做精确匹配**：canonicalization 是深度 key-sort + `JSON.stringify`，路径微调、值内多个空格都能绕过；模糊匹配在有需求证据前被拒绝。
- **compaction 不重置 chain**：跨越 compaction checkpoint 的链继续计数。
- **纯劝告**：`PostToolDecision` 已支持 block，但升级到 block 未实现。
- **不共享 subagent 链**：父 agent 与其 subagent 各自独立，重复同一调用永不合并。
- 合法的幂等轮询过了阈值照样被提醒——泄压阀是 `thresholds` / `exclude`。
- **过了最高阈值链条就静默**：提醒只在精确配置的计数上触发，不会持续。

**timeout-policy**
- **协作式，永不硬杀**：deadline 只通过 `exec.signal` 通知，忽略 signal 的工具不会停。
- **无全局默认预算**：只有在 `ToolDefinition` 上声明了 `timeoutMs` 的工具才有 deadline；shipped 的 `bash` / `read` / `write` / `edit` **刻意都不声明**。

## 陷阱

1. **目录名 ≠ npm 名**。`packages/guard/timeout-policy/` 的实际包名是 `@deepseek-ai/dsh-tool-call-timeout-policy`（README 标题也写 `dsh-tool-call-timeout-policy`），**不是** `dsh-timeout-policy`。cordis.yml 里写错名字会直接加载失败。plugin namespace 仍是 `timeout-policy`（`- id: timeout-policy`），三者不一致，务必对照 README 的 yaml 片段。
2. **声明 `timeoutMs` 等于承诺「我协作 `exec.signal`」**。只有会把 `exec.signal` 往下传的工具才该声明它，否则超时只是一句空话——参考实现是 `web_fetch`/`web_search`（经 `ctx.web` 转发给 provider）。
3. **`TOOL_TIMEOUT` 的替换是按 signal（`timeoutOf(d.signal, 'TOOL_TIMEOUT')`）判定的，不是按结果形状判定的**。原因：`tools/execute` 的 base `next()` 是 registry 的 dispatch-with-normalization thunk，会先把 provider 抛出的 upstream-abort 错误变成普通 error result，wrapper 再替换成 `TOOL_TIMEOUT`。
4. **被拒绝的调用照样计数**。检测挂在 `tools/post-execute`，而它对被 `tools/pre-execute` 拒绝的调用也会跑——「模型死磕一个被拒调用」正是最该打断的循环。
5. **未跟踪的调用对链是透明的**：被 `include`/`exclude` 排除的调用既不加也不清零计数器，所以 `grep X → todo_write → grep X` 在 `todo_write` 被排除时仍算两次连续 `grep X`。这正是排除功能的意义——记账类工具不能给循环「洗白」。
6. **`thresholds` 加载即失败**：空列表、非整数、小于 2、重复值都会抛异常，**绝不静默回落到默认值**；`argumentsPreviewChars` 同样只接受 `>= 1` 的整数。但 `include`/`exclude` 的模式**不做 referent 检查**（匹配不到任何已注册工具不是错误，`exclude: [mcp_*]` 在没装 MCP 的部署里合法），这一点与 `toolOrder` 相反。
7. **链是内存态、按 live agent 对象 keying 的 `WeakMap`**。从持久化恢复的 session 从空链开始；没有 agent 的直接 `ctx.tools.execute()` 调用被忽略。
8. **`argumentsPreviewChars` 只截断提醒文本，不截断检测**。链 key 永远比对完整 canonical 串。

## 去哪深入（文件路由）

| 问题 | 去哪 |
|---|---|
| 两个 guard 的定位与 ctx 挂点 | `packages/guard/README.md` |
| `additionalContexts` 如何变成 logged `user/message` | `docs/subsystems/tools.md` |
| 阈值配置、chain 语义、两段提醒模板原文 | `packages/guard/repeat-tool-reminder/README.md` |
| `TOOL_TIMEOUT` 结果的确切结构、wrapper 组合顺序 | `packages/guard/timeout-policy/README.md` |
| timeout 在 `dsh-timeout` 库 / capability 终止 / 本策略层之间怎么分工 | `.agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.md` |
| repeat guard 的决策记录（已归档，勿当现行权威） | `.agents/notes/archived/feature/2026-07-08-repeat-tool-guard.md` |

> 本组**没有** `docs/subsystems/guard.md`，别去找。

## 2026-08-31 · hard policy可组合面与冷恢复边界

- 公开seam足以写live policy consumer：`tools/pre-execute`异步allow/deny/ask，`ctx.tools.guard()`同步单调deny，`tools/execute`只包dispatch，`tools/post-execute`变换当前结果，`tools/result`只观察final outcome。Session/per-turn/per-tool计数与canonical-args deny可在pre/guard完成；rate-limit类结果可在post/result观察后让**后续**调用deny。
- rc.2没有内建的run总预算、per-turn/per-tool hard cap、持久duplicate ledger或通用rate-limit-stop policy。官方`repeat-tool-reminder`只对连续相同调用给劝告，canonicalizer私有、chain为`WeakMap<Agent,...>`，resume必然从零开始。
- 标准日志能做条件性纯fold：root `tool/call`记录`turn/step/callId/name/raw arguments`；Code Mode的`tool/code-dispatch-start`/settle记录normalized arguments。它们可重算已记录attempt计数，但不能证明body执行或effect完成，direct `ctx.tools.execute()`也不自动写Session event。
- shipped base的checkpoint policy会在top-level body前flush已记录`tool/call`；自定义composition可合法省略，nested Code Mode dispatch也只复用outer checkpoint。故cold log不是任意ToolRuntime调用或physical effect的exact ledger。
- generic pre/guard只有deny，没有stop-turn/run action；tool-owned successful body可`concludeTurn()`，但generic policy不能通过`PreToolDecision`/`PostToolDecision`伪造该marker。并行body已启动后，迟到的rate-limit观察也不能撤回它们。
- “run”若指整个Session，可按Session fold；若指某次Host/Agent activation，rc.2没有通用durable run id/start marker。core也没有统一tool `RATE_LIMIT` taxonomy；只有具体tool把稳定code/meta写进标准result时，恢复方才有可折叠证据。
- **认知状态**：verified_inference（固定tag ToolRuntime/AgentLoop/Code Mode/Session事件/checkpoint源码与官方repeat reminder；未做crash实测）。
