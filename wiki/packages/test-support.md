---
title: packages/test-support — 开发与测试基础设施
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/test-support/README.md
  - packages/test-support/acp-snapshot/README.md
  - packages/test-support/agent-loop-testkit/README.md
  - packages/test-support/client-runtime/README.md
  - packages/test-support/llm-mock-server/README.md
  - packages/test-support/llm-replay/README.md
  - packages/test-support/loader-smoke/README.md
  - packages/runtime-diagnostics/invariants/README.md
  - docs/testing.md
  - docs/subsystems/invariants.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

服务于仓库开发、测试与示例的支撑包，不是产品 API；兼容性只跟随它所服务的开发需求。一个包一旦获得产品契约和产品消费者，就要**搬出** `test-support/`。

## 稳定性

`Support — lower compatibility expectations`（`packages/README.md` 表格中 `test-support/` 行原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `test-support/acp-snapshot/` | `@deepseek-ai/dsh-acp-snapshot` | ACP 快照测试套件工具箱（launcher / harness / normalizers / factory 四层） |
| `test-support/agent-loop-testkit/` | `@deepseek-ai/dsh-agent-loop-testkit` | 为 `AgentLoop` 测试按依赖顺序挂载必备前置服务 |
| `test-support/client-runtime/` | `@deepseek-ai/dsh-client-test-runtime` | jsdom slot 测试运行时（注意 npm 名带 `-test-`，与目录名不同） |
| `test-support/llm-mock-server/` | `@deepseek-ai/dsh-llm-mock-server` | 可编排的 OpenAI 兼容 HTTP/SSE 故障服务器，无需 key |
| `test-support/llm-replay/` | `@deepseek-ai/dsh-llm-replay` | 用录制的 session JSONL 回放模型流，供 keyless 快照与 demo |
| `test-support/loader-smoke/` | `@deepseek-ai/dsh-loader-smoke` | 以子进程启动 Loader 组装的应用做冒烟测试 |
| （异地）`runtime-diagnostics/invariants/` | `dsh-invariants` | 开发期运行时契约断言注册表（`ctx.invariants`），group README 把它列在这里但物理上在另一组 |

## 三件套结构

这一组**不是** capability seam，没有 Service Definition / Provider / Consumer 三件套。它是横切的测试基建：
- 唯一带 Cordis 服务形态的是 `dsh-invariants`（`ctx.invariants`），它是一个**注册表服务**——每个 workspace 包发布 `./invariant` 伴生入口注册自己的 npm 全名，root 插件本身不含任何产品检查、不 import 产品包。
- `llm-replay` 是**插件**（替换真实 LLM adapter）；其余是库/工具箱，不进产品插件图。
- `client-runtime` 明确不参与产品插件图（无 `dsh.client`），只出现在 feature 包的 `devDependencies`。

## 扩展点

- **给一个包加运行时不变量**：依赖 `dsh-invariants`，在包内新建 `./invariant` 出口注册精确 npm 名；检查的必须是事件/数据关系，不是服务或方法是否存在。没有可查关系时必须写 `No runtime invariant:` 加包特定理由，否则 `verify-package-invariants` 会挂。选包规则：service `enabled` 且 allowlist 为空或有一条匹配全名，且 blocklist 无匹配（**blocklist 覆盖 allowlist**）；每条 pattern 是 `new RegExp(source)` 大小写敏感、默认不锚定，`/pattern/flags` 语法**不被解析**。
- **给一个 example 加快照套件**：用 `defineAcpSnapshotSuite(scenarioTable)`，它必须在 vitest **collection time** 调用。
- **测 `AgentLoop`**：`mountAgentLoopTestDependencies(ctx, options?)` 只挂 LLM / session / system-prompt / tool / agent 五类服务并在挂 loop **之前**返回；adapter、可选插件、`AgentLoop` 本身、agent、Context teardown 全部由调用方持有。
- **无 key 跑真实 adapter**：`startMockLlmServer(options)`；**无 key 跑真实 agent**：`llm-replay` + 场景目录里的 `session.jsonl`。

## Known Limitations

- `acp-snapshot`：`runScenario` 采集持久化的 `.jsonl`，所以快照配置必须 `persistenceCompression: 'none'`；**压缩 JSONL 与 SQLite 组合没有快照采集路径**。session fixture 保留 headers 与 payloads，但**省略 body sequence/time envelopes**（replay 时合成，runtime persistence 不变），格式仍是 canonical packed rows。built 模式（`DSH_EXAMPLE_MODE=lib`）要求先 `pnpm run build`。后端覆盖仍绑在 ACP driver 上。
- `agent-loop-testkit`：只共享强制前置 spine，其余刻意留给调用方以保持场景顺序可见。
- `client-runtime`：**只能通过仓库源码别名消费**——`lib/index.js` 在纯 Node 下不可 import（其 built 产物再导出的是浏览器 loader 脚本）。会话快照是 fixture 数据而非回放历史，`updateSnapshot` 直写快照存储，因此 fixture 可以表达生产投影永远不会产生的状态。
- `llm-mock-server`：随机权重反映测试压力而非生产分布；请求脚本按**到达顺序**共享一个游标（并发调用者要确定性故障必须各起一个实例）；真正的连接拒绝是 listener 生命周期阶段，请求级随机只能重置已接受的连接。
- `llm-replay`：first-call-order 脚本绑定**假设顺序委派**，并发同级 subagent 会非确定性绑定（源码里标 `XXX(concurrent-subagents)`）。只有普通 loop chunk 和被标记的本地 compaction 输出可以推导，其余要 `replay.override.json` 旁挂文件。
- `loader-smoke`：built 模式要求先构建且配置能向上经 `examples/node_modules` 解析出每个具名包；stdout/stderr 仅受 execa 默认 100 MB `maxBuffer` 约束；**超时只杀直接子进程**，faulty fixture 起的进程树可能活过冒烟测试。

## 陷阱

- 目录名 ≠ npm 名：`test-support/client-runtime/` 的包名是 `@deepseek-ai/dsh-client-test-runtime`。
- group README 表格里的 `invariants/` 指向 `../runtime-diagnostics/invariants/README.md`——它不在本组目录下，且 `runtime-diagnostics/` 在 `packages/README.md` 的分组表里**根本没有行**。
- `client-runtime` 存在于目录但**不在** group README 表格里（见 conflicts）。
- CI 的覆盖率闸门是 `pnpm run test:coverage`（per-file 100%），不是 `pnpm run test`；这是 `docs/testing.md` 明说的。

## 去哪深入（文件路由）

| 想查 | 去 |
|---|---|
| 测试分层策略、key 政策、何时必须加快照 | `docs/testing.md` |
| invariants 契约、选择规则 | `docs/subsystems/invariants.md`、`packages/runtime-diagnostics/invariants/README.md` |
| 快照 normalizer 到底改写了什么 | `packages/test-support/acp-snapshot/README.md`（Normalizers 一节最密） |
| 回放 fixture 怎么从 session log 重建流 | `packages/test-support/llm-replay/README.md` |
| 包级 invariant 的硬性规则 | `packages/AGENTS.md`（末条 `Every package owns ./invariant`） |
