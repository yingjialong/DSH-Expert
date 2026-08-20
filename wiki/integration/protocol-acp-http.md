---
title: ACP 与 HTTP API gateway（进程外集成的两条协议路线）
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/acp/acp/README.md
  - packages/acp/acp/src/index.ts
  - packages/acp/acp/src/codec.ts#turnEndToStopReason
  - packages/acp/acp/src/content.ts#supportsAcpImagePrompts
  - packages/acp/acp/package.json
  - docs/postmortem/0001-acp-default-export-drops-inject.md
  - packages/examples/acp-demo/src/index.ts
  - examples/acp-agent/cordis.yml
  - packages/subagent/subagent-acp/README.md
  - packages/host/README.md
  - packages/host/webserver/src/index.ts
  - packages/host/apiproxy/README.md
  - packages/host/apiproxy/src/fetch/handler.ts#toFetchHandler
  - packages/client/connection/src/index.ts
  - packages/client/connection/src/api-request-trust.ts#isTrustedApiRequest
  - packages/client/connection/src/rpc-host.ts#createSharedFetchHandler
  - docs/api-gateway.md
  - packages/api/README.md
  - packages/api/gateway/src/index.ts#TypertGatewayService
  - packages/api/gateway/src/types.ts#TypertGatewayErrorCode
  - packages/api/remotes/src/agent-lookup.ts#createApiRemoteAgentResolver
  - packages/api/remotes/src/remote-events.ts#API_REMOTE_FORWARDED_EVENTS
  - packages/typert/protocol/README.md
  - packages/typert/generator/README.md
  - packages/typert/registry/README.md
  - packages/typert/loader/README.md
  - packages/host/plugin-inventory/src/index.ts
  - package.json
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

# ACP 与 HTTP API gateway

路由约定：ACP 的**协议语义**看 `packages/acp/acp/README.md`（那张 Protocol contract 表是权威）；ACP 的**结算/取消时序**只能看源码 `packages/acp/acp/src/index.ts`（README 是它的散文投影）。HTTP 侧的**编程模型**看 `docs/api-gateway.md`，**方法清单与业务语义**看 `packages/host/apiproxy/README.md`，**鉴权**看 `packages/client/connection/src/api-request-trust.ts`。

## ACP 是什么，automation-only 意味着什么

`@deepseek-ai/dsh-acp` 是 Agent Client Protocol 的 **server 端**（agent 侧），在 stdin/stdout 上开 `AgentSideConnection`，依赖 `@agentclientprotocol/sdk` 固定版本 `0.25.1`[T1: packages/acp/acp/package.json]。它自我定位是 **transport adapter，不是 capability seam，也不是 UI 集成**。

"automation-only"是可执行的约束，不是措辞：
- **入站方法只有 5 个**：`initialize` / `authenticate`（no-op，因为 `authMethods: []`）/ `session/new` / `session/prompt` / `session/cancel`。出站只有 `session/update` 与 `session/request_permission`。源码里 `makeAgent` 返回的对象就这些成员，没有 `loadSession`。
- **只发 committed 内容**：`ctx.on('session/event')` 只处理 `assistant/message`，逐 block 转成 `agent_message_chunk`。raw delta、reasoning、tool 活动、plan、title 一律不上线。代价明写在 README：牺牲 token 级延迟换"干净的自动化结果"，未提交的 provider chunk 与重试不会泄漏半截文本。
- **不 advertise 任何 session / editor / terminal / fs / MCP capability**；`session/new` 收到非空 `additionalDirectories` 或非空 `mcpServers` 直接 `invalidParams` 拒绝，`cwd` 必须绝对路径[T1: packages/acp/acp/src/index.ts#validateSessionParams]。
- **图片是条件能力**：`initialize` 时算 `supportsAcpImagePrompts`，要求同时挂了 durable attachment store **且**配置的 exact provider/model 解析出显式 image input；只收 `image/png` `image/jpeg` `image/webp` `image/gif`。audio 与 embeddedContext 恒 false。
- 仓内的 ACP **client** 在 `packages/subagent/subagent-acp/`（作为 subagent provider），与本包是两端，别搞混。

## ACP server 的启动方式与能力边界

`pnpm run demo:acp` = `node --import tsx packages/examples/acp-demo/src/bin.ts --config examples/acp-agent/cordis.yml`[T1: package.json]。真正的应用装配在 `packages/examples/acp-demo/`：agent spine + JSONL persistence + checkpoint policy + sqlite session-query + acp bridge，一个有序 effect 保证 query service 先就绪、ACP session 先 quiesce 再拆 persistence。

插件本身只有两个 config 键：`provider` / `model`（都 optional，但可运行的 ACP composition 两者都要）。`stream` 是 runtime-only 测试注入口，不在 `Config` schema 里。

能力边界的几条硬线（README 的 Known Limitations 是选型依据）：
- **fresh sessions only** —— load / list / resume / delete / fork 全不支持。
- **一个 workspace**；resource link 只会被拍平成 `[resource_link name=… uri=…]` 文本，不会去取内容。
- **连接级生命周期** —— 没有 per-session close，一个连接释放它的全部 session。
- 每个 session **同时只允许一个 in-flight prompt**，第二个直接 `invalidParams`。

关于 `stopReason` 的准确语义（这是集成方最容易误读的地方）：bridge **不声称**这是 prompt 级的 turn 结果。区间从 prompt 进入 Agent inbox 开始，到 admission + whole-Agent idle + 有序输出投递三者都静默为止；steering 与注入的工作可能在 idle 之前也贡献输出。结算优先级是：显式取消 > 输出投递失败 > 区间级 Agent 失败 > 相关 turn 的 ending。

## HTTP API gateway（packages/host）：路由、鉴权、会话映射

分层是 `remotes → gateway → connection → webserver`[T1: packages/api/README.md]，注意 **Connection 和 WebServer 都不在 `packages/api` 下**：Connection 在 `packages/client/connection`，WebServer 在 `packages/host/webserver`。

**路由**：`dsh-host-webserver` 是纯 `node:http` 载体（`ctx.webServer`），不认识任何 harness 概念、不服务任何文件。它只有三张表：`exact` / `prefix` / 单一 fallback，匹配顺序固定为 exact → 最长 prefix → fallback，注册顺序无语义。`/api` 这个 prefix 路由是 **connection 插件**注册的，不是 webserver 自带的。SPA dist 由 `dsh-host-frontend-static` 占 fallback 席位。`host` 只接受 `127.0.0.1` 与 `0.0.0.0` 两个值。

**鉴权：没有鉴权**。存在的只是一道 reachability fence（`isTrustedApiRequest`），README 明说 "The fence is a reachability policy, not authentication"。它做三件事：
1. **Host 头栅栏（抗 DNS rebinding）** —— 对每个请求生效，不给"无 browser 标记"开后门，因为明文 HTTP 下浏览器的 image/navigation 读请求既无 `Origin` 也无 Fetch-Metadata。必须是 loopback 或命中 `trustedHosts`。
2. `sec-fetch-site: cross-site` 直接拒。
3. 带了 `Origin` 就必须等于 Host authority（字面量 `"null"` 拒）。

在此之上还有一层 **privileged 方法 loopback pin**：`PRIVILEGED_METHODS` 里的 15 个方法（`agentPreset.read/copy/openDocument/remove`、`host.pickDirectory`、`host.openPath`、整个 `settings.*` 与 `credentials.*` 配置面、`llm.discoverModels`）额外用**空信任表**再过一次 fence，等于钉死在 loopback[T1: packages/client/connection/src/index.ts]。`agentPreset.list` / `select` 故意不在其中，理由写在源码注释里：默认 composition 本来就带 bash，钉住"切换"而不钉住"创建"是"敞开的门旁边立一道栅栏"。

**会话映射**：wire 上只有 `sessionId`，没有 agent 句柄。Host 侧的统一映射策略是 `createApiRemoteAgentResolver`（`packages/api/remotes`）：复用活着的 Agent → 否则**自动 resume 冷 session**（同一 id 的并发 resume 去重）→ subagent 拥有的 id 用 `agent-busy` 围栏拒绝（`hasApiRemoteSubagentOwner`：`header.origin === 'subagent'`，或父 session 活着且 `agents.isOwnedBy`）。这一个 resolver 同时被 legacy API Proxy 与 Typert 的 `agent` / `session` lookup 复用，所以迁移与未迁移的方法共用一套身份策略。

**carrier 的分层规则**（`toFetchHandler`）：HTTP status 只表达载体层 —— 404 未知路径 / 415 非 `application/json` / 400 body 不是 JSON / 500 handler 崩溃；**业务错误一律 200 + `ServerResponse` 的 error 分支**。415 那一条是跨站写栅栏：浏览器的"简单 POST"不触发 preflight，强制 JSON media type 把它推进一个本服务器永不回答的 preflight。

## packages/api 的 Remote BFF 与 Typert RPC gateway 是什么关系

两个包、四张脸（Host/Client × gateway/remotes）：

| 包 | Host 面 | Client 面 |
|---|---|---|
| `dsh-api-gateway` | `ctx.typertGateway`：认领 endpoint、解析 descriptor、校验参数、解析身份、调用 Service、校验返回值 | `ctx.remote`：把生成的 descriptor 装成具体方法，经 Connection 发起调用 |
| `dsh-api-remotes` | 只做**身份策略**（配置 typert lookups）+ 转发事件白名单，本身不注册 service | 显式挑选并 `$mount()` 允许的 `/remote` contribution |

分工的关键：**gateway 是机制，remotes 是策略**。gateway 不知道哪些业务包该被暴露；remotes 用 build-time 的 value import 显式选（当前只挂了 Goal 与只读的 `pluginInventory/list`），Client **不会**在运行时发现 Host 有哪些 Service。

`/api` 上两条栈共存：Typert Gateway 通过 `connection.rpc.intercept('/api', …)` 注册**唯一**一个拦截器，只认领**两段式** endpoint（`<namespace>/<method>`）且有 strict descriptor 或活跃 SRC marker 的；认领不了的落回 legacy API Proxy。`createSharedFetchHandler` 的选择是互斥的：先看拦截器 `matches`，命中才走 gateway，否则整个请求交给 fallback。

转发事件是第三条通路：`ctx.remote.$on` 的合法 key 集合就是 `API_REMOTE_FORWARDED_EVENTS` 这一个数组（11 个，含 `settings/document-updated`、`commands/change`、`llm/adapters-updated`、`cordis/*` 一族）。数组用 `satisfies readonly TypertForwardableEvent[]` 在编译期把三件事钉死：名字必须是已声明事件、不得绑定 Scope、必须是单向（waterfall/bail 形状被排除）。

## Typert：类型图生成 + artifact 加载 + runtime registry

集成方需要知道它，因为 **Remote 的类型契约不是手写的，是构建产物**；改一次签名要按序重跑构建，否则 Client 端拿到的是旧契约。

四个包：`protocol`（纯声明：`@Remote` / `@RemoteScope` / `TypertRemoteService` / `bindTypertRemote` / `InvocationDescriptor` / 各种 merge-extensible map，不做分析也不注册 service）、`generator`（build-time，从 Host `ts.Program` 生成）、`registry`（`ctx.typert`，运行时存 reflection 与 Zod schema）、`loader`（扫 Loader entry，import 各包的 `./typert` 并注册）。

流水线顺序是硬的：`build:lib:host`（`tsc -b tsconfig.host.json` → `tsdown --env.DSH_BUILD_FACE host`，generator 在这一趟跑，**Host aggregate 是它唯一的 program 种子**）→ `build:lib:client` → `build:web`。业务包产出 `lib/typert.host.{js,d.ts}`（Host Loader 用）与 `lib/typert.remote-client.{js,d.ts,d.ts.map}`（`api-remotes` 用），分别通过 `./typert` 与 `./remote` 两个 export 暴露；generator 会校验这两个 export 与 `files` 清单存在，否则不给你生成。

strict 分析对 Remote 方法的要求很硬：public、非 static、实例方法、有具体实现、**不能是泛型**；参数必须是必填的简单标识符，禁止解构 / 默认值 / rest / optional。复杂 Host 对象（如 `Agent`）必须有唯一的 `TypertLookupMap` 声明，参数名 `agent` 会生成 wire 字段 `agentId`。取消要写成最后一个参数 `signal: AbortSignal`，它进 descriptor 而不进 `args`。

**SRC fallback** 是给 `node --import tsx/esm` 源码启动的 Host 用的：装饰器 initializer 在模块私有 `WeakMap` 里留下方法名与调用模式，Gateway 据此拼一个弱 descriptor（只查 JSON 安全性，不读 TS 类型、不生成 Zod）。但它**只解决 Host 侧 dispatch**：Client 拒绝挂载没有 strict codec 的 SRC descriptor，Client 的类型与 codec 永远来自最近一次生成的 artifact。

## 三种进程外集成方式的对比

| | JSON-RPC SDK（`packages/sdk`） | ACP（`packages/acp`） | HTTP API gateway（`packages/host` + `packages/api`） |
|---|---|---|---|
| 传输 | 子进程 stdio，newline-delimited JSON-RPC | 子进程 stdio，ACP JSON-RPC | 本机 HTTP `POST /api/*` + 两条 WebSocket downlink |
| 谁是目标用户 | 外部宿主（TS / Python），**这是唯一为第三方设计的通路** | 已经会说 ACP 的宿主（编辑器、父 agent） | dsh 自家 Web GUI |
| 能看到多少 | 全部 durable 事实（`session.event`）+ whole-agent 状态（`session.status`） | 只有 committed 的 assistant 文本/图片 | 全部（history / 队列 / jobs / projection / 设置面 / 搜索…） |
| session 生命周期 | 命名或新建，可复用 | 只能新建，随连接死 | 创建 / 列表 / fork / rename / 归档 / 冷 session 自动 resume |
| 鉴权 | 无（你 spawn 的子进程就是信任边界） | 无（`authMethods: []`） | 无鉴权，只有 loopback/trustedHosts reachability fence |
| 版本协商 | `initialize` 是 readiness 边界，`serverInfo.name` 线稳定 | 单版本 agent，`PROTOCOL_VERSION` 即结论 | **没有协议版本字段**，client 与 host 必须同版本发布 |
| 主要代价 | 你要自己准备 runtime 可执行文件与 `cordis.yml` | 拿不到流式/推理/工具活动，只能新建会话 | 面向自家前端，方法集与语义随版本变；跨版本没有兼容承诺 |

选型结论：**要嵌进自己的产品 → SDK**；**已经是 ACP 生态（编辑器）→ ACP**；**想复用 dsh 的 Web 后端能力 → HTTP，但要接受版本锁死与零鉴权**。

## 陷阱

1. **ACP 的 stdout 是协议本身**。composition 里不能有 stdout logger，也不能开 HMR —— `examples/acp-agent/cordis.yml` 的文件头就是这么写的。诊断只能走 stderr。
2. **namespace plugin 与 `export default` 互斥**。Loader 的 `unwrapExports` 优先取 `.default`，一旦有 default export，`name` / `inject` / `Config` 这些兄弟命名导出会被整体丢弃，`apply` 在没有任何注入的 fiber 里跑，首行读 `ctx.agents` 就抛 `cannot get property "agents" without inject`。这是 postmortem 0001 的 Bug #1，`acp-demo/src/index.ts` 的模块注释至今引用它。
3. **可选服务必须用 `ctx.get(name)`，不能用 `ctx.<name>`**。属性代理走的是**只向祖先**的 fiber walk，穿过 traceable shadow 时会走到 root 然后抛；`ctx.get` 走全局 isolate store，与拓扑无关。这是 postmortem 0001 的 Bug #2。ACP 拆连接时读 `ctx.get('subagents')` 就是这个模式（且是结构化读取，不依赖 subagent seam 包）。
4. **手搭 `ctx.plugin({...})` 的测试永远测不出加载路径**：`unwrapExports` 只被 Loader 调用。至少要有一条走真实 Loader + 真实进程的测试；不调模型的那条不需要 API key，应该进 CI。
5. **`max-tokens` 在 prompt 路径上报 `end_turn`，不是 `max_tokens`**。`turnEndToStopReason` 里确实有 `max-tokens → 'max_tokens'` 分支，但 `settleAfterQuiescence` 在调用它之前先短路成 `end_turn`（注释："Token-limit and other non-terminal endings are not prompt-level stop reasons"）。靠 `stopReason` 判断是否被截断会失效。
6. **`agentInfo.version` 是硬编码的 `'0.0.1'`**，与包版本 `0.1.0-rc.8` 无关，不要拿它做版本判断。
7. **`agent` / `session` lookup 是按 key 配的，没有 per-endpoint 的"只接受活对象"策略**。任何 Remote 方法拿到的 Agent 都可能是刚被冷恢复出来的，且业务方法**不许猜**它来自哪儿。
8. **Gateway 到 RPC 的错误映射会丢结构**：普通 dispatch 失败与业务异常都被压成 RPC `internal` + 空 details；17 个 `TypertGatewayErrorCode` 只对同进程调用者可见。只有用 `TypertLookupFailure` 包装的 lookup 策略错误（冷恢复失败、ownership 围栏）保留原始 code。
9. **Typert Remote endpoint 没有 privileged-method 机制**。Gateway 注册拦截器时用的是 `{ authority: 'trusted-host' }`，而 `PRIVILEGED_METHODS` 的 loopback pin 写在 fallback handler（即 legacy API Proxy 路径）里 —— 把一个敏感方法迁到 Remote，就等于把它从 loopback pin 里搬了出来。
10. **`typert-loader` 把包解析与 manifest 缓存到进程结束**，新增 `./typert` export 必须重启进程才生效。
11. **`api-remotes` 是全仓唯一的双 TS 面包**，引用它必须写 `tsconfig.host.json` 或 `tsconfig.client.json`，包根 `tsconfig.json` 只是 solution，不能进任何 aggregate 的依赖图。别照抄它的 `clientBundle(..., { hostPhase: true })`。
12. **postmortem 0001 里的 ACP 代码不是当前状态**：它写 `inject = ['agents', 'sessions', 'sessionPersistence']` 且讨论 `session/load`，而当前源码是 `inject = ['agents']`、根本没有 `loadSession`。postmortem 是历史档案，取教训不取 API。
13. `/api` 的 HTTP bridge **把每个 request body 整个读进内存**，`maxRequestBodyBytes` 默认 160 MiB —— 它同时就是单请求常驻内存上限。
14. `host/remote-event` 帧寄生在 legacy apiproxy 的 `HostFrame` 联合里，读起来像是 apiproxy 拥有 Remote 事件契约 —— 并不是，白名单归 `api-remotes`，消费动词是 `ctx.remote.$on`。这一条被上游自己列进了 Known Limitations。
