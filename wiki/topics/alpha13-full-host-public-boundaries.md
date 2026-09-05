---
title: 0.1.3-alpha.1 全量 Host 公开面、发布渠道与集成边界
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - package.json
  - packages/boot/app-boot/src/profile.ts
  - packages/api/session-controller/package.json
  - packages/api/session-controller/src/index.ts
  - packages/api/session-controller/src/commands.ts
  - packages/api/gateway/src/index.ts
  - packages/api/settings-controller/src/credentials.ts
  - packages/api/settings-controller/src/index.ts
  - packages/client/connection/src/client/index.ts
  - packages/client/connection/src/browser-auth.ts
  - packages/client/connection/src/rpc-host.ts
  - packages/client/connection/src/rpc.ts
  - packages/client/modules/src/index.ts
  - packages/client/locale/src/locale-settings.ts
  - packages/client/file-upload/src/index.ts
  - packages/host/frontend-static/src/index.ts
  - packages/host/plugin-inventory/src/index.ts
  - packages/mcp/mcp-client/src/transport.ts
  - packages/mcp/mcp-client/src/connection.ts
  - packages/mcp/mcp-client/src/tools.ts
  - packages/skill/skill/src/index.ts
  - packages/skill/skill-filesystem/src/index.ts
  - packages/experimental/agent-team/package.json
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent-loop/src/assistant-stream.ts
  - packages/core/tools/src/index.ts
  - packages/interaction/user-approval/src/index.ts
  - packages/llm/llm/src/types.ts
  - packages/session/session-persistence/src/index.ts
  - packages/session/session-persistence/src/handle.ts
  - packages/session/session-persistence-jsonl/src/generation.ts
  - packages/session/session-persistence-jsonl/src/storage.ts
  - packages/session/session-format-v1-to-v2/src/migration.ts
  - packages/session-query/session-log-export/src/index.ts
  - packages/preset/agent-presets/src/index.ts
  - packages/bundle/web-app/cordis.patch.yml
commit: d347e703908d0406b7a7ef80e3a0e594d86b2215
verified_at: 2026-09-06
asked_by: agent
---

八维条件：TS/JS Host 与自定义 Client · CLI/Profile 全量 Host · 多 Session/可选子任务 · JSONL 本地存储与平台内核锁 · 官方工具/插件公开面 · 无真实模型凭据调用 · Web 或桌面壳待验 · 主目标 A 与对照 R 分别固定下文 SHA。

发布状态是 2026-09-06 的只读快照，后续“最新版/可安装”问题必须重新查询。下面的源码结论只适用于各自固定 SHA；未执行运行时 conformance。

主目标 0.1.3-alpha.1 / d347e703908d0406b7a7ef80e3a0e594d86b2215（下称A）；对照 0.1.2-rc.1 / a66e4702047846cdaa10c66c9d3df3951f5ea70d（下称R）。两个tag与远端完整SHA一致；后续源码/manifest/diff均用固定SHA读取，无checkout/switch、依赖安装、真实MCP连接或conformance运行。查询跨至2026-09-06（Asia/Shanghai），日期敏感状态以此次查询为准。

认知状态：下列“VERIFIED_INFERENCE”包含源码/类型直接核验和明确写出的条件性推论；本库相关领域仅L1/L2，不能把静态检查升级为FACT/L3。UNKNOWN明确列出未取得的运行、闭包或项目证据。旧rc.2页只作路由，未拿旧缺口代替新版核验。

核心结论：A有新的SessionHandle/写锁/format-v2 assistant settlement、文件上传与浏览器认证；Client/Remote/controllers拓扑在R就已存在，ClientTransportHooks在A仍公开。Team有新行为但仍private experimental。A的源码tag已发布，却没有此次可取得的同版npm根包和抽查的关键companions，不能当成已经可安装验收的正式全量Host。

1. 发布渠道、Host / Client / Profile / plugin公开面

[VERIFIED_INFERENCE：官方HTTP元数据直接观测]
GitHub releases列表第一项仍是dsh-v0.1.3-alpha.1，published_at=2026-09-04T11:34:32Z，prerelease=true，assets=[]。根npm dist-tags latest/next=0.1.2-rc.1，alpha=0.1.2-alpha.5；精确GET根0.1.3-alpha.1返回404，R返回tarball URL。PyPI SDK为0.1.2rc1，要求runtime-bin==0.1.2rc1。
此外只读检查11个关键companion精确版本：web-app/base/api-session-controller/api-gateway/client-connection/client-web/mcp-client/skill/session-persistence/session-persistence-jsonl/session-format-v1-to-v2，A均404；R前10个均有tarball元数据，最后一个R不存在且R源码也未含该migration包。这个抽查足以否定“A已有上述官方npm安装链”，不等于穷尽全体registry包。
GitHub源码/自动source archive可用于源码研究；Release无附加assets，不代表有打包好CLI、browser lib、typert生成物、native helpers的可用整套产物。[UNKNOWN] 未验证R完整递归closure、A源码构建与全部companions/native/平台install-boot。本次禁止安装/conformance，不能报告其通过。
官方入口：[GitHub release](https://github.com/deepseek-ai/deepseek-harness/releases/tag/dsh-v0.1.3-alpha.1)、[releases API](https://api.github.com/repos/deepseek-ai/deepseek-harness/releases?per_page=3)、[npm metadata](https://registry.npmjs.org/@deepseek-ai%2Fdsh)、[PyPI](https://pypi.org/pypi/deepseek-harness-sdk/json)。GitHub /releases/latest返回404不能解释为无预发布；按列表与tag已取得记录。

[VERIFIED_INFERENCE：两SHA manifest/源码]
R与A都已经没有旧host/apiproxy与client/runtime包，替代为：
- dsh-api-session-controller：Host Session命令、Agent寻址/恢复、history/control streams；/client为Session对象、list/binding/Client状态。
- dsh-api-workspace-controller：Workspace命令与投影，含/client。
- dsh-api-settings-controller：settings与credentials Remote；并非Client状态store。
- dsh-client-store：React-free observable/snapshot primitives；localStorage persistence只是浏览器本地。
- dsh-api-gateway：Host TypertGatewayService + /client的ctx.remote；dsh-api-remotes选择Host事件转发。不是旧ApiProxy RPC的原样别名。
这些package manifests根公开types/runtime；controllers有/types、/typert、/remote，Session/Workspace还有/client，具体各自manifest为准。/remote是生成的Host-for-Client contribution；源码装饰器存在不等于分发产物已生成。
证据：[A API map](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/api/README.md#L24-L38)；[R session-controller exports](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/package.json)；[A session-controller exports](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/api/session-controller/package.json)；[Client store](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/store/README.md)。

ClientTransportHooks没有移除：A公开于dsh-client-connection/client，形状fetch、openStream?、loadBundle?、ownsHost?，启动前由page global __DSH_TRANSPORT__提供；默认served Web使用HTTP+WebSocket，worker预览使用替代carrier。ownsHost是transport owner对其自有Host的声明，不是HTTP鉴权票据或可信用户认证。
[Hooks与ConnectionHandle](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/connection/src/client/index.ts#L73-L139)。
全量Web自定义Client的现有公开路径是保留CLI/Profile的Host组合，Client用web boot、Connection/Gateway、generated Remote、controllers、store和ui-slots/ui-renderer进行组合；宿主若接管physical carrier，可使用上述hooks及Host公开Connection/Gateway面。官方源码明确支持这种carrier扩展；[UNKNOWN]没有取得一份对任意Electron/WKWebView完整桌面壳提供兼容/SLA保证的官方承诺，不能称其为现成桌面SDK。

Profile仍为$DSH_HOME/profiles/<name>下package.json中的dsh.profile.bundles与cordis.patch.yml；按ordered bundles→profile patches→launcher patches合成，Bundle声明dsh.bundle.patch。这是支持out-of-tree插件及组合裁剪的公开声明面，不能只把最新包名塞入旧默认Profile。
[Profile算法及manifest](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/boot/app-boot/src/profile.ts#L1-L65)；[Web bundle实际组合](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/bundle/web-app/cordis.patch.yml)。
Client face元数据仍由package.json dsh.client的platform/inject/external/immediately等与exports['./client']描述；external负责模块身份，inject负责服务依赖，不能互代。
[Client元数据解析](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/modules/src/index.ts#L190-L233)。

插件管理要区分三类：host-plugin-inventory的Remote list只是当前Loader inventory；ui-settings-plugins编辑已注册settings并非通用npm安装器；Cordis Loader有entry update/remove和Fiber生命周期，cordis-host-runner另有动态定义/run/stop/inventory等Remote，针对运行期代码，不等于受信市场/签名安装/版本锁闭包。本次未发现统一的公开“插件市场安装+信任+全依赖固定”事务API。
[inventory只读list](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/host/plugin-inventory/src/index.ts#L46-L92)；[settings插件Host空壳](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/ui-settings-plugins/src/index.ts#L1-L11)；[Loader entry](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/vendor/loader/src/config/entry.ts#L142-L180)；[动态runner](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/extensions/cordis-host-runner/src/index.ts)。

2. MCP与Skill

[VERIFIED_INFERENCE：A；所查MCP/Skill核心src在R→A diff为空，不能把这些称为1.3新发明]
MCP官方桥为dsh-mcp-client root plugin，transport配置只支持stdio与streamable-http。stdio command/args/cwd与env；HTTP url/headers。stdio合并scrubbed ambient env与显式env，显式env仍可有secret；不是“DSH自动让所有secret留在某外部Main”。实际spawn由MCP SDK承担，不等于通过整个DSH subprocess provider执行。
没有在该桥接Config/createTransport中发现authProvider/OAuth flow、CredentialProvider ref resolver或公开custom transport factory。SDK自己的可选能力不能当作该DSH包已接入。[transport完整分支](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/mcp/mcp-client/src/transport.ts#L9-L49)；[MCP配置](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/mcp/mcp-client/src/index.ts)。

工具发现走tools/list（分页）与tools/call；notifications/tools/list_changed触发序列化resync。fetch失败保留旧集合；真正swap先dispose旧集合再register新集合，注册冲突回滚本次新注册，结果是空集合而非自动恢复旧集合，不能把“full generation or none”解释为all-or-old。远端outputSchema支持子集时校验structuredContent，不支持时回退unconstrained JSON。
[swap精确语义](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/mcp/mcp-client/src/tools.ts#L125-L193)；[通知](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/mcp/mcp-client/src/connection.ts#L255-L267)。
默认per-call timeout 60s，signal传SDK。连接恢复默认enabled，500ms起、30s cap、10次连续失败预算；outage期间旧工具仍列出但调用失败，耗尽后撤工具。dispose关闭timer/client，等待connect attempt与sync chain再注销；关闭超时会日志警告，不能推出远端physical effects已经drain。
[MCP调用](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/mcp/mcp-client/src/tools.ts#L81-L95)；[dispose](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/mcp/mcp-client/src/connection.ts#L327-L348)。
resources/prompts作为MCP独立协议能力没有本桥的list/read/get前端或消费者证据；tool result里的resource_link/embedded resources投影不等于支持资源浏览和prompt invocation。也未发现public durable server/tool generation lease或schema→exact remote execute原子保证。MCP是可选插件，默认Web树未配置服务器，不等于开箱连接任何server。

Skill：
- dsh-skill root公开registerProvider/list/snapshot/get/register；候选有name/description/whenToUse/invocation/source/provider/rank/opaque locator/resourceBase等；global与scope chain合并，近scope覆盖同名，非同名继承仍可见。
- filesystem来源：project .dsh/skills、.agents/skills、custom dirs、user DSH/agents、可选bundled；默认roots可关闭；只扫描一层SKILL.md bundle或flat md。source/rank/invocation不是签名信任、安装审核或sandbox权限。
- provider.get每次读取当前body，校验选中的name；resourceBase提供相对资源目录提示，不是自动执行script，也不冻结资源bytes。watcher/provider invalidate更新catalog；body/resource内容不具备immutable digest/compare-and-read租约。
- modelInvocable/userInvocable分别控制模型与人类入口；tool-skill把catalog和加载body放入标准model-visible历史，UI picker由ui-skill提供。描述/whenToUse是路由提示，不能当作确定性自动触发策略；没有在这些公开面发现远程安装/签名/信任管理器或全Skill资源快照合同。
[Skill types](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/skill/skill/src/index.ts#L49-L116)；[provider/control](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/skill/skill/src/index.ts#L249-L276)；[get当前body](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/skill/skill-filesystem/src/index.ts#L200-L221)；[默认roots](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/skill/skill-filesystem/README.md#L55-L93)。
正式Web组合禁用base的filesystem/tool-skill rows，再由Preset装配Agent所需skill，registry保留Host；不能只看base rows认定每个preset可见同一集合。[Web覆盖层](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/bundle/web-app/cordis.patch.yml#L358-L379)。

3. Team/subagent/workflow/jobs/goal、settings/budgets、deterministic tools

[VERIFIED_INFERENCE]
Team A仍为experimental/private，R亦如此，正式release payload排除；有独立team-profile/web-profile和Client UI代码。Team roster/mailbox/task-board写Lead Session，send先durable queue+flush，再对target做receipt去重；A send统一Steer，支持cold teammate恢复。它是单进程协调和target Session去重，不是跨进程exactly-once/共识；共享cwd，不提供worktree/文件锁，task owner不因idle/cancel自动释放。
[private manifest](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/experimental/agent-team/package.json)；[Team mailbox/限制](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/experimental/agent-team/README.md#L128-L150)；[Team src入口](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/experimental/agent-team/src/index.ts#L153-L170)。

Subagent正式seam支持多provider、one-shot与continuable、spawn/fork、parent-child消息与cold child resume、list children/descendants；不同provider能力不一致，ACP/Codex/Claude不等于本地durable continuable能力。默认base包含in-process spawn/fork与对应工具，其他后端按组合启用。
WorkflowEngine.start(request)返回WorkflowRun；worker-thread provider隔离运行并有sync timeout/dispose grace/child caps，脚本可被终止不代表它拥有的所有外部效果可被撤销，也不是自动持久调度恢复引擎。Jobs local为进程内记录，per-owner并发cap，kill是requested/already-finished，重启不恢复进程任务。Goal有durable phase/revision/maxGoalRounds、process-local armed；resume/fork后需显式重新arm。A Web pause经goal-round-driver取消当前turn；round-count预算不是token/currency/时间预算。
[Subagent合同与限制](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/subagent/subagent/README.md)；[Workflow start](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/workflow/workflow/src/index.ts#L157-L168)；[worker限制](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/workflow/workflow-worker-thread/README.md#L38-L54)；[Jobs方法](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/jobs/jobs/src/index.ts#L78-L143)；[Goal pause取消](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/goal/goal-round-driver/src/index.ts#L284-L292)；[Goal预算范围](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/goal/goal/README.md#L151-L159)。

settings仍是owner注册namespace/schema、分层值、revision及冲突检测、update/replace/mutate；settings-controller公开Remote配置。各预算分别属于各owner：LLM maxTokens是输出cap，AgentLoop maxParallelToolCalls是并发pool，tools.timeoutMs需timeout-policy执行且合作取消，Goal rounds、Jobs concurrency、Workflow children/timeouts各有自己的范围。没有从上述面验证统一Host全局费用预算、跨工作load的统一drain或exactly-once。未知不能补成已有能力。
[Settings public handle](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/settings/settings/src/index.ts#L114-L148)；[AgentLoop config](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/core/agent-loop/src/index.ts#L309-L374)。

取消/审批：Agent.cancel与whenIdle仍分别是取消请求/当前driver+maintenance收口；session-controller.cancel返回accepted不等待whenIdle。tools/result仍是只读结果通知，不能推成异步cleanup receipt。
A ApprovalService.request明确要求open turn，idle ask直接throw且不写日志；ask/decided记录成对，取消/无answerer fail closed。R检查对应源码未含这条open-turn拒绝，因此不能沿用旧任意idle ask假设。
[cancel返回](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/api/session-controller/src/commands.ts#L459-L472)；[审批open-turn前置](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/interaction/user-approval/src/index.ts#L191-L225)。

确定性插件调用：
公开ctx.tools.execute(ToolExecutionInput)确实不要求模型自主产出tool-call，走pre-policy/approval/guards/around/post/result，可以由插件确定参数发起。它不自动创建标准Session tool/call和tool/result，也不替caller建立turn/step。标准AgentLoop是在assistant已声明tool call后自行写call/result；Session类型对tool/call的语义仍是“model requested”。
因此“调用物理工具无需模型选择”在primitive层成立；“任意logged final后、即使turn已结束，自动得到标准call/approval/result事务”不成立为已证实合同，open-turn审批就是明确前置。公开agent/turn-stopping是awaited serial且仍在turn内，但仅此不能自动证明完整synthetic tool history合法性。公开commands是command/run+command/done独立日志，不是tool pair替身。
[公开execute](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/core/tools/src/index.ts#L1319-L1335)；[标准tool event合同](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/core/session/src/types.ts#L314-L337)；[turn-stopping](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/core/agent/src/runtime-types.ts#L317-L343)；[command日志分离](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/interaction/commands/src/types.ts#L91-L111)。
[UNKNOWN] 本次没有一方无模型final→deterministic tool→标准tool日志的完整同形conformance证据；不能替调用方认定其完整目标已获DSH支持，也不设计补丁。

4. LlmAdapter、attempt/计费与request重建

[VERIFIED_INFERENCE]
A GenerateOptions仍包含provider/model/messages/system/tools/temperature/maxTokens/stop/signal/sessionId?/purpose?，没有公开turn/step/attempt稳定id。A新增LlmAttemptId用于Agent assistant-stream live frames，AssistantStreamAttempt以SessionId+attached lifecycle counter生成；counter不是跨恢复稳定全局计费键，也未加入GenerateOptions。
[GenerateOptions完整段](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/llm/llm/src/types.ts#L406-L443)；[live attempt构造](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/core/agent-loop/src/assistant-stream.ts#L18-L45)。

A成功assistant/message或失败/重试/取消assistant/attempt携带compact raw stream，以turn/step及event seq记录durable settlement；live frame有attemptId/revision并指向settled event seq，revision生命周期内单调。这显著改善attempt历史与Web流对齐，但它记录的是已settle的DSH逻辑stream，不等于provider每个HTTP或billing occurrence。process在结算前crash、SDK内部多请求、provider是否记费仍不能由日志排除；不能证明同逻辑step只计费一次。
[v2 event](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/core/session/src/types.ts#L282-L313)；[retry前attempt settlement](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/core/agent-loop/src/agent.ts#L344-L447)。

AgentLoop同step retry仍重新buildRequest/agent-request/prepareCall，复用assembly；request-error middleware可返回retry，llm/stream及SDK内部重试不回到AgentLoop hook。PreparedLlmCall的一次性dispatch不等于physical-attempt去重。相对R，A主要新增raw stream settlement/series等可观测与重建信息，不给adapter提供一个公开全局exactly-once billing token。
request/header保存resolved config、system与tool schema快照，model messages由Session surface derive；A有startsRequestSeries/series边界及surface replacement generation处理。可重建DSH-level request，不包括运行期ToolDefinition闭包、plugin全依赖、provider raw HTTP/auth、所有SDK内部重试和外部业务执行状态。
[A buildRequest/header](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/core/agent-loop/src/agent.ts#L489-L587)；[R对照buildRequest所在文件](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts)。

5. SessionHandle / migration / query / preset-model /依赖

[VERIFIED_INFERENCE]
A SessionPersistence.create/open返回SessionHandle，access=read/write；read允许与其他writer共存，write有单owner。Handle append只保证本backend实例中有序/可见，flush才保证crash durability并使空Session物化，close完成pending durability并释放owner；service.flush聚合active write handles。R仍service append(id,...)/load形式；AgentLoop.create由R同步变A异步，不能照搬。
[A persistence根exports与方法](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/session/session-persistence/src/index.ts#L14-L197)；[A handle完整合同](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/session/session-persistence/src/handle.ts#L9-L105)；[R persistence](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/session/session-persistence/src/index.ts)。

A JSONL内核锁：existing artifact write-open取lease；read不取写锁；新建lazy Session到materialization write才拿跨进程lease，不能将release简写理解成“所有read互斥/空session立即跨进程加锁”。lease two-process测试有跨进程写拒绝/read允许/进程退出后接管，本次只读源码未跑native测试。
[write open](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/session/session-persistence-jsonl/src/index.ts#L247-L279)；[materializing lease](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/session/session-persistence-jsonl/src/storage.ts#L304-L326)；[双进程测试](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/session/session-persistence-jsonl/tests/lease.two-process.e2e.ts)。

format-v2：相邻v0→v1、v1→v2 migration包+catalog。JSONL读稳定源快照、核对source fingerprint后发布immutable successor generation，保留旧源；v1→v2把chunk聚成assistant attempt/message streams并校验provenance/fork cut。迁移旧格式读取可产生新artifact，因此“只读业务历史查询”不能自动推成磁盘绝不写。更高未知格式拒绝，不是任意版本双向兼容或降级承诺。
[generation发布](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/session/session-persistence-jsonl/src/generation.ts#L662-L750)；[v1→v2](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/session/session-format-v1-to-v2/src/migration.ts#L33-L62)。

查询/导出：session-controller有list/search/page与stream控制/history，Session query系提供语料/结构与FTS等独立包；session-log-export在Web提供/export及经过Connection的GET/HEAD ZIP下载。不是应用所有数据/凭据/插件包的备份恢复包。
删除/retention：SessionPersistence public create/open/flush/stat/list没有delete；SessionController公开方法也未见delete。Workspace.delete/archiveSession是Workspace/grouping层，不可当作Session artifact删除或释放live Agent的事务。没有取得统一Session TTL/GC/全artifact retention policy的公开合同；不能凭现有方法自创它。[Web导出](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/session-query/session-log-export/src/index.ts#L73-L99)
[Workspace commands](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/api/workspace-controller/src/commands.ts#L92-L160)。

Preset/model：AgentPresets.select仍是blank条件下recompose后append agent-preset/selected，A通过turnBoundary projection检查开始状态；并非Session活动时任意切换。SessionController.selectModel解析exact config，更新live next-request selection并尝试保存Host默认model（失败warn）；Session自己的最终resolved config到后续request/header才留痕，不可将空Session切model应答当已保存同一Session exact request。
[Preset select](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/preset/agent-presets/src/index.ts#L686-L727)；[model select](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/api/session-controller/src/commands.ts#L124-L159)。
历史Session能看到记录的preset id、request schemas、已披露Skill catalog/body等；不能枚举所有历史runtime plugin/Skill资源/package revision闭包。Skill body是当次记录，资源目录与以后provider.get不被冻结；Preset standing generation与composition文件也不是Session中全量可恢复依赖manifest。
原生DSH迁移只拥有其Session格式、generation和persistence；应用runtime generation、外部数据绑定、凭据、物理效果与发布回滚不由上述DSH migration接管。这是owner边界推论，不诊断调用方应用。

6. Web安全、raw credentials与Client UI/WKWebView

[VERIFIED_INFERENCE；R→A browser-auth与所查settings-controller源码无diff，这些不是刚到1.3才新增]
A默认Connection在请求信任检查后要求signed browser cookie；launch token仅GET /兑换、303跳干净URL。frontend-static根/index调用authorizeIndex，API与Gateway WebSocket升级走requestRejection；static assets仍公开。Host要求loopback或trustedHosts匹配，Origin若存在须一致，cross-site拒绝。Host/Origin是防跨站，不是身份；cookie才是此默认Web认证。不存在method-specific loopback tier。
[root cookie交换](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/connection/src/browser-auth.ts#L232-L301)；[HTTP admission](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/connection/src/index.ts#L98-L125)；[frontend授权入口](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/host/frontend-static/src/index.ts#L67-L90)；[Gateway WS装配](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/api/gateway/src/index.ts#L198-L224)。
cookie为host-only/HttpOnly/SameSite=Strict，loopback HTTP下不设Secure；secret在Credentials owner record、Connection activation时加载，删除record不是立即使该已激活实例忘记secret。无logout API。自定义physical carrier不自动继承HTTP cookie边界，ownsHost也不能替代其自身信任证明。[Browser auth源码](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/connection/src/browser-auth.ts)。

credentials raw set/unset仍公开于CredentialsController；SettingsController构造自动mount它，即使provider缺失也保留namespace。没有disableCredentialsSet这样的公开配置。禁UI或让provider拒写不能证明raw secret没过wire。
可用公开扩展面确实比旧ApiProxy分层清楚：CredentialsController是root导出的service，Connection公开exact Fetch route registry，其匹配先于shared Gateway fallback；因此Host-owned plugin可对相应HTTP入口返回拒绝或替换相关composition。但这是“该入口可拦截”的条件性事实，不是内置全carrier method ACL；同一/api interceptor仅允许一个（Gateway已占），不能挂第二个通用interceptor假定叠加。in-process invoke/自定义carrier不经那条HTTP route时不受其保证，Client hooks单独过滤也不是Host权威禁用。
[raw set/unset](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/api/settings-controller/src/credentials.ts#L92-L118)；[自动mount](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/api/settings-controller/src/index.ts#L95-L108)；[Connection public route](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/connection/src/rpc.ts#L128-L184)；[exact route先于Gateway](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/connection/src/rpc-host.ts#L117-L135)。

Client可复用的是各公开/client插件、ui-slots/renderers、controller/store：Web bundle已列Skill picker、Subagent、Workflow、Jobs、Goal、Approval、附件/通用file upload、trajectory等。A任意文件upload有Host流式route、进度/取消与staged receipt，非任意文件内容自动塞模型。MCP工具可复用generic工具卡；没有查到正式独立MCP server管理UI包。Team UI在experimental/private profile，不在正式default Web。
内置Locale IDs严格为['zh','en']，有语言包register扩展，但没有默认三语（尤其日语）保证。[locale IDs](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/locale/src/locale-settings.ts#L14-L26)；[language-pack注册](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/locale/src/client/index.ts#L359-L384)；[file upload公开服务](https://github.com/deepseek-ai/deepseek-harness/blob/d347e703908d0406b7a7ef80e3a0e594d86b2215/packages/client/file-upload/src/index.ts#L56-L136)。
[UNKNOWN] 未找到WKWebView专门适配/测试或官方兼容承诺。本次未跑WebKit、cookie持久化、CSP/CORS、custom scheme、WebSocket、文件拖拽/上传与跨进程carrier实测；因此只能说Web+公开transport seams可作为集成输入，不能说WKWebView成品体验已经被上游验证。

收尾边界：以上六项均已按目标版本核验并给出条件/未知，没有将“源码有接口”写成“完整产品与安装闭包通过”。本次未检查调用方项目、未安装依赖、未运行真实MCP/模型/conformance。部分README存在过度概括：MCP注册失败不恢复旧代；Team非跨进程exactly-once；Release写锁不排斥read；BrowserAuth末尾文档与activation实现有措辞差异，以所列T1为准。
依据：A d347e703908d0406b7a7ef80e3a0e594d86b2215；R a66e4702047846cdaa10c66c9d3df3951f5ea70d；所有源码链接固定SHA。本地对象库 /Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness。
