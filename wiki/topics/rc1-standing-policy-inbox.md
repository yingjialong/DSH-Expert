---
title: rc.1 固定 standing 的 Agent scope 策略、步骤限制与 inbox 观察
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/core/tools/src/index.ts
  - packages/core/tools/tests/scoped.spec.ts
  - packages/core/system-prompt/src/index.ts
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent/src/inbox.ts
  - packages/core/agent/src/runtime-types.ts
  - packages/core/agent/src/dispatch.ts
  - packages/skill/skill/src/index.ts
  - packages/skill/tool-skill/src/index.ts
  - packages/goal/goal/src/index.ts
  - packages/jobs/jobs/src/index.ts
  - packages/schedule/schedule/src/index.ts
  - packages/subagent/subagent/src/types.ts
  - packages/interaction/user-approval/src/index.ts
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-06
asked_by: agent
---

八维条件：TS/JS · 同一非blank Session与标准AgentLoop · 多Agent/turn策略 · 文件系统形态无关 · 固定Preset standing不recompose · 无真实凭据调用 · live Host · 固定rc.1。

版本：dsh-v0.1.2-rc.1 / a66e4702047846cdaa10c66c9d3df3951f5ea70d，远端tag与本地对象一致；全部源码按完整SHA读取，无切换共享工作树、无运行conformance或外部项目检查。知识库仅作路由，已核对错误本中scope/Skill与assembly的历史陷阱。本次源码/类型/一方测试源码交叉核验统一VERIFIED_INFERENCE（本库L1/L2封顶，不标FACT）；未运行的组合列UNKNOWN。

1. restrict与assembly的真实时序

VERIFIED_INFERENCE：
agent.ctx.tools.restrict(filter)作用于该Agent所继承的global和全部ancestor ToolLayer，standing preset层在此就是ancestor，因此可以过滤其工具，不要求recompose或Session blank。view()先合成inherited并应用沿链各restriction的交集，再加入该scope own registrations，最后加入非native模式的reserved run_code。own layer不受此过滤；run_code不能在allow/deny中点名，且在非native模式下作为基础设施重新加入。因此allow:[]也不自动等于所有工具都不可见。
restrict注册与返回disposer生效于后续registry读取；lifting某restriction不移除其他restriction，交集仍生效。它不是definition替换或任意物理能力沙箱。
[restrict与guard公共合同](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/tools/src/index.ts#L1055-L1105)；[standing inherited与own豁免](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/tools/src/index.ts#L1121-L1183)；[祖先过滤/own保留测试](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/tools/tests/scoped.spec.ts#L217-L261)。

对下一assembly：只要restriction变化发生在该次SystemPrompt调用其tool providers之前，ctx.tools对应provider会读取新view。已克隆到assembly.tools的旧值不会因为tools/change或disposer自动重算；已启动provider请求也不会因此撤回。
精确顺序是claim → systemPrompt.assemble：variables/providers/schema/context收集→await system-prompt/assemble→返回 → await agent/pre-step → step/start → agent/request/provider。
system-prompt/assemble虽awaited，但在schema/context收集之后。其返回PromptAssembly是权威变换结果（complete section/suppressRuntimeContext有各自最终恢复规则），所以可以对已收集结果做后处理；**仅在该hook改mask，不会自动重跑collector**。agent/pre-step也太晚；agent/request只返回LlmCallConfig，不携带tools/messages覆写字段。
[collector与waterfall](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/system-prompt/src/index.ts#L536-L610)；[preStep顺序](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts#L234-L251)；[assembly事件合同](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/system-prompt/src/index.ts#L19-L31)。

未找到通用、每step在schema/context收集之前await的公开hook。边界上的已有事实：
- turn/start的session/event或agent/inbox/claimed是同步观察点，正常路径早于本次assembly；可同步更新已有policy state，但不是awaited async admission，返回Promise不会让loop等待。它们不覆盖任意外部直接调用systemPrompt.assemble。
- 上一turn-stopping是awaited且早于可能的后续step/turn，但只在拟结束时触发，不覆盖每个tool-loop step。
- 若进入assembly前就已设置policy，下一collector自然看到；若依赖async检查，只能区分“在外部已控制边界等待完成”和“在post-collection hook处理最终assembly”，不能把它们声称为上游新pre-assembly hook。
此处是时间边界说明，不提供产品patch。

2. Skill/Goal/Jobs/Schedule/subagent能按何种公开scope/policy变化

VERIFIED_INFERENCE（每个owner不同，不能用同一tool mask替代全部状态）：

工具与permission：tools.guard可在agent.ctx注册单调同步拒绝，tools/pre-execute是scope-filtered async策略，permission/approval有各自Session状态。ApprovalService.setPolicy(agent,policy)不recompose，会写policy并inject一条说明到next-step；'never'的含义是遇ask不交互/拒绝，不是禁用全部已经allow的调用。既有执行/定时器/直接service调用不因此自动停下。
[Agent guard](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/tools/src/index.ts#L1091-L1116)；[Approval setPolicy会inject](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/interaction/user-approval/src/index.ts#L169-L186)。

Skill：
- SkillRegistry有按scope合并的register/registerProvider/list/snapshot/get；invocation.modelInvocable/userInvocable是每条Skill定义的surface策略，非通用每Agent每turn allowlist。近scope覆盖同名，不过滤其他继承Skill；没有可直接套用的skills.restrict。
- shipped tool-skill的模型catalog pre-step listener会检查ctx.tools.get(skillTool.name,agent)===本插件的exact definition；被mask时不会照常发布该可见catalog，不能说这个consumer完全不观察工具可见性。
- 但用户显式/<name>加载是另一pre-step listener，读userInvocable Skill并注入body；其逻辑不以skillTool可见为统一开关。只隐藏skill tool不能关闭此入口、所有Skill provider读取或已经存在的Skill历史。资源文件也不因mask被冻结或变得不可读。
[Skill策略/来源类型](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/skill/src/index.ts#L49-L94)；[用户与模型两条consumer](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/tool-skill/src/index.ts#L163-L222)。
若要求某个第一方功能按Agent/turn精确启停其全部Skill消费，需要该consumer/provider拥有明确条件化策略；上游通用tool mask不能替它完成。此为缺少统一开关的边界，不建议具体实现。

Goal：公开goals.disarm(agent)控制live continuation authority；pause/resume/complete/clear基于GoalRef修改durable phase。它们不需要换preset，但也不等于擦掉Goal提示/历史、取消所有既有物理任务。Goal domain与round driver、command/tool/UI消费者分离；仅隐藏goal tools不会自动disarm已经armed的Goal。
[disarm](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/goal/goal/src/index.ts#L281-L299)；[pause/resume](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/goal/goal/src/index.ts#L342-L369)。

Jobs：JobRegistry公开start/list/get/read/kill/wait及attachController；后者scope-aware，start会拒绝没有controller服务的owner。它不是通用“取消所有现有jobs并隐藏全部通知”开关；kill是请求停止，wait与具体backend负责settlement。移除某tool/controller与producer任务寿命不可等价；既有jobs可继续产生done通知/consumer消息。
[jobs公开操作](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/jobs/jobs/src/index.ts#L74-L143)；[controller scope前置](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/jobs/jobs/src/index.ts#L169-L176)。

Schedule尤须注意own layer：rc.1 schedule plugin为未来发布的root Agents建立ScheduleRuntime，registerScheduleTools(ctx,agent.ctx,agent,...)把工具注册到exact Agent own layer。**因此该Agent的继承工具restrict本身甚至不必隐藏schedule工具。** runtime有自己的定时器、idle/maintenance驱动和durable reminder记录；根公开registerScheduleTools/domain helpers，但ScheduleRuntime只是内部import，不能伪造公开ctx.schedule.pauseAll。已有reminder的取消/管理走其正式工具/domain合同；仅model schema隐藏不停止timer或due followup。
[Schedule真实注册与cleanup](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/schedule/schedule/src/index.ts#L15-L74)；[Schedule生命周期说明](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/schedule/schedule/README.md#L115-L123)。

Subagent：发起时可传toolFilter，要求provider能力支持；该filter约束child继承工具，但child own reporting/structured-output保留。continuable child有独立activation、消息/interrupt/dispose owner；父Agent隐藏subagent工具不取消既有child或其后续通知，也不能推断新parent policy自动重新写入所有child起始filter。要按Agent/turn改变发布或运行能力，须由delegation consumer/对应child owner履行；不需要把这些误表述成preset必须重建。
[subagent toolFilter](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/subagent/subagent/src/types.ts#L139-L156)。

systemPrompt.section/context/providers可scope注册与dispose；suppressRuntimeContext只抑制dynamic context输出，明文不改变提供那些事实的service/权限。它也不删已进入Session的消息或关闭tool-skill的pre-step注入。
[context suppression限制](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/system-prompt/src/index.ts#L478-L489)。

3. N个tool-capable step + 至多一个无工具finalization

VERIFIED_INFERENCE/条件性候选：
同一个AgentLoop已有agent/pre-step（可reject或替换入步messages）、system-prompt/assemble（可变换最终assembly）、tools.restrict/guard/pre-execute、inbox、turn-stopping等公开原语。因而“由第一方policy维护预算、复用同一loop的入步及输出接缝”存在接口层候选，不必另造第二loop；**但没有查到rc.1现成maxSteps或finalAnswer-policy，不能称为开箱合同或已通过组合验收**。

必须区分：
- agent/pre-step可限制“允许进入的逻辑step数”；reject使turn blocked，不会自动生成final answer。step/start只在enter之后写；同step model retry不是额外step，故最多N+1 steps不等于最多N+1 physical requests。
- 在第N+1拟议step的pre-step才改mask，该步assembly已存在，不能自动变为无工具。需要在收集前policy已经生效，或由公开assembly后处理决定最终tools；后者是projection变换，不会撤回先前schema/context消费。pre-step返回并无公开assembly字段，agent/request也不能改tool schemas。
- “Native tools数组为空”和“完全无工具能力”不同：own tools/run_code豁免、PTC SDK文字/已有历史、模型自行产出tool-call，以及直接service路径都不能由简单继承mask排除；实际execution还受ToolRuntime策略。一个无tools声明不是模型一定只产文本的证明。
- 标准loop只在该step无tool calls或concludesTurn等完成条件后停止；tool结果concludesTurn不是“再调用一次无工具模型”的指令。fresh steering/additionalContexts会令loop继续拟议下一步；第一方上限policy若拒绝它，得到blocked而非保证已有final answer成功完成。
- async provider retry、compaction auxiliary requests、finalization被取消/拒绝或max-tokens，都可能导致没有合格final文本；至多一个finalization逻辑step不是“必有最终答案”。
[pre-step/step入步与循环](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts#L272-L338)；[同step retry及完成判断](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts#L341-L436)；[pre-step/request类型](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent/src/runtime-types.ts#L227-L251)；[assembly最终结果](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/system-prompt/src/index.ts#L588-L610)。
UNKNOWN：严格N+最多1 finalization对完整默认consumer组合、并发steering、各mode与retry的闭环，本次未运行conformance，不能据接口存在宣告全部保证。

4. steer/inject/inbox观察与veto

VERIFIED_INFERENCE：
Agent.steer→send(input,'next-step',true)，inject→send(input,'next-step',false)，followup→next-turn；send同步调用Inbox.splice后才wake。Inbox.mutate先validate，再Session.append('agent/inbox/spliced', {target,start,removedCount?,inserted,outcome?})，随后更新live数组，最后同步通知discarded/inserted。claim由loop内部调用，删除后发claimed；claim本身标internal，不是plugin主动控制入口。
[inbox mutation精确顺序](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent/src/inbox.ts#L128-L145)；[先commit再live通知](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent/src/inbox.ts#L157-L192)；[Agent send系列](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts#L122-L149)。

公开可观察事件：
- session/event中agent/inbox/spliced：含Session/event seq、target、规范化坐标与inserted；回调时事件已commit而live inbox数组尚未splice，可用于精确关联变化。
- agent/inbox/inserted {agent,message}、discarded {agent,message}、claimed {agent,message,turn}均scope-filtered emit。replace可产生discard+insert；claim的删除不等于discard，reject后的claimed message不会再作为user/message发布。
这些同步通知可以成为第一方“之前的健康/授权观察已失效”的触发事实；但没有一次原子远端撤权ack，更不会撤回已开始的physical effect。agent inbox事件不区分每种调用者方法，target语义应结合durable splice；其它上下文/配置变化也未被这些事件穷尽。
[事件公开类型](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent/src/runtime-types.ts#L186-L212)。

它们都是observe-only，不是可veto的pre-steer hook。Agent dispatcher明确容纳同步throw/异步reject，只warn；Session.event也在append commit后派发。throw不能阻止既定splice，异步Promise也不被当成admission等待。不能把从observer再删inbox或cancel的补偿行为叫“原子拒绝steering”。
[Agent事件错误容纳](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent/src/dispatch.ts#L120-L135)。
没有找到统一、位于所有Agent.send/steer/inject/Inbox直接mutation之前且可await/veto的公开hook。agent/pre-step能在未来claim后拒绝拟议step或筛选messages，但已经过inbox admission与assembly，不能冒充统一steering入口闸。此结论只限所核公开面，不把任意外部输入策略设计带入本答复。

依据：0.1.2-rc.1 / a66e4702047846cdaa10c66c9d3df3951f5ea70d；所有关键源码/测试链接固定该SHA。未读取调用方项目、未运行模型或conformance。
