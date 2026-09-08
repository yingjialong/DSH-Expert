---
title: rc.2 batch取消、Agent admission与同步schema value校验
description: rc.2 取消收敛、维护执行权、动态工具贡献及同步校验的公开边界。
type: reference
updated: 2026-09-08
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/core/agent-loop/src/tool-calls.ts
  - packages/core/agent-loop/tests/tool-calls.spec.ts
  - packages/core/agent-loop/src/agent.ts
  - packages/core/agent-loop/src/index.ts
  - packages/core/agent/src/index.ts
  - packages/core/agent/src/runtime-types.ts
  - packages/core/tools/src/index.ts
  - packages/core/tools/src/json-schema.ts
  - packages/core/tools/tests/json-schema.spec.ts
  - packages/core/agent-loop/tests/loop.spec.ts
  - packages/core/agent-loop/tests/scope-lifecycle.spec.ts
  - packages/core/tools/tests/scoped.spec.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-06
asked_by: agent
---

条件：TS/JS · 标准AgentLoop batch · parallel/exclusive调度 · 外部I/O不在结论范围 · standing ToolLayer · 无模型凭据调用 · live Host · 固定rc.2。

结论：取消后正常scheduler会停止补位并为尚未started的调用生成balanced ABORTED_BEFORE_DISPATCH；已started但未body的调用走ToolRuntime取消/错误路径，不保证最终错误一律同码。“所有pending绝不再lookup”不是公开保证。cancel(disposed)+whenIdle不是永久prompt admission关闭；生命周期释放应区分AgentHandle、裸Agent、factory/registry。validateJsonSchemaValue是无预算/无signal的同步函数，不能在同JS线程中途通过普通abort抢占；Worker终止是一种外层隔离能力，不是该API唯一或内建的取消方式。

版本：固定dsh-v0.1.1-rc.2 / b150a551b8d465e31e418e1b2eaf5e79bbb7d28e，远端tag与本地对象一致，全部源码按完整SHA读取，未切换上游。已查知识索引、错误本（含E025/E035）、覆盖度与未解问题，并复核相关固定rc.2专题。以下为本次源码/公开类型/一方测试源码交叉核验，按本库L1/L2封顶统一verified_inference，不升为fact；无假设补结论，未执行runtime测试或物理I/O。

1. batch取消

首先纠正状态口径：标准executeToolCalls不会在batch开始一次性写完所有tool/call。startCall在prepare之前才appendToolCall并started++；真正未进入startCall的pending此时只有assistant中的调用声明，取消收尾才补call+result。如果某调用已经有tool/call，它可能仍在ordered prepare/Approval而没有body，但scheduler已把它计为started。

A. 尚未scheduler-started（parallel池未补位、后续exclusive barrier）：
runGroup以signal.aborted初始化aborted；fillPool只在!aborted时start，prepare/commit/dispatch等待返回后再次采样signal，取消后停止补位；正常abort路径先等待started dispatch并按model顺序commit，然后对group.slice(started)调用appendSkippedToolCall。外层对其余groups再补同样pair并直接return。helper直接append tool/call、tool/result及AbortError/ABORTED_BEFORE_DISPATCH，不进入ToolRuntime prepare/dispatch，因而该synthetic路径不lookup工具、不执行pre/Approval/body，也不发ToolRuntime tools/result。
条件：该标准batch已进入scheduler，正常abort收尾实际运行、started工作settle、post/finalization与Session append等未发生终止scheduler的内部错误。若internal scheduler failure/append failure，catch只drain在途dispatch并抛首错，明确不伪造balanced结果；进程崩溃或never-settling工作同样不能获得完成保证。
证据：[scheduler主循环](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/tool-calls.ts#L66-L96)、[start/pool/abort/failure](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/tool-calls.ts#L128-L240)、[synthetic pair](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/tool-calls.ts#L248-L288)。

B. 已scheduler-started且已写tool/call，仍未body（如await pre-execute/Approval，或around wrapper等待）：
createExecution先读取当前visible definition并捕获finalizeContent，prepare随后检查caller signal；pre/Approval后再检查。body入口先检查融合signal，已abort就返回ABORTED_BEFORE_DISPATCH，位于dispatch-time resolveExecution之前，因此在该检查观察到abort时不进入body lookup/execute。
但不能断言该call最终必为ABORTED_BEFORE_DISPATCH：已有denial、guard拒绝、listener throw、无效参数、post-policy返回的其他error等可保留自身错误；post只在取消且结果仍success时替换为cancellationResult。finalizer与observer也可能已运行。这里不是A的synthetic shortcut。
证据：[createExecution先lookup](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1364-L1449)、[prepare控制流](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1463-L1505)、[body先abort再lookup](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1527-L1555)、[post保留error](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1609-L1619)。

C. body已开始的parallel：
不抛弃tool.execute Promise，合作取消并等其settle；取消时成功结果可改为ABORTED，已经失败的工具可保留其error，不能改称BEFORE_DISPATCH。started dispatch全部settle后按模型顺序写result；与任意外部adapter物理drain不是同一证明。

“pending不再lookup注销/同名replacement”的准确边界：
- 无取消时明确会重查：外层executionMode和fillPool分类读取current registry，一方test专门在exclusive barrier替换工具后证明pending执行replacement。
- 即使signal已abort，外层第一次executionMode也在runGroup读取aborted之前发生，executionMode内部resolveExecution并可调用isConcurrencySafe；createExecution也有早于取消判断的visible lookup。
- 因而只能说正常abort synthetic剩余路径不lookup/执行，及body在自己的abort检查命中时不会继续resolve/execute；不能说从cancel调用起所有相关代码零lookup。也没有definition generation pin/refcount/unregister自动drain合同。
证据：[executionMode](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1276-L1284)、[replacement test](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/tests/tool-calls.spec.ts#L144-L184)、[already-aborted / pre-execute abort tests](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/tests/tool-calls.spec.ts#L460-L522)、[exclusive跳过test](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/tests/tool-calls.spec.ts#L591-L632)。上述测试源码使用user cancel；disposed对活动turn触发同一signal路径的结论由Agent.cancel实现交叉推出，不冒称测试已用disposed实测。

prompt admission / disposal：
- Agent公开cancel/whenIdle/send/followup/steer等，没有Agent.dispose方法。cancel默认清inbox并abort当前activity；idle时不会设置永久disposed状态。活动disposed取消只阻止该aborted activity再latch wake，后来的输入仍可进inbox；回到idle后新followup仍可发起新driver。若signal之前已被别的cause abort，AbortController保持first cause，后续cancel(disposed)不把旧reason改写。
- whenIdle只跟随activityDone直到当前driver/maintenance替换停止，不是长期admission锁。未来再来prompt不在已兑现的idle保证内。
- AgentRegistry.create/resume返回AgentHandle{agent,dispose}，只有持有者有该exact lifecycle释放能力；registry.get返回裸Agent。标准handle.dispose顺序为cancel(disposed)→whenIdle→Agent scope.dispose→detachAgent→detachSession；上游实现明确注释此后继续send是sender错误，并非handle建立全Host所有入口的原子拒绝闸。
- factory provider卸载会drain自己创建的handles，属于更广的结构生命周期；AgentRegistry的closeInitiators/disposeInitiators是private，不能当公开closePromptAdmission或registry.disposeAll。其scope teardown的initiator边界不等于任意外部prompt carrier的协议闸。
因此rc.2没有一个专用公开方法原子关闭“全部相关Agents的一切新prompt/create/resume入口”。公开的exact disposal原语是owned AgentHandle.dispose及owner/factory Fiber生命周期；所有发送来源停止新输入是调用方必须已满足的前提，不能由cancel+idle反推。本答复不设计外部admission实现。
证据：[公开Agent](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent/src/runtime-types.ts#L64-L137)、[cancel/send/wake](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/agent.ts#L113-L200)、[handle ownership](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent/src/index.ts#L158-L174)、[handle实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/index.ts#L497-L518)、[private initiator关闭](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent/src/index.ts#L619-L641)。Agent root重导出runtime-types；ToolRuntime scheduler符号虽可在根找到，标internal，不是plugin扩展入口。

2. validateJsonSchemaValue

dsh-tools根正式导出validateJsonSchemaValue；签名validateJsonSchemaValue(schema:JsonSchemaNode,value:unknown,path='value'):string[]。第三参是错误路径标签，不是预算。接受已assertSupportedJsonSchema的schema；assertObjectJsonSchema通过的object-root冻结schema符合此前置，但冻结不增加取消或资源限制。
实现同步checkValue，显式frames栈while循环，不await、不yield、不读AbortSignal，无maxSteps/maxNodes/depth/time参数。oneOf逐分支对同一value验证、计数并要求exactly one；对象/数组子节点迭代，成功后还做lossless-JSON检查。不是支持$ref的递归schema解释器；schema环被assert拒绝，nested values可遍历，循环/非JSON值返回违规。5000层oneOf value测试证明避开JS递归栈，不证明执行时间/内存有界。
证据：[根导出](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L89-L101)、[value API](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/json-schema.ts#L646-L655)、[frames/oneOf](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/json-schema.ts#L486-L550)、[深层及非JSON测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/tests/json-schema.spec.ts#L395-L416)。

可核验的取消结论：DSH这一个函数没有中途合作取消接口；在它当前同步执行的JS线程里，普通timer、Promise.race超时或AbortSignal事件不能抢占该循环。外部隔离执行环境的终止可停止其所在执行单元，但“只能由Worker取消”不是DSH公共合同——也可能是进程等其他运行环境能力，且DSH不自动把此helper放入Worker。对具体Worker termination机制，本次未核验外部宿主，不报告其保证。仅给schema freeze/输出有界也不等于oneOf分支计算已有预算。
未触及任意插件physical drain，不重复ToolSchema/observer旧议题。
依据：rc.2 / b150a551b8d465e31e418e1b2eaf5e79bbb7d28e，固定commit路径见上；本地对象库 /Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness。

## 2026-09-08 · runMaintenance 与 Agent-local 工具贡献

本节同一固定 rc.2 SHA 复验，asked_by: agent，verified_inference；其他分节不因此刷新 verified_at。

- 正式 Agent 方法是 `runMaintenance<T>(task:(signal:AbortSignal)=>Promise<T>):Promise<T>`，没有查到 betweenTurns。只允许 true idle，同步取得 maintenance phase；turn driver 或另一 maintenance 已占用时同步 throw，不排队。
- 维护期间 public status 仍 idle；不能用 status===idle 判定维护锁空闲。task settle 后 finally 释放，已有 waking input 可立即启动 driver，因此 await runMaintenance 后不保证仍 idle。whenIdle 跟随维护及随后释放的 driver。
- 输入仍可入 Inbox，wake latch 等 task 结束；删除全部 pending 可抑制 replay。cancel 默认清 Inbox/latch、abort task signal，keepInbox 保留 pending；非 disposed cause 后的新 waking input 可以再次 latch。
- 无 Promise.race 强杀、无 task 返回后的强制 throwIfAborted、无注册回滚。任务忽略 signal 可以继续乃至成功；task 失败也会释放 phase 并可唤醒后续输入。
- 存活 Agent 可通过 `agent.ctx.tools.register(definition)` 与 `restrict(filter)` 作局部动态贡献，无 blank 前置；各自返回 exact disposer，归 Agent Context effect，而非归维护 Promise。接口不自动重写 Preset/Header 或写 agent-preset/selected。
- own 工具最后覆盖 inherited（含 standing）同名定义，撤销 own 后可重新露出未受限继承项。同层 duplicate 与 run_code 拒绝；register 不是 replace。
- restrict 过滤 global+ancestor inherited surface，不过滤本 scope 的 own 注册。仅 own 名称无 inherited 对应项时不允许作 filter 名；`{}` 无效，`{allow:[]}` 则明确排除 inherited 工具。多 restriction 交集，各 disposer 只撤销自己。
- cancel signal 不销毁 scope、不自动禁 register；真正 disposed Agent ctx 拒绝新增 effect。标准 owned dispose 等维护/driver quiescence 后才卸 scope。先撤旧再装新没有 all-or-old 事务或自动恢复。
- definition 仍在 body dispatch 时重查；call 初始捕获 finalizeContent/参数快照不 pin 整个 definition。维护只阻止该 Agent driver 同时运行，不锁整个 ToolRuntime，不阻止其他直接调用或独立后台任务。

### 同日补充：agent/created 与显式 schema scope

- `agent/created({agent}):void` 是同步 publication 特例：标准 factory 在 setup/commit、Session/Agent enter、session announcement 之后发它，再做 liveness check 和 session-start。此时同步注册/限制并读取工具 view 是公开能力，但已晚于 setup/import，不能当 pre-mount gate。
- 同步 throw 传播并触发标准创建 rollback；返回 Promise 不被 await，rejection 仅报告。async callback 即使首个 await 前 throw 也是 Promise rejection，不能当同步 veto。
- register/restrict/校验三次调用不自带共同事务；若自行吞掉错误，前面的贡献不自动消失。后续 listener/session-start/动态注册仍可改变 view。
- `schemas(scope?:ScopeKey)` 省略参数是 global view，即使经 `agent.ctx.tools` 调用也如此；Agent 视图要显式 `schemas(agent)`。返回当前 schema 深拷贝，不含完整 definition，不锁定未来请求或注册对象。
- own-only 名称不能用于 restrict；同名也有 inherited 项时可以列入，但只过滤 inherited、own shadow 仍可见。

证据：[agent/created 合同](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent/src/runtime-types.ts#L147)、[announce 同步处理](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent/src/index.ts#L543)、[factory 顺序](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/index.ts#L550)、[schemas 参数](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1229)。本补充先回传来源任务再沉淀，仍为源码级 verified_inference。

证据：[正式维护合同](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent/src/runtime-types.ts#L95)、[实现与 wake](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/agent.ts#L134)、[维护取消/replay 测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/tests/loop.spec.ts#L80)、[inherited/own view](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1130)、[祖先限制测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/tests/scoped.spec.ts#L204)、[inactive ctx 测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/tests/scope-lifecycle.spec.ts#L902)。本次内存读取正式 dsh-agent@0.1.1-rc.2 的 runtime-types.d.ts，确认 runMaintenance；未运行 maintenance+工具更新组合，不把上述原语当成全系统安全更新证明。
