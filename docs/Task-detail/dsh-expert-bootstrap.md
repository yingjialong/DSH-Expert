# 任务：dsh-expert-bootstrap

| 项 | 值 |
| --- | --- |
| 任务名称 | dsh-expert-bootstrap |
| 开始时间 | 2026-08-20 |
| 状态 | 进行中 |

## 一、任务背景

用户需要一个可长期依赖的 **DSH（DeepSeek Harness）专家**，要求：对外能回答任何 DSH 问题且**答案最新可靠**；对内能**自我学习、自我更新**（含知识库更新）、能自主上网查可靠信源。

经过 5 轮需求盘问（grilling），确立了完整设计。核心事实前提：

- DSH 于 **2026-08-13** 发布，developer preview，**约两天一个 rc**，官方明确警告破坏性变更。
- 助手训练数据截止于 DSH 发布之前 → **对 DSH 的一切"记忆"都是幻觉**。
- 上游有高质量结构化知识源（50 个包组 README 带稳定性分级、19 个顶层 doc 主题、20 篇 subsystems、cordis 教程/API、4 篇 postmortem），且模块图由 `pnpm run gen-module-graph` 生成、CI 校验新鲜度。

## 二、现有问题

| 问题 | 后果 |
| --- | --- |
| 凭记忆回答 | 100% 幻觉 |
| 手写知识摘要 | 几天内腐烂，且腐烂得悄无声息 |
| LLM Wiki 原始模式假设源材料静态 | 直接照搬会让"agent 的猜测"自我强化成"权威结论" |
| 提问者可能不是人类 | 人工审核闸门失效，错误知识无人纠正 |

## 三、验收标准

1. 存在可运行的 `dsh` / `dsh-sync` / `dsh-wiki` 三个 skill，`dsh` 可自动触发。
2. `CLAUDE.md` 落地 12 条硬约束（含"禁止凭记忆回答 DSH"）。
3. `wiki/` 五个治理文件齐备：`index.md` / `log.md` / `coverage.md` / `errors.md` / `open-questions.md`。
4. `playbooks/` 四本方法层剧本齐备。
5. 上游 DSH 与 Cordis 完成 blobless 克隆，可 grep、可 diff、可溯源。
6. 冷启动阶段一：**全域 L1**（50 个包组 + 全部 doc 主题在 `coverage.md` 中有条目与等级）。
7. 冷启动阶段二：要害领域达 **L2**（Cordis 内核、协议层、两个官方 SDK、核心链）。
8. `coverage.md` 的真实结果向用户汇报。

## 四、非目标

- 不实测运行 DSH（L3 留到真实提问触发或用户要求）。
- 不建独立 evals 回归集（改为复用 `log.md` + 锚点抽查）。
- 不复制上游文档正文进知识库。
- 不针对任何具体宿主项目做定制（知识必须去项目化）。

## 五、设计决策（5 轮盘问结论）

| # | 决策 | 结论 |
| --- | --- | --- |
| Q1 | 使用画像 | 使用者 + 插件作者 + **通用集成专家**（不绑定任何技术栈） |
| Q2 | 提问面 | 不设限 |
| Q3 | 知识底座 | 本地克隆为唯一事实源 + LLM Wiki 式沉淀（越问越聪明） |
| Q4 | 实测能力 | 具备，必要时直接做（见 Q34/Q35 边界） |
| Q5 | 作用域 | Cordis 纳入；第三方插件只做索引+可靠性标注；横向对比须标注主观 |
| Q6 | 信源分级 | T1 源码 > T2 上游 docs/官方站 > T3 Release/Discussions/Issues > T4 第三方；冲突以源码为准；英文 `.md` 为准 |
| Q7 | 更新机制 | 回答前秒级新鲜度自检 + 手动 `/dsh-sync` 重活；不开后台 cron |
| Q8 | 回答契约 | 禁止凭记忆、不确定必须明说、标注版本基线与出处 |
| Q10 | 版本基线 | **rc.7**（npm latest / PyPI 唯一可用），可回答 master/rc.8 差异 |
| Q11 | 条目 schema | status / anchors / commit / verified_at / freshness |
| Q12 | 失效检测 | 锚点 diff 标 stale（+30 天未核验兜底）；stale 必须复验才能引用 |
| Q13 | 错误本 | 建 `wiki/errors.md` |
| Q14 | wiki 结构 | 骨架映射上游真实结构；index 只放一行摘要；单页设上限；log.md 记时间线 |
| Q15 | 写回闸门 | 自动写回 + 末尾披露 + git 可回滚 + 低认知状态必须标注 |
| Q16 | 文档规则调和 | 分治：建设期走全局文档规则，日常问答不写 Task 文档 |
| Q17 | skill 落位 | 3 个 skill，装**项目级** `.claude/skills/` |
| Q18 | 掌握等级 | L0-L4，且**掌握等级封顶认知状态** |
| Q19 | 覆盖度地图 | 建 `wiki/coverage.md`，骨架由上游结构生成 |
| Q20 | 主动学习 | 手动 `/dsh-learn` + sync 后自动补学变更；**拒绝后台无监督自学** |
| Q21 | 学习优先级 | Cordis 内核 ≈ 集成表面/协议层 > 高稳定性核心包 > 其余 > experimental/POC；叠加提问频率动态调整 |
| Q22 | 方法层 | 建 `playbooks/`，含失败方法记录 |
| Q23/Q26 | 去项目化 | 八维坐标系；写成「条件→结论→锚点→状态」；剥离项目名/私有路径/业务逻辑 |
| Q24 | 回答格式 | 结论先行 + 出处尾注 + 四种话区分 + 学习增量 |
| Q25 | 外部上下文 | 不假设文件可访问；沉淀最高 `verified_inference` 并标注 |
| Q27 | 悬而未决 | 建 `wiki/open-questions.md`，sync 时自动重试 |
| Q28 | 书写语言 | 正文中文，技术术语/包名/配置键/API 名保留英文原文 |
| Q29 | 克隆策略 | `--filter=blob:none`，不 sparse，含 Cordis |
| Q31 | 冷启动 | 阶段一全域 L1 → 阶段二要害 L2；L3 不在冷启动做 |
| Q32 | 回归集 | 不建独立 evals，复用 `log.md` + 锚点抽查 |
| Q34 | 实测边界 | 预授权：`sandbox/` 内最小复现、只读操作、跑上游测试；**不预授权**：装全局软件、写项目外文件、常驻后台、对外写操作 |
| Q35 | 凭据与费用 | 不需凭据的实测直接做；**需要真实模型调用的必须停下说明**（花钱） |
| Q36 | 非人类提问者 | 记 `asked_by`；agent 会话**不得推翻已有 fact 级条目**；健康检查优先复核 agent 批次 |

## 六、ToDoList

- [x] 1. 需求盘问（5 轮，frontier 清空）
- [x] 2. 克隆上游 DSH + Cordis（blobless）
- [x] 3. `README.md`
- [x] 4. `docs/Task_history.md`
- [x] 5. `docs/Task-detail/dsh-expert-bootstrap.md`（本文件）
- [ ] 6. `git init` + `.gitignore` + 目录骨架
- [ ] 7. `CLAUDE.md`（12 条硬约束）
- [ ] 8. skill：`dsh`
- [ ] 9. skill：`dsh-sync`
- [ ] 10. skill：`dsh-wiki`
- [ ] 11. `wiki/` 五个治理文件初版
- [ ] 12. `playbooks/` 四本剧本初版
- [ ] 13. `docs/Module/` 四份模块文档
- [ ] 14. 冷启动阶段一：全域 L1，生成 `index.md` + `coverage.md`
- [ ] 15. 冷启动阶段二：要害领域 L2
- [ ] 16. 向用户汇报 `coverage.md` 真实结果

## 七、上游镜像基线

| 仓库 | HEAD | 提交时间 | 占用 |
| --- | --- | --- | --- |
| deepseek-ai/deepseek-harness | `141eb6fef83422698aef7a981029e843e8161534` | 2026-08-19T23:11:50+08:00 | 110M |
| cordiverse/cordis | `8cc9e33fab69e2d0476d126baaf2acb24e6a6ab4` | 2026-08-13T21:48:18+08:00 | 1.5M |

本机工具链：git 2.50.1 / node v22.22.3 / pnpm 10.13.1 / python 3.12.8 / uv 0.7.20。
（上游要求 node `^22.19.0 || >=24.0.0` ✓；packageManager 为 `pnpm@11.7.0`，本机 10.13.1 不匹配，将来实测需 corepack 切换。）

## 八、执行记录

见第九节"变动文件清单"与 `wiki/log.md`。

## 九、变动文件清单

（任务完成后填写）
