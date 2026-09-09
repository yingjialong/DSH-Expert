---
title: rc.2 History 消息分页与响应大小边界
description: 缩页重试的连续性、tail projections 以及不能保证字节上限的结构。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/host/apiproxy/src/api-proxy.ts#paginate
  - packages/host/apiproxy/src/api-proxy.ts#historyCutOf
  - packages/host/apiproxy/src/api/sessions.ts
  - packages/host/apiproxy/src/api/sessions.schema.ts
  - packages/client/runtime/src/client/sessions/session.ts#loadOlder
  - packages/host/apiproxy/tests/api-proxy-view.spec.ts
  - packages/host/apiproxy/tests/api-proxy-projections.spec.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-09
updated: 2026-09-09
asked_by: agent
---

# rc.2 History 消息分页与响应大小边界

条件：官方 Client + Host history API · live/cold Session · 消息与tool历史 · 载体另有大小约束 · 无模型调用 · 固定0.1.1-rc.2。以下静态源码/正式types核验为verified_inference，未运行分页实验或外部transport。

## 公开页合同

Client 内部 PAGE_MESSAGES=50，用于open/loadOlder/resync/stitch；SessionOptions/SessionFace无page-size设置，常量不是setter。底层history请求允许sessionId/beforeSeq?/maxMessages?，后者为正整数。返回events:HistoryEntry[]、hasMore、可选projections。没有独立firstSeq/nextCursor/replayState/projectionSeed字段；first seq由首event取得。

paginate取seq<beforeSeq的window（无beforeSeq取tail），向后计append-origin消息，达到quota后以该消息及sourceEventSeqs中最早seq为cut，返回完整连续raw range；hasMore=cut>0。它不要求整turn同页。

同sessionId/beforeSeq递减maxMessages重新调用并只采用最后完整响应，保持该次官方消息页合同；前提为日志身份/连续性未被破坏，未拼接不同响应。older页尾应为beforeSeq-1，Client loadOlder检查tail.seq+1===baseSeq，成功后使用新首event.seq继续。不能沿用首次失败尝试的假定首seq/hasMore。

tail重试是新的观察，不锁住首次snapshot。每次handler的source/preset相关await后，events与projections同步取同一cut；无beforeSeq时带完整projection baseline（若registry存在），older不带。降quota不将baseline改成只描述本页消息，也不保证重试响应严格更小。新增beforeSeq会把tail请求变为older，改变基线语义。

## maxMessages=1仍可超限

- 单消息/工具结果无字节quota；provenance sources可把cut拉回很远，带入许多chunks/raw事件。
- replacement不计消息quota；compaction/summary等区间内记录保留。未完成partial尾部可增长；无足够计数消息时cut=0，整个window返回。
- presenter view可很大；tail projections的全values/asOfSeq不随quota缩小。
- 冷重读仍可挂载standing并运行presenter/projection逻辑；业务只读不等于零插件执行/成本。

上游api-proxy-view.spec.ts:288–327明确maxMessages1返回128个chunk加message；replacement测试:245起证明不计quota。测试仅读未运行，不作任意大小保证。

## 裁切与载体边界

本版history无maxBytes、byte range、streamed-history、includeView=false或projection-key选择。透明压缩/分块后还原同一完整RPC对象属于载体语义，不改变history为消息分页，也不保证载体自己的大小门槛能满足。

view可选且缺presenter会generic fallback，缺view是合法degraded形状；但宿主删view不是原完整响应，也无请求开关要求该降级，不保证同样渲染或解决raw/projection过大。公开presenter可生成较小view属于producer变化。

不可任意裁raw内容/sourceEventSeqs/surface metadata/seq/hasMore并仍称标准连续历史；不能删除projections/keys、改asOfSeq或拼旧baseline并声称保留tail合同。可选字段不代表可无损删除。maxMessages1仍超载体上限时，上游没有保证所有历史可缩入该界限的公开history面。

## 证据

[分页实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L218)、[同步cut](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api-proxy.ts#L1504)、[公开types](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/host/apiproxy/src/api/sessions.ts#L268)、[Client连续性](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/client/runtime/src/client/sessions/session.ts#L379)。

正式包为@deepseek-ai/dsh-host-apiproxy@0.1.1-rc.2，./api导出的history types与源码一致；dsh-client-runtime正式Session types无page配置。tarball内存读取无安装。镜像根经核实 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness`，历史用git show固定SHA。
