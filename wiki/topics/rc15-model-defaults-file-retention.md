---
title: 1.5-rc.2 Session 模型默认值与文件 receipt 保留
description: 核验模型投影、persona、pi-ai 0.85.1 协议默认与附件日志生命周期。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-21
updated: 2026-09-21
asked_by: agent
anchors:
  - packages/api/session-controller/src/model-selection-projection.ts
  - packages/api/session-controller/src/agent.ts
  - packages/api/session-controller/src/commands.ts
  - packages/core/agent-default-model/src/index.ts
  - packages/core/system-prompt/src/index.ts
  - packages/core/session/src/index.ts
  - packages/llm/llm-pi-ai/src/adapter.ts
  - packages/llm/llm/src/retry-policy.ts
  - packages/session-query/session-log-export/src/archive.ts
  - packages/client/file-upload/src/index.ts
related:
  - "[[wiki/topics/rc15-controlled-profile-file-chain]]"
---

固定DSH0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203，pi-ai0.85.1及其Anthropic SDK0.123.0。6个DSH包与两第三方包共8个tarball sha512/公开面核验。未发模型请求或运行恢复/retention实验；完整回传成功后沉淀，L2 / verified_inference。

## Session选择与default

Host modelSelection state={lastUsed,pending}；Client projections.faceOf('modelSelection')={lastUsed,next}，next=pending??lastUsed，官方UI在next为空时用catalog.default。model/selection写pending；request/header更新lastUsed并精确匹配后清pending。

selectModel先append model/selection并改本Agent选择，再saveSelection到agent-default-model Settings；保存默认失败只warn、不回滚Session。此处提交内存事件不等于flush耐久。

每Agent/Session选择隔离，但default是共享配置：A保存默认成功后，没有自己pending/header的B或新Session可见新default。重启靠持久Session事件与Settings；缺失flush不保证恢复。实际选择优先pending→历史header→default；adapterDefaults.reasoningEffort=true的历史值不恢复成用户显式选择。

AgentDefaultModel Config仅provider/model，Settings schema另有reasoningEffort。请求effort优先于profile.reasoning，exact model不支持则拒绝；省略不等于off或统一high。

## persona与重建

SystemPrompt公开personaPrefix/personaSuffix（默认空）、section以及getSectionOrder(PromptSectionOrderName)。prefix order0、suffix10200，identity可在-1000；SECTION_ORDERS不是root公开值。

标准AgentLoop以assemble/render→prepared route→真实system/message及用户消息→request/header→deriveMessages构建输入。公开Session.fromRestore/eventAt/snapshotEvents/requestHeader/deriveMessages可还原目标格式逻辑surface；不是HTTP字节重放。file/image投影、replay-state、credential、headers、provider缓存另由runtime/adapter处理。手动emit或Client draft不证明真实producer。

## pi-ai与transitive SDK

pi-ai0.85.1精确依赖Anthropic SDK0.123.0。pi-ai anthropic-messages调用client.beta.messages.create；SDK resources/beta/messages/messages.mjs硬编码post('/v1/messages?beta=true')。query由SDK beta resource生成，不是DSH拼接，不证明思考开启或等同anthropic-beta header。

DSH off映射为省略common reasoning，但各formatter不同：

- 普通OpenAI Completions分支：显式effort映射reasoning_effort；省略时仅string map.off写入，provider compat另有特例。
- Responses reasoning模型：缺effort且非github-copilot、map.off!==null时写off或none；省略不必保持provider omitted默认。
- Anthropic streamSimple缺reasoning令thinkingEnabled=false，普通reasoning模型map.off!==null写disabled；显式有reasoning可走adaptive effort或budget。supportsMidConvoEffort另有adaptive/high分支。
- DSH自定义efforts未声明项置null；声明off:null使map.off缺席，与pi formatter的null行为不同。不能用“off send nothing”注释覆盖最终wire。

DSH provider.retryPolicy normal/maxRetries0关闭dsh-llm-retry额外尝试；adapter固定给pi-ai maxRetries0，三协议也给底层SDK requestOptions maxRetries0。此路径不做这些层的自动额外retry，但一次用户prompt可多step/request-error重调，discovery/auth/工具/redirect等不属于同一生成attempt；未实测全Host请求次数。

## file refs与receipt

session-log-export内部collectEventAttachmentRefs/attachmentRefsInArtifact没有root导出；公开Session事件读取与FileBlock/FileAttachmentRef字段可消费。内部export扫data.content、data.message.content、data.inserted消息、embedded stream block-end及递归content；files按hash+name去重，不只hash。

整个append-only日志引用不等于当前active surface或GC根规范。已取消/替换事件仍可在历史；内部helper不是公开GC API。

FileUploads保存成功并复核live Agent后，将receipt登记在WeakMap<Session,...>。bindPrompt绑定requestId，未commit guard回滚绑定；user/message.source.rpcId观察或retirePrompt清匹配绑定，Session disposed清整张map。

不单独监听agent/disposed；不能仅据Agent事件认定映射已清。Context/service卸载撤销生命周期并重新挂载新Map，不意味着删除durable文件；旧JS引用不是合法service存活证明。

未绑定receipt无公开TTL/按receiptId delete/release，可留至Session生命周期结束；无跨重启receipt恢复。AttachmentStore无delete/refcount/GC，receipt失效不等于对象回收。

## 证据

固定源码：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/model-selection-projection.ts`
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/llm/llm-pi-ai/src/adapter.ts`
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/file-upload/src/index.ts`

第三方发布物：[pi-ai0.85.1](https://registry.npmjs.org/@earendil-works%2Fpi-ai/0.85.1)、[Anthropic SDK0.123.0](https://registry.npmjs.org/@anthropic-ai%2Fsdk/0.123.0)。未做真实三协议wire计数、A/B重启或跨进程retention验收。
