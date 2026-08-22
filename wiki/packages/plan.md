---
title: packages/plan — plan 协作状态
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/plan/README.md
  - packages/plan/plan-mode/README.md
  - packages/plan/plan-mode/src/index.ts
  - packages/plan/plan-mode/src/types.ts
  - packages/plan/plan-mode/src/client.ts
  - docs/subsystems/plan.md
  - .agents/notes/implemented/simplification/2026-07-22-plan-specific-collaboration-state.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

Plan mode 是**被记录的、per-agent 的协作状态**，**不是**通用 mode registry，**也不是** capability seam——组 README 开门见山就这么定性 [T1: packages/plan/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文：「Plan collaboration state with a direct entry command and reviewed exit」）。

## 包清单

| 包名 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `plan-mode` | `@deepseek-ai/dsh-plan-mode` | `ctx.planMode` | 拥有 plan-mode 状态、guidance、命令与 review 流程（`PlanModeController extends Service`） |

组内只有一个包，这本身就是设计结论：`.agents/notes/implemented/simplification/2026-07-22-plan-specific-collaboration-state.md` 记录了「不做通用 mode 抽象」的取舍。

## 三件套结构

**本组刻意没有三件套。** 它是一个自足的 service + 状态所有者，不拆 Definition/Provider/Consumer，因为这几个角色不会独立演化。

它反过来是别人 seam 的 Consumer，跨组依赖有三条（跨文档综合）：

| 它消费谁 | 用来干什么 | 缺失时 |
|---|---|---|
| `ctx.userQuestions` | `exit_plan_mode` 的用户审批 review | 无法退出 plan mode |
| `ctx.commands` | 注册 `/plan [message]` 与 `/plan off` | 命令不注册，service 仍可被直接驱动 |
| `ctx.sessionProjections` | 注册 `plan` projection unit | 不受影响，projection 不注册 |

后两个都是「注册表被组合时才激活的可选 child fiber」模式。

## 扩展点

- **想在别的入口驱动 plan mode** → 依赖 `ctx.planMode`，调 `set(agent, active)` / `get(agent)`。README 明说：Web client 消费本插件自带的 `/plan` 命令，**其他入口应直接驱动同一个 service，不要再定义第二套 mode 词汇**。
- `set()` 的返回值是可分辨的：`committed`（agent idle，立即追加 `plan/mode` 事件）/ `queued`（agent 运行中，挂起到下一个被接受的 in-turn pre-step）/ `cancelled`（反转）/ `noop`。
- `get(agent)` 返回 `{ active, pending? }`——**把「用于组装当前 step 的已记录状态」与「用户 turn 中途的选择」分开**。
- **想换审批 UI 的呈现** → 不改本包，改 `ctx.userQuestions` 的 provider；review 问题声明了 `plan-review` 这个 presentation intent 并用 `approve` 指名哪个 label 表示批准，认识它的 UI 就把 plan 当决策来渲染，不认识的回落到通用选项流，**工具读到的答案字段两种情况完全相同**。
- **唯一的配置就是 `section`**（必填、非空的提示词文本）。README 明确：本包**不接受**任意命名的 mode、tool filter、sandbox 设置或 approval policy；**未知 key 在 load 时失败**。

## Known Limitations

摘自 `packages/plan/plan-mode/README.md#Known Limitations and Deferred Work`：

- Plan mode 是**引导而非强制**；需要强制限制的部署必须独立配置 sandbox 与 approval 控制。
- 在 turn 的**最后一个被接受的 pre-step 之后**做出的选择，若进程在下一个被接受的 in-turn pre-step 之前退出就会丢失，**UI 必须重新应用**。
- **fork 出来的 agent 继承已记录的 plan 状态，新 spawn 的 agent 从 inactive 开始**；没有创建时的 plan 选项。
- 被另一 agent 拥有的 live child **不能打开 `exit_plan_mode` review**（失败结果会告诉 child 把未决定的事项写进最终结果）；durable fork lineage 本身**不能**阻止一个被 resume 成 runtime root 的 session 打开 review。
- **只有 Web UI 有专门的 `plan-review` 渲染器**，其他 interaction provider 会用通用选项流呈现同一请求。

## 陷阱

1. **`plan/mode` 是 log-only、whole-value-replace 的 `SessionEventMap` 成员**，值形如 `{ active: boolean }`。`foldPlanMode(events)` 返回最后一条记录值或 `false`——**resume / fork / compaction 都直接从 session log 恢复 plan 状态**，没有第二份真相。
2. **`exit_plan_mode` 永远注册**，不管当前是否在 plan mode。理由是**保持 tool schema 在状态切换时稳定**（KV cache 友好）；不在 plan mode 时调用它会失败。
3. **`/plan off` 是被保留的精确参数**。`/plan` 裸调 = 进入；`/plan off` = 退出且不发模型输入，并且会**取消一个尚未到达请求的 pending 进入**；**其他任何非空参数**都是「先进入 plan mode，再把该文本通过 `agent.steer()` 变成下一 step 的普通 user message」。
4. **图片附件的三种行为不一样**：`/plan` 带图 → steer 一条只含 image block 的 user message；带文本带图 → image block 在前、文本 block 在后；**`/plan off` 带图 → 在任何 mode 变更之前直接返回错误**，好让 composer 保住这些图。
5. **「用户切换通知」只在特定条件下产生**：一次改变的用户选择只有在**上一条已记录的 request header 描述的是另一种状态**时，才贡献一条 plugin-sourced 的 `user/message` 通知（两条 commit 路径都遵守）。取消一个 pending 的进入**不贡献通知**，因为没有任何请求观察到过它。
6. **同 step 的 request-recovery 重试会复用冻结的组装**，把选择**继续挂起**给下一个 pre-step——这是「重试不改变已发出请求」的直接后果。
7. **projection 的 `pending` 是纯 replay 量**：`plan` projection unit 从 `command/run`(name=`plan`) 起一个候选目标 → 配对的 `command/done` 保留成功的选择、丢弃出错的 → `plan/mode` 提交已记录状态并清空候选。因此 host 重启、其他 tab、冷读都能只从 log 恢复它，而且**被拒的 `/plan off` 带图不可能留下一个 pending 的退出**。
8. **提示词顺序是 50**：active 时模型看到部署配置的 `section` 原文；inactive 时**不贡献任何文本**。进入/离开会从 order 50 起改变 system prompt（KV cache 影响点）。
9. **「dismissed review」和「rejected review」是两回事**：用户关掉请求改为说话 = dismissed，会如实报告给模型，告诉它**留在 plan mode 等消息**；其他 review 失败保留 seam 自己的消息；rejected 则是带 review feedback 的失败调用。
10. **types 与 client 的双入口**：`plan` unit 的 state key 从 `src/index.ts` 的 `declare module '@deepseek-ai/dsh-session-projection/types'` 并进 `SessionProjectionStateMap`（host fold state 表），wire 后的 client-visible key 再进 `SessionProjectionMap`；host consumer 走 `./types`，client aggregate 走 `./client`。写前端消费代码时别 import 包根。（0.1.1-rc.1 起 projection register 字段改名：`schema`→`stateSchema`、`view` 移入可选 `wire: { viewSchema, view }` 块——plan 的注册已随迁，页面其余结论经 diff 核对不变。）

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| `plan/mode` fold、step-boundary flush、配置、exit 工具的权威定义 | `docs/subsystems/plan.md` |
| `ctx.planMode` 的返回语义、命令语法、projection 规则、Model Experience | `packages/plan/plan-mode/README.md` |
| `PlanModeController` 实现 | `packages/plan/plan-mode/src/index.ts` |
| `SessionProjectionMap` key 声明 | `packages/plan/plan-mode/src/types.ts` |
| client 侧聚合入口 | `packages/plan/plan-mode/src/client.ts` |
| 生成的 `exit_plan_mode` schema | `docs/tool-catalog.md` |
| 为什么不做通用 mode registry | `.agents/notes/implemented/simplification/2026-07-22-plan-specific-collaboration-state.md` |
| review 依赖的提问 seam（intent / `BAD_INTENT` 规则） | `packages/interaction/user-questions/README.md` |
| `/plan` 依赖的命令 registry 与 `command/run`+`command/done` | `packages/interaction/commands/README.md` |
| projection 框架本身 | `docs/subsystems/session-projection.md`、`packages/session/session-projection/README.md` |
