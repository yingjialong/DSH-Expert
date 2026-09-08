---
title: rc.1 与 pi-ai 0.84.4 三协议边界
description: 本地模型目录、输出token调整、图片和replay降级以及凭据与HTTP拦截分层。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/llm/llm-pi-ai/src/adapter.ts
  - packages/llm/llm-pi-ai/src/catalog.ts
  - packages/llm/llm-pi-ai/src/config.ts
  - packages/llm/llm-pi-ai/src/context.ts
  - packages/llm/llm-pi-ai/src/replay.ts
  - packages/llm/llm-pi-ai/src/stream.ts
  - packages/llm/llm-pi-ai/src/discovery.ts
  - packages/llm/llm-pi-ai/src/index.ts
  - packages/llm/llm-pi-ai/tests/adapter.spec.ts
  - packages/llm/llm/src/index.ts
  - packages/llm/llm/src/content.ts
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/topics/rc1-multi-provider-byok]]"
  - "[[wiki/topics/rc1-initial-model-selection]]"
---

条件：官方DSH0.1.2-rc.1 adapter/Controller/Loop · pi-ai明确0.84.4 · Completions/Responses/Anthropic API-key三协议 · 本地目录配置 · synthetic凭据前提 · 不调用HTTP · 固定DSH `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。不涵盖其他wire/OAuth/Cloud。

## 本地模型与发现

- nonempty models 替换服务集合，各entry仍可继承同id内建字段；缺省/空models回内建列表，custom无模型则拒绝。modelOverrides仅支持内建route/已知id且不能与nonempty models同用。
- route api/baseURL优先于model/provider内建。模型capacity为entry→内建→route default，缺失最终用context262144、maxTokens32768、input text；这些不是远端能力证明。
- listModels/resolveModel/prepareCall走本地snapshot，pi-ai exact model必须在已配置集合中。独立discovery才会访问/models：内建catalog直接返回，custom仅Completions/Responses支持listing，Anthropic不支持此发现但可手工models。

## cap/reasoning

- DSH call显式maxTokens优先；adapter只把显式entry.maxTokens作为defaultMaxTokens。内建capability/route default不自动作为DSH记录的request cap。
- pi-ai0.84.4 buildBaseOptions 用显式值或model.maxTokens，再按contextWindow−估算输入tokens−4096 clamp，available至少1；不是统一超限reject。
- Completions写compat选择的max_tokens/max_completion_tokens；Responses再提升至至少16；Anthropic budget thinking可加thinking预算并再次clamp。因此wire cap不保证等于原始DSH值。
- 显式reasoningEffort优先profile.reasoning，exact能力不支持则拒绝。off在common options变成省略reasoning，不是所有endpoint显式关思考的证明。
- 硬拒有非法profile/缺custom配置/不支持compat、unknown model、unsupported effort、stop非undefined、named key缺失/非法、部分图片形态或服务缺失。具体AgentOptions/default cap校验不能被说成所有任意direct GenerateOptions都已全量检查。

## messages、图片与replay

- DSH Context映射systemPrompt、assistant text/reasoning/toolCall、user/toolResult；三协议formatter负责实际messages/input/tool-use等wire。返回tool arguments可由pi-ai已解析值再JSON.stringify，不是原始HTTP参数字节。
- official prompt先将上传保存为durable ref；request转换再readImageRequest，发生在pi adapter解析credential之后、streamSimple/provider HTTP之前。ImageAttachmentAccess只是host→执行world只读路径描述，不是审批票据。
- LlmRuntime可先把text-only模型的图片变为文字占位；adapter自身对仍存在的unsupported图片/缺AttachmentStore拒绝。非user history图片在预算offload前拒绝。
- 总图片budget采用估算与实际bytes两阶段offload，优先替换旧图为文字提示；单图编码目标不可达时可用最小版本。不是整请求一律拒绝，也不重写Session历史。
- 不可用replayState降为provider-neutral历史并warn；core去除不同adapter拥有的历史replay状态。不能由此假定所有扩展内容无损保留。

## credentials、headers 与拦截覆盖

- named apiKeyEnv每stream经CredentialProvider解析；无service才读launch environment。命名ref缺失不回ambient；无ref才交pi-ai自身auth context/store，并非必然无认证/无ambient读取。
- DSH profile headers去掉大小写不敏感attribution冲突，再添加Harness attribution。pi-ai0.84.4模型headers/defaults在前、options headers在后。auth选择优先不等于最终HTTP Authorization合并保证；最终SDK大小写处理不能自动当DSH统一合同。
- discovery另有draft-key→stored resolver及Bearer/attribution次序，不经过model stream middleware。
- request/header保存DSH config/system/tools，不保存raw API key或HTTP headers；不保证provider错误/自定义日志自动脱敏。模型wire完整body/header不是标准Session快照。
- Controller/Agent middleware覆盖各自入口，pre-step晚于assembly；llm/stream覆盖经过该Runtime的stream，可不委派。它不覆盖独立discovery或任意Host网络。Credentials拒绝也不是无ref/header-auth路径的统一gate。
- 官方Config没有DSH通用fetch/onPayload/HTTP审批hook；SDK支持这些option不等于DSH adapter已透传。prepareCall绑定一次profile/Models与registration，非所有retry永久冻结。pi-ai0.84.4三HTTP实现底层SDK retries=0，自有retry helper默认0，但DSH/其他middleware仍可重试。
- 取消/watchdog/iterator return不证明已发HTTP或server效果撤销。

证据：[本地解析](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm-pi-ai/src/catalog.ts#L801)、[snapshot/request](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm-pi-ai/src/adapter.ts#L288)、[图片转换](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm-pi-ai/src/context.ts#L241)、[replay](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm-pi-ai/src/replay.ts#L226)、[独立发现](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm-pi-ai/src/discovery.ts#L200)。

pi-ai0.84.4正式tarball在内存读取dist/api/simple-options.js、三formatter和dist/utils/provider-retry.js；这些是包内逻辑路径。DSH root PiAiAdapter/options/Config/profile types等复用本系列正式面核验，内部context/replay/discovery/helpers不作私有导入建议。仅阅读snapshot/header/stop/image/取消测试断言，未运行三wire或项目E2E。
