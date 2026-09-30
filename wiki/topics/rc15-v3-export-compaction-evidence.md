---
title: 1.5-rc.2 v3 导出与摘要压缩证据链
description: 区分 Session header、事件统计、模型摘要与无需模型的 surface 剪枝。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-10-01
updated: 2026-10-01
asked_by: agent
anchors:
  - packages/core/session/src/types.ts
  - packages/core/session/src/index.ts
  - packages/core/session/src/surface.ts
  - packages/llm/llm/src/message.ts
  - packages/llm/llm/src/assistant-stream.ts
  - packages/session-query/session-log-export/src/archive.ts
  - packages/session-query/session-log-export/tests/archive.host.spec.ts
  - packages/session/session-stats/src/projection.ts
  - packages/compaction/compaction/src/types.ts
  - packages/compaction/compaction/src/checkpoint.ts
  - packages/compaction/compaction-basic/src/region.ts
  - packages/compaction/compaction-basic/src/summarizer.ts
  - packages/compaction/compaction-basic/tests/compaction-basic.spec.ts
related:
  - "[[wiki/topics/rc15-public-embedding-migration]]"
---

固定 0.1.5-rc.2 / 本页 SHA，远端 tag 一致；已完整回传来源后沉淀。源码与 fixtures 静态交叉核验，未执行测试或真实模型调用，未检查外部项目/日志。

## 导出与统计陷阱

官方 ZIP 的 session.v3.jsonl 首行是 header，使用 createdAt（Unix epoch 毫秒），不含事件 seq/time。随后才逐条输出逻辑事件。archive.ts 部分注释仍称 v2，实际 version 使用 header.version，当前 SESSION_FORMAT_VERSION=3。live 导出前 flush，随后 persistence read handle 读取；不走会合成 closers 的 readColdSessionLog。导出是逻辑日志重序列化，不是底层物理文件原字节复制。

完整事件序列 seq 从 0 连续；按 seq 重放，不按 time 排序/去重。user/message 的 data 直接是 Message；assistant/message 的正文在 data.message.content，另有 stream/usage?/interrupted?。assistant/attempt 记录未落 surface 的尝试。嵌入 stream 的 packed delta 用 time0/dt，应用 expandAssistantStream 解包；不能要求每个 JSON 对象都有 time。

tool/call.data 为 turn/step/callId/name/arguments，arguments 是原始 JSON 字符串。tool/result.data.message 是 user-role 消息，其 content[0] 的 tool-result block 含 toolCallId/content/isError?；data.error 为可选内部错误身份。按 tool/call.seq 区分 occurrence，结合 turn/step/callId 关联；统计执行次数时剔除 replacement 副本，不能把 prune 替代结果计为新执行。

统计口径必须分开：步骤按 step/end，尝试/回答另计；人工输入还须 source.kind=user；全部历史与当前 surface 不同；usage 缺失为未知。header 不占事件序号，fork 继承以 tagged session/end-seed 表达。ZIP 没有 manifest 不等于导出缺失。

## 摘要成功证据链

同一 compactionId：

1. compaction/start：turn 为编号或 null（独立手动事务），可有 sourceCommandId。
2. compaction/summary：summary、shadowedRange、shadowedSeqs、shadowedTokenCount、provider/model，maxTokens/usage 可选。llmStreamCall:true 必须伴随 rawOutput，标记本 context 一次 ctx.llm.stream；未标记不等于绝未调模型。
3. 默认 basic 同步紧接 user/message：source 为 plugin=compact 且带 compactionId；content 为 frameSummary；surfaceOp 为 replace/startSeq/endSeq；sourceEventSeqs 为 start.seq、summary.seq 和 shadowedSeqs。
4. compaction/end：同一身份且无 error。没有通用 success:true；end 单独不是摘要与 replacement 已落地证据。

默认 summarizer 的 purpose=compaction；rawOutput 保留投影前内容，summary 仅保留 text。error/aborted/max-tokens、image 输出或空文本拒绝。rawOutput 中 tool-call 块不代表工具被执行。带 framing 的 checkpoint 必须严格小于被替换区域的 route-priced token 估算，提交前还检查 surface 稳定与工具配对边界。

shadowedTokenCount 是固定 heuristic 价格，不等于 route 价格或 provider usage。LLM 调用完成后仍可能因 shrink/stability 失败而无 summary 记录，故成功记录数不是所有辅助调用/费用的完整计数。

## 重建与证明边界

按 surfaceOp 重放，在 replacement 前按 surface 位置取 start/end 区间并核对 shadowedSeqs，之后该区间替换为新节点，区间外保留。原始事件不删除。可见 seq 经先前替换后可能不递增，start 可大于 end；禁止用数值闭区间代替 shadowedSeqs。

默认摘要输入由 system head、所选节点 deriveEventMessage、最近 request/header.tools 加固定 instruction 构造。可从日志与代码重建内部调用；不是导出中已有 provider wire dump。后续内部 request.messages 来自 deriveMessages；摘要是否完整保存业务约束、后续模型是否遵守，应分别核对，源码不保证语义无损。

可信默认 producer 下，完整链且 llmStreamCall:true 支持“经 DSH LLM seam 生成并落地摘要”。它不是外部 provider 接收/计费的独立凭证，测试 adapter 或 middleware 也可实现该 seam。compaction/prune 明确是 model-free 剪枝；token 下降或任意 replacement 不能证明 LLM 摘要。

## 关键证据

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session-query/session-log-export/src/archive.ts:110`：serializeSessionLog；:151 为持久读取。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session-query/session-log-export/tests/archive.host.spec.ts:233`：header/每事件一行 fixture，未运行。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/session/src/types.ts:321`：消息事件；:462 为 envelope。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/session/src/index.ts:570`：连续 seq 校验。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/llm/llm/src/assistant-stream.ts:202`：expandAssistantStream。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/compaction/compaction/src/types.ts:27`：summary 合同。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/compaction/compaction-basic/src/region.ts:209`：生命周期；:388 为 shrink，:452 为 commit，:528 为输入。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/compaction/compaction-basic/src/summarizer.ts:151`：一次 stream 与输出投影。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/compaction/compaction-basic/tests/compaction-basic.spec.ts:1280`：rawOutput 与调用标记 fixture；:1423 为路由事实，未运行。
