---
title: SDK JSON-RPC 线协议（stdio）— 源码级契约
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/sdk/protocol/src/transport.ts
  - packages/sdk/protocol/src/types.ts
  - packages/sdk/protocol/README.md
  - packages/sdk/server/src/server.ts
  - packages/sdk/server/src/index.ts
  - packages/sdk/client/src/client.ts
  - packages/sdk/client/src/api.ts
  - packages/sdk/client/src/dispose.ts
  - packages/sdk/client/tests/fake-runtime.ts
  - python/sdk/src/deepseek_harness/client.py
  - python/sdk/src/deepseek_harness/api.py
  - packages/core/session/src/types.ts
  - packages/core/agent/src/runtime-types.ts
  - packages/subagent/subagent/src/types.ts
  - examples/jsonrpc-agent/tests/snapshots/text-turn/notifications.expected.jsonl
  - examples/jsonrpc-agent/tests/snapshots/bash-tool/notifications.expected.jsonl
  - packages/examples/jsonrpc-demo/src/runner.ts
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

# SDK JSON-RPC 线协议（源码级）

选型层面见 `wiki/integration/integration-surfaces.md`。本页只讲**怎么把这条线实现出来**。参考实现有三份：TS 客户端 `packages/sdk/client/src/client.ts`、Python 客户端 `python/sdk/src/deepseek_harness/client.py`（不 import TS 类型，纯镜像）、以及一份 200 行的纯 stdio 假 runtime `packages/sdk/client/tests/fake-runtime.ts`（**最适合照抄的服务端骨架**）。

## 传输与分帧（stdio + newline-delimited JSON-RPC）

- 一行一帧，compact JSON，`\n` 结尾；`JsonRpcLineTransport` 用 `indexOf('\n')` 切行、`trim()` 后跳过空行 [T1: packages/sdk/protocol/src/transport.ts]。UTF-8 用 `StringDecoder` 增量解码，多字节字符跨 chunk 不会碎。
- 帧分类**只看两个字段**，不看 `jsonrpc` 版本号（从不校验）：`id`+`method`=请求，只有 `id`=响应，只有 `method`=通知。三者都不是就丢弃。
- **畸形 JSON 行被静默忽略**，不回 `-32700`。**不支持 batch**（顶层数组的 `id`/`method` 都是 undefined，直接丢）。
- `params` 被规范化成对象：数组或标量一律塌成 `{}`（`objectParams`）。所以别用位置参数。
- 请求 id：TS 端生成 `req_<32位hex>` 字符串，Python 端生成 uuid4 字符串；服务端原样回显。**响应不保证按请求顺序返回**——`drainLines` 对每行 `void this.handleLine(line)`，处理是并发的，宿主必须按 id 关联。
- stdout **只能**有协议帧，诊断走 stderr；`dsh-sdk-jsonrpc-server` 明说它不会检查也不会否决兄弟插件里的 stdout logger。

## 握手与初始化序列

1. 宿主 spawn runtime（`dsh-jsonrpc-agent` 或打包 exe），**config 必须显式给**：`DSH_CORDIS_CONFIG` 环境变量优先于 `argv[2]`，都没有就打 usage 到 stderr 并 `exit 1`，没有任何内建兜底 [T1: packages/examples/jsonrpc-demo/src/runner.ts]。
2. 发 `initialize`。服务端在进入 handler 前先 `await ctx.get('loader')?.await()` —— 这是**readiness 边界**：等当前插件树全部 settle（例如 MCP 初次 tool discovery）后才回包，所以第一个 prompt 能看到完整能力 [T1: packages/sdk/server/src/index.ts]。手搓的无 Loader context 则立即可用。
3. 回包只有 `{ serverInfo: { name, version } }`。`name` 是 wire-stable 的 `deepseek-harness-sdk-runtime`；`version` 硬编码 `'0.0.1'`，**与包版本 `0.1.0-rc.8` 无关，且客户端不校验**。
4. `initialize` 里的 provider 解析：已注册 adapter 直接复用；未被占用且名为 `deepseek-official` 时自动挂 `dsh-llm-deepseek`；**其他未被占用的 provider 直接抛错**（`no adapter registered for provider "X"`）。
5. `cwd` 被服务端 `resolve()` 一次（相对当前 runtime 进程 cwd）。TS 高层 API 在过线前就先转绝对路径，避免双重解析（注释里举的坑：`worker` → `worker/worker`）[T1: packages/sdk/client/src/api.ts]。

## 核心方法清单

| 方法 | 方向 | 用途 | 参数要点 | 锚点 |
|---|---|---|---|---|
| `initialize` | client→server | 进程级握手，设定后续所有 SDK session 的路由 | `{cwd, provider, model, maxTokens?}`；`maxTokens` 必须是正 safe integer，否则抛 `TypeError` | `packages/sdk/protocol/src/types.ts#InitializeParams` |
| `session/prompt` | client→server | 投递一条 user 消息 | `{sessionId, contentBlocks}`；未知 `sessionId` **惰性创建** agent+session；返回 `{messageId}` | `packages/sdk/server/src/server.ts#prompt` |
| `shutdown` | client→server | 优雅收尾 | 无参；返回 `{}`，**写完响应后** `setImmediate` 触发 flush → dispose root fiber → `process.exit(0)` | `packages/sdk/server/src/index.ts#apply` |

`initialize` **不是强制前置**：`session/prompt` 可以先发，服务端有默认值 `cwd=process.cwd()`、`provider='deepseek-official'`、`model='deepseek-official'`（注意 model 默认值也是 `deepseek-official`，几乎肯定不是你要的）。

## 通知/事件清单（server→client，全部是单向 notification）

| 方法 | payload | 语义 |
|---|---|---|
| `session.event` | `{sessionId, event}` | 一条 session-log 事件，**runtime 里每一个 session 都推，不做过滤** |
| `session.status` | `{sessionId, status}` | 整个 agent 的 `idle`/`running` 翻转；`running` 从唤醒输入开始，`idle` 表示没有 driver 在跑 [T1: packages/core/agent/src/runtime-types.ts#AgentStatus] |
| `subagent.started` | `{parentSessionId, childSessionId}` | 由 `session/created` + `header.parentSession` 推出 |
| `subagent.finished` | `{provider, agentId, parentSessionId, childSessionId, status, stopReason, lastAssistantMessage?}` | **只报 in-process 子 run**（服务快照的 `local` 标志为真）；远程 run 一律不报 |

`event` 是完整的 session-log 信封：`{type, seq, time, data, ignorable?, surfaceOp?, sourceEventSeqs?}`。`seq` 在 session 内单调。**`ignorable` 缺省 = 必需**：读者遇到不认识且无该标记的 type，按契约应拒绝重建 session 而不是静默丢弃 [T1: packages/core/session/src/types.ts#SessionEvent]。

流式输出与工具调用**全部走 `session.event`，没有专门的方法**。一次文本 turn 的实际顺序（快照实测）：`agent/inbox/spliced` → `session.status running` → `turn/start` → `agent/inbox/spliced`(移除) → `step/start` → `user/message` → `session/title` → `request/header` → `request/context` → 一串 `assistant/chunk`（`block-start` / `reasoning-delta` / `text-delta` / `block-end` / `usage` / `finish`）→ `assistant/message` → `step/end` → `turn/end` → `session.status idle`。

**工具调用不回到宿主**：模型的 tool call 以 `assistant/chunk{chunk.type:'tool-call-delta'}` 增量流出，落成 `tool/call{callId,name,arguments}`，由 runtime 内的插件执行，结果以 `tool/result` 事件出现。宿主只是观察者，**线上没有 tool 往返、没有审批请求**（server→client 请求是死能力，见陷阱）。

`turn/end.data.reason.kind`：`completed` / `aborted` / `blocked` / `error` / `max-tokens` / `interrupted`（merge-extensible）。`subagent.finished.stopReason`：`completed` / `aborted` / `error` / `max-tokens` / `refusal`；`status` 由 `maxTokensAsSuccess` 配置决定是否把 `max-tokens` 映射成 `ok`。

## 会话与并发模型

- **一个 runtime 进程 = 一棵 Cordis 树 = 多个 session**。服务端维护 `sessionId → AgentHandle` 的 Map，`getOrCreateSession` 用 `sessionCreations` 做并发合流（同 id 并发只创建一次，失败后可重试）。
- **同 session 的多个 prompt 直接排队不阻塞别的 session**：`prompt` 只是 `agent.followup(message)` 后立即返回；测试 `queues overlapping prompts for one session without blocking other sessions` 就是这条契约。
- 隔离边界：session 之间共享同一进程的插件、凭据、文件系统与 cwd。**要强隔离就一个任务一个进程**——仓内 `subagent-dsh-sdk` 正是每次 run 起一个全新 runtime。
- 会话作用域是**客户端的事**：两份参考客户端都自己维护 `sessionParents` 映射（只从 `subagent.started` 学习），再用祖先链把 `session.event` / `subagent.*` 过滤到一棵树。
- "一次 run" 的判定法（两端一致）：**先订阅，再发 prompt**；忽略一切通知直到看到 `session.event{type:'agent/inbox/spliced'}` 且 `data.inserted[].id === messageId`（durable 入队回执）；此后开始收集，直到该 session 的 `session.status idle` 为止 [T1: packages/sdk/client/src/api.ts#HarnessSession.run]。
- 服务端 agent 被外部处置（例如 agent-loop reload）后，`prompt` 会先用 `ctx.agents.get()` 校验再投递，否则抛 `session agent was disposed outside the server`；不校验的话 followup 会被静默吞掉。

## 错误处理与错误码

- `-32601`：**只在完全没装 request handler 时**产生（`method not found: X`）。真实服务端装了 handler，所以**未知方法走的是 `-32603`**，message 为 `unknown DeepSeek Harness SDK runtime method: <method>`。别按 JSON-RPC 惯例去匹配 `-32601`。
- `-32603`：handler 抛出的任何异常，message = `error.message`（非 Error 值 `String()` 化）。业务错误（provider 无 adapter、maxTokens 非法、session agent 已销毁）**全都压在这一个码里**，只能靠 message 区分。
- 错误响应的 `code` 与 `data` 原样保留给客户端（`JsonRpcResponseError` / Python `JsonRpcError`）。
- 客户端侧的四类错误面（TS）：`JsonRpcResponseError`、`RequestTimeoutError`、`SdkProtocolError`、`TransportClosedError`（消息里带 exit code + 最多 400 行 stderr tail）。Python 对应 `JsonRpcError` / 内建 `TimeoutError` / `SdkProtocolError` / `TransportClosedError`，共同基类 `HarnessError`。
- **没有 wire 级 cancel**：超时只是客户端放弃（abort 掉 pending 条目），服务端仍在跑。放弃一个 turn 的唯一手段是关进程。
- 关闭阶梯（TS `disposeRuntimeProcess`）：`shutdown` 请求（默认 1000ms 上限）→ stdin EOF（`disposeEofGraceMs` 6000）→ SIGTERM（`disposeGraceMs` 3000）→ SIGKILL（同 grace，超时抛错）。Windows 跳过 SIGTERM 这级。Python 版更粗：`shutdown` → 关 stdin → `terminate()` → `wait(shutdown_timeout)` → `kill()`。
- runtime 进程侧：stdin EOF 与 SIGTERM 都 dispose root 后 `exit 0`，SIGINT 退 130；**EOF 会切断进行中的 turn**，要有序收尾必须用协议 `shutdown`。

## 非 TS/Python 宿主的实现清单

最小可用宿主 = 一个 spawn + 一个读行循环 + 三个请求 + 四个通知。照 `packages/sdk/client/tests/fake-runtime.ts`（服务端方向）与 `client.py`（客户端方向）对照即可。

必须做：

1. **spawn**：`stdio: [pipe, pipe, pipe]`，环境里给 `DSH_CORDIS_CONFIG`（或 argv[2]）指向一个含 `@deepseek-ai/dsh-sdk-jsonrpc-server` 条目的 `cordis.yml`；模型凭据用 `DEEPSEEK_API_KEY` / `DEEPSEEK_BASE_URL`。
2. **写帧**：`JSON.stringify(frame) + "\n"`，写完 flush。请求帧 `{jsonrpc:"2.0", id, method, params}`；`params` 必须是对象。
3. **读帧**：按 `\n` 切、UTF-8 增量解码、跳空行、JSON 解析失败就跳过；按 `id`/`method` 三分类；用 id 表关联响应（id 可能是 string 或 number，建议统一转字符串做 key，Python 版就是这么做的）。
4. **`initialize`** 一次，拿到 `serverInfo` 即视为 ready（可选：校验 `name === 'deepseek-harness-sdk-runtime'`）。
5. **`session/prompt`**：自己生成稳定 `sessionId`（两端都用 `session-<hex>` 风格，纯客户端自定义，服务端不校验格式）；`contentBlocks` 至少支持 `{type:'text', text}`。
6. **通知分发**：**订阅要早于 prompt 写出**，否则会丢入队回执。至少处理 `session.event` 与 `session.status`。
7. **run 收敛**：入队回执（`agent/inbox/spliced` 含你的 `messageId`）→ 收集 → 该 session 的 `session.status idle`。想要"最终回答"就取区间内最后一条 `assistant/message` 的 `content` 里 `type==='text'` 的块拼接；想要结束原因就取最后一条 `turn/end` 的 `data.reason.kind`。
8. **关闭**：`shutdown` → 关 stdin → SIGTERM → SIGKILL，每级带超时；同时把 stderr 尾巴留着做诊断。

可以不做：客户端→服务端 notification（服务端不装 handler，静默丢弃）、服务端→客户端 request（服务端从不发）、batch、`-32700`、协议版本协商（不存在）、`session.event` 的全量类型建模（按 `type` 字符串分支即可，未知类型见 `ignorable` 规则）。

想流式渲染 token：订阅 `assistant/chunk`，`chunk.type` 里 `text-delta` / `reasoning-delta` 带 `index` 与 `text`，`block-start` / `block-end` 划分块边界，`usage` 带 token 统计，`finish` 带 `reason`。想展示工具活动：`tool/call` 拿 `callId`/`name`/`arguments`（arguments 是**未解析的 JSON 字符串**），`tool/result` 拿 `content` 与 `isError`。

## 陷阱

1. **未知方法回 `-32603` 不是 `-32601`**；`-32601` 只在服务端根本没装 handler 时出现。按码分流的宿主会判断错。
2. **`messageId` 不是结果句柄**，只标识入队的 `UserMessage`。没有 per-prompt 结果、没有 turn 归属；`finalResponse` 是"区间内最后一条 assistant 文本"，steering 与注入上下文可能抢在 idle 前贡献内容。
3. **`session.event` 是全量广播**，包含非 SDK 创建的 session。不做客户端过滤会串台。
4. **`serverInfo.version` 恒为 `'0.0.1'`**，硬编码在 `server.ts` 里，与 npm 包版本 `0.1.0-rc.8` 完全脱钩，且没有协议版本协商。别拿它做兼容性判断。
5. **`initialize` 没有重入保护**：类注释写"reinitialization is unsupported"，但代码里没有任何 guard——第二次调用会改掉后续新建 session 的 route，已存在的 session 不受影响，静默产生混合状态。
6. **`session/prompt` 可以先于 `initialize`**，此时默认 `model` 是 `'deepseek-official'`（一个 provider 名被当成 model 名），会以难懂的方式失败。
7. **Python 客户端有未绑定队列**：没有任何 subscriber 匹配的通知会堆进 `HarnessClient._notifications`（`queue.Queue()` 无上限）；TS 客户端则直接丢弃。长跑的 Python 宿主如果不 drain `next_notification()`，两次 `run()` 之间的通知会持续累积。
8. **`fake-runtime.ts` 头注释与 `client.ts` 的 `SdkProtocolError` 注释都提到 `session.finished` 通知与 `accepted: true` 字段——协议里都不存在**（实际是 `{messageId}` + `session.status idle`）。这是残留的旧协议描述，照注释实现会实现出一个不存在的方法。
9. **`subagent.finished` 只覆盖 in-process 子 run**；out-of-process backend 跑的子 agent 不会有这条通知，只有 `subagent.started`（若子 session 在同一 runtime 里创建）。
10. **stdout 污染是部署侧问题**：cordis.yml 里任何 console/stdout logger 都会把协议通道弄坏，服务端插件不检查也不否决。
