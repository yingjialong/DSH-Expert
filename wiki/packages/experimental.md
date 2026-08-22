---
title: packages/experimental — 私有实验包
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/experimental/README.md
  - packages/experimental/AGENTS.md
  - packages/experimental/agent-team/README.md
  - packages/experimental/tool-agent-team/README.md
  - packages/experimental/agent-team/src/index.ts
  - packages/experimental/agent-team/src/mailbox.ts
  - packages/experimental/agent-team/src/task-board.ts
  - packages/experimental/tool-agent-team/src/index.ts
  - docs/subsystems/agent-team.md
  - examples/headless-agent/team.cordis.snapshot.yml
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

用仓库真实运行时跑、但不进官方 release 的原型与内部插件。当前只装了一件东西：Agent Teams（隐式 Lead 的团队花名册 + 持久化对等邮箱 + 共享任务 DAG）及其模型面工具。

## 稳定性

`Unreleased`（`packages/README.md` 表格原文）。包一律 `private: true`，**无稳定性与支持承诺**；但工程、安全、文档、生命周期、测试、快照要求与 release 包完全一致（`packages/experimental/AGENTS.md` 明文）。

## 包清单

| 包目录 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `agent-team/` | `@deepseek-ai/dsh-experimental-agent-team` | `ctx.agentTeams` | 隐式 root 的 Team 域：花名册、持久化对等邮箱、共享任务 DAG、运行时协调 |
| `tool-agent-team/` | `@deepseek-ai/dsh-experimental-tool-agent-team` | — | 作用域化的模型面 Team 工具 + 协作策略 prompt |

## 三件套结构

- **Service Definition**：**本组没有独立的抽象 seam 包**。`agent-team` 同时是定义与实现——它直接注册具体的 `ctx.agentTeams` 服务，没有留出一个只依赖 cordis 的抽象层。这是本组与 `compaction`/`credentials`/`fs` 等 product 能力族最大的形态差异，符合「roles 未独立演化就不拆」的纪律。
- **Service Provider**：同上（`agent-team` 自身）。它要求 Agent、Session、Session 持久化、continuable-subagent 四类服务；**没有持久 Session 存储的组合不会激活它**。
- **Consumer**：`tool-agent-team`——十个模型面 schema（`spawn_teammate` … `team_task_update`）加一段固定策略 section，只在 Team 成员的 agent scope 里出现。它通过监听 Agent 发布事件、往该 Agent 的 scope 里安装注册项，所以全新创建和冷恢复拿到的工具/prompt 集合完全一致。
- **实际的 continuable-subagent provider 由配置选**：`tool-agent-team` 的 `freshProvider` / `forkProvider` 指向已注册的 continuable-subagent provider（示例配置里是 `spawn` / `fork`）。

## 扩展点

- **想扩展 Team 行为**：只能依赖 `@deepseek-ai/dsh-experimental-agent-team` 本身（没有更抽象的层可依赖）。注意依赖隔离规则见下。
- **想换模型面呈现**：`tool-agent-team` 是可替换的 Consumer，`ctx.agentTeams` 的域语义与授权（Lead-only 的 `spawn_teammate` / `interrupt_agent`、任务的 owner/revision 校验）**在服务内部强制**，不只写在工具描述里——所以换 Consumer 不会绕过授权。
- **`./invariant` 伴生**：把每个候选 Team 事件对着已提交的 Session 前缀重放，在 append 之前拒绝非法成员转换、复用名字、越界任务 id、不连续 revision、非法依赖、重复排队/确认记录、目标错误的确认。
- **promotion 路径**（`packages/experimental/AGENTS.md`）：把包移到它的产品角色组、从 npm 名去掉 `experimental-`、原子地更新每一处 import 与配置行，然后复审公开契约、限制、测试证据、release 载荷、运行时依赖方与稳定负责人。

## Known Limitations

- `agent-team`：**一个进程、一份共享 checkout**——成员共用 cwd 并立即看到彼此的编辑，本包不提供 worktree、远程成员、merge 或文件锁。
- `writeScopes` 是**建议性**的：Bash、格式化器、代码生成器、外部写入者都能绕过文件版本检查；视图只在与进行中任务重叠时告警，从不阻止 claim，也不授权写入。
- 花名册**扁平且不可变**：只有 Lead 能建直接 teammate，没有嵌套 Team、改名、删除或名字复用。
- **没有自动释放归属**：闲置、被打断、进程退出、工作失败都不会释放任务 owner。
- **邮箱不是跨进程 exactly-once**：保证只是「进程内重试 + 目标 Session 去重」，同一个 Team 上并发多个 harness 进程不受支持；本版本也没有邮箱时间线 UI。
- `tool-agent-team`：prompt 策略是协调不是限制（拦不住 Bash 与外部进程写重叠文件）；**不会自主组队**（除非用户显式要求）；没有 Web 端的花名册/任务板呈现。

## 陷阱

- **npm 名和目录名不一致**：目录是 `agent-team/` / `tool-agent-team/`，包名是 `@deepseek-ai/dsh-experimental-agent-team` / `@deepseek-ai/dsh-experimental-tool-agent-team`。组 README 的表格只写了目录名，容易照抄成错误的包名。
- **依赖隔离是硬规则**：release 包和 app **不得**在 `dependencies` / `optionalDependencies` / `peerDependencies` 里点名本目录的包；实验包可以依赖 release 包和彼此；测试可以走 `devDependencies`；示例可以显式加载。dsh release family 会排除本目录。
- **作用域 Team 定义会影子同名的 legacy 全局 continuable-subagent 控件**——同时挂了两者的组合**必须显式禁用 legacy 定义**。
- **`queued` 不是「让你重发」**：`sendMessage()` 的结果永远标识那条持久化消息，`queued` 只表示即时投递被推迟；工具层也重申「A `queued` result is accepted durable work and must not be retried」。
- **安静投递永远不唤醒**：目标是 live 才立刻注入并确认；目标 inactive 时安静消息就一直排队。要唤醒得用 waking 投递（`followup_task` 能冷恢复目标）。
- **名字永不复用**：`maxMembers` 统计**曾经预配过的每一个名字，包括失败成员**；`maxTasks` 只统计未删除任务，被删任务保留为 tombstone 以保证重放与 id 稳定。
- **`waitForChange()` 不重放已发生的变化**：它只等注册之后发生的一次边沿，返回后调用方必须重新读权威状态。`wait_agent` 在武装等待前会先检查是否还有 running/provisioning 的成员，没有就立刻返回 `noProgress`。
- **`TeamId` 等于 root 的 `SessionId`**，建 Team 在第一条成员/消息/任务记录之前是**无状态**的；ordinary fork 成为独立 runtime root 后，继承来的旧 root `TeamId` 记录会被忽略。
- **`interrupt()` 只取消 live teammate 的当前 turn（带 `keepInbox`）**，既不释放任务归属也不删除持久邮件。

## 去哪深入（文件路由）

| 想知道什么 | 去哪 |
|---|---|
| 子树规则：依赖隔离、release 排除、promotion 流程 | `packages/experimental/AGENTS.md` |
| Team 身份与花名册、provisioning 恢复、邮箱投递、任务板 DAG 与 revision 规则、config 五项 | `packages/experimental/agent-team/README.md` |
| 持久事件的字面形状与服务 API | `docs/subsystems/agent-team.md` |
| 十个工具的确切 schema | `docs/tool-catalog.md#deepseek-aidsh-experimental-tool-agent-team` |
| 模型看到的策略 section 与授权分工 | `packages/experimental/tool-agent-team/README.md` |
| 协调与隔离决策来源 | `.agents/notes/implemented/feature/2026-08-05-agent-teams.md` |
| 为何单独设实验包目录 | `.agents/notes/implemented/architecture/2026-08-18-experimental-agent-teams-packages.md` |
| 可跑的 Team 快照配置 | `examples/headless-agent/team.cordis.snapshot.yml` |
