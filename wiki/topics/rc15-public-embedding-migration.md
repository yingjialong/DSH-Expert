---
title: 1.5-rc.2 公开嵌入、存储迁移与无持久化边界
description: 对照 1.1-rc.2 核验正式包公开面，区分服务依赖、产品能力和物理权限。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-17
updated: 2026-09-17
asked_by: agent
anchors:
  - packages/boot/app-boot/src/index.ts#boot
  - packages/boot/app-boot/README.md
  - packages/api/session-controller/src/index.ts#SessionController
  - packages/api/session-controller/src/client/index.ts#inject
  - packages/api/session-controller/src/commands.ts#selectModel
  - packages/client/connection/src/client/index.ts#ClientTransportHooks
  - packages/client/connection/src/rpc.ts#HostConnectionHandle
  - packages/session/session-persistence/src/index.ts#SessionPersistence
  - packages/session/session-persistence/src/handle.ts#SessionHandle
  - packages/session/session-format/src/types.ts#SessionFormatCatalog
  - packages/session/session-format-catalog/src/generated.ts#sessionFormatCatalog
  - packages/session/session-format-v0-to-v1/src/codec.ts
  - packages/session/session-format-v0-to-v1/src/validation.ts
  - packages/core/agent-loop/src/index.ts#resumeWith
  - packages/core/agent-loop/src/agent.ts#runMaintenance
  - packages/core/session/src/index.ts#snapshotEvents
  - packages/interaction/user-approval/src/index.ts#ApprovalService
  - packages/interaction/user-questions/src/index.ts#UserQuestionService
  - packages/todo/tool-todo/src/index.ts
  - packages/session/session-projection/src/index.ts
  - packages/core/agent/src/model-selection.ts
  - packages/credentials/credentials/src/index.ts
  - packages/mcp/mcp-client/src/transport.ts
  - packages/mcp/mcp-client/src/tools.ts
  - packages/skill/skill/src/index.ts
  - packages/skill/skill-filesystem/src/index.ts
  - packages/session-query/session-query/src/corpus.ts
  - packages/session-query/session-query/src/cold-read.ts
  - packages/session-query/session-query-sqlite/src/index.ts
  - packages/session-query/session-log-export/src/index.ts
  - packages/client/file-upload/src/index.ts
  - packages/attachment/attachment/src/index.ts
  - packages/client/ui-conversation/src/client/apply.ts
  - packages/client/ui-chat/src/client/apply.ts
  - packages/interaction/commands/src/index.ts
  - packages/subagent/subagent/src/index.ts
  - apps/desktop/package.json
  - apps/desktop-host/package.json
related:
  - "[[wiki/topics/版本变更-0.1.1-rc.2-到-0.1.5-rc.2]]"
---

# 固定版本与证据范围

- S：0.1.1-rc.2 / b150a551b8d465e31e418e1b2eaf5e79bbb7d28e。
- T：0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203。
- 两个远端tag与本地对象一致。按固定SHA读取，不切换共享工作树。
- 首批47个相关正式包版本样本、补充13个目标包样本分别核对npm sha512、exports、实际成员与.d.ts；关键根入口另核对lib/index.js最终exports。没有安装依赖、运行模型、启动GUI或测试自定义artifact。
- 统一L2 / verified_inference；类型/源码直接证据与条件性推论不构成产品兼容验收。已先完整回传并确认成功，后沉淀。

条件（八维坐标）：TS/JS宿主 · 自定义Loader图与跨进程carrier · live Agent与cold Session分层 · 自定义物理存储或无Session落盘 · 复用官方controllers/Client/工具 · secret与物理权限由外部owner管理 · 平台行为未实测 · 固定S/T。

## 公开boot与Host/Client迁移

T app-boot README明确保留direct-config helpers供lower-level embedders/tests。正式根入口仍有boot(binName,absoluteConfigPath,patches?,prepare?,bareModuleBaseUrl?):Promise<Context>；prepare在Loader安装后、配置entry挂载前运行。新增loadProfileDirectory，未要求嵌入者改用默认完整应用。

S host-apiproxy及client-runtime已从T发布树移除。T公开api-session-controller、api-workspace-controller、api-settings-controller、api-gateway、client-connection、client-store；controller拥有各域状态，store只是基础设施。Session/Workspace提供/client；controllers及相关域提供/remote和/typert；这些不是全部启动依赖的穷尽清单。

正式Session /client apply提供ctx.sessions；ISessions/ISession/SessionFace为公开类型，内部ClientSessions不从入口runtime导出。prompt/cancel accepted仍是admission，不是物理完成。

Connection /client公开ClientTransportHooks：fetch必需、openStream/loadBundle/ownsHost可选；Host有HostConnectionService与HostConnectionHandle/Rpc/Fetch，Gateway有wireStream。单一Gateway拥有connection loop。Desktop与desktop-host均private:true，framed pipes/Node IPC/dsh-app://是应用内部实现，不是现成公共UtilityProcess adapter。

Loader独立npm版本S为1.0.2、T为1.0.3，不得给vendor包统一套DSH版本号。本次DSH tarball样本虽声明./src/*，但无src成员；不能据通配导入内部源码。Loader样本另有src，分别判断。

## Session存储与纯迁移

S正式persistence根导出PersistenceCoordinator及PersistenceBackend类型；T均移除。T公开扩展面为SessionPersistence及SessionHandle：

| service | handle |
| --- | --- |
| create(header,options?)、open(id,read/write,options?) | id/header/inheritedEventCount/access |
| flush、stat、list | read、append、flush、close、AsyncDisposable |

T无旧supportsRawArtifacts/readRaw/locate方法；保留SessionLocation类型不表示保留这些方法。JSONL私有Storage/repair不是旧Backend的公共替代品。

- S append承诺resolve后耐久；T append允许缓冲，保证接纳、顺序和同backend instance后续读可见；flush是crash durability屏障，write close完成待处理耐久并释放owner。
- T create成功即在本进程stat/list/open可见；其他进程在materialize后可见。write独占，read可与writer共存。service.flush覆盖调用时活跃write handles，失败汇总AggregateError。
- revision只能在同service instance、同Session内比较；不是通用跨进程CAS版本。
- 物理坏尾由backend隐藏，并在首次写append前处理；handle.read不持久化语义补平。
- AgentLoop.resume先open(write)、read，再interruptedTurnClosers并handle.append；最终耐久仍受flush/close合同约束。
- 更高层readColdSessionLog则open(read)、read、close，向返回的内存数组追加closers，不写回；查看与恢复是不同路径。

正式session-format、session-format-catalog和v0-to-v1/v1-to-v2/v2-to-v3公开纯JSON编解码与迁移。catalog currentVersion=3，固定codec/相邻链独立于挂载插件；公开readHeader、createRestore、encodeCurrentHeader/Event。restore逐row decodeRow，最后finish返回header/inheritedEventCount/events，无文件、加密或CAS操作。

条件性推论：可在自定义存储解密后使用这些库，物理owner负责发布；不等于任何外部artifact都能被接纳。

严格边界：

1. 输入为发布格式的物理header/rows；V0要求type=session、version/id/createdAt/delegationDepth及可选seedLength，不是任意逻辑header/events对象。
2. T逻辑header使用isSeeded，inheritedEventCount并列携带；不能只改version。Session.events改用eventAt/snapshotEvents；旧decodeStorageRecord/packChunkRuns不再从Session根导出。
3. header迁移成功不代表body成功；必须完成全部stages与finish/目标validation。
4. V0未知历史event即使ignorable也拒绝；已知payload、关系与序列需通过校验。recoverable不跳过任意中段损坏。
5. V3插入system/message并重映射seq/引用/继承cut，精确改写旧code preset与PTC词汇；不可假定字节/hash/序号保持。
6. 官方JSONL保留旧generation并发布successor；纯库不替自定义介质保留原件。无官方反向链，旧版无法读取新V3与其新增记录。

## Agent、审批、Question/Todo与模型

- S AgentLoop.create同步，T返回Promise；runMaintenance两版公开，T仍须idle。whenIdle跟踪maintenance/driver，不代表长期admission关闭或外部detached效果停止。
- T移除ctx.agent；Inbox为类型接口，经agent.inbox使用；hasPending/claim不再公共。
- T tools/pre-execute、guard、ApprovalService.request仍可等待外部answerer；仅allowed-once授予执行，需open turn及asked/decided日志。事件声明移至/types、CallId→ToolCallId、effectiveApprovalPolicy根函数移除。批准不pin定义generation，body仍按名称/scope查当前定义。
- Question从S registerProvider单一provider改为T user-questions/request Agent-scoped waterfall；live root/signal校验仍存在。
- Todo fold仍turn/start清空、turn/end保留；T强inject sessionProjections，TodoItem来自工具types。
- Projection init变为init(header,inheritedEventCount)，seq品牌化，通知按wire.view的Object.is变化；不能照搬旧state-reference通知假设。
- installModelSelection仍公开；provider/model变化会向下一admitted request追加notice，effort-only不追加。
- T SessionController.selectModel校验路由、selectForNextRequest，再尝试保存agentDefaultModel；保存默认失败只warn、不回滚Session选择。此API不等于只改当前Session且绝不影响未来默认。
- CredentialProvider核心reference/record API及两类通知基本保持；resolve返回secret、按operation调用，不是外部授权票据。T pi-ai显式apiKeyEnv在已挂credentials时miss不回退环境；reasoning wire不保证沿用旧依赖默认。

## MCP与Skill

两版MCP根JS均仅Config/apply/inject/name，仍无公开carrier factory、外部Client注入或持久generation/authority接口。T serverName改为registration scope预留，tools/list新增重复cursor拒绝。发现失败保留旧工具、成功后重注册不等于definition pin/drain。

stdio由MCP SDK spawn，不走ctx.subprocess；HTTP由SDK transport连接，env/headers是配置值。替换subprocess/credentials不自动接管第一方MCP物理I/O。

Skill主体registry/provider.list/get及opaque locator合同基本相同；T移除空invariant子路径。FileSystemSkillProvider非trusted body在ctx.fs存在时走resolve/stat/readText；缺服务或trusted bundled读node fs，并仍有host roots/watchers。替换ctx.fs不证明接管全部文件操作；自定义SkillProvider公开，但locator不成为authenticated immutable bytes，scope覆盖不成为deny过滤。

## 无Session持久化的公开路径

AgentLoop.createStoredSession在缺sessionPersistence时返回undefined，新live Agent/Session照常发布。SessionQuery可不挂persistence，live对象优先；list合并attached Sessions，page/follow可读live内存（还需cwd/address等访问条件）。Client尚未加载不等于Host对象已cold。

真正detach或重启后无live也无provider则not found；resume要求provider。官方session-log-export独立插件要求query+persistence+attachments，缺项500，不自动回退live导出。

T未找到正式Session memory provider，旧persistence-sqlite已删除。session-query-sqlite仍正式发布，支持path=:memory:；这是派生索引。openAt=never不导入/打开SQLite，保留exact read/filter/trace，全文搜索报SEARCH_DISABLED。

无Session persistence不等于所有插件零磁盘I/O；Workspace、settings、credentials、cache、telemetry等另有owner。没有建议明文临时JSONL替代零落盘。

## 官方UI硬依赖与拒绝能力

| 公开owner | 目标硬依赖摘要 |
| --- | --- |
| SessionController | agentDefaultModel, agents, attachments, fileUploads, llm, sessions, sessionProjections, sessionQuery, typert, workspaceRegistry |
| Session /client | connection, fileUpload, typert, remote, remote.commands, remote.session, remote.subagents |
| FileUploads Host | agents, attachments, commands, connection；注册streaming Fetch route和command receipt resolver |
| ui-conversation /client | slots, sessions, fileUpload, uiSession, uiWorkspace, locale, settingsScope |
| ui-chat /client | slots, sessions, uiSession, uiConversation, locale, settingsScope, remote, remote.session, sidebarRight |

不存在已核验的总disabled开关。ConversationConfig上传并发最小1，不能设0当关闭上传。

- AttachmentStore是公开abstract Service，可在provider拒绝image方法；saveFile/saveFileStream默认拒绝FILES_UNSUPPORTED，纯文本admit不写image存储。没有找到现成官方DenyAllAttachmentStore。
- Commands Runtime可为空registry，SubagentRuntime可无执行provider；挂服务/namespace不等于开放命令/子agent执行。
- 不应把读catalog一律拒绝当作空清单：Client refreshSubagents的拒绝会进入error状态。
- 只拒绝remote.upload不能覆盖streaming Fetch、prompt图片和command附件路径；各真实owner须满足其合同。Main deny不是DSH内置的全路径权限证明。
- Session/Settings controller公开nativeOpen:false只约束各自native opener；不自动约束独立open-in-app、目录picker或Desktop shell。
- Gateway.$mount和owner /remote支持显式namespace组合；api-remotes/client默认选择多个域，不是最小Session图。

条件性推论：上述公共面能表达“保留必需服务但拒绝写能力”，无需复制官方reducer；未知的是这套精确组合的完整Loader readiness、Client呈现、prompt/Question/Todo/history/重连及side effects闭包。没有组合实测，不标为可运行验收。

## 定位证据

关键文件均在固定T的 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/`，以anchors中的文件/符号执行git show。重点：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-persistence/src/index.ts`：新公开接口。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-format/src/types.ts`：纯迁移API。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/index.ts`：createStoredSession/resumeWith。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session-query/session-query/src/cold-read.ts`：查看用内存closers。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-chat/src/client/apply.ts`：Sidebar等硬依赖。

正式包元数据：[persistence目标](https://registry.npmjs.org/@deepseek-ai%2Fdsh-session-persistence/0.1.5-rc.2)、[persistence来源](https://registry.npmjs.org/@deepseek-ai%2Fdsh-session-persistence/0.1.1-rc.2)、[format catalog](https://registry.npmjs.org/@deepseek-ai%2Fdsh-session-format-catalog/0.1.5-rc.2)、[query sqlite](https://registry.npmjs.org/@deepseek-ai%2Fdsh-session-query-sqlite/0.1.5-rc.2)。
