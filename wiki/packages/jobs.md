---
title: packages/jobs — 后台作业 capability family
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/jobs/README.md
  - packages/jobs/jobs/README.md
  - packages/jobs/jobs-local/README.md
  - packages/jobs/tool-jobs/README.md
  - packages/jobs/jobs/src/index.ts
  - packages/jobs/jobs-local/src/index.ts
  - docs/subsystems/jobs.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

给「长跑工具」一套统一的、**owner 隔离**的后台作业协议：观察、取消、等待、完成通知，四件事只有一份契约 [T1: packages/jobs/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

| 包名 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `jobs` | `@deepseek-ai/dsh-jobs` | `ctx.jobs` | 定义 job registry 与生命周期契约（抽象 `JobRegistry`） |
| `jobs-local` | `@deepseek-ai/dsh-jobs-local` | 注册到 `ctx.jobs` | 进程内实现 `LocalJobRegistry`，内存记录、`<kind>-N` id |
| `tool-jobs` | `@deepseek-ai/dsh-tool-jobs` | 注册到 `ctx.tools` | 模型侧 `job_output` / `job_list` / `job_kill` 三工具 + 完成通知 + 提示词段 |

## 三件套结构

- **Service Definition**：`jobs`（`ctx.jobs`，`JobRegistry extends Service`）。producer 插件通过 declaration merging 扩 `JobKindMap` 来登记自己的不透明 id 命名空间。
- **Service Provider**：`jobs-local`（唯一 shipped 实现）。
- **Consumer**：`tool-jobs`（模型侧控制器）；此外真正**产生** job 的是别组的 producer（如 shell / terminal 等长跑工具）。

跨文档综合的关键点：`tool-jobs` **不只是 Consumer**——加载它同时会 `attachController`，而 `ctx.jobs.start()` 要求「有 attached controller 服务该 owner」。所以**没有在自己 composition 里挂 `tool-jobs` 的 agent，根本无法启动后台作业**，报错原文是 `background jobs unavailable: no job controller serves this agent (load @deepseek-ai/dsh-tool-jobs in its composition)` [T1: packages/jobs/jobs-local/README.md]。

## 扩展点

- 想让自己的长跑工具支持后台化 → 依赖 **`@deepseek-ai/dsh-jobs`**（Service Definition），扩 `JobKindMap`，调 `ctx.jobs.start(spec)`；`spec.run()` 由 registry 恰好调用一次。可选提供 `outputLimitBytes`（producer 拥有的 model-presentation policy，registry 原样带进 snapshot、不改写、不发明默认值）。
- 想换一个后端（例如持久化 / 跨进程）→ 实现 `JobRegistry` 抽象类并注册为 `ctx.jobs`。但见 Known Limitations：**当前契约是 in-process 的**。
- 想观察作业变化 → `onJobDone(listener)`（每条终态记录 + 精确 owner）与 `onJobsChanged(listener)`（可见集合变化，owner 粒度）。两者语义不同：`onJobsChanged` **不是** `onJobDone` 的超集，它不带 delivery 含义、不标记 reported。
- 想给某个 composition 授予/收回「可开后台作业」的能力 → `attachController(name)`；注册是 **owner-relative** 的：无 scope 上下文注册的 controller/listener 服务所有 owner，agent composition scope 下注册的只服务其下的 agent。

## Known Limitations

- `jobs`（seam）：**stream 输出只有一个消费游标**；**前台工作不可提升为后台**（producer 启动前就得选）；**契约是 in-process 的**——`JobStart.run()` 传的是回调与精确 `Agent` 对象，持久化/跨进程后端必须先重塑 identity、restart、ownership、observation 语义。
- `jobs-local`：**作业是进程内的**，随 harness 进程消亡；**「静默无效的 cancel」会拖住 teardown 并占用容量**——`cancel` 返回却不 settle `done` 时，registry 无法与「慢停止」区分，该 job 会在 service 剩余生命周期内一直占一个 bucket 槽位，只有显式抛错才能被安全 force-fail。
- `tool-jobs`：**driver 退休窗口内的 settlement 会遗失通知**（turn loop 最后一次 inbox 检查与 driver 提交 idle 之间，owner 仍读作 busy，通知被注入却无人唤醒；steering 有同样的洞，补它属于 `agent-loop`）；**耗尽的 wake 预算不随时间恢复**，只有用户撰写的输入能补充；**idle owner 上 pending 的通知不会挺过该 owner 的 disposal**；stream 读是单消费者；**unowned job 没有 session fence**。

## 陷阱

1. **Owner fence 就是全部安全边界**。`bash-1` 这类 id 是**可预测**的，所以 owned 访问靠比较 job 的 `SessionId` 与 caller 的 `SessionId` 来隔离。**unowned job 对任何 caller 开放**，一直活到 service disposal——外部调用方必须自己加策略或干脆别用。
2. **`read()` 是消费性的**：stream job 读一次就消掉游标；final-output job 的终态输出才是幂等可重读的。
3. **`kill()` 先调 producer 的取消再改状态**：取消抛错 → job 继续跑（状态不变）；成功 → 状态变 `stopping` 并标记 terminal delivery 已 reported。
4. **容量按 owner 计**：`maxConcurrentJobsPerOwner` 默认 `10`，统计该 owner 的 `running` + `stopping`；**所有 unowned job 共享另一个独立的 service bucket**；终态历史不占容量；**只有 producer 的 `done` settlement 才释放 stopping 的槽位**。满容量时 `start()` 在 producer 执行与 id 分配**之前**失败，registry 不排队、不抢占。
5. **完成通知走哪条 lane 取决于 owner 在干什么**：busy owner → 注入下一 step 的 inbox（多个 job 同时结束只花一个 step，而非各占一个 turn）；idle owner → **开一个 follow-up turn 唤醒它**。想要确定性 transcript（快照测试）必须配 `completionDelivery: quiet`，让 idle owner 也走注入 lane。
6. **唤醒有预算且是自激的**：`maxConsecutiveWakes` 默认 `3`；被唤醒的 turn 可能又启动一个 job 从而再次唤醒自己，所以设了上限；**本插件排队的通知永远不会补充它花掉的预算**，只有用户撰写的消息能恢复。
7. **一个 host registry 上可以有多个 `tool-jobs` mount**（每个 agent preset 一个）。registry 按 owner 的 scope chain 路由 settlement，因此一次完成**只会**被对应 agent 读到一次，不会因为挂了多个 preset 而重复。
8. **`job_*` 的提示词段与工具是分别注册的**：agent-scoped 的 tool filtering 可以隐藏工具，却**不会**移除那段 background-job guidance 提示词。
9. **jobs 的生命周期归 owner + backend，不归 producer 的 tool fiber**：producer / controller 热重载不会停掉在跑的 job。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| id 方案、owner-fenced 契约、snapshot 定义 | `docs/subsystems/jobs.md`（子系统参考，权威） |
| `start/get/list/read/kill/wait/onJobDone/onJobsChanged/attachController` 逐条语义 | `packages/jobs/jobs/README.md`、`packages/jobs/jobs/src/index.ts`（`JobRegistry`） |
| 容量、settlement first-wins、teardown、scope 分层 | `packages/jobs/jobs-local/README.md`、`src/index.ts`（`LocalJobRegistry`） |
| 三个工具的参数/返回/UI card、完成通知文本、config 默认值 | `packages/jobs/tool-jobs/README.md` |
| `job_output` / `job_list` / `job_kill` 生成的 schema | `docs/tool-catalog.md` |
| 为什么要有通用长跑运行时 | `.agents/notes/implemented/architecture/2026-06-20-generic-long-running-tool-runtime.md` |
| 为什么把 registry 抽成 seam | `.agents/notes/implemented/architecture/2026-07-26-job-registry-seam.md` |
