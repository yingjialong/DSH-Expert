# Contributing / 贡献指南

> **EN**: This project's contributions are mostly *knowledge entries*, not code. Every entry must carry verifiable upstream anchors and an honest epistemic status. Read `CLAUDE.md` first — it is the constitution of this knowledge base. The Chinese sections below are authoritative.

本项目的主要贡献形式是**知识条目**（wiki 页、playbook、错误本条目），其次才是机制与工具的改进。知识的可信度是本项目唯一的资产，因此贡献门槛围绕"可验证"设立。

## 一、动手前必读

1. **`CLAUDE.md`** —— 12 条硬约束是本知识库的宪法，尤其：
   - 第 4 条：信源分级，源码（T1）是唯一终审
   - 第 5 条：认知状态标注与掌握等级封顶（L1/L2 不得标 `fact`）
   - 第 7 条：沉淀双门槛（主题准入 + 去项目化）
   - 第 6 条：`stale` 条目必须复验后才能引用
2. **`wiki/index.md`** —— 现有知识地图，避免重复沉淀
3. **`docs/Module/knowledge-base.md`** —— 知识库结构与页面契约

## 二、知识条目的硬规则

| 规则 | 说明 |
| --- | --- |
| **必须有锚点** | 每条结论绑定 `upstream/` 中的具体文件（最好到符号），并记录核验时的 commit SHA。没有锚点的结论无法参与失效检测，等于埋雷 |
| **必须标认知状态** | `fact` / `verified_inference` / `hypothesis` / `speculation`；未验证的推断不得伪装成事实 |
| **只收 DSH 知识或本库元知识** | 与 DSH 无关的内容（其他项目经验、无关技术结论）即使写得再通用也不收——把 DSH 从知识的主题里拿掉它仍成立，就不属于这里 |
| **去项目化** | 剥离项目名、私有路径、业务逻辑。集成知识写成「条件（八维坐标）→ 结论 → 锚点 → 认知状态」 |
| **不复制上游文档正文** | 只写上游没有的三类东西：路由（问题 → 去哪找）、踩坑与实测结论、跨文档综合推理 |
| **答错要记账** | 结论被推翻时写入 `wiki/errors.md`，并反查是哪条既有沉淀污染了结论 |

## 三、frontmatter 契约

内容页（`wiki/packages/`、`wiki/topics/`、`wiki/integration/`）八项必填：

```yaml
title / status / mastery / freshness / anchors / commit / verified_at / asked_by
```

治理文件（index / log / coverage / errors / open-questions / conflicts）豁免 frontmatter。

## 四、提交前自查

```bash
# 锚点存在性：每个 anchor 必须能在 upstream/ 中找到
cd "$(git rev-parse --show-toplevel)"
rg -oN '^\s*-\s+([a-z0-9_./-]+\.(ts|py|md|yml|yaml|json))' -r '$1' wiki/ | sort -u | while read -r a; do
  [ -e "upstream/deepseek-harness/$a" ] || [ -e "upstream/cordis/$a" ] || echo "MISSING ANCHOR: $a"
done
```

或直接在 Claude Code 中运行 `/dsh-wiki` 做 lint（格式、锚点、封顶规则、主题门槛、查重）。

## 五、PR 流程

1. Fork + 分支（`knowledge/<主题>` 或 `feat/<机制>`）
2. 保证 `/dsh-wiki` lint 全绿
3. PR 描述中说明：涉及的上游 commit 基线、新增/修改的知识条目、认知状态判定依据
4. 机制类改动（skill / 目录结构 / 文档体系）同步更新 `docs/Module/` 对应模块文档

## 六、不接受什么

- 无锚点、无法回源码验证的结论
- 滥标 `fact`（尤其 L1/L2 领域）
- 复制上游文档正文充数
- 与 DSH 无关的知识（见 `CLAUDE.md` 第 7 条主题门槛）
- 未经实测就写"必然抛错 / 必然不可能"级别的强断言（先跑一行最小复现，见 `wiki/errors.md` E012 的教训）
