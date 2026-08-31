---
title: packages/preset — 每会话 Agent 组合（agent preset）
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/preset/README.md
  - packages/preset/agent-presets/README.md
  - packages/preset/agent-presets/src/index.ts
  - packages/preset/agent-presets/src/authoring.ts
  - packages/preset/agent-presets/src/discovery.ts
  - packages/preset/agent-presets/src/mount.ts
  - packages/preset/agent-presets/src/session.ts
  - packages/preset/agent-presets/tests/discovery.spec.ts
  - packages/preset/agent-presets/tests/authoring.spec.ts
  - packages/preset/agent-presets/tests/mount.spec.ts
  - packages/preset/agent-presets/tests/session.spec.ts
  - packages/preset/persona/README.md
  - packages/core/scope/src/index.ts
  - packages/core/session/src/index.ts
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent/src/index.ts
  - packages/host/apiproxy/src/api-proxy.ts
  - packages/host/apiproxy/src/api/agent-presets.ts
  - packages/host/apiproxy/src/api/sessions.ts
  - packages/host/apiproxy/tests/api-proxy-agent-preset.spec.ts
  - packages/client/runtime/src/client/sessions/manager.ts
  - apps/cli/config/agent-presets/
  - .agents/notes/implemented/architecture/2026-08-03-per-session-agent-presets.md
  - .agents/notes/implemented/architecture/2026-08-08-per-preset-standing-mounts.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-31
asked_by: agent
---

## 一句话定位

让同一个进程里的多个 session 各自跑不同的 agent 组合：一个 preset 就是一个装着 `agent.cordis.yml` 的目录，挂载后该 session 拥有自己的 tools 与 prompt sections，其他在跑的 session 不受影响。

## 稳定性

`Product — stable API`（[T1: packages/README.md 表格 `preset/` 行]）。

## 包清单

| 包名 | npm 名 | 职责 |
|---|---|---|
| `agent-presets` | `@deepseek-ai/dsh-agent-presets` | preset 词汇表、文件系统发现（trusted + user-authored roots）、带守卫的 per-agent mount；ctx key `agentPresets` |
| `persona` | `@deepseek-ai/dsh-persona` | 把 agent persona 做成一个可组合的 plugin row，使 preset 能换身份而不只是换工具；无 ctx key |

## 三件套结构（Service Definition / Provider / Consumer）

这个组**不是标准三件套**：`agent-presets` 一个包同时是 Service Definition 与唯一实现（`AgentPresets extends Service`，`ctx.agentPresets`），没有 provider 注册点。`persona` 是「被 preset 组合进去的 row」，属于 preset 内容而非 provider。

真正的 Consumer 分布在别的组：agent factory 的 `setup(agentCtx)`（唯一支持的 `mount()` 调用点）、`dsh-subagent`（子 agent 走 `composeFrom()` 而非 `mount()`）、`dsh-apiproxy`（在 wire 层回 `agent-preset-locked`）。

## 扩展点

- **依赖对象**：`@deepseek-ai/dsh-agent-presets`（即 Service Definition + 实现同体）；client 侧只需要 `dsh-agent-presets/types`（导出 `agent-preset/selected(sessionId, agentPreset)` 这个非 scoped 的 cordis 事件，避免 import Host runtime 类型）。
- **要写一个新 preset**：不写代码。建一个目录，放 `agent.cordis.yml`（顶层是 plugin row 列表），可选 `preset.yml` 只放 `name` / `description` 展示文本（`id` 来自目录名、`trust` 来自 root，都不可在此覆写）。
- **要给 preset 增加"能改身份"以外的能力**：写普通 Cordis 插件，让 preset 的 `agent.cordis.yml` 引用它即可；package 名从 host composition 解析，相对路径从 preset 目录解析。
- **不可扩展的方向**：向 root realm 发布 service 的 row 会在 mount 时被拒（会与下一个 preset 撞名）；真要拥有 service 就放进 `isolate` realm，或干脆放回 host composition。
- **关键 API**（[T1: packages/preset/agent-presets/src/index.ts#AgentPresets]）：`list()` / `resolve(id?)` / `mount(agentCtx, id?)` / `composeFrom(agentCtx, parentCtx)` / `composedPreset(agentCtx)` / `recompose(agentCtx, id)` / `standingKeyFor(id?)` / `read(id)` / `copy(from, id, name?)` / `remove(id)`，外加 `defaultId` / `roots` / `authorable`。
- **判断一个 session 当前跑哪个 preset**：用 `resolveSessionPreset(session)`（[T1: packages/preset/agent-presets/src/session.ts#resolveSessionPreset]），不要读 `SessionHeader.agentPreset`——header 只记「创建时」的 preset，切换后会不一致。

## Known Limitations

来自 [T1: packages/preset/agent-presets/README.md] 与 [T1: packages/preset/persona/README.md] 的 `## Known Limitations and Deferred Work`：

- 只有第一个 `user` root 下的 preset 可以 `remove()`；其他 root 下的可发现、可挂载、但删不掉。
- 只有「尚未产出任何内容」的空 session 能 `recompose()` 换 preset；改默认值只影响之后创建的 session。
- generation 只以 `agent.cordis.yml` 的 stamp（mtime + size）为准，改 skill 文件或旁边的资产不会触发新 generation。
- 被取代的旧 generation 永不回收，整个 subtree 挂到进程结束（`dsh-skill-filesystem` 默认 watch roots，每次「编辑再新建」会多一组 live watcher）。
- `copy()` 从不挂载校验，坏源产生同样坏的副本；discovery 的健康检查只是形状检查（能否解析 + 是否为具名 row 列表），不验证每个 row 的模块能否 resolve/activate。
- copy 是快照会漂移：升级部署不会更新已复制的 shipped preset，这一层没有 patch 语义（那是 bundle 层 `cordis.patch.yml` 的事）。
- root 扫描不 watch，每次 `list()` 每个 root 一次 `readdir`。
- `dsh-persona` 无法全局挂载：prompt registry 已经拥有 unscoped 的 `deployment:persona` 槽位，这个 row 只能在 scoped composition（即 preset）里用。

## 陷阱

- **`mount()` 只能在 agent factory 的 `setup(agentCtx)` 里调**。别处调会让失败的组合留下半成品 session。
- **子 agent 必须走 `composeFrom()`，不能重新 `mount()` 同一个 id**：重新 mount 可能拿到不同 generation（父启动后文件被改过）或直接失败（preset 被删），而且 `composeFrom()` 是同步的——in-process subagent driver 只有同步创建窗口可用。
- **preset 文件是输入，绝不是持久化目标**：Loader 认为 config 变了就会写回源文件（一个 row dispose 自己的 fiber 就够触发）。挂载的 subtree 因此把 `write()` 覆盖成 no-op。
- **模型可见 ⟺ 已记录**：切换 preset 会在提交后追加 `agent-preset/selected` session 事件，因为 preset 决定模型看到的 tool schema 与 prompt sections。
- **trust 不是强制项**：`trust: 'system' | 'user'` 只是给消费者展示用；preset 的权限等同于它命名的插件，user preset 等价于 shell 访问权。
- **shipped preset 名单不在本组文档里**：以 `apps/cli/config/agent-presets/` 的目录清单为准（当前为 `standard` / `code` / `cordis` / `minimal`）；README 明确拒绝在文档里再列一份。
- **cold transcript read 不是“不会激活preset”的纯数据读取**：`session.history`会先解析recorded effective preset，再调用`standingKeyFor()`；首读或stamp变化会真实Include/Loader activate该`agent.cordis.yml`。这条路径没有Agent，因此不会发`agent/created`；只在该事件检查composition identity既晚于正常Agent setup，也完全漏掉cold presenter mount。
- **“旧Session保留旧generation”只适用于仍live/joined的Agent**：Session log只存preset id，不存generation/stamp/digest。Agent dispose、cold presenter或Host重启后都按current roster重新解析同id；same-id overwrite会改变旧持久Session下一次实际装配的composition。

## 去哪深入（文件路由）

| 问题 | 去这里 |
|---|---|
| 组内包/ctx key 映射、组合分工原则 | `packages/preset/README.md` |
| 服务 API 全签名、mount 拒绝规则、authoring、Config、settings 集成 | `packages/preset/agent-presets/README.md` |
| 实现：discovery / mount / authoring / metadata / session 解析 | `packages/preset/agent-presets/src/{discovery,mount,authoring,metadata,session}.ts` |
| 挂载时的运行期不变式（service 发布到 root realm 的复查） | `packages/preset/agent-presets/src/invariant.ts` |
| persona row 的 `complete` / `includeRuntimeContext` 语义 | `packages/preset/persona/README.md` |
| 设计决策：为什么是 per-session preset | `.agents/notes/implemented/architecture/2026-08-03-per-session-agent-presets.md` |
| 设计决策：per-preset standing mount（一次挂载多 session 共享） | `.agents/notes/implemented/architecture/2026-08-08-per-preset-standing-mounts.md` |
| 出厂 preset 实物 | `apps/cli/config/agent-presets/{standard,code,cordis,minimal}/agent.cordis.yml` |

## 2026-08-30 · rc.2 专项复验：动态定义与冷恢复

- rc.2 没有 `registerPreset(id, definition)`、`AgentPresetProvider` 或 definition backend。`AgentPreset` 是文件系统发现结果，不是可注册的 composition definition；正式写入口只有整目录 `copy()`，不接受任意 composition 文本。
- 可支持的运行期扩展是：宿主在服务构造时已经配置的 root 下物化 `<id>/agent.cordis.yml`。`list()` / `resolve()` 每次重新扫描，所以下一次 `session.create({ agentPreset: id })` 可立即选择它；root 集合本身在服务构造时固定。
- root内容无memo/negative cache：`resolveMountable()`、`standingKeyFor()`与Host `composeAgent()`最终都重新走`list→discoverPresets→scanRoot`。已配置root内先完成原子目录发布、再调用Host create即可；但roots顺序first-root-wins，较早root的同id会shadow后投影。DSH没有外部publish与create的共同CAS，调用并发时只服从文件系统可见顺序。
- 创建时的 resolved id 写入 `SessionHeader.agentPreset`；空白 Session 后续切换才追加 `agent-preset/selected`。JSONL / SQLite persistence 持久化的是这个 id 与事件，不是 composition 内容、digest 或 generation。
- 冷恢复会从 header + 最后一个 selection event 取 recorded id，再向当前 roster 解析并于 Agent publication 前 mount。因此定义文件仍在 root 时无需每次手工注册；若定义只存在于宿主内存，则每次 boot 必须在恢复前重新物化。
- 缺定义不是统一的 fallback 契约：roster 仍装配但 exact id 缺失时，真实 Agent resume 不回退 default，且 publication 失败；只读 transcript / cold skill catalog 会退到 global presenter/scope；若整个 `agentPresets` 服务都未装配，rc.2 ApiProxy 会采用 rosterless Host composition。最后一条缺少专门一方测试，按实现证据仅标 `verified_inference`。
- 已运行 Session 的 standing mount 可在文件删除后继续；进程重启后仍依赖磁盘定义。相同 id 的内容可漂移，DSH 不验证 id 是否等于内容摘要。
- `dsh-v0.1.2-alpha.1` tag 把当前 id fold 改为 Session projection、增加 Typert remote 与 shipped root，但仍是文件系统 roster，无任意 definition 注册/持久化 seam；截至本次查询，preset 包无 alpha.1 npm 版本。

## 2026-08-30 · rc.2 create / blank select / prompt 时序

- `session.create` 在 Session 构造前先把请求 id/default 解析为当前 roster 的 preset id；Agent factory 随后以 Session/Agent 均未发布的 setup mount standing composition，setup/commit 成功后才依次发布 Session、Agent、`agent/session-start` 并返回。此时没有 `turn/start`，所以仍为 blank。
- `agentPreset.select` 只在 live Session 仍 blank 时生效：同 Session 的多个 select 在专用 queue 串行并在队内重检 blank；`recompose()` 先 ensure 新 standing generation，再同步 bind/rebind 原 Agent 的 scope parent，不销毁/重建 Agent或 Session；它返回后 ApiProxy 才追加 `agent-preset/selected`。header 保留创建事实，最后一条 selection event 是 resume/list 的 current id。
- `recompose()` 与 durable append **不是统一事务**：unknown/broken standing 与 cycle-check失败都发生在 parent link 写入前，旧 composition 保持；但 append 的JSON/reentry/invariant检查发生在 scope 已重绑之后，ApiProxy没有 append 失败时反向 rebind 的rollback。正常成功路径才同时拥有新 live composition与新 durable selection；不得把“event在提交后追加”扩成“任一步失败都恢复旧composition”。
- append失败后也没有公开的“回到旧exact generation”补偿面：`ScopeParentBinding`实例被AgentPresets私有WeakMap持有，`standingKeyFor(id)`/`recompose(id)`只解析current standing，同id文件已变化时不能指定historical key。旧/新standing generation会一直存活到whole-tree teardown；源码明确把refcount回收列为TODO，没有per-session release API。
- 首个 `session.prompt` 通过普通 Agent inbox 启动第一个 turn；prompt不进入 preset-select queue，也不执行同一 blank recheck，RPC accepted 后 `turn/start`由driver稍后追加。固定 tag未给并发select与首prompt的统一linearization/finalize契约。因此blank whole-preset switch不能当成任意Skill/MCP draft在首prompt边界原子固化的公共契约。
- resume 在持久日志 load 后、Agent publication 前按 `resolveSessionPreset(header, events)` 的 id向当前 roster resolve/mount；ordinary Session fork也按 source log current id重新 resolve当前 roster，而不是继承 source live standing generation。只有 subagent `composeFrom()` 明确绑定父 Agent 已运行的同一 standing generation。
- **认知状态**：verified_inference（`dsh-agent` public setup/publication、`dsh-scope` rebind、Session append与Host ApiProxy控制流交叉核对；未实测故意制造append失败或并发prompt/select交错）。

## 2026-08-30 · rc.2 无preset legacy Session

- `resolveSessionPreset()`在header与events都无ID时返回`undefined`。新建Session：有roster且请求省略ID会把当时default写入新header；无roster则保持preset-less Host composition。
- 既有preset-less Session若`session.create`显式命名任意preset，ApiProxy返回`agent-preset-conflict`；省略则允许adopt。冷resume在roster存在时把`undefined`交给`composeAgent()`，于是按**恢复时current default**mount，但不回写旧header；无roster则Host composition。
- history/presenter只读路径对无ID Session尝试default standing scope，失败后退global并继续服务；它不是Agent resume成功保证。rc.2 `skills.list`要求Session已attached。
- ordinary fork对无ID source：roster存在时按fork时default创建child并把该ID写child header；无roster时child仍无ID。这是current fallback，不是legacy digest/ref migration。
- **认知状态**：verified_inference（ApiProxy实现与preset-less adoption一方测试；resume/fork无ID分支由源码控制流交叉核对，未找到独立专项测试）。

## 2026-08-30 · rc.2 Session引用枚举、remove与version coexistence

- 公共`session.list`只枚举effective`agentPreset` id；header/event可回放同一选择。Session不保存plugin rows/package versions/definition digest，亦无Session→plugin引用表。
- `AgentPresets.remove(id)`不做Session ref check：删除user definition并清current standing pointer；已live Agent仍靠旧standing跑到进程结束。重启后recorded id在current roster缺失，真实resume失败。
- `agentPreset.read/copy/remove`只面对current roster；`session.export`只导出raw Session artifact，不含preset definition。无historical generation handle、ref-aware retention/tombstone、clear-reference、删前影响枚举或historical export。
- 同进程同id改文件时已join Session可留旧private standing、新Session用新generation；该generation不可公开寻址且不跨重启。只有不同且长期保留的immutable preset ids及其模块同时在current closure时，preset-scoped旧/新composition才可条件并存；DSH不自动管理plugin version coexistence，Host-plane全局plugin也不因此版本化。
- generation cache按id single-flight；失败entry会删除供下次重试。成功后仅比较`agent.cordis.yml`的`mtimeMs+size`：不同才建立later generation，同值内容漂移不刷新；stamp又在Loader读取前取得，没有byte/digest绑定。两位later-session racer会共享一个new generation，superseded scope则直到whole-tree teardown都不回收。
- 因此immutable content-addressed id不是DSH运行所必需——rc.2明确支持same-id编辑；但若要求cold reopen/restart精确重建，它是外部必要不变量，因为只有仍live的parent binding能寻址old generation，持久Session没有historical handle。
- **认知状态**：verified_inference（固定tagPreset authoring/standing、ApiProxy与Persistence语义交叉核对；未安装第三方多版本plugin）。

## 2026-08-31 · 预物化content-addressed preset的公共组合边界

- rc.2公共roster可以消费外部在已配置root内预先物化的`<digest-id>/agent.cordis.yml`：preset id语法允许小写hex/连字符，`list()`/`resolve()`每次重扫目录，create把resolved id写入标准`SessionHeader.agentPreset`，cold resume按header与标准`agent-preset/selected`fold出的id在unpublished setup内mount。因此selection已编码为preset id时不需要新增required SessionEvent。
- DSH不提供任意definition bytes写入口：`copy(from,id)`只能复制既有目录；外部owner必须自己负责canonical bytes/digest、原子目录发布、id-content一致性、依赖闭包、跨重启保留与ref-aware GC。DSH既不校验id等于内容摘要，也不把definition/plugin graph写进Session。
- 标准ApiProxy没有按recorded id调用out-of-tree definition resolver的注册点。目录必须在create/resume进入roster解析前存在；自定义Agent入口可用公开`AgentRegistry` setup，但这不是标准ApiProxy上的setup chain。
- standing mount是一preset generation一棵共享plugin subtree，不是每Session一份。多个Session复用同id会join同一实例；失效后新Session也不会自动重跑preflight。需要fresh instance时必须有新preset/file generation、Host重启，或让共享definition内部按`exec.agent`管理per-Session资源。
- 正式Host`@deepseek-ai/dsh-host-apiproxy/api`的`SessionsApi.create`公开`sessionId?`与`agentPreset?`，并承诺resolved id写header；但高层`@deepseek-ai/dsh-client-runtime`的`SessionRuntime.create`只公开`workspaceId?/cwd?/sessionId?`，没有preset参数。调用方必须区分Host API seam与Client convenience层。
- ordinary ApiProxy fork用`resolveSessionPreset(source)`取得source当前effective id并写入child header/setup，不取current default；但它重新按当前roster/standing解析同id，不持久克隆private live generation。只有source无任何recorded id的legacy/preset-less分支才会在有roster时采用fork时default。
- **认知状态**：verified_inference（固定rc.2正式root exports/`.d.ts`、roster/discovery/mount、ApiProxy create/resume与一方tests交叉核对；未写外部definition store）。

## 2026-08-31 · cold history presenter会激活standing composition

- `session.history`对attached/cold source都先调用`presenterScopeFor()`，live Agent直接作为scope；无live Agent但有roster时，根据header与最后一条`agent-preset/selected`解析effective id并调用`standingKeyFor()`。该调用不按页面内容或是否实际含tool event短路。
- `standingKeyFor()`经`ensureStanding()`在首读/文件stamp变化时创建standing scope并执行`mountPreset()`；Include/Loader会真实import/apply全部enabled plugin rows，然后才检查inactive row与root-realm service leak。真实一方test固定了“standing mount存在，但无Agent/Session/turn”。
- 因为cold路径不创建Agent，`agent/created` listener完全不运行。正常create/resume也先完成unpublished setup/preset mount，进入Session/Agent registry后才announce，因此该event只能观察或在同步throw时回滚publication，不能作为preset plugin activation之前的唯一准入门。
- roster缺失直接用global presenter；unknown/deleted/broken/unusable preset由`presenterScopeFor()`吞掉standing failure并退global，history继续返回generic card。单个presenter/JSON parse失败也只省略view；只有persistence inspect/source失败才使整个history返回`internal`。
- mount失败会dispose失败scope，收回遵守Cordis effect ownership的注册；DSH不承诺回滚plugin自行造成、未登记为effect的外部副作用。user preset的trust标签也不是代码sandbox。
- **认知状态**：verified_inference（固定rc.2正式host-apiproxy/agent-presets runtime JS、Agent lifecycle类型与cold/presenter/mount一方tests；未加载恶意plugin）。
