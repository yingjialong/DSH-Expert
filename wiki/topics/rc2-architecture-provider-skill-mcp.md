---
title: rc.2 核心职责、I/O provider 与 Skill/MCP 扩展归属
description: 固定旧版本，区分公开接缝可组合性与具体宿主运行等价性。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-18
updated: 2026-09-18
asked_by: agent
anchors:
  - packages/core/agent-loop/src/index.ts#AgentLoop.create
  - packages/core/agent-loop/src/agent.ts#preStep
  - packages/core/session/src/index.ts
  - packages/session/session-persistence/src/index.ts
  - packages/workspace/workspace/src/index.ts
  - packages/session/session-projection/src/index.ts
  - packages/core/tools/src/index.ts
  - packages/llm/llm/src/index.ts
  - packages/settings/settings/src/index.ts
  - packages/credentials/credentials/src/index.ts
  - packages/shell/shell/src/index.ts
  - packages/fs/fs/src/index.ts
  - packages/skill/skill/src/index.ts
  - packages/skill/tool-skill/src/index.ts
  - packages/mcp/mcp-client/src/index.ts
  - packages/mcp/mcp-client/src/tools.ts
  - packages/mcp/mcp-client/src/transport.ts
  - packages/client/connection/src/client/index.ts#ClientTransportHooks
related:
  - "[[wiki/topics/rc2-tool-result-drain-credentials]]"
  - "[[wiki/topics/rc2-agent-request-attempt-schema]]"
---

固定0.1.1-rc.2 / b150a551b8d465e31e418e1b2eaf5e79bbb7d28e，远端tag一致。17个正式npm包sha512/exports/声明核验；未运行Host/模型或检查外部项目。完整回传成功后沉淀；L2 / verified_inference。

条件（八维坐标）：TS宿主 · 自定义Loader/Client carrier · per-Agent scope · 外部I/O与存储provider · 官方调度与工具consumer · secret owner另有约束 · 跨进程行为未实测 · 固定rc.2。

## 版本限定的职责

- rc.2 host-apiproxy/client-runtime是正式有效包；ClientTransportHooks为createApiClient():IApiClient、fetch:RpcFetch、可选loadBundle，不含后续openStream/ownsHost合同。
- AgentLoop.create同步返回Agent。标准链是inbox→turn/start→claim→system/tool assembly→pre-step→step/start与user/message→request/LLM→assistant/tool日志→续step或turn/end；不是1.5的V3系统消息序列。
- Agent持有live driver/cancel/idle/scope；Session持有append-only事件、消息surface/deriveMessages；SessionStore管身份与生命周期；projection按事件fold派生state/view，不是独立权威历史。
- rc.2 SessionPersistence公开Coordinator/Backend及create/append/load等旧合同，不是新版SessionHandle。StorageBackend管理其它domain/KV，与Session存储没有自动跨介质事务。
- Workspace管理目录、标题、排序与Session归属；create/attach直接Host realpath/stat，替换ctx.fs不自动使远端SSH路径成为Host anchor。
- ToolRuntime负责scope可见性/schema/pre/guard/approval/execute/post/result；标准AgentLoop负责模型tool/call/result及排序。程序直接execute不自动等于模型日志pair。
- approval仅allowed-once授予当次ask，要求open turn/audit；不是generation pin或外部效果永久授权。
- LlmRuntime注册provider/adapter、resolve模型能力与prepared call；adapter管协议/认证；providerRetryPolicy、agent/request-error与llm/stream等有不同retry位置，不能用一个maxRetries配置概括全部层。

## I/O替换的条件性可组合结论

正式公开SettingsProvider、CredentialProvider、ShellExecutor、FileSystem、SessionPersistence/Backend、Storage backend facets、SkillProvider、ToolDefinition/ToolRuntime.register。保留driver/consumer而替换provider可复用调度链，但须兑现signal、错误/结果形状、执行世界、生命周期、耐久等合同；不是任意替换自动等价。

Settings基类仍管namespace/schema/变化；CredentialProvider返回secret且按operation解析；ShellExecutor为resolve/run/start；FileSystem负责target身份与文件操作。shell不等于全部subprocess，terminal另有provider，grep/glob可能走subprocess；Workspace/Skill本地watcher/MCP SDK还有独立I/O。不能把一个ctx.fs或shell override称为全Host物理authority证明。

## Skill

SkillProvider提供name/list/get，list可返回{candidates,complete}；registerProvider factory拿control.signal/invalidate。provider拥有发现、认证、读取与opaque locator；Registry拥有scope分层、同名优先级、缓存、list/snapshot/get与规范化。scope shadow不是deny过滤，locator不是DSH认证的immutable bytes。

官方tool-skill是consumer：注册skill工具；按exec.agent/cwd/signal查registry，校验modelInvocable，renderSkillContent输出正文，再由标准AgentLoop写result。pre-step catalog与该插件exact tool definition可见性匹配。

用户source.kind=user的首行/<name>是独立显式调用入口，按userInvocable加载并追加skill-invocation instructions，不等于Commands registry。自定义provider可复用这些官方consumer；resourceBase只是资源指引，不自动搬运或执行资源、不授fs权限。

## MCP归属

rc.2确有上游dsh-mcp-client，root运行时仅Config/apply/inject/name，stdio/Streamable HTTP配置。内部discover/call→ToolDefinition→ctx.tools.register；serverName按root预留。stdio由MCP SDK spawn，HTTP由SDK transport连接；无公开carrier factory/外部Client注入/自动CredentialProvider resolver。

out-of-tree MCP桥把发现结果转为公开ToolDefinition并注册同一ToolRuntime，是合法工具扩展面，可复用DSH policy/调度。MCP transport、catalog更新、authority和物理cancel由桥承担；不能因此称为DSH官方mcp-client的公开transport替换。工具definition/schema与body代际一致性、refcount/drain、远端exactly-once不是由register自动提供。

## 证据与限制

所有anchors按固定历史SHA读取；关键绝对位置：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/index.ts`（rc.2 create）。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/connection/src/client/index.ts`（rc.2 hooks）。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/skill/tool-skill/src/index.ts`（consumer）。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/mcp/mcp-client/src/transport.ts`（内部carrier）。

正式包：[旧host-apiproxy](https://registry.npmjs.org/@deepseek-ai%2Fdsh-host-apiproxy/0.1.1-rc.2)、[旧client-runtime](https://registry.npmjs.org/@deepseek-ai%2Fdsh-client-runtime/0.1.1-rc.2)、[skill](https://registry.npmjs.org/@deepseek-ai%2Fdsh-skill/0.1.1-rc.2)。未验证任意外部provider的运行等价性、完整安装/boot、物理排空或安全效果。
