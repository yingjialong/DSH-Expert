---
title: packages/hooks — Claude Code / Codex hook 桥接 + 共享线协议库
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/hooks/README.md
  - packages/hooks/hook-protocol/README.md
  - packages/hooks/hook-protocol/src/types.ts
  - packages/hooks/hook-protocol/src/index.ts
  - packages/hooks/hooks-claude-code/README.md
  - packages/hooks/hooks-claude-code/src/index.ts
  - packages/hooks/hooks-codex/README.md
  - packages/hooks/hooks-codex/src/index.ts
  - docs/persistence-catalog.md
  - .agents/notes/implemented/feature/2026-06-30-interception-extension-points.md
  - .agents/notes/implemented/feature/2026-06-30-hook-bridges.md
  - .agents/notes/implemented/feature/2026-06-30-hook-protocol-lib.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

让用户把**已有的** Claude Code / Codex `hooks.json` 直接指给一个 bridge 插件，那些外部 shell hook 就会在 harness 自己的**类型化拦截点**上忠实运行。三个包 = 一个共享协议库 + 两个方言桥。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包目录 | npm 名 | 形态 | 一句话职责 |
|---|---|---|---|
| `hook-protocol/` | `@deepseek-ai/dsh-hook-protocol` | **library（非插件）** | matcher、exit-code/stdout 解码、`runHook`、最严合并、`hook/*` 事件、detached 静默 |
| `hooks-claude-code/` | `@deepseek-ai/dsh-hooks-claude-code` | plugin | CC 方言：CC 形状 payload、env 与 `${CLAUDE_*}` 替换、7 个 hook 点的 Decision 映射 |
| `hooks-codex/` | `@deepseek-ai/dsh-hooks-codex` | plugin | Codex 方言：snake_case payload、纯正则 matcher、5 个 hook 点 |

## 三件套结构

**这一组不是 capability seam，也没有服务。** 组 README 与 `docs/capability-seams.md` 里都找不到本组的 ctx key。正确的心智模型是：

- **真正的扩展面（canonical extension surface）是 harness 自己的类型化拦截点**：`agent/session-start`、`agent/pre-step`、`tools/pre-execute`、`tools/post-execute`、`agent/turn-stopping`、`subagent/start`、`subagent/end`。所谓「native hook」就是挂在这些点上的普通 Cordis 插件。
- **本组三个包只是 bridge**：把外部 shell-hook 协议翻译到同一张扩展面上。
- **`hook-protocol` 是共享库**，它自己注册什么都不注册、inject 什么都不 inject；两个 bridge import 它，避免各写一遍相同的协议半边。

Codex 刻意重实现了 CC hook 协议的一个**子集**（同样的 `hooks.json` matcher-group 形状、同样的 exit-code/stdout 输出契约、同样的 command-hook 执行模型），所以真正共享的部分才被抽到库里，各自只留方言差异。

## 扩展点

**要新增拦截行为，答案几乎总是「写一个 native Cordis 插件」，而不是加 hook 方言。** 两个 bridge 的 README 都写了同一句话：native 插件能做 bridge 能做的一切，而且更强（类型化返回、无序列化边界）；**bridge 存在的唯一理由是兼容已映射的 command-hook 子集。**

若确实要写第三种方言桥，可复用的 primitive（全在 `hook-protocol`）：

| primitive | 作用 |
|---|---|
| `matcherDiagnostic(matcher, mode)` / `matchesMatcher(matcher, query, mode)` | 解析期诊断 + 运行期受控匹配。`claude` mode 把纯 `[A-Za-z0-9_\|]+` 当字面量（竖线 = 精确匹配的 alternation），其余当正则；`codex` mode 永远是无锚定正则 |
| `runHook(bash, hook, options, now)` | 经 `ctx.shell` 执行：转发**必填**的 `options.signal`、序列化 stdin payload、在 executor 的凭证擦除**之后**合并 `options.env`、遵守 `timeoutSec` 否则用 `options.defaultTimeoutMs`。**永不抛异常**——executor rejection 会变成 `exitCode: undefined` 的非阻塞 `HookOutput` |
| `parseHookOutput(exitCode, stdout, stderr, expectedEventName?)` | exit 2 = 用 stderr 阻塞；其他失败非阻塞。匹配的 hook-specific 权限决策覆盖 legacy 顶层决策 |
| `mergeHookOutputs(outputs)` | 折叠同一点上所有命中的 hook：权限优先级 **deny > ask > allow**，`continue:false` 首次出现即黏滞，block reason 以 `\n\n` 连接，`additionalContext`/`systemMessages` 按序累积 |
| `createDetachedRuns()` | emit 形状的点是 detached 运行（无扩展点 await 它们）。bridge 把 `drain()` 注册为 effect disposer：drain 先 fire abort signal（**杀掉仍在跑的 hook 进程，而不是等它跑满 timeout**），再等所有链 settle |
| `appendHookInvoked` / `appendHookResult` | 写 `hook/*` session 事件 |

`hook/*` 是**声明合并进 `SessionEventMap` 的 log-only 事件**（像 `compaction/*` 一样，**不是** `SurfaceEventType`，无 `surfaceOp`）：`hook/invoked` 与 `hook/result`，按 `handlerId` 配对。

## Known Limitations

组 README 无该章节；以下按包汇总。**注意两个 bridge 的 Known Limitations 章节异常详尽，是本仓最长的之一——排查「我的 hook 为什么没生效」应当直接去读原文。**

**hook-protocol**
- **`HookOutput.updatedInput` 会被解析但不被采纳**——输入重写是个延后的一致性设计问题，bridge 遇到会记日志 + 警告。

**hooks-claude-code**（只映射 CC 当前 30 个 hook 事件中的 7 个）
- **23 个事件不支持**：`Setup`、`InstructionsLoaded`、`UserPromptExpansion`、`MessageDisplay`、`PermissionRequest`、`PostToolUseFailure`、`PostToolBatch`、`PermissionDenied`、`Notification`、`TaskCreated`、`TaskCompleted`、`StopFailure`、`TeammateIdle`、`ConfigChange`、`CwdChanged`、`FileChanged`、`WorktreeCreate`、`WorktreeRemove`、`PreCompact`、`PostCompact`、`SessionEnd`、`Elicitation`、`ElicitationResult`。这些事件的配置在 group 解析**之前**就被忽略。
- 支持的 7 个也**全是 partial**：`PreToolUse` 的 `allow` 不做预批准、`defer` 不支持、`additionalContext` 被忽略；`PostToolUse` 的 `updatedToolOutput`/`updatedMCPToolOutput` 不支持且 `tool_response` 被压平成文本；`Stop` 的 `stop_hook_active` 恒为 `false` 且**连续阻塞上限未实现**（`TODO(stop-loop-guard)`）；`SubagentStart`/`Stop` 的 `agent_type` 恒为 `general-purpose` 且用的是**子 session id**（CC 报的是父 session）。
- 通用字段 partial：`systemMessage` 记日志但不呈现；`{"continue": false}` 被记录但**不会中止运行**；`suppressOutput`/`stopReason`/`terminalSequence` 不生效。
- handler 支持 partial：**只跑 shell 形式的 command handler**，`http`/`mcp_tool`/`prompt`/`agent` 一律跳过；`args`/`async`/`asyncRewake`/`shell`/`if`/`once`/`statusMessage` 均不遵守；**命中的 handler 串行执行且不去重**（CC 是并行 + 去重）。

**hooks-codex**（只映射 Codex 当前 10 个点中的 5 个）
- **5 个不支持**：`PermissionRequest`、`PreCompact`、`PostCompact`、`SubagentStart`、`SubagentStop`——配置在解析期**静默丢弃**。
- `PreToolUse` 只能 block，**每个工具都被表示成 `tool_input: { command }`**，所以非 shell 工具的参数不能忠实暴露给 hook。
- 每个映射事件报的是**静态配置的 `model`** 和 `permission_mode: "default"`，不是运行时真值。

## 陷阱

1. **`configPath` 是进程级、加载时解析一次的**。相对路径按**进程启动 cwd** 解析，**没有 per-session（`session/new.cwd`）配置发现**（两个 bridge 都标 `TODO(per-session-hook-config)`）。CC 的分层 project/user/plugin/policy 发现与热重载都未实现。
2. **但 hook 进程本身跑在 agent 的 session workspace 里**——agent-scoped 的点会把 session 的 `cwd` 作为 hook 进程工作目录。「配置按启动 cwd 解析、执行按 session cwd」这两件事要分开记。
3. **读/解析失败是被容纳的**：bridge 记一条 warning 然后什么都不注册，**不会让 boot 崩**（打错路径不该拖垮 agent）。所以「hook 完全没反应」的第一嫌疑是日志里那条 warning。
4. **默认超时 10 分钟**（`DEFAULT_HOOK_TIMEOUT_MS`）。CC 的 `UserPromptSubmit` 官方是 30 秒事件级超时，bridge **不遵守**，除非你显式覆盖 `defaultTimeoutMs`。
5. **`SessionStart` 拿不到 `hook/*` 记录**。hook 调用/结果记录必须落在一个已打开的 turn 内；`SessionStart` 跑在第 1 个 turn 之前，它允许的 context 会挂在 inbox 里直到某次唤醒投递打开 turn。
6. **detached 点 = 无人 await**：`SessionStart`（两个 bridge）与 `SubagentStart`/`SubagentStop`（CC）都是 detached，所以**注入的 context 可能赶不上第一次请求**（`TODO(session-start-gating)`）。
7. **注入的 context 带显式 plugin source**（`{kind:'plugin', plugin:'hooks-claude-code'}` / `'hooks-codex'`），durable message 永远不会被误当成用户 prompt。
8. **无条件阻塞的 `Stop` hook 会让每一步都强制续跑**，因为连续阻塞上限没实现——hook 自己必须自限。
9. `transcript_path` 由 `ctx.sessionPersistence.locate(session.header)` 解析；**查不到时 CC 桥发 `''`、Codex 桥发 `null`**（保 Codex 的 `string | null` 形状）。查找不会创建也不会 flush artifact，所以首个 turn-end checkpoint 之前可能没有路径，也可能不含当前打开的 turn。

## 去哪深入（文件路由）

| 问题 | 去哪 |
|---|---|
| 共享 vs 方言的分界表 | `packages/hooks/hook-protocol/README.md`（`## What's shared (here) vs. per-dialect`） |
| 五个 primitive 的确切语义 | 同上 `## Primitives`；类型看 `packages/hooks/hook-protocol/src/types.ts` |
| `hook/invoked` / `hook/result` 的载荷 | `docs/persistence-catalog.md`（生成的日志事件目录） |
| CC 的 7 个 hook 点 → Decision 映射表、config 字段 | `packages/hooks/hooks-claude-code/README.md` |
| Codex 的 5 个点映射表、方言差异清单 | `packages/hooks/hooks-codex/README.md` |
| 「native hook 就是普通插件」的原始论证 | `.agents/notes/implemented/feature/2026-06-30-interception-extension-points.md` |
| 桥与库的拆分决策 | `.agents/notes/implemented/feature/2026-06-30-hook-bridges.md`、`…-hook-protocol-lib.md` |
| `updatedInput` 为何延后 | `.agents/notes/proposed/feature/2026-06-30-pre-tool-input-rewrite.md`（**proposed，非现行**） |

> 本组**没有** `docs/subsystems/hooks.md`。子系统级说明分散在组 README 与上面两条 Agent Note 里。
