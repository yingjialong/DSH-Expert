---
title: packages/subagent — subagent capability family
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/subagent/README.md
  - packages/subagent/subagent/README.md
  - packages/subagent/subagent/src/index.ts
  - packages/subagent/subagent/src/descriptor.ts
  - packages/subagent/subagent/src/continuation.ts
  - packages/subagent/tool-subagent/README.md
  - packages/subagent/tool-subagent-report/README.md
  - docs/subsystems/subagent.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

一个 agent 把活委派给子 agent 的能力族：`ctx.subagents` 是**多 provider 并存**的注册表 + 共享请求/结果契约 + 持久 descriptor + 可续期子代（continuable children）编排；子代跑在本进程、别的进程还是远端由 provider 决定。全组 11 个包，是这批里最大的一组。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包名 | npm | 一句话职责 |
|---|---|---|
| `subagent` | `@deepseek-ai/dsh-subagent` | Service Definition：provider 注册、委派、续期、descriptor、深度词汇 |
| `subagent-in-process-driver` | `@deepseek-ai/dsh-subagent-in-process-driver` | 两个进程内 provider 共享的 run driver（**不注册 provider**） |
| `subagent-spawn-in-process` | `@deepseek-ai/dsh-subagent-spawn-in-process` | Provider：进程内全新子代，不带父对话历史 |
| `subagent-fork-in-process` | `@deepseek-ai/dsh-subagent-fork-in-process` | Provider：进程内子代，seed 为父的**已完成**轮次 |
| `subagent-acp` | `@deepseek-ai/dsh-subagent-acp` | Provider：新起子进程，按 Agent Client Protocol 驱动任意 ACP agent |
| `subagent-dsh-sdk` | `@deepseek-ai/dsh-subagent-dsh-sdk` | Provider：起一个完整的 DSH runtime 子进程，走 stdio JSON-RPC SDK |
| `subagent-codex` | `@deepseek-ai/dsh-subagent-codex` | Provider（可选 Profile Bundle）：真实 Codex `app-server --stdio` 子代 |
| `subagent-claude-code` | `@deepseek-ai/dsh-subagent-claude-code` | Provider（可选 Profile Bundle）：走官方 Claude Agent SDK 的子代 |
| `tool-subagent` | `@deepseek-ai/dsh-tool-subagent` | Consumer：绑定某一个 provider 的模型侧委派工具 |
| `tool-subagent-control` | `@deepseek-ai/dsh-tool-subagent-control` | Consumer：全局 `send_message` / `interrupt_agent` / `list_agents` |
| `tool-subagent-report` | `@deepseek-ai/dsh-tool-subagent-report` | Consumer：**只在子代 scope 内**存在的 `report` 回传通道 |

## 三件套结构

- **Service Definition**：`dsh-subagent`，ctx key `ctx.subagents`，`class SubagentRuntime extends Service` [T1: packages/subagent/subagent/src/index.ts#SubagentRuntime]；续期编排在 `class SubagentContinuationManager` [T1: packages/subagent/subagent/src/continuation.ts]。
- **Service Provider**：6 个（spawn / fork / acp / dsh-sdk / codex / claude-code），**多个可同时以不同 name 共存**，这是本组与 shell 那种「只能挂一个」的关键差异。
- **Consumer**：3 个工具包，方向分得很清——`tool-subagent` 委派，`tool-subagent-control` 是父→子，`tool-subagent-report` 是子→父，三者相互独立安装。

## 扩展点

- 想接新的子代运行方式：依赖 **`@deepseek-ai/dsh-subagent`**，实现 `SubagentProvider` 并 `ctx.subagents.registerProvider(provider)`（按 name 注册，重名 fail loud；注册是 effect-scoped——注销只阻止新的 start，**不撤回已经交给调用方的 run**）。
- **能力靠 `provider.capabilities` 显式广告**，因为服务必须在创建子代之前就拒掉不支持的 one-shot 请求：`outputSchema`、`depthLimit`、`toolFilter`、`persona`。不广告就等于该 provider 不支持，服务直接 reject。
- **续期能力靠方法存在性检查**：可选的 `SubagentProvider.prepareContinuable?()`，有它才能 `startContinuable`。它只返回一个游离的 `ContinuableCreateSpec`（`{ seed? }`）——**是数据，不是能力**：不带 Agent、`AgentHandle`、prompt 投递、结果、disposal 或 resume 操作，因为身份预留、组合、Agent 创建、prompt 投递、cold resume、所有权与 disposal 全归 continuation manager。
- 想给每个 continuable 子代注入部署级能力：`registerContinuableSetup(contribution)`，对常驻子代**立即撤销**、授予则要等下一次 Activation（`tool-subagent-report` 就是这么做的，所以它注册的是 child-scoped setup 而不是全局工具）。
- 进程内子代的组合**只有一个入口**：`applyChildComposition(childCtx, parent, composition)`，它先 join 父的 agent-preset 组合，再应用子代自己的 persona 与 tool filter。把 parent 作为参数是刻意的——让「不 join 就组合子代」在调用点上**不可表达**。
- 持久 descriptor 词汇归 Definition（`src/descriptor.ts`）：`snapshotSubagentDescriptor()` 在 provider 干活前校验并 detach，`foldSubagentDescriptor()` 从子代日志恢复。该事件是 **log-only**：无 `surfaceOp`、不进模型历史、compaction 之后仍在 append-only 日志里。
- 深度词汇也归 Definition：`AgentOptions.subagentDepth`、`assertSubagentMaxDepth`、`delegationDepthOf(agent)`。持久化的 `SessionHeader.delegationDepth` 是**权威且单调**的——运行期选项只能加深不能降低，resume 的子代不会被重新算成 top-level。

## Known Limitations

- `dsh-subagent`：ACP 子代仍是 one-shot 且**不可 trace 枚举**（父的 session corpus 里没有本地子 session）；**无 host-user 续期**——`followup()` 要求恰好是活的直接父，只有 `interrupt()` 接受持久父地址权威；续期消息**永不 steer**，只入队后续轮次；取消收敛期存在 wake gap（Issue #1838 拥有 agent-loop wake latch）；Activation 收件箱与所有权图是**进程本地**的；已接受但未落日志的消息**不会重放**；report **没有持久信箱**；生命周期事件只可观测，不能影响 run。
- `tool-subagent`：后台 run 的结果**不从这个工具返回**——one-shot 走通用 task 面，continuable 子代的输出留在它自己的 session 里，settlement notice 只说明它怎么结束的；等待中的 one-shot 实例**重名检测偏晚**（`TODO(subagent-dup-toolname)`）；子代策略按实例固定，换 model/persona/tool filter/深度上限都要另起一个不同名的工具。
- `tool-subagent-control`：入队消息**没有独立结果**，只返回收件箱 `messageId`；**不能 steer 当前轮**；listing 是快照不是投递承诺（跨进程准确性需要共享 lease，但 `interrupt_agent` 自己做权威 live-lineage 检查，所以发现结果过期不会授权）；**无分页无删除**。
- `tool-subagent-report`：接受**弱于持久投递**（无信箱、无幂等键、无回执、无重试、无 exactly-once）；父的 host-owned disposal 已开始时仍可能接受（`AgentHandle.dispose()` 不暴露「disposal 已开始」信号）；**嵌套上报只向上一跳**，孙代报给它的直接父，不会直达顶层协调者；**无速率限制**（默认 `next-step` 模式在嵌套子代频繁上报时会放大模型工作量，接受不读的部署应选 `quiet`）。
- `subagent-fork-in-process`：seed 是**一次性快照**；**没有任何出厂 composition 会创建 continuable fork 子代**——`prepareContinuable` 实现着、seam 也接受，但每个出厂 `cordis.yml` 都给 fork 委派工具设了 `backgroundMode: one-shot`，重开需要子代的 system prompt 与 tool schema 与父**逐字节相同**，而 `report` 返回通道当前正好破坏这一点。
- `subagent-codex` / `subagent-claude-code`：每 run 一个全新进程/查询，**无续期、无 resume、无池化、无进度流**；provider 名与工具绑定由 Profile 行固定，调用时**不能动态选 provider**；认证与账号状态保持原生（Bundle 只提供 CLI，不登录不改配置）；平台原生 payload 在委派时必需，**无 host-CLI 回退**；**无人工审批路径**（未知请求 fail closed）；助手载荷只有最终文本；**shared service 会拒绝**这两个 provider 的 outputSchema / persona / tool filter / 深度强制；无 wall-clock 超时与副作用回滚。Codex 的兼容性**由开发期证据钉住**（已验证的 0.147.0 协议基线）。
- `subagent-dsh-sdk`：每 run 一个全新 runtime 进程（启动整棵插件树，比 ACP 子代贵）；父**不能**强制子进程的 outputSchema/深度/tool filter/persona，只能改子代自己的 `cordis.yml`；子代 transcript 留在**子代自己的 session root**。
- `subagent-in-process-driver`：run **不暴露 `sendMessage`/`resume`**；结构化捕获只接受 `defineTool` 的 schema 子集。

## 陷阱

- **组 README 表格里的包名 `subagent-inprocess/` 与真实目录/包名不一致**：目录是 `subagent-in-process-driver/`，npm 名是 `@deepseek-ai/dsh-subagent-in-process-driver`（表格里的链接指向正确目录，只有显示名错）。
- Codex 与 Claude Code 是**独立可选 Profile Bundle**，装上后每个包只注册**休眠**的 Host provider。要真正给模型一个工具，得复制一份完整 Agent Preset、把对应 tool 行的 `disabled` 去掉、再开新 Session。移除某个包只在**下次 Profile 启动**时撤回该 provider 与其私有运行期闭包。
- `start()` 与 `startContinuable()` 的失败语义不同：`start()` reject 表示 provider 已清理完所有未发布的启动资源；发布之后的轮次或基础设施故障则通过 run 结算。`startContinuable()` 在**收件箱接受初始 prompt**时就 resolve `{ childId, messageId }`，不等轮次开始、也不等消息落 Session 日志。
- continuable 的 caller signal **只管到收件箱接受为止**，之后 Activation 归 manager 独立所有——调用方再取消既不会取消已接受的轮次，也不会 dispose 子代。
- `interrupt()` 与 `drainContinuableChildren()` 长得像但语义相反：前者保留未认领的收件箱工作、Activation 与已发布后代（准入同步、效果异步）；后者是 teardown，**不保留**待处理收件箱工作。
- `inheritsParentContext` 是**描述性而非可强制**的：它只说子代能否看到父已完成的对话（fork 能，spawn 与出进程 one-shot 不能），**不表示**工具、服务或权限的继承。
- 同进程的请求、descriptor、结果、事件载荷都是**借用的不可变可信值**，服务不克隆不冻结；序列化与敌意输入校验属于真实的进程/worker/持久化/模型边界。
- 委派深度看的是持久 header 的 `delegationDepth`，不是 descriptor——descriptor 刻意省略了 `subagentDepth` 与 `outputSchema`。

## 2026-08-22 agent 审核增量（delegation 边界与 child identity）

- **`delegation policy: never` 的精确语义是"每个 ask 自动 resolve `rejected`"**，不是"ask 无人应答"：`user-approval/src/index.ts`（约 L90 注释 "every ask resolves rejected"、L312 dispatch 前确定性拒绝）。官方测试 `inheritance.spec.ts` L213「rejects a child escalation deterministically even when an answerer would allow it」背书：**即使宿主配置了会放行的 answerer，child 的 ask 仍被确定性拒绝**，且审计对 `approval/asked` + `approval/decided` 仍然落 log。宿主若想让 child 的审批到达自己的 answerer，唯一路径是事后 `setPolicy()`（last-event-wins），而：
- **`ApprovalService.setPolicy()` 有模型可见副作用**（`user-approval/src/index.ts` 约 L226-239）：切换 policy 时会向该 agent 注入一条 source 为 `plugin:user-approval`、文案 "changed by the user" 的 user message 并排队到下一个 model step——child transcript 会出现这条消息，直接影响宿主的审计投影与 envelope 绑定。
- **每个 in-process child 都被注入 `SUBAGENT_DELEGATION_CONTEXT`**（`subagent/subagent/src/child-agent.ts` 约 L134-139）：模型面声明「权限范围在启动时固定、无法从 session 内部扩大、需审批的操作自动拒绝」；runtime-context order 120，排在 `sandbox:policy`(110) 与 `approval:policy`(115) 之后——never 是模型侧与执行侧的双半配套。
- **continuable child 的官方 SessionId 在首个 prompt 前就可取得**：continuation manager 的 `ContinuableCreateRequest.sessionId` 注释明确 "The manager has already reserved the durable child identity"（`types.ts` 约 L160-169）——child identity 由 manager 预留而非 provider 铸造，结合 `SessionHeader.parentSession` 的 durable lineage，父/宿主在 child 首 prompt 前即可绑定其官方 SessionId。
- **out-of-process run 的 id 不保证等于 child 官方 SessionId**：`SubagentRun.id` 对 remote provider 是 parent namespace 内 mint 的 id（`types.ts` 约 L259-263），`localAgent` 为 undefined；「run id == child 进程内官方 SessionId」仅 local run 有 MUST 保证。跨进程 provider（ACP/SDK/Codex/Claude Code）集成需另行确认 id 映射，官方 lineage 断言只对 in-process 成立。
- **rc.8 → rc.2 本组唯一 src 行为变化**是 `projection.ts` 的 projection API 迁移（`schema`→`stateSchema` + `SessionProjectionStateMap` 注册 `subagentTiming`/`subagent` 两个 state key）；delegation/continuation/driver 三个环节零 diff——rc.8 的 delegation 结论在 rc.2 可安全互证。

## 去哪深入

- 组结构、11 包分工、Profile Bundle 安装步骤 → `packages/subagent/README.md`
- `SubagentRuntime` 全量 API 表、capabilities、durable descriptor、深度、one-shot 与 continuable 生命周期、settlement 投递、collection model → `packages/subagent/subagent/README.md`
- 续期编排实现 → `packages/subagent/subagent/src/continuation.ts`
- 子代枚举（`listChildren` / `listDescendants` 的依赖与排序）→ `packages/subagent/subagent/src/list-children.ts`
- 子系统参考 → `docs/subsystems/subagent.md`
- 设计决策 → `.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md`、`.agents/notes/implemented/feature/2026-07-21-continuable-background-subagents.md`、`.agents/notes/implemented/simplification/2026-07-26-merge-subagent-control-service.md`、`.agents/notes/implemented/architecture/2026-08-10-fork-children-stay-one-shot.md`
