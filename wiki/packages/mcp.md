---
title: packages/mcp — Model Context Protocol 桥接
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/mcp/README.md
  - packages/mcp/mcp-client/README.md
  - packages/mcp/mcp-client/src/index.ts
  - packages/mcp/mcp-client/src/connection.ts
  - packages/mcp/mcp-client/src/tools.ts
  - packages/mcp/mcp-client/src/transport.ts
  - packages/README.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

把 harness 接到 MCP 生态：连接外部 MCP server，把它们的 tool 以 **server 限定名 `mcp__<serverName>__<rawName>`** 注册到 `ctx.tools`，让模型当作原生工具使用 [T1: packages/mcp/mcp-client/README.md]。

## 稳定性

**README 未列出**。`packages/README.md` 的 Hierarchy 表格**漏列了 `mcp/` 组**（同时漏列 `runtime-diagnostics/`），因此该组没有官方的 Release expectation 标注。见「陷阱 1」。

## 包清单

| 包名 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `mcp-client` | `@deepseek-ai/dsh-mcp-client` | 注册到 `ctx.tools` | MCP 客户端桥：连接一个 MCP server，发现并注册其 tool |

组内只有一个包。

## 三件套结构

**本组不构成 capability seam，没有 Service Definition。**

- 它没有自己的 ctx key，不定义任何 service；它是**纯 Consumer/Adapter**：向内消费 `ctx.tools`（必需）、`ctx.attachments`（可选）、`ctx.llm`（可选），向外说 MCP 协议。
- `src/index.ts` 是 namespace plugin 形态：`export const name = 'mcp-client'`、`export const inject = ['tools']`、`export const Config`、`apply`，**无 default export**（符合 `packages/AGENTS.md` 的 plugin export 规则）。
- 因此这里的「Provider」概念落在**外部进程**（stdio 子进程或 Streamable HTTP server），不在 DSH 的 service 注册表里。

三个被消费的 service（跨表综合）：`ctx.tools` 注册/注销工具；`ctx.attachments` 可选地校验并持久化图片结果批次；`ctx.llm` 可选地证明**精确调用路由**显式支持图片输入。

## 扩展点

- **一个 MCP server 一个 plugin 实例**，在 `cordis.yml` 里配。`transport` 为 `stdio`（`command` / `args` / `env` / `cwd`）或 `streamable-http`（`url` / `headers`）。
- 想扩「桥接哪些 MCP 能力」→ 目前只能改这个包：Resources 与 Prompts 无 harness consumer，尚未桥接。
- 想控制启动严格度 → `failOnStartupError`（默认 `false`：初始连接/发现失败时**照常激活但零工具**）。
- 想控制重连 → `reconnect.enabled`（默认 `true`）、`initialDelayMs`（默认 500，每次连续失败翻倍）、`maxDelayMs`（默认 30000，**同时也是「连接存活多久算重置预算」的阈值**）、`maxAttempts`（默认 10）。
- 单次调用超时 → `toolCallTimeoutMs`（默认 60000）。
- **HMR 热插拔**：编辑条目触发 disconnect + reconnect，不需重启进程；`serverName` 不变则重现完全相同的工具名。

## Known Limitations

摘自 `packages/mcp/mcp-client/README.md#Known Limitations and Deferred Work`：

- **Tools are the only bridged MCP capability** —— Resources 与 Prompts 无 harness consumer，推迟。
- **Startup timeout is inherited from the MCP SDK** —— DSH **尚未暴露**连接/发现超时；每次 initialize 或分页 `tools/list` 用 SDK 的 **60 秒默认值**，无响应的 server 或游标链会同时拖慢激活与 teardown。
- **Reconnect triggers on transport close** —— 崩掉的 stdio 子进程会触发；**Streamable HTTP 的失败是按请求暴露的**（外加 SDK transport 自身的 SSE 流恢复），所以不可达的 HTTP server 是「每次调用重试」而不是被 supervisor 重新拉起。
- **Image is the only durable rich-result bridge** —— PNG / JPEG / WebP / GIF 在**精确能力证明**后可进入 Native context；audio 与 embedded-resource 停留在 execution-local 并给出显式诊断；resource link 只保留 name 与 URI 的文本。
- **Unsupported MCP output schemas are not enforced** —— 广告的 schema 用了 harness 子集之外的词汇时，`structuredContent` 回落为无约束的 `JsonValue`。

## 陷阱

1. **`packages/README.md` 的组表格漏了 `mcp/`（和 `runtime-diagnostics/`）**，根 `AGENTS.md` 的 repository layout 也没有 `mcp/` 这一行。想从顶层目录树发现这个组会漏掉它。
2. **两个名字，别搞混**：raw MCP name（发到 wire 上的 `tools/call` 用的）和 public name `mcp__<serverName>__<rawName>`（注册到 `ctx.tools` 的）。**public name 永远不会被发给 server。**
3. **名字被规范化时会追加哈希**：public name 要满足 DeepSeek function-name 契约（64 字符、`[A-Za-z0-9_-]`）；一旦替换或截断改变了名字，就会追加 `(serverName, rawName)` 的**确定性 12 位 hex 哈希**，保证不同工具不会塌缩成同一个名字。名字是 `(serverName, rawName)` 的纯函数——连接顺序、re-sync、其他 server 都不会改名。
4. **四种命名冲突各有不同结局**：两个 server 发布同名 raw tool → 各自命名空间共存；**重复的 `serverName`** → 靠后的 plugin 实例在 load 时失败；一个 server 列出两次同名 tool → 整个 tool list 被判非法；**外部注册占了本 server 的命名空间** → 整代（generation）回滚（绝不留部分集合）并大声报错。
5. **失败模式按阶段不同**：re-sync 的**拉取阶段**失败 → 保留上一代已注册工具；**注册冲突** → 回滚本次尝试的一代，该 server 一个工具都不剩。
6. **重连预算是「每次故障」计的，且能被短暂成功重置**：连续失败达 `maxAttempts` 后注销该 server 的工具并停止重连，**直到 HMR reload 或 Host 重启**。一个存活超过 `maxDelayMs` 的连接会重置预算——所以偶发崩溃的 server 能无限恢复，而崩溃循环的（哪怕中间短暂连上）仍会耗尽上限。
7. **`reconnect.enabled: false` 不是「注销工具」**：连接丢失后工具**仍注册着但调用会失败**，直到 reload。这是刻意的手动恢复行为。
8. **图片进入模型上下文需要三个条件同时满足**：`ctx.attachments` 已挂载 + **精确调用 model 路由显式声明支持图片输入** + 图片批次整体解码并准入成功。**整批一起**准入，任一失败则整批变成诊断文本。`isError` 的调用在图片持久化**之前**就被拒。
9. **inline base64 从不写进 session event**：它只留在 execution-local 的 canonical value 里；provider 从 attachment store 读校验过的字节。
10. **KV cache 的实际表现**：工具集和 schema 不变时 prefix 稳定；**recover 后拿到完全相同的 list 也是 prefix 稳定的**；只有真正增删改重命名才可能从第一个变化的 schema token 起失效。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 组定位（只有一句话 + 一行表） | `packages/mcp/README.md` |
| 全部配置键、命名规则、行为清单、Model Experience | `packages/mcp/mcp-client/README.md` |
| plugin 契约、`Config` 的 union 定义（stdio / http 两支） | `packages/mcp/mcp-client/src/index.ts` |
| 连接生命周期与重连 supervisor | `packages/mcp/mcp-client/src/connection.ts` |
| 工具发现、命名规范化与哈希、注册/回滚 | `packages/mcp/mcp-client/src/tools.ts` |
| 两种 transport 的构造 | `packages/mcp/mcp-client/src/transport.ts` |
| 工具注册 seam 本身 | `packages/core/`（`ctx.tools`）、`docs/subsystems/tools.md` |
| 图片附件的准入与持久化规则 | `packages/attachment/README.md`、`docs/subsystems/attachment.md` |
