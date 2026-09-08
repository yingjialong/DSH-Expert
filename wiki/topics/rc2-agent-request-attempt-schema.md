---
title: rc.2 agent/request 的 attempt 边界、occurrence 身份与工具 schema 子集
description: 固定 rc.2 的请求控制流、工具 schema 子集与输入校验和审批顺序。
type: reference
updated: 2026-09-08
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/core/agent-loop/src/agent.ts#buildRequest
  - packages/core/agent/src/runtime-types.ts
  - packages/core/agent/src/dispatch.ts#agentEvents
  - packages/core/system-prompt/src/index.ts#assemble
  - packages/llm/llm/src/index.ts#prepareCall
  - packages/llm/llm-retry/src/index.ts
  - packages/core/agent-loop/tests/request-error.spec.ts
  - packages/core/agent-loop/tests/interception.spec.ts
  - vendor/cordis/src/events.ts#waterfall
  - packages/core/session/src/types.ts#SessionEvent
  - packages/llm/llm/src/message.ts#UserMessage
  - packages/core/tools/src/index.ts#ToolDefinition
  - packages/core/tools/src/json-schema.ts#assertObjectJsonSchema
  - packages/core/tools/tests/json-schema.spec.ts
  - packages/core/tools/src/schema.ts#defineTool
  - packages/core/tools/tests/scoped.spec.ts
  - packages/core/tools/tests/tools.spec.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-06
asked_by: agent
---

## 条件与版本

TS/JS plugin · 标准 AgentLoop/LLM pipeline · 多 Agent 与同 step retry · 文件系统形态无关 · standing tools 静态 · 不调用真实模型 · live Host · 固定 `dsh-v0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`。

远端 tag 与本地对象一致，所有证据按完整 SHA 读取；旧通用页仅作路由。本页为源码与一方测试源码交叉核验，未运行测试，认知状态为 `verified_inference`。

## request 顺序与等待范围

顺序为 turn/start → preStep 中 inbox.claim → SystemPrompt 同步 tool providers/schema clone/order → system-prompt/assemble waterfall → abort检查 → agent/pre-step → abort检查 → step/start及user/message → step内 renderPrompt → retry while内 buildRequest → agent/request → abort检查 → prepareCall → header/context记录 → stream → llm/stream waterfall → adapter。

证据：[assembly](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/system-prompt/src/index.ts#L487-L535) · [preStep/turn](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/agent.ts#L225-L284) · [step/retry](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/agent.ts#L332-L389) · [buildRequest](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/agent.ts#L426-L513)。

- `agent/request` 的 next 终端只返回 seedConfig，不调用 provider。某 listener 的 await next() 得到下游配置，外层 listener 仍可改写；全链返回后 prepareCall 还会解析 adapter、补默认值和验证，所以不能称为最终 provider config。其他 middleware 自行发 I/O 不受此终端事实约束。
- 同 step 的 request-error 返回 retry 后，重新 deriveMessages/buildRequest/agent-request/prepareCall/stream，复用该 step assembly.tools 与已渲染 system，不重跑 assembly/pre-step。进入新 step 才重做 assembly。
- fallback/route 改变若经 request waterfall 或 recovery→retry，走上述路径；若在 llm/stream、adapter或SDK内部发生，不会因此重进 agent/request。PreparedLlmCall 一次性 stream handle 不等于单次物理 HTTP。[prepare/handle](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm/src/index.ts#L814-L868) · [stream middleware](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm/src/index.ts#L985-L998)。
- **条件性推论**：该 hook 覆盖标准 AgentLoop 的每轮 buildRequest，不能单独保证每次 physical attempt 都重查；adapter/SDK内部HTTP重试、middleware额外dispatch、直接llm调用均不自动回到它。normal maxRetries:0只令可选llm-retry listener delegate，不能阻止其他listener返回retry或SDK自行重试。[retry](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm-retry/src/index.ts#L157-L210)。
- listener拒绝若最终传播出waterfall、未被外层恢复，buildRequest在prepare/stream前退出，该次标准provider调用为零且不走request-error恢复。signal等待中abort而listener正常返回时，紧接的throwIfAborted阻止后续调用；不合作Promise会一直等待，不是hard kill。此前attempt、日志、assembly或其他middleware的副作用不被撤回。[一方零请求测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/tests/request-error.spec.ts#L31-L48)。

## pre-step reject 的精确日志与短路（2026-09-06 固定 rc.2 复验）

`PreStepDecision` 的拒绝值只有 `{kind:'reject'}`，没有内置reason/message字段。最终waterfall结果为reject且没有abort/throw抢先改走catch时，loop将turn结束为`blocked`；turn/start已经记录，但该拟议step不写step/start/step/end、claimed user/message或request/header，不进入agent/request及provider。首次step拒绝的一方测试明确断言零adapter请求、无user/message与step/start、turn/start→turn/end及blocked。此前step、inbox或plugin独立事件不被撤回，不能推广成整个Session零模型调用或仅两条日志。[测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/tests/interception.spec.ts#L234-L255) · [loop路径](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/agent.ts#L245-L323)。

拒绝分支直接返回reject、不调用next，是waterfall的主动短路合同；先await next再reject会先运行下游，不能宣称其零副作用。prepend只在同一事件链unshift，不会提前到assembly前，也不保证永久第一。更外层listener仍可改写下游结果，只有loop最终收到reject才成立。拒绝时signal已abort则turn变aborted；throw走error；append失败不能保证blocked日志。[waterfall与prepend](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/vendor/cordis/src/events.ts#L224-L259)。

claimed prompt已从inbox取走，reject不自动退回；并发独立inject/steer可保留，不能当作清空全部pending。[一方独立context测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/tests/interception.spec.ts#L426-L455)。本次只读源码与测试，不报告runtime实测。

## 公开 occurrence 身份

payload 为 `{agent,turn,step,signal}`，next/返回值为 `Promise<LlmCallConfig>`；事件按Agent scope过滤，agentEvents把真实Agent与carrier绑定。Agent公开id/session/ctx/inbox/status/options。`Session.header.id + turn + step` 可关联live逻辑step；同step retry保持同一turn/step/turn signal，不能区分每次hook invocation或HTTP attempt。[事件](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent/src/runtime-types.ts#L232-L260)。

Session.events公开readonly日志快照，SessionEvent.seq是Session内事件位置；user/message.data.id为MessageId，可用SessionId+seq关联具体日志occurrence。但agent/request没有promptId/messages/attemptId/rpcId字段，一步可接收多条输入或插件context，不能把最后一条user/message默认当唯一人类prompt。agent/pre-step才带本批messages。[事件位置](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/session/src/types.ts#L395-L415) · [Message](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm/src/message.ts#L128-L153)。

## 工具 output 与 schema 子集

Native ToolSchema只含name/description/parameters，没有顶层title/metadata/outputSchema。ToolDefinition.output为必需的 `{schema,render,presentationMeta?}`：schema校验canonical value，render产生model content，presentationMeta产生top-level UI元数据；还有finalizeContent、presentCall/presentResult等公开方法。title可作为schema annotation或UI render-intent字段，不是Native工具顶层字段。ToolResult为content/isError/meta?。Code Mode另通过sdkSchemas投影output schema以生成SDK类型，不能把Native投影泛化到Code Mode。[字段](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L211-L302) · [投影](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1234-L1266)。

`assertObjectJsonSchema`是dsh-tools根导出的同步assert，只校验自有schema子集并要求object根，不是完整2020-12，也不执行value校验。支持单一type、oneOf、properties/required、boolean additionalProperties、items、scalar enum/const及description/title/default/examples；未知关键词拒绝，$schema/$id/$defs/$ref都不在白名单，本document $ref也拒绝。[子集](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/json-schema.ts#L1-L59) · [assert](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/json-schema.ts#L378-L405)。

校验用显式任务栈与seen集合拒绝对象环，允许sibling reuse；没有maxDepth/maxNodes/时间/AbortSignal预算参数。接受5000层的测试证明避免JS递归栈，不证明有执行资源预算。该路径无network/filesystem loader或ref resolver；但任意JS输入的getter/Proxy可在属性读取时执行自身代码，不能将普通JSON对象校验解释为JS沙箱。[遍历](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/json-schema.ts#L226-L279) · [$ref拒绝](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/tests/json-schema.spec.ts#L118-L128) · [5000层测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/tests/json-schema.spec.ts#L279-L286)。

未指定具体adapter/SDK时，内部HTTP重试/fallback配置与次数未知；外部健康观察如何失效及关联不属于DSH保证。本页不评价调用方项目。

## 2026-09-08 · 互斥 union 与额外输入校验

本节按同一固定 rc.2 SHA 复核，asked_by: agent，状态 verified_inference；上文未涉及部分不因此更改 verified_at。

- `validateJsonSchemaValue` 对 oneOf 的通过条件为恰好一个分支零 violation。官方测试确认 number/integer 对 1 匹配两分支而拒绝，1.5 只匹配 number 则接受。
- **条件性推论**：string（可带 type-correct enum）/null、boolean/null 两组值域互斥，保留全部分支约束时，anyOf→oneOf 的 JSON 值接受集合相同。不是内置 anyOf 支持或通用转换保证；number/integer 等重叠值域不能照搬。
- oneOf 至少两分支，同节点不能带 type；object-root 消费者仍要求根 object，可在 property 放 union。minimum/anyOf 本身不属于该版支持子集。
- 没有独立的 ToolDefinition.validate/refine 或扩展 schema keyword registry。公开输入准入面为工具自身 execute、同步 `ctx.tools.guard`、异步 `tools/pre-execute` waterfall。
- 裸 ToolDefinition 的 parameters 不自动校验 value；官方 raw-tool 测试证实缺 required 字段仍进入 execute。defineTool 的 execute wrapper 才先验证支持子集，再调用户 body；自定义下限仍由工具检查，可用公开 ToolArgsError 表达 INVALID_ARGS。
- 标准顺序是 arguments snapshot/freeze → pre-execute → 最终 ask 时 ApprovalService → allow 时 guards → around-dispatch → body 内参数验证/执行 → output/post/final。guard 在审批之后、body 之前；不能保证早于审批 UI/audit。execute 开头校验只能阻止自己的后续操作，不能撤销 earlier hooks 或日志。
- pre-execute 最终 deny 在 runtime 标准 ask 前；一个 listener 的局部 deny 可被外层 waterfall 改写。guard 返回 string 为单调拒绝、undefined 为不干预；按 Context 决定全局或 Agent scope，可只匹配既有 tool name，无需新工具。它不是 ToolDefinition 上的成员，也不能返回 Promise。
- integer 检查是 JSON number + Number.isInteger，不包含 >=1。额外下限检查可收紧 body 准入，但删除 minimum 仍扩大公布 schema 的值集合；不能宣称 schema 原义完整保留。不得以 coercion 把字符串变成 integer。

证据：[exact-one 实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/json-schema.ts#L486)、[重叠测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/tests/json-schema.spec.ts#L384)、[gate→ask→guard](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1461)、[defineTool 验证](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/schema.ts#L542)、[raw-tool 测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/tests/tools.spec.ts#L2664)、[scoped guard 测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/tests/scoped.spec.ts#L284)。未运行转换器/MCP/审批或实际副作用测试，不评价外部 schema 未提供部分。
