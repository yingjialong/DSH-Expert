# 知识库索引（index.md）

> **回答时先读本文件**，据此挑 3-5 篇全文，**永远不要全量加载知识库**。

> 默认回答基线：`0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`（npm `latest/next`）；GitHub 最新版与本次 master：`0.1.3-alpha.1` / `d347e703908d0406b7a7ef80e3a0e594d86b2215`，npm 根包尚无该版本。npm `alpha=0.1.2-alpha.5`；PyPI SDK/runtime-bin `0.1.2rc1`。Cordis 镜像 `2ceea231802cc23892b4ad10012c55c7dd4982d4`；DSH vendor `4.0.2` 单独按 DSH SHA 核验。同步于 2026-09-05。
>
> 旧版入口：rc.2 固定 `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`；alpha.5 固定 `db6bdc3576c2d4e7c965e8e3ed0c2a731eed87f5`。11 个历史发布 tag 已核对并保留；每次咨询独立固定 SHA，不切换共享工作目录。
>
> 2026-09-05 同步：DSH `49a606bc` → `d347e703`（292 提交、1780 文件），Cordis 前进 2 提交。141 个锚点路径命中 54 页：52 页原已 stale；固定 rc.2 专题保留；alpha.5 摘要复验发布 tag 后纠正 master/release 混用（E044）。新增[最新版本对照](topics/版本变更-0.1.2-alpha.5-到-0.1.3-alpha.1.md)，三个基线并发源码读取及缺失文件负对照通过；原 69 篇 stale 页未冒充已复验。
>
> 2026-09-02 第二次同步：0.1.2-alpha.4 → 0.1.2-alpha.5（44提交、718文件变更）。npm `alpha`=alpha.5，PyPI SDK仍为`0.1.2a3`，npm `latest/next`仍为rc.2。83个既有锚点命中；将alpha.2→alpha.4页标为`stale`，其他命中内容页早已stale，未全量复验。
>
> 2026-09-02 同步：0.1.2-alpha.3 → 0.1.2-alpha.4（297提交、2371文件变更）；从上一次知识基线alpha.2计算则累计414提交。npm `alpha`=alpha.4，PyPI SDK=`0.1.2a3`，npm `latest/next`仍为rc.2。114个既有锚点命中；将仅有3篇命中且原为`fresh`的页面标为`stale`，未完成全量复验。固定rc.2问题继续回tag `b150a551`。
>
> 2026-08-30 同步：0.1.2-alpha.1 → 0.1.2-alpha.2（234提交、1604文件变更）。npm发布在当天分批完成：当前245个非private tag package标识（含根CLI与Web frontend）均有alpha.2 tarball；真实递归install、native helper与第三方依赖闭包仍未验证。144个既有锚点命中；通用页此前已是`[stale]`，另将两篇含动态alpha.1发布断言的页面标stale。固定版本页继续按各自commit/tag使用。
>
> 2026-08-28 同步：0.1.1-rc.2 → 0.1.2-alpha.1（6421 文件变更）。63 个内容页锚点命中，已统一标为 `[stale]`；复验前不得用于新版本结论，固定 rc.2 问题须回 tag `b150a551` 核验。
>
> 2026-08-22 同步：rc.8 → 0.1.1-rc.2（207 提交）。47 页锚点命中、全部复验完毕（32 确认 / 9 更新 / 4 页结论被推翻已重写，见 [errors.md](errors.md) E008–E011 与 [topics/版本变更-0.1.0-rc.8-到-0.1.1-rc.2.md](topics/版本变更-0.1.0-rc.8-到-0.1.1-rc.2.md)）。

> 图例：`status` 全为 `verified_inference`（冷启动只到 L1/L2，按封顶规则不得标 `fact`）；`fresh` 仅对该页声明的版本与 commit 有效，`[stale]` 表示只能作为路由，须回本次目标版本的固定 SHA 复验。


## 治理文件

| 文件 | 用途 | 何时读 |
|---|---|---|
| [errors.md](errors.md) | 错误本：答错的、死路、命名/版本陷阱 | **每次回答前必读** |
| [conflicts.md](conflicts.md) | 文档与源码不符登记册（83 条） | 引用上游文档作结论前 |
| [open-questions.md](open-questions.md) | 悬而未决 | 查不到答案时先看是否已知 |
| [coverage.md](coverage.md) | 覆盖度地图（掌握等级） | 判断自己有多懂、该学什么 |
| [log.md](log.md) | 问答/学习/实测时间线 | 追溯历史结论 |

## 方法层剧本（playbooks/）

| 剧本 | 什么时候用 |
|---|---|
| [../playbooks/新项目集成选型.md](../playbooks/新项目集成选型.md) | 有人问「怎么把 DSH 集成进我的项目」 |
| [../playbooks/报错诊断.md](../playbooks/报错诊断.md) | 粘来一段 DSH 报错 |
| [../playbooks/插件开发.md](../playbooks/插件开发.md) | 要写 dsh-plugin |
| [../playbooks/版本升级评估.md](../playbooks/版本升级评估.md) | 问「升级会炸什么」 |

## 集成表面（integration/）—— 通用集成专家的核心

| 页面 | 一行摘要 | 掌握度 | 锚点 |
|---|---|---|---|
| [integration-surfaces.md](integration/integration-surfaces.md) [stale] | 外部项目接入 DSH 的五条通路（TS SDK / Python SDK / ACP / HTTP /api / CLI-headless）、各自官方支持程度与硬约束，以及非 TS/Python 宿主的选型判断 | L1 | 29 |
| [python-sdk.md](integration/python-sdk.md) [stale] | Python SDK（deepseek-harness-sdk）源码级用法：两层 API、runtime channel 选择链、零配置注入条件、会话树通知路由、测试即规格 | L2 | 31 |
| [protocol-jsonrpc.md](integration/protocol-jsonrpc.md) [stale] | SDK JSON-RPC 线协议源码级契约：分帧规则、握手/readiness 边界、3 请求 + 4 通知的完整清单、会话并发模型、错误码偏差、任意语言宿主的最小实现清单 | L2 | 17 |
| [protocol-acp-http.md](integration/protocol-acp-http.md) [stale] | ACP（automation-only stdio JSON-RPC）与 HTTP API gateway（/api + Typert Remote）两条进程外集成路线的能力边界、鉴权、会话映射与选型对比 | L2 | 28 |
| [electron-embedding.md](integration/electron-embedding.md) [stale] | Electron/嵌入运行时集成约束：carrier、Workspace 本地 anchor、Assistant Markdown、pi-ai 动态 routes、exact-model reasoning effort 与 Approval owner 边界 | L2 | 57 |
| [rc2-full-web-native-shell.md](integration/rc2-full-web-native-shell.md) [stale] | rc.2正式CLI custom Web Profile、dual-face Client plugin、transport/native capability owner、tarball闭包及alpha拓扑断点 | L2 | 28 |
| [alpha1-full-host-embedding.md](integration/alpha1-full-host-embedding.md) [stale] | alpha.1 GitHub/npm分叉、完整Host形态、Web Remote/controller/carrier、owner矩阵、插件/capability-off/lifecycle/compat/security边界 | L2 | 20 |

## 主题（topics/）

| 页面 | 一行摘要 | 掌握度 | 锚点 |
|---|---|---|---|
| [docs-map.md](topics/docs-map.md) [stale] | DSH docs/ 顶层 19 主题 + 7 个子目录的路由表：什么问题该查哪个文件、篇幅规模、generated/curated 归属、问题→文件反向索引 | L1 | 21 |
| [architecture-overview.md](topics/architecture-overview.md) [stale] | 一次用户输入从 agent.followup() 到 tool/result 的完整旅程：21 步逐步标注层/seam/durable-vs-live 事件，工具管线展开，三事件域判据，seam 三角色，11 个易误解术语，7 条源码交叉验证事实 | L2 | 13 |
| [cordis-primer.md](topics/cordis-primer.md) [stale] | Cordis 内核入门：Context/Fiber/Effect、同module多Fiber的Context/scope/cleanup边界、provide sibling与generator composite rollback、五种dispatch mode及DSH接缝。 | L2 | 45 |
| [subsystems-map.md](topics/subsystems-map.md) [stale] | 20 个子系统 → ctx 服务名 → packages 组 → 该查什么；含 spine/optional 判定规则与文档可信度分级 | L1 | 22 |
| [postmortems.md](topics/postmortems.md) [stale] | 0001 export default 吞掉 inject / 0002 !!js 位置错致 fs 工具永久禁用 / 0003 Web agent 验证替身服务器 / 0004 Landlock 提示被误判——现象·根因·修复·可泛化教训 | L2 | 13 |
| [版本变更-0.1.0-rc.8-到-0.1.1-rc.2.md](topics/版本变更-0.1.0-rc.8-到-0.1.1-rc.2.md) [stale] | 当前发布状态（无稳定版、最新预发布 rc.2、HEAD 与 tag 相同、CLI/SDK/PyPI 渠道差异）+ 版本升级影响：5 组破坏性变更、新增能力表、八维坐标与升级建议。 | L2 | 27 |
| [版本变更-0.1.1-rc.2-到-0.1.2-alpha.1.md](topics/版本变更-0.1.1-rc.2-到-0.1.2-alpha.1.md) [stale] | Host API controllers / Client Store 拓扑重组、`host-apiproxy` / `client-runtime` 删除、Remote generation stream 与 npm 仍停 rc.2 的升级边界。 | L2 | 18 |
| [版本变更-0.1.2-alpha.1-到-0.1.2-alpha.2.md](topics/版本变更-0.1.2-alpha.1-到-0.1.2-alpha.2.md) [stale] | alpha.2发布、Skill/Preset/MCP contract、MCP与ToolRuntime generation-borrow缺口、npm resolution实测。 | L1 | 28 |
| [版本变更-0.1.2-alpha.2-到-0.1.2-alpha.4.md](topics/版本变更-0.1.2-alpha.2-到-0.1.2-alpha.4.md) [stale] | alpha.3/alpha.4发布状态、统计、已识别的breaking信号与未复验边界。 | L1 | 6 |
| [版本变更-0.1.2-alpha.4-到-0.1.2-alpha.5.md](topics/版本变更-0.1.2-alpha.4-到-0.1.2-alpha.5.md) | 固定 alpha.5 发布 diff；纠正同期 master 的 SessionHandle 误归属（verified_inference / fresh）。 | L1 | 5 |
| [版本变更-0.1.2-alpha.5-到-0.1.3-alpha.1.md](topics/版本变更-0.1.2-alpha.5-到-0.1.3-alpha.1.md) | 默认 rc.1、最新 alpha 与旧 rc.2 的 SHA 路由；create/handle/format v2 断点及发布边界（verified_inference / fresh）。 | L2 | 13 |
| [plugin-development.md](topics/plugin-development.md) [stale] | 插件开发全路径：从零写 dsh-plugin 的步骤路由、四种插件形态骨架、依赖纪律（Service Definition 而非 Provider）、Config/schema、事件与扩展点选择、skill provider registry、bundle/profile 发布与八条陷阱 | L2 | 35 |
| [core-chain.md](topics/core-chain.md) [stale] | agent loop 函数级调用链、agent-loop 可替换性的三道机制保证、session 三层 derive 与持久化/projection seam、会话并发隔离 12 条「条件→结论」、preset standing mount 与 scope 父链、LlmAdapter 接入点、system-prompt/context 注入时机 | L2 | 27 |
| [rc2-session-binding-create-preset.md](topics/rc2-session-binding-create-preset.md) | 固定rc.2：SessionFace命令不随current改目标；create非落盘回执；无原子abandon但有同id blank preset select（verified_inference / fresh）。 | L2 | 16 |
| [rc2-tool-result-drain-credentials.md](topics/rc2-tool-result-drain-credentials.md) | 固定rc.2：result observer不等待Promise；adapter drain与Agent idle分离；per-operation credential与opaque ref边界（verified_inference / fresh）。 | L2 | 12 |
| [rc2-agent-request-attempt-schema.md](topics/rc2-agent-request-attempt-schema.md) | 固定rc.2：request配置提案与physical retry分层、occurrence身份及非2020-12的schema子集（verified_inference / fresh）。 | L2 | 12 |
| [alpha13-full-host-public-boundaries.md](topics/alpha13-full-host-public-boundaries.md) | 固定1.3-alpha.1对照1.2-rc.1：Host/Client、MCP/Skill、Team、attempt、handle及Web认证；发布状态需复查（verified_inference / fresh）。 | L2 | 36 |
| [config-and-tools.md](topics/config-and-tools.md) [stale] | 配置与工具目录 — 非机密值六层覆盖梯子 + 凭据四层梯子 + 四种运行模式的真实定义（Creator mode 的目录 id 是 cordis）+ dsh --profile/bundle 补丁层 + tool-catalog 路由 + permission preset/approval。排错类问题的第一入口。 | L2 | 26 |

## 包组（packages/）—— 既有 51 组历史路由，包数按页内版本理解

| 包组 | 一行摘要 | 稳定性 | 包数 | 锚点 |
|---|---|---|---|---|
| [acp](packages/acp.md) [stale] | 把 harness agent 通过 stdio JSON-RPC 暴露给自动化客户端的 ACP server；只发 committed 文本与图像，不做 UI 集成，不是 capability seam。 | Product | 1 | 9 |
| [api](packages/api.md) [stale] | Web GUI 的 Remote 栈：remotes、gateway及Session/Settings/Workspace controllers。 | Product | 5 | 11 |
| [attachment](packages/attachment.md) [stale] | 图片附件 seam（ctx.attachments）：源准入（8192px/20MiB/64Mpx）→ normalized 持久化（长边 2048px）→ readImageRequest 按路由预算派生请求版本；region reads 已移除。 | Product | 2 | 9 |
| [boot](packages/boot.md) [stale] | app bin 共用的启动库：.env 分层、fail-loud Loader 守卫、profile/bundle/patch 组合与热更、launcher 到 app 的命令行交接。 | Product | 2 | 10 |
| [bundle](packages/bundle.md) [stale] | dsh --profile 的可安装 patch 层；alpha.2 tag下共6个package manifest，具体composition待专项复验。 | Product | 6 | 12 |
| [client](packages/client.md) [stale] | Web GUI浏览器半边；含exact-preset create后的list/binding收敛、refresh sticky-ready与unknown分类。 | Product | 44 | 20 |
| [code-runtime](packages/code-runtime.md) [stale] | Code Mode 的执行底座：ctx.codeRuntime seam 加 worker-thread provider；isolation 只是标签不是安全声明，run 只返错不抛错。 | Product | 3 | 13 |
| [compaction](packages/compaction.md) [stale] | 会话压缩能力族：CompactionEngine seam、token 压力摘要 provider、无模型 tool-result 剪枝伴生、人类 /compact 命令。 | Product | 4 | 13 |
| [context](packages/context.md) [stale] | 请求上下文扩展组：工作区 AGENTS.md 指令加载、@file 引用 seam 与本地 provider、跨会话快照、时间与 tmux 位置上下文。 | Product | 6 | 12 |
| [core](packages/core.md) [stale] | 产品API主干；含request snapshot与runtime依赖边界、borrowed ToolDefinition、detached frozen schema、Context隔离与batch teardown。 | Product | 8 | 43 |
| [credentials](packages/credentials.md) [stale] | 凭据seam；含provider replacement、UI可替换但raw `credentials.set` wire仍存在的分层边界。 | Product | 3 | 15 |
| [e2b](packages/e2b.md) [stale] | E2B 远程运行时 POC：sandbox 生命周期所有者加 fs/subprocess 两个 adapter，让 bash、PTY、LSP 消费者无需分叉即可搬进沙箱。 | POC | 3 | 10 |
| [examples](packages/examples.md) [stale] | alpha.2 tag下只余agent-spine-demo package manifest；非产品API，历史demo结论待专项复验。 | Support | 1 | 11 |
| [experimental](packages/experimental.md) [stale] | 8个private实验package，含Agent Teams、inspector与webworker；无稳定性承诺。 | Unreleased | 8 | 13 |
| [extensions](packages/extensions.md) [stale] | Agent 自改运行时：模型自省 Cordis 服务/插件并动态定义、挂载、卸载自己写的双半 package，host 半跑在 node:vm 沙箱里。 | Product | 4 | 17 |
| [feedback](packages/feedback.md) [stale] | 人类反馈的两条互不相通契约：session log 里不可变的 feedback/record 事件，与挂在单条 assistant message 上可编辑的本地 sidecar。 | Product | 2 | 9 |
| [fs](packages/fs.md) [stale] | 文件系统 capability family：ctx.fs 的 12 个 primitive + 本地/沙箱/E2B 实现 + 纯事件策略门 + 模型工具，四层可独立替换。 | Product | 7 | 19 |
| [goal](packages/goal.md) [stale] | 同会话持久化目标：状态 event-sourced 进 session log，续跑权限 activation 从不持久化，state 与 scheduling 严格分家。 | Product | 4 | 13 |
| [guard](packages/guard.md) [stale] | loop卫生守卫与public policy hooks；含hard deny、Session-log条件性重建及run/rate-limit耐久缺口。 | Product | 2 | 14 |
| [hooks](packages/hooks.md) [stale] | Claude Code / Codex hook 桥接：把外部 shell-hook 协议翻译到 harness 自己的类型化拦截点，外加共享线协议库。 | Product | 3 | 13 |
| [host](packages/host.md) [stale] | Web Host/API；含Host create三边界、Host/Client blank分层、blank Session无delete、cold preset resume/history、commit-status与trust fence。 | Product | 7 | 33 |
| [identity](packages/identity.md) | 共享匿名关联 id（UUID v4）；它不是 Cordis plugin 而是普通共享库，telemetry、feedback 回执与 DeepSeek 请求头三处共用同一值。 | Product | 1 | 5 |
| [interaction](packages/interaction.md) [stale] | 人机协作平面；含Approval pair/owner signal、frozen arguments与definition重解析边界、无独立grant票据。 | Product | 5 | 15 |
| [jobs](packages/jobs.md) [stale] | 后台作业 capability family：ctx.jobs 契约、jobs-local 进程内实现、tool-jobs 三工具与完成通知；owner 隔离与唤醒预算是理解重点。 | Product | 3 | 10 |
| [llm](packages/llm.md) [stale] | LLM seam与adapters；含sessionless reasoning catalog、create后首prompt前selection、pi-ai、retry identity及工具wire边界。 | Product | 7 | 49 |
| [lsp](packages/lsp.md) [stale] | LSP 能力 seam：恰好四个语义操作、无 JSON-RPC 逃生口；lsp-stdio 通用 stdio 后端与模型侧 lsp 工具（一基 UTF-16 光标坐标）。 | Product | 3 | 14 |
| [mcp](packages/mcp.md) [stale] | MCP桥；含增量ToolRuntime贡献、内部64字符qualified name、out-of-tree adapter、static no-swap与schema/call check位置。 | README 未列出 | 1 | 19 |
| [plan](packages/plan.md) [stale] | plan mode 是 log-only 的 per-agent 协作状态而非 capability seam；/plan 命令进入，exit_plan_mode 经用户审批退出。 | Product | 1 | 7 |
| [preset](packages/preset.md) [stale] | 每会话组合；含standing identity、同步publication guard的live/persistence边界及exact restore。 | Product | 2 | 30 |
| [runtime-diagnostics](packages/runtime-diagnostics.md) [stale] | 包自有运行期不变式注册表 ctx.invariants：每个包发布 ./invariant companion，检查自己拥有的事件关系与可变数据关系。 | README 未列出 | 1 | 10 |
| [sandbox](packages/sandbox.md) [stale] | 进程限制能力族：ctx.sandbox.confine(argv, policy) 返回替代原 argv 的包装 argv，无可用后端就抛错；只管同世界子进程。 | Product | 4 | 14 |
| [schedule](packages/schedule.md) [stale] | Session 本地定时提醒：持久状态只存在原 Session 事件日志，到期项通过 Agent 普通 follow-up 队列回到同一段对话，无外部通知。 | Product | 1 | 13 |
| [sdk](packages/sdk.md) [stale] | stdio 上的 newline-delimited JSON-RPC 协议栈，让外部进程把 Harness runtime 当子进程驱动；不负责创建或构建开发者项目。 | Product | 3 | 16 |
| [session](packages/session.md) [stale] | 持久Session数据平面；含projection coldSnapshot无preset链、publication veto耐久边界与JSON快照。 | Product | 14 | 31 |
| [session-query](packages/session-query.md) [stale] | Session 检索能力族：逻辑语料、有界读取、血缘追踪、事件关系、语义过滤与 SQLite FTS5 全文搜索，独立于 compaction。 | Product | 4 | 16 |
| [settings](packages/settings.md) [stale] | 用户配置 namespace seam；含 `load`/`publish` provider 生命周期、update/replace/mutate 热提交、revision 与 composition base 不可由 user layer 删除的边界。 | Product | 2 | 9 |
| [shell](packages/shell.md) [stale] | bash/pwsh 执行器 seam 与本地、沙箱两类 provider，加模型侧 bash/pwsh 工具及两个走 ctx.terminals 的常驻版 | Product | 10 | 13 |
| [skill](packages/skill.md) [stale] | Skill Registry/filesystem；含selected root、locator double-open TOCTOU与strict validator边界。 | Product | 4 | 18 |
| [spill](packages/spill.md) [stale] | 超大工具输出落盘并换成有界预览加 locator；查 SpillStore seam、本地文件布局与 post-execute 策略 | Product | 3 | 9 |
| [storage](packages/storage.md) [stale] | session 日志以外数据枢纽；含 json/sqlite/domain 路由及 `:memory:` 仅随单一 live connection 存活的生命周期。 | Product | 4 | 10 |
| [subagent](packages/subagent.md) [stale] | 子 agent 委派能力族 11 包：多 provider 注册表、continuable 子代编排、父子双向三类模型侧工具 | Product | 11 | 12 |
| [subprocess](packages/subprocess.md) [stale] | 进程基座seam、本地provider与win32 helper；shell/lsp/terminal/acp建在它上面。 | Product | 3 | 10 |
| [terminal](packages/terminal.md) [stale] | 持久 PTY 会话能力族：ctx.terminals seam、bash/pwsh backend、六个 terminal_* 工具；会话进程本地，不跨重启 | Product | 3 | 9 |
| [test-support](packages/test-support.md) [stale] | 测试与开发基建：ACP 快照套件、AgentLoop testkit、LLM mock 与 replay、Loader 冒烟；support 级兼容性，非产品 API | Support | 6 | 10 |
| [todo](packages/todo.md) [stale] | 单包 todo 能力：todo_write 整表替换并写 todo/write 会话事件；单 agent session 独占，刻意无 provider seam | Product | 1 | 6 |
| [typert](packages/typert.md) [stale] | 类型图流水线：generator 构建期分析、registry 运行时 ctx.typert、loader 自动发现、protocol 声明 Remote 契约 | Product | 4 | 7 |
| [util](packages/util.md) [stale] | 零依赖共享原语；alpha.2新增deque、time、values，完整owner与稳定性待专项复验。 | Support | 12 | 11 |
| [web](packages/web.md) [stale] | web 能力族：ctx.web 单 seam 同时承载 search 与 fetch，四个 provider 加 tool-web；HTTP fetch 无 SSRF 防护 | Product | 6 | 12 |
| [workflow](packages/workflow.md) [stale] | 模型自写编排脚本能力族：ctx.workflowEngine seam、worker-thread 引擎、workflow 与 ralph 两个工具消费者 | Product | 4 | 10 |
| [webhook](packages/webhook.md) [stale] | 已认证外部事件的可信规则运行时：fire-and-forget 创建普通 Workspace Session，GitHub adapter 负责签名与有界 JSON intake | Product | 2 | 10 |
| [workspace](packages/workspace.md) [stale] | workspace 实体单包：ctx.workspaceRegistry 管理目录、标题与有序会话归属；realpath 为身份权威，模型不可见 | Product | 1 | 8 |
