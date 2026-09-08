---
title: rc.1 MCP resync、结果投影与关闭边界
description: 两阶段 catalog 更新、authority 缺口、SDK 请求取消和 transport 生命周期的区分。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/mcp/mcp-client/src/index.ts
  - packages/mcp/mcp-client/src/connection.ts
  - packages/mcp/mcp-client/src/tools.ts
  - packages/mcp/mcp-client/src/transport.ts
  - packages/mcp/mcp-client/package.json
  - packages/mcp/mcp-client/README.md
  - packages/mcp/mcp-client/tests/apply.spec.ts
  - packages/mcp/mcp-client/tests/mcp-client.spec.ts
  - packages/mcp/mcp-client/tests/reconnect.spec.ts
  - packages/core/tools/src/index.ts
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/packages/mcp]]"
  - "[[wiki/topics/rc1-programmatic-tool-turn-lifecycle]]"
---

条件：官方 MCP plugin · stdio/Streamable HTTP · 同 scope namespace · 原 ToolRuntime/AgentLoop · 无密钥模型调用 · SDK1.30.0 · 物理效果另有 owner · DSH rc.1 `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。

## Catalog 与 authority

- raw client.request 拉 tools/list 分页，不用 SDK listTools 的逐页 output cache；直至 nextCursor 为 falsy，无本桥总页数/循环 cursor/整体发现 deadline。
- fetch/build 阶段失败保留旧注册；全列表成功才 dispose 旧代、全量 register 新代。注册冲突撤销本次新项、留下零工具，不恢复旧代；strict initial 可抛，后续 contain。
- list_changed 在 connect 前安装，全部同步经 syncChain；开始时检查 current generation。相同 schema 也重建闭包/重注册，不是差异比较。
- definition 闭包捕获该代 Client/rawName；已经进入 body 的调用仍用它，未进入 body 的 ToolRuntime call 在 approval/guard 后还按 name/scope 查当前定义。schema snapshot、callId 或批准不 pin definition/authority。
- reconnect期间可保留旧注册而闭包指向失效 Client；reconnect=false 不注销，也不禁 list_changed。无 freeze/veto/admission-close/review-generation 公共口。serverName 唯一只防同 scope/root namespace 重复，不是远端身份或同职责唯一保证。

## 结果范围

- inputSchema 原样作 parameters，raw ToolDefinition 不自动通过 defineTool 校验全部输入。MCP server 承担输入语义；非对象参数退空对象。
- McpResult 保留 content 与可选 structuredContent。支持的 advertised output schema 由 ToolRuntime 要求/验证 structuredContent；不支持 schema 退 JsonValue，而非全 JSON Schema 支持。server isError 转工具错误。
- 图片需 exact route 声明 image、attachments service、有效图片批量准入后才进入 Native durable refs。拒绝 rich projection 时保留 raw canonical value并给文字诊断；audio/embedded resource 不作 rich model内容，resource_link 只留 name/URI。
- taskSupport=required 在 execute 发 tools/call 前拒绝，非 catalog 过滤；不实现 task lifecycle。Resources/Prompts 无桥接 consumer。
- canonical raw value 可供程序调用者，不等于全部自动进入 Session；标准工具日志由 AgentLoop 记录模型 projection。

## 取消、timeout、dispose

- tools/call 传 exec.signal 与 timeout（默认60000ms）；initialize/list 每 request 用 SDK 默认60000ms，不是整个分页的总期限。
- SDK1.30 cancel/timeout 删除 response/progress handlers、清timer、异步通知 cancelled 后 reject，不等待 server 停止 ACK。本地 Promise 结束不证明远端效果结束/未发生。
- supervisor 默认500ms指数退避、上限30000ms、maxAttempts10；连接需稳定至少 maxDelayMs 才重置 outage 预算。耗尽排队注销，reload/restart 才恢复。HTTP request failure 不必然触发 transport onclose 重连。
- failed-generation close 后等 onclose 至多额外5000ms，失败停止重连；dispose 则清timer/关闭current，close timeout记 incomplete，随后等 connection attempt/syncChain 并注销。无所有远端工具物理效果的 drain receipt或总deadline。
- SDK1.30 stdio.close 为 EOF→最多2秒→根child TERM→最多2秒→KILL，KILL后不再等close；DSH额外close等待不是进程树保证。stdio spawn 属于 SDK，不走 ctx.subprocess，仅复用 env scrub。
- SDK HTTP.close abort本地controller/清timer/onclose，不停止server进程。正常ToolRuntime取消收敛只针对其 awaited工作，不能把SDK已reject后的远端执行纳入保证。

## 正式公共面

dsh-mcp-client rc.1 root JS 仅 Config/apply/inject/name；声明另有 transport Config、McpResult、ReconnectConfig/ResolvedReconnectPolicy。startConnection/ConnectionHandle/createTransport/syncTools/publicToolName/definition与rich helpers 未从 root 导出，不能用私有源码路径组装另一桥。

可复用的独立面是 DSH ToolDefinition/ToolRuntime/approval/guard/Session 记录契约，及 MCP SDK 自己的 Client/Transport/Protocol。它们不自动实现自有桥的 catalog/authority 冻结语义。

证据：[sync](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/mcp/mcp-client/src/tools.ts#L122)、[supervisor](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/mcp/mcp-client/src/connection.ts#L150)、[output/task](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/mcp/mcp-client/src/tools.ts#L278)、[dispose](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/mcp/mcp-client/src/connection.ts#L324)、[SDK spawn ownership](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/mcp/mcp-client/src/transport.ts#L1)。

固定tag核对，精确plugin tarball root declarations/JS exports与SDK1.30 dist/esm/shared/protocol.js、client/stdio.js、client/streamableHttp.js在内存只读核验；上游apply/output/reconnect测试仅读断言，未运行synthetic/E2E，不评价外部项目。
