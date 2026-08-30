---
title: rc.2 完整 Web Profile 与原生壳插件边界
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - package.json
  - apps/cli/package.json
  - apps/cli/README.md
  - apps/cli/reference/README.md
  - apps/cli/src/profile-boot.ts
  - apps/cli/src/process-shutdown.ts
  - apps/cli/tests/built-bin.e2e.ts
  - packages/boot/app-boot/package.json
  - packages/boot/app-boot/src/index.ts
  - packages/boot/app-boot/src/profile.ts
  - packages/bundle/base/package.json
  - packages/bundle/base/cordis.patch.yml
  - packages/bundle/web-app/package.json
  - packages/bundle/web-app/cordis.patch.yml
  - packages/client/modules/package.json
  - packages/client/modules/src/index.ts
  - packages/client/connection/package.json
  - packages/client/connection/src/client/index.ts
  - packages/client/runtime/package.json
  - packages/client/runtime/src/client/index.ts
  - packages/api/remotes/package.json
  - packages/api/remotes/src/remote-events.ts
  - packages/host/apiproxy/package.json
  - packages/core/agent/src/index.ts
  - packages/core/tools/src/index.ts
  - packages/credentials/credentials/src/index.ts
  - packages/llm/llm/src/index.ts
  - packages/session/session-persistence/src/coordinator.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-30
asked_by: agent
---

# rc.2 完整 Web Profile 与原生壳插件边界

> 八维坐标：宿主语言 TypeScript Client plugin + 任意原生壳 · 运行形态同机 `dsh` CLI子进程 + 官方Web Client · 并发与会话隔离由DSH Session拥有 · 物理平台能力由Host Provider拥有 · 工具复用官方ToolRuntime · 模型/凭据经DSH公开seam · 本地桌面部署 · 固定正式版`0.1.1-rc.2`。

## 发布闭包

- `@deepseek-ai/dsh@0.1.1-rc.2`是CLI：tarball只有`bin: dsh -> lib/bin.js`，无JS `exports`。Profile是CLI产品入口，不是可import的CLI subpath。
- `@deepseek-ai/dsh-app-boot`正式root export公开`boot()`与Profile helpers；tarball含`lib/index.js`、`lib/types/{index,profile}.d.ts`。
- `@deepseek-ai/dsh-base`与`dsh-web-app`正式公开root、`/invariant`、`/cordis.patch.yml`、`/package.json`；tarball实际含patch YAML，manifest声明`dsh.bundle.patch`。
- 根CLI integrity为`sha512-UP1UIh6q3Gme/yXRn/QL2P8IsVlv8Shpg22TRJIZPsCRWLm4CBiA1MUvXmJAfsOEETBMLAl+xWPtFw6ICsN3wg==`，shasum为`1a5112369f1c46b13a6e6f21de8af5e6afd45074`。
- 发布tarball里的DSH companion dependency多为`^0.1.1-rc.2`，Koffi/Sharp等也是caret；锁定CLI版本本身不等于冻结递归闭包，最终解析需看锁文件与每个tarball integrity。

## Custom Profile何时等价于shipped Web

- `PROFILE_TEMPLATES.web`精确为`[@deepseek-ai/dsh-base, @deepseek-ai/dsh-web-app]`。未知Profile由plugin命令初始化时默认只有base。
- Profile manifest的`dsh.profile.bundles`按顺序应用；随后是Profile patch、Home patch和argv overlays。后层可以整行替换config、disable旧行或insert新行。
- in-box bundle先从运行中的dsh installation解析，out-of-tree bundle再从Profile `node_modules`解析；`dsh plugin`把声明`dsh.bundle.patch`的依赖加入bundle list。
- 因此custom Profile以base+web-app开头、再加外部bundle，且后续层不破坏shipped行时，得到同一Web产品图加增量。DSH没有独立“完整图未变”证明；权威是最终合成树。
- bundle membership变化只在Host restart生效；既有Profile/Home patch可HMR。built-bin E2E覆盖任意custom bundle的settle、reload、revert与signal disposal。
- CLI Profile实际挂Loader产品图；SDK/ACP wire不会因为底层装更多插件自动变完整。

## Dual-face Client plugin

一个正式out-of-tree Web Client package需要同时具备：Host root、真实`./client` export、tarball内`lib/client.js`、`dsh.client.platform: web`，并在Host Loader树有enabled row。

`dsh-client-modules` Host half扫描这些Loader entries，解析`./client`，hash并服务`/plugins/<id>/client.js`，写入`window.__DSH_BOOT__`；Client half是React-free module table。

- `dsh.client.inject`只是preflight/HMR名册，不决定Cordis激活顺序。
- Cordis Service `inject`决定fiber等待；非baseline module value依赖用`dsh.client.external`。
- package set变化需restart；bundle内容重建可走client HMR。

正式可消费入口：

| package | rc.2入口 |
|---|---|
| `dsh-client-modules` | root、`/client` |
| `dsh-client-connection` | root、`/client` |
| `dsh-client-runtime` | root、`/client` |
| `dsh-api-remotes` | root、`/client`、`/types` |
| `dsh-api-gateway` | root、`/client`、`/types` |
| `dsh-typert-registry` | root、`/client`、`/types` |
| `dsh-host-apiproxy` | root、`/api`、`/api/*`、`/client` |

manifest中的`./src/*`不算：正式tarball没有`package/src/`。

## Transport与原生壳

`dsh-client-connection/client`公开`ClientTransportHooks`：

- `createApiClient(): IApiClient`：legacy ApiProxy unary + mux/host streams；
- `fetch: RpcFetch`：Typert/generic Connection unary；
- `loadBundle?(url)`：shell拥有bundle bytes时替换HTTP script loading。

served Web Client不设置global，走HTTP POST +两条WebSocket。README明确以worker preview的postMessage tunnel为替换先例；但没有WKWebView/Swift指南、成品carrier factory、语言无关协议版本或任意Host→Client event bus。

Host root公开`ctx.connection.rpc.handle/intercept`，Client公开`ctx.connection.rpc.call`，只形成Client→Host unary扩展面。`api-remotes`事件allowlist与Remote mounts在build-time固定；新增Host事件不是runtime discovery。

条件性结论：直接加载loopback Web Host可沿用官方carrier；file:// + IPC时，JavaScript插件可用public hooks拥有DSH codec，原生壳只搬运物理消息。它是接口形状支持的推论，不是WebKit官方保证。

## 原生capability owner

公共组合面包括Cordis`Context/Service`，`dsh-scope`，`dsh-agent`的unpublished`setup`，`dsh-tools`的register/pre/execute/post/result，approval、user-questions、credentials、LLM adapter、Session与AgentPreset roots。

- clipboard/selection/Accessibility replace/TTS/notification没有rc.2官方Service Definition或Provider。外部插件可定义Host Service/Provider/Agent Consumer；模型工具必须进入`ctx.tools`与official approval/result链。
- OS权限、entitlement、物理副作用安全属于Host平台owner；DSH不提供macOS隔离保证。
- Keychain没有一方provider，但可实现公开abstract`CredentialProvider`并替换`credentials-local`；不得并行维护第二secret truth。
- Cloud LLM可实现公开`LlmAdapter`并注册routes；DSH继续拥有catalog/capability/retry/request assembly/header。

## Native trigger到Session

React-free Client Runtime公开Workspace list/create/connect/start、Session list/open/binding、`SessionFace.prompt(queue|steer)`、cancel、history、models等。

- Host `session.create`公开`agentPreset?`并把resolved id写Session header；blank Session也可`agentPreset.select`。
- `IWorkspaces.connectWorkspace(workspaceId)`没有preset参数。要求创建时指定preset的Client插件必须走公开IApiClient create，或blank create后再走公开select，不能把该参数虚构到Workspace facade。
- 选区文本进入模型应经Session prompt成为`user/message`；follow-up用queue，打断用steer，Stop用cancel。
- Runtime公开`PendingWait.respond`及ApiProxy公开approval/question payload。官方ui-user-questions `/client`公开`PendingQuestion`；ui-conversation内部`PendingApproval`没有从正式`/client` barrel导出，外部不能私有import。
- 平台主动把selected-text trigger推到Client没有通用DSH Host→Client seam；需平台WebView桥或显式构建Remote contribution。

## Installed / mounted / published

- installed只表示可resolve；没有fiber/Service/tool。
- mounted表示enabled Loader/Profile/Preset row激活；`disabled: true`不挂载。
- published-to-agent表示tool/schema/prompt/skill在exact Agent scope可见。
- Web overlay把process singleton留Host plane，显式disable base中的agent-plane Consumers，再由standing AgentPreset按Session scope发布。
- capability-off可用安装无row、disabled row或dormant Provider无Consumer；不能发布fake tool或影子Session状态。
- `model-visible means logged`：进入model request的输入必须由Session log重建。选区prompt可复用标准user message。
- rc.2的外部SessionEvent只有`ignorable:true`能越过unknown-type read gate；它不能承载required composition truth。alpha.1后来连该例外也移除。
- Session只耐久AgentPreset id；definition/content digest不随Session保存，resume按当前roster解析。

## Web激活与关闭

- base提供Agent/Loop、Session、LLM、ToolRuntime、persistence、settings/credentials、sandbox/approval/permission、fs/subprocess及各Host registry。
- web-app增加Storage/Workspace/projection cache、WebServer/ApiProxy/Connection、Client modules/runtime/remotes、UI roster、references、code runtime与preset roster；default为standard。
- standard preset发布shell/fs/jobs/skills/goals/plan/compaction/subagent/workflow/question/todo/web等Agent Consumers。MCP与full-text content search默认不启用。
- tag根Node要求`^22.19.0 || >=24.0.0`，packageManager为pnpm 11.7；正式CLI manifest自身没有`engines`。
- tag lock的关键native closure：`node-addon-require-builtin 0.1.4`、patched `node-pty 1.2.0-beta.15`、Koffi 3.1.1、Sharp 0.35.3/libvips 1.3.2、Landlock family 0.1.1。
- boot失败先dispose partial root。SIGTERM drain后exit 0，SIGINT exit 130；默认grace 5秒，超时/dispose失败/第二signal force exit；late fail-loud release上限2秒。

## alpha.1断裂

alpha.1删除`dsh-host-apiproxy`与`dsh-client-runtime`，改为Session/Workspace/Settings Controllers与Client Store；`ClientTransportHooks.createApiClient`消失，换成`fetch/openStream?/loadBundle?/ownsHost?`和generation owner。所有rc.2 legacy `/client`、IApiClient、SessionFace与transport hook依赖都需重新编译迁移。

## 未知

- 未找到rc.2 WKWebView/Swift/macOS shell官方指南或一方测试。
- 官方是否会提供通用Host→Client native event seam：未知。
- macOS平台能力的具体schema、权限与failure vocabulary由外部插件定义，DSH无现成答案。
