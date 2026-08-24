---
title: DSH 文档路由地图（docs/ 顶层 19 主题）
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - docs/architecture.md
  - docs/graph-atlas.md
  - docs/AGENTS.md
  - docs/subsystems/README.md
  - docs/cookbook/extension-cookbook.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

# DSH 文档路由地图

**只做路由，不复述内容**。所有路径相对 `upstream/deepseek-harness/`。

## 顶层 19 主题

| 主题文件 | 一句话内容 | 什么问题该来这里查 | 篇幅 |
|---|---|---|---|
| `docs/architecture.md` | 全局有序地图：Cordis→profile/bundle→core 包→事件域→turn flow→seam→扩展点表 | 「改 `packages/` 前必读」；「我这个新行为该挂在哪个扩展点」（末尾 "Where new behavior goes" 表是全库最高频查表） | 131 行 |
| `docs/agent-lifecycle.md` | 一次 turn/step 的 Mermaid 时序图（curated） | 事件发出的**先后顺序**；`agent/pre-step` 拒绝/改写后发生什么；compaction 在哪个位置介入 | 82 行 |
| `docs/tool-execution-pipeline.md` | 工具调用从 `tool/call` 到 `tool/result` 的 Mermaid 流程图（curated） | pre/guard/approval/around/post 各自的位置；denial 走哪条边；`finalizeContent` 何时跑；Code Mode 子调用怎么重入 | 62 行 |
| `docs/event-producer-consumer.md` | 全部 harness Cordis 事件的「谁发 / 谁听」矩阵 + `@mode`（hybrid generated） | 「谁会收到这个事件」「我加监听会不会撞上别人」「这个事件是 emit 还是 waterfall」 | 76 行 |
| `docs/capability-seams.md` | 全部 `ctx.*` 服务的角色分类表：owner / implementations / consumers / 备注（hybrid generated） | 「这个能力的 ctx key 叫什么」「有哪些现成 provider」「换 provider 会影响谁」 | 483 行 |
| `docs/glossary.md` | 领域术语的唯一权威定义（seam / scope / turn / step / round / goal / Ralph） | 术语歧义；写文档或代码前对齐用词 | 45 行 |
| `docs/cordis-primer.md` | Cordis 五个核心思想 + 四种 dispatch mode + waterfall 语义 + loader `!!js` | 「waterfall 不调 `next()` 会怎样」「`inject` 是干嘛的」「`emit`/`waterfall`/`parallel`/`serial` 区别」 | 44 行 |
| `docs/defensive-patterns.md` | 7 条已出过事故的 bug 类别规则（生命周期/并发/子进程/teardown） | 写 dispose、kill、异步状态、临时文件、symlink 删除之前 | 33 行 |
| `docs/config-catalog.md` | 每个可加载包的 `config:` 声明逐字粘贴 + `Requires:` 注入服务（generated） | 「`cordis.yml` 里这个插件能配什么字段」「它依赖哪些服务」 | 3317 行 · 约 109 个包节 + 3 个分类节 |
| `docs/tool-catalog.md` | 每个模型可见工具的 name/description/JSON-Schema + 工具包映射表（generated，真实 boot 后读 `ctx.tools.schemas()`） | 「模型看到的工具长什么样」「这个工具需要哪些 seam」「有没有 shipped alias」 | 2221 行 · 26 个包节 |
| `docs/persistence-catalog.md` | `SessionEventMap` 全部成员 + envelope 声明 + surface/log-only 徽章（generated） | 「会话日志里能出现哪些事件」「哪些事件进模型历史」「payload 字段是什么」 | 1007 行 · 48 个事件类型（仅 3 个 surface） |
| `docs/module-graph.md` | 226 个 `dsh-*` 包的 peerDependencies 依赖图 + 依赖表（generated） | 「这个包依赖谁 / 被谁依赖」「加依赖会不会成环」 | 1682 行 · 226 行依赖表 |
| `docs/graph-atlas.md` | 上面几张图的索引 + 各自 maintenance mode | 忘了哪张图在哪；确认某页是 generated/hybrid/curated | 24 行 |
| `docs/api-gateway.md` | Typert API Gateway 当前态参考：`@Remote`/`@RemoteScope`、生成管线、运行时调用、SRC fallback | 前后端 RPC；「加一个浏览器能调的 Host 方法」；`ctx.remote.<ns>` 的 `inject` 怎么写；改了 Remote 签名要重跑什么 | 164 行 |
| `docs/development.md` | 贡献者 setup + tsconfig Host/Client 双 aggregate + git hooks + CI + TODO 标记语义 | 「为什么有两个 tsconfig aggregate」「build 顺序为什么固定」「`FIXME`/`TODO`/`XXX` 怎么选」 | 173 行 |
| `docs/testing.md` | 测试分层（unit / coverage / e2e / snapshot / web）+ 各层硬规则 | 「我这个改动要补哪一层测试」「什么时候必须加 snapshot」「为什么 gate 是 `test:coverage` 不是 `test`」 | 49 行 |
| `docs/AGENTS.md` | 文档标准本身：tier 分类表、写作规则、字数预算、slop checklist | 「这条事实该写进哪个文件」「为什么 `verify-doc-budgets` 红了」 | 75 行 |
| `docs/rescope.md` | vendored Cordis 系列包的 upstream→`@deepseek-ai/*` 改名映射表 | 「`cordis` 包为什么 import 不到」「`declare module` 该写哪个名字」「哪些东西**不**改名」 | 53 行 |
| `docs/web-styling.md` | 浏览器端样式归属与组件规则（`--dsw-*` token、CSS Modules） | 写 `packages/client/*` 组件样式前；「颜色该从哪拿」 | 25 行 |

## 子目录路由（不在 19 主题内，但同样高频）

| 目录 | 内容 | 什么时候来 |
|---|---|---|
| `docs/subsystems/` | 46 个子系统参考页 + 每页的 generated `cordis-surface` region | 需要**类型定义 / 方法签名 / 事件精确签名**时——architecture.md 只给行为叙述，类型都在这里 |
| `docs/cookbook/` | 8 篇 step-by-step 指南（adding-a-package / -a-tool / -an-llm-adapter / -a-conversation-node / -a-settings-card / -a-vendored-package + extension-cookbook 索引） | 真要动手加东西时；`extension-cookbook.md` 是「功能→能力」的入口索引 |
| `docs/cordis-api/` | Cordis 核心 API 参考（context / events / fiber / registry / service / inherited） | 框架层 API 精确签名 |
| `docs/cordis-tutorial/` | 7 步 Cordis 手把手教程（01 first-plugin → 07 into-the-harness） | 完全不懂 Cordis 时的入门路径 |
| `docs/postmortem/` | 4 份事故复盘（0001-0004），唯一允许写「故事」的层 | 想知道某个防御规则的来历 |
| `docs/user/` | 面向产品用户的指南（`index.md` + `guide/` + `develop/`） | 用户视角的使用文档，不是贡献者文档 |
| `docs/i18n/` | 双语配对契约、术语表、翻译规则 | 改双语对时 |

## 路由前必须知道的三条元规则

1. **generated vs curated**：`config-catalog` / `tool-catalog` / `persistence-catalog` / `module-graph` 由 `scripts/gen-*.ts` 生成；`capability-seams` / `event-producer-consumer` 为 hybrid；`agent-lifecycle` / `tool-execution-pipeline` 由 `scripts/gen-doc-graphs.ts` 输出但内容是 curated。**这些文件禁止手改**，改源码后跑 `pnpm run gen-doc-graphs` / 对应 `gen-*`，`pnpm run doc-sync` 统一验证新鲜度。
2. **一个事实只有一个家**（`docs/AGENTS.md` 的 tier 表）：类型定义→`subsystems/`，行为叙述→`architecture.md`，决策理由→`.agents/notes/`，每包契约→包 README，事故→`postmortem/`。找不到就按这条反推该去哪个 tier。
3. **`.zh.md` 是翻译对，不是独立事实源**。每个英文文档都有 `<name>.zh.md` + `<name>.i18n.yaml` 配对记录；查事实一律读 `.md`，`.zh.md` 只在确认中文措辞时才读。

## 快速反向索引（问题 → 文件）

- ctx key 叫什么 → `capability-seams.md`
- 这个事件谁在听 → `event-producer-consumer.md`
- 这个事件的 payload → `persistence-catalog.md`（durable）或 `subsystems/<x>.md`（live Cordis 事件签名）
- 这个插件能配什么 → `config-catalog.md`
- 模型看到的工具 schema → `tool-catalog.md`
- 事件的先后顺序 → `agent-lifecycle.md`（turn/step）、`tool-execution-pipeline.md`（工具内）
- 新行为挂在哪 → `architecture.md` 末尾扩展点表 → `cookbook/extension-cookbook.md`
- 包之间依赖 → `module-graph.md`
- 术语到底啥意思 → `glossary.md`
