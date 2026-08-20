# 模块：上游镜像（upstream/）

## 一、功能

DSH 与 Cordis 的本地只读克隆，是知识库的 **T1 唯一事实源**。所有 DSH 事实的终审依据都在这里。

已 gitignore：它们有自己的 git 历史，不进本项目仓库。

## 二、镜像清单

| 仓库 | 路径 | 建库时 HEAD | 占用 |
| --- | --- | --- | --- |
| deepseek-ai/deepseek-harness | `upstream/deepseek-harness` | `141eb6fef83422698aef7a981029e843e8161534`（2026-08-19T23:11:50+08:00） | 110M |
| cordiverse/cordis | `upstream/cordis` | `8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4`（2026-08-13T21:48:18+08:00） | 1.5M |

## 三、克隆策略：为什么是 blobless

```bash
git clone --filter=blob:none <url> upstream/<name>
```

| 方案 | 工作树可 grep | 历史可 diff | 磁盘 | 采用 |
| --- | --- | --- | --- | --- |
| 完整克隆 | ✅ | ✅ | 最大 | ❌ 没必要 |
| **blobless（`--filter=blob:none`）** | ✅ | ✅（历史 blob 按需拉） | 中 | ✅ |
| shallow（`--depth=1`） | ✅ | ❌ | 最小 | ❌ 无法做版本 diff |
| sparse-checkout | 部分 | ✅ | 小 | ❌ 提问不设限，排除任何目录都会造成盲区 |

关键点：blobless 克隆**会完整拉取 checkout 所需的 blob**，所以当前工作树是完整的、可以随便 grep；只有**历史版本的 blob** 才是按需拉取。这正好匹配需求：日常查当前代码要快，偶尔做版本 diff 时容忍一次网络往返。

版本 diff 能力是刚需——`rc.7 → rc.8` 两天内 1604 个文件变更，"这个版本和那个版本差在哪"是高频问题。

## 四、上游的高价值知识源

| 位置 | 数量 | 价值 |
| --- | --- | --- |
| `packages/<group>/README.md` | 50 组 | 强制包含 purpose / APIs / extension points / Model Experience / **Known Limitations** 章节 |
| `packages/README.md` | 1 | 包组总表 + **稳定性分级**（Product-stable / POC / Unreleased / Support）。⚠️ 漏列 `mcp` 与 `runtime-diagnostics` |
| `docs/*.md` | 19 主题 | 架构、生命周期、工具目录、配置目录、术语表等 |
| `docs/subsystems/` | 20 篇 | 子系统细节 |
| `docs/cookbook/` | 9 篇 | 扩展开发的操作手册 |
| `docs/cordis-api/` + `docs/cordis-tutorial/` | 6 + 8 篇 | 内核 API 与渐进教程 |
| `docs/postmortem/` | 4 篇 | **真实事故复盘**，知识库最值钱的内容之一 |
| `docs/module-graph.md` | 1 | **由 `pnpm run gen-module-graph` 生成，CI 校验新鲜度** —— 权威且自动更新的依赖图 |
| `docs/user/guide/` `docs/user/develop/` | 13 篇 | 面向使用者与插件作者 |

所有文档均有 `.zh.md` 中文版，但**以英文 `.md` 为准**（翻译可能滞后）。

## 五、维护

由 `dsh-sync` skill 负责。要点：

- `pull --ff-only` 失败时**不要强推**，直接 `git reset --hard origin/master`（本地镜像是只读事实源，没有需要保留的本地改动）
- 每次同步后必须做锚点 diff，否则同步等于白做

## 六、本机工具链（用于实测）

git 2.50.1 / node v22.22.3 / pnpm 10.13.1 / python 3.12.8 / uv 0.7.20

| 上游要求 | 本机 | 状态 |
| --- | --- | --- |
| node `^22.19.0 \|\| >=24.0.0` | v22.22.3 | ✅ |
| `pnpm@11.7.0` | 10.13.1 | ⚠️ 不匹配，实测前需 corepack 切换 |
| Python SDK 要求 `>=3.10` | 3.12.8 | ✅ |

## 七、变更记录

| 日期 | 变更 |
| --- | --- |
| 2026-08-20 | 首次克隆 DSH 与 Cordis |
