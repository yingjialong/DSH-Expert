---
title: 子系统路由地图（阶段一 20 篇）
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - docs/subsystems/README.md
  - docs/subsystems/core.md
  - docs/subsystems/llm-streaming.md
  - docs/subsystems/persistence.md
  - docs/subsystems/filesystem.md
  - packages/README.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

# 子系统路由地图

上游 `docs/subsystems/` 共 **51 篇**（见 `docs/subsystems/README.md` 的 Owns 表），本页只覆盖阶段一分配的 **20 篇**，补的是上游 README 没有的两列：**ctx 服务名** 和 **packages 组**。上游 README 已经给了每页的 "Owns" 一句话，别在这里重抄。

## 先看这三条判定规则（跨文档综合，单篇里没有）

1. **是不是 agent-loop spine？** 只有六个包是：`session/ system-prompt/ tools/ agent/ agent-loop/ scope/`，全在 `packages/core/`，由 `docs/subsystems/core.md#the-spine-package-by-package` 列表定义。其余子系统页几乎都会在开头自称 "**one optional capability, not part of the agent-loop spine**"（compaction / code-runtime / lsp / plan / permission-presets / client-modules 逐字如此）。这句话就是判据：出现它 = 可以不装，装了也不改 loop。
2. **想找扩展点？先找 capability seam 三角**：Service Definition（定义 `ctx.X` 抽象类）/ Service Provider（后端）/ Consumer（模型可见的 tool 或 command）。compaction、lsp、code-runtime、filesystem、persistence 页都按这个三角组织，包名通常是 `dsh-X` / `dsh-X-<backend>` / `dsh-tool-X`。要换后端就实现 Provider，不要改 Definition。
3. **想加类型变体？** 两个 repo-wide 模式只在 `core.md#repo-wide-type-patterns` 讲一次：`…Map → derived-union`（declaration merging 加变体，六张规范表：`ContentBlockMap` / `MessageSourceMap` / `FinishReasonMap` / `TurnTriggerMap` / `TurnEndReasonMap` / `SessionEventMap`）和 `Branded<B>`（来自 `packages/util/brand`，纯类型包，无运行时）。其他 20 页里的 branded id 全部反向链回 core.md。

## 路由表

| 子系统页 | ctx 服务（页内 `## Cordis API` 标题） | packages 组 | 什么问题来这查 |
|---|---|---|---|
| `core.md` | `ctx.agents` `ctx.agentLoop` `ctx.agentPresets` `ctx.agentDefaultModel` | `core/`（+ `preset/agent-presets`） | Agent 怎么被 create/resume、`AgentHandle.dispose()` 的 capability 语义、`AgentSetup` 事务回滚、send/followup/steer/inject 路由、branded id 与 …Map 模式 |
| `llm-streaming.md` | `ctx.llm` — `LlmRuntime` | `llm/` | `Message`/`ContentBlock`、`StreamChunk` 闭合联合、`LlmAdapter` 必须遵守的 8 条契约、`LlmFailure` 归一化、`ResolvedRetryPolicy`、`ReplayEnvelope`、`PreparedLlmCall` |
| `persistence.md` | `ctx.sessionPersistence`（abstract seam） | `session/` | flush 检查点与批写窗口、崩溃后 `turn/end{interrupted}` 修复、`SessionHeader`、`SESSION_FORMAT_VERSION` 拒读规则、JSONL vs SQLite 后端差异 |
| `filesystem.md` | `ctx.fs` — `FileSystem`（abstract seam） | `fs/` | `FsTarget`、read/write/edit 结果、observed-file 状态、`FsErrorCode` 13 个码的分工、为什么文件 IO 没有 `timeoutMs` |
| `compaction.md` | `ctx.compaction` `ctx.toolResultPruner` | `compaction/` | `compaction/*` 三个 log-only 事件与那把"锁"、`CompactionResult`、`ManualCompactionErrorCode`、tool-result pruning、tool-call/result 配对边界 |
| `jobs.md` | `ctx.jobs` — `JobRegistry`（abstract seam） | `jobs/` | `JobId`/`JobKindMap`/`JobStatus`、producer 契约、owner 授权与 `maxConcurrentJobsPerOwner`（默认 10） |
| `approval.md` | `ctx.approval` — `ApprovalService` | `interaction/` | `ApprovalOutcome` 四态与 fail-closed、`ask`/`never` policy、`approval/asked`+`approval/decided` 审计对、answerer waterfall |
| `permission-presets.md` | `ctx.permissionPresets` | `interaction/`（读 `sandbox/sandbox-policy`） | preset 表 = sandbox mode + approval policy 的捆绑、派生态 `custom`、`optionOf`/`names` |
| `commands.md` | `ctx.commands` — `CommandRuntime` | `interaction/` | 斜杠命令注册/发现/直接执行、`CommandDescriptor`、`ParsedCommand`、`CommandInputDescriptor.images` |
| `plan.md` | `ctx.planMode` — `PlanModeController` | `plan/` | `plan/mode` log-only 状态与 fold 恢复、pending selection 何时落盘、`plan:policy` prompt section（order 50）、`exit_plan_mode` 审阅弧 |
| `credentials.md` | `ctx.credentials` — `CredentialProvider`（abstract seam） | `credentials/` | 配置里只放 `CredentialRef`（环境变量名）不放值、per-operation 解析 = 热更新机制、`CredentialInfo.writable`、`credentials/reference-updated` 与 `credentials/record-updated`（0.1.1-rc 由 `credentials/updated` 拆分；records 空间配合 `ctx.authorization` flow） |
| `lsp.md` | `ctx.lsp` — `LspService` | `lsp/` | 四个导航操作与坐标系、`LspProvider` 扩展名独占注册、`LspError` 稳定码 |
| `code-runtime.md` | `ctx.codeRuntime` — `CodeRuntime`（abstract seam） | `code-runtime/` | `CodeRunRequest`/`Result`、bindings 作为程序全局、`CodeRunFailure` 六种正交失败 |
| `attachment.md` | `ctx.attachments` — `AttachmentStore`（abstract seam） | `attachment/` | 图片内容寻址引用、persist-before-event 规则、`validateImage` 全量先验、`<DSH_HOME>/attachments/v1` |
| `goal.md` | `ctx.goals` — `GoalService` | `goal/` | 同会话目标的 `GoalRef` revision CAS、`goal/change` 事件重放、激活与轮次归属 |
| `feedback.md` | `ctx.messageFeedback` | `feedback/`（+ `client/ui-message-feedback`） | 逐条 assistant message 的可编辑评价、`ifVersion` 乐观并发、storage-domain sidecar、**一整节 Boundaries and limitations** |
| `agent-team.md` | `ctx.agentTeams` — `TeamService` | `experimental/`（未发布） | Team 花名册快照、durable mailbox（queued-minus-delivered）、共享任务 DAG 的 revision CAS、`foldTeam()` |
| `client-modules.md` | `ctx.clientModules` | `client/`（消费 `host/webserver`） | `dsh.client` 声明扫描、`__DSH_BOOT__` 引导图（0.1.1-rc 起改为 `globalThis["__DSH_BOOT__"]` 的 global injection row，经 `webserver/index-inject` 事件收集，不再由 index tap 注入首个 script）、`/plugins/<id>/client.js` 路由、`rebuilt()` 与 HMR |
| `extensions.md` | `ctx.cordisInspect` `ctx.dynamicCordisRunner` | `extensions/` | agent 自我改造：运行时插件/服务检视、动态挂载卸载。**注意：本页几乎只有生成的 Cordis API，没有正文**，设计要看 `packages/extensions/README.md` |
| `invariants.md` | `ctx.invariants` — `InvariantRegistry` | `runtime-diagnostics/`（**未在 packages/README.md 组表中**） | 包自有运行时不变量的注册表、`InvariantInstaller`/`InvariantFailure`、每包 `./invariant` 伴生插件的"空伴生"契约 |

## 服务名 → 页面 反查

`agents`/`agentLoop`/`agentPresets`/`agentDefaultModel`→core，`llm`→llm-streaming，`sessionPersistence`→persistence，`fs`→filesystem，`compaction`/`toolResultPruner`→compaction，`jobs`→jobs，`approval`→approval，`permissionPresets`→permission-presets，`commands`→commands，`planMode`→plan，`credentials`→credentials，`lsp`→lsp，`codeRuntime`→code-runtime，`attachments`→attachment，`goals`→goal，`messageFeedback`→feedback，`agentTeams`→agent-team，`clientModules`→client-modules，`cordisInspect`/`dynamicCordisRunner`→extensions，`invariants`→invariants。

## 页面本身的可信度分级（重要，决定你敢不敢直接抄类型）

每页底部 `## Cordis API` 段落夹在 `<!-- BEGIN GENERATED cordis-surface -->` 标记之间，由 `scripts/gen-cordis-catalog.ts` 从源码生成，`pnpm run doc-sync` 里的 `verify-cordis-catalog` 校验新鲜度 —— 这部分等价源码。正文里的 ` ```ts type-equiv ` 代码块由 `pnpm run verify-type-equiv` 与源码做漂移检查，也等价源码。

**但两个例外必须回源码看**：
- ` ```ts ignore-check ` 块不参与校验（`core.md` 的模式示意、postmortem 里的片段都是这种）。
- **declaration merging 进别的包的变体不在 type-equiv 覆盖内** —— `compaction.md` 明说：`compaction/*` 三个事件写在 `declare module '@deepseek-ai/dsh-session/types'` 里，`verify-type-equiv` 的提取器只匹配顶层具名声明，所以那三行只能以表格形式登记，字段以源码为准。同理，任何 `…Map` 的插件扩展变体在文档里都可能落后于源码。

## 分组归属的坑

`packages/README.md` 的组表（48 行）是权威分组清单，但盘上实际有 **53 个组目录**：`runtime-diagnostics/` 和 `mcp/` 都存在却不在表里（`invariants.md` 正指向 `packages/runtime-diagnostics/invariants`，而该组连 README.md 都没有）。根 `AGENTS.md` 的 Repository layout 更旧：它写的 `self-modification/` 实际叫 `extensions/`，`support/` 实际叫 `test-support/`，并且漏掉十几个组。**认组名以 `ls packages/` 为准，认职责以 `packages/README.md` 为准，别信根 AGENTS.md 的目录树。**
