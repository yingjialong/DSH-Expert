---
title: packages/host — Web GUI 的 host 半（API gateway + HTTP 承载 + 目录选择 seam）
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/host/README.md
  - packages/host/apiproxy/README.md
  - packages/host/apiproxy/src/api-proxy.ts
  - packages/host/apiproxy/src/api/sessions.ts
  - packages/host/apiproxy/src/api/events.ts#EventsApi.mux
  - packages/host/apiproxy/src/api/approvals.ts#ApprovalResponsePayload
  - packages/host/apiproxy/tests/api-proxy-approval.spec.ts
  - packages/host/webserver/README.md
  - packages/host/webserver/src/index.ts
  - packages/host/frontend-static/README.md
  - packages/host/frontend-static/src/index.ts
  - packages/client/connection/src/index.ts
  - packages/client/connection/src/api-request-trust.ts
  - packages/host/directory-picker/README.md
  - packages/host/directory-picker/src/index.ts
  - packages/host/directory-picker-auto/README.md
  - packages/host/directory-picker-native/README.md
  - packages/host/directory-picker-browse/README.md
  - packages/host/plugin-inventory/README.md
  - docs/subsystems/web-server.md
  - docs/subsystems/workspace.md
  - docs/capability-seams.md
  - .agents/notes/implemented/architecture/2026-07-19-gui-layering-and-rpc-protocol.md
  - .agents/notes/implemented/architecture/2026-07-28-directory-picker-capability-seam.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-31
asked_by: agent
---

## 一句话定位

dsh Web GUI 的 host 侧：所有 client 形态共用的 API gateway（`ctx.apiProxy`）、它骑乘的裸 HTTP 服务器（`ctx.webServer`）、SPA dist 兜底、workspace 目录选择 seam，以及一个只读的 Loader 插件清单投影。浏览器侧在 `packages/client/`。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文，组 README 亦称全组 product）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `apiproxy/` | `@deepseek-ai/dsh-host-apiproxy` | 共享 API gateway 与线协议契约，提供 `ctx.apiProxy`；**不注册任何路由** |
| `webserver/` | `@deepseek-ai/dsh-host-webserver` | `node:http` 路由/upgrade 承载，提供 `ctx.webServer` |
| `frontend-static/` | `@deepseek-ai/dsh-host-frontend-static` | 占据 webserver 唯一 fallback 座位的 SPA dist 服务器 |
| `directory-picker/` | `@deepseek-ai/dsh-host-directory-picker` | **Service Definition**：`ctx.directoryPicker` 目录选择 seam |
| `directory-picker-native/` | `@deepseek-ai/dsh-host-directory-picker-native` | `kind: 'native'` 后端：在 host 显示器上开一个原生选择器 |
| `directory-picker-browse/` | `@deepseek-ai/dsh-host-directory-picker-browse` | `kind: 'browse'` 后端：应用内目录浏览的 list/create 原语 |
| `directory-picker-auto/` | `@deepseek-ai/dsh-host-directory-picker-auto` | boot 时采样一次宿主环境，挂上匹配的那个后端行 |
| `plugin-inventory/` | `@deepseek-ai/dsh-host-plugin-inventory` | 当前 Loader 条目的只读投影，发布 Remote `pluginInventory/list` |

注意目录名与 npm 名的映射规律：本组一律加 `host-` 前缀（`apiproxy` → `dsh-host-apiproxy`）。

## 三件套结构

本组里**只有目录选择器是真 seam**（`docs/capability-seams.md` 中 `ctx.directoryPicker` 的 Role 是 `seam`；`ctx.webServer`、`ctx.apiProxy` 都是 `core`）：

- **Service Definition**：`@deepseek-ai/dsh-host-directory-picker`。抽象 `DirectoryPicker` 服务，**唯一方法是 `capability()`**，返回一个判别联合描述「操作者怎么选目录」。
- **Service Provider**：`-native`（`{ kind: 'native', pick(signal) }`）与 `-browse`（`{ kind: 'browse', list(path?), createDirectory(path, name) }`）。两者**在用户交互形态上就不同**，不只是实现不同。
- **Consumer**：`apiproxy`（把 `host.pickDirectory` / `host.listDirectory` / `host.createDirectory` 委派下去，并把后端的 typed failure 1:1 映射成 wire error code）。
- **组合器**：`-auto` 不是第三个后端，它是一个 node-half-only 插件，boot 时把选中的后端**作为真实 Loader entry 挂进内存根树**。

其余包是「服务 + 消费者」直连：`frontend-static` 消费 `ctx.webServer` 的单一 fallback 座位；`connection`/`modules`/`hmr`（在 `packages/client/`）注册各自的路由。

## 扩展点

- **加一种新的目录选择交互** → 依赖 `@deepseek-ai/dsh-host-directory-picker`（Service Definition），通过**声明合并**往 merge-extensible 的 `DirectoryPickerCapabilities` map 里加自己的 variant，然后注册 `ctx.directoryPicker`。消费者 `switch (capability().kind)`，遇到未知 kind 应当**隐藏目录选择而不是报错**。
- **`capability()` 返回的对象在整个 service 生命周期内必须稳定**——这是 seam 的硬性契约（`-auto` 之所以「每 boot 只采样一次」就是为了满足它）。
- **每个后端包还有一个 browser entrypoint**，向 ui-workspace 的 directory-flow slot 注册配套交互，所以**一行 composition 同时选定 host capability 和 client flow**。
- **加 HTTP 路由** → 依赖 `ctx.webServer`：`register(route)` 加具名 `exact`/`prefix` 路由，`registerUpgrade(route)` 加精确 pathname 的 upgrade 路由，两者都返回 disposer；`registerFallback(handler)` 是**单所有者**座位（第二次注册抛异常）。HTTP 匹配顺序**固定**：全表 exact → 最长 prefix → fallback。
- **往 index.html 注入启动输入**（0.1.1 起）→ 首选**结构化注入**：监听 `webserver/index-inject` 事件（emit 模式，每次 index 渲染与 worker boot-payload 请求都触发），往传入的 `IndexInjection[]` 表里 push 当前行；fallback owner 通过 `renderIndex(html)` 渲染（结构化行在前，raw 变换在后）。`tapIndex(transform)` 退居**escape hatch**——只用于结构化行表达不了的 raw HTML 变换，在结构化行之后按注册顺序应用。
- **`apiproxy` 是 transport-independent 的**：它自己不注册路由，carrier（如 HTTP）自行包裹 `ctx.apiProxy`。`AbstractApiClient` 持有全部协议不变式（rpcId 铸造、信封包装/拆解、zod 解析、SSE 帧解码、unary 超时、microtask 批量的 `subscribeEnvelopes`），平台子类只提供 `doFetch`。`InProcessApiClient` over `toFetchHandler(api)` 是「不走网络但走完整线序列化/校验」的同构点。
- **加一个可被浏览器配置的插件** → 注册自己的 settings 命名空间即可，`apiproxy` 的 `settings.*` 域**无需改动**就会服务它（「仓库外分发的插件也能变成浏览器可配置」是明写的设计目标）。

## Known Limitations

组 README 无该章节；以下按包汇总。

**apiproxy**（gateway，限制最多）
- 转发的 Remote 事件**寄生在遗留的 `HostFrame` union 上**（`host/remote-event`），读起来像是本包拥有 Remote 事件契约——其实不是（allowlist 属于 `dsh-api-remotes`，消费动词是 `ctx.remote.$on`）。
- pending interaction 状态在 host 侧；`src/api-proxy.ts` 同时持有 process-local `pendingQuestions` 与 `pendingApprovals`。`packages/host/apiproxy/README.md` 声称“只有 question”已落后于同 commit 的源码与官方测试，见 `conflicts.md` C078。
- **pending question / approval 都不跨 host 重启**：registry 持有等待 promise 的 settlement callback，是 host 进程内存。`events.mux` 每次重开都会以稳定 `rpcId` 重放 still-pending question / approval（覆盖 browser reload 与 client reconnect），但 host 重启不会持久恢复它们。
- **无协议版本字段**：client 与 host 同版本发布，`host.describe` 只有在出现独立发布的 client 时才会加版本协商字段。
- 搜索失败会带上 provider 诊断信息——gateway 假定是单用户本地服务，多用户 carrier 必须替换成公开安全的诊断。
- cold-list 提示只会向「可见性更高、排序更旧」的方向退化。
- 插件 config（`ApiProxyService.Config`，`src/index.ts`）三字段的校验是 zod 源码级事实：`nativeOpen: z.boolean()`（无默认，显式钉死 `host.describe.canOpenPath` 能力，覆盖平台探测失真处）；`sessionExportCompressionLevel: z.number().step(1).min(0).max(9).default(6)`（类型 `SessionLogCompressionLevel = 0|1|…|9`）；`coldBlankProbeMaxBytes: z.natural().default(1024)`。注意 `nativeOpen` 只存在于插件 config 层——直接调 `createApiProxy(ctx, defaults)` 的 `ApiProxyDefaults` 里对应的是 `canOpenPath?: () => boolean` 回调，两层别混。

**webserver**
- **无 TLS、无 auth、无 origin 策略**。绑非 loopback 地址就等于把服务暴露给那个网络；加固或反代明确不在 dev-facing v1 范围内。
- socket 选项固定（config 只能选 bind host 与 port）。

**directory-picker 家族**
- Service Definition：**不支持多根**（browse 契约每次列举只暴露一条 ancestry 链）。
- `-native`：Linux 需要桌面工具（Zenity 或 KDialog），两者都没有时 `pick` 直接以可操作错误拒绝，**不会退化成手输路径**；Windows 无机制兜底。
- `-browse`：不读 Windows hidden 属性（`hidden` 在所有平台都指 dot 前缀）；无盘符根枚举；**全文件系统范围**，没有 per-deployment browse root。
- `-auto`：检测是从启动上下文推断操作者位置，**没有任何启动侧信号能证明这件事**——tmux 从 SSH 启动后 detach 会丢 `SSH_*` 标记；本机启动后用 `ssh -L` 访问会从 `127.0.0.1` 到达、解析成 `native`，然后在无人值守的工作站上弹出选择器。Linux 探测**只读 `PATH`**。只在 boot 时解析一次。

**frontend-static**：起步 MIME 表极简，只覆盖 Vite 产出的资产集加 PWA manifest，其余扩展名一律 `application/octet-stream`。index 服务**已显式化**（0.1.1 起）：`distIndex` 可读时 dist root 与 configured index path 渲染 `index.html`（200）、其余存在的文件直接服务，**dist root 内缺失或非文件目标（含缺失的 configured index）返回空 404**——不再对任意 miss 兜底 200；当前 client 没有 History API pathname 路由，加一条需要显式 server 规则而非放宽 fallback。

**plugin-inventory**：**只有时点状态**（无持久失败历史、无订阅，没有 live root Fiber 就报 `null`，不区分原因）；**无来源也无变更能力**（不知道条目是哪个 bundle/profile/override 引入的，也不能启用/禁用/增删插件）。

## 陷阱

1. **`webserver` 的 `host` 只接受两个值**：`127.0.0.1`（默认姿态）与 `0.0.0.0`（刻意的网络暴露）。listen 失败（EADDRINUSE 等）会抛出激活并**让 Loader 组合失败**，失败的候选 fiber 被 dispose。
2. **`webserver` 什么都不打印**——URL 那行归 shell 输出。它也不认识任何 harness 概念、不服务任何文件。
3. **`-auto` 与某个具体后端行同时挂载会 fail loud**（`directoryPicker` 服务重复 + `single` 洞里的 client flow 重复）。想固定某种交互就**直接组合 `-native` 或 `-browse` 行**，那才是 seam 文档化的替换点；`-auto` 里没有「pin 一个 kind」的 config 字段。
4. **每个 `/api` POST 都必须声明 `application/json`**，否则在 dispatch 之前就被 415 拒——这是为了让浏览器无需 CORS preflight 就能发出的「simple request」永远不可能盲打一个有副作用的方法。
5. **一整套方法被钉死在 loopback + same-origin**：`host.pickDirectory`、`host.openPath`、整个配置平面（`settings.describe`/`openDocument`/`update`/`replace`/`mutate`，`credentials.describe`/`set`/`unset`）以及 `agentPreset.read`/`copy`/`openDocument`/`remove`。围栏在浏览器 carrier `dsh-client-connection` 里。
6. **secret 只朝一个方向过线**：只能出现在 `settings.update`/`mutate`、`credentials.set`、`llm.discoverModels` 三种载荷内部，**任何响应的任何层都不返回 secret 值**。但它们确实会骑在 client 的出站信封上，`subscribeEnvelopes()` 的观察者能看到。
7. **`session.export` 不是 RPC**，是 host-only 下载面：`GET /api/session.export?...` 流式吐 ZIP，内容是 persistence backend `readRaw` 的**逐字原始字节**，绝非从已解析事件重建。
8. **`plugin-inventory` 刻意不声明同进程 Cordis `Context` 合并**——它是 Remote-only 的，client 包必须经 `api-remotes` assembly 消费，不能 import Host 实现。因此它**不出现在 `docs/capability-seams.md` 的 ctx key 表里**，别以为是漏了。
9. **`agentPreset.select` 只在 session 还是 blank 时允许**：跑过一个 turn 之后换 preset 会让已记录的 tool call 悬空，因此返回 `agent-preset-locked`。
10. **`command.execute` 只带调用方/连接取消，不受 30 秒传输健康截止约束**——command handler 合法地可以活得更久。

## 2026-08-23 agent 复审增量（session.create 的 caller-chosen identity）

- **`session.create` 的 payload 允许 caller 自带 `sessionId`，且该 id 会成为持久身份**：`packages/host/apiproxy/src/sessions/schema.ts`（约 L102-110）`sessionId` 为 optional；`api-proxy.ts` 的 create 实现里 caller id 优先（约 L2080/L2095），`checkPersistedIdentity = sessionId !== undefined`；`ensureSession`（约 L1558-1648）按 client-chosen identity 语义 **adopt / resume** 既有会话（同 cwd 通过、跨 workspace 被 `SessionCwdConflict` 拒绝），无该 id 时才由 host 铸造新 SessionId。官方留此口子的动机是**调用方重试去重**（上游注释 "deduplicated across concurrent retries"）。
- **收敛键是 `sessionId`，不是 transport `rpcId`**：Host 用 `sessionCreations` 对同 id 的并发 create 做 single-flight；live Agent 直接复用，cold persisted id 经 list/inspect 后 resume，不存在才 create。相同 cwd + 相同 explicit effective preset 收敛；省略 preset 会 adopt 既有选择；不同 explicit preset 返回 `agent-preset-conflict`，不同 cwd 返回 `session-conflict`。Workspace attach 在 Session/Agent publication 之后，attach 失败不会抹掉已创建 Session，同 id 重试可继续补 attach。
- **响应丢失后的正式只读对账面是 `session.list`，不是重放 create**：`SessionSummary`公开 `sessionId/cwd/agentPreset`；attached Session 的 preset 来自 header + 最后一条 `agent-preset/selected` 的 effective fold，cold persisted fallback 只能读 header。rc.2 没有独立 `session.getHeader` RPC，`session.history`也不返回 header。
- **对不受信 client 侧宿主的含义**：若宿主把 `session.create` 直通给不受信的 UI/进程，等于允许它自选或 adopt 任意同 workspace SessionId 作为上下文。围栏做法是在宿主代理层**剥离 payload.sessionId**（只透传 `{workspaceId}`）——官方 client runtime 的 `connectWorkspace` 路径本来就不带 sessionId（`packages/client/runtime/src/client/workspaces/service.ts` 约 L112），剥离不影响官方路径。
- **closely related**：`SessionCreateError` 携带 `requestedSessionId`，preallocated id 失败时可凭它对账（见 [integration/electron-embedding.md](../integration/electron-embedding.md) §6）。

## 2026-08-30 · rc.2 create/prompt admission与Web trust边界

- caller预分配`sessionId`是create的正式幂等键：同id/cwd可single-flight/adopt/resume，不同cwd冲突。成功响应在Agent setup/publication与可选Workspace attach后；attach失败时Session已发布并在error details返回id。React-free`SessionRuntime.create`成功前已让list/binding同步可寻址。
- prompt的`accepted:true`在附件/时区准入、UserMessage构造与`followup/steer`持久Inbox splice后返回；Host可在HTTP响应丢失后继续turn。prompt `rpcId`耐久写进MessageSource，但`AbstractApiClient`每次自行mint，Host不按它去重、没有status/query seam；重发会重复prompt。raw history扫描只能事后证明存在，不提供与在途调用的linearization。
- `dsh-client-connection`对`/api`prefix和`/api/events.mux`、`/api/events.host`两条WebSocket upgrade复用Host/Origin/Fetch-Metadata trust predicate，并有一方tests；该predicate明确不是authentication。
- 初始HTML由`frontend-static`fallback直接服务，不经过上述fence；predicate也未从正式package root导出。WebServer公开route/upgrade/fallback原语，但没有全server middleware。因此rc.2没有统一覆盖HTML+API+两WS的公共admission seam。
- **认知状态**：verified_inference（固定tagApiProxy/Runtime/Connection/WebServer/Frontend控制流与一方tests；未启动真实Web Host）。

## 2026-08-31 · cold `session.models`与history presenter的公开可观测边界

- `session.models({sessionId})`先走共享`agentFor()`；ordinary cold identity经Persistence `list/inspect`后构造Host setup并调用`ctx.agents.resume()`，不是纯读header/catalog。Host setup从header+events fold recorded effective preset，并在unpublished Agent scope调用`AgentPresets.mount()`；成功后才读Session selection与model catalog。
- missing recorded id在`composeAgent→presets.resolve()`阶段就失败，尚未调用`ctx.agents.resume()`；可发现broken row或真实Loader/mount失败则进入resume的unpublished setup后rollback。两者都fail closed且不发布Agent，但只有后一类可称“真实resume setup mount失败”。AgentLoop一方tests固定setup/commit rejection后`ctx.agents`与`ctx.sessions`都无该id。
- `session.history`不走`agentFor()`。cold路径尝试`standingKeyFor(effectivePreset)`，unknown/broken/unusable时catch并用`scope=undefined`查询global ToolRuntime layer。公开response没有fallback reason或scope字段；`HistoryEntry`只有`event`与可选`view`。
- 要从公开response正向证明global presenter，必须让历史含tool event且global layer有可辨识`presentCall/presentResult`，再断言`events[].view`。仅“history成功+无Agent”只能证明fail-soft且未resume，不能区分global无presenter、roster缺失、JSON/pairing/presenter软失败；view缺失由官方Client渲染generic card，但Host不发显式generic marker。
- **认知状态**：verified_inference（固定rc.2ApiProxy/API Remote resolver/AgentLoop/Preset控制流与cold/agent-lookup/resume/mount/view一方tests；未运行新的全链fixture）。

## 去哪深入（文件路由）

| 问题 | 去哪 |
|---|---|
| 组内 8 个包与 ctx key | `packages/host/README.md` |
| 每个 RPC 域（session/workspace/settings/credentials/llm/agentPreset/command/skill）的确切语义与错误码 | `packages/host/apiproxy/README.md`（**很长，按域查**） |
| 四象限线消息 union、zod 两级解析、`RpcMethodMap` | 同上 `## Contract layer (/api)`；实现在 `src/api-proxy.ts` |
| 路由匹配顺序、fallback 单所有者、dispose 语义 | `packages/host/webserver/README.md`、`docs/subsystems/web-server.md` |
| 结构化 index 注入（`IndexInjection`、`webserver/index-inject`、`renderIndex`） | `packages/host/webserver/src/injections.ts`、`packages/host/webserver/README.md` |
| picker seam 的判别联合、`DirectoryPickerError`、`DirectoryListing.crumbs` | `packages/host/directory-picker/README.md`、`docs/subsystems/workspace.md` |
| `-auto` 判定 `native` 需要的全部信号 | `packages/host/directory-picker-auto/README.md` |
| GUI 分层与 RPC 协议的原始 RFC | `.agents/notes/implemented/architecture/2026-07-19-gui-layering-and-rpc-protocol.md` |
| picker seam 决策 | `.agents/notes/implemented/architecture/2026-07-28-directory-picker-capability-seam.md` |
| 浏览器半 | `packages/client/README.md`；组合样例 `packages/bundle/web-app/cordis.patch.yml` |
