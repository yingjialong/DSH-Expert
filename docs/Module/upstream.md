# 模块：上游镜像（upstream/）

## 一、功能

DSH 与 Cordis 的本地只读克隆，是知识库的 **T1 唯一事实源**。所有 DSH 事实的终审依据都在这里。

已 gitignore：它们有自己的 git 历史，不进本项目仓库。

## 二、镜像清单

| 仓库 | 路径 | 克隆命令 |
| --- | --- | --- |
| deepseek-ai/deepseek-harness | `upstream/deepseek-harness` | `git clone --filter=blob:none https://github.com/deepseek-ai/deepseek-harness.git upstream/deepseek-harness` |
| cordiverse/cordis | `upstream/cordis` | `git clone --filter=blob:none https://github.com/cordiverse/cordis.git upstream/cordis` |

克隆后的基线由 `/dsh-sync` 维护，当前知识库基线见 `CLAUDE.md` 版本基线节。

### 2026-09-05 同步与多版本入口

| 用途 | 版本 / 固定完整 SHA |
| --- | --- |
| 默认咨询（npm `latest/next`） | `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d` |
| GitHub 最新版与本次 master | `0.1.3-alpha.1` / `d347e703908d0406b7a7ef80e3a0e594d86b2215` |
| 历史 rc.2 | `0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e` |
| 历史 alpha.5（npm `alpha`） | `0.1.2-alpha.5` / `db6bdc3576c2d4e7c965e8e3ed0c2a731eed87f5` |
| Cordis 独立镜像 | `2ceea231802cc23892b4ad10012c55c7dd4982d4`；DSH 自带内核仍从对应 DSH SHA 的 `vendor/cordis/` 核验 |

11 个 `dsh-v*` tag 已逐一核对远端 SHA 和本地 manifest。源码读取使用 `git -C <镜像绝对路径> show <SHA>:<文件路径>` 或 `git grep ... <SHA> -- <路径>`；禁止为不同咨询切换共享工作目录。新旧三个基线的 `AgentLoop.create` / `SessionPersistence.create` 并发读取、rc.2 专题的 16 个锚点和缺失文件负对照均通过；详细结果见 `wiki/log.md` 的本次同步记录。历史 blob 为按需缓存，本次验证不承诺全部历史文件均已离线缓存，也不代表 runtime 安装或运行验证。

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

- 核对工作区与跟踪分支后 fast-forward；DSH 当前跟踪 `origin/master`，Cordis 跟踪 `origin/main`。失败时保留内容并报告，不得用 `reset --hard` 覆盖
- 每次同步后必须做锚点 diff，否则同步等于白做
- 新基线变动与固定历史版本分开判定：保留历史 tag 和已验证条目的版本归属，未完成复验的通用页保持 stale

## 六、实测所需工具链（上游要求）

| 上游要求 | 说明 |
| --- | --- |
| node `^22.19.0 \|\| >=24.0.0` | 上游 `package.json` engines；实测 DSH 本体前先核对 |
| `pnpm@11.7.0`（`packageManager`） | 与本机版本不符时用 `corepack` 切换，再跑上游测试 |
| Python ≥ 3.10 | Python SDK（`deepseek-harness-sdk`）要求 |

诊断类实测（配置解析、插件加载、启动日志、报错文本）通常不需要跑上游完整构建，只装 `@deepseek-ai/dsh` 即可。
