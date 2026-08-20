---
title: packages/session-query — Session 检索能力族
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/session-query/README.md
  - packages/session-query/session-query/README.md
  - packages/session-query/session-query/src/index.ts
  - packages/session-query/session-query/src/config.ts
  - packages/session-query/session-query-sqlite/README.md
  - packages/session-query/tool-session-query/README.md
  - packages/session-query/session-log-export/README.md
  - docs/subsystems/session-query.md
  - packages/bundle/base/cordis.patch.yml
  - packages/bundle/web-app/cordis.patch.yml
  - examples/acp-agent/session-query.cordis.yml
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

对 live 与持久 session 日志做**已授权的检索**：逻辑语料、有界读取、血缘（lineage）追踪、事件关系、语义过滤，以及 SQLite 全文搜索。与 compaction 无关，也和 `session/` 组的持久化内部实现解耦。

## 稳定性

`Product — stable API`（[T1: packages/README.md]：`session-query/` = "Session retrieval family: logical corpus, bounded reads, lineage, event relationships, semantic filtering, and SQLite full-text search"）。

## 包清单

| 包名 | npm 名 | 职责 / ctx key |
|---|---|---|
| `session-query` | `@deepseek-ai/dsh-session-query` | 定义 trusted 读、关系查询、搜索操作；`ctx.sessionQuery` |
| `session-query-sqlite` | `@deepseek-ai/dsh-session-query-sqlite` | 用 SQLite FTS5 实现全文搜索；同样注册到 `ctx.sessionQuery` |
| `tool-session-query` | `@deepseek-ai/dsh-tool-session-query` | 把 workspace 授权过的查询暴露给模型；registers on `ctx.tools` |
| `session-log-export` | `@deepseek-ai/dsh-session-log-export` | Web `/export` 命令、共享浏览器下载状态、结果 Modal（走 Host 的 ZIP 端点）；`ctx.sessionLogDownload` |

## 三件套结构

- **Service Definition**：`@deepseek-ai/dsh-session-query`。特别之处是它**不是纯抽象**——`SessionQueryEngine extends Service` 已经把精确读、过滤、追踪**实现好了**，只留 `searchSessions` / `searchEvents` 两个抽象方法给后端。它没有 provider coordinator、没有 fallback 实现、也**没有独立的具体插件**。
- **Service Provider**：`@deepseek-ai/dsh-session-query-sqlite`（`SqliteSessionQueryEngine`），第一个也是目前唯一的实现，继承上面那批已实现的方法，只负责全文观测、对账、排序、cursor generation 与查询执行。
- **Consumer**：`@deepseek-ai/dsh-tool-session-query`（model-facing，五个工具 `session_search` / `session_event_search` / `session_trace` / `session_event_trace` / `session_event_read`）与 `@deepseek-ai/dsh-session-log-export`（human-facing，`/export`）。

## 扩展点

- **写一个新的搜索后端**：继承 `SessionQueryEngine`（[T1: packages/session-query/session-query/src/index.ts#SessionQueryEngine]），只需实现 `searchSessions(request, exec?)` 与 `searchEvents(request, exec?)`；分页续接必须返回自己拥有的 branded `SessionSearchCursor`，snippet 不得暴露 provider 专属数值分数。
- **写一个新的检索 Consumer（工具/UI）**：依赖 `@deepseek-ai/dsh-session-query`，**不要**依赖 `-sqlite`。
- **共享的文档投影**：`extractSessionEventText()` 与 `buildSessionEventSearchDocuments()` 定义第一方文档投影——reasoning 块、结构边界、stream chunk、request header、以及本 build 不认识的 declaration-merged 变体都**不产生文档**。
- **错误码**：`SessionQueryError.code` 是封闭 union，字面量定义在 `packages/session-query/session-query/src/config.ts`（如 `SESSION_QUERY_SOURCE_CONFLICT`、`SESSION_QUERY_PERSISTENCE_FAILED`、`SESSION_QUERY_CORRUPT_SESSION`、`SESSION_QUERY_INVALID_FILTER`、`SESSION_QUERY_INVALID_LINEAGE`、`SESSION_QUERY_SEARCH_DISABLED`、`SESSION_QUERY_TOOL_FAILED`）。
- **要限制输出体积**：本组**不做**字节/字符截断，也不 import spill 后端；需要有界内联输出就挂 `@deepseek-ai/dsh-spill-policy`（执行后替换渲染文本但保留完整结果）。

## Known Limitations

- **没有调用方授权**（`session-query` 与 `-sqlite` 都写了）：这是 trusted 的 context 级基础设施，授权必须由 model tool 或 UI 自己做。
- **Service Definition 里没有注册表也没有 model-facing tool**：extractor / search-provider 注册表、沿引用源事件递归遍历，都还不存在。
- **同步查询执行**：`DatabaseSync` 在 MATCH 期间阻塞 JS 线程，已经开始跑的语句无法中断。
- **是 token 召回而非任意子串**：`unicode61` tokenizer 不匹配大 token 内部的子串；要字面扫描请用 `filterEvents()`。
- **派生索引单 owner**：一个进程里只能有一个 service 拥有一个索引路径，外部写者与跨进程共享**不支持**（generation 与 TEMP shadow 状态是连接私有的）。
- **模型工具侧**：搜索最多返回部署上限并让模型自己收窄查询，**没有续接 token**；workspace 身份是保守的 `cwd` 精确字符串相等，**符号链接等价路径不共享授权**。
- **导出侧**：下载端点要求后端提供 per-Session 的 raw artifact——shipped 的 JSONL 后端支持明文与 zstd，**SQLite 导出未包含**；这是浏览器下载而非 Host 侧写文件；preflight 只报 ZIP 开始流式传输之前发现的失败。

## 陷阱

- **shipped 组合里全文搜索是关掉的**（跨文档综合）：`packages/bundle/base/cordis.patch.yml` 与 `packages/bundle/web-app/cordis.patch.yml` 都以 `path: ':memory:'` + `openAt: never` 挂载 `session-query-sqlite`。也就是说 `ctx.sessionQuery` 是可用的（精确读、标题、血缘追踪照常，session export 与 subagent-fork 的 Workspace 继承依赖它们），但 `searchSessions` / `searchEvents` 会以 `SESSION_QUERY_SEARCH_DISABLED` 失败，node:sqlite 从不被 import。要开内容搜索得在后续 patch 层把 `openAt` 覆盖成 `first-search` 或 `startup`，通常再配一个持久 `path`。
- **`tool-session-query` 默认不挂**：shipped host composition 不挂它；仓内唯一的挂载示例是 `examples/acp-agent/session-query.cordis.yml`。
- **`openAt` 三态的副作用不只是时机**：`startup` 会在 service 激活时 import `node:sqlite` 并开句柄（索引非法则在发布前失败）；`first-search` 让服务 ACTIVE 但推迟 import——这是为了让 Node 22 的 experimental warning 不出现在启动输出里（**不是消除**，第一次搜索时仍会出现）。
- **绝对不要把 `path` 指向 session-persistence 的数据库**：这是派生索引专用库，README 明确警告。POSIX 上目录/库以 owner-only（`0700`/`0600`，再受 umask 影响）创建。
- **FTS5 语法被当作数据**：查询是必填、trim 过、空白归一化的**字面短语**；引号、`OR`、`NEAR`、`*` 都不是可执行 MATCH 语法。
- **有谓词预算**：为了把 MATCH 保持在受支持的外层谓词位置，跨 session 请求最多编译 14 个组合过滤谓词，单 session 请求最多 13 个（固定的目标 session 谓词占一格）；每个范围端点算一个谓词。超预算或超过 SQLite 的 32,766 绑定上限会在 prepare 之前以 `SESSION_QUERY_INVALID_FILTER` 失败。
- **text 过滤子句故意独立于 FTS provider**：调用方文本被转义成 Unicode、大小写不敏感的正则，每段空白匹配一个或多个空白字符——这是字面语义文本扫描，不是全文查询。
- **cursor 会失效**：cursor 绑定归一化后的请求与 service 实例，相关 generation 变了就失败。单 session 的 cursor 能挺过无关 session 的变化，跨 session 的不能。
- **live 优先**：同一个 id 同时有 live 与 persisted 时产出一条记录，live 赢，但 `live` / `persisted` 两个字段都报告可用性；两边不可变 header 冲突时以 `SESSION_QUERY_SOURCE_CONFLICT` 失败。
- **查询从不触发崩溃修复**：session query 绝不调用后端那个会修复崩溃的 `load()`。
- **模型工具的自引用防护**：`session_search` 永远排除调用方自己的 session；当前 session 的 `session_event_search` 会在触发它的那个 step 之前就停住，防止活动中的 assistant 输出和已记录的 tool call 匹配到自己。两个搜索工具**只能与同级工具串行执行**（内部消耗 generation-bound cursor），另外三个精确 trace/read 工具才 opt-in 并行。
- **血缘输出会打码**：未授权的祖先/后代边界被替换成不含任何隐藏 session id 的 marker；缺失的 id 与跨 workspace 的猜测行为完全一致。

## 去哪深入（文件路由）

| 问题 | 去这里 |
|---|---|
| 组内四包 / ctx key 映射 | `packages/session-query/README.md` |
| 每个读方法的逐条契约、过滤器语义、持久层可选性与错误分类 | `packages/session-query/session-query/README.md` |
| `SessionQueryEngine` 抽象类、错误码字面量 | `packages/session-query/session-query/src/index.ts`、`src/config.ts` |
| 语料合并、追踪、文档抽取、cursor 的实现 | `packages/session-query/session-query/src/{corpus,tracing,documents,extraction,filters,cursor,sources}.ts` |
| FTS5 搜索契约、谓词预算、索引生命周期、完整配置表 | `packages/session-query/session-query-sqlite/README.md` |
| 五个模型工具的授权规则、配置、system prompt 原文 | `packages/session-query/tool-session-query/README.md` |
| `/export` 命令契约、Web 挂载点、Modal 行为 | `packages/session-query/session-log-export/README.md` |
| 逻辑记录、有界读、trace、filter、结果页（子系统参考） | `docs/subsystems/session-query.md` |
| 实际的默认挂载与 `openAt: never` 注释 | `packages/bundle/base/cordis.patch.yml`、`packages/bundle/web-app/cordis.patch.yml` |
| 唯一的模型工具挂载示例 | `examples/acp-agent/session-query.cordis.yml` |
| 设计决策 | `.agents/notes/implemented/feature/{2026-07-10-sqlite-session-query-provider,2026-07-13-session-query-tracing,2026-07-24-model-facing-session-query-tools,2026-08-02-session-search-not-shipped-default,2026-08-10-web-session-log-export}.md`、`.agents/notes/implemented/architecture/2026-08-13-session-content-search-opt-in.md` |
