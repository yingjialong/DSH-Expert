---
title: packages/web — web 能力族（search + fetch）
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/web/README.md
  - packages/web/AGENTS.md
  - packages/web/web/README.md
  - packages/web/web/src/index.ts
  - packages/web/web/src/types.ts
  - packages/web/tool-web/README.md
  - packages/web/web-fetch-http/README.md
  - packages/web/web-search-exa/README.md
  - packages/web/web-search-perplexity/README.md
  - packages/web/web-search-deepseek/README.md
  - docs/subsystems/web.md
  - .agents/notes/implemented/architecture/2026-06-24-web-capability-seam.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

provider-中立的 web search 与 fetch：**一条 seam 上两种操作、每种可有多个 provider**，让面向模型的 schema 在后端切换时保持不变。

## 稳定性

`Product — stable API`（`packages/README.md` 表格中 `web/` 行原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `web/web/` | `@deepseek-ai/dsh-web` | Service Definition：`WebRuntime`（`ctx.web`）、两套 provider 注册表、选择策略、请求/结果词汇、`WebError` 分类 |
| `web/web-search-exa/` | `@deepseek-ai/dsh-web-search-exa` | Search provider，id `exa` |
| `web/web-search-perplexity/` | `@deepseek-ai/dsh-web-search-perplexity` | Search provider，id `perplexity` |
| `web/web-search-deepseek/` | `@deepseek-ai/dsh-web-search-deepseek` | Search provider，id `deepseek-official`（走 Messages 模型轮次） |
| `web/web-fetch-http/` | `@deepseek-ai/dsh-web-fetch-http` | Fetch provider，id `http`（匿名公网 HTTP(S)） |
| `web/tool-web/` | `@deepseek-ai/dsh-tool-web` | Consumer：`web_search` / `web_fetch` 工具 schema |

（provider id 常量：`EXA_PROVIDER_ID='exa'`、`PERPLEXITY_PROVIDER_ID='perplexity'`、`DEEPSEEK_PROVIDER_ID='deepseek-official'`、`LOCAL_FETCH_PROVIDER_ID='http'` [T1: packages/web/web-*/src/provider.ts]）

## 三件套结构

- **Service Definition**：`@deepseek-ai/dsh-web`。`WebRuntime extends Service` 默认导出，挂 `ctx.web`；类型在 `src/types.ts`（`WebSearchRequest/Result/Source`、`WebFetchRequest/Result`、`WebFetchBody`、`WebSearchProvider`、`WebFetchProvider`、`WebError extends HarnessError`）[T1: packages/web/web/src/types.ts]。
- **Service Provider**：四个 provider 包，各自 `ctx.web.registerSearchProvider(...)` / `registerFetchProvider(...)`。
- **Consumer**：`@deepseek-ai/dsh-tool-web` —— **唯一**拥有面向模型的工具名、描述、prompt 指引、JSON schema 和渲染的包。

search 与 fetch 不共享请求 schema、也不共享业务逻辑，但**刻意做成一条 seam**：一个 provider 选择策略 owner、一套 abort/error 词汇、一个「本 harness 如何上网」的产品配置面。

## 扩展点

- **接一个新搜索/抓取后端**：依赖 **`@deepseek-ai/dsh-web`**（绝不依赖某个具体 provider 包），实现 `WebSearchProvider` / `WebFetchProvider` 并注册。注册返回 disposer，**随调用 fiber 释放**；同一 capability kind 内 id 重复抛 `WebError` 码 `WEB_DUPLICATE_PROVIDER`。
- **provider 的 `available()` 契约**：必须是**便宜的本地检查**（凭据是否存在、配置能否解析），**绝不能发起网络调用**。`dsh-tool-web` 从不调用它——工具只走 `ctx.web.search()/fetch()` 并路由抛出的 code，好让 provider 选择只有一个 owner。
- **选择策略**（永不依赖注册顺序、配置顺序或 HMR 顺序，且在**执行时**解析）：配置 `searchProvider` / `fetchProvider`（或等价环境变量 `$DSH_WEB_SEARCH_PROVIDER` / `$DSH_WEB_FETCH_PROVIDER` 喂同一字段）显式指定 id；没指定时，**恰好一个可用 provider** 才自动选中。分支码：`WEB_PROVIDER_CONFIGURED_MISSING`（配了但没注册）、`WEB_PROVIDER_CONFIGURED_UNAVAILABLE`（注册了但 unavailable）、`WEB_PROVIDER_UNAVAILABLE`（无可用）、`WEB_PROVIDER_AMBIGUOUS`（多个可用但没配 id）。
- **换一套模型可见表达**：另写 Consumer 依赖 `ctx.web`。`tool-web` 的 Config 含 `searchMaxResults`（默认 `8`）、`searchMaxQueries`（默认 `4`）、`fetchMaxOutputChars`（默认 `200000`）。
- **`WebFetchBody` 是 CLOSED 判别联合**（`html` | `text`），由 Definition 包拥有——消费者用 `switch` 穷尽，新增一种会让它们**编译失败直到处理**。加 `pdf` 分支是跨三个 web 包的编译强制变更。

## Known Limitations

- **`web-fetch-http` 没有 SSRF / 内网防护**——不拦私网、loopback、link-local、multicast，不做 DNS-resolve-then-validate，不做逐跳复验。README 原文：这个 provider 是一个 **SSRF primitive，在能触达敏感内网的部署里 must not be enabled**。
- Definition 层**没有观测面**：没有 provider 变更事件、没有能力状态查询；可用性只能靠执行 `search()`/`fetch()` 再路由抛出的 `WebError` 码来观察，无 provider 时是笼统的 `WEB_PROVIDER_UNAVAILABLE`，不枚举每个 provider 的原因。
- `WebSearchRequest` 只有 `query` + `maxResults`；recency、域名过滤、地域提示、搜索深度这些 provider-中立控制项被推迟到 Exa 和 Perplexity 都能诚实兑现为止。同理 Exa 只暴露 `searchType`/`numResults`/`highlightsPerResult`，Perplexity 只暴露 `model`/`maxTokens`/`searchRecency`。
- **没有 batch 级原生搜索计数器**：`searchMaxQueries` 只约束 `ctx.web.search` 的**调用次数**，一次调用内 provider 可能做多次原生搜索（模型型 provider 配了 `maxUses` 时最多 `searchMaxQueries × maxUses` 次）；`searchMaxResults` 只限返回给调用方的合并来源数。
- **没有 web 专属权限策略**：两个工具都不请求 `ctx.approval`；需要确认的部署必须自己加 `tools/pre-execute` 策略，本包不定义持久化的 URL/域名授权。
- `web_fetch` 只解码文本内容（html/xhtml 与 `text/*` 加 JSON/XML 家族）；缺 `Content-Type` 或二进制类型抛 `WEB_UNSUPPORTED_CONTENT_TYPE`，可提取文本的 PDF 是被点名的 deferred work。**charset 只从 `Content-Type` 头取**（默认 UTF-8），HTML 的 `<meta charset>` 被忽略，声明了但不认识的 charset 直接抛而不回退。
- `web-search-deepseek`：一次搜索**等于一整个 Messages 模型轮次**（延迟 + 生成 token）；`available()` 是同步契约，查不了异步凭据存储，所以选中一个无 key 的 provider 会在搜索时以 `WEB_PROVIDER_CREDENTIAL_MISSING` 失败，而 `web_search` schema 仍保持注册。
- Exa：**没有非空 highlight 的结果会被整条丢弃**，因此返回来源数可能少于请求数。Perplexity：缺结构化 `search_results[]` 时的 citation 回退来源**只有 URL**，工具只能渲染裸主机名标签。
- 两个搜索 provider 的 **abort 分类基于错误形状**：只有 name 为 `AbortError` 的 `DOMException` 映射到 `WEB_ABORTED`；带自定义 reason 的中止（例如 `dsh-timeout` 的 `TimeoutReason`）会呈现为 `WEB_PROVIDER_ERROR`。
- `maxResults` 对 DeepSeek / Perplexity 是**事后由 seam 截断**执行的——超发的来源仍然消耗 token 和延迟。

## 陷阱

- **带凭据的 provider 请求必须拒绝重定向**（`packages/web/AGENTS.md` 组级规则）：HTTP client 要配置成在跟随任何重定向**之前**失败，回归测试须证明重定向目标未被联系、且每个带凭据的 provider 都启用了该策略。
- `fetch()` 的**非 2xx 是结果不是抛错**；只有「无法安全取得或表示资源」才抛 `WebError`。写调用方时别把 404 当异常。
- provider 注册的是**capability，不是 tool**；把工具名/描述写进 provider 包违反 seam 纪律。
- Exa 的「结果数不足」和 seam 的「truncated」是两回事：前者是 provider 丢弃，后者是 `maxResults` 截断置位 `truncated`。

## 去哪深入（文件路由）

| 想查 | 去 |
|---|---|
| search/fetch 请求与结果、可用性、`WebError` 子系统参考 | `docs/subsystems/web.md` |
| 选择策略表与 Service API 三行表 | `packages/web/web/README.md` §Selection / §Service API |
| 完整类型与 `WebError` code 分类 | `packages/web/web/src/types.ts` |
| 工具 Config 全表与 HTML→markdown 转换细节 | `packages/web/tool-web/README.md` |
| 生成的 `web_search` / `web_fetch` schema | `docs/tool-catalog.md` 锚点 `#deepseek-aidsh-tool-web` |
| 为什么 search 与 fetch 合成一条 seam、SSRF 为何推迟 | `.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.md` |
