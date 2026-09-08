---
title: rc.1 逻辑重试、输入身份与 Client Action 授权边界
description: 固定 rc.1 的请求相关身份、RPC 一致性、输入取消边界与重试语义。
type: reference
updated: 2026-09-08
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/core/agent-loop/src/agent.ts
  - packages/compaction/compaction-basic/src/index.ts
  - packages/compaction/compaction-basic/src/summarizer.ts
  - packages/llm/llm-retry/src/types.ts
  - packages/llm/llm/src/message.ts
  - packages/llm/llm/src/types.ts
  - packages/api/session-controller/src/types.ts
  - packages/api/session-controller/src/commands.ts
  - packages/core/agent/src/inbox.ts
  - packages/client/connection/src/rpc.ts
  - packages/api/gateway/src/types.ts
  - packages/api/gateway/src/index.ts
  - packages/client/connection/src/rpc-host.ts
  - packages/client/connection/tests/node-half.host.spec.ts
  - packages/api/session-controller/src/index.ts
  - packages/api/session-controller/src/agent.ts
  - packages/attachment/attachment/src/admission.ts
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-06
asked_by: agent
---

条件：TS/JS · CLI/Profile Host与官方Client API · 同Session多attempt/队列编辑 · 持久Session日志 · 公开Gateway与Controller · 不用source字符串代替凭据授权 · 同进程及自定义carrier边界分别判断 · 固定rc.1。

## 2026-09-08 · shared RPC 一致性与 prompt admission 取消

本节按同一固定 SHA 复验，asked_by: agent，verified_inference；不刷新下方其他主题的原核验日期。

- `createSharedFetchHandler('/api')` 从 URL.pathname 取 endpoint，选择 shared interceptor；stock exact Fetch route 只支持 GET/HEAD，不接管合法注册下的 POST。无匹配 interceptor 则先 404。
- shared RPC helper 要求 POST/application-json/合法 clientRequestSchema，然后强制 `message.method===endpoint`。`/api/session/list` 与另一 method 不一致时，不调用 business handler，返回 `gateway/bad-request` 的 server-response。该业务错误用 Response.json 默认 HTTP 200，不是只看 2xx 就可判接受。
- Controller.prompt 仅入口 `signal.throwIfAborted()`；随后 commands.prompt 不接这个 signal。主线 await 边界为 resolveAgent；图片路径还等 per-Agent image-admission Promise chain 与 resolveModelInfo；所有输入都 await admitPromptContent。纯文本 helper 不访问附件存储，但仍有调用方 await continuation；图片内部等 saveImages。
- 完成这些等待后 createUserMessage，再 steer/followup，返回 accepted。中间没有对该 signal 的复查或下传。**条件性推论**：入口之后 abort、其他步骤仍成功时可以继续入箱，不是“取消后一定不再 admission”；该 signal 也不等于 Agent turn signal。
- 相同 requestId 不去重；它只进入 source.rpcId，每次成功 admission 都 mint 新 MessageId。重复仍受独立准入条件约束，不能说必接受，也不能当成旧 receipt 重放。

证据：[shared handler 与 mismatch](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/connection/src/rpc-host.ts#L115)、[同 helper 的 mismatch 测试](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/connection/tests/node-half.host.spec.ts#L400)、[prompt 入口](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/index.ts#L320)、[入箱前 await](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/commands.ts#L288)、[image admission queue](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/agent.ts#L363)、[文本/图片 admission](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment/src/admission.ts#L57)。未运行取消竞态/重复请求 E2E；直接测试用 /rpc，/api 共享相同 helper 的结论来自实现交叉核验。

固定dsh-v0.1.2-rc.1 / a66e4702047846cdaa10c66c9d3df3951f5ea70d，本次只读核对远端tag与本地对象一致。下列均按该完整SHA读取，不用rc.2/1.3补齐。认知状态统一VERIFIED_INFERENCE（源码/公开类型及控制流交叉核验，本库L1/L2封顶）；运行组合未验证之处为UNKNOWN。本次没有安装或runtime测试，没有检查调用方项目。

1. logical step不是immutable physical operation

VERIFIED_INFERENCE：
- 标准顺序是assembly→agent/pre-step→step/start→本批user/message→step()的buildRequest→agent/request→prepareCall→request/header/context→GenerateOptions→llm/stream→adapter。agent/request提供{agent,turn,step,signal}，下一层GenerateOptions只投影sessionId、signal，主请求purpose缺省；没有公开turn/step/attempt id字段。
- step()在while之外固定本step的assembly.tools和rendered system；每次while重新this.session.deriveMessages()、buildRequest、agent/request与prepareCall。request-error返回{kind:'retry'}且未abort则continue同一个step，不再写step/start，也不重做assembly/pre-step。
- 因此同一逻辑step的后续attempt可读取改变后的messages、由agent/request得到不同provider/model/options、由prepareCall解析不同当前注册与adapter默认值。一个prepared调用内部绑定其registration，不意味着下一retry仍绑定上一registration。
- 标准normal llm-retry本身主要是policy/计数/backoff并返回retry，不必改正文或route；但它不向其他middleware保证配置不变。overflow recovery恰好是合法改变body的例子：compaction-basic确认CONTEXT_WINDOW_EXCEEDED后修改Session surface，只有generation推进才retry；同step下一attempt重派生messages，可有不同正文与request series。不能把这识别为“同step非法更换immutable operation”。
- signal是当前turn的控制信号；同step retries共享它，同turn不同step也共享它。同Session同signal既可能有多个主attempt，也可能有多个不同逻辑step。压缩辅助调用通常还携带同sessionId/signal，但purpose:'compaction'且直接ctx.llm.stream，不经agent/request。purpose只是公开分类字段，不是签名授权。
证据：[step与retry](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts#L341-L407)；[request配置与投影](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts#L444-L543)；[overflow改变surface后retry](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/compaction/compaction-basic/src/index.ts#L180-L223)；[compaction直发](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/compaction/compaction-basic/src/summarizer.ts#L146-L164)；[同turn signals一方测试](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/tests/agent-initiator.spec.ts#L166-L201)。

llm/stream与adapter retry：
- llm/stream的公开合同允许around/retry/replay，但loop-built request以exact对象标记并deep-freeze，listener不能原地改写这个请求。它可委派或yield自己的chunks。该层或SDK内部再次physical请求，不会自动重进agent/request，不强制新开step。
- adapter如何形成wire body、重试时是否改headers/token或额外选route、是否派生signal，取决于具体adapter/SDK，不由GenerateOptions统一保证；未指定其全部实现时为UNKNOWN。不能从“冻结GenerateOptions”推出所有HTTP bytes与认证数据永远一样。
[llm/stream合同](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm/src/index.ts#L54-L67)；[GenerateOptions](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm/src/types.ts#L392-L428)。

公开retry identity有但不应过推：dsh-llm-retry公开RetryId类型以及llm/retry{retryId,turn,step,provider,mode,policyKey,retry,...}、llm/retry-started日志。这描述该retry policy owner的恢复进展，不是所有recovery/adapter共享的授权令牌；overflow另有自己的恢复路径，SDK内部不必写这些记录。agent/request-error的{kind:'retry'}是控制流决定，不是附带签名、immutable-body摘要或全局计费幂等键的receipt。
isAgentLoopRequest只识别同进程exact request对象；公开mark函数及WeakSet不是安全attestation，不自动跨IPC或crash保留。
[retry日志公开类型](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm-retry/src/types.ts#L1-L48)；[request标记](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm/src/call-config.ts#L66-L77)。

条件性因果映射：SessionId+turn+step/step-start seq可标识逻辑步骤；同进程标准单Agent driver顺序、受控middleware和对exact request实例的观察，可把loop构造的attempt关联到这个逻辑步骤。DSH并没有直接在adapter参数中给出该tuple，也没有保证“一tuple一physical请求”。要按逻辑step至多结算一次，是外部结算owner语义；上游允许同step多个不同body的尝试，既不要求保存正文重放，也不提供据此自动完成一次结算的合同。

2. prompt/updateQueue输入身份、Action intent与source authority

VERIFIED_INFERENCE：
正式SessionPromptRequest={requestId:SessionRequestId,sessionId,mode:'queue'|'steer',content:PromptContentPart[],clientTimeZone?}。没有source、ActionId/actionIntent、revision、metadata、expectedRevision字段。Host prompt自行创建source={kind:'user',rpcId:request.requestId,clientTimeZone?}，createUserMessage另mint MessageId，再steer/followup。requestId的合同是Client输入相关身份并写入exact message，不是任意Action授权或自动prompt去重；代码没有按requestId去重的分支。
SessionUpdateQueueRequest={sessionId,itemId:MessageId,action}，action仅edit(content)/remove/steer；edit运行时只收text content。它查仍pending的item，edit用{...message,content:newContent}保留原id/source，remove删项，steer移除next-turn项后投向Agent.steer；没有revision/CAS precondition。因此同一MessageId可在后续edit拥有不同内容，不能把它同时当不可变正文版本号。
[请求类型](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/types.ts#L302-L337)；[QueueAction](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/types.ts#L148-L152)；[Host制造source及Message](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/commands.ts#L288-L340)；[queue处理](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/commands.ts#L389-L426)。

“AgentInput”名称：本次对该固定树packages检索未找到名为AgentInput的公开符号。DSH Host Agent接收的是已标识UserMessage，经send/followup/steer/inject/Inbox进入调度；不要将外部同名DTO当成本版本官方合同。
Message公开id/role/content/source；MessageSourceMap是merge-extensible，stock plugin source={kind:'plugin',plugin:string}&ContextFormed，form可为instructions/catalog/snapshot/notice/relay/recall等。它不内建通用Action intent/revision语义。
第一方Host插件可用公开MessageSourceMap类型扩展携带有类型的JSON来源数据，但那是该插件自己定义、验证和解释的语义，不是SessionController.prompt自动接受的新字段。compile-time merge也不自动扩展已生成的Remote codec或赋予Client authority。是否针对特定自定义source能完成read/restore/下游codec的全链契约，须核对其具体扩展；本次未提供，不声称已验证任意Action schema。
[source扩展与stock plugin](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm/src/message.ts#L72-L107)；[Message字段与构造身份](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm/src/message.ts#L127-L200)。

持久化/claim：
Inbox变更以agent/inbox/spliced写入Session，inserted保存完整identified messages；重建Inbox从session.ownEvents重放splices。claim删除待处理列表后发claimed，成功enter的message再原样写user/message。因此在日志真正已持久化且正常恢复前提下可回查MessageId/source及对应event seq；edit的历史splices可分辨各次记录，但没有自动ActionRevision字段。
claim后pre-step reject可能不写user/message，不能以缺少后者断言从未输入；反之发生过source记录不等于当前仍授权。Session.append是live commit，不等于磁盘flush，crash窗口/repair也不能凭accepted响应抹掉。
[Inbox恢复](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent/src/inbox.ts#L28-L38)；[splice持久记录](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent/src/inbox.ts#L178-L192)；[claim](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent/src/inbox.ts#L64-L77)；[成功入步写message](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts#L284-L296)。

Client提供source=plugin的authority：
标准prompt DTO无source，Host也不读取它；即使非标准额外字段跨过某低层边界，标准handler仍构造kind:user，不会仅凭该字段接受plugin权限。若自定义Host endpoint接受Client Action并生成plugin-source日志，权限来自该可信Host handler验证的caller/action政策，而非source字符串。任意可信Host插件可制造plugin source，日志中的plugin名、form、revision或id本身都不是签名、当前授权lease或认证票据。浏览器cookie认证只证明该请求通过默认Web认证，不把Client任意数据升级为第一方执行授权。

3. 不复制私有codec的拒绝位置与carrier范围

VERIFIED_INFERENCE：
HostConnection公开ConnectionRpcHandler(endpoint,payload,signal)，endpoint是namespace/method；Gateway公开InvokeRemoteRequest={namespace,method,args,signal?}和invoke/stream，Gateway自身保留descriptor、lookup、参数与结果codec。故在拥有相应Host边界时，按公开endpoint或InvokeRemoteRequest决定拒绝，然后对允许的调用继续交给官方Gateway，不需要复制私有业务codec。这里Remote namespace/method是公开寻址，不是直接把任意Service对象暴露给Client。
[Connection handler](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/connection/src/rpc.ts#L99-L107)；[Gateway public请求](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/types.ts#L9-L19)；[invoke/stream正式处理](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/index.ts#L292-L329)。

目标入口已核对：credentials/set(ref,value)、credentials/unset(ref)为CredentialsController方法；llm/discoverModels(settingsNs,request,signal)的request.apiKey是一次性raw key入口，不走credentials.set。方法级阻断整个discovery可不解析内部key；若只拒绝含key的discovery，则可用公开LlmModelDiscoveryRequest字段，但仍须对不可信输入做对应边界判断，而不是信任Client类型断言。
[credentials方法](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/settings-controller/src/credentials.ts#L92-L117)；[discovery Remote](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm/src/index.ts#L612-L627)；[apiKey字段](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/llm/llm/src/types.ts#L233-L249)。

范围限制：
- 默认Gateway在Connection注册唯一/api interceptor。自定义Connection owner可包裹交给它的handler，但一个已经加载的默认Connection不提供可叠加第二interceptor的列表：重复同channel会throw。
- rc.1 exact Fetch routes只有GET/HEAD，不能用它新增POST拦截宣称覆盖上述方法（那是另版本能力）。
- 官方mux由Gateway直接装配，升级依赖Connection.requestRejection(headers)，逐logical stream则直接openWireStream，不通过HTTP unary interceptor。Gateway校验mode，unary set/unset/discovery不能合法以stream调用；所以这几条纯unary方法不应被误称为“可由正常mux直接绕过”。但这不代表所有stream的其它敏感业务都被unary policy覆盖。
- in-process可直接Gateway.invoke/stream或wireStream.open；这些是否经过自定义Connection策略取决于实际carrier wiring。ClientTransportHooks.fetch/openStream只控制使用它们的Client路径，不能成为Host权限隔离。若要所有Client调用共享政策，必须证明每个被提供的入口都经同一可信决策或各自等价检查，默认库不自动给这个证明。
- 可信Host插件直接ctx.credentials.set、llm.discoverModels或其他Service方法属于内部信任调用，绕过所有Client Connection/Gateway；不能用Client API策略声称沙箱化任意Host插件。若政策本来只约束Client API，应明确范围，内部可信调用不必假装也来自Client。
[唯一interceptor](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/connection/src/rpc-host.ts#L179-L199)；[GET/HEAD](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/connection/src/rpc.ts#L109-L129)；[unary和mux各自装配](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/index.ts#L198-L217)；[mode拒绝](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/gateway/src/index.ts#L298-L328)。

UNKNOWN：未运行自定义Connection+官方Client/unary/mux/in-process完整组合，未证明你的全部carrier都强制同一policy；也未验证特定Action schema、拒绝后兼容表现、或一次结算系统。以上仅是DSH类型/控制流事实和条件性后果，无项目诊断或补丁。
依据：0.1.2-rc.1 / a66e4702047846cdaa10c66c9d3df3951f5ea70d，固定SHA关键路径/行号见链接。
