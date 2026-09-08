---
title: rc.1 多 provider、BYOK 与 Cloud adapter 接缝
description: 区分多路由选择、手工协议与凭据组合、产品 Cloud 身份和账务责任。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/llm/llm/src/index.ts#LlmAdapter
  - packages/llm/llm/src/types.ts#GenerateOptions
  - packages/llm/llm/tests/service.spec.ts
  - packages/llm/llm-pi-ai/package.json
  - packages/llm/llm-pi-ai/src/index.ts#apply
  - packages/llm/llm-pi-ai/src/config.ts#PiAiProviderProfile
  - packages/llm/llm-pi-ai/src/provider.ts#supportedProtocols
  - packages/llm/llm-pi-ai/src/catalog.ts#resolveRouteModels
  - packages/llm/llm-pi-ai/tests/dynamic-config.spec.ts
  - packages/credentials/credentials/src/index.ts#CredentialProvider
  - packages/api/session-controller/src/catalog.ts#buildModelCatalog
  - packages/api/session-controller/src/commands.ts
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/topics/rc1-initial-model-selection]]"
  - "[[wiki/packages/credentials]]"
---

条件：TypeScript Host 插件 · 完整 CLI/Profile · 同 Host 多 Session/route · Host 凭据服务 · 官方 AgentLoop/Session 不替换 · BYOK 与独立 Cloud route · 部署由宿主管理 · 固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。

## 可确认的组合能力

- `ctx.llm.registerAdapter(providers,adapter)` 可注册一 adapter 的多 route，也可让多个 adapter 各占不同 route；重复 route 报 DUPLICATE_ADAPTER，初始空列表拒绝。registration 随 fiber 释放，可 replace routes。
- `LlmAdapter.providerInfo/listModels/resolveModel` 提供目录/元数据，stream 提供输出。官方 Session 目录逐 provider 汇总，空 model list 不生成非空 selector group；配置目录登记不能替代实际 adapter 注册。
- Session 用公开 selectModel 选一个 provider/model。标准 GenerateOptions 是单一 pair；多 route 可配/可选不等于同 step 自动并行调用多个 provider，也不等于 step 与物理请求一一对应。
- `@deepseek-ai/dsh-llm-pi-ai@0.1.2-rc.1` 的 Config.providers 是按 route key 索引的字典。手工 route 支持的 api 精确为 `openai-completions`、`openai-responses`、`anthropic-messages`；不能从 catalog provider 的额外协议推出任意手工协议都可配。
- 自定义 profile 用 baseURL、api、apiKeyEnv、models 等公开字段。未知 route 需给协议/endpoint/非空模型目录；models 可手工 `{id}`，非空且同 route 不重复。能力默认值不是 endpoint 实测证明。

## Host 凭据与 Cloud 边界

- `@deepseek-ai/dsh-credentials@0.1.2-rc.1` 公开 abstract CredentialProvider，可由 Host 插件替换默认 provider；需遵守 refs 与 records 完整合同，不只是实现 resolve。
- 显式 apiKeyEnv 是 CredentialRef，pi-ai 每 operation 调 `ctx.credentials.resolve(ref)`；无 service 时才直接读 launch environment。指定 ref 缺失 fail loud，不回退无关 ambient key；省略 ref 才走 provider-native auth。
- BYOK 路径不要求产品 Cloud 账号，目标模型服务自己的凭据仍必需。Host provider replacement 不自动证明密钥没经过 Client 输入/wire；凭据解析、UI、传输须分层判断。
- 独立 Cloud adapter 可作为普通已激活 Host plugin 注册无冲突 route，提供 catalog 并被 Session 选择，不需替换 AgentLoop/Session。新增包是否可热插拔未获一般保证；installed 也不等于 activated。
- LlmAdapter 仍须遵守 attributionHeaders、消息/stream/取消等该版合同。DSH 提供这些模型接缝、生命周期与日志，不自动交付产品账号、余额、服务端账本或计费事务。
- GenerateOptions 没有产品 account/billing transaction/idempotency-key 标准字段；SessionId、requestId、usage、logical step 均不自动构成扣费幂等或远端结算回执。自有 adapter 的认证/transport 映射是其实现语义。

## 证据与验证范围

[多 route 注册](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm/src/index.ts#L373)、[官方目录](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/catalog.ts#L15)、[三协议](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm-pi-ai/src/provider.ts#L28)、[手工模型](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm-pi-ai/src/catalog.ts#L829)、[credential resolver](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm-pi-ai/src/index.ts#L169)。

固定 tag 已核对。精确 llm-pi-ai/credentials rc.1 tarball 在内存读取 manifest 与 root declarations，确认公开 profile/Config/supportedProtocols/CredentialProvider；未安装。pi-ai 正式依赖范围为 `^0.84.2`，实际递归版本需结合 lock，不能仅凭 DSH 根版本冻结。

上游注册冲突、settings 新增 route、key rotation 测试仅阅读断言。未进行 endpoint、账号/计费或完整 UI E2E；本页只确认公开接缝与条件性后果。
