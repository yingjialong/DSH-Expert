---
title: rc.2 tools/result 异步观察、owner drain 与按操作解析凭据
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/core/tools/src/index.ts#ToolRuntime.notifyResult
  - packages/core/tools/tests/scoped.spec.ts
  - packages/core/agent/src/runtime-types.ts#Agent.whenIdle
  - packages/core/agent-loop/src/agent.ts#whenIdle
  - packages/core/agent-loop/src/tool-calls.ts#appendToolResult
  - vendor/cordis/src/index.ts
  - vendor/cordis/src/events.ts#EventsService.register
  - vendor/cordis/src/fiber.ts#Fiber.effect
  - packages/credentials/credentials/src/index.ts#CredentialProvider.resolve
  - packages/credentials/credentials/src/types.ts#CredentialRef
  - packages/llm/llm-pi-ai/src/index.ts#resolveApiKey
  - packages/llm/llm-pi-ai/tests/dynamic-config.spec.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-05
asked_by: agent
---

## 条件与版本

八维坐标：TS/JS · DSH plugin/adapter 与外部物理执行方 · 每次工具执行与 owner 工作分别管理 · 文件系统形态不影响本结论 · Session preset/static ToolDefinition 不变 · 每 operation 解析凭据且模型不直接消费 secret · live Host，外部执行可跨进程 · `dsh-v0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`。

2026-09-05 远端 tag 与本地对象一致；本页全部证据按完整 SHA 读取，不随共享工作树或默认基线变化。该 SHA 的 vendored `@deepseek-ai/cordis` 标识为 `4.0.1`。以下为类型/实现直接核验及条件性推论，统一 `verified_inference`；未执行测试或真实模型调用。

## tools/result 不等待异步 listener

公开事件签名返回 `undefined`，mode 为 `emit`。`ToolRuntime.notifyResult` 同步调用 observer，然后只执行 `void Promise.resolve(returned).catch(reportFailure)`；调用方随后得到已经物化的 finalResult。同步 throw/异步 reject 仅 warn，不改变结果。运行时容纳 Promise 不等于公开异步完成合同。[声明](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L190-L197) · [实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L1633-L1676)。

一方测试以 `Promise.reject(...) as never` 验证 observer 失败容纳、后续 observer 仍执行、emit 模式与 warn；本次只阅读测试源码，未把它计为异步撤权实测。[测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/tests/scoped.spec.ts#L677-L717)。

`tools/result` 是 live notification，不是 durable `tool/result` 或撤权回执，不覆盖进程崩溃、never-settling body、observer 移除后的外部清理。

## adapter 自管 drain 的公开组合边界

- `ctx.on` 的注册由 Fiber effect 拥有，disposer 移除 listener，不自动追踪已经启动的后台 Promise。[listener ownership](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/vendor/cordis/src/events.ts#L245-L259)。
- `ctx.effect` 接受异步 disposer，Fiber 卸载等待其 settlement。adapter 可自行保有撤权 Promise，并由自己的 whenIdle/disposer 等待；这是已有公开原语的可组合性，不是 DSH 提供的通用 revoke coordinator。[effect](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/vendor/cordis/src/fiber.ts#L403-L441)。
- `Agent.whenIdle` 只跟随其 driver/maintenance 的 activityDone，不会发现任意插件上的同名方法，也不自动等待 tools/result observer 的后台工作。[公共合同](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent/src/runtime-types.ts#L87-L104) · [实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/agent.ts#L195-L200)。
- 同一 Fiber 的独立 effects 在 `_unload` 中以 Promise.all 并行处理；单个 composite effect 内才按逆序串接。`_unload` 捕获 disposer 错误并记录 logger.error，因此“Fiber 已卸载”不单独证明撤权成功；成功/失败仍需由撤权 owner 表达。[卸载实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/vendor/cordis/src/fiber.ts#L675-L695)。

条件性推论：若撤权属于某次 `ToolDefinition.execute` 所拥有 operation 的完成语义，其公开合同要求 owned work 到 quiescence 后才 settle，不能把它变成 result 后无人等待的 Promise。若是独立后置 owner 工作，可以自行追踪等待，但工具完成、Agent idle 与物理撤权完成仍是不同观察点。`tools/execute` 和 `tools/post-execute` 是可 await 的 waterfall，不改变 result 的同步合同。[执行/扩展点合同](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L153-L235)。

## 每 operation 解析 credential 的合同与限制

`@deepseek-ai/dsh-credentials` 根公开 `CredentialProvider`、`credentialRef`、`ResolvedCredential` 等。`resolve(ref)` 返回 `{value, source}` 或 undefined，明确要求每 operation 重读、不跨 operation 缓存，使变更在下一 operation 生效且无须重启 plugin；`describe(ref)` 只返回 configured/source/writable。[公开合同](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/credentials/credentials/src/index.ts#L114-L198)。

DSH `CredentialRef` 是 POSIX 环境变量名的 branded string，constructor 校验 `/^[A-Za-z_][A-Za-z0-9_]*$/`；不是任意外部存储 locator 或授权 capability。resolve 只接 ref，没有 operation identity/Agent/token 参数，不替外部 owner 校验授权，也不把解析、物理 I/O 与 concurrent rotation/revoke 变为原子事务。[ref grammar](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/credentials/credentials/src/index.ts#L14-L43)。

`llm-pi-ai.resolveApiKey` 在 operation 中调用 resolve；一方测试保持同一配置，更新 ref 对应值后下一 mock request 使用新值。此证据只覆盖该 LLM adapter，不能扩展为所有工具/MCP adapter 已接入凭据 seam。[解析实现](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm-pi-ai/src/index.ts#L166-L188) · [rotation 测试源码](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/llm/llm-pi-ai/tests/dynamic-config.spec.ts#L148-L163)。

条件性推论：若凭据解析和物理 I/O 确由外部 owner 完成，DSH 调用面继续接收既定参数与结果，单纯凭据轮换不要求新增 DSH API 或替换 Session preset/static ToolDefinition。外部 opaque ref 是 adapter/carrier 自己的协议，不自动等同 DSH CredentialRef。`ToolExecution.token` 是 process-local symbol，只支持 live 精确执行关联，不是跨进程 credential generation 或 durable revoke receipt。[ToolExecution/ToolRunContext](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L304-L411)。

“DSH 只看无 secret 的数据”必须由外部实现证明。ToolRuntime 可见 arguments、canonical value、content、error/meta、additionalContexts；JSON/freeze 校验不识别或自动删除 secret。AgentLoop 把 content、error.info、meta 写入 durable tool/result；canonical value 虽不直接写入该事件，仍执行期可见。调用 CredentialProvider.resolve 的代码会拿到真实 value。[结果类型](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/tools/src/index.ts#L555-L580) · [日志投影](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/core/agent-loop/src/tool-calls.ts#L268-L288)。

外部 current identity、ref 映射权限、rotation 与在途 I/O 的交错、撤权错误传播、实际 secret 隔离均不由这些 DSH 类型规定；本页不以未验证假设补齐，也不评价具体宿主实现。
