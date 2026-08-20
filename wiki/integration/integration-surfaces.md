---
title: DSH 集成表面全景（外部项目如何接入 DSH）
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/sdk/README.md
  - packages/sdk/protocol/README.md
  - packages/sdk/client/README.md
  - packages/sdk/server/README.md
  - python/README.md
  - python/sdk/README.md
  - python/sdk-runtime/README.md
  - packages/acp/acp/README.md
  - packages/host/README.md
  - packages/host/apiproxy/README.md
  - packages/client/connection/README.md
  - packages/api/README.md
  - docs/api-gateway.md
  - apps/cli/README.md
  - apps/cli/reference/README.md
  - examples/jsonrpc-agent/README.md
  - docs/user/guide/python-sdk.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

# DSH 集成表面全景

DSH 自称 developer preview，README 明写 **"THERE WILL BE COMPATIBILITY-BREAKING CHANGES"**；全仓版本基线 `0.1.0-rc.8`（root `package.json`）。下面五种方式是外部项目让 DSH 接管 agent 层的全部通路。

## DSH 有哪几种集成方式

| 方式 | 传输 / 形态 | 官方支持程度 | 适合什么宿主 | 锚点 |
|---|---|---|---|---|
| TypeScript SDK | 子进程 + stdio newline-delimited JSON-RPC | Product — stable API，npm 已 public 发布 | Node/TS 宿主，且**自己知道 runtime 可执行文件在哪** | `packages/sdk/client/README.md` |
| Python SDK | 同上，但自带 bundled runtime | Product，PyPI 发布 `deepseek-harness-sdk` | Python 宿主；也是唯一"零 Node 依赖"通路 | `python/sdk/README.md` |
| ACP server | 子进程 + stdio ACP JSON-RPC | Product，但**automation-only**（明确不是 UI 层） | 已实现 ACP client 的宿主（编辑器/父 agent） | `packages/acp/acp/README.md` |
| HTTP API gateway | 本机 HTTP `POST /api/*` + 两条 WebSocket downlink | Product，但为**自家 Web 前端**而生，无鉴权、无协议版本协商 | 只在 loopback 且能接受"随 dsh 一起发版"的宿主 | `packages/host/apiproxy/README.md` |
| CLI / Web app 直接用 | 进程调用 / 浏览器 | Product | 不需要嵌入、只要"跑一次任务"或给人用 | `apps/cli/README.md` |

跨方式的共同点：**DSH runtime 永远是一个独立进程**，宿主通过协议驱动它；没有"把 DSH 当库 link 进宿主进程"的官方通路（除非宿主本身就是 Node/Cordis）。

## 方式一：TypeScript SDK（packages/sdk）

三个包：`dsh-sdk-protocol`（纯库，定义线协议与 `JsonRpcLineTransport`）、`dsh-sdk-client`（客户端 `DeepSeekHarness` / `HarnessClient`）、`dsh-sdk-jsonrpc-server`（跑在 runtime 里的 serving 插件）[T1: packages/sdk/README.md]。

- 与 Python SDK 的**关键差异**：launch spec 完全显式（`launch: { command, args }`），**不做 bundled-runtime 解析** —— "packaged-executable discovery stays Python-side"是写在 Known Limitations 里的[T1: packages/sdk/client/README.md]。所以 TS 宿主必须自己准备 runtime 可执行文件 + `cordis.yml`。
- `packages/sdk/` 组明确声明：*"Callers supply the runtime executable and its `cordis.yml`; this group does not create, configure, build, or launch developer projects."*
- 分层：`DeepSeekHarness.run()` 是"拥有一段活动区间"的高层 API；`HarnessClient.prompt()` 只返回入队回执，不等待。
- 错误类型是公开面：`JsonRpcResponseError` / `RequestTimeoutError` / `SdkProtocolError` / `TransportClosedError`（后者带 exit code 与 bounded stderr tail）。
- 关闭走 `shutdown` → stdin-EOF → SIGTERM → SIGKILL 阶梯（`shutdownTimeoutMs` 1000ms、`disposeEofGraceMs` 6000、`disposeGraceMs` 3000）；这条阶梯**故意不走 `dsh-subprocess` 服务**，是 seam 的明文例外。
- 仓内自用证据：`dsh-subagent-dsh-sdk` 就是用这个 client 把子 agent 跑成完整 peer harness（`packages/subagent/subagent-dsh-sdk/README.md`）。

## 方式二：Python SDK（deepseek-harness-sdk）

两个 dist：`deepseek-harness-sdk`（import 名 `deepseek_harness`）+ `deepseek-harness-runtime-bin`（import 名 `deepseek_harness_runtime`）[T1: python/README.md]。

- **零 Node 依赖**：runtime-bin wheel 里是单文件 Node executable `dsh-jsonrpc-agent-pkg-<platform>-<arch>`，外加 target-native ripgrep `-rg` sidecar；macOS 还多一个 `-spawn-helper`。**任何 sidecar 缺失都是硬启动错误**，哪怕你选的 cordis composition 根本不用搜索/PTY 工具[T1: python/sdk-runtime/README.md]。
- 平台矩阵是**固定三行**（`python/sdk-runtime/platforms.json`）：`manylinux_2_28_x86_64` / `manylinux_2_28_aarch64` / `macosx_14_0_arm64`。**没有 Windows wheel，没有 macOS x64 wheel**；build hook 主动拒绝 `py3-none-any`。tutorial 的 prereq 写 Python ≥3.10 + Linux x64/arm64 或 macOS 14+ arm64。
- **runtime 本身永远要求显式 config**（`$DSH_CORDIS_CONFIG` 或 argv positional），没有就大声退出。"零配置"是 client 侧的显式参数注入，不是 runtime 里的隐藏兜底 —— 注入发生在 `HarnessClient.start()`，条件：解析到 bundled runtime **且** `cordis` 与非空 `DSH_CORDIS_CONFIG` 都没设；一旦给了 `runtime_bin` / `bridge_bin` / `launch_args_override`，注入**整体失效**[T1: python/sdk/src/deepseek_harness/client.py#_inject_bundled_default_config]。
- 公开 API：`DeepSeekHarness` / `DeepSeekHarnessConfig` / `Session` / `RunResult` / `HarnessClient` / `HarnessConfig` / `SdkProtocolError`（`python/sdk/src/deepseek_harness/__init__.py`）。`RunResult` 带 `finish_reason`（`completed` / `max-tokens` / `error`…），这是 TS SDK 的 `RunResult` **没有**的字段。
- 环境变量注入点（`python/sdk/src/deepseek_harness/api.py#DeepSeekHarnessConfig`）：`DSH_SESSION_ROOT` / `DSH_CORDIS_CONFIG` / `DSH_CWD` / `DEEPSEEK_BASE_URL` / `DEEPSEEK_API_KEY`。
- runtime-bin 的插件集合 = `python/sdk-runtime/package.json` 的依赖闭包，**加一个插件 = 加一行依赖再重新构建 exe**。已含 `@deepseek-ai/dsh-mcp-client`，所以外部 cordis.yml 可以接 stdio / Streamable HTTP MCP server（只支持 Tools，**Resources 与 Prompts 不支持**）。

## 方式三：ACP server（packages/acp，automation-only）

`@deepseek-ai/dsh-acp` 在 stdin/stdout 上开 `AgentSideConnection`，驱动 `ctx.agents`。它自我定义为 **transport adapter，不是 UI 集成也不是 capability seam**：不暴露 editor navigation、transcript replay、commands、modes、elicitation、reasoning、plans、titles、tool presentation[T1: packages/acp/acp/README.md]。

- 支持的方法只有 `initialize` / `authenticate`（no-op）/ `session/new` / `session/prompt` / `session/cancel` + 反向 `session/update`、`session/request_permission`。
- Known Limitations 是选型硬约束：**fresh sessions only**（load/list/resume/delete/fork 全不支持）；**一个 workspace**（`additionalDirectories` 与 `mcpServers` 非空即拒）；**只发 committed 文本/图片**（无 token 级流式、无 reasoning、无 tool activity）；**连接级生命周期**（无 per-session close）。
- 图片只在"挂了 durable attachment store **且** 配置的 exact provider/model 声明 image input"时才 advertise；只收 PNG/JPEG/WebP/GIF。
- 跑法：`pnpm run demo:acp`（`examples/acp-agent/`），`DSH_PERMISSION_MODE` 选 `workspace-write` / `danger-full-access`。

## 方式四：HTTP API gateway（packages/host + packages/api）

这是 dsh Web GUI 的宿主半边，不是给第三方设计的公共 API。两条栈并存：

- **legacy API Proxy**（`packages/host/apiproxy`）：`POST /api/<method>` + SSE/WebSocket 下行，四象限 discriminated union，Zod 双层校验。方法集是 `RpcMethodMap`，未知方法在 envelope parse 阶段就 fail loud。
- **Typert Remote gateway**（`packages/api` + `docs/api-gateway.md`）：`POST /api/<namespace>/<method>`，由 `@Remote` / `@RemoteScope` 装饰器在**构建期**生成 Host/Client 契约。运行依赖方向 `remotes → gateway → connection → webserver`；Gateway 只认两段式 endpoint，其余 fall back 给 API Proxy。**只支持 unary**，事件流/分页/投影必须走另外的协议。
- 传输由 `packages/host/webserver` 承载（`node:http`，`host` 只接受 `127.0.0.1` 或 `0.0.0.0`），路由匹配顺序固定：exact → longest prefix → fallback。
- 安全姿态见"陷阱"。非 TS 宿主理论上能直接打 HTTP，但要接受"客户端与 host 同版本发布、无版本协商字段"这个前提（`host.describe` 的版本协商字段被明确推迟到"存在独立发布的 client"之后）。

## 方式五：CLI / Web app 直接使用（apps/cli、apps/web）

`@deepseek-ai/dsh` 提供 bin `dsh`（`apps/cli/package.json`），engines `^22.19.0 || >=24.0.0`。四种入口模式[T1: apps/cli/README.md]：

- `dsh --profile <name>` —— profile 是**有序的 bundle patch 层栈**，目录在 `$DSH_HOME/profiles/<name>`；层序：bundles → profile `cordis.patch.yml` → home `$DSH_HOME/cordis.patch.yml` → `--patch` overlay。patch **整行替换 `config`，不做深合并**。
- `dsh --profile headless "job"` —— 一次性任务：建新会话、跑、把最后一条 assistant 文本打到 stdout，`completed` 退 0 否则退 1；不挂 ApiProxy/Host/HTTP/Web。**最省事的"任何语言都能集成"方式**，代价是每次冷启动 + 只拿到最终文本。
- `dsh web` = `--profile web`，默认 `http://127.0.0.1:3080`。
- `dsh plugin --profile <name> <pnpm args>` —— 转发 pnpm 到 profile 目录装插件；bundle 成员变化需重启 profile，普通 patch 编辑走热重载。
- `--dump-default-config` / `--dump-config` 可以在不 boot 的情况下打印合成后的树（带来源注释），是排查 composition 的第一手段。

## 非 TS/Python 宿主怎么办（协议层是唯一通路）

按"侵入度从低到高"：

1. **进程调用 `dsh --profile headless "task"`** —— 零协议实现，只能拿最终文本、不能流式、不能续会话。适合 CI、批处理、脚本粘合。
2. **自己实现 SDK 线协议**（推荐给需要流式/多轮的宿主）。协议表面很小，全部写在 `packages/sdk/protocol/README.md`：client→server 只有 `initialize` / `session/prompt` / `shutdown`；server→client 只有 `session.event` / `session.status` / `subagent.started` / `subagent.finished`。传输是 newline-delimited JSON-RPC 2.0，一行一帧，**畸形 JSON 行被静默忽略**。runtime 端只要在 cordis.yml 里保留 `@deepseek-ai/dsh-sdk-jsonrpc-server` 这一行。`serverInfo.name` 是 wire-stable 的 `deepseek-harness-sdk-runtime`。
3. **实现 ACP client** —— 如果宿主生态已有 ACP 实现（编辑器侧常见），复用现成库比第 2 条便宜，但要吞下 fresh-session-only 等限制。
4. **打 HTTP `/api`** —— 能力最全（session 列表、fork、settings、模型目录都在里面），但契约最不稳、且带 loopback 信任栅栏。

第 2 条是**成本/能力比最好的自建路径**：Python SDK 本身就是"不 import TS 协议包、只镜像这些 shape"的参考实现（`packages/sdk/protocol/README.md` 原话："the Python SDK ... mirrors these shapes but does not import them"），说明重新实现是被预期的用法。

## 选型的判断维度

- **宿主语言是 Python** → 走 Python SDK。理由：只有它自带 runtime 二进制，宿主机不需要 Node。
- **宿主语言是 Node/TS，且能自己管好 runtime 产物** → 走 TS SDK。理由：同一份协议、类型齐全；但要接受"你自己负责找可执行文件"。
- **宿主是其他语言，需要流式与多轮** → 自实现 SDK JSON-RPC（上面第 2 条）。理由：协议面只有 3 个请求 + 4 个通知，且官方已有一个非 TS 的参考实现。
- **宿主是其他语言，只要"跑完给结果"** → 直接 `dsh --profile headless`。理由：零协议成本；不要为了"更正式"去实现 wire 协议。
- **宿主已是 ACP client** → 走 ACP。理由：复用既有实现；但先确认不需要 resume/fork/多 workspace。
- **需要会话隔离与并发** → SDK 与 ACP 都是"一个 runtime 进程内多 session"（SDK 按 `sessionId` get-or-create agent；ACP 一个连接可拥有多 session，各自独立 workspace/cancel/disposer）。需要**强隔离**就一个任务一个进程 —— 仓内 `subagent-dsh-sdk` / `subagent-acp` 都是"每次 run 一个全新进程，不做进程池"。
- **需要沙箱** → 沙箱是 composition 的事，不是集成方式的事：由 `cordis.yml` 里挂的 fs/shell/sandbox 插件决定。SDK 的 bundled 默认配置与 `examples/jsonrpc-agent/minimal.cordis.yml` 用的是 `danger-full-access`，**必须只在一次性 checkout 或容器里跑**。
- **需要复用已有工具** → 挂 `@deepseek-ai/dsh-mcp-client`，把外部 MCP server 的 tools 注册到 `ctx.tools`（模型看到的名字是 `mcp__<serverName>__<rawName>`）。runtime-bin 已内置这个插件，但**不含任何 MCP server 程序与凭据**。
- **模型与凭据** → 三条互不相同的路子：SDK 走进程环境变量 `DEEPSEEK_API_KEY` / `DEEPSEEK_BASE_URL`（可指向任意 OpenAI 兼容代理）；CLI/Web 走 `$DSH_HOME/.credentials.yaml` + settings；自定义 provider 目录要挂 `llm-pi-ai`。SDK 的 `initialize` 只在 provider 未被占用且名为 `deepseek-official` 时自动挂 `dsh-llm-deepseek`，**其他未被占用的 provider 会让 initialize 直接失败**。
- **部署环境是 Windows** → 直接排除 Python SDK（无 wheel）与依赖 PTY 的 composition（persistent PTY 需要 POSIX 终端）。
- **版本基线** → 所有 npm 包与 Python 两个 dist 共用 root `package.json` 的版本；SDK 对 runtime 是 `==` 硬锁。把"升级 DSH"当成一次可能 breaking 的整体升级来规划，不要分开升。

## 陷阱

1. **PyPI 包名**：官方是 `deepseek-harness-sdk`（`python/sdk/pyproject.toml` 的 `name`），import 名才是 `deepseek_harness`。PyPI 上的 `deepseek-harness` 是**无关第三方包**，装错会得到完全不同的东西。
2. **版本严格锁死**：`deepseek-harness-sdk` 依赖 `deepseek-harness-runtime-bin==<exact>`（仓内占位 `0.0.0.dev0`，发布时由 root `package.json` 版本替换，且发布 tag 必须是 `python-v<repository-version>`）。不能单独升 runtime，也不能 pin 住 SDK 却让 runtime 漂移。
3. **stdout 就是协议**：SDK 与 ACP 两条 stdio 通路都要求 stdout 只有协议帧。`dsh-sdk-jsonrpc-server` 的 Known Limitations 明写它**不会检查也不会否决**兄弟插件里的 stdout logger —— 你自己在 cordis.yml 里加个 console logger 就能把通道污染掉，诊断一律走 stderr。
4. **没有 mid-turn cancel**：SDK 线协议**没有 prompt-cancel 也没有 per-session close** 方法，放弃一个 turn 的唯一手段是关掉 runtime 进程。需要取消语义就得选 ACP（有 `session/cancel`）或 HTTP（有 `session.cancel`）。
5. **`messageId` 不是结果句柄**：`session/prompt` 返回的 `SessionPromptResult.messageId` 只标识入队的 `UserMessage`，**不标识后续的 assistant 消息、turn 结束或 prompt 结果**。`run()` 的 `finalResponse` 是"这段活动区间内最后一条已提交的 root-session assistant 文本"，不是因果归属于该 prompt 的回答 —— steering 和注入上下文可能在 idle 之前贡献内容。
6. **`session.event` 是全量的**：server 会把 runtime 里**每一个 session** 的事件都推出来，不做过滤；按 session 树收敛是**客户端自己的事**（TS 与 Python 都在客户端做 `subscribeSessionTree`）。
7. **协议无版本协商**：握手只带 `serverInfo.version`（`0.0.1`，且客户端不校验）。HTTP 侧同样"no protocol version field"。跨版本混搭无任何保护。
8. **HTTP `/api` 无鉴权**：`packages/client/connection` 的 trust fence 是**可达性策略而非认证**，靠 `Host` 头做 loopback / `trustedHosts` 判定；webserver 明写"No TLS, auth, or origin policy"。`dsh web --host 0.0.0.0` 被**故意不支持**并以 usage error 退出。别把它当内网服务暴露。
9. **文档路径错位**：网站上的 `guide/quickstart` 在仓里**没有同名文件** —— 它是 `docs/user/guide/index.md` 的发布路由（`website/docs.ts` 里 `route: 'guide/quickstart.md'`）。`docs/user/index.md` 的 meta refresh 指的是发布后的路径，直接在仓库里按 quickstart 搜会一无所获。
10. **默认 composition 是 danger-full-access**：Python SDK tutorial 与 `minimal.cordis.yml` 都用它，bash 与绝对路径编辑器可以改 runtime 进程能看到的**任意路径**。集成时第一件事是决定沙箱策略，而不是先跑通。
