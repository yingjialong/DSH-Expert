---
title: packages/preset — 每会话 Agent 组合（agent preset）
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/preset/README.md
  - packages/preset/agent-presets/README.md
  - packages/preset/agent-presets/src/index.ts
  - packages/preset/agent-presets/src/session.ts
  - packages/preset/persona/README.md
  - apps/cli/config/agent-presets/
  - .agents/notes/implemented/architecture/2026-08-03-per-session-agent-presets.md
  - .agents/notes/implemented/architecture/2026-08-08-per-preset-standing-mounts.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-30
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
- 创建时的 resolved id 写入 `SessionHeader.agentPreset`；空白 Session 后续切换才追加 `agent-preset/selected`。JSONL / SQLite persistence 持久化的是这个 id 与事件，不是 composition 内容、digest 或 generation。
- 冷恢复会从 header + 最后一个 selection event 取 recorded id，再向当前 roster 解析并于 Agent publication 前 mount。因此定义文件仍在 root 时无需每次手工注册；若定义只存在于宿主内存，则每次 boot 必须在恢复前重新物化。
- 缺定义不是统一的 fallback 契约：roster 仍装配但 exact id 缺失时，真实 Agent resume 不回退 default，且 publication 失败；只读 transcript / cold skill catalog 会退到 global presenter/scope；若整个 `agentPresets` 服务都未装配，rc.2 ApiProxy 会采用 rosterless Host composition。最后一条缺少专门一方测试，按实现证据仅标 `verified_inference`。
- 已运行 Session 的 standing mount 可在文件删除后继续；进程重启后仍依赖磁盘定义。相同 id 的内容可漂移，DSH 不验证 id 是否等于内容摘要。
- `dsh-v0.1.2-alpha.1` tag 把当前 id fold 改为 Session projection、增加 Typert remote 与 shipped root，但仍是文件系统 roster，无任意 definition 注册/持久化 seam；截至本次查询，preset 包无 alpha.1 npm 版本。

## 2026-08-30 · rc.2 create / blank select / prompt 时序

- `session.create` 在 Session 构造前先把请求 id/default 解析为当前 roster 的 preset id；Agent factory 随后以 Session/Agent 均未发布的 setup mount standing composition，setup/commit 成功后才依次发布 Session、Agent、`agent/session-start` 并返回。此时没有 `turn/start`，所以仍为 blank。
- `agentPreset.select` 只在 live Session 仍 blank 时生效：同 Session 的多个 select 在专用 queue 串行并在队内重检 blank；`recompose()` 确保新 standing mount 后只重绑原 Agent scope，不销毁/重建 Agent或 Session；提交成功后才追加 `agent-preset/selected`。header 保留创建事实，最后一条 selection event 是 resume/list 的 current id。
- 首个 `session.prompt` 通过普通 Agent inbox 启动第一个 turn；此后再 select 得到 `agent-preset-locked`。但 prompt admission 不进入 preset-select queue，且没有独立 finalize/lock event；固定 tag 未给“并发 select 与首 prompt”的原子相对顺序保证。因此 blank whole-preset switch 不能当成任意 Skill/MCP draft 在首 prompt 边界原子固化的公共契约。
- resume 在持久日志 load 后、Agent publication 前按 `resolveSessionPreset(header, events)` 的 id向当前 roster resolve/mount；ordinary Session fork也按 source log current id重新 resolve当前 roster，而不是继承 source live standing generation。只有 subagent `composeFrom()` 明确绑定父 Agent 已运行的同一 standing generation。
- **认知状态**：verified_inference（`dsh-agent` public setup/publication契约、Host ApiProxy实现与 agent-preset一方测试交叉核对；未实测并发 prompt/select交错）。
