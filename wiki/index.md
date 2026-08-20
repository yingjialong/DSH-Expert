# 知识库索引（index.md）

> **回答时先读本文件**，据此挑 3-5 篇全文，**永远不要全量加载知识库**。

> 上游基线：DSH `141eb6f` · Cordis `8cc9e33` · 核验于 2026-08-20

> 图例：`status` 全为 `verified_inference`（冷启动只到 L1/L2，按封顶规则不得标 `fact`）；`fresh` 表示锚点未被上游改动。


## 治理文件

| 文件 | 用途 | 何时读 |
|---|---|---|
| [errors.md](errors.md) | 错误本：答错的、死路、命名/版本陷阱 | **每次回答前必读** |
| [conflicts.md](conflicts.md) | 文档与源码不符登记册（75 条） | 引用上游文档作结论前 |
| [open-questions.md](open-questions.md) | 悬而未决（114 条） | 查不到答案时先看是否已知 |
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
| [integration-surfaces.md](integration/integration-surfaces.md) | 外部项目接入 DSH 的五条通路（TS SDK / Python SDK / ACP / HTTP /api / CLI-headless）、各自官方支持程度与硬约束，以及非 TS/Python 宿主的选型判断 | L1 | 29 |
| [python-sdk.md](integration/python-sdk.md) | Python SDK（deepseek-harness-sdk）源码级用法：两层 API、runtime channel 选择链、零配置注入条件、会话树通知路由、测试即规格 | L2 | 31 |
| [protocol-jsonrpc.md](integration/protocol-jsonrpc.md) | SDK JSON-RPC 线协议源码级契约：分帧规则、握手/readiness 边界、3 请求 + 4 通知的完整清单、会话并发模型、错误码偏差、任意语言宿主的最小实现清单 | L2 | 17 |
| [protocol-acp-http.md](integration/protocol-acp-http.md) | ACP（automation-only stdio JSON-RPC）与 HTTP API gateway（/api + Typert Remote）两条进程外集成路线的能力边界、鉴权、会话映射与选型对比 | L2 | 28 |

## 主题（topics/）

| 页面 | 一行摘要 | 掌握度 | 锚点 |
|---|---|---|---|
| [docs-map.md](topics/docs-map.md) | DSH docs/ 顶层 19 主题 + 7 个子目录的路由表：什么问题该查哪个文件、篇幅规模、generated/curated 归属、问题→文件反向索引 | L1 | 21 |
| [architecture-overview.md](topics/architecture-overview.md) | 一次用户输入从 agent.followup() 到 tool/result 的完整旅程：21 步逐步标注层/seam/durable-vs-live 事件，工具管线展开，三事件域判据，seam 三角色，11 个易误解术语，7 条源码交叉验证事实 | L2 | 13 |
| [cordis-primer.md](topics/cordis-primer.md) | - [Cordis 内核入门](topics/cordis-primer.md) — Context/Service/Plugin/Fiber/Registry/Effect 模型、ctx-key 服务解析、五种事件 dispatch mode、与 DSH tools 服务的接缝，以及 `export default` 吞 inject 等 9 条陷阱。任何 DSH 插件问题的前置知识。 | L2 | 35 |
| [subsystems-map.md](topics/subsystems-map.md) | 20 个子系统 → ctx 服务名 → packages 组 → 该查什么；含 spine/optional 判定规则与文档可信度分级 | L1 | 22 |
| [postmortems.md](topics/postmortems.md) | 0001 export default 吞掉 inject / 0002 !!js 位置错致 fs 工具永久禁用 / 0003 Web agent 验证替身服务器 / 0004 Landlock 提示被误判——现象·根因·修复·可泛化教训 | L2 | 13 |
| [plugin-development.md](topics/plugin-development.md) | [插件开发全路径](topics/plugin-development.md) — 从零写 dsh-plugin 的步骤路由、四种插件形态骨架、依赖纪律（Service Definition 而非 Provider）、Config/schema、事件与扩展点选择、skill provider registry、bundle/profile 发布与八条陷阱 | L2 | 35 |
| [core-chain.md](topics/core-chain.md) | core-chain.md — agent loop 函数级调用链、agent-loop 可替换性的三道机制保证、session 三层 derive 与持久化/projection seam、会话并发隔离 12 条「条件→结论」、preset standing mount 与 scope 父链、LlmAdapter 接入点、system-prompt/context 注入时机 | L2 | 27 |
| [config-and-tools.md](topics/config-and-tools.md) | 配置与工具目录 — 非机密值六层覆盖梯子 + 凭据四层梯子 + 四种运行模式的真实定义（Creator mode 的目录 id 是 cordis）+ dsh --profile/bundle 补丁层 + tool-catalog 路由 + permission preset/approval。排错类问题的第一入口。 | L2 | 26 |

## 包组（packages/）—— 50 组 / 226 包

| 包组 | 一行摘要 | 稳定性 | 包数 | 锚点 |
|---|---|---|---|---|
| [acp](packages/acp.md) | 把 harness agent 通过 stdio JSON-RPC 暴露给自动化客户端的 ACP server；只发 committed 文本与图像，不做 UI 集成，不是 capability seam。 | Product | 1 | 9 |
| [api](packages/api.md) | Web GUI 的 Remote 栈：remotes 拿 BFF 身份策略与能力选型，gateway 实现 Host/Client 两侧共用的 Typert unary RPC 端点。 | Product | 2 | 11 |
| [attachment](packages/attachment.md) | 不可变图片附件的准入与持久化 seam（ctx.attachments）加本地内容寻址实现；未发送的浏览器草稿刻意不在此能力内。 | Product | 2 | 9 |
| [boot](packages/boot.md) | app bin 共用的启动库：.env 分层、fail-loud Loader 守卫、profile/bundle/patch 组合与热更、launcher 到 app 的命令行交接。 | Product | 2 | 10 |
| [bundle](packages/bundle.md) | dsh --profile 的可安装 patch 层：base/headless/web-app 三个在库 bundle，身份由 manifest 的 dsh.bundle.patch 定义。 | Product | 3 | 12 |
| [client](packages/client.md) | Web GUI 浏览器半边共 40 个包：两阶段 boot、React-free 对象层、slot 组合系统与 29 个 ui-* 功能插件。 | Product | 40 | 16 |
| [code-runtime](packages/code-runtime.md) | Code Mode 的执行底座：ctx.codeRuntime seam 加 worker-thread provider；isolation 只是标签不是安全声明，run 只返错不抛错。 | Product | 3 | 13 |
| [compaction](packages/compaction.md) | 会话压缩能力族：CompactionEngine seam、token 压力摘要 provider、无模型 tool-result 剪枝伴生、人类 /compact 命令。 | Product | 4 | 13 |
| [context](packages/context.md) | 请求上下文扩展组：工作区 AGENTS.md 指令加载、@file 引用 seam 与本地 provider、跨会话快照、时间与 tmux 位置上下文。 | Product | 6 | 12 |
| [core](packages/core.md) | 产品 API 主干：session 日志、system-prompt 组装、tools 注册与执行流水线、Agent seam 与注册表、默认模型、唯一那份具体 loop。 | Product | 8 | 18 |
| [credentials](packages/credentials.md) | 凭据引用能力族：配置只带引用不带密钥，seam 定义 resolve/describe/set/unset，本地 provider 用四层环境与文件优先级实现。 | Product | 2 | 10 |
| [e2b](packages/e2b.md) | E2B 远程运行时 POC：sandbox 生命周期所有者加 fs/subprocess 两个 adapter，让 bash、PTY、LSP 消费者无需分叉即可搬进沙箱。 | POC | 3 | 10 |
| [examples](packages/examples.md) | 现成可跑的 demo bundle：agent-spine-demo 共享主干，acp-demo 与 sdk-jsonrpc-demo 各加一个入口点；非产品 API。 | Support | 3 | 11 |
| [experimental](packages/experimental.md) | 私有实验包组：只含 Agent Teams（ctx.agentTeams 域服务与模型面工具），private、无稳定性承诺、release 包不得依赖。 | Unreleased | 2 | 13 |
| [extensions](packages/extensions.md) | Agent 自改运行时：模型自省 Cordis 服务/插件并动态定义、挂载、卸载自己写的双半 package，host 半跑在 node:vm 沙箱里。 | Product | 4 | 17 |
| [feedback](packages/feedback.md) | 人类反馈的两条互不相通契约：session log 里不可变的 feedback/record 事件，与挂在单条 assistant message 上可编辑的本地 sidecar。 | Product | 2 | 9 |
| [fs](packages/fs.md) | 文件系统 capability family：ctx.fs 的 12 个 primitive + 本地/沙箱/E2B 实现 + 纯事件策略门 + 模型工具，四层可独立替换。 | Product | 7 | 19 |
| [goal](packages/goal.md) | 同会话持久化目标：状态 event-sourced 进 session log，续跑权限 activation 从不持久化，state 与 scheduling 严格分家。 | Product | 4 | 13 |
| [guard](packages/guard.md) | loop 卫生守卫：重复工具调用的劝告式提醒 + tools/execute 上的协作式 per-call deadline；不是能力，是纯消费者。 | Product | 2 | 9 |
| [hooks](packages/hooks.md) | Claude Code / Codex hook 桥接：把外部 shell-hook 协议翻译到 harness 自己的类型化拦截点，外加共享线协议库。 | Product | 3 | 13 |
| [host](packages/host.md) | Web GUI 的 host 半：共享 API gateway ctx.apiProxy、裸 HTTP 承载 ctx.webServer、SPA 兜底、目录选择 seam、Loader 插件清单投影。 | Product | 8 | 22 |
| [identity](packages/identity.md) | 共享匿名关联 id（UUID v4）；它不是 Cordis plugin 而是普通共享库，telemetry、feedback 回执与 DeepSeek 请求头三处共用同一值。 | Product | 1 | 5 |
| [interaction](packages/interaction.md) | 人机协作平面：user-questions 与 user-approval 两条形状不同的 seam，加上 permission-presets、commands 两个产品面与 ask_user_question 工具。 | Product | 5 | 14 |
| [jobs](packages/jobs.md) | 后台作业 capability family：ctx.jobs 契约、jobs-local 进程内实现、tool-jobs 三工具与完成通知；owner 隔离与唤醒预算是理解重点。 | Product | 3 | 10 |
| [llm](packages/llm.md) | LLM seam 与两个 twin adapter（deepseek-official / pi-ai），外加 token-meter 计量与 llm-retry 重试执行器；HarnessError 基类也住在 dsh-llm 里。 | Product | 5 | 16 |
| [lsp](packages/lsp.md) | LSP 能力 seam：恰好四个语义操作、无 JSON-RPC 逃生口；lsp-stdio 通用 stdio 后端与模型侧 lsp 工具（一基 UTF-16 光标坐标）。 | Product | 3 | 14 |
| [mcp](packages/mcp.md) | MCP 客户端桥：把外部 server 的 tool 以 mcp__<serverName>__<rawName> 注册到 ctx.tools；注意 packages/README 组表格漏列了该组。 | README 未列出 | 1 | 7 |
| [plan](packages/plan.md) | plan mode 是 log-only 的 per-agent 协作状态而非 capability seam；/plan 命令进入，exit_plan_mode 经用户审批退出。 | Product | 1 | 7 |
| [preset](packages/preset.md) | 每会话 agent 组合：一个 preset 目录装一份 agent.cordis.yml，挂载后该 session 独享自己的 tools 与 prompt sections，其他 session 不受影响。 | Product | 2 | 12 |
| [runtime-diagnostics](packages/runtime-diagnostics.md) | 包自有运行期不变式注册表 ctx.invariants：每个包发布 ./invariant companion，检查自己拥有的事件关系与可变数据关系。 | README 未列出 | 1 | 10 |
| [sandbox](packages/sandbox.md) | 进程限制能力族：ctx.sandbox.confine(argv, policy) 返回替代原 argv 的包装 argv，无可用后端就抛错；只管同世界子进程。 | Product | 4 | 14 |
| [schedule](packages/schedule.md) | Session 本地定时提醒：持久状态只存在原 Session 事件日志，到期项通过 Agent 普通 follow-up 队列回到同一段对话，无外部通知。 | Product | 1 | 13 |
| [sdk](packages/sdk.md) | stdio 上的 newline-delimited JSON-RPC 协议栈，让外部进程把 Harness runtime 当子进程驱动；不负责创建或构建开发者项目。 | Product | 3 | 16 |
| [session](packages/session.md) | 持久 Session 数据平面：持久化 seam 加 JSONL/SQLite 后端与 checkpoint 策略、投影 seam、日志推导标题、外发遥测，四条独立 seam。 | Product | 13 | 21 |
| [session-query](packages/session-query.md) | Session 检索能力族：逻辑语料、有界读取、血缘追踪、事件关系、语义过滤与 SQLite FTS5 全文搜索，独立于 compaction。 | Product | 4 | 16 |
| [settings](packages/settings.md) | 用户可编辑配置的 namespace 注册与分层解析 seam，加一个文件后端 provider；查配置读写、热重载、secret 脱敏来这里 | Product | 2 | 7 |
| [shell](packages/shell.md) | bash/pwsh 执行器 seam 与本地、沙箱两类 provider，加模型侧 bash/pwsh 工具及两个走 ctx.terminals 的常驻版 | Product | 10 | 13 |
| [skill](packages/skill.md) | skill provider 注册表与本地文件发现，加模型侧目录与 skill 加载工具；查分层去重、invocation policy | Product | 4 | 7 |
| [spill](packages/spill.md) | 超大工具输出落盘并换成有界预览加 locator；查 SpillStore seam、本地文件布局与 post-execute 策略 | Product | 3 | 9 |
| [storage](packages/storage.md) | session 日志以外数据的存储枢纽：命名后端 json/sqlite 加 domain 数据形态；查后端注册与域路由 | Product | 4 | 10 |
| [subagent](packages/subagent.md) | 子 agent 委派能力族 11 包：多 provider 注册表、continuable 子代编排、父子双向三类模型侧工具 | Product | 11 | 12 |
| [subprocess](packages/subprocess.md) | 进程基座 seam 加本地 provider：可执行查找、托管进程树、PTY 原语；shell/lsp/terminal/acp 都建在它上面 | Product | 2 | 10 |
| [terminal](packages/terminal.md) | 持久 PTY 会话能力族：ctx.terminals seam、bash/pwsh backend、六个 terminal_* 工具；会话进程本地，不跨重启 | Product | 3 | 9 |
| [test-support](packages/test-support.md) | 测试与开发基建：ACP 快照套件、AgentLoop testkit、LLM mock 与 replay、Loader 冒烟；support 级兼容性，非产品 API | Support | 6 | 10 |
| [todo](packages/todo.md) | 单包 todo 能力：todo_write 整表替换并写 todo/write 会话事件；单 agent session 独占，刻意无 provider seam | Product | 1 | 6 |
| [typert](packages/typert.md) | 类型图流水线：generator 构建期分析、registry 运行时 ctx.typert、loader 自动发现、protocol 声明 Remote 契约 | Product | 4 | 7 |
| [util](packages/util.md) | 零依赖共享原语：Branded 类型、DSH home 路径、timeout 分类、输出保留、原子写、原生命令、启动环境快照 | Support | 7 | 11 |
| [web](packages/web.md) | web 能力族：ctx.web 单 seam 同时承载 search 与 fetch，四个 provider 加 tool-web；HTTP fetch 无 SSRF 防护 | Product | 6 | 12 |
| [workflow](packages/workflow.md) | 模型自写编排脚本能力族：ctx.workflowEngine seam、worker-thread 引擎、workflow 与 ralph 两个工具消费者 | Product | 4 | 10 |
| [workspace](packages/workspace.md) | workspace 实体单包：ctx.workspaceRegistry 管理目录、标题与有序会话归属；realpath 为身份权威，模型不可见 | Product | 1 | 8 |
