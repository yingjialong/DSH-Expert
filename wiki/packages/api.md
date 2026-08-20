---
title: packages/api — Remote BFF 与 Typert RPC 网关
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/api/README.md
  - packages/api/gateway/README.md
  - packages/api/remotes/README.md
  - packages/api/remotes/src/remote-events.ts
  - packages/typert/protocol/README.md
  - docs/api-gateway.md
  - docs/subsystems/typert.md
  - packages/README.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

Web GUI 的**应用层 Remote 栈**：`remotes` 拿 BFF 策略（Agent/Session 身份解析 + 选哪些业务 Remote），`gateway` 实现 Host/Client 两侧共用的 Typert **unary RPC** 端点 [T1: packages/api/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

| 包名 | npm 名 | 一句话职责 |
|---|---|---|
| `remotes` | `@deepseek-ai/dsh-api-remotes` | 两侧 BFF：Host 侧 Agent/Session 身份策略；Client 侧装配并 `$mount()` 各 Remote 贡献 |
| `gateway` | `@deepseek-ai/dsh-api-gateway` | 两侧 Typert RPC 端点：Host 提供 `ctx.typertGateway`，`/client` 入口提供 `ctx.remote` |

组 README 的 ctx key 一栏：`remotes` **不注册 service**（它配置 `ctx.typert`、消费 `ctx.remote`）；`gateway` 是 `ctx.typertGateway` / `ctx.remote`。

## 三件套结构

本组不是典型 capability family，而是**两侧对称的传输/装配层**，按"契约 / 实现 / 装配"读更准：

- **契约（Service Definition 位）**：`dsh-typert-protocol` 的 `@Remote` / `@RemoteScope` 标注与 `TypertRemoteService` 基类，以及共享的 `TypertClientRemote` 接口。契约不在本组，在 `typert/` 组。
- **实现（Provider 位）**：`api-gateway`（Host 的 `TypertGatewayService`，Client 的 `ClientRemote`）。
- **装配（Consumer 位）**：`api-remotes`——选哪些业务 Remote 上线、身份策略怎么定；Client 业务包依赖这个 facade，**不依赖 gateway 实现**。

**运行期依赖方向（跨文档综合，单篇看不出）**：`remotes → gateway → connection → webserver` [T1: packages/api/README.md]。注意 `connection` 实际住在 `client/connection`、`webserver` 住在 `host/webserver`，不在本组。

## 扩展点

- **加一个业务 RPC 方法** → 让业务 Service 继承 `TypertRemoteService`，方法上打 `@Remote` / `@RemoteScope`；继承链被别的基类占用时用 `bindTypertRemote()`。
- **让 Client 能调它** → 在 `api-remotes` 的 Client 装配里显式 `import` 该包生成的 `/remote` 制品并 `ctx.remote.$mount()`。**没有运行时发现机制**，必须是 build-time value import。
- **改身份解析策略** → `createApiRemoteAgentResolver()`；Host composition 可用 effect-scoped `ctx.typert.lookups.configure()` 覆盖解析行为。
- **转发一个 Host 事件到浏览器** → 往 `packages/api/remotes/src/remote-events.ts` 的 `API_REMOTE_FORWARDED_EVENTS` 数组里加一项，**就这一处**：类型投影、consumer key face、Host 转发循环全部从它派生 [T1: packages/api/remotes/README.md#Forwarded Host events]。
- **支持取消** → Remote 方法把 `signal: AbortSignal` 作为**最后一个 Host 参数**；它是 descriptor 元数据，不是 wire 参数。
- **复用 Client 面**（例如未来做 TUI）→ 只要提供同样 React-free 的 `ctx.remote` 契约即可复用 `api-remotes` 的 Client 半边。

## Known Limitations

组 README 自己的 `## Known Limitations and Deferred Work`：

- `connection` 与 `webserver` 仍留在 `client/connection` / `host/webserver`；将来可纯包级搬到 `api/connection`、`api/webserver`，service 契约不变。
- 旧的 **API Proxy 仍在 `host/apiproxy`**，作为尚未迁到 Remote 的方法的 fallback；它消费 `api-remotes` 拥有的 Host resolver，所以新旧方法共用一套身份策略。

`gateway` 侧要点：dispatch 失败与业务异常在 Connection 适配层被统一映射成 RPC `internal`（details 为空），只有 `TypertLookupFailure` 携带的 lookup-policy 错误原样返回；结构化 `TypertGatewayError` 分类只对同进程调用者可见。SRC mode 只支持无解构/无默认值/无 rest 的唯一标识符参数，且只校验 JSON 安全性；Client 面只接受 strict 生成的贡献；**只支持 unary**，增量 Session 数据走同一 Connection 上的独立 named-stream 协议；lookup resolver 是按 key 配置的，单个参数/端点无法在同一 `agent`/`session` key 下选 live-only 策略；转发事件**无投影、无脱敏、无 Scope 绑定、重连后不重放**。

`remotes` 侧要点：能力集由显式 build-time value import 固定，Client 不在运行时发现 Host 的活跃 Service；标准 Web Host 的 resume 默认值与 Agent-scope 设置仍由 legacy API Proxy 提供。

## 陷阱

1. **`api-remotes` 是全仓唯一被允许拆 tsconfig face 的普通包**（`api/remotes`）：Host 入口要进 Host Typert 图，而 `src/client/index.ts` 必须等 Host tsdown 生成完 `/remote` 声明才能编译。**不要照抄这个拆分**——普通 Client 插件即使同时有 `src/index.ts` 和 `src/client/index.ts` 也是单 project。
2. `src/remote-events.ts` 和 `src/types.ts` **被两个 face 的 `files` 同时列出**，这是刻意的（allowlist 必须是唯一控制点），且**没有 gate 保证两 face 源文件互斥**。看到重复列出不要"顺手清理"。
3. `ctx.remote.$dispatch()` 是**载体层**的另一半，业务消费者只 `$on`、永不调用 `$dispatch`；没人订阅的事件名直接丢弃。
4. **撤回一个已被观测到的 strict 定义会直接失败**，而不是降级成弱校验。
5. Client 侧方法查找与调用**用普通对象和函数，不是 Proxy**；Client 入口里没有 Host Service，也没有 Host Cordis interface merge。
6. 生成的 cancellation-aware Client 方法接受一个可选的末位 `AbortSignal`，Client 会把它与**贡献的 mount 生命周期**合并——撤回贡献会中止 in-flight 调用，并让已持有的方法句柄开始 reject。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 组分工与 `remotes → gateway → connection → webserver` 方向 | `packages/api/README.md` |
| Host `invoke()` 校验/解析流程、strict vs SRC、错误分类 | `packages/api/gateway/README.md#Host service` |
| Client `$mount` / `$on` / 命名空间 Service 生命周期 | `packages/api/gateway/README.md#Client service` |
| 身份解析、冷 session resume、subagent ownership fence | `packages/api/remotes/README.md` |
| 转发事件 allowlist（改这里就够） | `packages/api/remotes/src/remote-events.ts` |
| 双 face 构建边界为何存在 | `packages/api/remotes/README.md#Build boundary` |
| `@Remote` / `@RemoteScope` 标注本身 | `packages/typert/protocol/README.md` |
| 网关的跨文档主篇 | `docs/api-gateway.md`、`docs/subsystems/typert.md` |
