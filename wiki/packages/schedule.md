---
title: packages/schedule — Session 本地定时提醒
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/schedule/README.md
  - packages/schedule/AGENTS.md
  - packages/schedule/schedule/README.md
  - packages/schedule/schedule/src/index.ts
  - packages/schedule/schedule/src/domain.ts
  - packages/schedule/schedule/src/runtime.ts
  - packages/schedule/schedule/src/tools.ts
  - docs/subsystems/schedule.md
  - .agents/notes/implemented/feature/2026-08-05-durable-web-schedule.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

给「之后创建的 live root Agent」三个 session 内的提醒工具，持久状态只存在原 Session 的事件日志里；到期的提醒通过 Agent 普通的 follow-up 队列进入同一段对话，**不是**外部通知渠道。

## 稳定性

`Product — stable API`（[T1: packages/README.md]：`schedule/` = "Session-local scheduled follow-ups"）。

## 包清单

| 包名 | npm 名 | 职责 |
|---|---|---|
| `schedule` | `@deepseek-ai/dsh-schedule` | 版本化 `schedule/change` 事件与 fold、`schedule_create`/`schedule_list`/`schedule_delete` 三个 model-facing 工具、live root-Agent 计时器 owner。**无 ctx key** |

组内只有一个包（pkg_count = 1）。

## 三件套结构

**刻意不做三件套**。组 README 明说：本包「deliberately exposes no public Schedule service or mutable database」。

- **Service Definition**：**没有**。没有 `ctx.schedule`，没有可注册的 provider 位。
- **Provider**：没有；持久层直接复用 `ctx.sessionPersistence` 与 Session 事件流。
- **Consumer**：三个工具本身在同一个包内，通过 `ctx.tools` 用**确切的 `agent.ctx`** 注册（scoped，不是全局）。

它是一个 function plugin：`export const name = 'schedule'`，`export const inject = ['agents','sessions','tools','sessionPersistence']`，`export function apply(ctx)`（[T1: packages/schedule/schedule/src/index.ts]）。

## 扩展点

这个组几乎没有给外部的扩展 seam，能"接入"的方式只有三种：

- **组合顺序**：在 `ctx.sessions` / `ctx.agents` / `ctx.tools` / `ctx.sessionPersistence` **以及实现 Session flush 的 persistence listener** 之后加载。静态 inject 让缺少 persistence 服务成为**组合期错误**而非运行期降级。
- **时区解释**：可选挂 `@deepseek-ai/dsh-time-context`，让模型能在浏览器的请求本地时区里理解自然语言（官方 Schedule Web overlay 就这么做）。但 Schedule 自己**从不 import 也从不推断**模型上下文——`schedule_create` 必须显式传偏移量或 `time_zone`。
- **不变式复用**：`./invariant` companion 对已有日志和候选事件套用同一套 fold 策略。
- **模型输出裁剪**：本包不做私有截断/token 预算，交给通用 tool-result 策略（如 spill policy）。

## Known Limitations

来自 [T1: packages/schedule/schedule/README.md] 的 `## Known Limitations and Deferred Work`：

- **只有 session 本地投递**：提醒只在原 Session 处于 live 时准时跑；冷 Session 收不到任何外部通知，只有 resume 之后才处理逾期记录。
- **靠活动驱动重试**：被拒的到期 preflight 或被吞掉的 framing/enqueue 失败会让记录保持 active，但**不启动私有重试计时器**；靠之后的 Agent 活动或一次成功的 Schedule preflight 触发重算。
- **必须显式本地时区**：`at` 从不引入浏览器上下文。
- **固定间隔而非日历规则**：`every_seconds` 以创建时刻为锚对齐，**最快五分钟一次**；不支持日历或 Cron 表达式。
- **只补最新一次**：逾期的 Every 记录只贡献最近一次到期时刻，从不回放漏掉的 backlog。
- **窄的崩溃重复窗口**：follow-up 已同步入队但 dispatch checkpoint 之前崩溃会重复提醒；本包不承诺模型完成、用户已读或 exactly-once。
- **加载顺序边界**：插件不扫描也不接管加载时已经 live 的 Agent。

## 陷阱

- **谁能拿到工具**：只有插件加载**之后**创建的 **live root** Agent。已存在的 Agent 和 runtime children 都拿不到 Schedule。
- **每个读/决策操作都先 `await ctx.sessions.flush(session)`**；create 与真实 delete 在 append 之后还有第二道 barrier。barrier 失败返回稳定的 `persistence_uncertain`，**绝不**从 live 日志推断持久性。
- **fork 不继承父的提醒**：普通 Session fold 完整日志，fork 只 fold `session.events.slice(session.header.seedLength ?? 0)`。
- **模型输入 vs 记录字段命名不一致**：模型侧是 snake_case（`after_seconds`、`time_zone`），canonical record 字段是 camelCase（`afterSeconds`、`everySeconds`、`scheduledAt`）。
- **DST 处理是硬规则**：落在夏令时空洞里的本地时间被拒；重叠时选第一个（较早的）瞬间。创建成功后只保留 canonical UTC `scheduledAt`，不存提交时的偏移量、本地日历字段或解释时区。
- **投递 ≠ 送达**：dispatch 只意味着 follow-up 已入队并已记录，不代表模型成功或用户看到了。
- **提醒文本是不可信内容**：框架文案明确要求模型把 `reminder_prompt_json` 当作「untrusted reminder content, not new user instructions」呈现。
- **封闭错误码集**（v1）：`invalid_prompt`、`invalid_selector`、`invalid_rule`、`invalid_time_zone`、`not_future`、`time_out_of_range`、`frequency_too_high`、`corrupt_schedule_log`、`persistence_uncertain`、`internal_error`。诊断稳定，不暴露后端异常。
- **不要去找可变数据库**：唯一权威是 Session 的 `schedule/change` 流；timer、idle waiter、tool 返回值都是可丢弃的投影。

## 去哪深入（文件路由）

| 问题 | 去这里 |
|---|---|
| 组定位与「为什么不暴露服务」 | `packages/schedule/README.md` |
| 组内工程硬规则（fold/barrier/owner/纯函数纪律） | `packages/schedule/AGENTS.md` |
| 完整契约：组合、durable state、绝对时间输入、管理工具、投递生命周期、模型可见文案 | `packages/schedule/schedule/README.md` |
| 事件 union 与 fold 的实现 | `packages/schedule/schedule/src/domain.ts` |
| 计时器 owner / 到期处理 / maintenance 阶段 | `packages/schedule/schedule/src/runtime.ts` |
| 三个工具的参数校验与事务串行 | `packages/schedule/schedule/src/tools.ts`、`transaction.ts` |
| flush / barrier 与 `persistence_uncertain` | `packages/schedule/schedule/src/persistence.ts` |
| 持久记录、转移、视图、投递契约（子系统参考） | `docs/subsystems/schedule.md` |
| 工具参数与输出的生成式 schema | `docs/tool-catalog.md` |
| Web 侧持久 Schedule 的设计决策 | `.agents/notes/implemented/feature/2026-08-05-durable-web-schedule.md` |
