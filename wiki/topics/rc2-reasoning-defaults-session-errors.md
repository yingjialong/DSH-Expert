---
title: rc.2 reasoning 默认值与 Session 错误分层
description: 核心默认注入、pi-ai Anthropic wire 与历史打开及调用失败的边界。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/llm/llm/src/index.ts#resolveCallWithInfo
  - packages/llm/llm-pi-ai/src/adapter.ts#reasoningInfo
  - packages/llm/llm-pi-ai/src/catalog.ts#resolveModelReasoning
  - packages/llm/llm-deepseek/src/adapter.ts
  - packages/llm/llm-deepseek/src/serialize.ts#resolveThinking
  - packages/host/apiproxy/src/api-proxy.ts#buildModelCatalog
  - packages/api/remotes/src/agent-lookup.ts#createApiRemoteAgentResolver
  - packages/client/runtime/src/client/sessions/session.ts#doOpen
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-09
updated: 2026-09-09
asked_by: agent
---

# rc.2 reasoning 默认值与 Session 错误分层

条件：标准 DSH Host/Session driver · 多 provider exact model · pi-ai custom route 或内置 DeepSeek · 文件系统/工具复用不限 · 无真实模型请求 · 部署实际 endpoint 未验证 · DSH 0.1.1-rc.2 固定本页 SHA、pi-ai0.82.1。静态核验，verified_inference。

## 默认值经过核心解析

LlmRuntime.resolveCallWithInfo 使用 requested ?? reasoning.defaultEffort，校验后把 effective effort 写入调用配置；不能只读 adapter serializer 推断标准默认。

pi-ai reasoningEfforts 是 exact model 支持与 wire mapping，非默认声明。profile.reasoning 缺席时 defaultEffort 缺席；设了且 exact model 支持才公布，不支持时描述省略、实际调用拒绝 UNSUPPORTED_REASONING_EFFORT。无 reasoning capability 则不公布整个控制。

内置 DeepSeek 未配默认时公布 high，故标准普通调用会注入 high，发送 thinking enabled + reasoning_effort high。thinking disabled 公布 off-only/default off；off wire 为 thinking disabled、不发 reasoning_effort off。显式 low/high/max 为 enabled+相应 effort，disabled 部署拒绝非 off。session-title 特例 disabled。绕过 LlmRuntime 直接 adapter.stream 的双省略 serializer 可不写这些字段，不等同标准 Session 行为。

llm.models 复制 adapter 元数据，不探测远端默认。公开 resolveModelInfo/prepareCall 可报告本地声明/effective 配置；没有通用远端默认探测。显式 route.reasoning 固定默认属于配置改变，不是只读发现。

## pi-ai0.82.1 Anthropic

DSH 将 off 与双省略都变为 common options 无 reasoning；formatter 决定最终 wire。手工 reasoningEfforts 未声明的 level 被置 map[level]=null；声明 off:null 则 map.off 缺席，二者语义不同。

- model.reasoning=false：不写 thinking。
- reasoning omitted/off，model.reasoning=true：thinkingEnabled=false；map.off!==null 时发 disabled，否则省略 thinking。
- forceAdaptiveThinking:true + enabled level：thinking adaptive/display summarized，加 output_config.effort（映射字符串优先；fallback 对非 low/medium/high 回 high）。max 档要依据确切 max mapping。
- 非 adaptive + enabled level：thinking enabled/budget_tokens/display summarized，不写 output_config.effort。默认预算 minimal1024、low2048、medium8192、high16384，可配置；max/xhigh 归 high budget，仍受输出与上下文 cap 限制。

所以 stock 配置可表达 enabled/disabled，但没有在此 Anthropic serializer 下独立组合 enabled + effort high/max 的开关。目标若拒绝 adaptive，forceAdaptiveThinking 不会自动兼容；budget 与目标服务 effort 的对应本次未知，不作认证。

其他协议不能套该结论：普通 Responses 在无 reasoning 且 map.off!==null 时发 reasoning.effort=map.off??none；普通 OpenAI completions 只在 off 映射为字符串时写 reasoning_effort，特殊 thinkingFormat 分支另论。defaultEffort 缺席不能统一解释为 off 或远端原始默认。

## 历史 open 与真正调用失败

历史读取不 resume Agent，也不做模型调用。缺工具 presenter、参数解析/presenter throw 可保留 raw event、缺 view，并退 generic 展示；不是 UNKNOWN_TOOL 打开失败。

| 边界 | 公开结果/观察位置 |
| --- | --- |
| session.history | 身份缺失 session-not-found，其他 history unavailable 为 internal；Client snapshot.openState/error、openError；ChatView显示 code/message |
| Client Session.open | 可将错误存入 snapshot 后完成 Promise；不能只看 fulfilled；后续 gap repair 也有独立边界 |
| agentFor cold resume | session-not-found、subagent agent-busy；其他 setup/resume 失败 internal，message含 resume failed；非统一透传 preset 错误 |
| prompt admission | 无 adapter route 为 model-unavailable；新图片不支持 attachment-error/details.reason MODEL_DOES_NOT_SUPPORT_IMAGES；返回结果及 promptError |
| provider/step | UNSUPPORTED_REASONING_EFFORT、UNSUPPORTED_CONTENT 等；agent/error、lastAgentError、turn/end reason.error；非 LlmError 可为 UNKNOWN |
| 实际工具执行 | 无 definition 为 UNKNOWN_TOOL；tool/result isError/error.info；不是历史展示缺 presenter |

session.models.currentAvailable 判断 route 服务存在性，不能从 catalog groups 成员关系代推。pi AUTH/TRANSPORT 是后置文本分类的一部分，不证明网络 I/O 已发生。官方 turn-error 节点读 turn/end failure，与 openError 分开。

本 SHA 的 llm.models/历史实现位于 dsh-host-apiproxy 与 dsh-client-runtime，未发现 dsh-api-llm package manifest；不套用晚版拆包。精确 pi 包为 dsh-llm-pi-ai，依赖范围 ^0.82.1，本次上游锁与咨询提供的实际依赖均为0.82.1。

## 证据

- [核心默认解析](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm/src/index.ts#L780)、[pi metadata](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm-pi-ai/src/adapter.ts#L131)、[DeepSeek metadata](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm-deepseek/src/adapter.ts#L390)。
- [history/错误](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L2154)、[Client doOpen](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/client/runtime/src/client/sessions/session.ts#L617)。
- [pi-ai0.82.1 tarball](https://registry.npmjs.org/@earendil-works/pi-ai/-/pi-ai-0.82.1.tgz)：dist/api/anthropic-messages.js:593–634、753–780，dist/api/simple-options.js:30–51，openai-completions.js与openai-responses.js对应format分支；内存只读，未安装。

镜像根经核实 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness`，历史源码按 git show 固定 SHA 读取。未访问外部 endpoint 或调用方项目，不认证外部服务兼容性。
