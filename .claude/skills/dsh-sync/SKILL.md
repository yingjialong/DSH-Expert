---
name: dsh-sync
description: 同步 DSH 上游镜像并执行知识防腐——拉取 deepseek-harness 与 cordis 最新代码、diff 知识库锚点、把受影响条目批量标记为 stale、生成版本变更摘要、补学变更部分、重试悬而未决问题。在用户输入 /dsh-sync 时触发，或在 dsh skill 的新鲜度自检发现本地镜像落后于远端时由其调用。
---

# dsh-sync — 同步上游与知识防腐

## 为什么这个 skill 是刚需

rc.7 → rc.8 之间（两天）**1604 个文件变更、+54064 / −10533 行**。不同步的知识库不是"稍旧"，而是**会给出错误答案**。

## 执行流程

### 步骤 1 — 记录旧基线

```bash
cd "$(git rev-parse --show-toplevel)"
OLD_DSH=$(git -C upstream/deepseek-harness rev-parse HEAD)
OLD_CORDIS=$(git -C upstream/cordis rev-parse HEAD)
echo "OLD_DSH=$OLD_DSH"; echo "OLD_CORDIS=$OLD_CORDIS"
```

### 步骤 2 — 拉取

```bash
git -C upstream/deepseek-harness fetch --all --tags --quiet && git -C upstream/deepseek-harness pull --ff-only --quiet
git -C upstream/cordis fetch --all --tags --quiet && git -C upstream/cordis pull --ff-only --quiet
NEW_DSH=$(git -C upstream/deepseek-harness rev-parse HEAD)
git -C upstream/deepseek-harness log -1 --format='%H %cI %s'
git -C upstream/deepseek-harness tag | tail -5
```

若 `pull --ff-only` 失败（上游 force-push 或本地被改动），**不要强推**：报告并改用 `git reset --hard origin/master`（本地镜像是只读事实源，无需保留本地改动）。

### 步骤 3 — 变更概览

```bash
git -C upstream/deepseek-harness diff --shortstat $OLD_DSH..$NEW_DSH
git -C upstream/deepseek-harness log --oneline $OLD_DSH..$NEW_DSH | head -40
# 新 tag / release
git -C upstream/deepseek-harness log --oneline --grep='release(' $OLD_DSH..$NEW_DSH
# 破坏性信号
git -C upstream/deepseek-harness log --oneline $OLD_DSH..$NEW_DSH | rg -i 'breaking|rename|remove|deprecat|refactor!'
```

### 步骤 4 — 锚点 diff，批量标 stale（核心）

收集知识库全部锚点，与本次变更文件取交集：

```bash
cd "$(git rev-parse --show-toplevel)"
# 本次变更的文件列表
git -C upstream/deepseek-harness diff --name-only $OLD_DSH..$NEW_DSH > /tmp/dsh-changed.txt
# 知识库里出现过的锚点路径
rg -oN '^\s*-\s+([a-z0-9_./-]+\.(ts|py|md|yml|yaml|json))' -r '$1' wiki/ | sort -u > /tmp/dsh-anchors.txt
comm -12 <(sort -u /tmp/dsh-changed.txt) /tmp/dsh-anchors.txt
```

对**每一个命中的锚点**：

1. 找到引用它的 wiki 页，把 frontmatter 的 `freshness` 改为 `stale`
2. 在 `wiki/index.md` 对应行标注 `[stale]`
3. 看具体 diff 判断是否真的影响结论：
   ```bash
   git -C upstream/deepseek-harness diff $OLD_DSH..$NEW_DSH -- <锚点文件>
   ```

**时间兜底**：`verified_at` 距今超过 30 天的条目，即使锚点未变也标 `stale`。

### 步骤 5 — 补学变更部分（主动学习的自动档）

只补学**"上游变了且我已沉淀过"**的部分——这是已知会腐烂的区域，目标明确、成本可控：

1. 逐条复验步骤 4 标出的 stale 条目，更新结论、`commit`、`verified_at`，改回 `fresh`
2. 结论被推翻的，**写入 `wiki/errors.md`**（记录：旧结论、为何失效、哪个 commit 改的）
3. 更新 `wiki/coverage.md` 中相关领域的核验时间

**全新领域不在此自动学习**——那必须由 `/dsh-learn` 手动触发并当场汇报。

### 步骤 6 — 抽查历史问答（替代独立回归集）

```bash
rg -n '锚点|anchors' wiki/log.md | head -50
```

对锚点被本次变更命中的**历史问答**做复验：当时的答案在新版本下是否仍成立？不成立则更新对应 wiki 页并记入 `errors.md`。

### 步骤 7 — 重试悬而未决

读 `wiki/open-questions.md`，对标记为"可能被新版本解答"的条目重新查证。解决的移入 wiki 正文并从该文件删除。

### 步骤 8 — 生成版本变更摘要

若本次跨了 rc 版本，在 `wiki/topics/` 下建/更新 `版本变更-rcX-到-rcY.md`，记录：

- 破坏性变更（重命名、删除、签名变化、配置键变化）
- 新增能力（新包组、新工具、新配置项）
- 对**集成方**的影响（按八维坐标分类：哪些宿主形态会受影响）
- 升级建议

### 步骤 9 — 更新基线与汇报

更新 `CLAUDE.md` 版本基线节与 `README.md` 的 Key facts / 关键事实表（若 npm `latest` tag 变了）：

```bash
curl -s "https://registry.npmjs.org/@deepseek-ai%2Fdsh" | python3 -c "import sys,json;print(json.load(sys.stdin)['dist-tags'])"
curl -s "https://pypi.org/pypi/deepseek-harness-sdk/json" | python3 -c "import sys,json;print(json.load(sys.stdin)['info']['version'])"
```

向用户汇报（固定格式）：

```
同步完成：<OLD短SHA> → <NEW短SHA>（N 次提交，M 文件变更）
新版本：npm latest=<x> next=<y> · PyPI sdk=<z>
破坏性变更：<列表或"未发现">
标记 stale：<n> 条 · 已复验修复：<m> 条 · 结论被推翻：<k> 条（已记入 errors.md）
悬而未决解决：<j> 条
覆盖度变化：<领域> Lx→Ly
```

### 步骤 10 — 提交

```bash
cd "$(git rev-parse --show-toplevel)"
git add -A && git commit -q -m "sync: 上游 <旧短SHA> → <新短SHA>，标记 <n> 条 stale"
```

知识库的每次变动都必须可 diff、可回滚。
