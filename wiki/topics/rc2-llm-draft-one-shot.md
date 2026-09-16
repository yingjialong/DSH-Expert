---
title: rc.2 无 Session 草稿 LLM 单次调用
description: 公开 PiAiAdapter、隔离 Cordis 组合及超时和 discovery 边界。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-16
updated: 2026-09-16
asked_by: agent
anchors:
  - packages/llm/llm-pi-ai/src/index.ts#apply
  - packages/llm/llm-pi-ai/src/adapter.ts#PiAiAdapter
  - packages/llm/llm-pi-ai/src/config.ts#ResolvedPiAiProviderProfile
  - packages/llm/llm-pi-ai/src/discovery.ts
  - packages/llm/llm/src/index.ts#LlmRuntime
  - packages/llm/llm/src/types.ts#GenerateOptions
---

条件：Node/TS · 独立 Cordis root · 无 Session/AgentLoop · 文本且无工具 · 临时模型草稿与显式凭据 resolver · 不写活跃 Settings/catalog · rc.2 固定本页 SHA。只读核验，未执行模型请求或完整 PoC。

`dsh-llm-pi-ai` root 公开 PiAiAdapter/PiAiAdapterOptions；构造不需要 Context，但 options 必须带 profiles thunk、resolveApiKey、auth（pi-ai CredentialStore/AuthContext）。profiles 返回已解析的 ResolvedPiAiProviderProfile，含 piProvider/configuredMaxTokens 等，不能直接传原始草稿。内部 resolveProfiles 没有从 root 导出；测试 adapterOf 私有导入它，不是公开调用示例。

公开原始 config 路线是独立 Context 挂 LlmRuntime 与 llm-pi-ai plugin，并仅在该 root 使用 ctx.llm.stream。无 Settings seam 时 entry config 生效；临时 CredentialProvider 可按显式 apiKeyEnv 供给草稿 key，不必改活跃 catalog。独立 root 不等于继承活跃服务的 child scope。支持 openai-completions/openai-responses/anthropic-messages；文本请求无需 attachment 服务。包依赖仍须满足。

GenerateOptions 必填 provider/model/messages，tools/sessionId 可省略；createUserMessage 可构造手写消息。stream 返回 AsyncIterable，须消费并检查 terminal finish；LlmRuntime 将 adapter 异常变 failure chunk，不能只以 iterator 结束或没有 throw 判断成功。

pi-ai adapter 固定 maxRetries:0；独立组合不挂 retry middleware/AgentLoop 才能据此排除本层重试。timeoutMs 传 SDK，streamIdleTimeoutMs 仅 outstanding read 超时。signal 与 consumer abort 合并传 watchdog/SDK；提前结束消费会 abort 并 await iterator.return。resolveApiKey 在 watchdog 建立前 await，签名无 signal，因此不是任意 resolver/cleanup 的全链 hard deadline。

公开 ctx.llm.discoverModels('llm-pi-ai',draft) 与生成测试不同：已知 provider 用安装 catalog；未知 route 的 OpenAI 两协议可 GET /models，Anthropic 不支持该 listing。它不写设置，不证明 reasoning 能力或生成成功。此前“无 remote model discovery”的无条件表述过宽。

证据根 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness`；[公开 exports/apply](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/llm/llm-pi-ai/src/index.ts)、[adapter](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/llm/llm-pi-ai/src/adapter.ts)、[discovery](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/llm/llm-pi-ai/src/discovery.ts)、[一方公开 plugin harness](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/llm/llm-pi-ai/tests/adapter.spec.ts) 均按固定 SHA 读取。
