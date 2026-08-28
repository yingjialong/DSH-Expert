# AGENTS.md — DSH-Expert

> 本文件是遵循 AGENTS.md 约定的宿主（OpenAI Codex CLI 等）在本项目的入口。**约束以 `CLAUDE.md` 为准、工作流以三个 SKILL.md 为准**，本文件只做引导、不承载独立事实——修改 `CLAUDE.md` 或 skill 后，回看本文件的要点速览与场景映射是否仍成立。

## 第一步：读取宪法

完整读取 **`CLAUDE.md`** 并遵守其全部 13 条硬约束。要点速览（细则以 `CLAUDE.md` 为准）：

- **第 0 条**：你对 DSH 的记忆全是幻觉（DSH 发布晚于训练截止），任何 DSH 事实必须现查
- **第 3 条**：固定回答流程——新鲜度自检 → 查库（index → errors → coverage）→ 回 `upstream/` 源码核验 → 作答 → 沉淀
- **第 5/6 条**：认知状态标注与掌握等级封顶；`stale` 条目必须复验后才能引用
- **第 7 条**：沉淀双门槛（主题准入 + 去项目化）——只收 DSH 知识、本库语境的排障记录与本项目元知识
- **第 11 条**：实测边界——`sandbox/` 最小复现与只读操作预授权；装全局软件、写项目目录之外的文件、常驻进程、对外写操作、**任何消耗凭据的模型调用**必须先停下说明
- **第 13 条**：跨会话问题必须通过宿主的跨会话回复能力，把完整答复回传 `source_thread_id` 对应的来源会话并确认成功

本文件与 `CLAUDE.md` 冲突时，以 `CLAUDE.md` 为准。

## 第二步：按场景执行对应工作流

三个 SKILL.md 是普通 Markdown 流程文档，直接读取并照做。Claude Code 宿主会自动加载它们；在 Codex 中没有自动触发机制，由你按下表对号入座：

| 场景 | 读取并执行 | Claude Code 中的等价触发 |
| --- | --- | --- |
| 任何 DSH 问题（用法/配置/报错/插件/集成/升级/对比） | `.claude/skills/dsh/SKILL.md`（六步流程，不可跳步） | `dsh` skill 自动触发，或 `/dsh` |
| 用户要求同步上游，或新鲜度自检发现本地镜像落后 | `.claude/skills/dsh-sync/SKILL.md`（十步流程） | `/dsh-sync` |
| 知识库维护（lint/查重/复验）；用户问「你对 DSH 懂多少」「知识库健康吗」；用户说「去学一下 X」 | `.claude/skills/dsh-wiki/SKILL.md`（维护/学习双模式，健康报告见 A7） | `/dsh-wiki`、`/dsh-learn` |

SKILL.md 中的命令块用 `git rev-parse --show-toplevel` 定位仓库根，对任何宿主可移植。

**Codex 沙箱差异**：Codex 默认沙箱禁网且只读。`git ls-remote`（新鲜度自检）、`git fetch`、`curl`（dsh-sync 步骤 2/9）等网络命令失败时，**优先向用户申请网络/写盘放行，而不是直接落入「本地镜像可能落后」的降级分支**——新鲜度自检是本项目答案可信度的唯一硬保障，被沙箱静默掏空等于失效。`/tmp` 临时文件与 git commit 在只读模式下同样需要用户放行。

## 开始前的自检

- [ ] 已完整读取 `CLAUDE.md`
- [ ] 当前目录在本仓库根，且 `upstream/deepseek-harness/` 与 `upstream/cordis/` 存在（不存在则提示用户按 README Quick Start 克隆）
- [ ] `rg`（ripgrep）可用——工作流命令块依赖它；缺失时提示安装（`brew install ripgrep`），或以 `grep -rn` 等价替代
- [ ] 提问者可能是人类也可能是另一个 session 的 agent——沉淀时记 `asked_by`，且不得推翻 `fact` 级条目（`CLAUDE.md` 第 12 条）
- [ ] 请求携带 `source_thread_id` 或等价来源标识时，已按 `CLAUDE.md` 第 13 条规划跨会话回传，不能只在当前会话输出 final
