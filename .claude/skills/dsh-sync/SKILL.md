---
name: dsh-sync
description: 同步 DSH 上游镜像并执行知识防腐——拉取 deepseek-harness 与 cordis 最新代码、diff 知识库锚点、把受影响条目批量标记为 stale、生成版本变更摘要、补学变更部分、重试悬而未决问题。只在用户明确输入 /dsh-sync，或明确要求同步、更新上游镜像时触发；只读检测发现新版本不得自动调用本 skill。
---

# dsh-sync — 同步上游与知识防腐

## 授权前置

本 skill 会修改本地上游镜像、知识库与版本基线，**只有用户针对当前任务明确要求同步或更新时才能执行**。`git ls-remote`、GitHub/npm/PyPI 最新版本查询等只读检测不构成调用本 skill 的授权。

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
git -C upstream/deepseek-harness fetch --all --tags --quiet && git -C upstream/deepseek-harness merge --ff-only --no-stat '@{upstream}'
git -C upstream/cordis fetch --all --tags --quiet && git -C upstream/cordis merge --ff-only --no-stat '@{upstream}'
NEW_DSH=$(git -C upstream/deepseek-harness rev-parse HEAD)
git -C upstream/deepseek-harness log -1 --format='%H %cI %s'
git -C upstream/deepseek-harness tag | tail -5
```

执行前核对镜像工作区与跟踪分支，不假定两个仓库都使用 `master`。无跟踪分支或 fast-forward 失败时报告原因，保护已有内容，不得用 `reset --hard` 覆盖。保留历史 tag / commit；按 `AGENTS.md` 第 2 条，同步不得改变其他进行中咨询已固定的 SHA。

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

对**每一个命中的锚点**，先判断条目的版本归属：

1. 明确固定历史版本且 `commit`、锚点仍可核对的条目，按原 SHA 保留；新 HEAD 同路径变动不使旧快照失效。动态发布状态另行复验，不能拿 master 改动覆盖 release tag 的结论。
2. 跟随新基线的通用条目或含已过期动态断言的条目，把 `freshness` 改为 `stale`，并在 `wiki/index.md` 标注。分别统计原已 stale、本轮新增 stale、固定版本保留的条目。
3. 看固定两端 SHA 的 diff 判断影响；历史条目必须使用其自己的 `commit`，不能只比较两个 master 快照就宣称已复验：
   ```bash
   git -C upstream/deepseek-harness diff $OLD_DSH..$NEW_DSH -- <锚点文件>
   ```

**时间兜底**：`verified_at` 距今超过 30 天的条目，即使锚点未变也标 `stale`。

### 步骤 5 — 补学变更部分（主动学习的自动档）

只补学**"上游变了且我已沉淀过"**的部分——这是已知会腐烂的区域，目标明确、成本可控：

1. 逐条复验步骤 4 本轮新增的 stale 条目，更新结论、`commit`、`verified_at`，改回 `fresh`；固定版本条目保持原版本归属。原已 stale 且未完成整页复验的内容保持 stale，只作检索路由，不宣称支持新基线的直接引用。
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

更新规则主文件 `AGENTS.md` 的版本基线节、`wiki/index.md` 与 `README.md` 双语事实表，保留 `CLAUDE.md` 软链接。分别记录 npm 默认基线、GitHub 最新 release、master 与旧版本的完整 SHA，不能假定渠道同步发布：

```bash
curl -s "https://registry.npmjs.org/@deepseek-ai%2Fdsh" | python3 -c "import sys,json;print(json.load(sys.stdin)['dist-tags'])"
curl -s "https://pypi.org/pypi/deepseek-harness-sdk/json" | python3 -c "import sys,json;print(json.load(sys.stdin)['info']['version'])"
```

交付前按固定 SHA 验证默认版、GitHub 最新版与至少一个旧版的 manifest 和实际接口；确认旧 tag 保留、共享 HEAD 未因咨询切换，旧版缺失文件不能回退读取当前工作树。此检查只证明源码咨询路径可用；是否执行 runtime/install 测试须另行如实记录。

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
