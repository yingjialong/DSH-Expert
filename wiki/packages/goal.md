---
title: packages/goal — 同会话持久化目标（state 与 scheduling 严格分家）
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/goal/README.md
  - packages/goal/goal/README.md
  - packages/goal/goal/src/index.ts
  - packages/goal/goal-round-driver/README.md
  - packages/goal/tool-goal/README.md
  - packages/goal/command-goal/README.md
  - docs/subsystems/goal.md
  - docs/capability-seams.md
  - .agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md
  - .agents/notes/implemented/feature/2026-07-19-same-session-goal-round-driver.md
  - .agents/notes/implemented/feature/2026-07-19-model-facing-goal-tools.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

在**同一个 session 内**持久化一个「当前完成目标」并驱动它自动续跑：状态是 event-sourced 写进 session log 的，续跑权限（activation）**从不持久化**，只活在进程内存里。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `goal/` | `@deepseek-ai/dsh-goal` | 目标状态与生命周期，提供 `ctx.goals`；owns `goal/change` 事件与严格 replay |
| `goal-round-driver/` | `@deepseek-ai/dsh-goal-round-driver` | 同会话续跑驱动：把 armed goal 变成一轮轮 `<goal_round>` prompt |
| `tool-goal/` | `@deepseek-ai/dsh-tool-goal` | model-facing `get_goal` / `create_goal` / `update_goal` |
| `command-goal/` | `@deepseek-ai/dsh-command-goal` | 人机 `/goal` 命令 |

## 三件套结构

`docs/capability-seams.md` 把 `ctx.goals` 标为 Role `core`（Implementations 与 Consumers 两列都是 `-`），**不是可换 provider 的 seam**。真正的分层是「domain / policy / 两类 Consumer」：

- **Domain（唯一状态所有者）**：`goal/` 提供 `ctx.goals`，还发布独立的 `./invariant` companion（对每个 attached session 维护独立 fold，在候选事件进入 durable log **之前**拒绝畸形/不连续/非法迁移/时间戳回退/轮次不连续）。
- **Policy / 调度 Consumer**：`goal-round-driver`。
- **Model-facing Consumer**：`tool-goal`。
- **Human-facing Consumer**：`command-goal`。

组 README 的核心纪律一句话：**consumers 依赖 `dsh-goal`，绝不依赖具体的 agent loop**。

## 扩展点

`goal/README.md` 有专门的 `## Extension points` 一节，要点：

- **策略插件调 service verb + 监听 scoped 的 `goal/changed` 事件**。`goal/changed` 在 durable 事件提交**之后**才 fire，且 listener 失败被容纳。
- **续跑 Consumer 的契约**：把轮次以带 `GoalMessageSource` 的 `user/message` 形式 admit。**普通人类 turn 永远不增加 `roundsStarted`。**
- **一律用 `Agent` 接口和事件，不要 import `dsh-agent-loop`。**
- Service 契约要点：`ctx.goals` 只接受注册在其 id 下的那个精确 live `Agent` 实例；`get()` 返回 detached `GoalView`；所有变更走 `GoalRef { id, revision }` 的 compare-and-set 栅栏，陈旧 ref 被拒。verb：create / edit / pause / resume / complete / block / clear，外加生命周期专用的 `disarm()`。
- `disarm()` 是**唯一不写 revision、不发 mutation** 的例外——它只摘掉进程内的续跑权限。

**三个 config 分属三个包，故意不重复**（重复会产生分歧策略）：
- `defaultMaxGoalRounds`（默认 256）在 `dsh-goal`
- `blockedAfterConsecutiveRounds`（如 3）在 `dsh-tool-goal`
- `goal-round-driver` **没有任何可调 config**

## Known Limitations

组 README 无该章节；以下按包汇总。

**goal**
- **State，不是 scheduling**：本包不决定 armed goal 何时续跑、不重试异常失败、不取消活跃 turn。
- 只有轮次预算：`maxGoalRounds` 不计 token / 金额 / 墙钟 / provider 配额。
- 无独立评估器：记录 complete/blocked 的调用方就是权威。
- **只有一个 current goal**：并行目标与独立 goal 数据库刻意缺席。
- **信任进程内生产者**：有 `Session` 直接访问权的插件能追加伪造的 `goal/change`。严格 replay 只能检测畸形/不一致并在该记录处让 goal 访问失败——是完整性检测，不是插件隔离。

**goal-round-driver**
- 无独立评估器；**只做同会话执行**（不 spawn 新 agent、不 fork session prefix、不做 Ralph 式独立尝试）。
- **accepted-queue 卸载竞态**：Cordis 插件卸载是异步的，已被 inbox 接受的 goal prompt 可能在卸载开始前就消耗掉它那一轮。
- 轮次上限 ≠ 资源预算；无异常自动重试。

**tool-goal**
- 语义意图与「同一阻塞条件是否真的持续」仍是模型判断，运行时只强制「已 admit 的轮次计数」。
- **只挂这个工具包不会产生 goal round**：自主的 `complete`/`blocked` 路径在没有 continuation driver admit goal-sourced user turn 时是休眠的。
- prompt 注册与工具过滤是独立的——scope 可能隐藏了工具却仍保留其 guidance。

**command-goal**
- 只有纯文本交互（无模态编辑表单/替换确认回调）；无 per-command 轮次上限参数；无持续状态挂件。
- **只有 Web 有 command adapter**：headless / ACP / JSON-RPC 用不了 `/goal`。

## 陷阱

1. **Activation 从不持久化**。全新缓存和每一次 `agent/session-start` 边缘都会 disarm，**即使 replay 发现 durable phase 是 active**。所以 session resume / fork / 更换 driver 会保住目标、阶段、revision、已 admit 轮数，但**不会自动开工**——必须有一次显式的 resume mutation 重新 arm。
2. **session log 是唯一 durable 权威**。每次变更都追加带完整变更后快照的 `goal/change`，clear 用带 revision 的 tombstone。因此目标状态**不依赖** inbox placement / claim / admission / discard。
3. **严格 replay 会把日志损坏变成「goal 访问失败」**：增量 replay 在第一个损坏事件处保留游标，直到日志被修复为止。
4. **`{ kind: 'user' }` 是 host attestation，不是随便能伪造的标记**。`Agent.followup()` 和 `steer()` 在调用方省略 source 时会赋这个值，所以插件、调度器等非人类生产者**必须自己传 source**，否则会继承人类权限。
5. **create/edit/pause/resume 需要 runtime-root agent 当前 turn 里有一条被接受的 `{ kind: 'user' }` 消息或 steering 事件**。durable fork lineage 不会把 resumed root 降级，但**活的 subagent 属主关系会**。
6. **驱动器不通过关联 goal message 与 `turn/end` 来分类前序活动**，所以 provider 错误和 token 上限**不是** prompt 级的 goal outcome，也不会映射成 goal blocker code。
7. **blocked 只有一个 durable phase**：provider 限流、配额、执行错误、请求人工介入全都记进这一个 phase，配一个 policy-owned lower-kebab-case code + 规范化自由文本；`tool-goal` 用的码是固定的 `model-reported`。
8. `goal/changed` 会产生持久化义务：driver 在排队工作前 `await ctx.sessions.flush()` 并在 await 之后重新检查 revision 与竞争输入；flush 失败经 `agent/error` 到达时会先 disarm。

## 去哪深入（文件路由）

| 问题 | 去哪 |
|---|---|
| 四个包的分工与 ctx key | `packages/goal/README.md` |
| goal identity、生命周期快照、activation、变更记录 | `docs/subsystems/goal.md`（子系统参考） |
| `ctx.goals` 的 verb 签名与事件 | `docs/subsystems/goal.md#cordis-surface`（生成区） |
| replay 规则、invariant companion、activation 语义 | `packages/goal/goal/README.md` |
| 轮次契约、idle checkpoint、取消与卸载语义 | `packages/goal/goal-round-driver/README.md` |
| 三个工具的参数、权限检查、canonical 返回值 | `packages/goal/tool-goal/README.md` |
| `/goal` 命令契约 | `packages/goal/command-goal/README.md` |
| 设计源头 | `.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md`（domain）、`…-same-session-goal-round-driver.md`（driver）、`…-model-facing-goal-tools.md`（权限拆分） |
