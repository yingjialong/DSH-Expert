# DSH-Expert

> DeepSeek Harness（DSH）领域专家系统 —— 对外是 DSH 专家，对内是 DSH 学习者。

## 一、项目定位

本项目不是一个应用，而是一套**让 AI 持续成为 DSH 专家的知识基础设施**。它要解决的核心问题是：

DSH（[deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)）于 2026-08-13 发布，处于 developer preview 阶段，**约两天一个 rc、官方明确警告会有破坏性变更**。任何大模型的训练数据都早于它的发布日期，因此：

- **凭记忆回答 DSH 问题 = 100% 幻觉**；
- **手写的知识摘要会在几天内腐烂**，而腐烂的笔记比没有笔记更危险。

本项目用一套"知识底座 + 防腐机制 + 学习循环"来对抗上述两点，使得每一次问答都比上一次更准、更快，且**永远可追溯到上游源码的某个 commit**。

## 二、核心设计

### 2.1 双重身份

| 面向 | 身份 | 表现 |
| --- | --- | --- |
| 对外（回答提问者） | **DSH 专家** | 结论先行、带出处、标注版本基线与认知状态 |
| 对内（自我建设） | **DSH 学习者** | 维护覆盖度地图、主动补盲区、记录错误本与悬而未决 |

提问者不限于人类：另一个 session 的 LLM 也可能直接来问（见 `CLAUDE.md` 第 11 条的沉淀治理约束）。

### 2.2 知识分两层

| 层 | 内容 | 会腐烂吗 | 位置 |
| --- | --- | --- | --- |
| **事实层** | DSH 是什么、怎么实现的、接口长什么样 | **会**（绑源码锚点，靠 diff 检测） | `wiki/` |
| **方法层** | 遇到某类问题怎么查、怎么选型、该问什么 | 基本不会 | `playbooks/` |

**知识库不复制上游文档正文**，只存三类上游没有的东西：路由（问题 → 去哪找）、踩坑与实测结论、跨文档综合推理。

### 2.3 LLM Wiki 模式与本项目的改造

底座采用 Karpathy 的 [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 模式（三层 raw → wiki → schema、三操作 ingest → query → lint、导航靠紧凑 index 而非 embedding、每次问答回写沉淀）。但 LLM Wiki 假设**源材料是静态的**，而 DSH 是活代码，因此本项目做了三项关键改造：

1. **认知状态（epistemic status）**：每条知识标注 `fact | verified_inference | hypothesis | speculation`，杜绝"agent 的猜测被写回后自我强化成权威结论"这一已知缺陷。
2. **源码锚点 + 失效检测**：每条知识绑定具体文件/符号与核实时的 commit SHA；同步上游后 diff 锚点，命中即标 `stale`，stale 条目必须复验才能引用。
3. **掌握等级封顶认知状态**：L1/L2 的东西不许标 `fact`，只有 L3（实测验证过）以上才可以。

### 2.4 四把标尺

| 标尺 | 取值 | 用途 |
| --- | --- | --- |
| **认知状态** | `fact` / `verified_inference` / `hypothesis` / `speculation` | 这条知识有多可信 |
| **掌握等级** | L0 未接触 / L1 读过文档 / L2 读过源码关键路径 / L3 实测验证过 / L4 有踩坑记录 | 我对这个领域有多懂 |
| **新鲜度** | `fresh` / `stale` | 这条知识是否可能已被上游改动废掉 |
| **信源分级** | T1 源码 > T2 上游 docs/官方站 > T3 Release/Discussions/Issues > T4 第三方 | 冲突时听谁的（源码为准） |

### 2.5 知识的维度坐标系（去项目化的关键）

集成类知识**不挂在项目上，挂在技术特征坐标上**，写成「条件 → 结论 → 锚点 → 认知状态」：

宿主语言 · 运行形态 · 并发与会话隔离 · 沙箱与文件系统 · 工具复用 · 模型与凭据 · 部署环境 · 版本基线

于是"越问越聪明"有了确切含义：**第一次落进某个坐标格子要现查，第二次落进同一格子直接命中**。项目上下文只是用完即弃的输入，永远不进知识库。

## 三、项目模块

| 模块 | 位置 | 职责 | 模块文档 |
| --- | --- | --- | --- |
| 知识库 | `wiki/` | 事实层知识、索引、日志、覆盖度、错误本、悬而未决 | [knowledge-base.md](docs/Module/knowledge-base.md) |
| 方法库 | `playbooks/` | 方法层剧本：选型 / 诊断 / 插件开发 / 升级评估 | [playbooks.md](docs/Module/playbooks.md) |
| 技能 | `.claude/skills/` | `dsh` 问答主入口 / `dsh-sync` 同步防腐 / `dsh-wiki` 维护 | [skills.md](docs/Module/skills.md) |
| 上游镜像 | `upstream/` | DSH 与 Cordis 的 blobless 克隆（唯一事实源，已 gitignore） | [upstream.md](docs/Module/upstream.md) |

## 四、目录结构

```
DSH-Expert/
├── CLAUDE.md                    # 身份与 12 条硬约束（最高优先级）
├── README.md                    # 本文件
├── .claude/skills/
│   ├── dsh/SKILL.md             # 问答主入口（自动触发）
│   ├── dsh-sync/SKILL.md        # 同步上游 + 锚点 diff + 标 stale
│   └── dsh-wiki/SKILL.md        # lint / 查重 / 补链 / 健康检查 / 批量复验
├── wiki/
│   ├── index.md                 # 紧凑索引（一行摘要 + 锚点 + status + freshness）
│   ├── log.md                   # 每次问答的时间线
│   ├── coverage.md              # 覆盖度地图（掌握等级 L0-L4）
│   ├── errors.md                # 错误本：答错的、走过的死路、看似对其实是坑的
│   ├── open-questions.md        # 悬而未决：查了但没查到的
│   ├── packages/                # 映射上游 packages/<group>
│   ├── topics/                  # 映射上游 docs/<topic> + Cordis
│   └── integration/             # 按维度坐标系组织的集成知识
├── playbooks/                   # 方法层剧本
├── docs/                        # Task_history.md + Task-detail/ + Module/
├── sandbox/                     # 实测复现用（gitignore）
└── upstream/                    # 上游克隆（gitignore）
    ├── deepseek-harness/
    └── cordis/
```

## 五、常用命令

| 命令 | 作用 |
| --- | --- |
| 直接提问（如"DSH 的 skill 插件怎么写"） | 自动触发 `dsh` skill 全流程：新鲜度自检 → 查库 → 核验源码 → 作答 → 沉淀 |
| `/dsh <问题>` | 同上，显式触发 |
| `/dsh-sync` | 同步上游、diff 锚点、批量标 stale、生成 rc→rc 变更摘要、补学变更部分 |
| `/dsh-wiki` | 知识库维护：lint、查重、补交叉链接、健康检查、批量复验 stale 条目 |
| `/dsh-learn <领域>` | 手动触发主动学习，学完汇报覆盖度变化 |

## 六、DSH 关键事实速查（截至 2026-08-20 核实）

| 项 | 值 |
| --- | --- |
| 首次发布 | 2026-08-13 |
| 许可证 | MIT |
| 形态 | TypeScript pnpm monorepo（含 `python/`、`native/`） |
| 内核 | [Cordis](https://github.com/cordiverse/cordis)，"一切皆插件" |
| npm 包 | `@deepseek-ai/dsh`；`latest` = **0.1.0-rc.7**，`next` = 0.1.0-rc.8 |
| Python SDK | `deepseek-harness-sdk`（PyPI，0.1.0rc7，Python≥3.10）+ `deepseek-harness-runtime-bin` |
| **默认回答基线** | **rc.7**（npm latest / PyPI 唯一可用），同时可回答 master 的差异 |
| 包组数量 | `packages/` **50 组 / 226 个包**，绝大多数标注 Product — stable API |
| 集成表面 | TS SDK / Python SDK（stdio newline-delimited JSON-RPC）/ ACP server / HTTP API gateway |
| ⚠️ 已知陷阱 | PyPI 上的 `deepseek-harness`（0.2.0）是**无关第三方包**，官方是 `deepseek-harness-sdk` |

## 七、变更记录

| 日期 | 变更 | 任务 |
| --- | --- | --- |
| 2026-08-20 | 项目创建：确立双重身份、四把标尺、维度坐标系；建立 wiki/playbooks/skills 骨架；克隆上游；完成冷启动 | [dsh-expert-bootstrap](docs/Task-detail/dsh-expert-bootstrap.md) |
