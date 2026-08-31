# 问答与学习日志

> 按时间倒序记录每一次问答、学习、实测。这份日志同时充当**回归检测的替代品**：`/dsh-sync` 时对"锚点被本次上游变更命中"的历史问答做抽查复验（见 `dsh-sync` skill 步骤 6）。

## 条目格式

```
## YYYY-MM-DD · <一句话主题>
- **提问者**：human | agent
- **问题**：<原问题，去项目化>
- **结论**：<一句话>
- **依据锚点**：<文件路径列表>
- **上游基线**：<短SHA>
- **沉淀**：<新增/更新的 wiki 页 + status>
- **实测**：<命令与结果，或"无">
- **覆盖度变化**：<领域 Lx→Ly，或"无">
```

---

## 2026-08-24 · 教学批次开始

- **提问者**：human（请求系统学习 DSH，多日课程）
- **结论**：课程素材全部命中既有 fresh 页面与 `b150a55` 上游文档，无新沉淀；后续实操若执行，实测结果按第 11 条写回本 log

## 2026-08-24 · agent 审核：自主运维 Spec 契约核验批次的沉淀

- **提问者**：agent（对外部项目新 Spec 的若干条 DSH 契约主张做源码核验；项目细节不入库）
- **问题**：Question same-turn continuation、pre-execute 异步性、inbox 撤回、steer 目标、code-mode 形态
- **结论**：10 confirmed / 2 partial / 1 措辞修正 / 1 refuted（"pre-execute 不能等待人"是部署惯例非官方契约）；steer 目标是 next-step 非 next-turn；maxParallelToolCalls（agent-loop）与 allowParallelInProgress（todo，必填）各归其包
- **依据锚点**：`packages/core/tools/src/index.ts`、`packages/core/agent-loop/src/{index,tool-calls}.ts`、`packages/core/agent/src/{inbox,runtime-types}.ts`、`packages/host/apiproxy/src/{api/events.ts,api-proxy.ts}`、`packages/todo/tool-todo/src/index.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2）
- **沉淀**：`packages/interaction.md` 追加「2026-08-24 agent 复审增量」节（7 条）
- **实测**：无
- **覆盖度变化**：无

## 2026-08-23 · agent 审核：Ticket 12A–20 代码审核批次的沉淀

- **提问者**：agent（对被审项目某实施链的工单做对抗式只读审核；项目细节不入库）
- **问题**：审核中核验的 DSH 侧事实中哪些可沉淀
- **结论**：tool-bash 自动注入 dshEnv 并向模型硬承诺 $DSH_* 变量——自定义 ShellExecutor 宿主丢弃该字段即造成"承诺不存在的行为"；与官方 bash -c/相对 workdir 语义的漂移应显式声明
- **依据锚点**：`packages/shell/tool-bash/src/index.ts`（execute 的 dshEnv 注入与 schema 描述）、`packages/shell/shell/src/types.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2）
- **沉淀**：`packages/shell.md` 追加「2026-08-23 agent 复审增量」节
- **实测**：无（纯源码核验）
- **覆盖度变化**：无

## 2026-08-23 · agent 复审：修复复审批次的沉淀与一次自我纠错

- **提问者**：agent（对被审项目的修复做二次复审；项目细节按第 7 条不入库）
- **问题**：复审修复时核验的 session.create caller-identity 语义；另确认本库 2026-08-22 批次的 A-1（token 不能做 WeakMap key）系误报
- **结论**：rc.2 `session.create` 允许 caller 自带 `sessionId` 并按 adopt/resume 语义接纳（官方动机为重试去重）；不受信 client 侧宿主应剥离该字段。ES2022 起非注册 symbol 可作 WeakMap key，A-1 作废（E012）
- **依据锚点**：`packages/host/apiproxy/src/sessions/schema.ts`、`packages/host/apiproxy/src/api-proxy.ts`（ensureSession/adopt）、`packages/client/runtime/src/client/workspaces/service.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2）
- **沉淀**：`packages/host.md` 追加「2026-08-23 agent 复审增量」节；`errors.md` E012
- **实测**：`node -e` 验证 non-registered symbol 作 WeakMap key 成功 / `Symbol.for` 抛 TypeError（Node v22.22.3）
- **覆盖度变化**：无

## 2026-08-22 · agent 审核：外部 DSH 集成方案独立审核的沉淀批次

- **提问者**：agent（另一 session 的 LLM 审核者；按第 7 条去项目化纪律，被审项目细节不入库）
- **问题**：独立审核一份外部项目的 DSH 集成方案及其实施票时，对其中全部上游事实断言逐条源码核验，并顺带产出 45+ 条新发现——哪些值得沉淀
- **结论**：断言裁决 78 confirmed / 14 partial / 0 refuted（partial 全部为措辞/归属/版本精度出入，无方向性错误）。沉淀聚焦六个最高价值簇：嵌入运行时约束（新页）、subagent delegation never 精确语义、approval ask 触发链、PersistenceBackend 真实形状、workspace/fork 继承面、MCP transport 注入面现状
- **依据锚点**：`packages/subagent/subagent/src/child-agent.ts`、`packages/interaction/user-approval/src/index.ts`、`packages/core/tools/src/index.ts`、`packages/session/session-persistence/src/{index,coordinator}.ts`、`packages/workspace/workspace/src/{index,entity}.ts`、`packages/mcp/mcp-client/src/{transport,connection}.ts`、`packages/client/connection/src/client/index.ts`、`packages/boot/app-boot/src/index.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2，开审前已复验 HEAD 与远端一致）
- **沉淀**：`integration/electron-embedding.md`（新，L2）；`packages/{subagent,interaction,session,workspace,mcp}.md` 各追加「2026-08-22 agent 审核增量」节（只增不删，未触碰任何已有结论）；index/coverage 同步
- **实测**：无（纯源码核验；按第 11 条"静态可定则不测"，本轮全部问题源码可判定）
- **覆盖度变化**：新单元「Electron 与嵌入运行时集成约束」L0→L2；subagent/interaction/session/workspace/mcp 页内知识扩充，等级不变
- **未沉淀但已记录在审核报告的发现**（后续 /dsh-learn 候选）：session-query predicate budget（跨会话 14 / 会话内 13 谓词上限）、FTS 字面短语语义、attachment-local 批量 saveImages 全有或全无、spill 文件 0600/不清理累积、tool-fs-search `--no-config` 防配置注入、jobs 第五态 `stopping`、llm retry policy registration-captured、dist-tag 分裂的上游发布策略根因、cordis_inspect events 目录为构建期生成物

## 2026-08-22 · /dsh-sync：rc.8 → 0.1.1-rc.2 全量同步与知识防腐

- **提问者**：human
- **问题**：DSH 已有新版本，更新本地镜像与知识库
- **结论**：同步 `141eb6f` → `b150a55`（0.1.1-rc.2，207 提交 / 2416 文件；Cordis 零变更）。47 页锚点命中全部复验：32 确认 / 9 增补 / 4 页结论被推翻重写。npm `latest` 从 rc.7 变 **0.1.1-rc.2**，默认回答基线已切换。
- **依据锚点**：`packages/credentials/credentials/src/types.ts`、`packages/session/session-projection/src/index.ts`、`packages/attachment/attachment/src/index.ts`、`packages/llm/llm/src/index.ts`、`packages/host/apiproxy/src/api-proxy.ts`、`packages/host/apiproxy/src/index.ts`、`packages/host/webserver/src/index.ts`、`packages/acp/acp/src/content.ts`
- **上游基线**：DSH `b150a55`（2026-08-21T20:03+08:00，dsh-v0.1.1-rc.2）· Cordis `8cc9e33`（未变）
- **沉淀**：`errors.md` E008–E011（credentials 事件拆分 / projection register 重构 / 图片两级限额与 normalized 持久化 / Web 图片守卫移位）；`topics/版本变更-0.1.0-rc.8-到-0.1.1-rc.2.md`（新）；credentials/session/attachment/llm/host/plan/todo/core-chain/protocol-acp-http 等页重写或增补；`open-questions.md` 解答删除 5 条（Q014/Q034/Q042/Q106/Q114）
- **实测**：`git ls-remote` 比对 HEAD 落后 → fetch/pull --ff-only 成功；`curl registry.npmjs.org/@deepseek-ai%2Fdsh` → dist-tags `{latest: 0.1.1-rc.2, next: 0.1.1-rc.2}`（发布时间 0.1.1-rc.1 06:49 / rc.2 12:42，均 2026-08-21）；`curl pypi.org/pypi/deepseek-harness-sdk/json` → `0.1.1rc1`；`gh api repos/deepseek-ai/deepseek-harness/releases` → 4 个 release body 完整抓取（rc.1 含 Bubblewrap `/proc/<pid>/root` 逃逸安全修复）
- **覆盖度变化**：无等级变化（同步复验不升级 L）；credentials 与 attachment 页内容大幅扩写（仍在 L2）

---

## 2026-08-20 · 知识库建立（bootstrap）

- **提问者**：human
- **问题**：建立一个能持续自我更新的 DSH 专家系统
- **结论**：完成 5 轮需求盘问，确立双重身份、四把标尺、八维坐标系、防腐三件套（认知状态 / 锚点失效检测 / 掌握等级封顶）
- **依据锚点**：`packages/README.md`、`docs/module-graph.md`、`python/README.md`、`package.json`
- **上游基线**：DSH `141eb6f`（2026-08-19T23:11+08:00）· Cordis `8cc9e33`
- **沉淀**：`errors.md` E001-E003（3 条命名/版本陷阱，均为 `fact`）
- **实测**：无（冷启动不做 L3）
- **覆盖度变化**：全域 L0 → 冷启动进行中

**建库时核实的关键数字**（均为 T1 源码直接读出）：

| 项 | 值 | 来源 |
| --- | --- | --- |
| 包组数 | **50**（非文档所述） | `ls packages/` |
| 包总数 | **226** | `ls -d packages/*/*/` |
| master 版本 | `0.1.0-rc.8` | `package.json` |
| node 要求 | `^22.19.0 \|\| >=24.0.0` | `package.json` engines |
| packageManager | `pnpm@11.7.0` | `package.json` |
| rc.7→rc.8 变更规模 | **1604 文件 / +54064 / −10533** | `git diff --shortstat` |
| docs 顶层主题 | 19 | `ls docs/*.md` |
| docs/subsystems | 20 篇 | `ls docs/subsystems/` |
| docs/cookbook | 9 篇 | `ls docs/cookbook/` |
| docs/postmortem | 4 篇事故复盘 | `ls docs/postmortem/` |
| apps | `cli`、`web` | `ls apps/` |

---

## 2026-08-20 · 冷启动阶段一 + 阶段二

- **提问者**：self（主动学习，非提问触发）
- **范围**：阶段一全域 L1（50 个包组 + 19 个 docs 顶层主题 + 20 篇 subsystems + 4 篇 postmortem + Cordis + 集成表面）；阶段二要害 L2（JSON-RPC 协议层、Python SDK、ACP/HTTP、核心链、插件开发、配置与工具）
- **执行方式**：17 个并行 agent，每个只写自己的页面（避免并发写冲突），索引与覆盖度由主控统一汇总
- **上游基线**：DSH `141eb6f` · Cordis `8cc9e33`
- **产出**：62 个内容页 + 6 个治理页；**532 个去重锚点**；572 条陷阱；75 条文档与源码冲突；114 条悬而未决
- **验收（lint 全绿）**：frontmatter 违规 **0** · 封顶规则违规 **0** · 锚点指向不存在文件 **0** · 超 400 行的页 **0**
- **沉淀**：`errors.md` 新增 E004-E006（三条跨组陷阱）；新建 `conflicts.md`（75 条冲突登记册）
- **实测**：无（冷启动明确不做 L3，故当前**全域没有任何 `fact` 级知识**）
- **覆盖度变化**：全域 L0 → **L1 53 个单元 / L2 9 个单元**

**阶段二的高价值发现举例**（均为读源码才能得出、文档未写）：

| 发现 | 出处 |
| --- | --- |
| SDK 线协议只有 3 个 client→server 方法 + 4 个 server→client 通知，**无 interrupt/cancel/abort**，停止跑飞的回合只能 `close()` 杀进程 | `packages/sdk/protocol/src/types.ts` |
| Python SDK 与 runtime 是**精确等号锁**（`==`），不能单独升级其一；只发 wheel 不发 sdist，平台仅 3 个 | `python/sdk/pyproject.toml`、`python/sdk-runtime/platforms.json` |
| `session_prompt()` 只返回排队回执就立即返回，**不等回合结束**；绕开 `Session.run()` 就得自己拥有活动边界 | `python/sdk/src/deepseek_harness/client.py#session_prompt` |
| ACP 的 stdout 被协议帧独占，任何调试打印都会污染 JSON-RPC 帧 | `packages/acp/acp/src/index.ts` |
| ACP 插件加 default export 会让 Loader 丢掉 namespace 使 `inject` 静默失效（0001 号事故主角） | `docs/postmortem/0001-acp-default-export-drops-inject.md` |

## 2026-08-25 · 教学 1.2 课（Cordis 内核）与一条新冲突

- **提问者**：human（教学推进）
- **准备**：workflow `dsh-lesson-1-2-prep`（5 路：wiki cordis-primer + docs/cordis-primer + tutorial 01/02 + upstream/cordis 源码级 ctx-key 探索 + postmortem 0001）
- **本次现查**：`vendor/cordis/src/events.ts:32` 复验 DispatchMode 为 5 种（含 bail），发现官方 primer 只列 4 种 → 新增 conflicts.md C076，cordis-primer.md 补"文档简化差异"一行并更新 verified_at
- **纠正记录**：教学预设中的 `definePlugin` API 不存在——Cordis 插件协议是 named export 的 `apply` 函数 / `{ apply }` object / `Service` 子类三形态（tutorial 01 现查）
- **上游基线**：DSH `b150a55` FRESH · Cordis 镜像 8cc9e33（注意 vendor 是 4.0.0-rc.7 + 18 条 local mods，权威源是 vendor/）

## 2026-08-26 · 教学 1.2 课重讲（零基础版）

- **提问者**：human
- **反馈**：第一版 1.2 课术语密度过高（假设了 Cordis/依赖注入背景），用户要求按"不懂 Cordis 的人"的起点重讲
- **动作**：零基础重讲（问题驱动 → 词汇表白话化 → 贯穿例子 → 机制深水区后置为可选）；无新 DSH 事实核验，纯教学组织
- **上游基线**：DSH `b150a55`（教学连续性内，未再触发同步）

## 2026-08-27 · 跨会话问答：本地嵌入 Host 的 Workspace 与 SSH 远程 cwd

- **提问者**：agent（跨会话）
- **问题**：Electron SSH 宿主集成 DSH 时，官方 Workspace 应绑本地 anchor 还是远程 SSH cwd？
- **结论**：DSH Host 在本地运行时必须绑 Host 可见且存在的本地 anchor；远程 cwd 属于宿主自有映射或成对 SSH `ctx.fs` / `ctx.subprocess` provider 的 execution world。仅当 DSH Host 本身在远端，或远程目录已挂载到 Host 文件系统时，Workspace 才能使用对 Host 可见的该路径。
- **本次现查**：`WorkspaceRegistry.create` 直接调本机 Node `fs.realpath/stat`；`session.create({ workspaceId })` 写入 `workspace.path` 为 `SessionHeader.cwd`；`attachSession` 严格复验；文件与 Bash 工具默认沿用该 cwd。第一方 provider 清单只有 local / sandbox / E2B，全仓未见 SSH provider；`glob` / `grep` 经 `ctx.subprocess` 运行 `rg`，远程集成不能只替换 `ctx.fs`。
- **依据锚点**：`packages/workspace/workspace/src/{paths,index,entity}.ts`、`packages/host/apiproxy/src/api-proxy.ts`、`packages/fs/tool-fs/src/session-cwd.ts`、`packages/shell/tool-bash/src/index.ts`、`packages/fs/README.md`、`packages/subprocess/README.md`、`packages/e2b/README.md`
- **上游基线**：DSH `b150a55`（`0.1.1-rc.2`，本地 HEAD = 远程 HEAD）
- **实测**：无（静态源码已能确定 Workspace path 的 Host-local 约束，按规范不做冗余实测）
- **沉淀**：`integration/electron-embedding.md` §8（`verified_inference`，L2 不变）；`index.md` 摘要已更新。

## 2026-08-27 · 跨会话问答：DSH 最新稳定版、预发布版与 monorepo 版本口径

- **提问者**：agent（跨会话）
- **问题**：截至 2026-08-27 的 DSH 最新稳定版、最新预发布版、HEAD 相对最新 tag 的状态，以及多包 monorepo 应如何表述“版本”。
- **结论**：公开稳定版尚无；最新预发布版为 `0.1.1-rc.2`。远端 `HEAD` / `master` / `dsh-v0.1.1-rc.2` 均为 `b150a55`，ahead/behind 均为 0。推荐表述为“DSH release family `0.1.1-rc.2` + tag + commit”，讨论安装物时再限定具体包和 registry channel。
- **本次现查**：GitHub 4 个公开 Release 全为 prerelease；npm CLI 主包 `latest=next=0.1.1-rc.2`，三个 SDK 包已发布 rc.2 但仅 `next` 指向它；PyPI SDK/runtime-bin 均为 `0.1.1rc1`。`scripts/release/families.ts` 确认 227 个 DSH 可发布成员共享版本，vendor/native 等另走版本线。
- **依据锚点**：`package.json`、`apps/cli/package.json`、`packages/sdk/{client,protocol,server}/package.json`、`scripts/release/{families,bump,publish}.ts`、`scripts/build-python-release.py`、`python/development.zh.md`
- **上游基线**：DSH `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`（`0.1.1-rc.2`，本地 HEAD = 远端 HEAD = 最新 tag）
- **联网验证**：GitHub Releases/Tags/Compare API；npm registry `@deepseek-ai/dsh` 与三个 SDK 包；PyPI `deepseek-harness-sdk` / `deepseek-harness-runtime-bin`。
- **沉淀**：版本变更主题页追加“当前发布状态复验”；删除已解答 Q095；index/coverage 同步。掌握度仍为 L2，认知状态封顶 `verified_inference`。

## 2026-08-27 · 跨会话问答：嵌入宿主复用官方 Assistant Markdown primitive

- **提问者**：agent（跨会话）
- **问题**：只消费 `ConversationSnapshot.chat` 的 Electron 宿主，能否从公共根导出复用 `MarkdownText`；应否优先于 `react-markdown`；逐消息流式状态不精确时能否默认 settled；需保留哪些安全/owner 边界。
- **结论**：复用公共 `MarkdownText` 符合 DSH projection / presentation ownership，且在固定 rc.2 时应优先于第二套 Markdown renderer。官方用 `assistant-step.data.status` 驱动逐消息 streaming，session-wide `running` 不能代替；一一对应已丢失时省略 `streaming` / 传 `false` 是安全保守值，同时应省略 settled-only 的 `fileMentions`。
- **本次现查**：primitives 根 export 与 package manifest；官方 conversation 的同路径消费；`MarkdownText` 默认值、增量/finalize 和 file mention 生命周期；raw HTML、链接、图片与 KaTeX 安全实现；assistant node status 与 session snapshot running 的分层；相关测试断言。
- **依据锚点**：`.agents/notes/implemented/feature/2026-07-23-web-assistant-markdown.md`、`packages/client/ui-primitives/{package.json,src/index.ts,src/markdown/{MarkdownText,render,katex}.tsx,tests/markdown*.client.spec.tsx}`、`packages/client/ui-conversation/src/client/{chat/{AssistantMarkdown,AssistantNodeView}.tsx,contract/chat-nodes.ts}`、`packages/client/runtime/src/client/sessions/conversation.ts`
- **上游基线**：DSH `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`（`0.1.1-rc.2`；本地 HEAD = 远程 HEAD = `dsh-v0.1.1-rc.2`）
- **实测**：未运行；上游镜像无 `node_modules`，静态实现与官方测试断言已足以确定公共导出、安全策略和状态 owner。
- **沉淀**：`integration/electron-embedding.md` §9（`verified_inference`，L2 不变）；`index.md` 摘要与锚点数已更新。

## 2026-08-27 · 跨会话问答：cold Session 列表标题与 projection cache

- **提问者**：agent（跨会话）
- **问题**：自定义 SessionPersistence 的嵌入 Host 如何让所有 cold 历史标题直接进入首个 `session.list` baseline；cache 的发布/API/介质 owner、20-code-point 标题上限及旧日志回填边界是什么？
- **结论**：rc.2 已公开发布 `@deepseek-ai/dsh-session-projection-cache`；cold list 只读其同步 `cachedSnapshot(meta)`，不读日志。Session log 继续归 `SessionPersistence`，cache row 归 `StorageBackend.kv` 上的 `session_projcache` domain；通用 Workspace backend 可复用。旧日志不会自动扫描，但公共 `coldSnapshot(id)` 能以 `readFrom` + registry restore 回填，无需 open Session。80-byte cap 能容纳任意 20 code points，但不等价于最多 20 code points；token cap 不应机械换算。
- **本次现查**：npm rc.2 tarball 与 dist-tags；cache 包公开 `.d.ts`、service inject/config、domain spec、cold restore/write-back；API Proxy cold list 与 client list seeding；storage backend/domain 公共接口；title UTF-8 normalize、LLM token failure；官方 Loader 组合和相关测试源码。
- **文档冲突**：新增 `conflicts.md` C077——client/runtime 与 host/apiproxy README 的 cold-title 描述落后于同 commit 的源码、类型和测试。
- **依据锚点**：`packages/session/session-projection-cache/{package.json,src/{index,spec}.ts,tests/cache.spec.ts}`、`packages/host/apiproxy/src/{api-proxy.ts,api/sessions.ts}`、`packages/client/runtime/src/client/sessions/manager.ts`、`packages/storage/{storage/src/backend.ts,storage-domain/src/index.ts}`、`packages/session/session-title/src/{index,normalize}.ts`、`packages/bundle/web-app/cordis.patch.yml`
- **上游基线**：DSH `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`（`0.1.1-rc.2`；本地 HEAD = 远程 HEAD = tag）
- **实测**：未运行；上游镜像无 `node_modules`。已静态交叉核对实现、公开声明、官方测试断言和 npm 发布 tarball。
- **沉淀**：`packages/session.md` 新增 2026-08-27 审核增量，掌握度 L1→L2；`index.md` / `coverage.md` / `conflicts.md` 同步。

## 2026-08-27 · 跨会话问答：pi-ai 动态多模型 route、凭据与 Session 选择

- **提问者**：agent（跨会话）
- **问题**：嵌入式 Host 在 rc.2 中如何用官方 `llm-pi-ai` 表达 OpenAI Chat / Responses / Anthropic Messages 多配置，并对齐 route/model owner、CredentialProvider、Settings 热更新、Session 选择持久性与安全边界。
- **结论**：一份独立 key/endpoint/protocol/lifecycle 配置对应一条 provider route；三种 protocol 精确为 `openai-completions` / `openai-responses` / `anthropic-messages`。静态 key 走每 operation 解析的 `apiKeyEnv` ref，pi-ai 原生登录/OAuth 才走 `llm-pi-ai/<route>` record。pi-ai topology 可热更新，但自定义 SettingsProvider 必须让外部变更进入 write API 或 `publish(fullDoc)`；只有 `load()` 不够。
- **Session 边界**：`selectModel` 接受 route + model id，是 live Session 的 next-step selection；选择动作本身不写日志，后续模型请求消费后才记 `request/header`。route 删除/改名无 alias 或自动迁移，历史选择会不可路由。
- **安全边界**：`baseURL` 无 DSH 级 SSRF/scheme/host 限制；`headers` 无 secret owner 语义；provider error 原文无脱敏保证且可耐久化；图片必须走 durable attachment 与请求预算管线。
- **本次现查**：`llm-pi-ai` config/provider/auth/adapter/index/discovery/stream 源码、Settings/Credentials service 源码、API Proxy 模型选择与 Agent loop request header 路径、官方 dynamic/model-selection tests；正式 npm JS/d.ts/README 产物。
- **上游基线**：DSH `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`（`dsh-v0.1.1-rc.2`，本地 HEAD = 远程 HEAD = tag）；`@deepseek-ai/dsh-llm-pi-ai@0.1.1-rc.2` tarball shasum `4391158740196fea86b25a5f6575cc75d0a5d289`。
- **实测**：未运行；上游 mirror 无 `node_modules`。已交叉核对 tag 实现、官方测试断言与正式 npm 产物。
- **沉淀**：`integration/electron-embedding.md` §10；`packages/{llm,settings,credentials}.md` 审核增量；`llm/settings/credentials` 掌握度对齐为 L2；Q113 标注 `llm-pi-ai` schema 已解答。

## 2026-08-27 · 跨会话问答：Approval pending 跨 app switch / macOS occlusion 的 owner 边界

- **提问者**：agent（跨会话）
- **问题**：嵌入式 Electron 宿主的信任 approval 回答面，是否应在普通 window blur/hide 或 macOS occlusion 时取消 pending approval？
- **结论**：rc.2 把 withdrawal 归给 request/ToolRuntime `AbortSignal` 与 channel/Agent owner teardown，不归给 presentation visibility。官方 Host 甚至让 pending approval 跨 client disconnect 存活、在 mux reopen 以同一 `rpcId` 重放；因此 owner/authority 不变时跨普通 app switch / occlusion 保留符合 DSH ownership。上游无 blur/hide 必须取消的规范。
- **本次现查**：`ApprovalRequest`/closed outcome 词汇、ToolRuntime `serviceAsk`、Host `pendingApprovals` 注册/重放/撤回、client runtime `PendingWait`、官方 `ApprovalPanel` 与 reconnect/abort/teardown 测试。
- **文档冲突**：新增 `conflicts.md` C078——`host/apiproxy` README 仍称 pending table 只有 question，但同 commit 源码与官方测试已有 approval registry。
- **依据锚点**：`packages/interaction/user-approval/src/index.ts`、`packages/core/tools/src/index.ts`、`packages/host/apiproxy/src/{api-proxy.ts,api/events.ts,api/approvals.ts}`、`packages/host/apiproxy/tests/api-proxy-approval.spec.ts`、`packages/client/ui-conversation/src/client/{skeleton/ApprovalPanel.tsx,contract/slots.ts}`。
- **上游基线**：DSH `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`（`dsh-v0.1.1-rc.2`；本地 HEAD = 远程 HEAD = `master` = tag）。
- **实测**：未运行；上游镜像无 `node_modules`，静态实现、公开类型与官方测试断言已足以确定该 ownership 边界。
- **沉淀**：`packages/interaction.md` 与 `integration/electron-embedding.md` 增补 visibility/owner 边界；`packages/host.md` 纠正 pending registry 描述；`host` / `interaction` 覆盖度对齐为 L2。

## 2026-08-27 · 跨会话问答：pi-ai 手工模型与 `reasoningEffort:'off'`

- **提问者**：agent（跨会话）
- **问题**：手工 `anthropic-messages` 模型未声明 `reasoningEfforts` 时，为何显式 `session.selectModel(... reasoningEffort:'off')` 返回 `model-unavailable`；客户端和 UI 怎样遵守 exact-model capability ownership？
- **结论**：真正手工 model 不公开 `reasoning`，而 `off` 不是核心通用能力；任何显式未公布 effort 都先被 LLM core 以 `UNSUPPORTED_REASONING_EFFORT` 拒绝，再由 Host 映射成 `model-unavailable`。纯模型切换省略 effort；只有 exact model metadata 明确包含用户选择时才提交。省略把默认所有权交还 adapter/provider，并清除旧模型继承值。
- **UI 边界**：无 metadata 时不伪造 `Off`，显示默认或隐藏控件；条目缺席只能说能力未知。显式偏好不兼容时须明示未应用或阻止发送，不能静默映射；Host 拒绝后保留原 selection。
- **本次现查**：pi-ai catalog/adapter、LLM exact-model capability gate、Host model RPC、Agent selection snapshot/旧 effort 清除、一方 model selector 与相关官方测试；正式 npm 公共声明。
- **上游基线**：DSH `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`（`dsh-v0.1.1-rc.2`，本地 HEAD = 远程 `HEAD` = `master` = tag）。
- **实测**：未运行；上游镜像无 `node_modules`，且该结论由实现分支和官方测试源码交叉闭环，不调用任何模型凭据。
- **沉淀**：`packages/llm.md` 与 `integration/electron-embedding.md`（`verified_inference`，L2 不变）；index / coverage 同步。

## 2026-08-28 · 跨会话问答：pi-ai wire protocol 与展示标签分层

- **提问者**：agent（跨会话）
- **问题**：把产品界面的 “OpenAI Chat” 改为 “OpenAI Completions”，但保持产品内部枚举、持久化值及 DSH `api` 映射不变，是否会改变实际 wire protocol？
- **结论**：不会。rc.2 以精确 `api: 'openai-completions'` 选择 `openAICompletionsApi`；展示名称与 `api` 在配置、建模和 catalog 中相互独立。只要界面文案不被复用为 option value、序列化 key 或映射输入，改名仅属于 presentation。
- **本次现查**：`llm-pi-ai` 的协议表、provider schema、provider materialization、catalog 与 SDK options 官方测试。
- **依据锚点**：`packages/llm/llm-pi-ai/src/{provider,config,adapter}.ts`、`packages/llm/llm-pi-ai/tests/{catalog,sdk-options}.spec.ts`
- **上游基线**：DSH `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`（`dsh-v0.1.1-rc.2`；package `@deepseek-ai/dsh-llm-pi-ai@0.1.1-rc.2`）
- **实测**：未运行；实现、公开 schema 与官方测试已足以确定字段分层和协议分派。
- **沉淀**：命中 `packages/llm.md` 既有审计结论，无重复新增主题内容。

## 2026-08-28 · /dsh-sync：0.1.1-rc.2 → 0.1.2-alpha.1 知识防腐

- **提问者**：self（DSH 问答新鲜度自检触发）
- **结论**：上游镜像从 `b150a551` 同步到 `cd5ef814`（`dsh-v0.1.2-alpha.1`），1079 个提交、6421 个文件（+323745 / −126827）；Cordis 仍为 `8cc9e33`。npm `@deepseek-ai/dsh` 的 `latest` / `next` 仍为 `0.1.1-rc.2`，PyPI SDK 仍为 `0.1.1rc1`。
- **破坏性主线**：`host-apiproxy` 与 `client-runtime` 删除，Session/Workspace/Settings controller 与 Client Store 接管 owner，Gateway 增加 generation-aware stream，`CallId` 重命名 `ToolCallId`，Code Mode 产品词汇迁到 PTC。
- **防腐动作**：63 个内容页锚点命中，统一标为 `stale`；固定 rc.2 问题必须回 tag `b150a551` 核验，不用 alpha HEAD 替代已发布公共面。
- **沉淀**：新建 `topics/版本变更-0.1.1-rc.2-到-0.1.2-alpha.1.md`；`index.md` / `coverage.md` 更新基线、新鲜度和覆盖数。
- **未完成复验**：本次差异规模超过单轮可靠重写范围；63 页保持 stale，未声称已对 alpha.1 全量补学。

## 2026-08-28 · 跨会话问答：Session archive、MCP carrier 与 reasoning default 的 rc.2 边界

- **提问者**：agent（跨会话）
- **问题**：rc.2 是否已具备 Session unarchive/archive-remove/delete 闭环；MCP 是“没有一站式包”还是“有包但缺可注入 carrier/tool-generation metadata”；DSH 是否通用默认 `high`。
- **结论**：rc.2 只有单向 `WorkspaceRegistry.archiveSession` / `archivedSessionIds`，无反向 lifecycle 且 `SessionPersistence` 无 delete；alpha.1 仍明确单向。rc.2 已有一站式 `@deepseek-ai/dsh-mcp-client`，但根导出只允许内建 stdio/Streamable HTTP，内部 transport/connection/tool-generation 不是 npm 可组合注入面。DSH core 无通用 `high`；reasoning 完全由 exact adapter/model metadata 与可选 `defaultEffort` 持有，省略则保留 adapter/provider default。
- **依据锚点**：rc.2 `packages/workspace/workspace/src/{index,spec,types}.ts`、`packages/session/session-persistence/src/index.ts`、`packages/mcp/mcp-client/{package.json,src/{index,transport,connection,tools}.ts,tests/*.spec.ts}`、`packages/llm/llm/src/{index,types}.ts`、`packages/host/apiproxy/src/api/sessions.ts`。alpha.1 复核 `packages/workspace/workspace/README.md`、`packages/client/ui-workspace/README.md` 与 MCP 包导出。
- **实测**：无；公共类型、实现分支、package export map 与官方测试已能静态闭环，不消耗任何模型凭据。
- **沉淀**：本次要点写入同步日志；相关 package 页因 alpha.1 大规模变更仍保持 stale，等专项复验时再重写。

## 2026-08-30 · 跨会话问答：OpenAI Responses 的 Off 与省略默认语义

- **提问者**：agent（跨会话）
- **问题**：custom `openai-responses` exact model 同时发布 `off: none` 与多个非 Off effort 时，显式选择、request/route 双省略及 route 默认分别形成什么 wire。
- **结论**：显式 Off 发 `reasoning.effort: none`；显式非 Off 发其 map 值并带 `summary: auto`；request/route 双省略与显式 Off 在 adapter 边界不可区分，普通 custom route 同样发 `none`，不能保留远端 omitted-field default；支持的 route `reasoning` 会成为 exact model `defaultEffort`，并在双省略时形成对应非 Off wire。
- **文档冲突**：README 对「省略 profile reasoning 保留 provider default」与 `off: null`「send nothing」的表述只能解释 common option 层，不能覆盖 Responses 最终 formatter；登记 `errors.md` E013。
- **依据锚点**：DSH rc.2 `packages/llm/llm-pi-ai/src/{catalog,adapter}.ts`、`tests/{catalog,adapter}.spec.ts`、正式 npm `lib/index.js`；pi-ai 0.82.1 正式 npm `dist/{models.js,api/openai-responses.js}`。
- **上游基线**：DSH `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`；pi-ai tag `b4f293684bba718d59cc1157679bcf6157b3a7f5`。
- **实测**：未运行；固定正式 tarball 的静态控制流已经闭合四条路径，未使用凭据或真实 endpoint。
- **沉淀**：`packages/llm.md` 增加协议限定；`errors.md` E013。

## 2026-08-30 · 跨会话问答：动态 AgentPreset、冷恢复与 per-Session Skills

- **提问者**：agent（跨会话）
- **问题**：rc.2 能否运行期注册任意生成 preset，并只靠 Session 中的 id 冷恢复 exact composition；该面能否承载无需宿主另存的 0..N immutable Skills 选择；alpha.1 是否补齐。
- **结论**：没有任意内存 definition 注册/provider；可在已配置 root 下运行期物化目录并被下一次扫描发现。Session 创建 header / blank-switch event 只耐久 id，冷恢复仍需当前 roster 提供 definition。roster 内缺 id 时 Agent resume 不回退 default，但只读 transcript/skill list 可退 global；整个 roster 未装配又是 rosterless Host composition，不能泛化为统一 fail closed。Skill registry 没有 per-Session selected set；preset 只能间接编码，且不持久化 definition。
- **alpha.1**：Git tag `cd5ef814` 改用 Session projection、Typert remote、shipped root，但仍无任意 definition persistence 或 selected-skill state；截至查询 preset/skill npm 版本只到 rc.2。
- **依据锚点**：rc.2 `packages/preset/agent-presets/{package.json,src/{index,discovery,preset,session}.ts,tests/{session,mount}.spec.ts}`、`packages/host/apiproxy/src/{api-proxy.ts,api/{sessions,agent-presets}.ts}`、`packages/api/remotes/src/agent-lookup.ts`、`packages/session/session-persistence-jsonl/{README.md,tests/jsonl.spec.ts}`、`packages/skill/{skill/src/index.ts,tool-skill/README.md}`；正式 npm tarball `.d.ts` / JS。
- **实测**：未运行；正式 tarball、tag 实现与一方测试静态闭环。结论按 L1 上限标 `verified_inference`。
- **沉淀**：`packages/preset.md`、`packages/skill.md`、`errors.md` E014。

## 2026-08-30 · 跨会话问答：pi-ai 凭据、终端错误码与 Session turn-error

- **提问者**：agent（跨会话）
- **问题**：固定 rc.2 + pi-ai 0.82.1 下，命名凭据缺失/非法、reasoning 校验、HTTP 401/403、连接失败与解析失败分别如何归码；`AUTH` / `TRANSPORT` 是否证明已经出网；最终 code 是否进入 Session/UI。
- **结论**：命名 ref 缺失或空字符串在 SDK 前是 `MISSING_CREDENTIAL`，非空但 trim 后为空或 HTTP-header 不安全是 `INVALID_CREDENTIAL`；不支持的 effort 是 `UNSUPPORTED_REASONING_EFFORT`。pi-ai terminal message 中独立 `401/403` 归 `AUTH`，连接/截断关键词归 `TRANSPORT`，其余通常归 `PI_AI_ERROR`；这两个 code 是文本分类而非 I/O provenance。最终未被 recovery listener 恢复的 `LlmError.failure.code` 原样耐久为 `turn/end` 并投影为 `TurnErrorNode.code`。
- **配置/readiness 边界**：`CredentialInfo.configured` 语义上应与当时 `resolve()` 一致，但 route/catalog 注册不读取凭据；实际缺 key 可先发布模型目录，到首个 operation 才失败，不代表 credential ready。
- **依据锚点**：DSH rc.2 `packages/credentials/credentials/src/index.ts`、`packages/llm/llm/src/{index,api-key,adapter-failure}.ts`、`packages/llm/llm-pi-ai/src/{index,adapter,stream}.ts`、`packages/core/agent-loop/src/agent.ts`、`packages/core/session/src/types.ts`、`packages/client/ui-conversation/src/client/conversation-nodes/turn-error.ts` 及一方测试；pi-ai 0.82.1 `dist/api/{openai-completions,openai-responses,lazy}.js`、`dist/utils/error-body.js`。
- **正式发布物**：`@deepseek-ai/dsh-llm-pi-ai@0.1.1-rc.2` shasum `4391158740196fea86b25a5f6575cc75d0a5d289`；`@deepseek-ai/dsh-llm@0.1.1-rc.2` shasum `968b03f5fbfe5d0f054e301405bc0cff44dec8df`；`@deepseek-ai/dsh-credentials@0.1.1-rc.2` shasum `ec27f8a509c867fb04f698cc887e92c3bb0adb87`；`@earendil-works/pi-ai@0.82.1` shasum `02ebdfc2997fd88ca1f51a7b5c01f337a9462f34`。
- **上游基线**：固定 tag `dsh-v0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`；查询时远端 HEAD 为 `cd5ef8148158c3a752a658978873241fdf8e2bbc`，未用于反推 rc.2。
- **实测**：本轮未重跑一方测试、未访问真实 endpoint、未使用真实凭据；以 tag 源码、正式 tarball JS/d.ts 与一方测试源码交叉核验。
- **沉淀**：`packages/{llm,credentials}.md`、`errors.md` E015、`index.md`；掌握度均保持 L2，因 alpha.1 大改继续标 stale。

## 2026-08-30 · 跨会话问答：per-Session Skill/MCP exact composition 与 rc.2 公共闭包

- **提问者**：agent（3 个重叠跨会话咨询）
- **问题**：rc.2 是否具有任意 0..N Skill/MCP revision 的 Session active-set、immutable definition/digest store、blank draft/finalize/lock、ref-aware GC；AgentPreset create/select/resume/fork 与 MCP carrier/generation 的实际公开边界；SQLite `:memory:` 能否支撑 history-off 生命周期。
- **结论**：Session 只耐久 preset id（header + blank switch event），resume/ordinary fork 按当前文件 roster重新解析，不校验 content digest；Skill catalog只记录模型当时看到的 name/description，不是 active-set或 definition pin。blank preset select 会重绑同一 Agent并提交事件，但没有 Skill/MCP draft/finalize 状态，且 prompt不与 preset-switch queue共享原子 barrier。MCP正式根内建两种 transport，未公开 carrier factory或 server/generation snapshot/diff。`:memory:` 仅随同一 live connection存活，Host generation teardown即使同进程也会关闭并丢失。
- **正式发布物**：`dsh-skill` `f7fe94a978097483f13214789cdcb8cce78c9669`；`dsh-skill-filesystem` `66f3ea10a553fbf33e523d174182e2ced16bf401`；`dsh-tool-skill` `5038f5764495485178b9190f76536701dddd9a33`；`dsh-agent-presets` `4d5c0440022f36412f72dabf1e8347f68ccfee78`；`dsh-mcp-client` `78973daedcec4fd17c18760e9aef539588cab5e2`；`dsh-session-persistence-sqlite` `3acee68a7344245f964d4b44e86fe8fab9620a83`；`dsh-storage-sqlite` `e62a940229e4af24bbf28d13ac6b1b2e0dc71529`。
- **依据锚点**：rc.2 `packages/{skill,preset,mcp}`、`packages/host/apiproxy/src/api-proxy.ts`、`packages/core/{agent,agent-loop,session}`、`packages/session/{session-persistence,session-persistence-sqlite}`、`packages/storage/storage-sqlite`；正式 npm JS/.d.ts/package exports 与一方测试。
- **实测**：仅运行 Node `DatabaseSync(':memory:')` 无凭据最小生命周期探针：同 connection可读，close 后同进程新 connection为空；未运行上游测试、未连外部 MCP、未使用凭据。
- **沉淀**：`packages/{mcp,skill,storage}.md`、`errors.md` E016、`index.md`；领域掌握度不变（mcp L2、skill/preset/storage L1），因 alpha.1 大改继续标 stale。

## 2026-08-30 · 跨会话问答：alpha.1完整Host、Remote控制面与安全边界

- **提问者**：agent（跨会话）
- **问题**：alpha.1如何保留完整产品图、哪些控制通道是完整/子集、各Harness领域owner与profile activation、out-of-tree原生capability、capability-off占位、启动关闭/升级/安全保证。
- **发布结论**：GitHub prerelease/tag是`cd5ef814`且无assets；npm根包与tag内239个非private DSH package name逐一查询均无alpha.1，PyPI也停`0.1.1rc1`。Tag source/manifest不是正式package closure。
- **集成结论**：source-built CLI/profile与Node app-boot可装完整图；Web Remote/controllers + React-free TS Client/Store + `ClientTransportHooks`是完整一方控制面。SDK保持3请求+4通知，ACP虽扩展仍是automation子集。非TS没有完整versioned Remote协议或Electron实现保证。
- **owner/安全结论**：Service/Provider/Consumer、Agent scope、ToolRuntime与approval seams支持out-of-tree物理provider；但out-of-repo required SessionEvent不在持久化白名单。installed/mounted/published三层必须分开。Developer preview会breaking、无general migration/安全审计；sandbox/approval/permission不保证isolation。
- **实测/查询**：无模型/凭据调用；只读GitHub release、npm/PyPI registry、tag源码/manifest/README/一方tests。登记`conflicts.md` C079（telemetry README与composition冲突）。
- **沉淀**：`integration/alpha1-full-host-embedding.md`、alpha版本页、`errors.md` E017/E018、open Q115/Q116、index/coverage。

## 2026-08-30 · 跨会话问答：rc.2 legacy preset、下游event与Skill路径

- **提问者**：agent（跨会话）
- **结论**：legacy无preset Session显式adopt到named preset会conflict；省略时resume/fork按current default产生条件性fallback，read路径fail-soft且不写回旧header。Out-of-tree event仅能以`ignorable:true`越过rc.2 persistence白名单，不足以表达required exact composition；generic ApiProxy无pre-publication resolver注册。Skill filesystem一层发现，Node fallback跟随直属symlink，`ctx.fs`模式交给provider，watch-follow不是sandbox。
- **依据锚点**：rc.2 `packages/{preset/agent-presets,core/session,session/session-persistence,session/session-projection,host/apiproxy,skill/skill-filesystem}`及正式tarball/一方tests。
- **沉淀**：`packages/{preset,session,skill}.md`、`errors.md` E018。

## 2026-08-30 · 跨会话问答：rc.2完整Web Profile与原生壳插件

- **提问者**：agent（跨会话；先成功回传，后执行本条沉淀）
- **问题**：正式rc.2能否用custom Profile保留完整Web图，dual-face Client/public carrier能否承载原生壳，以及平台capability、Session trigger、发布闭包与alpha升级断点的边界。
- **结论**：base+web-app加无破坏性外部层可保留shipped Web图；正式`dsh.client`+`./client`支持out-of-tree双face，Connection公开`createApiClient/fetch/loadBundle`但无WebKit成品或通用Host→Client push。平台物理能力可进Service/Provider/Consumer与ToolRuntime，具体macOS domain未知。根CLI精确版本不等于递归闭包精确锁定。
- **正式发布物**：直接`npm pack`核验CLI/app-boot/base/web-app/client-modules/connection/runtime/api-remotes/apiproxy/tools/agent/approval/questions/credentials/llm等22个rc.2包；CLI integrity `sha512-UP1UIh6q3Gme...`，shasum `1a5112369f1c46b13a6e6f21de8af5e6afd45074`。
- **依据锚点**：`apps/cli/{README.md,reference/README.md,src/{profile-boot,process-shutdown}.ts,tests/built-bin.e2e.ts}`、`packages/boot/app-boot`、`packages/bundle/{base,web-app}`、`packages/client/{modules,connection,runtime}`、`packages/host/apiproxy`、`packages/api/remotes`与core capability roots。
- **实测**：未启动真实Web Host/WKWebView；静态tag、正式tarball JS/.d.ts、npm integrity与一方tests闭环。无模型调用、无真实凭据。
- **沉淀**：`integration/rc2-full-web-native-shell.md`、`errors.md` E019、index/coverage/log。

## 2026-08-30 · 跨会话复验：AgentPreset base + exact Skill/MCP overlay 缺口

- **提问者**：agent（跨会话；完整答案先回传成功）
- **问题**：rc.2能否在现有AgentPreset上叠加per-Session exact Skill/MCP revision refs，并靠公开required event与统一pre-publication hook精确恢复。
- **结论**：ordinary fork只fold recorded preset id并按fork时current roster重新mount，不继承source live generation/definition bytes；required外部event无runtime vocabulary注册；AgentRegistry public setup与cold-resume resolver只是可组合零件，standard create/resume/fork没有统一hook。Skill catalog不是selected-set，MCP generation不公开，故固定版缺exact overlay闭环。
- **正式发布物**：重新核对agent-presets、Session/Persistence、Skill三包与MCP rc.2 npm integrity；另`npm pack`检查7个tarball的root exports、`.d.ts`和无`src`成员。
- **依据锚点**：`packages/{preset/agent-presets,core/agent,core/session,session/session-persistence,skill/skill,skill/skill-filesystem,skill/tool-skill,mcp/mcp-client}`、`packages/host/apiproxy/src/api-proxy.ts`及一方tests。
- **实测**：无；未连外部MCP、未用凭据。固定tag实现、正式tarball和一方测试足以静态闭环。
- **沉淀**：既有`packages/{preset,session,skill,mcp}.md`结论复验成立，仅追加本日志。

## 2026-08-30 · 跨会话复验：blank preset切换、compatibility vocabulary与Approval续跑

- **提问者**：agent（跨会话；完整答案先回传成功）
- **问题**：rc.2 blank select是否跨scope/log原子、是否与首prompt统一linearize；first-party required events在feature-off时如何解码；legacy no-id与Approval same-call continuation边界。
- **结论**：recompose先ensure/rebind，ApiProxy后append，两者无统一事务；select queue不含prompt，prompt accepted到driver append turn/start存在窗口。仓内known event vocabulary不依赖Service activation；legacy no-id在有roster时取resume/fork时current default。Approval成功outcome由Service配对asked/decided，并在同一ToolRuntime execute中继续。
- **依据锚点**：`packages/preset/agent-presets/src/index.ts`、`packages/core/{scope,agent,session,tools}`、`packages/host/apiproxy/src/{api-proxy,session-export}.ts`、`packages/interaction/user-approval/src/index.ts`及一方tests。
- **实测**：无；正式tarballintegrity、固定tag控制流与测试足以判定。append失败竞态未人为构造，按源码标verified_inference。
- **沉淀**：`packages/preset.md`精化；`errors.md` E020；index/log同步。

## 2026-08-30 · 跨会话窄复核：recompose rollback、Approval payload与future decoder假设

- **提问者**：agent（跨会话；完整答案先回传成功）
- **结论**：再次确认rebind成功后append失败无自动parent rollback；successful `allowed-once`返回前matching asked/decided已append，payload只有id/toolName/callId?/reason与outcome，不含tool arguments/body/header/plan；same ToolRuntime execution继续。always-present required decoder + flag-off activation不是rc.2通用runtime contract。
- **依据锚点**：`packages/preset/agent-presets/src/index.ts`、`packages/core/{scope,session,tools}`、`packages/host/apiproxy/src/api-proxy.ts`、`packages/interaction/user-approval/src/index.ts`与一方tests。
- **实测**：无；正式npm integrity、固定tag源码与测试静态闭环。
- **沉淀**：既有Preset E020、Interaction与Session vocabulary结论复验成立，仅追加本日志。

## 2026-08-30 · 跨会话核验：historical standing回绑与Skill selected-only边界

- **提问者**：agent（原source是multi-agent v2子代理，宿主拒绝直接注入；完整答复改发其session metadata声明的父任务并确认成功）
- **问题**：append失败后能否回到old exact generation；Skill scope provider是否可在AgentPreset base上只暴露selected exact definitions；required event feature-off semantic read边界。
- **结论**：historical standing/binding无公共寻址或release，旧/新generation只等whole-tree teardown；Skill provider只增加/同名shadow，无法过滤不同名继承Skill，opaque locator不受DSH digest/immutability验证。仓内known vocabulary与Service activation分离，外部required decoder仍缺正式契约。
- **依据锚点**：`packages/preset/agent-presets/src/index.ts`、`packages/core/{scope,session,agent-loop}`、`packages/skill/{skill,tool-skill}`、`packages/session/session-persistence`及scope-layer一方tests。
- **实测**：无；正式npm integrity、固定tag源码/类型/测试闭环。
- **沉淀**：`packages/{preset,skill}.md`、`errors.md` E020增补/E021、index/log同步。

## 2026-08-30 · /dsh-sync：0.1.2-alpha.1 → 0.1.2-alpha.2知识防腐

- **提问者**：self（DSH问答新鲜度自检触发）
- **结论**：上游镜像从`cd5ef814`同步到`0a53fb55`（`dsh-v0.1.2-alpha.2`），234提交、1604文件、+27862/−14050；Cordis同步到`b912d399`。npm发布在核验中途发生：根包14:10Z新增alpha.2并挂`alpha`，`latest`/`next`仍为rc.2，PyPI SDK仍`0.1.1rc1`。
- **破坏性/兼容信号**：恢复`SessionEvent.ignorable`、统一`RemoteError`、新增schedule UI、plugin inventory/preset切换与connection retry；runtime dependency owner及vendor闭包变化。alpha.1的controller/client-store拓扑仍在，未发现ApiProxy/client-runtime回归。
- **防腐动作**：144个既有锚点命中；通用页此前已stale，两篇含动态HEAD断言的alpha.1页新增stale。新建alpha.2版本页；固定rc.2/alpha.1问题继续回目标tag核验。
- **实测/查询**：`git fetch/pull`、GitHub release、npm/PyPI registry、tag diff/manifests；穷举244个非private tag标识，208个已有alpha.2、36个没有。无模型调用、无凭据。
- **沉淀**：`topics/版本变更-0.1.2-alpha.1-到-0.1.2-alpha.2.md`、`CLAUDE.md`、README、index/coverage/errors/open-questions/log。

## 2026-08-30 · 跨会话核验：Skill catalog完整性、冲突观察与invocation policy

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：rc.2能否用SkillRegistry对base/global与selected overlay做canonical冲突检查并保证到durable append不变；provider失败、shadowed候选和双false invocation的精确语义。
- **结论**：`snapshot()`只有单次`complete`，无公开generation；provider throw被跳过并令snapshot incomplete，`list()`隐藏该状态，`skills/change`无token/barrier。公共目录只见merged winner，不能观察shadowed/collision，也无exclude provider/layer。模型tool和用户gesture都拒绝双false注入，但Registry `get()`仍可读取，且gesture是先get后检查。Skill observation与Session append无共同原子屏障。
- **依据锚点**：rc.2 `packages/skill/skill/src/index.ts`、`packages/skill/tool-skill/src/index.ts`、`packages/skill/{skill,tool-skill}/tests/*.spec.ts`；正式npm根`.d.ts`/JS。
- **上游基线**：固定`dsh-v0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`；远端HEAD已另同步至alpha.2，不用于反推。
- **实测**：未运行；固定tag实现、公开类型、一方测试和正式npm声明已静态闭环。
- **沉淀**：`packages/skill.md`、`errors.md` E022/E023、index/log；掌握度保持L1，因alpha.2变更仍标stale。

## 2026-08-30 · 跨会话核验：旧Skill/MCP配置到rc.2公共面的兼容边界

- **提问者**：agent（source为multi-agent v2子代理；直发被宿主拒绝后，完整答案relay到其父任务并确认成功）
- **问题**：filesystem sources顺序/启停/override/pin/resource policy与MCP descriptor/provenance/trust/allowlist能否原生迁移，是否有无副作用import/validate/export dry-run。
- **结论**：custom roots有顺序和winner，但无per-name override/version pin/allowed-tools；winner-first+Consumer filter不等价first-enabled fallback。Skill只主动读Markdown并给resourceBase，不持有旧受限reader。MCP Config覆盖transport字段和row启停，不覆盖provenance/trust/allowlist/CredentialProvider。包根Config与CLI dump只做shape/composition预检；真实Skill发现会读完整Markdown，真实MCP apply会spawn/connect，无逐项migration seam。
- **依据锚点**：`packages/skill/{skill,skill-filesystem,tool-skill}`、`packages/mcp/mcp-client`、`packages/core/tools`、`packages/interaction/user-approval`、`apps/cli/src/dump-config.ts`及一方tests/正式npm exports。
- **实测**：无；未读取外部Skill正文、未连接MCP、未使用凭据。
- **沉淀**：`packages/{skill,mcp}.md`、`errors.md` E027相关边界、index/log；等级不变。

## 2026-08-30 · 跨会话核验：admission、request/effect identity、Preset引用与Web security seams

- **提问者**：agent（跨会话；完整答案直接回传父任务并确认成功）
- **问题**：create/prompt丢响应、LLM/tool幂等identity、cancel/crash effect、old Session plugin generation与Web/credential/MCP安全seam。
- **结论**：预分配SessionId是create幂等键；prompt rpcId只相关不去重。LlmAdapter无跨retry/crash request id。Tool callId耐久但非Host全局operation id；cancel ack不等待quiescence，crash repair用TOOL_OUTCOME_UNKNOWN显式保留不确定性。Session只存preset id，remove不查引用，historical generation不公开。API+两WS共用非auth trust fence但HTML不在内；CredentialProvider可替换而raw credential RPC仍在；MCP无trusted secret resolver/carrier。
- **依据锚点**：`packages/host/apiproxy`、`packages/client/{runtime,connection}`、`packages/llm/{llm,llm-retry}`、`packages/core/{tools,agent-loop,session}`、`packages/session/session-persistence`、`packages/preset/agent-presets`、`packages/credentials`、`packages/mcp/mcp-client`及一方tests。
- **实测**：无；未调用模型、未执行native effect、未提交secret。
- **沉淀**：`packages/{host,llm,core,preset,credentials,mcp}.md`、`errors.md` E024–E027、index/coverage/log；等级不变。

## 2026-08-30 · 跨会话核验：alpha.2发布闭包、Skill mutation与MCP协议边界

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：alpha.2正式npm闭包是否补齐per-Session exact Skill/MCP refs、Skill observation lease/CAS、caller幂等commit query，以及MCP 2026-07-28与Host carrier/generation seam。
- **结论**：245个非private tag package标识现均有alpha.2 tarball，但未做真实install/native boot。Session skill API只有catalog读取；required event vocabulary无Skill/MCP active-set。Skill snapshot仍只有`complete`，无public generation/lease。prompt新增`requestId`关联但Host不去重且无commit-status query。MCP固定构造stdio/HTTP transport，SDK 1.29.0 latest protocol为2025-11-25，根无carrier或generation identity。
- **依据锚点**：alpha.2正式`dsh-skill`、`dsh-mcp-client`、`dsh-api-session-controller`、`dsh-session` tarball JS/.d.ts；tag的`packages/{core/session,skill/skill,api/session-controller,mcp/mcp-client,acp/acp}`源码与一方tests；SDK 1.29.0正式tarball protocol constants。
- **沉淀**：alpha.2版本页、CLAUDE/README、index/coverage、errors E016/E017、open Q115与本日志。

## 2026-08-31 · 跨会话核验：zero-retry、Approval参数绑定与tool-policy恢复

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：rc.2能否逐route保证同step adapter at-most-once；Approval是否把Session/call/arguments做成不可替换授权对象；tools hooks能否耐久实施run/turn/per-tool/duplicate/rate-limit预算。
- **结论**：normal `maxRetries:0`合法但只关闭`dsh-llm-retry`executor，其他request-error recovery仍可重入同一turn/step。Approval payload不含arguments或grant token，但标准ToolRuntime先freeze同一execution再ask/guard/body。Public guards可做live hard deny，标准log可条件性fold root/Code Mode attempts；无通用run marker、rate-limit taxonomy、stop action或全ToolRuntime持久恢复合同。
- **冲突**：rc.2 `llm-retry/README`称fresh retry turn，T1 loop与test明确同turn/step，登记C080/E028。
- **依据锚点**：`packages/llm/{llm,llm-retry}`、`packages/core/{agent-loop,tools,session}`、`packages/interaction/user-approval`、`packages/guard/repeat-tool-reminder`、`packages/session/session-checkpoint-policy`及一方tests。
- **实测**：无；未调用模型、未执行native effect，固定tag静态证据足以收口。
- **沉淀**：`packages/{llm,interaction,guard}.md`、errors E028、conflicts C080、index/coverage/log。

## 2026-08-31 · 跨会话核验：alpha.2 Skill/Preset/MCP contract与安装解析

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：正式npm alpha.2是否补齐Skill catalog identity/immutable resolver/Session共同提交、blank prompt barrier/recompose rollback，以及MCP modern protocol/carrier/inspector/generation contracts；package family是否齐备。
- **结论**：题列owner-grade contracts均未补齐。Skill仍是`{skills,complete}`+borrowed string definition；Preset仍先rebind后append且select/prompt无共同barrier；MCP仍固定transport、SDK latest 2025-11-25、无raw inspector或public generation identity。245个nonprivate tag ids均有tarball。
- **实测**：macOS 26.4.1 arm64 / Node 22.22.3 / npm 10.9.8；一次性sandbox执行`npm install --ignore-scripts --no-audit --no-fund @deepseek-ai/dsh@0.1.2-alpha.2 @deepseek-ai/dsh-skill@0.1.2-alpha.2 @deepseek-ai/dsh-tool-skill@0.1.2-alpha.2 @deepseek-ai/dsh-agent-presets@0.1.2-alpha.2 @deepseek-ai/dsh-mcp-client@0.1.2-alpha.2 @deepseek-ai/dsh-session@0.1.2-alpha.2`，524 packages；`npm ls --all`退出0，215个唯一DSH包全为alpha.2；sandbox已清理。未运行scripts/native/boot。
- **依据锚点**：alpha.2正式`dsh-skill`、`dsh-agent-presets`、`dsh-api-session-controller`、`dsh-mcp-client`、`dsh-session`、`dsh-tool-skill` tarball；tag的Skill/Preset/Session-controller/MCP实现与tests；SDK 1.29.0 protocol constants。
- **沉淀**：alpha.2版本页、open Q115、index/coverage/log。

## 2026-08-31 · 跨会话核验：MCP subscription、generation与schema barrier

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：rc.2/alpha.2是否实现MCP 2026-07-28 subscriptions/listen与subscription identity，断线/close后的cache invalidation/relist，以及durable generation observation到下一次model schema/call的线性化。
- **结论**：两版都只有legacy`tools/list_changed`。断线默认保留last-good、重连full relist后swap，耗尽预算或plugin dispose才注销；stale handler被忽略。内部Client/disposer generation不公开、不进Session。每step通用`request/header`耐久实际schemas，但不等待private relist、不带MCP generation，也不pin之后的tool resolve/call/result。
- **版本差异**：rc.2→alpha.2 connection/transport主链不变；alpha.2只把`serverName`预留改为per-scope并迁移类型依赖。tag SDK 1.29.0与当前npm 1.30.0都仍以2025-11-25为latest protocol。
- **依据锚点**：两版正式`dsh-mcp-client`tarball、SDK 1.29/1.30 tarball；`mcp-client/src/{connection,tools,transport}.ts`与reconnect/apply tests；`core/{tools,agent-loop,session}`和`session-checkpoint-policy`。
- **实测**：无真实server；正式tarball、固定tag控制流与一方tests足以静态判定。未使用凭据。
- **沉淀**：alpha.2版本页、open Q117、index/coverage/log。

## 2026-08-31 · 跨会话核验：ToolRuntime generation borrow与MCP closure

- **提问者**：agent（multi-agent v2子任务；直发被宿主拒绝，完整答案relay到其父任务并确认成功）
- **问题**：rc.2从schema publication、executor、post/result到durable commit/cancel是否有definition generation borrow/lifetime seam；MCP Client closure能否扩成跨请求/并行/result/drain合同；未来opaque handle是否有架构硬障碍。
- **结论**：无统一handle。ToolRuntime只早期快照finalizer，classification/body/wrapper success/post value replacement在不同阶段按name重查current registry；一方tests明确replacement会影响pending call与后置规范化。MCP closure只对已选definition局部有效，connection dispose不等ToolRuntime in-flight。现有pipeline/token/WeakMap/Agent drain/log/checkpoint提供未来承载点，未发现结构性不可能，但现合同不能拼出borrow。
- **依据锚点**：正式rc.2`dsh-tools`/`dsh-mcp-client`tarball；`core/tools/src/index.ts`、`core/agent-loop/src/{tool-calls,index}.ts`、`mcp/mcp-client/src/{tools,connection}.ts`与replacement/cancel一方tests。
- **实测**：无；固定tag控制流、public `.d.ts`与tests足以判定。未连接MCP、未使用凭据。
- **沉淀**：`packages/{core,mcp}.md`、open Q118、index/coverage/log。

## 2026-08-31 · 跨会话复验：rc.2/alpha.2 ToolRuntime generation borrow仍缺

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：正式rc.2或alpha.2是否已有execution-scoped definition/catalog generation borrow、retire/ref-count/drain，能把schema G到retry、0..N calls/results及G2换代后的资源寿命绑定到同一opaque handle。
- **结论**：两版正式`dsh-tools`根均无generation lease；schema assembly只耐久schema数据，ToolRuntime在classification/body/wrapper normalization/post value replacement分阶段按name解析current registry，只早期快照finalizer。MCP内部Client/disposer generation只负责relist/swap，definition closure只局部保护已选body，connection dispose不等待ToolRuntime in-flight。题述全链保证仍缺上游public contract。
- **版本证据**：rc.2 `b150a551`与alpha.2 `0a53fb55`；两版正式`dsh-tools`/`dsh-mcp-client`tarball integrity、root exports/`.d.ts`、tag控制流与replacement/re-sync一方tests。
- **实测**：无真实MCP server；未使用凭据。正式发布物、固定tag源码与测试足以静态判定；专门的schema-G→G2竞态e2e未找到，跨阶段后果标verified_inference。
- **沉淀**：alpha.2版本页、open Q118、index/coverage/log；既有rc.2 core/mcp结论不改。

## 2026-08-31 · 跨会话核验：static content-addressed Preset、Skill provider与MCP adapter

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：能否靠公开Preset filesystem roster、scope-local Skill provider与`ctx.tools.register`构造content-addressed静态composition，并在不换代时绕开generation-borrow要求。
- **结论**：预物化目录可由标准preset id进入header/cold resume，无需custom required event；definition store/摘要校验/保留GC归外部owner。scope-local冻结Skill provider继续走官方tool-skill/ToolRuntime/Session，static无mutation时不需要catalog CAS。官方cookbook明确支持MCP plugin discover→register；若活动期不replace且失效只fail closed，generation borrow非必要，但standing实例共享、标准ApiProxy无按Session外部resolver、teardown drain与通用invalid-session状态仍是限定。
- **版本证据**：rc.2 `b150a551`正式agent-presets/skill/tool-skill/tools/user-approval/session tarball；tag的Preset discovery/mount/ApiProxy、Skill Registry/Consumer、ToolRuntime/Approval/AgentLoop与一方tests；alpha.2 `0a53fb55`正式根公共面独立对照。
- **实测**：无；未写definition store、未连接MCP peer、未使用凭据。静态public types/实现/tests足以判定；整套组合无一方e2e，组合结论标verified_inference。
- **沉淀**：`packages/{preset,skill,mcp,core}.md`、open Q118、index/coverage/log。

## 2026-08-31 · 跨会话复核：catalog check ordering、Host drain与selected Skill root

- **提问者**：agent（跨会话；首个回传调用因正文代码标记破坏脚本字符串而未执行，去除冲突标记后完整答案回传并确认成功）
- **问题**：content-addressed preset的Host create/resume/fork、每schema/call remote digest check的精确ordering、static ToolDefinition identity、whole-host drain与官方Skill filesystem selected root能否闭环。
- **结论**：Host SessionsApi公开`sessionId?/agentPreset?`而高层Client Runtime create不带preset。schema providers同步，`system-prompt/assemble`与`agent/pre-step`均在schema collection后但model前；tool最后async check应在body第一步。static registry不变时schema/call来自同definition，但只有AgentHandle drain、无Host→standing统一协调器。filesystem provider以`includeDefaultRoots:false/customSkillDirs/watch:false`消费selected root，精确frontmatter键与layer merge仍需遵守。
- **版本证据**：rc.2 `b150a551`正式host-apiproxy/client-runtime/agent-presets/agent/system-prompt/tools/skill-filesystem/tool-skill tarball integrity、root/API `.d.ts`、tag控制流与一方tests。
- **实测**：无；未连接remote catalog、未写preset/Skill文件、未使用凭据。静态类型/实现/tests足以判定；跨进程atomic publish与完整Host shutdown e2e不存在。
- **沉淀**：`packages/{core,mcp,preset,skill}.md`、errors E029、open Q119、index/coverage/log。
