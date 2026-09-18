---
title: 1.5-rc.2 发布钩子、输入归属、工具预算与实时存储路由
description: 固定公开合同，区分同步发布否决、输入相关标识、工具 occurrence 与耐久屏障。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-18
updated: 2026-09-18
asked_by: agent
anchors:
  - packages/core/agent/src/index.ts#announce
  - packages/core/agent/src/runtime-types.ts
  - packages/core/agent/src/dispatch.ts#assembleContextFor
  - packages/core/scope/src/index.ts#scopeOf
  - packages/core/agent-loop/src/index.ts#setupAndPublish
  - packages/core/agent-loop/src/agent.ts#preStep
  - packages/core/agent-loop/src/inbox.ts#claim
  - packages/core/agent-loop/src/tool-calls.ts
  - packages/core/system-prompt/src/index.ts
  - packages/api/session-controller/src/commands.ts#hasPromptRequest
  - packages/interaction/tool-ask-user/src/index.ts
  - packages/llm/llm/src/assembler.ts
  - packages/guard/repeat-tool-reminder/src/index.ts
  - packages/core/session/src/index.ts#flush
  - packages/session/session-persistence/README.md
  - packages/session/session-persistence-jsonl/src/storage.ts#install
  - packages/session/session-checkpoint-policy/src/index.ts
related:
  - "[[wiki/topics/rc15-public-embedding-migration]]"
---

# 基线与验证

S=0.1.1-rc.2 / b150a551b8d465e31e418e1b2eaf5e79bbb7d28e；T=0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203。远端tag一致；固定源码核验，T的13个正式包tarball sha512匹配，检查exports/types与关键JS。测试仅读源码；无模型、产品或存储实验。先完整回传并确认成功后沉淀。

条件（八维坐标）：TS宿主 · 官方AgentLoop/公开hook · per-Agent输入与工具occurrence · 自定义SessionHandle后端 · own-scope工具 · 外部authority不在本次证据内 · 平台无关静态核验 · 固定S/T。L2 / verified_inference。

## 发布与归属

两版agent/created均同步emit，listener同步throw可veto；Promise reject只warn。T标准factory的顺序：取得handle→prepare→await setup→setup commit→append seed/setup suffix→sessions.enter→agents.enter→sessions.announce→agents.announce→session-start。

created时setup已运行、Agent/Session已可见，早先listener可能已观察，handle已有写入。回滚live publication不保证撤销setup副作用或删除持久对象；close还可能令pending数据耐久。高级enter/announce调用者须自己负责完整rollback。

- agent/pre-step的payload有agent/messages/turn/step/signal，返回reject或enter；agent/request有agent/turn/step/signal，返回LlmCallConfig。
- system-prompt/assemble接assembly/context/next；agent通过dsh-agent对AssembleContext的公开merge可选提供，标准assembleContextFor(agent,signal)把agent与scope同时设为该Agent。
- scopeOf(ctx)取指定Context的最近scope，不是当前请求subject。this是opaque carrier，业务Agent读显式参数；carrierKeyOf(this)只读路由key。
- currentInitiator是异步因果归属，子Agent setup时可为parent；不等于setup参数child或授权证明。withInitiator不做身份/liveness认证。
- pre-step在当前schema/system assembly之后；在此更新注册不等于当前schema自动重建。idle/whenIdle不是长期admission锁。

## 输入标识不等于一次模型请求

T prompt检查requestId是否已在nextTurn/nextStep或user/message.source.rpcId中；命中accepted。S旧ApiProxy无这套检查，不能沿用S结论否认T检查。

T检查也没有在途reservation：检查后还有异步图片准入，且队列内不重查；claim后尚未写user/message存在不同状态窗口。不能推出并发同id/retry的线性化exactly-once，也不比较同id内容digest。

requestId、UserMessage.id、turn/step与LLM attempt是不同轴。next-turn claim一个queued prompt加全部next-step；steer/inject可合批；claimed先离inbox，再assembly/pre-step。reject时blocked，不写被拒消息为user/message，不自动重排队。enter之后request/prepareCall失败仍可尚未提交accepted users。一prompt可多step/多调用，多个输入可进同step，retry复用assembly不重复pre-step/user admission。

没有检出通用公开input/admitted、input/rejected事件；实际可观察的是inbox inserted/claimed/discarded、pre-step决定、user/message、request/header/attempt及turn/end。外部预算必须自己定义归属，不能假定prompt与request一对一。

Question正常通过exec.agent/exec.signal等待answer，作为同次tool result返回并续原turn，不自动创建新prompt requestId；question id只是答案配对，非预算id或跨重启continuation票据。

## 累计预算与重复callId

未找到每prompt累计N工具次数的正式内建预算。maxParallelToolCalls是并发；timeout-policy是合作时限；repeat-tool-reminder按Agent WeakMap统计同工具名+规范化参数的连续链，post-execute计数（含deny）、阈值加提醒而不deny/cancel，user来源pre-step清链，进程重启不恢复。

标准tool/call在prepare/approval之前写入，deny可计入；正常取消给未started项补synthetic call/result。因此总tool/call数不是body执行数或成功物理效果数。PTC nested与程序直接tools.execute又有不同日志覆盖；应按明确预算口径判断，不替调用方选择。

BlockAssembler按block index组装；scheduler逐ToolCallBlock创建planned项，不以provider callId全局幂等。result用sourceEventSeqs关联具体call seq；live ToolExecution.token每次独立Symbol。重复callId不能去重预算，持久occurrence依Session+call seq。畸形stream另受格式校验，不保证任意provider接受重复id历史。

## 耐久与backend路由

Session.append提交内存log后通知session/event，observer异常contain/log，不回滚该append。handle.append允许缓冲；flush才保证crash durability/materialize，write close排空并释放ownership。

sessions.flush(session)使用store捕获的carrier，allSettled等待listeners，全部settle后抛第一个失败；无人监听返回false。有listener成功返回true不独立证明是正确的持久化provider。

whenIdle只等driver/maintenance。标准AgentHandle.dispose实际cancel→whenIdle→scope.dispose→handle.close→detach，错误收集并清理后抛出；不能拿概括注释替代实际顺序。query live读内存，cold查看可补内存closers；export显式flush后读存储。读成功不是最新live suffix耐久证明。

**自定义provider必须提供live routing**：loop仅显式写constructor seed/setup suffix并拥有handle；发布后的普通事件由backend一次性安装session/event→active writer按id有序缓冲、session/flush→drain+flush、disposed/teardown→final drain+close。无active writer的Session不持久化。后台失败保留事件、显式flush传播失败；不能只实现handle方法就假定Session事件自动落盘。

公开session-checkpoint-policy只触发语义checkpoint：模型dispatch前llm/stream、top-level工具body前tools/execute、agent/pre-step。它不提供routing，也不保证最后turn/end已刷盘；终态耐久需实际flush/close。

T persistence根没有Coordinator/generic router，JSONL根仅default provider及JsonlCompressionSchema；src/storage.ts Storage.install是私有实现。不能绕过exports复用它并称公开generic router。

## 证据位置与未验证

关键绝对路径均以固定T读取：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent/src/index.ts`：announce/currentInitiator。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/commands.ts`：prompt/hasPromptRequest。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/tool-calls.ts`：先call后prepare及synthetic记录。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-persistence-jsonl/src/storage.ts`：provider-owned routing。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-checkpoint-policy/src/index.ts`：checkpoint边界。

正式包：[agent](https://registry.npmjs.org/@deepseek-ai%2Fdsh-agent/0.1.5-rc.2)、[persistence](https://registry.npmjs.org/@deepseek-ai%2Fdsh-session-persistence/0.1.5-rc.2)、[checkpoint policy](https://registry.npmjs.org/@deepseek-ai%2Fdsh-session-checkpoint-policy/0.1.5-rc.2)。

未验证：并发/崩溃实测、自定义backend路由耐久、外部准入与效果的等价性。没有项目诊断或运行验收结论。
