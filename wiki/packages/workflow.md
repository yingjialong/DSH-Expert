---
title: packages/workflow — 动态工作流能力族
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/workflow/README.md
  - packages/workflow/workflow/README.md
  - packages/workflow/workflow/src/index.ts
  - packages/workflow/workflow/src/runtime-types.ts
  - packages/workflow/workflow-worker-thread/README.md
  - packages/workflow/tool-workflow/README.md
  - packages/workflow/tool-ralph/README.md
  - docs/subsystems/workflow.md
  - .agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md
  - .agents/notes/implemented/feature/2026-07-19-fresh-agent-ralph-workflow-tool.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

跑**模型自己写的**编排脚本、由脚本 fan-out 出子 agent；seam 定义 script / run / result / error / event 契约，engine 决定如何隔离与执行。

## 稳定性

`Product — stable API`（`packages/README.md` 表格中 `workflow/` 行原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `workflow/workflow/` | `@deepseek-ai/dsh-workflow` | Service Definition：`WorkflowEngine`（abstract）挂 `ctx.workflowEngine`，定义执行与生命周期事件 |
| `workflow/workflow-worker-thread/` | `@deepseek-ai/dsh-workflow-worker-thread` | Provider：一次 run 一个 Node worker thread |
| `workflow/tool-workflow/` | `@deepseek-ai/dsh-tool-workflow` | Consumer：通用 `workflow` 工具（模型写 script） |
| `workflow/tool-ralph/` | `@deepseek-ai/dsh-tool-ralph` | Consumer：固定策略的 `ralph` 工具（fresh-agent 轮次循环） |

## 三件套结构

- **Service Definition**：`@deepseek-ai/dsh-workflow`。`export abstract class WorkflowEngine extends Service`（默认导出），另有 `WorkflowError extends HarnessError`、`WorkflowErrorCode`、`WorkflowEventName`、`isFatalWorkflowError()`、`WorkflowRunId` [T1: packages/workflow/workflow/src/index.ts]。包根是 **Host face**；**浏览器安全**的 `@deepseek-ai/dsh-workflow/types` 子路径只含 run 身份、元数据、结果和 observe-only 生命周期负载，不 import `Agent` / Cordis service / Host context 声明；Host-only 的 `WorkflowStartRequest` 与 `WorkflowRun` 留在包根后面。
- **Service Provider**：`@deepseek-ai/dsh-workflow-worker-thread`（当前唯一 engine）。包根导出默认 engine 插件与其 `Config`；worker protocol / runtime / session 模块对实现私有；运维用的 `./worker` 入口是 engine 的 spawn 目标。
- **Consumer**：`tool-workflow`（通用）与 `tool-ralph`（固定策略）。**Ralph 刻意做成普通插件**——没有往 `agent-loop` 里加 Ralph mode 或 fresh-agent 循环，同会话的 goal domain 也保持独立。

## 扩展点

- **换一个执行引擎**（独立进程、容器、真沙箱）：依赖 **`@deepseek-ai/dsh-workflow`** 实现 `WorkflowEngine`，工具侧完全不用改。
- **`start(request): WorkflowRun` 的同步校验义务**：在 run 存在**之前**就必须拒绝畸形 meta block、无法解析的 script、不可用的 provider 路由、不支持的 per-run limit。
- **`WorkflowStartRequest`** = `{ meta, script, args?, subagentProvider?, maxTotalAgents?, parent, signal? }`。`parent` 把每个子 agent 归属到调用者；`subagentProvider` 为整个 run 路由所有 child 而**不把 provider 选择暴露给脚本**；`maxTotalAgents` 只能**下调**引擎的部署上限、同样对脚本不可见。`meta` 与 `args` 是纯数据，不是脚本片段。
- **`WorkflowRun`** = `{ id, meta, result, cancel(reason?), dispose() }`；`WorkflowResult` = `{ value, stopReason, error?, agentsStarted }`，`value` 是纯 JSON 或 `null`。
- **写自己的 Consumer**：依赖 `ctx.workflowEngine`。`tool-workflow` Config 有 `toolName`（默认 `workflow`）、`maxResultChars`（默认 `50000`）；`tool-ralph` Config 有 `maxRounds`（默认 `256`，既是默认值也是调用覆盖的上限）、`maxHandoffChars`（默认 `16384`）、`maxResultChars`（默认 `16384`）。
- **事件**是 observe-only：携带 `WorkflowRunInfo`（`id` + `meta`）而**不是 live run**，监听者拿不到取消/释放权限。`workflow/start` 与 `workflow/end` 配对；`workflow/phase` / `workflow/log` 暴露脚本叙述；`workflow/agent-start` / `workflow/agent-end` 按 `seq` 配对每次子调用——**异步 provider start 被拒的 child 两个事件都不发**。

## Known Limitations

- **只有前台收集**：调用者持有一个 live run 并 await；背景 start/poll、spill handle、detached collection 都推迟。父轮次会一直阻塞到整个 workflow settle，取消会把部分输出作为 error 丢弃。
- **无日志化、无 resume**：脚本、子进度、中间值都不做 checkpoint，进程重启无法续跑。
- **无保存的/嵌套的 workflow**：seam 只启动调用方给的脚本，脚本内**拿不到 `workflow()` 钩子**做递归编排。
- **无 token 预算词汇**：引擎能 cap 并发、item 数、child 数，但 request 和 result 都不核算跨 child 的模型 token。Ralph 也只有 round 数这一个总量上界。
- **run 归持有者所有，不由 service 追踪**：卸载 engine 插件只阻止新 start，**不撤销已接受的 run**；每个消费者必须自己在所有路径上 `dispose()`。
- worker-thread engine：**一次 run 付一个 worker**，无池、无热运行时、无跨 run 脚本缓存；终止只能报告 host 观察到的启动数（`agentsStarted` 不含强制终止时仍排在并发后面的 worker 侧调用）；**跨 realm 错误在脚本内 `instanceof Error` 为 false**，作者必须判 `name` / `code` 这类稳定字段。
- Ralph：**完成是 worker 自述**，没有独立评估器/验证器；只有前台，无 job id / 后台收集 / 断点续跑 / 调度器；**workspace 是唯一跨轮长期记忆**，一份有界报告是显式交接，未落盘的对话推理随每个 child 消失；一轮一个新 child，无轮内 fan-out、无模型切换、无 fork context；**普通 child 失败对整个 run 是终止性的**，不重试。

## 陷阱

- **worker / `node:vm` 不是安全边界**——README 原文明说：模型写的代码能逃出 `node:vm` 拿到 worker 的进程权限。worker 提供的是有用的**收容**（脚本 CPU 不占 host event loop、`worker.terminate()` 给 dispose 一个真正的终点、worker 以空环境启动因此 ambient 凭据不经 `process.env` 传入、host/worker 用 structured-clone 且脚本边界做纯 JSON 校验），但**缺失的全局变量是可移植性 API，不是隔离**。要跑真正不可信的脚本，需要在同一条 seam 后换一个引擎。
- **`WorkflowRun.result` 一旦返回就永不 reject**：执行失败以 `stopReason: 'error'` resolve，取消在引擎的有界宽限内以 `cancelled` resolve。别对它写 `.catch` 逻辑。
- **fatal 标志决定错误是否穿透 `parallel()` / `pipeline()`**：`WorkflowError` 带 code 和 `fatal`；fatal 错误总是逃逸而不是退化成某一项的 `null`。fatal 码包括 `SCRIPT_PARSE`、`META_INVALID`、`INVALID_ARGUMENT`、`UNSUPPORTED_OPTION`、`UNSUPPORTED_SCHEMA`、`AGENT_CAP`、`ITEM_CAP`、`AGENT_START`、`AGENT_RESULT`、`RESULT_UNSERIALIZABLE`、`CANCELLED`。
- **正常 resolve 但 stop reason 非 completed 的 child 不是基础设施异常**：`agent()` 返回 `null`，让脚本自己处理普通子失败。
- `tool-workflow` 的 `args` **必须是对象**：顶层数组或标量要自己包一层字段，否则 wire schema 不诚实。
- **只有 root transport 执行（`exec.parent` 缺席）才把 run 投影进会话**；嵌套 transport 调用正常执行但**不写记录**。首次 Session append 失败会关闭该 run 的后续记录、发一条 warning，并留下「无记录」或「合法的连续前缀」，**不改变工具结果和清理**。
- Ralph 的 subagentProvider 必须存在、支持 structured output、且报告 `inheritsParentContext: false`；无效/缺失/超长的报告会**让 workflow 失败**，而不是被截断或误判成 cap 耗尽。

## 去哪深入（文件路由）

| 想查 | 去 |
|---|---|
| start request、`WorkflowMeta`、结果、live run、`workflow/*` 事件 | `docs/subsystems/workflow.md` |
| seam 契约、事件、失败纪律全文 | `packages/workflow/workflow/README.md` |
| 信任与隔离边界、脚本契约 | `packages/workflow/workflow-worker-thread/README.md` |
| `workflow` 工具 schema 与生命周期 | `packages/workflow/tool-workflow/README.md`、`docs/tool-catalog.md#deepseek-aidsh-tool-workflow` |
| Ralph 固定策略、报告格式 | `packages/workflow/tool-ralph/README.md`、`docs/tool-catalog.md#deepseek-aidsh-tool-ralph` |
| 被推迟的 workflow API 全清单 | `.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.md` |
