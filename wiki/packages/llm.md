---
title: packages/llm — LLM capability family
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/llm/README.md
  - packages/llm/llm/README.md
  - packages/llm/llm/src/index.ts
  - packages/llm/llm/src/adapter-failure.ts
  - packages/llm/llm/tests/service.spec.ts
  - packages/llm/llm-deepseek/README.md
  - packages/llm/llm-pi-ai/README.md
  - packages/llm/llm-pi-ai/src/config.ts
  - packages/llm/llm-pi-ai/src/catalog.ts
  - packages/llm/llm-pi-ai/src/provider.ts
  - packages/llm/llm-pi-ai/src/auth.ts
  - packages/llm/llm-pi-ai/src/index.ts
  - packages/llm/llm-pi-ai/src/adapter.ts
  - packages/llm/llm-pi-ai/src/stream.ts
  - packages/llm/llm-pi-ai/tests/catalog.spec.ts
  - packages/llm/llm-pi-ai/tests/adapter.spec.ts
  - packages/llm/llm-pi-ai/tests/dynamic-config.spec.ts
  - packages/host/apiproxy/src/api/sessions.ts
  - packages/host/apiproxy/src/api-proxy.ts
  - packages/host/apiproxy/tests/api-proxy-models.spec.ts
  - packages/core/agent/src/model-selection.ts
  - packages/core/agent/tests/model-selection.spec.ts
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent-loop/tests/contract-regressions.spec.ts
  - packages/client/ui-conversation/src/client/conversation-nodes/turn-error.ts
  - packages/client/ui-model-selection/src/client/ModelSelect.tsx
  - packages/client/ui-model-selection/tests/model-select.client.spec.tsx
  - packages/llm/llm-retry/README.md
  - packages/llm/llm-retry/src/index.ts
  - packages/llm/llm-retry/src/types.ts
  - packages/llm/llm/src/retry-policy.ts
  - packages/llm/llm/tests/retry-policy.spec.ts
  - packages/compaction/compaction-basic/src/index.ts
  - packages/llm/token-meter/README.md
  - docs/subsystems/llm-streaming.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-31
asked_by: agent
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
- `llm-pi-ai`：settings 能加/覆盖/删除 **user layer** 路由，但删除 composition base 路由只会重新继承 base；完全动态的路由应让插件裸挂载、全部放进 `llm-pi-ai.providers` user layer。`headers` 是普通设置，凭据不会被 redactor 看见；路由 catalog 永不自刷新；一个手工路由只能一种 wire protocol（`openai-completions` / `openai-responses` / `anthropic-messages`）；手工 route 必须有 endpoint、协议和非空 model list。`baseURL` 没有 scheme/host/私网防线。图片只从 durable attachment 生成受 `maxRequestImageBytes`、`requestImagePixelBudget`、`requestImageMaxBytes` 约束的 inline request version；无凭据路由能否工作取决于 provider-native auth；**`GenerateOptions.stop` 不受支持**。

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
14. **`off` 不是核心通用能力**：reasoning effort 是 exact route/model 的 adapter-owned opaque id。精确模型未公开 `reasoning` 时，显式传任何 effort（包括 `off`）都会在 provider I/O 前以 `UNSUPPORTED_REASONING_EFFORT` 被拒；`session.selectModel` 再把它映射为 `model-unavailable`。省略字段才表示把默认所有权交还 adapter/provider，而不是显式关闭推理。

## 2026-08-27 审核增量：pi-ai 动态多 route 与 Session 选择

- **配置 ownership**：`Config` 是 `{ providers?: Record<route, PiAiProviderProfile> }`，字典键就是 provider route。`apiKeyEnv`、`api`、`baseURL`、`headers` 与 transport/retry/image policy 都归 route；`models[]` 只归 model identity/capability。不同 key、endpoint 或 wire protocol 必须拆 route；仅共享全部 route 事实、只差 model id 时才合并进同一路由。[T1: `packages/llm/llm-pi-ai/src/config.ts`、`src/catalog.ts`]
- **协议与命名**：手工 route 的公开 runtime 协议精确为 `openai-completions`、`openai-responses`、`anthropic-messages`；“OpenAI Chat”对应第一项。core route 仅要求非空且全局唯一，model id 仅要求非空且 route 内唯一；若还要兼容 credential record，内部 route 应取 lowercase-kebab 稳定 id。显示名只写 `displayName`，不能用可编辑标签当 route identity。[T1: `src/provider.ts`、`src/config.ts`、`src/auth.ts`]
- **凭据两条路**：显式静态 key 配 `apiKeyEnv: CredentialRef`，每次 stream operation 通过 `ctx.credentials.resolve(ref)` 重读，不跨 operation 缓存 secret；pi-ai 原生登录/OAuth 才使用 `llm-pi-ai/<route>` record。设置了 ref 但解析缺失会 `MISSING_CREDENTIAL`，不回退 ambient key。[T1: `src/index.ts#resolveApiKey`、`src/auth.ts#recordKeyFor`、`tests/dynamic-config.spec.ts`]
- **热更新**：插件通过 `SettingsScope.watch()` 与 adapter registration `replace()` 原子更新 route set，当前 operation 保留旧 snapshot、下一 operation 读新配置；无需因 pi-ai topology 重启 Host。动态 route 应让 Cordis 裸挂载 `llm-pi-ai`，用户层删除用 `settings.mutate([{op:'unset', path:['providers', route]}])`；composition base 路由不能由 user layer 真正删除。[T1: `src/index.ts`、`tests/dynamic-config.spec.ts`]
- **选择生命周期**：`session.selectModel` 传 `{provider: route, model: id, reasoningEffort?}`，保存的是 live Session 的 next-step selection；点击本身不写 Session log，只有后续模型请求消费选择时才写 `request/header`。route 删除/改名后旧 pair 不自动迁移，`session.models().current` 可仍显示旧值但 `routable:false`，下一 prompt 返回 `model-unavailable`；route 还在而 model 被删时则可能晚到 adapter 的 `UNKNOWN_MODEL`。[T1: `packages/host/apiproxy/tests/api-proxy-models.spec.ts`]
- **reasoning effort ownership**：手工 pi-ai model 无同 id 的 installed-catalog base 且省略 `reasoningEfforts` 时，不公开 exact-model `reasoning`；DSH 刻意不把 pi-ai 的伪 `off` 暴露给 selector，因为它只是省略 common `reasoning` option，不能保证关闭 provider 默认推理。纯模型切换应省略 effort；只有 exact model 的 `reasoning.efforts` 明确包含用户选择时才显式提交。省略会清除旧模型继承的 effort，但最终 wire 仍由目标 adapter/profile 决定：在 pi-ai 0.82.1 的 `openai-responses` 中，exact model 一旦显式发布 `off`，显式 Off 与 request/route 双省略都会进入 `thinkingLevelMap.off` fallback，分别发声明值或 `none`，并不保留远端 provider 的 omitted-field default；只有完全不发布 `off`（map 值为 `null`）才省略 `reasoning`。route `reasoning` 若受 exact model 支持，则成为 `defaultEffort` 并在双省略时优先物化。`groups` 是建议目录：条目缺席只能说明能力未知/未公布，不能证明不可路由。[T1: `packages/llm/llm-pi-ai/src/{catalog,adapter}.ts`、`packages/llm/llm/src/index.ts`、`packages/core/agent/src/model-selection.ts`、`packages/host/apiproxy/src/api-proxy.ts`；pi-ai 0.82.1 `dist/api/openai-responses.js#streamSimple,#buildParams`]
- **安全边界**：`baseURL` 是 Host 网络能力而非受保护 URL，宿主设置 owner 必须自做 SSRF/origin policy；secret 不得进入 `headers` 或 URL。provider `errorMessage` 没有 secret/PII redaction 保证，并可进入 durable turn end，普通 UI/日志不能把原文当可信内容。[T1: `src/stream.ts`、`src/discovery.ts`、`src/adapter.ts`]
- **认知状态**：verified_inference（tag 源码、正式 npm JS/d.ts 与官方测试源码交叉核对；未重新执行测试）。

## 2026-08-30 审核增量：pi-ai 凭据与终端错误码边界

- **出网前顺序**：pi-ai adapter 先校验 stop、route/model、exact reasoning，再解析命名 `apiKeyEnv`，最后才进入 pi-ai `streamSimple()`。不支持的 effort 是 `UNSUPPORTED_REASONING_EFFORT`，未知 route/model 分别是 `NO_ADAPTER` / `UNKNOWN_MODEL`；命名 ref 解析缺失或空字符串是 `MISSING_CREDENTIAL`，非空但 trim 后为空或不能安全放进 HTTP header 的值是 `INVALID_CREDENTIAL`。这些内建校验不会主动产生 `AUTH` / `TRANSPORT`。[T1: `packages/llm/llm-pi-ai/src/{adapter,index}.ts`、`packages/llm/llm/src/{index,api-key}.ts`、`tests/adapter.spec.ts`]
- **`AUTH` / `TRANSPORT` 是文本分类，不是 I/O provenance**：pi-ai terminal error message 含独立 `401` / `403` 时归 `AUTH`；含 stream truncation、fetch/network/connection/socket/ECONN 等词时归 `TRANSPORT`；其余通常是 `PI_AI_ERROR`。formatter / `buildParams` 也在同一 pi-ai catch 内，因此错误文案若意外命中正则，即使尚未发 HTTP 也可能得到这两个 code；反过来也不能只凭 code 证明 endpoint 已收到请求。[T1: `packages/llm/llm-pi-ai/src/stream.ts`；pi-ai 0.82.1 `dist/api/{openai-completions,openai-responses}.js`]
- **合法 reasoning 不跨域改码**：exact model 已发布 `medium` 且 route 默认也为 `medium` 时，reasoning 只负责能力校验与 wire 物化，本身没有 `AUTH` / `TRANSPORT` 分支；前者属于凭据/认证文本，后者属于传输或命中文本分类。无效显式/default effort 会在 credential 与 provider I/O 前失败。
- **Session 投影保留最终 failure code**：Llm runtime 把 `LlmError.failure.code` 放进 terminal finish；AgentLoop 可按 recovery listener 重试，最终未恢复失败原样写入 durable `turn/end.reason.error`；conversation projection 再复制到 `TurnErrorNode.code`。因此 UI 看到的是最终 attempt 的 code，不保证是首个失败，也不携带是否出网的阶段证据。[T1: `packages/llm/llm/src/adapter-failure.ts`、`packages/core/agent-loop/src/agent.ts`、`packages/core/session/src/types.ts`、`packages/client/ui-conversation/src/client/conversation-nodes/turn-error.ts`]
- **认知状态**：verified_inference（固定 tag、正式 npm JS/d.ts 与一方测试源码交叉核对；未重跑测试，未调用真实 endpoint）。

## 2026-08-30 · rc.2 model-request retry identity边界

- 公共`GenerateOptions`有可选`sessionId`，但没有turn/step/attempt/retryId/requestId/idempotency key。`LlmFailure.requestId`是provider失败后回报的事实，不是DSH预先下发的identity。
- AgentLoop在同一turn/step内循环重建request并拥有retry；`dsh-llm-retry`在首个失败后mint并耐久记录`RetryId`，同open step/policy chain后续复用，但它不进入`GenerateOptions`或`LlmAdapter.stream()`参数。
- Persistence在Host crash后关闭open turn为`interrupted`，不续跑同一物理model request。固定版公共参数因此不足以让adapter安全派生跨transport retry、Host crash与Session resume稳定的provider idempotency key；`sessionId+messages hash`不是官方identity。
- 一方tests覆盖同turn/step retry、failed-attempt chunk丢弃与durable retry budget/event；没有测试或类型承诺adapter拿到跨attempt稳定id或provider exactly-once。
- **认知状态**：verified_inference（固定tag LLM/AgentLoop/llm-retry类型、控制流与一方tests；未调用provider）。

## 2026-08-31 · rc.2 zero-retry与request-error组合边界

- `RetryPolicyConfig`是正式根导出，normal `maxRetries`允许`0`；provider profile可逐route配置，自定义`LlmAdapter`可覆写`providerRetryPolicy(provider)`。省略则是5次normal retry，不是零重试。
- `@deepseek-ai/dsh-llm-retry`只是可选的`agent/request-error` consumer，自身不接policy配置。`maxRetries:0`使它在首个失败上直接delegate、不写`llm/retry`，但**不是全局never-retry flag**：compaction或其他listener仍可返回`{kind:'retry'}`，AgentLoop就会再次进入adapter。要做同step at-most-one，必须同时审计完整listener composition与adapter/middleware自己的调用行为。
- pi-ai明确把SDK `maxRetries`钉为`0`，所以一次DSH adapter invocation不会被SDK暗中倍增；direct `ctx.llm.stream()`也不消费AgentLoop retry plugin。
- T1 `AgentLoop.step()`在同一`turn/step`的`while`内retry，一方test也只观察到一条`step/start`。同tag `llm-retry/README.md`声称fresh numbered turn，与实现/测试冲突，见`conflicts.md` C080。
- **认知状态**：verified_inference（固定tag root types、executor/compaction控制流与一方tests；没有另跑`maxRetries:0`最小实测）。

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
