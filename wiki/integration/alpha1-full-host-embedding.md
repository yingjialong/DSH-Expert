---
title: alpha.1 完整 Host 与嵌入控制面
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - README.md
  - SAFETY.md
  - package.json
  - apps/cli/package.json
  - apps/cli/src/profile-boot.ts
  - packages/boot/app-boot/package.json
  - packages/boot/app-boot/src/index.ts
  - packages/boot/app-boot/src/profile.ts
  - packages/bundle/base/cordis.patch.yml
  - packages/bundle/web-app/cordis.patch.yml
  - packages/api/gateway/package.json
  - packages/api/gateway/src/client/index.ts
  - packages/api/remotes/src/client/index.ts
  - packages/api/session-controller/src/index.ts
  - packages/api/workspace-controller/src/index.ts
  - packages/api/settings-controller/src/index.ts
  - packages/client/connection/package.json
  - packages/client/connection/src/client/index.ts
  - packages/sdk/protocol/README.md
  - packages/acp/acp/README.md
  - packages/core/session/src/known-event-types.ts
commit: cd5ef8148158c3a752a658978873241fdf8e2bbc
verified_at: 2026-08-30
asked_by: agent
---

# 0.1.2-alpha.1 完整 Host 与嵌入控制面

## 发布基线先决条件

- `dsh-v0.1.2-alpha.1` 是官方 GitHub prerelease，tag 与 HEAD 都是 `cd5ef8148158c3a752a658978873241fdf8e2bbc`；release 无附件，只有 source tarball/zipball。
- npm/PyPI **没有 alpha.1 闭包**：npm `@deepseek-ai/dsh` 的 `latest` / `next` 均仍为 `0.1.1-rc.2`，PyPI 两个官方 dist 均仍为 `0.1.1rc1`。对 tag 下 239 个非 private `@deepseek-ai/dsh-*` package name逐一查 npm，`0.1.2-alpha.1` 命中数为 0。
- 因此 alpha.1 的 package exports、`files` 与 companion dependency 只能称为 **Git tag manifest/source事实**，不能称为正式 npm tarball公共闭包。第三方现在可用的 alpha.1 方式只有 source checkout/build；`npm install @deepseek-ai/dsh@0.1.2-alpha.1` 不存在。

## 保留完整产品图的 Host 形态

| 形态 | 完整插件图 | 公开边界 | 限定 |
|---|---|---|---|
| source-built `dsh --profile web/headless/acp/sdk` 独立进程 | 可以；profile = base + mode bundle + user layers | CLI/profile/app-boot tag源码 | 只有 Web/Node service面能控制完整图；ACP/SDK wire仍是子集 |
| Node进程内 `@deepseek-ai/dsh-app-boot.boot()` + Cordis Loader | 可以，只要宿主装配与官方 bundle/preset同一闭包 | `app-boot` root、`@deepseek-ai/cordis-plugin-loader` / include、Cordis Context/Service | Node/Cordis形态；alpha.1 npm未发布；宿主自己拥有root fiber与shutdown |
| Web Host + `@Remote` Client | 官方完整一方控制面 | API Gateway/Remotes、三个controller、Client Connection/Store、各 `/client` / `/remote` intended exports | capability选择由build-time imports固定；必须同版本图；无语言无关的Remote版本协议 |
| TypeScript/Python SDK JSON-RPC | Runtime可挂base，但wire只暴露3请求+4通知 | `dsh-sdk-protocol` root | 无cancel/session-close、Workspace/history/projection/settings/approval等完整控制面 |
| ACP v1 | alpha.1已扩成持久Session、model/MCP/permission/cancel自动化面 | `dsh-acp` root +标准ACP | 明确排除history replay、fork/delete、Workspace实体、commands/plans/todos/terminals/settings/DSH presentation |

结论：不接受子集时，选择的是完整 Host composition，不是 SDK/ACP。跨进程的人类控制面是官方 Web Client组合；同进程Node宿主可直接组合Service。非React Client可复用React-free Remote/controller/store；非TypeScript Client没有等价的正式完整协议。

## 完整 Web控制面与 carrier

- Session Controller Remote：`list/search/create/selectModel/modelCatalog/rename/fork/prompt/attachment/updateQueue/cancel/page/follow/control`，另有Session-addressed skill和file-reference namespaces。history/follow保留raw tool call/result/failure与projection/control baseline。
- Workspace Controller：create/rename/delete/reorder、Session reorder/archive、complete follow baseline。
- Settings/Credentials Controller：redacted describe、update/replace/mutate、credential describe/set/unset与Host-native open intents。
- API Remotes还mount AgentPreset、Commands、Goal、LLM discovery、dynamic Cordis、plugin inventory、feedback、references、subagent等generated contributions。
- `approval/request` 与 `user-questions/request` 作为Agent-scoped waterfall经Remote event stream交给Client回答；普通事件失败隔离，waterfall可result或next。
- `@deepseek-ai/dsh-client-connection/client` 在tag manifest中是公开client subpath，导出 `ConnectionHandle`、`ClientTransportHooks`、`RpcFetch` 等。自有shell可在page global提供 `fetch`、可选 `openStream/loadBundle` 与 `ownsHost`，从而让官方React-free Client栈跑在IPC/postMessage等物理载体上。
- 这不是语言无关wire：业务endpoint/codec来自Typert生成的 `/remote` value imports，Client capability set由API Remotes build-time imports固定，没有runtime discovery或protocol-version negotiation。非TS实现只能重新实现内部HTTP/WebSocket framing，官方未把它承诺成第三方协议。
- 官方webserver文档明确说Electron使用`file://`和IPC bridge，不使用Node HTTP server；仓库没有`dsh-electron`实现、Electron embedding guide或第三方desktop兼容保证。

## Tag中的Harness owner矩阵

| 领域 | alpha.1状态 | 官方profile activation |
|---|---|---|
| AgentLoop / Agent / Scope | Product public spine + concrete `dsh-agent-loop` | base mounted；Agent model-facing composition在preset |
| Session/Event/Persistence/Projection/Query/Title | public seams + JSONL/SQLite/query/cache/title/telemetry providers | base mounted；checkpoint policy同图 |
| Workspace | public entity + Storage Domain + Remote/Client controller | Web bundle mounted；headless/SDK不天然提供Workspace UI |
| LLM/provider/model/retry/token/compaction | public adapter registry、DeepSeek/pi-ai providers、retry与compaction seams | base挂LLM/providers/retry/token；Web把model-facing compaction consumer移入preset |
| ToolRuntime/result/spill/checkpoint | public `ctx.tools` pipeline、spill seam/provider、durability policy | base host服务；具体tool schema按profile/preset发布 |
| MCP | 实现的optional tool bridge；无独立Service Definition，release expectation仍未在packages表声明 | CLI dependency与examples存在，base/web/standard均不默认mount |
| Skill | public Registry/Provider/Consumer | base Registry；Web的filesystem/tool consumer由per-Session preset mount |
| context/system prompt | public section/Agent instruction/time/reference seams | base + preset；exact可见输入必须log |
| approval/permission/question | public approval、permission preset、user-question seams | base服务；preset有ask tool，Web UI以Remote waterfall回答 |
| Todo/Plan/Goal | durable Session domains/tools/commands | standard/ptc/cordis preset；minimal子集；Plan是guidance非enforcement |
| subagent/jobs/workflow | public provider registries、local engine、worker workflow与tools | host registries在base，model-facing Consumers在full presets |
| shell/fs/subprocess/sandbox | public seams + local/sandbox providers | base host world，tool consumers在preset；平台选择bash/pwsh |
| terminal | public PTY seam/providers/tools | CLI闭包已安装；minimal/custom preset可mount，standard Web不默认发布persistent terminal |
| code runtime/PTC | public seam + worker-thread provider | Web/headless mode bundle；PTC preset/presentation决定Agent可见面 |
| web/attachment | public provider registries、HTTP fetch/search、durable image store | base服务，tool-web在full preset；MCP只桥image rich result |
| hooks | Claude Code/Codex bridge packages已实现 | CLI closure安装，但base/profile默认未mount |
| telemetry | SessionTelemetry seam + OTel provider | base row实际默认 `FEEDBACK_ONLY`；`DSH_TELEMETRY_DISABLED`非空会禁用 |
| settings/credentials/authorization | public seams与file/local providers | base mounted；Web controller提供redacted配置面 |
| schedule/webhook | 实现的product optional plugins | CLI closure安装，默认base/web不mount |
| E2B | POC family | 无shipped default |
| Agent Teams/WebWorker runtime | experimental/private/unreleased | 不属于正式产品图保证 |

## Out-of-tree原生capability插件边界

- 依赖Service Definition root，Provider提供物理能力，Consumer决定是否注册model-facing tool；不要依赖concrete provider/manager。
- Cordis public形态：`Context` / `Service`、`ctx.plugin()`、`ctx.provide()`、静态/命名 `inject`、`ctx.inject()`、`ctx.effect()`、`ctx.on()`；required Service消失时Consumer自动dispose，回来后重载。
- Agent-specific注册落在Agent/preset scope；Host singletons留root plane。Agent factory的`setup`在Session/Agent publication与首请求前完成，适合exact capability mount。
- Tool通过`ctx.tools.register()` / `defineTool`进入唯一ToolRuntime；policy走`tools/pre-execute`/guard，dispatch lifetime走`tools/execute`，transformation走post，immutable final outcome走`tools/result`。不要自建parallel ToolResult/Approval状态机。
- approval与ordinary question分开：effect policy通过`ctx.approval`，普通问题通过`ctx.userQuestions`/ask-user；Web/ACP Client只是回答者。
- “model-visible means logged”：模型输入必须从Session log重建。Tag允许仓内package声明合并`SessionEventMap`，但`KNOWN_SESSION_EVENT_TYPES`是构建期仓内白名单，源码明确说out-of-repo event registration deferred；alpha.1 persistence对未知事件一律拒绝。因此第三方不能仅凭declaration merge新增可恢复的required durable event。
- 安全依赖面：package root与真实public subpath（`/types`、`/client`、`/remote`、`/typert`等manifest声明项）；禁止`src/*`、tests、fixture、未导出manager。alpha.1无npm tarball，所以第三方还不能验证tag manifest与最终发布闭包一致。

## Installed / mounted / published-to-agent

- **installed**：package在profile/CLI dependency closure中可resolve；没有fiber、service、listener或tool。CLI tag manifest已安装MCP/hooks/terminal/schedule/webhook等不一定默认mount的包。
- **mounted**：enabled Loader/preset row已activate，拥有fiber/effects并可能注册Service/Provider。Provider可处于dormant状态；mount不等于Agent能调用。
- **published-to-agent**：Consumer在exact Agent scope发布tool/schema/prompt/skill，且request header/catalog/log能重建它。`tools.restrict()`影响exact Agent可见面，但不卸载Host物理capability。
- 合法占位是同版本package安装但无row、`disabled: true` Loader/preset row，或官方明确支持的dormant Provider。未来启用用同一profile/preset/patch闭包；不要注册假tool或影子Session状态机。
- Web bundle将process singletons留Host plane，把model-facing Consumers作为preset rows；这是future capability保持owner不漂移的现成模板。

## 启停、闭包、兼容与安全

- Node engine：`^22.19.0 || >=24.0.0`；CI有22.19/24/26兼容信号。
- `app-boot.boot()`安装Loader、运行可选prepare、mount root include、await settlement并audit activation；任一步失败先dispose partial root再抛。CLI对SIGTERM/SIGINT与fatal rejection统一await root fiber disposal。
- profile模板：base+web/headless/acp/sdk；sdk-minimal独立。effective tree由bundle patches→profile patch→home patch→ordered overlays组成。`cordis.yml`只是empty root；`cordis.snapshot.yml`仅在`DSH_SNAPSHOT=replay`且basename匹配时启动。
- companion审计必须覆盖CLI/app-boot、vendored Cordis 4.0.1、Loader 1.0.2、Include 1.0.6、base+mode bundle、controller/gateway/client/preset闭包、LLM/MCP SDK、persistence/storage schema、`node-addon-require-builtin`、`node-pty`、Koffi、Sharp、Landlock package family与平台runner。Python形态还需SDK/runtime-bin exact pair及rg/macOS spawn-helper/Windows payload。
- Session format仍v0，无general migration；unknown event拒绝，只有文档列出的bounded legacy record variants可读。SQLite/storage schema version mismatch fail closed，不自动迁移。old profile引用已删除ApiProxy/client-runtime也不会自动重写。
- root README明确developer preview且会breaking；AGENTS要求tagged stable release前优先正确API而非兼容shim。未找到SemVer兼容窗口、第三方desktop embedding保证、下游迁移手册或长期稳定Remote/SDK wire；SDK还明确无protocol-version negotiation。ACP v1是标准协议，但它只覆盖automation子集。
- `SAFETY.md`明确：未做security audit，不可视为secure/production-ready。sandbox、approval、permission只能reduce risk，不保证isolation/damage prevention；不得作为untrusted workload唯一security control。Plan mode只是guidance。Sandbox-local对不可用runner fail closed，但不升级为全应用安全保证。

## 未知

- npm何时发布alpha.1，以及最终tarball是否仍与tag manifest一致：未知。
- 是否会提供完整语言无关Remote协议或正式Electron IPC实现：未找到，未来计划未知。
- security audit何时发生：未知。
