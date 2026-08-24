---
title: packages/interaction — 人机协作平面
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/interaction/README.md
  - packages/interaction/user-questions/README.md
  - packages/interaction/user-approval/README.md
  - packages/interaction/commands/README.md
  - packages/interaction/permission-presets/README.md
  - packages/interaction/tool-ask-user/README.md
  - packages/interaction/user-questions/src/index.ts
  - packages/interaction/user-approval/src/index.ts
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

人类与运行中 agent 协作的那一层——提问、审批、权限预设、slash command；组 README 强调这些是 **product 包（真人驱动的真实界面）**，并且它们「通过既有 agent/session 契约集成，而不是改动 loop」[T1: packages/interaction/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

| 包名 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `user-questions` | `@deepseek-ai/dsh-user-questions` | `ctx.userQuestions` | provider-neutral 的「向人类提问/取答」seam |
| `user-approval` | `@deepseek-ai/dsh-user-approval` | `ctx.approval` | channel-neutral 的一次性审批决策 seam |
| `permission-presets` | `@deepseek-ai/dsh-permission-presets` | `ctx.permissionPresets` | 把 `sandbox/mode` + `approval/policy` 打包成用户可选的 preset |
| `commands` | `@deepseek-ai/dsh-commands` | `ctx.commands` | 人类命令（slash command）的注册与分发 |
| `tool-ask-user` | `@deepseek-ai/dsh-tool-ask-user` | 注册到 `ctx.tools` | 把「向人提问」暴露成 model-facing 的 `ask_user_question` 工具 |

## 三件套结构

本组含**两条独立的 seam**，形状不同，不要混：

- **user-questions seam**
  - Service Definition：`user-questions`（`UserQuestionService` / `ctx.userQuestions`）
  - Service Provider：**不在本组**——由 Web host runtime 提供；组内无 shipped provider
  - Consumer：`tool-ask-user`（模型侧）、`plan-mode`（`exit_plan_mode` review，跨组消费）
- **approval seam**
  - Service Definition + 实现：`user-approval`（`ApprovalService` / `ctx.approval`）
  - "Provider" 形态特殊：answerer 不是注册对象，而是 **`approval/request` waterfall listener**；返回 outcome 即应答，调 `next()` 即委派
  - Consumer：tools pipeline 的 `ask` 决策路径、sandboxed bash 的升级重试、ACP automation bridge
- `permission-presets` 与 `commands` 是**这两条 seam 之上的产品面**，不是第三条 seam：preset 只写 `sandbox/mode` 与 `approval/policy` 两个 knob 事件。

## 扩展点

依赖纪律：**扩展插件依赖 Service Definition，不依赖具体 provider**。

- 想接一个新 UI 来回答模型提问 → 依赖 `@deepseek-ai/dsh-user-questions`，实现 `UserQuestionProvider`（只需 `ask(request)`），调 `ctx.userQuestions.registerProvider(provider)`。**一个 context 只允许一个 provider**，重复注册抛 `DUPLICATE_PROVIDER`。
- 想让模型能提问 → 直接挂 `tool-ask-user`（`inject = ['tools', 'userQuestions']`），不要自己再定义一个提问工具。
- 想接管审批决策 → `ctx.on('approval/request', ...)` 作为 waterfall listener；**每个部署只组合一个终端 answerer**，sibling listener 顺序不是优先级机制。
- 想加 slash command → `ctx.commands.register(definition)`；挂在 `agent.ctx` 之下的插件自己声明 `commands` injection，会得到 agent-scoped 定义并 shadow 同名全局定义。
- 想加新的「决策呈现形态」→ `AskUserQuestionIntent`（目前只有 `{ kind: 'plan-review', approve }`）；intent **只改呈现**，不认识 tag 的 UI 回落到通用选项列表，调用方读到的答案字段完全相同。
- 想扩 preset 表 → `permission-presets` 的 `Config`；注意名为 `custom` 的表项会在 load 时抛错。

## Known Limitations

按包摘录（原文在各包 `## Known Limitations and Deferred Work`）：

- `user-questions`：**One provider per context**（无路由/fan-out；无 provider 时 `ask()` 抛 `NO_PROVIDER` 而非降级）；**词汇表只有 question-form 形状**（可选项 + 可选自由文本），file picker / diff-preview confirmation 尚无 seam 词汇。
- `user-approval`：请求**必须处在一个打开的 turn 内**；只有 one-shot grant（有 `allowed-once`，**没有 `allow-always` / 记忆规则 / 撤销 / grant store**，session policy 只有 `ask` / `never`）；请求**不携带 tool arguments**；**没有内置 answerer**，headless 部署解析为 `unavailable` 并 fail closed。
- `tool-ask-user`：pending 问题会**阻塞整个 tool call**（未声明 `timeout-policy` 预算，只随 turn 的 `exec.signal` 取消）；runtime 拥有的 subagent **不能提问**（`DELEGATED_CALLER`）；native 答案渲染成紧凑 JSON 文本。
- `commands`：只支持**非结构化文本输入**；取消是**协作式**的，dispatch 停止等待但 handler 必须自己响应 signal。
- `permission-presets`：只打包**两个 knob**；`custom` 是**只可派生、不可选中**的；preset 表是 process-level，改表要 reload plugin；settings 里存的默认 preset 若从表中移除，Permission settings 注册会失败直到 `settings.yaml` 的 `permissionPresets` 段被更新或重置。

## 陷阱

1. **组 README 表格里那一行写的是 `permission/`，但真实目录与 npm 名都是 `permission-presets`**（链接确实指向 `permission-presets/README.md`）。按 `permission` 去搜包会搜不到。
2. **「durable lineage 不是权限」**是本组反复出现的一条规则：`ask()` 通过 live `AgentRegistry` 认证**精确身份**，只接受 runtime root。带历史 delegation depth 的 session 被 resume 成新的 runtime root 后**可以**提问；而 durable depth 为 0 的 live child 若被另一 agent 拥有则**被拒**。`tool-ask-user` 与 `plan-mode` 的 exit review 共享这条边界。
3. **审批 fail closed 是默认行为，不是错误**。没有 answerer 时结果是 `unavailable`，service 自己**永远不会**去提示人类。
4. **审计事件是 log-only**：`approval/asked` / `approval/decided` 不进模型上下文；模型只看到发起方最终的 tool 结果。而 audit append 在 commit 前失败会 reject，绝不返回「未记录的决定」。
5. **command 结果不进模型历史**。`CommandResult` 由 adapter 直接渲染；registry **从不隐式**把 `rawInput` 提交给 agent——要让它变成模型输入，必须由 command producer 显式走 `Agent`（`plan-mode` 的 `/plan [message]` 就是这个模式）。
6. **`recordInput` 默认 true**；当命令有权威 domain event 拥有 payload 时才设 false，以避免 `command/run` 重复记录 `args`。
7. **preset 在 session 创建时被 pin 死**：创建时把 `permissionPresets/preset`、`sandbox/mode`、`approval/policy` 钉进该 session，之后改 settings **不会**影响已有 session。
8. **`approval/policy: never` 会写进模型可见的 runtime-context 快照**，明确告诉模型不要请求 sandbox 升级（不要设 `sandbox_permissions`）。
9. `commands` 的 `CommandRuntime` 继承的是 `TypertRemoteService`（不是普通 `Service`）——它天然是一个远程可投影的 service。

## 2026-08-22 agent 审核增量（ask 的触发链——只配 policy 不产生 ask）

- **`approval.request` 只在 `tools/pre-execute` waterfall 返回 `{kind:'ask'}` 时被调用**（`packages/core/tools/src/index.ts` 约 L1693-1698 的链路）；官方 `tool-bash` 内置的 `ctx.approval` 消费只覆盖 sandbox escalation 路径（`tool-bash/src/index.ts` 的 `approveBashEscalation`）。因此 **`approval/policy: 'ask'` 只是 ask 到来后的默认应答策略，不会让普通工具调用变成 ask**——宿主要让某类调用进 HITL，必须在第一方 bundle 注册 `tools/pre-execute` listener 显式返回 `ask`。只配 policy 是**零生效且无任何报错**的坑。
- **ask 路径三重 fail-closed**：无 approval service → deny（tools/index.ts 约 L1693-1698）；无 agent → deny（L1700-1705）；answerer 链空 / 抛错 / 返回非词汇值 → `unavailable`（user-approval/src/index.ts 约 L317-329），且 `unavailable` 在 tools 侧再折为 deny。
- **审批强制 open turn 且审计 turn-enclosed**：`ApprovalService.request` 校验 `hasOpenTurn`（约 L127-134、L257-265），`approval/asked` + `approval/decided` 必须 turn-enclosed append 进 session log——这是 `ToolExecution` 公共形状无 turn 字段时做"审批与轮次关联"的官方先例（从 `Agent.session` 的 open turn 捕获）。
- **`permission-presets` 无命令级 allowlist 存储**：`PresetSpec` 只捆绑会话级 sandbox/mode + approval/policy（workspace-write=workspace-write+ask、danger-full-access=danger-full-access+never）——官方生态中不存在"长期命令授权"的最近似物，佐证 approval seam 刻意不承担 durable grant。
- **`mode:'code'` 的 collapse 拒绝发生在 pre-execute / approval / guard 之前**（tools/index.ts 约 L1374-1381）：审批与守卫层**观察不到**被 collapse 的调用——Code Mode 部署下做审批审计需知此盲区。
- **`timeoutMs` 是 ToolDefinition 声明字段，registry 不强制执行**：实际由 `@deepseek-ai/dsh-tool-call-timeout-policy` 以 `tools/execute` wrapper 实施（tools/index.ts 约 L249-255 JSDoc）——不装 timeout-policy 包，声明等于没写。

## 2026-08-24 agent 复审增量（pre-execute 异步性、Question 线协议与 inbox 撤回 API）

- **`tools/pre-execute` 是官方支持的异步长等待 waterfall，不是"必须同步返回"的闸**：事件签名为 `(exec, next) => Promise<PreToolDecision>`，官方注释明说 "Async gates must observe `exec.signal`; the registry rechecks cancellation after they settle **but never abandons their promise**"（`packages/core/tools/src/index.ts` 约 L144-152）；`packages/core/agent-loop/src/tool-calls.ts` 约 L215 "Ordered pre-execute may await; only dispatch/body overlaps"。等待人的三个官方位置（pre-execute listener 内、approval ask、tool body）都存在——宿主把"pre-execute 不等人"作为自己的部署纪律是可以的，但**不得写成官方契约**。registry 在 ask 决策后自己 `await serviceAsk → approval.request`（tools/index.ts 约 L1479-1481）。
- **Question 的线协议与 answer 投递面**：mux 帧联合含 `question/requested{sessionId, questions}` 与 `question/resolved{sessionId, questionRpcId, outcome: 'answered'|'cancelled'}`（`apiproxy/src/api/events.ts` 约 L74-75）；rpcId 由 host 铸 `RpcId(randomUUID())`，respond 按 rpcId 路由到进程内 pending Map 的 **Promise resolve**（`api-proxy.ts` 约 L1318-1335/:3595-3638）——`UserQuestionService` 公共面只有 `registerProvider()`/`ask()`，**没有 respond 方法**；宿主 answer 走 provider promise 或 wire `POST /api/respond`。
- **自由文本 answer 官方任何层无长度/字符集 bound**：tool schema 无 maxLength、wire zod `custom: z.string().optional()` 无界、service `ask()` 只查 EMPTY_QUESTIONS/BAD_INTENT/CALLER_NOT_LIVE/DELEGATED_CALLER；唯一校验是 provider/wire 层的选项子集语义（selected ⊆ offered labels、custom 非空白）。**宿主要 bound 自由文本必须自建**（UI/answer 路径层）。
- **inbox 撤回是公共 API**：durable `MessageId`（branded string，createMessage 以 randomUUID 铸）随 `user/message`/`agent/inbox/spliced` 事件入 log；未认领消息可 `Inbox.remove(messageId)`（durable 记录取消）/`replace(id, newMessage)`/`clear()`（`packages/core/agent/src/inbox.ts` 约 L57-126）；认领后不可撤。`cancel(cause, {keepInbox: true})` 保留 pending；`agent/pre-step` waterfall 可 reject step。
- **steer 的目标是 next-step（最近 step 边界 + wake），不是 next-turn**；仅 abort 后的 waking 输入 reclassify 为 next-turn；`followup` 才是 next-turn（"sole ordinary message of its own turn"）。宿主 admission 拦截额外输入无官方钩子（天然是宿主 carrier 行为），但撤回/保留有上述 inbox API 支撑。
- **两个易混的部署配置各归其包**：`AgentLoop.Config.maxParallelToolCalls`（`packages/core/agent-loop/src/index.ts` 约 L253-260，默认 10，`1`=串行；只作用于 opt-in `isConcurrencySafe` 的并行组，exclusive call 是 barrier）≠ `ToolTodo.Config.allowParallelInProgress`（`packages/todo/tool-todo/src/index.ts` 约 L37-42，**必填** boolean，"a deployment choice, not a fixed rule"）。
- **code-mode 不是独立包**：是 `dsh-tools` 内建的 presentation mode（`tools` 配置 `mode: 'native'|'code'|'both'`，默认 native）+ 可选 `packages/code-runtime` 组合；不设 code mode 时 `run_code` 不出现。描述为"注册/不注册 code-mode 包"不准确。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 本组整体与 ctx key 表 | `packages/interaction/README.md` |
| 提问 seam 的 API/类型/intent 规则 | `packages/interaction/user-questions/README.md`、`src/index.ts`（`UserQuestionService`、`UserQuestionError`） |
| 审批 outcome 词汇、policy、waterfall 契约 | `packages/interaction/user-approval/README.md`、`src/index.ts`（`ApprovalService`）、`docs/subsystems/approval.md`（含生成的 Cordis surface 区块） |
| 命令注册/分发/`command/run`+`command/done` 生命周期 | `packages/interaction/commands/README.md`、`docs/subsystems/commands.md`、`.agents/notes/implemented/feature/2026-07-19-plugin-command-registration.md` |
| preset 表、settings namespace、session 投影 | `packages/interaction/permission-presets/README.md`、`docs/subsystems/permission-presets.md`、`.agents/notes/implemented/feature/2026-07-06-sandbox.md` |
| `ask_user_question` 的 schema 与结果形状 | `packages/interaction/tool-ask-user/README.md`、`docs/tool-catalog.md` |
| 提问 seam 设计取舍 | `docs/subsystems/user-questions.md` |
| 审批 seam 设计取舍 | `.agents/notes/implemented/feature/2026-07-06-approval-seam.md` |
| 自动化侧（非人类）的等价通道 | `packages/acp/README.md`、`packages/sdk/README.md`、`packages/boot/README.md` |
