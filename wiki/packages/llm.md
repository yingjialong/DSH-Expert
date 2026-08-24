---
title: packages/llm — LLM capability family
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/llm/README.md
  - packages/llm/llm/README.md
  - packages/llm/llm/src/index.ts
  - packages/llm/llm-deepseek/README.md
  - packages/llm/llm-pi-ai/README.md
  - packages/llm/llm-retry/README.md
  - packages/llm/token-meter/README.md
  - docs/subsystems/llm-streaming.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

LLM seam 及其 provider 适配器。组 README 的原话很关键：**`llm` 包同时拥有 Service Definition 和 Consumer 两个角色**——抽象 service、content-block 词汇表、stream-chunk 组装器都在它手里 [T1: packages/llm/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

| 包名 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `llm` | `@deepseek-ai/dsh-llm` | `ctx.llm` | LLM service + 共享流式词汇（Message/ContentBlock/StreamChunk/BlockAssembler/HarnessError） |
| `token-meter` | `@deepseek-ai/dsh-token-meter` | `ctx.tokenMeter` | replay-aware token 计量与三个 session projection |
| `llm-retry` | `@deepseek-ai/dsh-llm-retry` | 监听 `agent/request-error` | provider 作用域的重试策略**执行器** |
| `llm-deepseek` | `@deepseek-ai/dsh-llm-deepseek` | 注册到 `ctx.llm` | 直连 DeepSeek 适配器（raw fetch + `eventsource-parser` SSE），路由名 `deepseek-official` |
| `llm-pi-ai` | `@deepseek-ai/dsh-llm-pi-ai` | 注册到 `ctx.llm` | 基于 `@earendil-works/pi-ai` 的多 provider 适配器，路由由 config 的 `providers` dict 决定 |

## 三件套结构

- **Service Definition**：`llm`（`LlmRuntime extends Service`、抽象基类 `LlmAdapter`）。
- **Service Provider**：`llm-deepseek`、`llm-pi-ai`（两个 adapter 实现同一 seam，走完全不同的内部实现，这是刻意的「twin adapters」设计验证）。
- **Consumer**：`llm` 自己（vocabulary + `BlockAssembler`）、`token-meter`、`llm-retry`。组 README 明说 **retry 与 token 计量是独立的 Consumer，不是 adapter 的一部分**。

**`HarnessError` 基类住在 `dsh-llm` 里**——因为它是所有包都会 import 的叶子包，这样共享同一个错误基类不用新增依赖边。`LlmError`、`ToolArgsError`、`InvariantError` 等都继承它。这是一条跨组的隐藏依赖事实。

## 扩展点

**扩展插件依赖 `@deepseek-ai/dsh-llm` 这个 Service Definition，不要依赖 `llm-deepseek` / `llm-pi-ai`。**

- 加一个 provider → 继承 `LlmAdapter`（唯一必须实现的方法是 `stream()`），调 `ctx.llm.registerAdapter(providers, adapter)`。注册是 all-or-nothing、随 fiber 释放；返回的 disposer 还带 `replace(providers)`（候选集先整体校验，冲突时旧路由继续服务，切换是一个同步段、无可观察空档；`replace([])` 合法，而空的**初始**注册不合法）。
- 可选覆写：`providerRetryPolicy()`（provider 拥有的恢复配置）、`providerInfo()`、异步 `listModels()`、`resolveModel()`（精确身份、容量、输出默认值、可选 reasoning effort、可选 `inputModalities`）、`prepareCall()`（把 exact model 元数据与最终 dispatch 绑定到**同一 adapter generation**——动态 adapter 应覆写它，以免 settings 变更把一代的能力与另一代的 endpoint 拼在一起；默认实现等价于 `resolveModel()` + 直接 `stream()`）。其余默认值是：bounded normal 重试、用路由和 model id 当名字、不广播任何 model、不返回容量/输出默认/reasoning 元数据。
- 拦截/包裹每次流式调用 → `ctx.on()` 挂 `llm/stream` **waterfall** listener（缓存、日志、路由）。但**不要在这里做重试**：发出 chunk 之后再重试没有持久的 attempt 边界，所以官方重试走 `agent/request-error`。
- 配置面扩展 → `registerConfigurableProviders(entries)` 声明「可由配置激活的路由」（registered 或 dormant），`registerModelDiscovery(settingsNs, discover)` 提供「向 endpoint 询问它有哪些 model」的能力。
- 拓扑变更观察 → 监听 payload-free 的 **`llm/adapters-updated`**（在 mutation 之后发出），然后重新读 `listProviders()` / `listModels()` / `listConfigurableProviders()`，**不要轮询**。

## Known Limitations

- `llm`：**本 service 不执行重试、缓存或限流**（注册时存策略，但 `llm/stream` 仍是单次尝试的包装）；`GenerateOptions` 采样参数**只有** `temperature`/`maxTokens`/`stop`（没有 `tool_choice`/`top_p`/penalty）；producer-gated 变体（`prefill`、per-tool `strict`、block `cache` hint、`agent` message-source）**在有生产者之前不会加回来**；`BlockAssembler` 只处理核心 block 种类（plugin 新增的 block 若其流从未被 `block-end` 关闭，`blocks()` 会抛错）；**`APP_IDENTITY.url` 指向一个还不存在的仓库**；`GenerateOptions.sessionId` 是本地声明的 brand（引 dsh-session 的 `SessionId` 会成环）。
- `token-meter`：固定启发式是近似的（**四字符一 token** + 结构开销）；每次测量都克隆当前 surface，读是 O(surface)；provider usage **只在 canonical envelope 完全一致时**才可复用；缺失的 legacy `sourceEventSeqs` 保守处理。
- `llm-retry`：**agent turn 是唯一的重试边界**（直接调 `ctx.llm.stream()` 的消费者仍是单次）；**always 模式会重试永久性失败**（认证、配额、非法请求、协议、不可恢复的 context 错误都会一直重试直到成功/取消/dispose）；有限预算会**叠加**；恢复策略按 waterfall 顺序组合；`llm/retry` 记录的是**调度**而非完成。
- `llm-deepseek`：settings 的 `models` 列表**整体替换**组合层列表；`tool_choice` 未映射；用 raw `fetch` 而非 `@cordisjs/plugin-http`（`TODO(http)`）；plugin 新增的 block 类型被跳过，空 tool 输出以字面量 `(no output)` 过线；图片是**输入-only 的 durable attachment**。
- `llm-pi-ai`：条目最多的一个包（11 条）。重点：图片请求预算 0.1.1 起拆成两键——`maxRequestFilesBytes`（Files API 上传路径）+ `maxInlineRequestImageBytes`（inline 路径），另有 `maxImagesPerRequest` 与一组 Files API offload 旋钮（`imageOffloadByteQuantum` / `inlineImageOffloadByteQuantum` / `imageOffloadCountQuantum` / `filesApiTimeoutMs` / `fileExpiresAfterSeconds` / `fileRefreshMarginSeconds` / `fileQuotaCleanupBatch`）；**仅靠 OAuth 认证的 provider 不被提供**；provider-native 发现**只读进程环境变量**，看不到 harness credential seam；settings 能加/覆盖路由但**不能删除 composition 路由**；分层 merge **对 dict key 没有 delete**；`headers` 里塞的凭据**不会被 redactor 看到**；路由的 catalog **永不自刷新**；**一个路由只能一种 wire protocol**（`supportedProtocols()` 返回且仅返回 `openai-completions` / `openai-responses` / `anthropic-messages`，表序即默认序；Azure/Codex 等不在内是凭据形态表达不了，catalog 路由不受此限）；modality 声明**不被校验**，over-claim 会把被拒图片永久留在 session log 里（恢复手段只有换模型/fork/新 session）；无凭据路由能否工作取决于协议；**`GenerateOptions.stop` 不受支持**。

## 陷阱

1. **两条 DeepSeek 路径的路由名不同**：`llm-deepseek` 拥有 `deepseek-official`，而 pi-ai 的 catalog 名是 `deepseek`。**故意不同**，好让一个 composition 同时挂两条；但对 `deepseek-official` 重复注册仍抛 `LlmError('DUPLICATE_ADAPTER')`。
2. **失败不会跨 stream API 抛出**。`LlmRuntime` 把「最终 adapter 选择、同步 dispatch、iterator 构造、迭代」四处的失败统一归一化成流协议里唯一的终态形式 `finish { kind: 'error' | 'aborted', failure }`。**但** `llm/stream` middleware、嵌套调用、adapter cleanup、下游消费者的错误**仍然是抛出的**——因为那是 plugin/consumer 故障而非 model-request 结果。部分 delta 之后失败可能留下未闭合的 block，消费者应丢弃。
3. **provider/model 元数据是发现面，不是路由白名单**。adapter 可以接受 `listModels()` 里没有的 model id；**消费者不得因为 model 未列出就拒绝请求**。
4. **`prepareCall()` 是一次性的**。它把精确的 adapter 注册和不可变 retry policy 一起捕获成一个可取消的 one-shot 调用，防止 HMR 把 A adapter 的能力结果和 B adapter 的请求拼在一起。**重用 handle 或改它的 call-config 字段会失败**（`INVALID_PREPARED_CALL`）。捕获经 `adapter.prepareCall()` 完成：exact model 元数据与 dispatch 绑定同一 adapter generation，`PreparedLlmCall` 因此多出一个只读 `inputModalities`；直接 `ctx.llm.stream()` 也走同一条 generation 绑定路径（每次调用一次 `adapter.prepareCall()`），不再分别读两次动态连接事实。
5. **`LlmCallConfig` 是 per-conversation 状态，不是可随手调的 per-call 旋钮**。它记进 session log 的 `request/header`；`agent/request` waterfall 提议替换 → `prepareCall()` 校验并物化 adapter 默认值 → loop 记录生效值和「哪些字段来自 adapter 默认」的 marker。**下一次提议会省略被 marker 标记的默认值**，好让换路由时重新解析自己的值；未标记的显式字段则保留。
6. **浏览器侧要从 `@deepseek-ai/dsh-llm/message` 子路径 import**，不要 import 带 service 的包根。同理 `llm-retry` 的事件 payload 在 `@deepseek-ai/dsh-llm-retry/types` 子路径。
7. **Adapter replay state 的保留条件很严**：只有历史 provider 路由与目标 provider 路由**当前由同一个 adapter 实例拥有**时，`LlmRuntime` 才保留它，然后由 adapter 决定能否跨 model/provider 恢复或转换。
8. **`token-meter` 的 occupancy 是近似值且是设计如此**：三个字段是各自 last-wins 的独立记录，**不是**一次请求的原子观测。换模型时新容量会和旧路由的采样配对，直到下一次请求上报 usage。它**不是**任何决策输入——compaction 读的是 `measure()`。
9. **`projectedTokens` 存在的理由**：compaction 通过直接 `ctx.llm.stream()` 汇总、自己不追加 usage，所以光看 `pressureTokens` 会一直报「压缩前」的 prompt 直到整整一个 turn 走完。占用率显示应读 `projectedTokens`。
10. **`contextBreakdown` 三个数不会加总等于 `projectedTokens`**——前者全是固定启发式估算（CJK 文本与 JSON schema 在「四字符一 token」下**严重低估**），后者有 provider 锚点。只能当作近似组成来展示。
11. **`llm-retry` 的 always 模式与「provider 给的 Retry-After」交互反直觉**：超过 `maxDelayMs` 上限的 provider 延迟会让 normal 模式**委派**（delegate），而 always 模式改用自己配置的本地 backoff，**这样它就不会被那条指令终止**。
12. **重试策略配在 adapter 上，不配在 `llm-retry` 上**：`llm-retry` 自己没有 policy config。多 provider 的 `llm-pi-ai` 把 `retryPolicy` 放进每个 provider profile 里。
13. **text-only 模型的图片在 runtime 层被投影，不是被拒绝**：dispatch 前，若 exact model 的 `inputModalities` 已声明且不含 `'image'`、而 messages 含图片（含嵌套 tool-result 里的图片），`LlmRuntime` 会用 `projectImagesForTextModel()` 把图片替换成确定性占位文本（`textOnlyImageText`，带 attachment sha256 digest 前 8 位），只改本次 transient request、不动 durable session history。因此经 `ctx.llm.stream()` 给 text-only 模型发图不会触发 llm-deepseek adapter 自己的 `UNSUPPORTED_CONTENT` 图片门（投影发生在进 adapter 之前）；`inputModalities` 未声明（`undefined`）则不投影、交由 adapter 处理。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| Message/ContentBlock/StreamChunk 协议、adapter 契约 | `docs/subsystems/llm-streaming.md`（权威子系统参考） |
| `ctx.llm` 全部方法逐条语义、错误码、`LlmAdapter` 扩展点 | `packages/llm/llm/README.md`、`packages/llm/llm/src/index.ts` |
| Message 值构造器 / 冻结语义 | `packages/llm/llm/src/message.ts` |
| 图片投影 / 卸载 helpers（`projectImagesForTextModel`、`offloadRequestImagesWithPolicy`、`contentHasImage`） | `packages/llm/llm/src/content.ts` |
| `LlmCallConfig`、`markAgentLoopRequest`、`deepFreeze` | `packages/llm/llm/src/call-config.ts` |
| 强制 App attribution header | `packages/llm/llm/src/attribution.ts`、`.agents/notes/implemented/architecture/2026-06-21-mandatory-app-attribution-headers.md` |
| API key 规范化与 `assertUsableApiKey` | `packages/llm/llm/src/api-key.ts` |
| token 计量契约与三个 projection | `docs/subsystems/token-meter.md`、`packages/llm/token-meter/README.md`、`src/surface-fold.ts` |
| 重试事件、policy key、invariant | `packages/llm/llm-retry/README.md`、`src/index.ts` |
| DeepSeek 配置项、thinking / reasoningEffort、图片处理 | `packages/llm/llm-deepseek/README.md` |
| pi-ai 的 profile dict、catalog 覆盖、compat 字段 | `packages/llm/llm-pi-ai/README.md` |
| 为何要两个 adapter | `.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md` |
| 为何失败统一成终态 finish | `.agents/notes/implemented/architecture/2026-07-29-terminal-llm-stream-failures.md` |
| replay token meter / routed model context 的设计 | `.agents/notes/implemented/architecture/2026-07-15-replay-token-meter-service.md`、`2026-07-20-routed-model-context-and-compaction-policy.md` |
