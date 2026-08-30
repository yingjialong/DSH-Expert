# 覆盖度地图（coverage.md）

> **对内是学习者**的仪表盘。它既是能力体检表，也是学习待办清单。

> 想知道「我现在到底懂多少」——看这张表，而不是听我自称。

> 上游基线：DSH `0a53fb55`（仓库tag / npm根包`alpha`为`0.1.2-alpha.2`；根包`latest` / `next`仍为`0.1.1-rc.2`）· 2026-08-30同步。alpha.1→alpha.2跨234提交/1604文件；当前245个非private tag package标识均有alpha.2 tarball，真实install/native闭包未验证。144个既有锚点命中；通用页此前已stale，两篇动态alpha.1页新增stale。


## 掌握等级定义

| 等级 | 含义 | 可给出的最高认知状态 |
|---|---|---|
| **L0** | 未接触 | — |
| **L1** | 读过 README 与文档，知道它是干什么的、在哪 | `verified_inference` |
| **L2** | 读过源码关键路径，能说清接口、扩展点、数据流 | `verified_inference` |
| **L3** | 实测验证过，跑过、见过真实输出/报错 | `fact` |
| **L4** | 有踩坑记录，知道边界、失败模式、与文档不符之处 | `fact` |

**封顶规则**：掌握等级决定我能给出的最高认知状态。L1/L2 的东西**不得标 `fact`**。


## 一、集成表面（最高优先级 —— 通用性杠杆）

| 单元 | 掌握度 | 锚点 | wiki 页 | 备注 |
|---|---|---|---|---|
| 集成表面全景 | **L1** | 29 | [integration/integration-surfaces.md](integration/integration-surfaces.md) | 9 条关键发现 |
| Python SDK | **L2** | 31 | [integration/python-sdk.md](integration/python-sdk.md) | 8 条关键发现 |
| SDK JSON-RPC 线协议（stdio） | **L2** | 17 | [integration/protocol-jsonrpc.md](integration/protocol-jsonrpc.md) | 8 条关键发现 |
| ACP 与 HTTP API gateway（进程外集成的两条协议路线） | **L2** | 28 | [integration/protocol-acp-http.md](integration/protocol-acp-http.md) | 8 条关键发现 |
| Electron 与嵌入运行时集成约束 | **L2** | 57 | [integration/electron-embedding.md](integration/electron-embedding.md) | 2026-08-27：carrier / Workspace / Markdown / pi-ai / reasoning effort / Approval owner 边界 |
| rc.2完整Web Profile与原生壳插件边界 | **L2** | 28 | [integration/rc2-full-web-native-shell.md](integration/rc2-full-web-native-shell.md) | 正式tarball、custom Profile、dual-face Client、platform capability owner与alpha断点 |
| alpha.1完整Host与嵌入控制面 | **L2** | 20 | [integration/alpha1-full-host-embedding.md](integration/alpha1-full-host-embedding.md) | `[stale]`：GitHub/npm分叉、Remote/controllers、owner矩阵、public plugin与安全边界 |

## 二、主题与内核

| 单元 | 掌握度 | 锚点 | wiki 页 | 备注 |
|---|---|---|---|---|
| docs 文档路由地图 | **L1** | 21 | [topics/docs-map.md](topics/docs-map.md) | 7 条关键发现 |
| DSH 整体骨架（跨 5 篇综合） | **L2** | 13 | [topics/architecture-overview.md](topics/architecture-overview.md) | 8 条关键发现 |
| Cordis 内核（DSH 的插件/服务/事件模型） | **L2** | 35 | [topics/cordis-primer.md](topics/cordis-primer.md) | 8 条关键发现 |
| 子系统路由地图（docs/subsystems 20 篇） | **L1** | 22 | [topics/subsystems-map.md](topics/subsystems-map.md) | 8 条关键发现 |
| 四篇事故复盘的提炼（DSH postmortems） | **L2** | 13 | [topics/postmortems.md](topics/postmortems.md) | 8 条关键发现 |
| 插件开发全路径（dsh-plugin） | **L2** | 35 | [topics/plugin-development.md](topics/plugin-development.md) | 8 条关键发现 |
| 核心链 core / session / preset / llm（源码级 L2） | **L2** | 27 | [topics/core-chain.md](topics/core-chain.md) | 10 条关键发现 |
| 配置、工具目录与运行模式 | **L2** | 26 | [topics/config-and-tools.md](topics/config-and-tools.md) | 8 条关键发现 |
| alpha.1→alpha.2版本变更 | **L1** | 16 | [topics/版本变更-0.1.2-alpha.1-到-0.1.2-alpha.2.md](topics/版本变更-0.1.2-alpha.1-到-0.1.2-alpha.2.md) | 发布渠道、ignorable、Skill/Session correlation、MCP public seam与依赖owner重组 |

## 三、包组（51 组 / 251 包）

| 包组 | 掌握度 | 稳定性 | 包数 | 锚点 | 扩展点 | 陷阱 | 悬疑 |
|---|---|---|---|---|---|---|---|
| [acp](packages/acp.md) | L1 | Product | 1 | 9 | 5 | 7 | 1 |
| [api](packages/api.md) | L1 | Product | 5 | 11 | 12 | 7 | 1 |
| [attachment](packages/attachment.md) | L1 | Product | 2 | 9 | 10 | 7 | 0 |
| [boot](packages/boot.md) | L1 | Product | 2 | 10 | 14 | 10 | 1 |
| [bundle](packages/bundle.md) | L1 | Product | 6 | 12 | 8 | 9 | 0 |
| [client](packages/client.md) | L1 | Product | 44 | 16 | 14 | 14 | 2 |
| [code-runtime](packages/code-runtime.md) | L1 | Product | 3 | 13 | 12 | 10 | 2 |
| [compaction](packages/compaction.md) | L1 | Product | 4 | 13 | 9 | 8 | 2 |
| [context](packages/context.md) | L1 | Product | 6 | 12 | 7 | 8 | 2 |
| [core](packages/core.md) | L1 | Product | 8 | 20 | 12 | 10 | 2 |
| [credentials](packages/credentials.md) | L2 | Product | 3 | 15 | 6 | 8 | 1 |
| [e2b](packages/e2b.md) | L1 | POC | 3 | 10 | 6 | 9 | 2 |
| [examples](packages/examples.md) | L1 | Support | 1 | 11 | 6 | 8 | 1 |
| [experimental](packages/experimental.md) | L1 | Unreleased | 8 | 13 | 6 | 10 | 2 |
| [extensions](packages/extensions.md) | L1 | Product | 4 | 17 | 8 | 8 | 2 |
| [feedback](packages/feedback.md) | L1 | Product | 2 | 9 | 7 | 9 | 2 |
| [fs](packages/fs.md) | L1 | Product | 7 | 19 | 12 | 11 | 2 |
| [goal](packages/goal.md) | L1 | Product | 4 | 13 | 8 | 10 | 2 |
| [guard](packages/guard.md) | L1 | Product | 2 | 9 | 8 | 10 | 2 |
| [hooks](packages/hooks.md) | L1 | Product | 3 | 13 | 17 | 13 | 3 |
| [host](packages/host.md) | L2 | Product | 7 | 29 | 16 | 15 | 3 |
| [identity](packages/identity.md) | L1 | Product | 1 | 5 | 4 | 4 | 2 |
| [interaction](packages/interaction.md) | L2 | Product | 5 | 18 | 9 | 10 | 2 |
| [jobs](packages/jobs.md) | L1 | Product | 3 | 10 | 7 | 10 | 2 |
| [llm](packages/llm.md) | L2 | Product | 7 | 35 | 10 | 14 | 3 |
| [lsp](packages/lsp.md) | L1 | Product | 3 | 14 | 6 | 12 | 2 |
| [mcp](packages/mcp.md) | L1 | README 未列出 | 1 | 7 | 7 | 10 | 3 |
| [plan](packages/plan.md) | L1 | Product | 1 | 7 | 7 | 11 | 2 |
| [preset](packages/preset.md) | L1 | Product | 2 | 14 | 11 | 8 | 1 |
| [runtime-diagnostics](packages/runtime-diagnostics.md) | L1 | README 未列出 | 1 | 10 | 10 | 9 | 1 |
| [sandbox](packages/sandbox.md) | L1 | Product | 4 | 14 | 24 | 10 | 1 |
| [schedule](packages/schedule.md) | L1 | Product | 1 | 13 | 11 | 10 | 0 |
| [sdk](packages/sdk.md) | L1 | Product | 3 | 16 | 23 | 10 | 1 |
| [session](packages/session.md) | L2 | Product | 14 | 32 | 24 | 13 | 2 |
| [session-query](packages/session-query.md) | L1 | Product | 4 | 16 | 18 | 13 | 1 |
| [settings](packages/settings.md) | L2 | Product | 2 | 9 | 12 | 7 | 1 |
| [shell](packages/shell.md) | L1 | Product | 10 | 13 | 10 | 9 | 1 |
| [skill](packages/skill.md) | L1 | Product | 4 | 7 | 11 | 8 | 1 |
| [spill](packages/spill.md) | L1 | Product | 3 | 9 | 9 | 9 | 0 |
| [storage](packages/storage.md) | L1 | Product | 4 | 10 | 9 | 9 | 1 |
| [subagent](packages/subagent.md) | L1 | Product | 11 | 12 | 14 | 13 | 2 |
| [subprocess](packages/subprocess.md) | L1 | Product | 3 | 10 | 9 | 10 | 1 |
| [terminal](packages/terminal.md) | L1 | Product | 3 | 9 | 6 | 6 | 2 |
| [test-support](packages/test-support.md) | L1 | Support | 6 | 10 | 8 | 4 | 2 |
| [todo](packages/todo.md) | L1 | Product | 1 | 6 | 5 | 5 | 1 |
| [typert](packages/typert.md) | L1 | Product | 4 | 7 | 10 | 5 | 2 |
| [util](packages/util.md) | L1 | Support | 12 | 11 | 7 | 5 | 1 |
| [web](packages/web.md) | L1 | Product | 6 | 12 | 7 | 6 | 2 |
| [workflow](packages/workflow.md) | L1 | Product | 4 | 10 | 8 | 7 | 2 |
| [webhook](packages/webhook.md) | L1 | Product | 2 | 10 | 8 | 8 | 2 |
| [workspace](packages/workspace.md) | L1 | Product | 1 | 8 | 7 | 8 | 2 |

## 四、已知盲区（L0 —— 诚实清单）

> 这些单元**尚未接触**。被问到时我会现查，但不要指望我有沉淀。


| 盲区 | 说明 | 优先级 |
|---|---|---|
| **全域 L3（实测）** | 冷启动明确不做实测运行，因此**当前没有任何 `fact` 级知识**。所有结论最高 `verified_inference` | 按需（真实提问触发） |
| `apps/cli` | CLI 应用自身的 flag 家族、source-launch 钩子（只到 boot 库边界） | 中 |
| `apps/web` | Web 应用前端本体（`packages/client` 已 L1，但 app 层未读） | 中 |
| `native/` | 原生组件，完全未接触 | 低 |
| `vendor/` | vendored 依赖，完全未接触 | 低 |
| `scripts/` | 构建与生成脚本（`gen-module-graph` 等只知其存在） | 中 |
| `website/` | 官网源码，对答题价值低 | 低 |
| `docs/i18n/` | 翻译流程与术语表（5 篇） | 低 |
| `docs/rescope.md`、`graph-atlas.md`、`persistence-catalog.md`、`web-styling.md` | 已在 docs-map 中登记路由，但未精读 | 中 |
| 第三方插件生态 | `dsh-plugin` topic、awesome 列表：只做索引与可靠性标注，不背书质量 | 低 |
| 与其他 harness 横向对比 | 按约定只在明确要求时做，且必须标注为主观判断 | 按需 |

## 五、汇总

| 指标 | 值 |
|---|---|
| 已建立页面 | 75 页（69 内容页 + 6 治理页） |
| 掌握度分布 | L2 **35** · L1 **34** · L3/L4 **0** |
| 锚点总数 | 887（69个内容页frontmatter锚点总数；正文重复行内引用不计） |
| 已记录陷阱 | 572 条（分布在各页「陷阱」一节） |
| 文档与源码冲突 | 79 条（见 conflicts.md） |
| 悬而未决 | 110 条（见 open-questions.md） |
| 认知状态 | 全部 `verified_inference`（无 `fact`，因无 L3） |
| 新鲜度 | `stale` 65页 · `fresh` 4页（identity、webhook、固定rc.2 Host页、alpha.2版本页）；固定版本问题回目标tag核验 |

## 六、下一步学习建议（按 `dsh-wiki` B3 优先级）

1. **实测把要害领域推到 L3** —— 当前全域无 `fact` 级知识，这是最大的能力缺口
2. 精读 `docs/persistence-catalog.md` 与 `graph-atlas.md`（多个包组的悬疑都指向这两篇）
3. `apps/cli` 的 flag 家族（排错类问题的高频入口）
4. `packages/client` 的 29 个 `ui-*` 插件逐包细读（Web GUI 问题的必经之路）
