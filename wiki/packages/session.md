---
title: packages/session — 持久 Session 数据平面
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/core/session/src/json.ts
  - packages/core/session/tests/json.spec.ts
  - packages/session/README.md
  - packages/session/session-persistence/README.md
  - packages/session/session-persistence/src/index.ts
  - packages/session/session-persistence/src/coordinator.ts
  - packages/session/session-persistence/tests/coordinator-contract.ts
  - packages/session/session-persistence-jsonl/README.md
  - packages/session/session-persistence-sqlite/README.md
  - packages/session/session-checkpoint-policy/README.md
  - packages/session/session-projection/src/index.ts
  - packages/session/session-projection-cache/README.md
  - packages/session/session-projection-cache/src/index.ts
  - packages/session/session-projection-cache/src/spec.ts
  - packages/session/session-projection-cache/tests/cache.spec.ts
  - packages/session/session-stats/README.md
  - packages/session/session-title/src/index.ts
  - packages/session/session-title/src/normalize.ts
  - packages/session/session-telemetry/src/index.ts
  - packages/session/session-telemetry-otel/README.md
  - packages/host/apiproxy/src/api-proxy.ts
  - packages/host/apiproxy/src/api/sessions.ts
  - packages/host/apiproxy/tests/api-proxy-projections.spec.ts
  - packages/client/runtime/src/client/sessions/manager.ts
  - packages/storage/storage/src/backend.ts
  - packages/storage/storage-domain/src/index.ts
  - packages/bundle/web-app/cordis.patch.yml
  - docs/subsystems/persistence.md
  - docs/subsystems/session-projection.md
  - docs/subsystems/session-title.md
  - docs/subsystems/session-telemetry.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-31
asked_by: agent
---

## 一句话定位

围绕 `core/session` 那个 live 内存服务的**持久家族**：持久化 seam + 存储后端 + checkpoint 策略、投影（projection）seam、日志推导的标题、以及外发遥测。全部是 **product** 包。

## 稳定性

`Product — stable API`（[T1: packages/README.md]：`session/` = "Durable session data plane: persistence seam + JSONL/SQLite backends, projection seam, log-backed titles, session reporting"）。

## 包清单（13 个，按四个子域分）

| 包名 | npm 名 | 职责 / ctx key |
|---|---|---|
| `session-persistence` | `@deepseek-ai/dsh-session-persistence` | **持久化 Service Definition** + 共享写协调；`ctx.sessionPersistence` |
| `session-checkpoint-policy` | `@deepseek-ai/dsh-session-checkpoint-policy` | 语义耐久 checkpoint；wraps `ctx.llm` 和 `ctx.tools`，零配置 |
| `session-persistence-jsonl` | `@deepseek-ai/dsh-session-persistence-jsonl` | JSONL 后端（默认 `.jsonl.zstd`）；registers on `ctx.sessionPersistence` |
| `session-persistence-sqlite` | `@deepseek-ai/dsh-session-persistence-sqlite` | opt-in SQLite 后端，packed 物理 chunk 行；registers on `ctx.sessionPersistence` |
| `session-projection` | `@deepseek-ai/dsh-session-projection` | **投影 Service Definition** + drive registry；`ctx.sessionProjections` |
| `session-projection-cache` | `@deepseek-ai/dsh-session-projection-cache` | 持久化/恢复投影 checkpoint；`ctx.sessionProjectionCache` |
| `session-stats` | `@deepseek-ai/dsh-session-stats` | 注册 `sessionStats` 投影单元（全日志轮次数与各类墙钟时间） |
| `session-title` | `@deepseek-ai/dsh-session-title` | 标题状态、fallback、provider 注册、refresh；`ctx.sessionTitle` |
| `session-title-llm` | `@deepseek-ai/dsh-session-title-llm` | **库不是插件**：模型型标题 provider 的共享实现策略 |
| `session-title-first-prompt-llm` | `@deepseek-ai/dsh-session-title-first-prompt-llm` | `first-prompt` cadence provider；registers on `ctx.sessionTitle` |
| `session-title-all-prompts-llm` | `@deepseek-ai/dsh-session-title-all-prompts-llm` | `all-prompts` cadence provider；registers on `ctx.sessionTitle` |
| `session-telemetry` | `@deepseek-ai/dsh-session-telemetry` | **遥测 Service Definition**：捕获、脱敏、投影、交付；ctx key `sessionTelemetry` |
| `session-telemetry-otel` | `@deepseek-ai/dsh-session-telemetry-otel` | OTel 后端，`FULL` / `FEEDBACK_ONLY` / `DISABLED` 三模式 |

## 三件套结构（本组是四条独立的 seam）

| seam | Service Definition | Provider | Consumer |
|---|---|---|---|
| 持久化 | `session-persistence`（`SessionPersistence extends Service`） | `session-persistence-jsonl` / `session-persistence-sqlite` | `session-checkpoint-policy`、`core` 的 resume/fork、`session-query`、`session-log-export` |
| 投影 | `session-projection`（`SessionProjectionRegistry`） | 注册 `ProjectionDefinition` 的领域包，如 `session-stats`、goal、todo | api-proxy 的 history tail page、`session/projection` push frame、session 列表行 |
| 标题 | `session-title`（`SessionTitleService`） | `session-title-first-prompt-llm` / `session-title-all-prompts-llm`（**同时只允许一个**） | Web/CLI 的 session picker、列表行 |
| 遥测 | `session-telemetry`（`SessionTelemetryBackend` / `SessionTelemetrySink`） | `session-telemetry-otel`（**部署方唯一加载的入口**） | `/feedback` 命令的确认文案（读 `sharing`） |

`session-title-llm` 是**库**（不是 Cordis 插件）：两个 provider 插件调它的 `registerSessionTitleLlmProvider()`，保证注册、路由、prompt、取消、校验行为不漂移。

## 扩展点

- **写一个新的持久后端**：继承 `SessionPersistence`（[T1: packages/session/session-persistence/src/index.ts#SessionPersistence]），实现 `locate` / `supportsRawArtifacts` / `readRaw` / `create` / `append` / `prepare` / `load` / `inspect` / `list`。持久单元**就是** `SessionEvent`（事件溯源，日志是唯一真相），没有平行的"persisted message"类型；不可回放的元数据（format version、cwd、lineage、seed 边界、origin、delegation depth）走 `SessionHeader`（由 `dsh-session` 拥有，此处 re-export）。
- **注册一个投影单元**：`ctx.sessionProjections.register(definition)`，`ProjectionDefinition<K,S> = { key, stateSchema, init(), apply(state,event), wire?, stateVersion }`——纯同步 fold 加声明加**可选**的 client view（`wire = { viewSchema, view(state) }`），不是不透明 getter。key 必须先并进 `SessionProjectionStateMap`（host fold 状态表，`types.ts` 新引入的 merge-extensible 类型表）；要 client 可见还要并进 `SessionProjectionMap`（wire 值表）。省略 `wire` 即注册 **host-only 单元**：state 不出现在 client snapshot、不进 change feed，但照常写投影 cache。读法对应分开：`snapshot(session)` 只返回 client-visible 值；`stateOf(session, key)` 单读一个单元的 live host state（返回的是借用引用，不得改）。返回的是 effect disposer。
- **写一个标题 provider**：`ctx.sessionTitle.register(provider)`（**只接受一个**，第二次注册立即 throw）。要复用模型调用逻辑就依赖 `session-title-llm` 而不是抄一份。
- **写一个遥测后端**：实现 `SessionTelemetrySink` 的三个成员——`emit(record)`（**必须非阻塞入队**，它在 `session/event` 或显式回放期间同步跑）、可选 `flush()`（turn 结束后的 fire-and-forget 提示）、`shutdown()`（排空并等 SDK 停）。再用 `live` 或 `on-demand` 构造 `SessionTelemetryCoordinator`，自选触发点调 `captureSession(session, throughSeq?)`。必须填 `sharing`（`full` | `feedback-only` | `disabled`）。
- **写脱敏规则**：挂一个 `sessionTelemetry/record` listener。**不挂就等于零脱敏**。
- **依赖纪律**：扩展插件依赖上面四个 Service Definition 包，不要依赖 `-jsonl` / `-sqlite` / `-otel` 这些具体 provider。

## Known Limitations

（13 个包的 `## Known Limitations and Deferred Work` 提炼，按主题合并）

- **没有删除/保留期 API**：持久化 seam 无删除接口，日志在 `root` 下无限累积；投影 cache 同样无淘汰。清理属于带外运维。
- **`list()` 不分页不过滤**：返回每个已存 session 的 header，本地规模够用，规模化时无索引。
- **崩溃故事只有 repair-time 合成 closer**：后端在 load 时合成 `tool/result` / `step/end` / `turn/end {interrupted}`；没有"继续中断的 turn"的部分恢复。一个已持久但无结果的调用**无法证明**它的外部副作用是否完成，恢复只记录未知结果而不自动重试。
- **checkpoint 不是 exactly-once**：策略持久记录的是执行意图；有副作用的工具应把 `exec.callId` 当幂等键透传。流式 `assistant/chunk` 没有 per-chunk checkpoint，硬崩可能丢掉当前内存批次或未落盘的写。
- **JSONL 后端**：只加载「配置的编码 + 当前 `SESSION_FORMAT_VERSION` (v0)」，pre-release 格式**无迁移**；旧的扁平文件布局不再加载；压缩文件不能直接按行读；每个 session 只允许一个 live writer。
- **SQLite 后端**：schema 17 是过渡设计，不保证 schema 稳定性或迁移；`DatabaseSync` 与 Zstandard 调用**阻塞 JS 线程**，busy wait 也阻塞事件循环；外部 SQL 读者必须理解物理 tag（packed 行的 `type` 是 `text-chunks` / `reasoning-chunks` / `tool-call-chunks`，**不是** `SessionEventMap` 成员）。
- **投影**：每个 tail page 携带**全部** client-visible（带 `wire`）key，host-only 单元不出现，无 per-key 退订；unit 表是**进程级**的，所以 key 存在与否**不是** per-session 的能力信号——客户端必须读 VALUE 而不是把 key 缺失当成功能缺失。registry cell 只在内存，重启靠 fold 重建（挂 `session-projection-cache` 才能从持久行 seed）。
- **`session-stats` 只在 web-app bundle 里挂载**，其他装配没有 `sessionStats` key，消费者退回窗口级计数。
- **标题**：没有"取消固定"（unpin 回自动标题）、没有搜索、没有列表索引；registry 刻意只接受一个实现。`first-prompt` provider 对 fork 从不自动跑；`all-prompts` 输入溢出时保留旧标题、不做摘要的摘要。
- **遥测**：**best-effort 交付**——游标标记的是"已交接"，不是"已送达"；reload 窗口内被拆掉的 session 无法重新 adopt；崩溃时后端队列里的东西丢失。`FEEDBACK_ONLY` 在反馈前不留任何遥测侧副本，崩在反馈前就什么都不上传；on-demand 脱敏用的是**当时挂载的策略**下的**当前值**。
- **OTel 后端**：`@opentelemetry/sdk-logs` 仍来自上游 experimental 树，SDK API 变动只落在这个包；认证、TLS、限流等真实 OTLP 部署行为一律跟随上游 SDK。

## 陷阱

- **`session-telemetry-otel` 的 README 配置示例里包名是错的**：它写 `name: '@deepseek-ai/dsh-session-sessionTelemetry-otel'`，但真实 npm 名是 `@deepseek-ai/dsh-session-telemetry-otel`（见 `package.json` 与 `packages/bundle/base/cordis.patch.yml`）。照抄那段 YAML 会加载失败。`@deepseek-ai/dsh-session-sessionTelemetry-otel` 只是 OTel **instrumentation scope** 的名字（另一个是 `…/ops`）。
- **加载后端不等于开了 checkpoint 策略**：`session-persistence-*` 与 `session-checkpoint-policy` 是**两个插件**，只加后端是合法的，但崩溃会丢掉还在批处理窗口或未完成写里的事件。第一方持久化 app 会显式挂两个。
- **遥测默认关**：`session-telemetry-otel` 的 `mode` 默认 `DISABLED`，shipped bundle 里是"挂载但关闭"，靠 `DSH_TELEMETRY_MODE` 显式选 `FULL` / `FEEDBACK_ONLY`。
- **投影 cache 是 fold 捷径不是权威**：行可能陈旧（`seq` 精确表明有多旧）但绝不会错。`ver` 与 live unit 的 `stateVersion` 不匹配就**丢弃而不迁移**。日志领先、cache 跟随——live checkpoint 先把缓冲事件刷持久，cache 行才落地。
- **记录绑定的是日志生命周期而非 id**：每条 cache 记录存了它 fold 自的 header 身份（`createdAt`、`cwd`），读取时验证，避免"删了再建同 id"或"持久存储被换掉"而 seed 出幻影值。
- **`agentPreset` 在磁盘上是必需的持久字段**：恢复不同的组合会回放模型已无法执行的历史。`delegationDepth` 同样必需，顶层 session 为 `0`，缺失或非法直接拒绝整个日志。
- **cwd 目录名是有损的**：分隔符替换与截断是故意的，归一化后相同的 cwd 会共用同一个 project 目录；session id 仍然区分出不同的 session 目录。
- **标题事件是 log-only 且不开 turn**：provider 的迟到完成直接通过 `Session` 追加一条独立事件；`rename()` 用 `user` source 追加，会**钉住**该 session——之后的用户消息不再触发自动改名，显式 `refresh` 才是解钉。
- **模型可见 ⟺ 已记录**：`session-title-llm` 在派发前追加一条 log-only 的 `session/title-llm-request` 事件（含 provider id、源 seq、路由、system prompt、消息列表、输出上限），且该 envelope 带 `purpose: 'session-title'`，DeepSeek adapter 会据此关掉 thinking。

## 2026-08-22 agent 审核增量（PersistenceBackend 真实形状与版本双轨）

- **`PersistenceBackend` 是读写混合面，不只是写侧四原语**（`session-persistence/src/coordinator.ts` 约 L127-215）：写侧只有 `appendBatch`（原子 materialize+append）、`commitRepair`（截断+合成 closer，允许非原子）与可选 `close`；**`flush` 不在 backend 接口上**——它是 `PersistenceCoordinator` 私有方法（约 L1326），由 `session/flush` 事件驱动 write-behind drain，backend 感知不到。读侧还有 `loadStored` / `readStoredRevision` / 可选 `loadStoredFrom?`（seek-capable suffix read；SQLite 实现使 `readFrom` 只读后缀，顺序介质 JSONL 省略并由 coordinator 全量读后跳过）/ `list` / `locate?`。引用"writer 允许集"类清单时须覆盖读侧职责。
- **`supportsRawArtifacts` / `readRaw` 属于 `SessionPersistence` 服务契约**（`session-persistence/src/index.ts` 约 L102/L119），**不属于** `PersistenceBackend`；且 `readRaw` **不是流式**——返回完整 `SessionRawArtifact.content` 字符串；watermark 续读原语是 `readFrom`（fromSeq suffix read，约 L203-221）。
- **外部并发写者的既定语义是"延迟收敛"而非拒绝**：coordinator 注释 "Revision retries converge once the durable log remains unchanged for one read/check round trip; continuous external writers may delay completion"——多写者会拖慢收敛，不会报错。
- **版本机制是双轨的**：`SESSION_FORMAT_VERSION=0` 层面 no migration；但读取路径对 legacy 事件形状（steering/message、旧 turn/end envelope、pre-identity message）做 deterministic in-place 升级，并**显式拒绝** request/header-delta、mode/set、fallback reason 三种 v0 遗留形状（coordinator.ts `assertSupportedEvents` / `migrateLegacy*`）。
- **SQLite backend 细节**：`tornMarker` 类型是 number（`SqliteStore implements PersistenceBackend<number>`）；schema 校验含 `application_id` 与 before-mutation 复检（schema.ts 约 L265-274 "schema changed before mutation"）；`user_version=0` 的库视为**全新库初始化**而非拒绝。rc.8→rc.2 之间 `SCHEMA_VERSION` 仍为 17、`DEFAULT_MAX_RETRIES` 仍为 5，无新断裂；schema 15→17 的实质是物理 chunk 行压缩（上游笔记：105-session 语料 709.57 MB → 75.01 MB，−89.4%，打包行上限 1,024 事件 / 1 MiB），"不迁移"是 pre-release 政策下的明确否决决策，SQLite 后端仍为 opt-in、官方默认组合继续用 JSONL。
- **checkpoint 失败是 fail-closed 的**："Checkpoint failures are fail-closed at the model and tool side-effect boundaries: the downstream adapter or tool body is not invoked"（`session-checkpoint-policy/src/index.ts` 约 L58-59）——flush 不完成则 LLM 请求与 tool 副作用都不发起。宿主在 checkpoint 边界上叠自己的授权/派发闸时，这条直接决定时序。

## 2026-08-27 agent 审核增量（cold 列表标题、投影 cache 与标题长度）

适用条件：Host 组合已挂 `sessions`、`sessionPersistence`、`sessionProjections`、`sessionTitle` 与 API Proxy；客户端以 `session.list` 为 reconnect baseline；历史 Session 已持久化但未 attach。结论保持宿主无关：

- **cold list 的标题来自投影 cache，不直接读日志**：attached row 读 live registry snapshot；cold row 只同步调用可选 `ctx.sessionProjectionCache.cachedSnapshot(meta)`。无插件或无可用 row 时整个 `projections` 列缺失，client 把它当“尚无 title”；打开后 registry 才从内存日志 lazy fold。`SessionPersistence.locate()` 只影响 cold blank probe，与标题 cache 无关。
- **rc.2 的 cache 是公开发布包**：`@deepseek-ai/dsh-session-projection-cache@0.1.1-rc.2` 从包根公开 default/named service、`Config`、`cachedSnapshot()`、`write()` 与 `coldSnapshot()`；官方 Web bundle 以 `writeEveryEvents: 200`、`writeIntervalMs: 5000` 挂载。两项都是部署选择且必填，不是库默认。
- **owner 分成两条公共 seam**：Session log 仍归 `SessionPersistence`；cache row 归 `session_projcache` domain，经 `StorageBackend.kv` 持久化。通用 backend 可以同时承载 `workspace` 与 `session_projcache` 两个 unit；若复用，插件必须注册命名 backend，并提供 `storageBackendServiceKey(name)` 生命周期服务。不能用 SessionPersistence 直接替代 StorageBackend。
- **旧日志不会自动扫描，但能用公共 cold seam 回填**：cache init 与 `session.list` 都不枚举旧日志。`coldSnapshot(id)` 在无 row 时走 `readFrom(id, 0)` → registry restore → fail-soft write-back，不 open Session，也不把事件 fold ownership 移给宿主。要让首个 list baseline 完整，需在 `title` unit 注册后、对外开放 list 前，对旧 id 做一次有限并发 warm-up；上游没有 bulk migration API。无 checkpoint 的成本是每 Session 全日志读与全量 projection fold。
- **`coldSnapshot`完全不走presenter/preset**：调用链只有cache rows→`SessionPersistence.readFrom`→`SessionProjectionRegistry.restore`→fail-soft write-back；不调用ApiProxy`presenterScopeFor`、`AgentPresets`或`standingKeyFor`，不创建live Session/Agent，也不发`agent/created`。header.agentPreset甚至不参与cache identity（只用createdAt/cwd）。因此recorded preset valid/missing/broken/mount-failed对该fold无差别；若另行standingKeyFor执行preset code，那是独立步骤。
- **write-back 是 fail-soft**：`coldSnapshot()` 可能在写 cache 失败时仍返回 snapshot并只记 warning。把首屏标题当硬验收时，应在 warm-up 后再用 `cachedSnapshot(meta)` 验证 `title` key 已落盘。旧日志没有 `session/title` 时只会恢复 `title: null`，不会触发模型补标题。
- **标题上限是 UTF-8 bytes，不是 code points**：`normalizeSessionTitle()` 清理控制字符并用 `for...of` 按 code point 安全截断。80 bytes 能容纳任意 20 个最多四字节的 Unicode scalar value，但也能容纳 80 个 ASCII，所以它只解决 byte cap 不抢先截断，不能独立实施“最多 20 code points”。fallback 若也要同等容量，`fallbackMaxBytes` 同设 80；`fallbackMaxBytes <= maxTitleBytes` 是硬约束。
- **输出 token cap 不与字符数线性换算**：官方字段是 `maxOutputTokens`；命中 `max-tokens` 会使 provider 结果失败，不会接受一个截断标题。扩大 code-point 展示上限不构成机械调整 token cap 的依据。`targetCjkCharacters` 只是 prompt 目标，不是校验。既有 `session/title` 事件不会因配置变更被重写或补长。
- **版本内文档冲突**：`client/runtime` 与 `host/apiproxy` README 仍把 cold title 笼统写成必须等 open/resume；同 commit 的 API 类型、实现和测试已支持 cold `session.list` projection cache。源码结论优先，README 只描述“无 cache plugin/row”的降级组合（见 `conflicts.md` C077）。

## 2026-08-30 · out-of-tree SessionEvent 持久化边界

- rc.2 package root/`./types`允许TypeScript declaration merge `SessionEventMap`，`Session.append()`与projection registry也都是公开seam；但`KNOWN_SESSION_EVENT_TYPES`是DSH构建时生成的仓内白名单，源码明确说out-of-repo event registration deferred。
- rc.2 persistence只允许unknown event在envelope带`ignorable:true`时读取。外部plugin可在“同一plugin始终存在”时保留并fold这类事件，但语义上它允许不认识的reader跳过，不能承载exact restore必须理解的composition事实。
- alpha.1进一步移除unknown-event的ignorable例外，一律拒绝不在build白名单的事件。第三方required durable domain需上游登记/注册契约，不能只靠类型合并。
- AgentRegistry direct create/resume有公开unpublished`setup`；rc.2 Web ApiProxy的generic create/resume/fork没有out-of-tree setup resolver注册点。`agent/created`/`agent/session-start`均已越过publication边界。
- **认知状态**：verified_inference（两tag `known-event-types.ts`、persistence coordinator、Agent setup与ApiProxy控制流交叉核对）。

### `agent/created` veto与persistence不是同一rollback边界

- AgentLoop publication先`session/created`、后`agent/created`；同步agent listener throw最终清Agent/Session registries并发paired disposal，但PersistenceCoordinator已可能在session/created启动`initFor`。第一方coordinator对fresh/no-event Session是lazy materialization，dispose前无append的一方test明确“never materialized”；若早期observer追加event，retirement会flush并留下artifact。
- 公开`SessionPersistence.create`只说backend **may** lazy，不是must。第三方eager provider可先落header；cold resume已有artifact也不会因live veto被删。rc.2无generic persistence delete，所以只能把同步guard写成live registry rollback，不能无条件写磁盘零痕迹。

## 2026-08-31 · `snapshotJsonValue`的single-read深快照边界

- `@deepseek-ai/dsh-session`根正式导出`snapshotJsonValue(value)`。它以iterative walk生成完全detached的当前realm plain object/array；共享alias按出现位置各自复制，null-prototype与foreign-realm intrinsic plain containers可接受，`__proto__`被安全写成own data key。
- 返回`undefined`是无诊断的“非lossless JSON”sentinel：undefined/function/symbol/bigint、NaN/Infinity/-0、任一invalid child、sparse/decorated array、hidden/symbol own key、exotic/custom-prototype container与cycle都拒绝。getter/Proxy trap抛错则**传播异常**，不转成undefined。
- getter不被一律拒绝：每个property/array slot只读一次，验证与复制共用该值。一方test让getter第一次返回合法JSON、第二次会返回exotic，snapshot仍成功且只读一次。因此准确语义是消除validate→copy双读漂移，不是拒绝stateful getter；第一次读到的值就是snapshot事实。
- snapshot本身未freeze；要建立immutable local ownership，需在校验后另调`deepFreeze`。但从snapshot返回时remote original graph已完全断开，原图后续深层mutation不会影响snapshot。
- **认知状态**：verified_inference（固定rc.2正式session root JS/.d.ts、`src/json.ts`与`tests/json.spec.ts`；未跑hostile Proxy新fixture）。

## 去哪深入（文件路由）

| 问题 | 去这里 |
|---|---|
| 组内四个子域的包/ctx key 映射 | `packages/session/README.md` |
| 持久化服务方法逐条契约（含 `load` vs `inspect` 的差异、错误类型） | `packages/session/session-persistence/README.md`、`src/index.ts` |
| 磁盘布局、packed chunk 行、zstd、project 目录 | `packages/session/session-persistence-jsonl/README.md` |
| SQLite schema 17、压缩阈值、varint 溯源编码、读写路径 | `packages/session/session-persistence-sqlite/README.md` |
| checkpoint 打在哪三个语义点 | `packages/session/session-checkpoint-policy/README.md` |
| 投影单元定义（state vs `wire` client view）、`stateOf()`、变更 feed、`snapshot()` 一致切面 | `packages/session/session-projection/src/index.ts` |
| 投影 cache 的写策略与失效规则 | `packages/session/session-projection-cache/README.md` |
| `sessionStats` 各字段的 fold 语义 | `packages/session/session-stats/README.md` |
| 标题服务 API、fallback 规则、provider 生命周期 | `packages/session/session-title/README.md`、`src/index.ts` |
| 共享 LLM 标题实现的路由与失败契约、必填配置表 | `packages/session/session-title-llm/README.md` |
| 遥测 sink 契约、capture points、sharing 披露 | `packages/session/session-telemetry/README.md`、`src/{index,coordinator}.ts` |
| OTel 三模式、exporter/processor 透传配置、匿名 `user.id` | `packages/session/session-telemetry-otel/README.md` |
| 子系统参考 | `docs/subsystems/{persistence,session-projection,session-title,session-telemetry,session,session-reference}.md` |
| 设计决策 | `.agents/notes/implemented/architecture/{2026-06-11-event-sourced-sessions,2026-06-14-session-persistence,2026-07-19-zstandard-jsonl-session-logs,2026-07-24-project-session-directories,2026-08-05-session-preparation,2026-08-08-bounded-session-persistence-write-batching,2026-08-10-session-log-version-mechanism}.md` |
