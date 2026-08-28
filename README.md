# DSH-Expert

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**A knowledge base that turns Claude Code or Codex into a DeepSeek Harness (DSH) expert — with every answer traceable to an upstream commit.**

[English](#english) · [中文](#中文)

---

## English

### What is this

DSH ([deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)) is an open-source agent harness released on **2026-08-13**. It is in developer preview: roughly one release candidate every two days, with officially warned breaking changes.

That creates two hard problems for any LLM-based assistant:

1. **Every model's training data predates DSH.** Answering from memory is 100% hallucination — there is no "I vaguely remember this" for a tool released after your cutoff.
2. **Hand-written notes rot within days.** And a stale note is more dangerous than no note, because it carries false authority.

DSH-Expert is not an application. It is a **knowledge infrastructure** that solves both: a curated wiki of DSH knowledge with built-in rot detection, plus a set of agent skills that enforce a disciplined answering workflow on top of it. Clone this repo, open Claude Code or Codex inside it, and the agent becomes a DSH expert that must ground every claim in upstream source code.

### How it works — three anti-corruption mechanisms

The wiki is based on the [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) pattern (compact index instead of embeddings, write-back after every Q&A), hardened for a fast-moving upstream:

| Mechanism | Rule |
| --- | --- |
| **Epistemic status** | Every entry is labeled `fact` / `verified_inference` / `hypothesis` / `speculation`. A per-domain mastery level (L0–L4) **caps** the allowed status: only domains verified by hands-on testing (L3+) may carry `fact`. |
| **Source anchors + staleness detection** | Every entry binds to concrete upstream files/symbols plus the commit SHA it was verified against. `/dsh-sync` diffs the anchors against upstream changes; anything hit is marked `stale` and **cannot be cited until re-verified**. |
| **Scope gates on deposition** | Only DSH knowledge (or this repo's own meta-knowledge) may enter the wiki, and it must be de-projectized: integration knowledge is keyed to an eight-dimension coordinate system (host language, runtime shape, concurrency, sandbox, tool reuse, model/credentials, deployment, version baseline) instead of to any specific project. |

On top of the wiki sit three agent skills (Claude Code loads them automatically from `.claude/skills/`; Codex and other AGENTS.md-convention hosts are routed to the same workflows through `AGENTS.md`):

| Skill | Trigger | What it does |
| --- | --- | --- |
| `dsh` | any DSH question (auto) | Freshness check → wiki lookup → verify against upstream source → answer with citations → write back new knowledge |
| `dsh-sync` | `/dsh-sync` | Pull upstream, diff anchors, batch-mark stale entries, re-learn what changed |
| `dsh-wiki` | `/dsh-wiki`, `/dsh-learn` | Wiki health checks: lint, dedup, stale re-verification, coverage report; manual domain learning |

When a delegated question carries a source-thread identifier, the agent must return the complete answer through the host's cross-session reply capability and confirm delivery; completing only the local conversation is not sufficient.

### Relationship to upstream

This is an **unofficial community project** and is not affiliated with DeepSeek. All knowledge is anchored against local blobless clones of:

- [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) — the harness itself
- [cordiverse/cordis](https://github.com/cordiverse/cordis) — the plugin/service/event kernel DSH is built on

When upstream docs and source code disagree, **source code wins** and the conflict is recorded in `wiki/conflicts.md`.

### Quick start

Requirements: `git`, [ripgrep](https://github.com/BurntSushi/ripgrep#installation) (`brew install ripgrep`), and [Claude Code](https://claude.com/claude-code) or [Codex CLI](https://github.com/openai/codex) installed.

```bash
# 1. Clone this repository
git clone https://github.com/yingjialong/DSH-Expert.git dsh-expert
cd dsh-expert

# 2. Clone the upstream mirrors (gitignored; they are the T1 source of truth)
git clone --filter=blob:none https://github.com/deepseek-ai/deepseek-harness.git upstream/deepseek-harness
git clone --filter=blob:none https://github.com/cordiverse/cordis.git upstream/cordis

# 3. Start your AI CLI in the repo root
claude   # Claude Code: CLAUDE.md + the three skills load automatically
codex    # Codex: AGENTS.md routes it to the same constraints and workflows
```

Then just ask, in any language:

```
How do I integrate DSH into a long-running Python service with multiple users?
```

The `dsh` skill fires automatically and runs its six-step workflow: freshness self-check → wiki lookup (index → errors → coverage → hit pages) → source verification in `upstream/` → decide whether a live test is needed → answer (conclusion first, epistemic status per claim, version baseline declared) → write new knowledge back.

`sandbox/` (gitignored) is created on demand for minimal reproductions; it never enters the repository.

### Repository structure

```
dsh-expert/
├── CLAUDE.md                # Identity + 13 hard constraints for the agent (the constitution)
├── AGENTS.md                # Entry shell for AGENTS.md-convention hosts (Codex etc.)
├── .claude/skills/          # dsh / dsh-sync / dsh-wiki
├── wiki/                    # The knowledge base (facts layer)
│   ├── index.md             # Compact index — read this first, never load the whole wiki
│   ├── errors.md            # Mistake log: wrong answers, dead ends, traps
│   ├── coverage.md          # Mastery map L0–L4 per domain
│   ├── conflicts.md         # Upstream doc-vs-source discrepancies
│   ├── open-questions.md    # Unsolved questions
│   ├── log.md               # Q&A / learning / experiment timeline
│   ├── packages/            # One page per upstream package group
│   ├── topics/              # Architecture, Cordis, config, version migrations…
│   └── integration/         # Integration knowledge keyed by the 8-dimension coordinates
├── playbooks/               # Method layer: triage, integration choice, upgrade assessment…
└── docs/Module/             # Docs for this repo's own modules
```

### Key facts (as of 2026-08-22)

| Item | Value |
| --- | --- |
| Default answer baseline | **0.1.1-rc.2** (npm `latest`; it jumped from 0.1.0-rc.7 — rc.8 only ever shipped on `next`) |
| Python SDK | `deepseek-harness-sdk` on PyPI (0.1.1rc1, one rc behind npm) |
| ⚠️ Trap | The PyPI package `deepseek-harness` (0.2.0) is an **unrelated third-party package**; the official one is `deepseek-harness-sdk` |
| Breaking changes rc.8 → 0.1.1-rc.2 | See `wiki/topics/版本变更-0.1.0-rc.8-到-0.1.1-rc.2.md` |

### Contributing & License

Contributions of DSH knowledge (with anchors and honest epistemic status) are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md). MIT — see [LICENSE](LICENSE).

---

## 中文

### 这是什么

DSH（[deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)）是 2026-08-13 发布的开源 agent harness，处于 developer preview 阶段：**约两天一个 rc、官方明确警告会有破坏性变更**。这给任何 LLM 助手带来两个硬问题：

1. **所有模型的训练数据都早于 DSH 发布**——凭记忆回答等于 100% 幻觉，不存在"我大概记得"。
2. **手写知识摘要几天内就腐烂**，而腐烂的笔记比没有笔记更危险：它带着虚假的权威性。

本项目不是应用，而是一套**知识基础设施**：一个内置防腐机制的知识库 + 一组强制执行规范工作流的 agent skill。clone 本仓库、在仓库目录内启动 Claude Code 或 Codex，agent 就变成一名 DSH 专家——每条结论都必须落到上游源码上。

### 工作原理 — 三道防腐机制

知识库基于 [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 模式（紧凑索引而非 embedding、每次问答回写沉淀），并针对高速变动的上游做了强化：

| 机制 | 规则 |
| --- | --- |
| **认知状态** | 每条知识标注 `fact` / `verified_inference` / `hypothesis` / `speculation`；按领域的掌握等级（L0–L4）**封顶**可用状态——只有实测验证过（L3+）的领域才允许标 `fact` |
| **源码锚点 + 失效检测** | 每条知识绑定具体文件/符号与核验时的 commit SHA；`/dsh-sync` 拉取上游后 diff 锚点，命中的条目标 `stale`，**复验前禁止引用** |
| **沉淀双门槛** | 只收 DSH 知识（或本库元知识）；集成知识挂到八维坐标系（宿主语言 · 运行形态 · 并发与会话隔离 · 沙箱与文件系统 · 工具复用 · 模型与凭据 · 部署环境 · 版本基线），不挂在任何具体项目上，保证跨项目可复用 |

三个 agent skill（Claude Code 从 `.claude/skills/` 自动加载；Codex 等遵循 AGENTS.md 约定的宿主经 `AGENTS.md` 引导到同一套工作流）：

| skill | 触发 | 职责 |
| --- | --- | --- |
| `dsh` | 任何 DSH 问题（自动） | 新鲜度自检 → 查库 → 回上游源码核验 → （必要时）实测 → 带出作答 → 沉淀写回 |
| `dsh-sync` | `/dsh-sync` | 同步上游、锚点 diff、批量标 stale、补学变更部分 |
| `dsh-wiki` | `/dsh-wiki`、`/dsh-learn` | 知识库健康检查（lint / 查重 / 复验 / 覆盖度报告）；手动指定领域学习 |

当跨会话委派携带来源会话标识时，agent 必须通过宿主的跨会话回复能力把完整答案回传来源会话并确认送达；只完成当前本地会话不算交付完成。

### 与上游的关系

本项目是**非官方社区项目**，与 DeepSeek 无隶属。全部知识锚定以下两个仓库的本地 blobless 克隆：

- [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) — harness 本体
- [cordiverse/cordis](https://github.com/cordiverse/cordis) — DSH 底层的插件/服务/事件内核

上游文档与源码冲突时**以源码为准**，冲突记入 `wiki/conflicts.md`。

### 快速开始

前置：`git`、[ripgrep](https://github.com/BurntSushi/ripgrep#installation)（`brew install ripgrep`）、已安装 [Claude Code](https://claude.com/claude-code) 或 [Codex CLI](https://github.com/openai/codex)。

```bash
# 1. clone 本仓库
git clone https://github.com/yingjialong/DSH-Expert.git dsh-expert
cd dsh-expert

# 2. clone 上游镜像（已被 gitignore；它们是 T1 唯一事实源）
git clone --filter=blob:none https://github.com/deepseek-ai/deepseek-harness.git upstream/deepseek-harness
git clone --filter=blob:none https://github.com/cordiverse/cordis.git upstream/cordis

# 3. 在仓库根目录启动你的 AI CLI
claude   # Claude Code：自动加载 CLAUDE.md 与三个 skill
codex    # Codex：经 AGENTS.md 引导到同一套约束与工作流
```

然后直接用任何语言提问：

```
我想把 DSH 集成进一个多用户长驻 Python 服务，怎么选型？
```

`dsh` skill 自动触发并执行六步流程：新鲜度自检 → 查库（index → errors → coverage → 命中页）→ `upstream/` 源码核验 → 判断是否实测 → 作答（结论先行、每条主张带认知状态、声明版本基线）→ 沉淀写回。

`sandbox/`（gitignored）用于最小复现实测，按需创建，永不入库。

### 仓库结构

```
dsh-expert/
├── CLAUDE.md                # agent 的身份与 13 条硬约束（本项目宪法）
├── AGENTS.md                # AGENTS.md 约定宿主（Codex 等）的入口壳
├── .claude/skills/          # dsh / dsh-sync / dsh-wiki
├── wiki/                    # 知识库（事实层）
│   ├── index.md             # 紧凑索引 —— 先读它，永远不全量加载知识库
│   ├── errors.md            # 错误本：答错的、死路、陷阱
│   ├── coverage.md          # 覆盖度地图（每领域掌握等级 L0–L4）
│   ├── conflicts.md         # 上游文档与源码不符登记册
│   ├── open-questions.md    # 悬而未决
│   ├── log.md               # 问答/学习/实测时间线
│   ├── packages/            # 一组一页，映射上游 packages/<group>
│   ├── topics/              # 架构、Cordis、配置、版本变更…
│   └── integration/         # 按八维坐标系组织的集成知识
├── playbooks/               # 方法层剧本：诊断、选型、升级评估…
└── docs/Module/             # 本仓库自身模块的文档
```

### 关键事实（截至 2026-08-22）

| 项 | 值 |
| --- | --- |
| 默认回答基线 | **0.1.1-rc.2**（npm `latest`；从 0.1.0-rc.7 直接跳过来 —— rc.8 只上过 `next` tag） |
| Python SDK | PyPI `deepseek-harness-sdk`（0.1.1rc1，落后 npm 一个 rc） |
| ⚠️ 陷阱 | PyPI 上的 `deepseek-harness`（0.2.0）是**无关第三方包**，官方包是 `deepseek-harness-sdk` |
| rc.8 → 0.1.1-rc.2 破坏性变更 | 见 `wiki/topics/版本变更-0.1.0-rc.8-到-0.1.1-rc.2.md` |

### 贡献与许可

欢迎贡献带锚点与诚实认知状态的 DSH 知识 —— 见 [CONTRIBUTING.md](CONTRIBUTING.md)。MIT 许可证 —— 见 [LICENSE](LICENSE)。
