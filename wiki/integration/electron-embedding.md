---
title: integration/electron-embedding — Electron 与嵌入运行时集成约束
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/client/connection/src/client/index.ts
  - packages/client/runtime/README.md
  - packages/client/runtime/src/client/contract/workspaces.ts
  - packages/client/runtime/src/client/workspaces/service.ts
  - packages/boot/app-boot/src/index.ts
  - vendor/loader/src/index.ts
  - vendor/hmr/README.md
  - patches/node-pty@1.2.0-beta.15.patch
  - apps/cli/src/dump-config.ts
  - packages/host/apiproxy/README.md
  - packages/host/apiproxy/src/api/events.ts
  - packages/workspace/workspace/src/paths.ts
  - packages/workspace/workspace/src/index.ts
  - packages/workspace/workspace/src/entity.ts
  - packages/host/apiproxy/src/api-proxy.ts
  - packages/fs/tool-fs/src/session-cwd.ts
  - packages/shell/tool-bash/src/index.ts
  - packages/fs/README.md
  - packages/subprocess/README.md
  - packages/e2b/README.md
  - packages/client/ui-primitives/package.json
  - packages/client/ui-primitives/src/index.ts
  - packages/client/ui-primitives/src/markdown/MarkdownText.tsx
  - packages/client/ui-primitives/src/markdown/render.tsx
  - packages/client/ui-primitives/tests/markdown.client.spec.tsx
  - packages/client/ui-conversation/src/client/chat/AssistantMarkdown.tsx
  - packages/client/ui-conversation/src/client/chat/AssistantNodeView.tsx
  - packages/client/ui-conversation/src/client/contract/chat-nodes.ts
  - packages/client/runtime/src/client/sessions/conversation.ts
  - .agents/notes/implemented/feature/2026-07-23-web-assistant-markdown.md
  - packages/llm/llm-pi-ai/src/config.ts
  - packages/llm/llm-pi-ai/src/catalog.ts
  - packages/llm/llm-pi-ai/src/provider.ts
  - packages/llm/llm-pi-ai/src/auth.ts
  - packages/llm/llm-pi-ai/src/index.ts
  - packages/llm/llm-pi-ai/src/adapter.ts
  - packages/llm/llm-pi-ai/tests/catalog.spec.ts
  - packages/llm/llm-pi-ai/tests/adapter.spec.ts
  - packages/llm/llm-pi-ai/tests/dynamic-config.spec.ts
  - packages/llm/llm/src/index.ts
  - packages/llm/llm/tests/service.spec.ts
  - packages/settings/settings/src/index.ts
  - packages/credentials/credentials/src/index.ts
  - packages/host/apiproxy/src/api/sessions.ts
  - packages/host/apiproxy/tests/api-proxy-models.spec.ts
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent/src/model-selection.ts
  - packages/core/agent/tests/model-selection.spec.ts
  - packages/client/ui-model-selection/src/client/ModelSelect.tsx
  - packages/client/ui-model-selection/tests/model-select.client.spec.tsx
  - packages/llm/llm-deepseek/src/index.ts
  - packages/bundle/base/cordis.patch.yml
  - packages/interaction/user-approval/src/index.ts#ApprovalService.request
  - packages/core/tools/src/index.ts#ToolRuntime.serviceAsk
  - packages/host/apiproxy/tests/api-proxy-approval.spec.ts
  - packages/client/ui-conversation/src/client/skeleton/ApprovalPanel.tsx#ApprovalPanel
  - packages/client/ui-conversation/src/client/contract/slots.ts#PendingApproval
anchors_note: 逐条见正文各行内锚点
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-27
asked_by: agent
---

# Electron 与嵌入运行时集成约束

> 八维坐标：宿主语言 TypeScript · 运行形态 Electron（main + UtilityProcess）或任意「宿主拥有物理传输/安装树」的嵌入 runtime · 并发与会话隔离 N/A · 沙箱与文件系统 app-owned 安装树（宿主可解析路径）· 工具复用官方 bundle · 模型与凭据经宿主 carrier · 部署环境打包产物（asar 等）· 版本基线 0.1.1-rc.2。
>
> 适用问题：把 DSH host 或 client 装进一个**不由 DSH 自己启动/自己管文件布局**的进程里（Electron、打包产物、sidecar 进程），哪些官方 seam 可用、哪些能力结构性缺席。

## 1. Client 侧 carrier 的正门：`__DSH_TRANSPORT__`

- **条件**：宿主（如 Electron renderer）要让官方 client runtime 跑在自定义物理传输上。
- **结论**：rc.2 的官方接缝是 `@deepseek-ai/dsh-client-connection/client` subpath 的 `ClientTransportHooks`——connection `apply()` 从 `globalThis.__DSH_TRANSPORT__` 读取 `createApiClient` / `fetch`，构造并 provide `ConnectionHandle`；`handle.start()` 才创建包内 `ConnectionController`；可选 `loadBundle` 由 client-web boot 消费。宿主只写物理 transport adapter，不声明/拼装 handle 或 controller。
- **版本演进**：rc.8 时该接缝形态是 client-web boot 的显式 `BootSeams` 参数；rc.2 统一并入 `__DSH_TRANSPORT__` 全局钩子。rc.8 公共面**没有** carrier factory（`apply()` 固定 Web/fixture 路径），跨 rc 升级时要按此重查。
- **锚点**：`packages/client/connection/src/client/index.ts`（约 L56–L169）。
- **认知状态**：verified_inference（源码级；未实测）。

## 2. `/client` subpath 是硬性要求

- **条件**：插件 bundle 或宿主代码 import client runtime / connection。
- **结论**：必须用 `@deepseek-ai/dsh-client-runtime/client` 这类 `/client` subpath。裸包名 import 会 inline 第二个模块实例，其私有 scope-tag Symbol 永不匹配——表现为状态静默错乱而非报错（README Known Limitations 明文）。
- **锚点**：`packages/client/runtime/README.md`。

## 3. `loader.internal` 缺席时连锁失效的能力

`loader.internal`（进程内模块加载器）不是所有宿主都可用。缺席时（源码可证的后果，不依赖具体宿主）：

| 能力 | 缺席时行为 | 锚点 |
| --- | --- | --- |
| `bareModuleBaseUrl` | **参数不参与解析**：bare 包名静默退回普通 Node resolution；插件必须物理落在宿主普通 import 能解析到的位置 | `packages/boot/app-boot/src/index.ts`（约 L492–L504） |
| 官方 HMR（`vendor/hmr`） | 插件直接 throw（"The package throws if the loader service has no internal module loader available"）——嵌入宿主没有官方热重载 | `vendor/hmr/README.md` |
| Loader 配置 schema | 注意 `bareModuleBaseUrl` 是 `boot()` / `mountRootInclude()` 的**函数参数**，不是 loader 配置键（`vendor/loader/src/index.ts` 的 `Loader.Config` 只有 `baseUrl`） | 两处各一 |

- **认知状态**：verified_inference。具体某宿主（如 Electron UtilityProcess）中 internal 是否可用属宿主实测问题，本页不下结论。

## 4. asar 与 native 伴生文件

- **条件**：宿主把 DSH 装进 asar 类只读归档。
- **结论**：**loader 对 asar 内资源无任何官方支持或文档**（全仓 `rg -i asar` 仅命中 node-pty patch）。native 伴生文件布局有一个官方先例：`patches/node-pty@1.2.0-beta.15.patch`（rc.7 起）定义 spawn-helper 三级解析——`DSH_NODE_PTY_SPAWN_HELPER` env → `execPath` 同级 → `asar.unpacked` 兜底，patch 注释明说是给 "external embedded-runtime consumer" 的通道；Python `sdk-runtime` 也以 `-spawn-helper` / `-rg` sidecar 布局分发单文件 exe。嵌入宿主打包 PTY/native helper 可复用该官方 patch 与 env 通道。
- **锚点**：`patches/node-pty@1.2.0-beta.15.patch`。

## 5. 启动断言与配置对照的官方工具

- **`assertEntriesActivated`**（app-boot）：entry/fiber 级激活断言——每个 enabled entry 的 fiber 必须 ACTIVE；FAILED 时取回原始 rejection；**PENDING 时把未满足的 inject 服务名逐个点名写进错误信息**（宿主诊断绑定缺口的最低成本信号）。**没有**「服务/工具/projection 激活集枚举」的官方 API——需要激活集清单时须自行经 ctx.registry 构建；`cordis_inspect what:"events"` 读的是构建期生成目录（`pnpm run gen-cordis-api` 产物），不是运行时 registry。
- **`dsh --profile <name> --dump-config`**：官方 boot-free 配置组合对照工具——用与 boot 完全相同的 applyEntryPatches 单调用组合、`!!js` 原样不求值、逐层注释来源。宿主做「构建期配置快照 vs 运行时组合」双向对照时，这是官方对齐物。
- **`boot()` 的 prepare hook**：在任何配置树 entry 挂载前运行（app-boot/src/index.ts 约 L747/L772），宿主可在此提供 launcher-owned context slots——carrier 集成的官方接缝位置。
- **锚点**：`packages/boot/app-boot/src/index.ts`、`apps/cli/src/dump-config.ts`。

## 6. Workspace/Session 创建面的接口级约束

- **跨域 face `SessionsPort.create` 只接受 `{ workspaceId}`**（`packages/client/runtime/src/client/contract/workspaces.ts`）——比具体类 `SessionRuntime.create`（还收 cwd/sessionId）窄一个量级；官方用接口形状约束「session 创建走 workspace 路径」。
- **wire 面 `workspace.create` 只收 `{ path }`**，title 默认取 path basename；`WorkspaceRegistry.create(anchor, title)` 的 title 参数已无生产调用方、处于上游废弃流程（index.ts TODO 注释），**不要把 title 当稳定 API**。
- **`WorkspaceRegistry.create` 按 canonical path 幂等**：同 realpath 重复调用返回既有 entity 不改 title——重启后重复注册不会产生第二个 Workspace。
- **`SessionCreateError` 携带 `requestedSessionId`**（workspaces/service.ts 约 L107–L120）：caller-preallocated id 失败时可凭它对齐后续 stream/list 对账。
- **`connectWorkspace` 的 coalescing 竞态根因**：create 的 summary 落地时先无 cwd，host frame 到达前第二个并发调用会漏掉 reuse 扫描而多铸一个隐藏 blank session——自建并发创建路径必须复现这层合并逻辑。
- **认知状态**：verified_inference。

## 7. 跨 host restart 的 pending 语义（rc.2 README 新增 Known Limitation）

- pending question **不跨 host restart**：registry 持有 tool call 的 resolve/reject，是进程内存；events.mux 重放只覆盖浏览器 reload/重连。
- `events.mux` 的 `since` resume hook 契约上 "unimplemented in v1 (ignored if passed)"——重连恢复 = 重开 stream + 重新拉 history/list，不能假设增量续传。
- **锚点**：`packages/host/apiproxy/README.md`、`packages/host/apiproxy/src/api/events.ts`（约 L57–L59）。

## 8. 本地 Workspace identity 与远程 execution world 必须分层

- **条件**：DSH Host 运行在本地 Electron / sidecar 进程，但宿主管理的项目工作目录位于 SSH 远端。
- **Workspace 边界**：`WorkspaceRegistry.create(path)` 直接使用 DSH Host 所在机器的 Node `fs.realpath` 与 `stat`，并要求路径是真实存在的目录；Workspace 记录只有 canonical `path`，没有 SSH host、connection id、URI 或 remote cwd 字段。`session.create({ workspaceId })` 会把 `workspace.path` 写成 `SessionHeader.cwd`，`attachSession()` 再以同一套本地 realpath 严格复验。
- **执行边界**：`read` / `write` / `edit` 与 `bash` 默认都从 `SessionHeader.cwd` 取工作目录。`ctx.fs` / `ctx.subprocess` 是可替换的 execution-world seam，但 rc.2 仓库内的第一方实现只列 local / sandbox 与 E2B POC，没有 SSH provider。成对的远程 fs / subprocess provider 必须共享同一 execution world；`glob` / `grep` 又绕过 `ctx.fs` 通过 `ctx.subprocess` 跑 `rg`，不能只换 fs 就假定搜索自然生效。
- **结论**：本地 DSH Host 的官方 Workspace 应绑定宿主管理、Host 可见且实际存在的**本地 anchor**；`{ connectionId, remoteCwd }` 留在宿主自己的映射或 SSH 工具/provider 层。仅当 DSH Host 本身运行在远端，或远端目录已挂载进 Host 本地文件系统时，才可直接传对该 Host 可见的目录路径；后者传的仍是本地挂载路径，不是远程 locator。
- **锚点**：`packages/workspace/workspace/src/paths.ts#realpathNormalize`、`packages/workspace/workspace/src/index.ts#WorkspaceRegistry.create`、`packages/workspace/workspace/src/entity.ts#attachSession`、`packages/host/apiproxy/src/api-proxy.ts#session.create`、`packages/fs/tool-fs/src/session-cwd.ts#sessionResolveOptions`、`packages/shell/tool-bash/src/index.ts#resolveWorkdir`、`packages/fs/README.md`、`packages/subprocess/README.md`、`packages/e2b/README.md`。
- **认知状态**：verified_inference（Workspace L1 / Electron 集成 L2 封顶；本次源码级复验，未做运行实测）。

## 9. Assistant Markdown 属于 presentation leaf；逐消息状态才驱动 streaming

- **条件**：嵌入宿主只消费官方 `ConversationSnapshot.chat`，将其中的 assistant text 投影为自己的简化消息视图，并希望渲染不可信 Markdown。
- **公共 API 与 ownership**：`@deepseek-ai/dsh-client-ui-primitives` 的公开根入口导出 `MarkdownText`；官方 `ui-conversation` 自己也从该根入口导入，并在 presentation leaf 对 assistant `text` block 使用它。宿主在 JSX 叶子复用该 primitive，不会接管 DSH Definition 对 raw SessionEvent 的折叠、关联或生命周期判定；Markdown AST 也不应回灌 session projection。官方视图逐个渲染 text block，宿主若先拼接多个 block，仍是合法的宿主简化视图，但不等同于官方混合 block 展示语义。
- **流式状态**：官方精确信号是该 `assistant-step` 的 `data.status === 'running'`；`ConversationSnapshot.running` 是 session-wide 状态，不能归属到某一条消息。若简化投影已经丢失逐 node 的一一对应，省略 `streaming` / 传 `false` 是安全的保守展示：默认值本就是 `false`，会走完整 GFM + math/highlight 解析，协议白名单不变；代价是实际仍增长时失去增量缓存并可能出现临时排版变化。不能长期误传 `true`，否则 settled 后的 KaTeX、高亮和 file mention finalize 不会发生。
- **安全与 owner 边界**：raw HTML 只按字面文本渲染，不进入 HTML DOM；链接只允许绝对 HTTP(S)/`mailto:`，相对或危险协议解除为普通文本；Markdown 图片只允许绝对 HTTP(S)，会直接形成远端请求，本地/相对/危险协议只留 alt。`fileMentions` 必须由 owner 用真实文件词表解析，renderer 不猜路径，且只在 settled render 生效；缺少该 owner 或无法证明消息已 settled 时应省略。Electron 的 CSP、导航和新窗口策略仍由宿主负责。
- **选择结论**：固定在 rc.2 时优先复用 `MarkdownText`，可与官方 conversation 共享 GFM、数学、增量渲染、DOM 与安全策略；另引 `react-markdown` 会形成第二套 renderer。源码没有规定第三方宿主“必须”这样选，因此这是强集成推论，不是规范性 MUST。
- **锚点**：`.agents/notes/implemented/feature/2026-07-23-web-assistant-markdown.md`、`packages/client/ui-primitives/{package.json,src/index.ts,src/markdown/{MarkdownText,render}.tsx,tests/markdown.client.spec.tsx}`、`packages/client/ui-conversation/src/client/{chat/{AssistantMarkdown,AssistantNodeView}.tsx,contract/chat-nodes.ts}`、`packages/client/runtime/src/client/sessions/conversation.ts`。
- **认知状态**：verified_inference（Electron / client presentation 集成 L2 封顶；源码与测试断言级复验，未运行测试）。

## 10. 动态多模型配置：route 是身份，Settings 与 Credentials 分权

> 八维坐标：宿主语言 TypeScript · 运行形态嵌入式 DSH Host + Renderer 配置面 · 并发与会话隔离 per Session selection · 文件系统无关 · 工具复用官方 Agent loop · 模型走 `llm-pi-ai` 且 secret 走 `ctx.credentials` · 部署为可动态更新的可信 Host · 版本 `0.1.1-rc.2`。

- **所有权映射**：一份拥有独立 key / endpoint / wire protocol / 生命周期的产品配置对应一条 provider route；同 route 的 `models[]` 只表达共享全部 route 事实的 model ids。稳定内部 route 与可编辑 `displayName` 分离，因为 route 会被 Session 和 default 选择引用。
- **组合姿势**：Cordis 只裸挂 `@deepseek-ai/dsh-llm-pi-ai`，所有可删 route 放 `llm-pi-ai.providers` user settings。官方 runtime protocol 只有 `openai-completions` / `openai-responses` / `anthropic-messages`，一条 route 只取一项。
- **热更新**：pi-ai 自身监视 settings 并原子 `replace()` adapter routes，不要求 Host restart。但自定义 `SettingsProvider.load()` 只在 init 读一次；外部介质变更必须经 `ctx.settings.update/replace/mutate` 或子类 `publish(fullDoc)` 进入生命周期。删除 user route 用 `mutate(unset)`，composition base 不能被 user layer 删除。
- **secret 边界**：静态 key 用每 route 唯一 `apiKeyEnv` ref，`llm-pi-ai` 每次 stream call 重新 `resolve(ref)`；pi-ai 原生登录/OAuth 才走 `llm-pi-ai/<route>` record。Settings 删 route 不会自动删 secret，跨 seam transition 属宿主 owner。`headers` 是普通 settings，不得放 API key。
- **Session 选择**：Renderer 只传 `{provider: route, model: id, reasoningEffort?}` 给官方 `session.selectModel`。它是 live Session 的 next-step 选择，点击本身不记日志；后续模型请求消费后才把 `request/header` 持久化。route 删除/改名不会自动迁移历史 Session，旧选择会变 `routable:false` 并在 prompt 前 `model-unavailable`。
- **兼容迁移**：`llm-deepseek` 的固定 route `deepseek-official` 可与其他 pi-ai routes 并存，但全局 route 不能重复。旧 Session/default 尚引用它时保留 native adapter 是最小安全迁移；统一 adapter 是宿主产品选择，非 DSH 强制。
- **网络与错误**：`baseURL` 没有 SSRF/scheme/host 防线，由可信 Host 设置 owner 校验；provider error body 无 secret/PII redaction 保证且可进 durable turn end，普通 UI/日志只应显示受控摘要。图片必须先进官方 durable attachment，pi-ai 不抓任意远程 URL。
- **锚点**：`packages/llm/llm-pi-ai/src/{config,provider,auth,index,adapter}.ts`、`tests/dynamic-config.spec.ts`、`packages/settings/settings/src/index.ts`、`packages/credentials/credentials/src/index.ts`、`packages/host/apiproxy/src/{api/sessions.ts,api-proxy.ts}`、`packages/host/apiproxy/tests/api-proxy-models.spec.ts`、`packages/core/agent-loop/src/agent.ts`、`packages/llm/llm-deepseek/src/index.ts`、`packages/bundle/base/cordis.patch.yml`。
- **认知状态**：verified_inference（tag 源码、正式 npm JS/d.ts 与官方测试源码交叉核对；未重新执行测试）。

## 11. Approval presentation 的 visibility 不是 request lifecycle

> 八维坐标：宿主语言 TypeScript · 运行形态 Electron Main-owned 信任回答面 + 嵌入式 DSH Host · 并发与会话隔离按 Agent/Session owner · 文件系统无关 · 工具复用官方 ToolRuntime · 模型与凭据无关 · 部署为 macOS desktop · 版本 `0.1.1-rc.2`。

- **条件**：宿主保持 sender / nonce / current-pending 回答权威，且 Stop、owner teardown、channel teardown 等已有路径仍会 abort 或 fail closed；变化只是窗口因 app switch / OS occlusion 暂时不可见。
- **DSH owner 边界**：`ApprovalRequest.signal` 是唯一内建 withdrawal 输入；ToolRuntime 把 caller-owned `exec.signal` 传入 ApprovalService。审批 Host registry 明确跨 client/mux disconnect 存活，新 mux 以同一 `rpcId` 重放；官方测试覆盖了断线后重连仍可回答。
- **presentation 边界**：官方 `ApprovalPanel` 只发 `allowed-once` / `rejected`，没有 `blur` / `hide` / `visibilitychange` / unmount cancel；面板移除由 `approval/resolved` 驱动。上游把明确 prompt dismissal 映射为 `cancelled`，但未把 macOS occlusion 或普通失焦定义为 dismissal。
- **结论**：owner/authority 不变时，让 pending approval 与回答 View 跨普通 app switch / occlusion 保持，符合 rc.2 ownership；将 blur/hide 自动映射成取消只能是宿主自定义的更严 UX policy，不是 DSH 规范。
- **未知与限制**：DSH 没有 Electron `BrowserWindow` / `WebContentsView` 或 macOS occlusion 专项规范/测试；宿主内部 `closed` 怎样映射 DSH 的 `cancelled/rejected/unavailable` 必须由其 answerer owner 明确。官方 `tool-call-timeout-policy` 位于 approval 之后，不覆盖 pending approval 等待。
- **锚点**：`packages/interaction/user-approval/src/index.ts#ApprovalRequest/#ApprovalService.decide`、`packages/core/tools/src/index.ts#ToolRuntime.serviceAsk`、`packages/host/apiproxy/src/{api-proxy.ts,api/events.ts,api/approvals.ts}`、`packages/host/apiproxy/tests/api-proxy-approval.spec.ts`、`packages/client/ui-conversation/src/client/{skeleton/ApprovalPanel.tsx,contract/slots.ts}`、`packages/client/ui-conversation/README.md`。
- **认知状态**：verified_inference（Electron 集成 L2 封顶；官方源码与测试断言交叉核对，未运行测试）。

## 12. Reasoning effort 必须由精确模型能力正向证明

> 八维坐标：宿主语言 TypeScript · 运行形态 Electron Renderer + 嵌入式 DSH Host · 并发与会话隔离 per Session selection · 文件系统与工具无关 · 模型走 `llm-pi-ai` 手工 route · 凭据无关 · 部署为动态模型目录 · 版本 `0.1.1-rc.2`。

- **精确根因链**：真正的手工 pi-ai model（无同 id 的 installed-catalog base）省略 `reasoningEfforts` 后会物化为 `reasoning:false`，adapter 因而不向 `session.models` 的精确模型条目公开 `reasoning`。Renderer 若仍显式提交 `reasoningEffort:'off'`，LLM core 会把它视为未公布的 adapter-owned id，抛 `UNSUPPORTED_REASONING_EFFORT`；Host 将异常映射为 `model-unavailable`。该链与 `anthropic-messages` 协议无关。仅凭“配置未写字段”或宽泛错误码不足以唯一归因：同 id catalog model 可继承能力，运行错误消息还应命中 `does not support reasoning effort "off"`。
- **最小客户端契约**：纯 provider/model 切换只调 `session.selectModel({sessionId, provider, model})`；只有 `SessionModels.groups` 中 exact provider/model 的 `reasoning.efforts[].id` 明确包含用户选择时，才显式附带 effort。`session.prompt` 本身没有 effort 字段，官方 selection 已是 Session 的 live next-step 状态，不需要每次 prompt 前无条件重选。省略会清掉旧模型继承的 effort并恢复目标模型的 adapter/provider default；若 adapter 公布了 `defaultEffort`，Host 可将它物化进返回的 `selected`，客户端以该返回值为准。
- **目录边界**：`groups` 是建议目录而非路由白名单。精确行存在但 `reasoning` 缺席，才可说“未提供可选推理等级”；精确行缺席或 provider catalog 加载失败时只能说“能力未知/未公布”，不能声称模型不支持或不可路由。
- **最小 UI**：effort 选项只来自 exact-model metadata；未公开 `reasoning` 时不显示 `Off`，可隐藏 effort 控件或显示只读“使用模型/提供方默认”。用户显式选择若与新模型不兼容，不能静默映射成另一个 effort，也不能把“默认”写成“已关闭推理”；应明确显示“该强度未应用，本次使用默认”，或在该偏好是硬约束时阻止发送并要求用户确认。被 Host 拒绝时保留此前成功 selection 并展示受控错误摘要。
- **一方对照**：官方 composer 点击模型时只提交 `{provider, model}`；adapter 未公开 `reasoning` 时不渲染 effort 行；selection 失败通过 Toast 告知并保留上一次成功状态。官方没有规定第三方 UI 是否维护跨模型 effort 偏好，因此“提示后使用默认”与“阻止发送”是产品选择，不是 DSH MUST。
- **锚点**：`packages/llm/llm-pi-ai/src/{catalog,adapter}.ts`、`packages/llm/llm-pi-ai/tests/{catalog,adapter}.spec.ts`、`packages/llm/llm/src/index.ts`、`packages/llm/llm/tests/service.spec.ts`、`packages/host/apiproxy/src/{api/sessions.ts,api-proxy.ts}`、`packages/host/apiproxy/tests/api-proxy-models.spec.ts`、`packages/core/agent/src/model-selection.ts`、`packages/core/agent/tests/model-selection.spec.ts`、`packages/client/ui-model-selection/src/client/ModelSelect.tsx`、`packages/client/ui-model-selection/tests/model-select.client.spec.tsx`。
- **认知状态**：verified_inference（tag 源码、正式 npm 公共声明与官方测试源码交叉核对；上游镜像无 `node_modules`，未执行测试或真实模型调用）。

## 陷阱速查

1. 裸包名 import client runtime → 双实例 Symbol 错配，静默失败（§2）。
2. internal 缺席时传 `bareModuleBaseUrl` 以为锚定了安装树 → 参数静默无效（§3）。
3. 依赖官方 HMR 做开发体验 → internal 缺席宿主直接 throw（§3）。
4. 以为 loader 能解析 asar 内模块 → 无官方支持；走 app-owned 安装树 + node-pty 式 env/sidecar 通道（§4）。
5. 以为 app-boot 能枚举激活集 → 只有 entry 级断言；枚举自己建，配置对照用 `--dump-config`（§5）。
6. 并发 create/connect 不复现官方 coalescing → 产生隐藏 blank session（§6）。
7. 把 pending interaction 当持久状态跨 host 重启 → 进程内存，restart 即丢（§7）。
8. 把 SSH 远端 cwd 当成 Workspace remote locator → 本地 Host 只会做本机 realpath；绑本地 anchor，远程 cwd 留在宿主/provider 映射层（§8）。
9. 用 session-wide `running` 给所有 assistant 消息标流式，或自行猜路径/file mention → 前者混淆 projection 生命周期，后者越过 owner 词表；逐 node 状态未知时保持 settled renderer 并省略 mention（§9）。
10. 把每个 model 配置都塞进同 route，或用显示名作 route → key/protocol/endpoint owner 错位，且改名会使历史 Session 不可路由（§10）。
11. 只实现 `SettingsProvider.load()` 就期待 Main/外部存储热更新 → `load()` 只跑 init，必须走写 API 或 `publish(fullDoc)`（§10）。
12. 把 app switch / macOS occlusion 当成 DSH 审批撤回 → 可见性不是 owner signal，官方甚至跨 client disconnect 保留 pending（§11）。
13. 给所有模型硬塞 `reasoningEffort:'off'` → `off` 不是通用能力；按 exact-model metadata 正向证明，纯模型切换省略（§12）。
