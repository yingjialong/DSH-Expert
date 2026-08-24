---
name: dsh-wiki
description: 维护 DSH 知识库本身——健康检查、lint 格式与 frontmatter、查重与合并、补交叉链接、批量复验 stale 条目、审计 agent 提问产生的沉淀、生成覆盖度报告；也承载主动学习 /dsh-learn（手动指定领域，学完汇报覆盖度变化）。在用户输入 /dsh-wiki、/dsh-learn，或询问"你现在对 DSH 懂多少""知识库健康吗""去学一下 XXX"时触发。
---

# dsh-wiki — 知识库维护与主动学习

两种模式：**维护模式**（`/dsh-wiki`）与**学习模式**（`/dsh-learn <领域>`）。

---

## 模式 A：维护（`/dsh-wiki`）

### A1 — lint：格式与 frontmatter 合规

对 `wiki/` 下**内容页**（`packages/`、`topics/`、`integration/`）逐个检查，违规必修。治理文件（index / log / coverage / errors / open-questions / conflicts）豁免 frontmatter 检查，只查内容质量：

| 检查项 | 要求 |
| --- | --- |
| frontmatter 齐全 | `title / status / mastery / freshness / anchors / commit / verified_at / asked_by` 八项必填 |
| **锚点非空且真实存在** | 每个 anchor 必须能在 `upstream/` 中找到；找不到 → 标 `stale` 并复验 |
| **封顶规则** | `mastery` 为 L1/L2 的页面**不得**出现 `status: fact` → 降级为 `verified_inference` |
| 术语未被翻译 | 包名 / 配置键 / API 名保持英文原文 |
| 未复制上游正文 | 抽查段落，若与 `upstream/docs/` 高度重复 → 删除并改为路由链接 |
| 单页长度 | 超过 ~400 行则拆分 |
| **主题门槛** | 正文必须属于 DSH 知识或本项目元知识（`CLAUDE.md` 第 7 条）：与 DSH 无关的教程式 / 调优式段落、结论主语不是 DSH 的独立断言 → 删除或收敛为一句前提 |
| 去项目化 | 不得出现具体项目名、私有路径、业务逻辑 |

锚点存在性校验：

```bash
cd "$(git rev-parse --show-toplevel)"
rg -oN '^\s*-\s+([a-z0-9_./-]+\.(ts|py|md|yml|yaml|json))' -r '$1' wiki/ | sort -u | while read -r a; do
  [ -e "upstream/deepseek-harness/$a" ] || [ -e "upstream/cordis/$a" ] || echo "MISSING ANCHOR: $a"
done
```

### A2 — 查重与合并

同一事实散落在多页会导致更新时漏改一处，进而自相矛盾。找出重复主题，合并为一页 + 其余页改为交叉链接 `[[页面名]]`。

### A3 — 补交叉链接

孤立页（无人链入也不链出）是索引失效的信号。为每页补上：所属包组页、相关 topic 页、相关 playbook。

### A4 — 批量复验 stale

按优先级复验 `freshness: stale` 的条目：

1. `asked_by: agent` 的（无人纠错过，风险最高）
2. `status: fact` 的（被信任度最高，错了危害最大）
3. 被 `wiki/log.md` 引用次数多的（高频知识）
4. 其余

复验 = 回 `upstream/` 源码核对 → 更新结论/`commit`/`verified_at` → 改回 `fresh`。结论被推翻的写入 `wiki/errors.md`。

### A5 — 审计 agent 批次

```bash
rg -l 'asked_by: agent' wiki/
```

逐条复核。发现 agent 会话中**推翻过 `fact` 级条目**的（违反 `CLAUDE.md` 第 12 条），或**写入过与 DSH 无关内容**的（违反第 7 条主题门槛），回滚该改动并记入 `errors.md`。

### A6 — 矛盾检测

跨页扫描互相冲突的结论（LLM Wiki 模式的核心健康指标）。发现冲突：
- 回 T1 源码裁决
- 更新错误的一方
- 把"曾经存在的冲突"记入 `wiki/errors.md`（防止再次分叉）

### A7 — 覆盖度报告

重算 `wiki/coverage.md`，向用户汇报：

```
知识库健康报告
规模：<n> 页 · <m> 条锚点
掌握度分布：L0 <a> / L1 <b> / L2 <c> / L3 <d> / L4 <e>
新鲜度：fresh <f> / stale <g>
认知状态：fact <h> / verified_inference <i> / hypothesis <j> / speculation <k>
本次修复：lint <p> 处 · 合并 <q> 页 · 复验 <r> 条 · 发现矛盾 <s> 处
最大盲区（建议下一步学）：<领域列表>
悬而未决：<n> 条
```

### A8 — 提交

```bash
git add -A && git commit -q -m "wiki: 健康检查，复验 <r> 条，修复 <p> 处"
```

---

## 模式 B：主动学习（`/dsh-learn <领域>`）

**只在手动触发时运行。禁止后台无监督自学**——那是"猜测被写成事实"的温床。

### B1 — 确定目标与当前等级

先过 `CLAUDE.md` 第 7 条主题门槛：领域必须是 DSH 相关（或本项目自身），否则拒绝执行并在 `log.md` 留一行拒绝轨迹。通过后在 `wiki/coverage.md` 中查该领域当前等级，明确本次要达到的目标等级。

### B2 — 按等级定义执行

| 目标 | 做什么 | 可给出的最高认知状态 |
| --- | --- | --- |
| **L1** | 读 `packages/<group>/README.md`（含稳定性分级与 Known Limitations）、对应 `docs/` 主题 | `verified_inference` |
| **L2** | 读源码关键路径：Service Definition（seam）、Provider、Consumer 三件套；接口、扩展点、数据流；`peerDependencies` 依赖关系 | `verified_inference` |
| **L3** | 实测验证（见 `CLAUDE.md` 第 11 条边界）：在 `sandbox/` 跑最小复现、跑上游单测、看真实报错 | `fact` |
| **L4** | 记录边界、失败模式、与文档不符之处 → 写入 `errors.md` | `fact` |

DSH 的 capability family 普遍是三件套结构（Service Definition / Service Provider / Consumer），L2 学习必须把这三层分清，因为**扩展插件应依赖 Service Definition 而非具体 Provider**——这是最容易搞错的地方。

### B3 — 学习优先级（当用户未指定领域时）

1. **Cordis 内核**（`docs/cordis-primer.md`、`docs/cordis-api/`、`upstream/cordis`）—— 不懂它答不了任何插件问题
2. **集成表面与协议层**（`packages/sdk` 的 JSON-RPC、`packages/acp`、`packages/host`、`python/sdk`、`docs/api-gateway.md`）—— 通用性杠杆，一次投入对所有宿主栈有效
3. **高稳定性核心链**（`core` / `session` / `llm` / `preset` / `skill` / `fs` / `shell`）
4. 其余 Product 包组
5. `experimental/`（Unreleased）、`e2b/`（POC）—— 最后

叠加动态调整：`wiki/log.md` 中被问过的领域自动提级（问过一次就会问第二次）。

### B4 — 沉淀与汇报

按 `CLAUDE.md` 写作规范写回，更新 `coverage.md`（不得新增非 DSH 领域单元），然后汇报：

```
学习完成：<领域> Lx→Ly
新增/更新：<n> 页 · 新锚点 <m> 条
关键发现：<3-5 条要点>
与文档不符：<列表或"无">（已记入 errors.md）
新增悬而未决：<列表或"无">
```
