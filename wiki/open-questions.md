---
title: DSH 悬而未决
description: 记录版本相关的未知边界、已核验证据及后续验证入口。
type: reference
status: active
updated: 2026-09-17
---

# 悬而未决（Open Questions）

> 记录**我查了但没查到答案**的问题：上游文档没写、源码看不出、需要实测或需要问上游。

> 与错误本互补：`errors.md` 记「答错的」，本文件记「没答上的」。作用是**知道自己不知道什么**。

> `/dsh-sync` 时对 `retry_on_sync: true` 的条目重新查证。

> 冷启动填充于 2026-08-20 · 上游 `141eb6f` · 原 114 条；2026-08-22 同步 0.1.1-rc.2 时重试：**5 条已解答删除**（Q014 credentials/updated 事件签名→事件已拆分见 errors.md E008；Q034 apiproxy config 校验→zod 源码确认；Q042 supportedProtocols→三协议源码确认；Q106 ACP 图片失败→AcpContentError 分类源码确认；Q114 credentials writable 语义→describe() 源码确认），**1 条更新进展**（Q043 catalog 清单需 pi-ai 运行时）；2026-08-27 联网复验 npm registry 后删除 Q095（SDK 包已公开发布，当前同版产物位于 `next`）；2026-08-30新增alpha.1发布/完整client两项未知。现存 110 条。多数仍需实测或等上游补文档。


| # | 来源单元 | 问题 |
|---|---|---|
| Q001 | `acp` | 组级 README（packages/acp/README.md）没有 Known Limitations 章节，仓库的 verify-package-readme-limitations gate 是否只覆盖包级 README，未从 scripts 侧核实 |
| Q002 | `api` | docs/api-gateway.md 与两个包 README 的重叠边界（谁是权威）未逐条比对 |
| Q003 | `boot` | apps/cli 自身的 flag 家族与 source-launch 钩子未细读（本次只到 boot 库边界） |
| Q004 | `client` | 29 个 ui-* 功能插件里，README 未列出的 4 个（ui-deliverables 等）各自职责未逐个读其包 README |
| Q005 | `client` | client/connection、client/modules、client/hmr、client/locale 的详细契约本次只到组级与 AGENTS.md 层面，未逐包细读 |
| Q006 | `code-runtime` | first-party 的 dsh-code-runtime-python 执行后端“delivered separately”具体交付在哪里（另一个仓库？未发布？），在 upstream 镜像内查不到 |
| Q007 | `code-runtime` | docs/subsystems/code-runtime.md 的失败分类与包 README 的 CodeRunFailure.kind 是否逐项一致，未逐条比对 |
| Q008 | `compaction` | compaction/summary 事件的完整字段列表未在组内 README 展开，需读 docs/persistence-catalog.md 生成区确认（L2） |
| Q009 | `compaction` | ManualCompactionError 的 commit 码在「部分变更」情形下调用方该如何恢复，README 只说刻意中立，未给恢复指引 |
| Q010 | `context` | file-reference 的 Service Definition 与 fs 组的 read 工具之间「命名空间对齐」由部署负责，但没有任何机制或校验保证——是否有推荐的对齐做法未见文档 |
| Q011 | `context` | agent-instructions 的 budget 诊断（baseline 可带空 change list 发布 budget diagnostic）具体渲染成什么模型可见文本，README 未给原文 |
| Q012 | `core` | agent-tool-presentation 缺席组 README 表格是遗漏还是刻意（它无 ctx key、是 preset row 而非服务），需查 preset 组 README 或 Agent Note 交叉确认 |
| Q013 | `core` | docs/subsystems/core.md 的 generated 区里 agent/* 事件的精确 dispatch mode 与 scope 过滤规则未在本页展开，L2 时应逐条核 |
| Q015 | `e2b` | e2b 组的 README 未列 fs-e2b/subprocess-e2b 的 Release expectation 是否也是 POC（组级标 POC，包级未单独标注） |
| Q016 | `e2b` | ctx.e2b 的完整公开方法集（除 getSandbox / cwd / runtimeRoot 外）需读 src/index.ts 确认（L2） |
| Q017 | `examples` | agent-spine-demo 的 apply() 实际挂载顺序是否与 README 列出的树顺序完全一致，未读 src/index.ts 逐行核对（L2） |
| Q018 | `experimental` | ctx.agentTeams 是否真的没有可替换的抽象 seam（当前判断基于组 README 与包 README 未提及 Service Definition/Provider 拆分），需 L2 读 src/index.ts 的服务声明确认 |
| Q019 | `experimental` | promotion 时 agent-team 应落到哪个产品角色组（subagent/ 还是新组）文档未指明 |
| Q020 | `extensions` | docs/subsystems/extensions.md 的生成区列出了 ctx.cordisInspect 的完整方法签名，但组 README 未提及 inspect provider 的注册契约细节（如 provider 优先级、跨页路由失败时的行为）——需要读 cordis-host-runner/src/inspect-registry.ts 源码才能确认。 |
| Q021 | `extensions` | cordis_define 的语法预检查具体用什么编译（vm.Script? ts 语法？），README 只说 “prechecks each half's syntax by compiling it (running nothing)”，未验证。 |
| Q022 | `feedback` | message-feedback 的 storage-domain 表结构（`message_feedback` domain 的 `sessions` 表）具体列定义未在 README 中给出，需读源码。 |
| Q023 | `feedback` | docs/subsystems/feedback.md 未通读，其生成区是否还列了本组之外的 ctx key 未确认。 |
| Q024 | `fs` | fs-e2b 在 packages/e2b/ 下，本次未读其 README，它对 12 个 primitive 的实现取舍（尤其 editText 是否用原生 compare-and-edit）未确认。 |
| Q025 | `fs` | docs/subsystems/filesystem.md 全文未通读，其中的 error taxonomy 与 outcome 表是否与包 README 完全一致未逐条核对。 |
| Q026 | `goal` | docs/subsystems/goal.md 的「goal type catalog」记录了字面数据形状，本次未通读，GoalView / GoalRef / phase 枚举的确切字段未逐条核对。 |
| Q027 | `goal` | goal 的 ./invariant companion 的具体注册名与检查项清单未读源码确认。 |
| Q028 | `guard` | 本组没有 docs/subsystems/guard.md，additionalContexts 如何被 loop 缓冲并追加为 injected user/message 的确切时序需去 docs/subsystems/tools.md 核对，本次未通读。 |
| Q029 | `guard` | deadline() / timeoutOf() 来自 @deepseek-ai/dsh-timeout（packages/util/timeout），其 MAX_TIMER_DELAY_MS 与本组的交互未验证。 |
| Q030 | `hooks` | 本组没有 docs/subsystems/hooks.md，hook/* 事件的确切载荷字段需去 docs/persistence-catalog.md 查，本次未通读。 |
| Q031 | `hooks` | stderrSummaryMaxChars 的参考默认值 DEFAULT_STDERR_SUMMARY_MAX_CHARS = 500 来自 README 陈述，未在 src/types.ts 中直接核对。 |
| Q032 | `hooks` | 「Code Mode defers sub-call contexts until the outer run_code result」这条在两个 bridge 的 PostToolUse 映射里都出现，但其与 packages/code-runtime 的具体交接点未验证。 |
| Q033 | `host` | packages/host/apiproxy/README.md 的 Contract layer 一节极长（session/workspace/settings/credentials/llm/agentPreset/command/skill 多个域），本次只做了要点提取，各域完整错误码集合未逐条核对。 |
| Q035 | `host` | docs/subsystems/web-server.md 与 workspace.md 未通读。 |
| Q036 | `identity` | DSH_TELEMETRY_DISABLED 的实际读取点不在本组源码内（README 声称由 telemetry 后端消费），未定位到具体包 |
| Q037 | `identity` | OpenTelemetry backend 把 id 报为 Resource user.id 的具体实现包未定位（session telemetry 相关，未在本批范围内核对） |
| Q038 | `interaction` | shipped 的 UserQuestionProvider 由「Web host runtime」提供，但具体是 packages/host 还是 packages/client 下的哪个包未定位 |
| Q039 | `interaction` | ACP automation bridge 提供 one-shot machine decision 的具体代码位置（packages/acp 内）未核对 |
| Q040 | `jobs` | 哪些 producer 包实际扩展了 JobKindMap（shell / terminal / workflow 等组）未逐一核对，因此现存的 job kind 全集未确认 |
| Q041 | `jobs` | docs/subsystems/jobs.md 里的 job type catalog 具体条目未展开阅读 |
| Q043 | `llm` | llm-pi-ai 的 configurable-provider 目录里实际 ship 的路由清单（README 只点名 openai-codex 是唯一随 catalog 出货的）——0.1.1-rc.2 源码核验：`catalogProviderIds()` 运行时取自 pi-ai 依赖的 `getBuiltinProviders()`，静态源码读不出，仍需实测或读 node_modules 里的 pi-ai 包 |
| Q044 | `llm` | token-meter 三个 projection 的 checkpoint 序列化格式未展开 |
| Q045 | `lsp` | 仓库内是否 ship 了现成的 lsp-stdio 服务器预设 overlay（README 说预设属于 cordis.yml overlay），未在 examples/ 下核对 |
| Q046 | `lsp` | pinned TypeScript e2e 建立的那条兼容性下限对应的测试文件位置未定位 |
| Q047 | `mcp` | packages/README.md 漏列 mcp/ 是纯遗漏还是刻意（例如该组是否计划排除在官方发布之外），未找到任何说明性文档 |
| Q048 | `mcp` | mcp-client 的 ./invariant companion 检查的具体关系未读（src/invariant.ts 未展开） |
| Q049 | `mcp` | 「精确能力证明」在 ctx.llm 侧走的是哪个 API（推测与 resolveModelInfo 的 modality 元数据相关）未确认 |
| Q050 | `plan` | Web 客户端里 plan-review 专用渲染器的具体文件位置（packages/client 下）未定位 |
| Q051 | `plan` | exit_plan_mode 被 review 批准后返回的 canonical { approved: true } 之外，plan 参数的完整 schema 字段未从 docs/tool-catalog.md 展开核对 |
| Q052 | `preset` | standing mount 的 generation 回收需要 ensureStanding 处的 joined-agent 计数（README 指向源码里的 TODO），当前没有回收计划的时间点。 |
| Q053 | `runtime-diagnostics` | runtime-diagnostics 组是否属于 Product 稳定性档位无法从文档确认（表格漏列）；只能从 package.json 非 private 与 packages/AGENTS.md 的硬性门禁推断按产品件对待。 |
| Q054 | `sandbox` | 带外 runner 状态通道（替代 stderr 方言与带内诊断）被标记为 deferred，没有计划时间。 |
| Q055 | `sdk` | Python SDK 的 responder 面向「future approval flows」，但审批流的协议方法尚未定义，无法从文档判断落地形态。 |
| Q056 | `session` | 遥测的持久 outbox（spool、per-sink cursor、at-least-once）明确 deferred，等待某个部署提出崩溃丢失要求；无时间点。 |
| Q057 | `session` | session-persistence-sqlite 的「统一关系型数据库设计（多后端 + 可配置 schema）」被 deferred，无路线图。 |
| Q058 | `session-query` | 除 SQLite FTS5 外是否会有第二个 searchSessions/searchEvents 实现，文档未表态；session-query README 只写「the first implementation is session-query-sqlite」。 |
| Q059 | `settings` | 描述里提到的 composition `base` 具体由谁供给（哪个包/哪段 cordis.yml 写入 base 层）未在本组 README 内说明，需读 docs/subsystems/settings.md 或 preset 组确认 |
| Q060 | `shell` | tool-bash-persistent / tool-pwsh-persistent 为何放在 shell 组而非 terminal 组（是历史归属还是刻意按「模型侧 shell 工具」聚合），组 README 未说明也未列出它们 |
| Q061 | `skill` | skill-filesystem 的具体扫描根（project / custom / user 三类根的默认路径与优先级 rank 数值）未在组 README 与包 README 摘要中给出，需读 docs/subsystems/skills.md 或该包源码确认 |
| Q062 | `storage` | storage-json / storage-sqlite 各自的必填配置键（root / 数据库文件路径）未在本次抽读范围内确认，需读两包 README 的 Config 章节 |
| Q063 | `subagent` | Issue #1838（agent-loop wake latch）的当前状态未在仓库内确认，是外部 issue 编号 |
| Q064 | `subagent` | subagent-in-process-driver 不注册 provider 也不注册 ctx key，它被两个进程内 provider 依赖的具体导出面未抽读 |
| Q065 | `subprocess` | docs/subsystems/subprocess.md 提到的 DSH_* 环境完整清单未在本次抽读中展开，需读该子系统页确认与 shell-env 内置三项的关系 |
| Q066 | `terminal` | TerminalBackendSession 接口的完整成员未逐一读取（本轮 L1 只读 README 与 index.ts 顶层导出） |
| Q067 | `terminal` | docs/subsystems/terminal.md 中「bounded reads / 分页」的确切参数语义未核对到源码 |
| Q068 | `test-support` | dsh-invariants 究竟属于 test-support 还是 runtime-diagnostics 组，两处文档口径不一致，未找到说明其归属的 Agent Note |
| Q069 | `test-support` | runtime-diagnostics 组的 Release expectation 无处可查（packages/README.md 未列出） |
| Q070 | `todo` | todo/write 事件在 SessionEventMap 中是否带 ignorable 标记未核对（只读了 README 与 docs/subsystems/session.md 的引用路径，未逐字段验证） |
| Q071 | `typert` | protocol 被排除在 group README 表格外是否刻意（因为它是纯声明包、没有 Cordis key），未找到说明该取舍的 Agent Note |
| Q072 | `typert` | packages/typert/registry/src/client/ 子目录的 Client 侧职责未展开（本轮只读 host face） |
| Q073 | `util` | util 组是否有意把 launch-environment 排除在 README 表格外（例如尚在过渡期），未找到相关 Agent Note |
| Q074 | `web` | web-search-deepseek 未出现在 packages/web/web/README.md 的角色表（该表只列 exa/perplexity/fetch-http/tool-web），但 group README 与目录都有它 —— 是否为该子表未同步，无法从文档确认 |
| Q075 | `web` | turndown 的 512 级 lexical guard 的具体触发条件只在 README 中描述，未读源码确认 |
| Q076 | `workflow` | workflow 脚本可用的全部全局钩子（agent()/parallel()/pipeline()/phase()/log() 之外还有哪些）未逐一核对到源码，README 只提及部分 |
| Q077 | `workflow` | worker-thread engine 的 Config 字段全表未读取（本轮只读 README 与包根导出说明） |
| Q078 | `workspace` | 该组的核心设计 Agent Note 位于 .agents/notes/proposed/（domain KV storage and workspace），即设计仍是 proposed 状态而实现已是 Product — stable API，两者关系未见文档说明 |
| Q079 | `workspace` | storageDomain 的 domain data form 具体表结构未展开（属于 packages/storage/ 组） |
| Q080 | `docs 文档路由地图` | docs/user/guide 与 docs/user/develop 下的具体页面清单未逐一枚举（本次只做目录级路由）。 |
| Q081 | `docs 文档路由地图` | website/ 的 VitePress 只投影 docs/ 的部分双语源，具体投影哪些页面未核实。 |
| Q082 | `DSH 整体骨架（跨 5 篇综合）` | agent/pre-step 的 14 个监听器之间的实际注册顺序（compaction-basic、hooks-*、plan-mode、agent-instructions 等）由 cordis.yml 行序决定，具体 shipped 顺序未核实。 |
| Q083 | `DSH 整体骨架（跨 5 篇综合）` | tools/execute 的 timeout-policy 包在 event-producer-consumer 矩阵中以无链接的裸名 `timeout-policy` 出现，其确切包路径未核实（tools/index.ts 注释提到 @deepseek-ai/dsh-tool-call-timeout-policy）。 |
| Q084 | `DSH 整体骨架（跨 5 篇综合）` | agent-loop 的 executeToolCalls 把 additionalContexts splice 进 inbox 的 next-step，与 tool-execution-pipeline 图里「批次结算后 FIFO 注入 user/message」的确切时序关系未逐行核实。 |
| Q085 | `Cordis 内核（DSH 的插件/服务/事件模型）` | vendored `vendor/cordis/src/` 相对 upstream rc.8 的实际差异，除 `vendor/README.md` 列出的 18 条外是否还有未记录的漂移？本次只做了行数与文件级 diff，未逐行归因。 |
| Q086 | `Cordis 内核（DSH 的插件/服务/事件模型）` | `FiberState` 在源码中是 `const enum`，而 `docs/cordis-tutorial/06-composition-and-hmr.md` 让用户在运行时 `import { FiberState } from '@deepseek-ai/cordis'` 并比较 `fiber.state === FiberState.PENDING`。在 tsc 发射路径（`tsconfig.base.json` 未设 `preserveConstEnums`）下这个运行时导出是否确实存在？教程是在 tsx/esbuild 下跑的，是否只在 tsx 下成立、`import` 到构建产物 `lib/` 时会失败？未实测。 |
| Q087 | `Cordis 内核（DSH 的插件/服务/事件模型）` | `ctx.timer` / `ctx.loader` / `ctx.hmr` 的注入条件与 mixin 细节（inherited.md 只给了一句摘要）——`vendor/timer`、`vendor/loader`、`vendor/hmr` 三个包的源码本次未读。 |
| Q088 | `Cordis 内核（DSH 的插件/服务/事件模型）` | Cordis 的 `Group` 与 `Include` 在 DSH 配置组合中的确切分工（谁负责 `isolate`、谁负责 patch 层、patch 为何不跨 include 边界），只从 `vendor/README.md` 的 local mod #11/#12/#15 描述间接得知，未读 `vendor/group/src/index.ts` 与 `vendor/include/src/index.ts` 源码。 |
| Q089 | `Cordis 内核（DSH 的插件/服务/事件模型）` | `internal/service`、`internal/get`、`internal/set` 这三个拦截型事件在 DSH 内的真实用途：`docs/event-producer-consumer.md` 只显示 `internal/service` 有 agent-presets 与 gateway 两个 listener，无 dispatcher，具体拦截语义未查。 |
| Q090 | `子系统路由地图（docs/subsystems 20 篇）` | packages/runtime-diagnostics/ 未被 packages/README.md 收录且无自身 README，是有意的内部组还是遗漏？ |
| Q091 | `子系统路由地图（docs/subsystems 20 篇）` | persistence.md 所说的第三个 provider 是历史遗留表述，还是指某个未在 packages/session/ 下的实现？ |
| Q092 | `四篇事故复盘的提炼（DSH postmortems）` | entry disabled 的插值能力是在哪个 PR 补上的（即 postmortem 0002 之后何时从「不插值」改为「每次挂载决策时插值」）？本次未查提交历史。 |
| Q093 | `四篇事故复盘的提炼（DSH postmortems）` | 0003 提到的 RFC/note 与 dsh web --dev 只挂载 HMR receiver 的细节，未进一步核对 apps/web 与 dsh web 的实际实现。 |
| Q094 | `集成表面全景` | PyPI 上 `deepseek-harness`（无 -sdk 后缀）这个第三方包的具体归属与内容无法从上游镜像证实——仓库内只有官方 dist 名的证据，需联网到 PyPI 才能核对。 |
| Q096 | `集成表面全景` | HTTP /api 的完整方法清单（RpcMethodMap 的实际成员）未逐一核对——本页只依据 apiproxy README 的散文描述，若需给非 TS 宿主写 HTTP 客户端，需要进一步读 packages/host/apiproxy/src/api/ 下的类型定义。 |
| Q097 | `集成表面全景` | 是否存在把 DSH 自身暴露为 MCP server 的通路：镜像内 packages/mcp 只有 mcp-client 一个包，未见 server 侧实现，但未穷尽搜索全仓是否有其他 MCP 服务端入口。 |
| Q098 | `Python SDK` | bundled 单文件 exe（dsh-jsonrpc-agent-pkg-*）与 node 载体闭包都不入 git，本次镜像里不存在，因此 exe/node 两种载体的实际启动行为、serverInfo.version 取值、以及 test_bundled_runtime.py 的真实通过情况均无法在 L2 阶段验证（该测试自身会 skip）。 |
| Q099 | `Python SDK` | session_prompt 之外的 contentBlocks 具体可用类型（ContentBlock 来自 @deepseek-ai/dsh-llm）未在 Python 侧做任何校验，normalize_input 只对 str 包成 {type:text}；非文本块（图片/附件）在 Python SDK 上的实际可用形状需读 TS 侧 ContentBlock 定义确认。 |
| Q100 | `Python SDK` | 多线程并发在同一个 HarnessClient 上跑不同 session 的 Session.run() 虽从代码看是安全的（各自订阅+过滤+写锁），但 tests/ 中没有对应用例覆盖，属于未经验证的推断。 |
| Q101 | `插件开发全路径（dsh-plugin）` | 第三方插件包是否有官方 npm 命名约定（例如必须 dsh-* 前缀或某个 npm keyword）？仓库内只找到 GitHub `dsh-plugin` topic 的发现约定，教程示例包名 dsh-hello-plugin 未见任何校验强制 |
| Q102 | `插件开发全路径（dsh-plugin）` | packages/client/tsdown.client.ts 的 clientBundle preset 未发布，仓库外的 client 插件要自行复现 lazy-CJS factory 产物格式——但该格式没有独立规范文档，只能读 preset 源码；是否有计划发布尚不明 |
| Q103 | `插件开发全路径（dsh-plugin）` | dsh.bundle manifest 除 `patch` 之外是否还有其它字段（版本约束、依赖 bundle 声明等）？publish.md 与 apps/cli/reference/README.md 都只出现 `{ "patch": "./cordis.patch.yml" }` 一种形态 |
| Q104 | `ACP 与 HTTP API gateway（进程外集成的两条协议路线）` | Typert Remote endpoint 若需要等价于 PRIVILEGED_METHODS 的 loopback pin，上游是否有计划（intercept 的 ConnectionRpcHandlerOptions 支持 authority: 'loopback'，但目前 gateway 只注册一个全局 'trusted-host' 拦截器，无法按 endpoint 区分）——未在文档中找到说明。 |
| Q105 | `ACP 与 HTTP API gateway（进程外集成的两条协议路线）` | ACP 的 approval/request waterfall 在多个 frontend 共存（如同时挂 ACP 与 Web）时的监听顺序与短路语义，未在 ACP README 或 docs 中找到跨 frontend 的仲裁说明。 |
| Q107 | `核心链 core / session / preset / llm（源码级 L2）` | rc.2 的 JSONL 跨进程竞争实际失败模式仍未实测。2026-09-05 重试最新版：`0.1.3-alpha.1` / `d347e703` 的 SessionWriteLease 已用内核锁排斥竞争写 owner，read handle 可共存；`lease.two-process.e2e.ts` 有双进程测试但本次未运行。此进展只适用于最新版，不能反推 rc.2；固定 SHA 与绝对路径见[最新版本对照](topics/版本变更-0.1.2-alpha.5-到-0.1.3-alpha.1.md)。 |
| Q108 | `核心链 core / session / preset / llm（源码级 L2）` | `SessionStore` 是否在 Cordis root 之外还有第二实例（例如 worker 线程内的 code-runtime）尚未核实，影响"进程内唯一 store"这一表述的边界。 |
| Q109 | `核心链 core / session / preset / llm（源码级 L2）` | `ctx.agentPresets.mount()` 在一方产品里除 apiproxy 的 `composeAgent()` 外是否还有其他生产调用点（例如 acp / headless profile）未穷尽检索。 |
| Q110 | `配置、工具目录与运行模式` | “Standard / Code / Minimal / Creator” 这套英文名只在 packages/client/ui-agent-preset/src/client/locales.ts 的 BUILT_IN_PRESET_KEYS 里；preset.yml 里的源文案是中文（标准模式 / PTC 模式 / 极简模式 / 创造模式）。是否还有 CLI/TUI surface 用第三套名字，本仓无法验证（TUI 不在这个仓库里）。 |
| Q111 | `配置、工具目录与运行模式` | DSH_TOOLS_MODE（host `tools` 行）与 per-agent agent-tool-presentation 同时设置时的最终优先级只从 config 声明与注释推断，未实测运行（L2 封顶，无实测）。 |
| Q112 | `配置、工具目录与运行模式` | headless profile 是否真的完全没有 agent preset 机制（即 `dsh --profile headless` 无法选运行模式）只从 packages/bundle/headless/cordis.patch.yml 缺 agent-presets 行推断，未实测。 |
| Q113 | `配置、工具目录与运行模式` | $DSH_HOME/settings.yaml 的 `llm-pi-ai` namespace 已在 2026-08-27 逐字段核对（见 `packages/llm.md`）；permission / agent-loop / agent-default-model / agent-presets / shell / llm-deepseek / web-search-deepseek 的完整 schema 仍未逐一核对。 |
| Q115 | `alpha完整Host嵌入` | alpha.2的245个非private tag package标识均有tarball；根+Skill/Preset/MCP/Session相关包的`--ignore-scripts`依赖解析与`npm ls --all`已通过（215个DSH包全为alpha.2）。lifecycle scripts、native helper、全部可选profile与真实boot仍未闭环。 |
| Q116 | `alpha完整Host嵌入` | 是否会提供完整语言无关Remote协议、正式Electron IPC carrier实现或第三方desktop embedding兼容保证；截至alpha.2仍未找到正式承诺。 |
| Q117 | `MCP modern generation` | 官方是否、何时会发布MCP 2026-07-28 subscriptions/listen、公开且耐久的server/tool generation snapshot，以及fresh observation→request/header→generation-bound call/result barrier；rc.2/alpha.2与SDK 1.29/1.30均未提供，未来API名与版本未知。 |
| Q118 | `ToolRuntime generation borrow` | 上游是否会把schema publication、body/output projection、post replacement、durable result与cancel/drain绑定到同一opaque ToolDefinition generation handle，并加入retire/refcount/release与外部carrier identity/digest seam；rc.2与正式npm alpha.2均无此合同。static no-swap语义可条件性不依赖它，但运行期replacement/teardown仍缺；未来兼容形状与版本未知。 |
| Q119 | `Host standing composition drain` | 上游是否会提供Host-wide admission-close→Agent drain→standing preset/plugin dispose的统一public coordinator，或给standing ToolDefinition registration公开waitForIdle/refcount；rc.2只有可手工排序的AgentHandle/Fiber primitives，未来API与版本未知。 |
| Q120 | `Skill authenticated bytes` | 上游是否会为Skill provider增加authenticated immutable blob/fd、digest-bound locator或compare-and-read contract，使admission validator与consumer load共享同一bytes；rc.2只有opaque locator与独立get，未来API与版本未知。 |
| Q121 | `0.1.3-alpha.1发布闭包` | 2026-09-06 GitHub tag `d347e703` 已发布且assets为空；根及11个关键companion精确npm alpha.1均404，R前10个有tarball元数据。完整递归依赖/native/build/boot未知；下一步须先获得同版发布产物及明确安装/conformance授权，不能用R包或其他SHA补成A闭包。详见[固定专题](topics/alpha13-full-host-public-boundaries.md)。 |
| Q122 | `0.1.3-alpha.1确定性工具日志` | `d347e703`公开tools.execute可由插件调用但不自动写Session tool pair；ApprovalService要求open turn，commands使用独立日志。没有取得无模型决定的final→deterministic tool→标准Session/approval完整同形一方验收证据；下一步先寻找公开编排/recording合同或正式例子，再在授权后实测，不用伪造model tool-call代替证据。 |
| Q123 | `0.1.3-alpha.1 WKWebView` | `d347e703`有公开ClientTransportHooks与Web组合，未找到WKWebView专门兼容承诺或一方测试；cookie持久化、custom scheme、WebSocket、文件上传与native bridge组合未实测。下一步需要明确WebKit/系统版本及运行时授权，不由通用Web能力推出桌面壳已验收。 |

| Q124 | `1.5-rc.2公开精确组合` | `fb2c4b9e698e30edb738bca4cf0618587db7d203`公开面支持无Session persistence的live运行、AttachmentStore拒绝、空Commands/Subagent registry和nativeOpen:false；但保留官方Session/Workspace/Conversation/Chat并关闭这些产品能力的完整组合未运行验证。已确认Client还需fileUpload、remote.commands/subagents及sidebarRight等，不能从可替换类型直接证明整图成立。下一步为目标版本公开组合的Loader/文本交互/Question/Todo/history/重连及副作用验收；不扩大为调用方项目评审。见[专题](topics/rc15-public-embedding-migration.md)。 |
