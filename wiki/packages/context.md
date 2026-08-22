---
title: packages/context — 请求上下文扩展
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/context/README.md
  - packages/context/agent-instructions/README.md
  - packages/context/file-reference/README.md
  - packages/context/file-reference-local/README.md
  - packages/context/session-reference/README.md
  - packages/context/time-context/README.md
  - packages/context/tmux-context/README.md
  - packages/context/file-reference/src/grammar.ts
  - packages/context/session-reference/src/uri.ts
  - docs/subsystems/session-reference.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

一组**不定义任何工具**、只往模型可见的请求上下文里加东西的 product 插件：工作区指令、时间、tmux 位置、跨会话引用、`@file` 引用。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包目录 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `agent-instructions/` | `@deepseek-ai/dsh-agent-instructions` | — | 加载 `AGENTS.md` / `CLAUDE.md` 系工作区指令链，并在文件变化后补发增改删通知 |
| `file-reference/` | `@deepseek-ai/dsh-file-reference` | `ctx.fileReferences` | 文件引用发现 seam + 浏览器安全的 `@file` 语法 |
| `file-reference-local/` | `@deepseek-ai/dsh-file-reference-local` | 注册 `ctx.fileReferences` | 本地文件系统的 provider（每 agent 一份有界索引） |
| `session-reference/` | `@deepseek-ai/dsh-session-reference` | `ctx.sessionReferenceResolver` | 把别的 session 做成有界只读快照注入当前会话 |
| `time-context/` | `@deepseek-ai/dsh-time-context` | — | 当前时区时间 + 已过时长上下文 |
| `tmux-context/` | `@deepseek-ai/dsh-tmux-context` | — | tmux session/window/pane 位置与布局上下文 |

## 三件套结构

本组**不是**一个统一的 capability family，而是若干独立插件 + 一对完整 seam：

- **Service Definition**：`@deepseek-ai/dsh-file-reference`（`ctx.fileReferences`，只返回路径候选，不读内容）。同时是 `@Remote` 的，浏览器侧直接 `ctx.remote.fileReferences.list`，无需 API Proxy 路由。
- **Service Provider**：`@deepseek-ai/dsh-file-reference-local`。
- **Consumer**：宿主 UI（补全框），不是模型工具。
- `session-reference` 是**单包服务**（既定义又实现 `ctx.sessionReferenceResolver`），Consumer 是它自己注册的外层 `agent/pre-step` 监听器；它消费 `ctx.sessionQuery`。
- `agent-instructions` / `time-context` / `tmux-context` 是纯 function plugin，只通过 `ctx.systemPrompt.context()` 或 agent inbox 注入，不注册服务。

## 扩展点

- **要换文件引用来源**（远程 workspace、虚拟 FS）：依赖 `@deepseek-ai/dsh-file-reference`，实现并注册 `ctx.fileReferences`。可复用它导出的 `activeAtToken()` / `formatFileMention()` 与 `FILE_REFERENCE_PROMPT`。
- **要换跨会话快照策略**：`session-reference` 只依赖 `ctx.sessionQuery`（**不要求 SQLite FTS**），可作为宿主 opt-in 服务替换。
- **要加自己的上下文**：走 `ctx.systemPrompt.context()`（scope 即调用上下文，`agent.ctx` 则只对该 agent 生效），不要改 loop。
- **`agent-instructions` 的 fs 依赖是可选的**：它不静态 inject `fs`，无 provider 的产品树照样能启动，指令加载退化为 no-op。

## Known Limitations

- `file-reference`：候选路径只是建议，seam **不保证**后续的模型文件工具能访问同一命名空间；选中文件不读内容，仍需模型显式调 `read`。
- `file-reference-local`：只扫宿主文件系统；超过 `maxEntries`（默认 10000）会漏路径；**不解析 `.gitignore`**，只按配置的目录 basename 排除（默认 `.git`、`node_modules`）。
- `session-reference`：不搜消息正文（只看折叠后的标题）；假定宿主有权读 `ctx.sessionQuery` 暴露的每个 session，**不是模型可用的搜索工具**；只投影文本；快照无实时联动。
- `agent-instructions`：`bash cd` 不触发嵌套发现；无 watcher（靠成功的 `read`/`write`/`edit` 触发）；不解释 `.claude/rules/` 与 `@path` import；per-directory 去重基于内容（trim 后逐字节相同才合并）；**符号链接会被跨信任边界跟随**；内容只做截断不做摘要。
- `time-context`：只提供 prompt 溯源，不替别的工具填时区字段；一个 turn 内混了多个浏览器时区就让模型追问而不是猜；只到秒。
- `tmux-context`：每 turn 首步采样一次；只报自己的位置，不抓兄弟 pane 文本；只报布局不报像素尺寸。

## 陷阱

- **只有 `agent-instructions` 在默认 bundle 里**。`dsh-agent-spine-demo` 默认挂载它（bundle config 的 `workspaceContext` 必须给 `{ maxBytes }` 或显式 `false`）；`time-context`、`tmux-context`、`session-reference`、`file-reference`、`file-reference-local` 全是 opt-in。
- **符号链接指令文件是真实的信任边界洞**：clone 下来的仓库可以让树外文件内容以「较低权威的 workspace 指导」身份进入 prompt（它不会覆盖 system/developer/直接用户指令）。加载不可信仓库时要用文件系统策略门或 OS sandbox 约束 `ctx.fs`。
- **`maxReferences` 硬上限是 3**：`session-reference` 的配置项默认 3 且「must be at most 3」，不是能随便调大的软限制。超预算直接 `SESSION_REFERENCE_BUDGET_EXCEEDED` 而不是返回部分上下文。
- **session-reference 的快照排除自身**：被引用会话里由 session-reference 注入的消息不会再传递，防止递归；被压缩过的源只贡献最新 checkpoint + 之后的对话，不还原被遮蔽文本。
- **tmux 判定基于 tty**：只有进程的控制终端与 `$TMUX_PANE` 的 `#{pane_tty}` 匹配才算「在 tmux 里」，故意排除从 tmux 祖先继承了 `$TMUX` 的 VS Code 集成终端。
- **`@` 补全的触发位置有语法约束**：`activeAtToken()` 只在输入起始或空白之后识别 `@path`，所以 email 样式文本不会打开补全。
- `time-context` 的 `refreshIntervalMs` 省略或设 0 表示**每次合格尝试都注入**（历史成本最高），正数只是降低而非消除该成本。

## 去哪深入（文件路由）

| 想知道什么 | 去哪 |
|---|---|
| 组内包/ctx-key 全表与 opt-in 状态 | `packages/context/README.md` |
| 指令加载的生命周期、`<system-reminder>` 原文模板、digest/去重/resume 协调 | `packages/context/agent-instructions/README.md` |
| `@file` 语法、Remote 方法、`FILE_REFERENCE_PROMPT` | `packages/context/file-reference/README.md`（语法实现在 `src/grammar.ts`） |
| 本地索引配置（`maxResults`/`maxEntries`/`excludedDirectories`） | `packages/context/file-reference-local/README.md` |
| `dsh-session:<base64url>` URI 编解码、快照投影规则、预算三项 | `packages/context/session-reference/README.md`（URI 在 `src/uri.ts`）、`docs/subsystems/session-reference.md` |
| 时间/时区决策与 fallback 顺序 | `packages/context/time-context/README.md` |
| tmux 采样与字段分隔 | `packages/context/tmux-context/README.md` |
| 工作区上下文的 per-agent/session 隔离与生命周期拆分决策 | `.agents/notes/implemented/feature/2026-06-24-workspace-context.md` |
| 可跑的 resume 场景 | `examples/headless-agent/workspace-context-resume.cordis.snapshot.yml` |
