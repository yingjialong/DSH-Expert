---
title: packages/todo — todo / planning 能力族
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/todo/README.md
  - packages/todo/tool-todo/README.md
  - packages/todo/tool-todo/src/index.ts
  - packages/todo/tool-todo/src/types.ts
  - docs/subsystems/session.md
  - docs/tool-catalog.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

只有一个包的 model-facing todo 能力：`todo_write` 把整张任务清单整体替换并写进会话日志。之所以是单包而非 seam，是因为**一个 agent session 独占这张表，不存在可替换的 provider 契约**。

## 稳定性

`Product — stable API`（`packages/README.md` 表格中 `todo/` 行原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `todo/tool-todo/` | `@deepseek-ai/dsh-tool-todo` | 注册 `todo_write` 工具，存储并暴露该会话的 todo 列表；registers on `ctx.tools` |

## 三件套结构

**刻意不拆**。group README 的原话是「It is a single **product** package because one agent session owns the list; there is no replaceable provider contract.」——Service Definition / Provider / Consumer 三个角色全部塌缩进 `tool-todo` 一个包。DSH 的纪律是「seam 要么完整三角色，要么就不拆」，todo 属于后者。

## 扩展点

- 这里**没有可注册的 provider seam**。要改行为只有三条路：
  1. **Config**：`allowParallelInProgress` 是**必填、无默认**的部署选择——模型侧指令和输入校验一起随它变（`true` 请模型标注所有在做的任务并接受任意个；`false` 请求恰好一个，超了报 `Error: invalid todos: at most one task may be in_progress (got <n>)`）。
  2. **消费 `todo/write` 会话事件**：UI 订阅事件流自行渲染，当前列表 = 最近一条 `todo/write`（重放时 last-write-wins）。
  3. **session projection**：当组合挂了 `ctx.sessionProjections`（`@deepseek-ai/dsh-session-projection`）时，本包在注入的子 context 下注册 `todos` projection unit——`init = null`、`apply` 取每条 `todo/write` 的整张列表且在每个 `turn/start` 清回 `null`、`view` 为 identity、`stateVersion = 2`。key 通过 Service Definition 包的 `/types` 出口合并进 `SessionProjectionMap`。没挂注册表的组合不受影响。
- **导出形态**是 function/namespace plugin：导出 `name` / `inject` / `apply`，**没有 default export**。误加 `export default` 会被 Loader 的 `unwrapExports` 折叠模块并丢掉 `inject`（`docs/postmortem/0001-acp-default-export-drops-inject.md`）。

## Known Limitations

- **单 owner 作用域**：列表属于调用它的那**一个** agent session；没有 subagent / shared / swarm 作用域，非 agent 调用者（无 `exec.agent`）直接被拒。
- **item 形状刻意最小**：只有 `content` 加三态 `status`（`pending` / `in_progress` / `completed`）；整表替换语义下不需要稳定 id、优先级、active-form 字段。
- **整表替换是唯一操作**：没有部分更新，也**没有读回工具**——模型每次必须重发整张列表。

## 陷阱

- **校验比 schema 更严**：`execute` 还会拒绝空 `content`、重复 `content`、以及任何超出 `content` / `status` 的 item key。扩展 item 形状（加 id、加嵌套）会**大声失败**而不是被静默压平，目的是让日志快照等于模型自认为写下的内容。
- **持久日志不变量不跟随 config**：在允许并行时写下的日志，在部署收紧到 `allowParallelInProgress: false` 之后**仍然必须能重放**，所以 invariant 对 active 数量保持沉默。别把 config 校验和日志校验搞混。
- 成功结果的模型可见文本是固定的一句：`Updated todo list: <pending> pending, <inProgress> in progress, <completed> completed.`——想 pin 快照就 pin 这句。
- Web 客户端的展示语义是「最新一条 `todo/write` 且其后没有 `turn/start`」，即**下一轮开始时计划条会清掉**；这不是 bug。

## 去哪深入（文件路由）

| 想查 | 去 |
|---|---|
| 工具契约、校验、渲染、projection 全文 | `packages/todo/tool-todo/README.md` |
| `todo/write` 事件负载定义 | `docs/subsystems/session.md` |
| 生成的 `todo_write` schema | `docs/tool-catalog.md` 锚点 `#deepseek-aidsh-tool-todo` |
| 单 owner 作用域为何是刻意取舍 | `packages/todo/tool-todo/README.md` §Single owner |
| default export 会丢 `inject` 的事故复盘 | `docs/postmortem/0001-acp-default-export-drops-inject.md` |
