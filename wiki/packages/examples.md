---
title: packages/examples — 现成可跑的 demo bundle
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/examples/README.md
  - packages/examples/agent-spine-demo/README.md
  - packages/examples/acp-demo/README.md
  - packages/examples/jsonrpc-demo/README.md
  - packages/examples/agent-spine-demo/src/index.ts
  - packages/examples/acp-demo/src/bin.ts
  - packages/examples/jsonrpc-demo/src/bin.ts
  - packages/examples/jsonrpc-demo/src/packaged-bin.ts
  - examples/AGENTS.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

预组合好的插件 **bundle**：一个很薄的叶子 `cordis.yml` 直接 load 它，就不必手工拼主干和入口点。`-demo` 的 npm 后缀本身就是「非产品面」的标记。

## 稳定性

`Support — example infra`（`packages/README.md` 表格原文）。README 强调这些**不是 product API**：产品 seam 与入口点留在各自的归属组，demo bundle 只负责挑选具体组合。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `agent-spine-demo/` | `@deepseek-ai/dsh-agent-spine-demo` | 可复用的 agent 主干 bundle：无执行器、无 UI，一个 Cordis bundle 插件装齐每个 agent 都要的固定服务集 |
| `acp-demo/` | `@deepseek-ai/dsh-acp-demo` | ACP 自动化应用 bundle（bin：`dsh-acp-demo`） |
| `jsonrpc-demo/` | `@deepseek-ai/dsh-sdk-jsonrpc-demo` | 只有 bin 的应用，boot 一棵外部 `cordis.yml` 的插件树（bin：`dsh-jsonrpc-agent`） |

## 三件套结构

本组把 Service Definition / Provider / Consumer 的分离**上升到组合层**（`packages/examples/README.md` 原话）：

- **bundle 拥有共享主干**：`agent-spine-demo` 的 `apply(ctx, config)` 把一整棵子插件树挂在 bundle fiber 下（timer / llm 抽象服务 / session / session-title / system-prompt / tools / skill 注册表 + 本地 provider / agent / goal 三件 / llm-retry / jobs-local / invariants + 四个 `/invariant` 伴生 / tool-bash / agent-instructions / tool-skill / tool-jobs / agent-loop）。
- **叶子拥有后端**：LLM adapter（`llm-deepseek` / `llm-pi-ai` / `llm-replay`）、bash executor（`ctx.shell`）、模型型 session-title provider、非本地 skill provider。
- **app 包拥有入口点**：ACP transport、JSON-RPC stdio、以及各自的 stdout / reload 纪律。
- `acp-demo` 在 spine 之上再加 JSONL 持久化、checkpoint policy、session-query-sqlite 和 `dsh-acp`；`jsonrpc-demo` 干脆什么都不预设，全靠外部配置。

## 扩展点

- **要做自己的应用**：load `@deepseek-ai/dsh-agent-spine-demo` 当 bundle，然后只补入口点和可换后端。bundle 会把 `agents` 等字段转发给拥有它的子插件（`pickSpineConfig()` 只复制本 bundle 拥有的字段，冲突的 `dshHome` 在组合期直接失败）。
- **要换 loop 或去掉某个主干成员**：**做不到**——只能组合另一个 bundle（见 Known Limitations）。
- **要关某些可选件**：config 里 `toolBash: false`、`toolJobs: false`、`goals: false`、`workspaceContext: false`、`includeRuntimeContext: false`、`invariants.enabled: false` + 包 allowlist/blocklist 正则。
- **bundle 而不是共享 YAML include 的理由**：YAML include 能去重 config 但**不能拥有 bin、也不能提供入口点默认值**；bundle 的子插件注册在 root isolate-keyed store 里，注入的叶子兄弟无需加载顺序耦合就能看见它们。

## Known Limitations

- `agent-spine-demo`：主干集合**大部分写死在代码里**——`apply()` 总是挂载核心服务，config 只能省略 goals/skills/bash/task-control 工具；换 loop 或去掉别的主干成员意味着换一个 bundle。invariant 服务与伴生是固定成员，`invariants.enabled: false` 只是抑制检查而不移除注册（Session 自身的常开校验与冻结另算）。
- `acp-demo`：JSONL 持久化写死；兄弟插件仍可能污染 stdout（app 无法阻止别人往 stdout 写非协议字节）；只支持全新的自动化 session，resume 与人类交互属于别的入口点。
- `jsonrpc-demo`：bin **无法证明配置真的提供了 JSON-RPC 服务**（缺 `dsh-sdk-jsonrpc-server` 的合法配置照样 boot 成功、什么都不服务）；没有内置或默认配置；stdin EOF 会切断进行中的工作（需要有序完成就用协议层的 `shutdown` 请求）。

## 陷阱

- **不要把 `packages/examples/` 和仓库根的 `examples/` 搞混**：根目录 `examples/` 放的是可跑的 `cordis.yml` **叶子**（`headless-agent`、`acp-agent`、`jsonrpc-agent`、`web-cordis`、`web-schedule`、`mcp-memory`），本组放的是那些叶子加载的 **bundle**。
- **目录名 ≠ npm 名**：`jsonrpc-demo/` 的包名是 `@deepseek-ai/dsh-sdk-jsonrpc-demo`（多了 `sdk-`），bin 名又是第三个：`dsh-jsonrpc-agent`。按目录名去 `pnpm add` 会找不到。
- **产品级一次性执行不在这里**：那是 `dsh --profile headless`；本目录任何包都不提供它。
- **ACP demo 的 stdout 就是协议线**，诊断一律走 stderr；配置里不能挂 stdout logger。同理 JSON-RPC demo。
- **JSON-RPC 的配置发现只有两个通道且按序取首个非空**：`$DSH_CORDIS_CONFIG`，然后位置参数 `argv[2]`。都不指向存在的文件就打印一行用法到 stderr 并 exit 1——**没有工作目录 fallback，也没有内置默认**。另外它**不使用 `DSH_SNAPSHOT`**（而 `acp-demo` 用 `DSH_SNAPSHOT=replay` 切到兄弟 `cordis.snapshot.yml`）。
- **退出码有区分**：`jsonrpc-demo` 的 stdin EOF 与 `SIGTERM` 是 exit 0，`SIGINT` 是 **exit 130**。
- **Python SDK 走的是另一个 bin 实现**：`dsh-jsonrpc-agent-pkg` 单文件可执行运行时用 `lib/packaged-bin.js`（打包的 bare 插件从其封闭运行时树解析，相对路径插件仍相对配置文件）。
- `llm-retry` 会在新的编号 step 里重放失败请求：重试状态、provider 错误、失败的部分 chunk 都不进模型历史，但**每次 provider 尝试都可能计费**，`always` 模式没有尝试次数上限。

## 去哪深入（文件路由）

| 想知道什么 | 去哪 |
|---|---|
| 三个 bundle 的角色分工与「bundle vs leaf」的界限 | `packages/examples/README.md` |
| spine 加载的完整插件树、故意留在 bundle 外的四类东西、Config 字段清单 | `packages/examples/agent-spine-demo/README.md` |
| ACP app 的组合表、完整 config 路由表、bin 用法 | `packages/examples/acp-demo/README.md` |
| 外部配置发现、退出生命周期、stdout 纪律 | `packages/examples/jsonrpc-demo/README.md` |
| 可跑的叶子配置本身 | `examples/AGENTS.md`、`examples/headless-agent/`、`examples/acp-agent/cordis.yml` |
| Python 侧的打包运行时 | `python/sdk-runtime/README.md` |
| ACP 传输层本体 | `packages/acp/acp/README.md` |
| JSON-RPC server 插件本体 | `packages/sdk/server/README.md` |
