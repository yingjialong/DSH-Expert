---
title: DSH 问答与学习日志
description: 记录 DSH 咨询、核验、学习与知识沉淀的历史证据。
type: reference
status: active
updated: 2026-09-20
---

# 问答与学习日志

## 2026-09-20 · DSH-015-B3-AUDIT-01

- asked_by: agent；固定fb2c4b9e与指定cordis4.0.2/loader1.0.3。四正式包sha512/types/root JS核验；tag一致。完整回传成功后沉淀。
- 确认reflect.store公开可达但非root ReflectService导出；隔离symbol与状态分离；Loader不足覆盖独立preset；FiberState只有const enum；工具registry与wire不同。无Profile执行或外部项目评审。
- sandbox/b3audit01为临时包检查目录，收尾清除；专题/索引更新，L2 / verified_inference。

## 2026-09-19 · DSH-015-B2-CATALOG-01

- asked_by: agent；固定S b150a551 / T fb2c4b9e完整SHA与tag一致。6个目标正式包sha512/exports/types/JS核验，测试仅读。完整回传成功后沉淀。
- 澄清catalog内部restoreCurrent非root函数、current与transformed差别；S缺depth写0、seedLength缺席与0不同；合法unfinished不等物理截断，strict/current不证明原字节完整。
- 未执行示例或真实迁移；sandbox/catalog01仅发布物检查，收尾清除；新增去项目化专题与索引。

## 2026-09-19 · DSH-015-B1-CONNECTION-01

- asked_by: agent；固定fb2c4b9e698e30edb738bca4cf0618587db7d203，tag一致。4个正式包sha512/exports/types及关键JS核验，测试仅读。完整回传isError=false后沉淀。
- 确认官方apply无Web仍初始化BrowserAuth、无公开auth-free factory；公开Handle可替代的条件与caller Fiber注册/Fetch/wire信任义务分别说明，不建议内部BrowserAuth cast。
- 无组合运行、无模型或用户凭据访问。sandbox/b1connection01临时包检查目录收尾清除；新增去项目化专题/索引。

## 2026-09-18 · DSH-RC2-ARCH-05 旧版本架构接缝

- asked_by: agent；精确0.1.1-rc.2 / b150a551b8d465e31e418e1b2eaf5e79bbb7d28e，未套用默认1.5。17个npm包sha512/exports/types核对，固定源码核验。
- 完整答复回传成功后沉淀：公开provider可组合但非全I/O接管；Skill registry/consumer/provider分层；out-of-tree MCP不等于官方adapter。无运行测试或调用方项目检查。
- sandbox/rc2arch05为本轮临时包检查，收尾清除。新增专题/索引；L2 / verified_inference。

## 2026-09-18 · DSH-15-KV-UI-04 共享KV、checkpoint与Workspace UI

- asked_by: agent；固定fb2c4b9e698e30edb738bca4cf0618587db7d203，tag核对一致。完整回传isError=false后沉淀。
- 7个正式包sha512/exports/声明/关键JS核对；Domain仅本实例队列及快照、无reload/CAS replay；Workspace Remote是同Host控制/观察面。
- 核读policy单位与crash测试，含dispose后flush不增negative control、nested跳过；未执行测试。确认connectWorkspace真实SessionId合同和默认Composer无Session不可编辑，未给项目方案判断。
- 新增专题与索引，L2 / verified_inference；sandbox/kv-ui04临时检查目录收尾清除。

## 2026-09-18 · DSH-15-REVIEW-03 发布、输入预算与实时存储

- asked_by: agent；固定S b150a551b8d465e31e418e1b2eaf5e79bbb7d28e / T fb2c4b9e698e30edb738bca4cf0618587db7d203，远端tag一致。完整回传成功后沉淀。
- 13个T正式包tarball经sha512、exports/types/关键JS核验；scope定位为core/scope。无产品执行或模型调用；测试仅读同步veto/异步reject等断言。
- 新增专题澄清publication已setup、requestId检查非在途CAS、每prompt次数非内建预算、provider-owned live routing及checkpoint-policy不替代router。未认证任何外部后端/项目。
- sandbox/review03为本轮临时发布物检查，收尾清除；L2 / verified_inference。

## 2026-09-18 · 1.5-rc.2受控profile、文件链与存储/UI扩展面

- asked_by: agent；咨询DSH-15-FILES-02，固定fb2c4b9e698e30edb738bca4cf0618587db7d203；远端tag一致。完整自包含回传工具返回isError=false后才沉淀。
- 23个正式npm包：urllib读取registry/tarball、sha512比对全部通过；exports/实际成员/.d.ts及关键JS核验。最初tool-read路径不存在，按目标树定位正式tool-fs，未用缺失路径推断缺包。
- 新增证据：loadProfileDirectory不自动base、userLayer=false与boot分层；FileBlock无条件handle文本；custom fetch不转接onProgress；未消费receipt无公开删除/对象GC；present不保存字节；/client为ModuleLoader工厂；backupRecord可选及cache flush顺序。
- L2 / verified_inference；未安装/运行Host、GUI、模型、解析或用户迁移。临时tarball检查位于sandbox本轮专用目录，收尾清除；只写去项目化专题、索引与本日志。

## 2026-09-17 · 1.1-rc.2→1.5-rc.2公开嵌入及两项补充

- asked_by: agent；来源与目标固定b150a551b8d465e31e418e1b2eaf5e79bbb7d28e / fb2c4b9e698e30edb738bca4cf0618587db7d203。主答复与补充各自完整回传，工具isError=false确认后才写入本条。
- 首批47、补充13个正式npm包版本样本：urllib只读下载tarball、hashlib sha512比registry integrity、tarfile检查exports/声明/关键根JS导出；无npm install、无产品执行。临时检查目录为sandbox内本轮专用路径，收尾清理。
- 回源核验重点：Coordinator/Backend删除、handle耐久、纯format迁移及V0拒绝、Question waterfall、Todo投影硬依赖、公开Client/Chat硬依赖；补充确认无persistence live路径、cold查看内存closers、export需provider、query-sqlite memory/never与deny能力边界。
- 未对任意加密artifact、CAS、物理drain或精确UI装配作运行验收，不以源码证据假报兼容。新增rc15-public-embedding-migration专题，L2 / verified_inference；历史版本结论不覆盖。

## 2026-09-17 · 指定升级 0.1.5-rc.2，并累计比较 0.1.1-rc.2

- asked_by: human。起始本库及两个上游工作区均干净；用户明确授权升级指定版本。执行 fetch tags 与 ff-only 到 fb2c4b9e698e30edb738bca4cf0618587db7d203；未追随远端 1.6-alpha.2，未升级独立 Cordis。
- 本地镜像 d347e703→fb2c4b9e：1,162 提交、6,935 文件；对比 b150a551→fb2c4b9e：3,225 提交、11,427 文件、+618339/-248238（git diff --no-renames，包含 merge/文档/测试/生成物）。初次默认 diff 的 rename 检测触及限制，故报告统一使用 no-renames 口径。
- 固定 tag 远端与本地 SHA/manifest 一致；旧版同步 AgentLoop.create 与新版 async create、SessionHandle、V3 migration、Inbox、persona、SubprocessHandle、MCP cursor 和代理例外已回源核验。最新 alpha 只验证源码路由，不改目标基线。
- 锚点审计扫描 frontmatter anchors 字段，未把 related 链接算作源码锚点：原69页stale中68页命中；原30页fresh中29页命中；426条唯一路径命中；30页共395条固定历史锚点存在。28个命中历史专题保留原SHA；唯一通用fresh页identity临时失效后复验更新，E048登记两处旧表述。原stale页未全量复验。
- npm根包及8个关键companion的0.1.5-rc.2元数据/tarball URL可用，controller包名从目标manifest取api前缀；初次猜测无api前缀产生404，不作为发布缺失证据。PyPI SDK/runtime-bin均0.1.5rc1。GitHub目标release assets为空，未声称桌面安装包已验收。
- 悬而未决复查：Q107继续区分旧版未知与新版写租约源码证据；Q117–Q120未发现足以关闭其完整合同问题的证据；本轮关闭0项。不运行无关领域实验。
- 产出：新增累计对比，更新AGENTS默认基线、保留CLAUDE软链接；更新index、README双语、coverage、上游模块说明、identity与errors。本轮无模型/GUI/runtime调用，无用户Session迁移，认知等级最高L2 / verified_inference。
- 验证：固定SHA manifest/接口/历史缺失文件负对照、关键路径与专题锚点存在性、Markdown头部和git diff --check。完整结果见本轮交付；命令为git show/cat-file/diff/rev-list与只读registry/API查询。

## 2026-09-17 · rc.1 non-waking next-turn notice

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d/tag一致；完整回传后沉淀。
- false不主动wake但不禁止活跃driver自动下一turn；claim每次仅一个nextTurn，notice不会自动合并followup。区分入箱/claim/user-message和冷恢复/正常dispose。
- 静态源码核验，未执行notice实验；verified_inference。

## 2026-09-17 · rc.1 ConversationController图片方法与装配

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d；完整答案回传后沉淀。
- 公开class有sendSession/serializeDraftImages不同路径；官方apply硬编码构造，不能从继承直接推出全入口包装。复核RPC source身份正向证据与独立request图政策。
- 静态源码核验，未运行装配/abort实验；verified_inference。

## 2026-09-17 · rc.1 图片预算与入口扩展

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d/tag一致；完整回传后沉淀。
- 区分源admission、normalized图和pi-ai请求三层预算；公开drop slot不覆盖paste，未发现统一async consent hook。清理不等于Host request回滚。
- 静态类型/实现核验，未跑文件故障/浏览器/模型，verified_inference。

## 2026-09-17 · Settings publish/persist 补正

- asked_by: agent；固定rc.1/a66e4702047846cdaa10c66c9d3df3951f5ea70d；完整纠正已回传后沉淀。
- 正式protected publish非publishDocument；区分namespace候选persist、file reconcile先缓存后publish、invalid namespace last-good与跨进程revision范围。
- 正式types与源码核验，未执行子类PoC；verified_inference，错误本E047。

## 2026-09-17 · rc.1 Settings校验与Client staging原语

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d/tag一致，完整答案回传后沉淀。
- resolved-value validator不等于caller授权；Include path/isolate不从递归Client graph排除；prefetch/import/invalidate与Cordis ACTIVE/dispose分层。
- 静态源码公开类型核验，独立staging闭包未实测；verified_inference。

## 2026-09-17 · rc.1 连续事件重投

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d，完整答案回传后沉淀。
- 每次open遍历仍pending的Map，无仅重投一次分支；区分断线delivery归零与有效next结算。工具未完成日志不证明Gateway pending仍存活。
- 静态实现与一方单次replacement测试源码复核，未运行多代重连；verified_inference。

## 2026-09-17 · rc.1 问答 delivery 与 pending identity

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d；完整答案回传后沉淀。
- connected早于后续问答展示；pending按本地key删除、scope binding另有对象guard，不能混同。列明公开observer限制，不用工具日志计数推断Host是否重投。
- 静态源码核验，未复现外部现象；verified_inference。

## 2026-09-17 · rc.1 wrapper 后置结算与 idle

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d/tag一致；完整答案回传后沉淀。
- body完成不等于execute完成，标准driver等待tools/execute wrapper剩余Promise；cancel ACK不等drain，外部/detached调用不自动纳入whenIdle。
- 静态类型/调用链核验，未运行实验，verified_inference。

## 2026-09-17 · rc.1 主动重连与问题 scope 所有权

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d/tag一致；完整答案成功回传后沉淀。
- reconnect撤销delivery而非直接dispose Session scope；新pending key不继承旧draft。manager-owned scope不能以外部fiber.dispose冒充完整drop/rebuild；delegate与纯断线分支分开。
- 静态源码类型核验，未做故障竞态实测；verified_inference。

## 2026-09-17 · rc.1 patch identity 与 Client 注册

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d/tag一致，完整回传成功后沉淀。
- name仅匹配非重命名，无delete patch；ClientBundleRegistration只管factory到达，非Host client-row插入。唯一Controller子类+原Client完整bootstrap闭包未实测，不作不可能性判定。
- 静态源码/manifest核验，verified_inference。

## 2026-09-17 · rc.1 public Controller 继承分派

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d，远端tag一致；完整答案回传成功后沉淀。
- Gateway动态Reflect.get实际service方法，public override/super不需private commands；覆盖面限该Controller入口。核对base/web/standard的独立plan command与系统next-step producer，未将高级插件当默认。
- 源码类型与组合静态核验，未执行继承端到端PoC；verified_inference。

## 2026-09-17 · rc.1 occurrence 取消与 steer authority

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d、tag一致；完整答案回传后沉淀。
- 区分inbox claimed观察、awaited pre-step/request/turn-stopping与无expected-occurrence参数的cancel；不存在所核公开面的统一pre-steer veto。复核问题pending sessionId、本地key与非持久draft。
- 静态源码类型核验，未执行并发/浏览器故障测试；verified_inference。

## 2026-09-17 · rc.1 零 provider 与默认标签

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d，远端tag一致，完整回传成功后沉淀。
- 默认选择与provider registry分离；空catalog仍可显示默认/历史provider-model并blocked，不激活adapter。区分Host catalog读RPC与上游模型请求。
- 静态调用链核验，未跑Client或读取外部项目，verified_inference。

## 2026-09-16 · rc.1 图片输入与 submission 失败

- asked_by: agent；固定a66e4702047846cdaa10c66c9d3df3951f5ea70d，完整答案先回传成功再沉淀。
- 入口signal检查不覆盖Host异步admission，abandon只是echo；图片paste/drop未按inputModalities隐藏，text-only由Host准入拒绝。补图片专题，未执行真实上传/模型/浏览器竞态测试，verified_inference。

## 2026-09-16 · rc.2 FileSystem 全文与窗口

- asked_by: agent；固定b150a551b8d465e31e418e1b2eaf5e79bbb7d28e，tag一致；完整答案已成功回传再沉淀。
- readText/streamText完整正文，tool裁窗口；write.before允许null但字符串不能截断，edit两侧与after保持全文。diff metadata与模型确认文字分开说明。
- 静态公开类型与Provider/consumer源码核验，无文件修改实测；verified_inference。

## 2026-09-16 · rc.1 pi-ai 显式 key 无额外认证读取

- asked_by: agent；完整答案回传成功后写回。DSH固定a66e4702047846cdaa10c66c9d3df3951f5ea70d；正式pi-ai0.84.2 tarball内存sha512与lock一致。
- hand-declared route用harnessApiKeyAuth；显式key让pi-ai跳过CredentialStore和ambient读取；显式ref miss在streamSimple之前失败。仅生成认证路径，不扩成整个进程零环境读取。
- 静态调用链及正式包校验，未执行模型或凭据调用；verified_inference。

## 2026-09-16 · rc.1 credential record 合法形状与 Host 引用消费

- asked_by: agent；完整答案回传成功后沉淀。固定 a66e4702047846cdaa10c66c9d3df3951f5ea70d，远端 tag 一致。
- 明确 ApiKeyRecord 四种 key/env 组合、GrantRecord JSON/owner 约束、分离通知轴，以及 apiKeyEnv/recordKeyFor 的不同键空间；Host 消费不等于全路径不经过 Client。
- 只读官方类型、local provider parser、adapter resolver；未调用模型，verified_inference。

## 2026-09-16 · rc.1 carrier framing 与认证职责

- asked_by: agent；独立来源完整答案先回传成功后沉淀；固定 a66e4702047846cdaa10c66c9d3df3951f5ea70d、远端 tag 一致。
- 补 hooks global 首读、JSON-lines 条件性 framing、wireStream 取消/错误、requestRejection 与 authorizeIndex 职责区别；不存在公开 predicate injection 的等价替换保证。
- 源码/类型静态核验，JSONL/native transport 未实测；verified_inference。

## 2026-09-16 · 两条独立咨询：rc.2 one-shot 与 rc.1 carrier

- asked_by: agent；两条分别完整回传各自来源且确认成功后沉淀，证据按 b150a551b8d465e31e418e1b2eaf5e79bbb7d28e / a66e4702047846cdaa10c66c9d3df3951f5ea70d 隔离。
- rc.2：公开 PiAiAdapter 与无 Session stream；原始草稿用隔离 plugin 解析，不把内部 resolveProfiles 当 root export。SDK 无重试，但 credential await 不受后建 watchdog 硬截止。存在 OpenAI draft listing，不等于生成测试或 reasoning 探测。
- rc.1：hooks 首读前注入、already-authenticated Fetch seam、wireStream 异步形状；WK custom scheme 端到端兼容未知。
- 静态源码/类型核验，未发真实模型请求、未跑完整 native/browser PoC；verified_inference。

## 2026-09-16 · rc.1 submission 返回值与 Client generation

- asked_by: agent；独立来源完整回传成功后沉淀。固定 a66e4702047846cdaa10c66c9d3df3951f5ea70d，远端 tag 一致。
- 复核 beginSubmission 的同步 handle、prompt/cancel 的 RemoteResult、公开 list/current/binding/open；补 generationId 在 apply 内局部递增的源码依据。
- 静态类型/实现核验，测试只定位未运行；verified_inference。没有项目审查或端到端完成断言。

## 2026-09-16 · rc.1 Session 状态与 cancel 可调用性

- asked_by: agent；固定 a66e4702047846cdaa10c66c9d3df3951f5ea70d，远端 tag 一致；完整答案已成功回传后沉淀。
- openState 仅历史状态，prompt/cancel 无 current/open/running 本地门禁；Host cancel 要求 attached Agent，成功仅接受而非 drain。保留 binding 可对非当前仍有效 Session cancel，removed/cold/disconnect 分层。
- 补既有浏览器 Session 专题；源码检查，未运行完整浏览器或网络竞态，verified_inference。

## 2026-09-16 · rc.1 Settings Controller/Remote seam

- asked_by: agent；完整答案先回传来源会话。
- 固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。`api-settings-controller` 仅 root、`./types`、`./typert`、`./remote` exports；Settings Remote 提供 describe/update/replace/mutate，revision 冲突为 `settings/conflict`，无独立 Node/HTTP binding。
- 证据：package.json、README、src/index.ts/types.ts 与 settings-controller 测试源码；未执行端到端浏览器/HTTP，`verified_inference`。

## 2026-09-16 · rc.1 Attachment 与 Session 图片输入 seam

- asked_by: agent；完整答案已先回传来源会话。
- 固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`；核实 AttachmentStore、LocalAttachmentStore、ISession 图片输入，未找到通用 InputRef/InputStore 或 picker/import API。
- 新增固定专题与索引；只读源码/类型检查，未运行端到端浏览器或模型测试，`verified_inference`。

## 2026-09-16 · rc.1 Node-side 公开入口复核

- asked_by: agent；完整答案先回传来源会话并确认成功。
- 固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。公开 Node 侧仍是 CLI/Profile launcher 与分层 Host/Client 模块；未发现排除 SDK/ACP 后的一体式 Node Host adapter、server-side binding 或 `NodeClient.connect()`。
- 证据：apps/cli README、client-connection 与 api-session-controller package exports/client apply、sdk/client launch（SDK 按题设排除）。公开 exports/package tree 只读核验，未 install/boot 实测；`verified_inference`。

## 2026-09-16 · rc.1 Connection 自动认证 record

- asked_by: agent；固定 a66e4702047846cdaa10c66c9d3df3951f5ea70d，远端 rc.1 tag 一致；完整答案先成功回传后写回。
- 官方 Connection 激活必经 modifyRecord(client-connection/browser-session)，仅缺失时写 grant 签名秘密；无保留官方 apply 同时关闭初始化的公开配置。删除 record 只在下一次 activation 更换已缓存 secret。
- 补既有 Connection 专题及索引；读取 apply、BrowserAuth 与一方 activation/invalid-record 测试源码，未执行测试或模型调用，verified_inference。

## 2026-09-16 · rc.1 官方浏览器 Session 与凭据公开面补正

- asked_by: agent；指定 0.1.2-rc.1 / a66e4702047846cdaa10c66c9d3df3951f5ea70d；远端 tag 与本地对象一致，固定 SHA 读取。
- 完整修正答案先成功回传来源，再沉淀。确认浏览器 apply/ctx.sessions、cancel/Host resume、交互 waterfall、Connection owner 与 CredentialProvider 九个读写方法和两类通知。
- 上一答复没有逐符号核验，把缺少一体式 Node 函数扩成分层组合限制；本次撤回该推断并解除凭据 API UNKNOWN。见新专题及 E045。
- 验证为源码、类型与 package exports 交叉检查；未执行完整浏览器/Host 启动或真实模型请求，status 为 verified_inference。

> 按时间倒序记录每一次问答、学习、实测。这份日志同时充当**回归检测的替代品**：`/dsh-sync` 时对"锚点被本次上游变更命中"的历史问答做抽查复验（见 `dsh-sync` skill 步骤 6）。

## 2026-09-09 · rc.1 pending 选择与临时 request override

- **提问者**：agent；固定rc.1完整答案成功回传后沉淀。
- **核验**：tag/SHA、Controller选择与header消费、projection、Agent assembly/request/retry、Session快照、LLM默认标记；正式Controller types内存复核。
- **结果**：pending精确匹配header才消费，之后读最近header；last-selection不是默认永久值；retry重走request但复用assembly；缺effort不等off。
- **沉淀**：补入[模型选择专题](topics/rc1-initial-model-selection.md)，verified_inference；无运行/模型测试或外部项目检查。

## 2026-09-09 · rc.2 History 页与响应大小

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定tag/SHA，读取paginate/cut/types/schema、Client page/continuity、消息组/基线测试；内存核对runtime和host-apiproxy正式types。
- **结果**：缩页是合法新读，非snapshot pinning或字节上限；1条可含大provenance/raw尾部/projections；无history按字节/裁view参数。
- **更正**：上轮网关包名简写不准确，正式名为dsh-host-apiproxy，已回传澄清并修正文档；源码路径不变。
- **沉淀**：[固定专题](topics/rc2-history-page-boundaries.md)，verified_inference；未运行外部transport或检查来源项目。

## 2026-09-09 · rc.2 reasoning 默认与 Session 错误

- **提问者**：agent；独立固定 rc.2 SHA，完整答案成功回传后沉淀。
- **核验**：固定 tag、LLM核心/两个adapter/API/历史和UI路径；内存读取pi-ai0.82.1 formatter，上游lock与来信依赖一致。
- **结果**：标准内置DeepSeek默认high由核心注入；pi缺默认不等off；Anthropic enabled+budget与adaptive+effort分支不同；缺历史presenter不必然open失败。
- **沉淀**：[固定rc.2专题](topics/rc2-reasoning-defaults-session-errors.md)，verified_inference；未访问外部endpoint、未运行模型或检查来源项目。

## 2026-09-09 · rc.1 复用 scope key 与登记 owner

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 tag/SHA，读取 scope primitive/store、Skill register/collect、Cordis child ownership；内存核对正式 dsh-scope types，仅读 empty-layer 测试断言。
- **结果**：无 parent 不改父链；同 key 可见性依赖同 Registry；登记随 minting child fiber，非自动随 key 的原 Agent；dispose 仅撤销自身登记。
- **沉淀**：补入[Skill 专题](topics/rc1-skill-read-authority.md)，verified_inference；无运行实测或来源项目检查。

## 2026-09-09 · rc.1 Skill source 与输入路径

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 tag/SHA，复核 Agent send/claim/record、Skill source 增广、官方 pre-step/UI consumer 与 cold catalog；内存核对 Skill/tool-skill/session 正式 types。
- **结果**：inject 不 wake，不等同即时 user/message；官方两类 Skill 注入均改写 pre-step decision；cold 先选 Host Registry 再解析 standing scope，不传 caller signal。
- **沉淀**：补入[Skill 专题](topics/rc1-skill-read-authority.md)，verified_inference；无模型/应用实测或外部项目检查。

## 2026-09-09 · rc.1 prompt 与 inserted 关联

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.1 tag/SHA，读取 Controller admission、Agent send/wake/claim、Inbox mutation、contained dispatch、request/receipt 与 MessageId 创建；正式 Controller/Agent types 内存核对。
- **结果**：通知同步先于正常 wake/claim；Controller 有 admission await；receipt 不含 id，requestId 存于 source.rpcId；edit 保留 id 但仍发 discarded+inserted，不能把事件当作新 Controller 投递凭证。
- **沉淀**：补入[standing/inbox 专题](topics/rc1-standing-policy-inbox.md)，verified_inference；未运行应用/测试/模型，未检查来源项目。

## 2026-09-09 · rc.1 Web boot 与 required consumer

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.1 tag/SHA，读取 Web boot/page/renderer、模块装配、Cordis/Loader wait、ui-cordis/runner inject 与 shipped row；内存核对三个正式 rc.1 npm 包。
- **结果**：稳定 PENDING 在 settle 后被 activation audit 拒绝；独立插件 DOM 可先存在；run fulfilled 包括失败呈现路径，不是 shell 成功 ACK；consumer 无 capability-off Config。
- **沉淀**：补入[生命周期专题](topics/rc1-loader-client-lifecycle.md)，verified_inference；未运行应用/外部目标，未检查来源项目。

## 2026-09-09 · rc.1 Session 清单与 Preset 代际

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.1 tag/SHA，读取 corpus、JSONL list/format、discovery/standing/mount；内存读取精确 npm Provider 元数据与 Preset types，未安装。
- **结果**：无分页不等于完整原子快照；部分 header 被跳过；定义清单不枚举代际，但公开 livePresetMounts 覆盖仍安装的旧代际。
- **沉淀**：[版本化专题](topics/rc1-session-preset-enumeration.md)，verified_inference；无运行时实测、模型调用或来源项目检查。

## 2026-09-08 · rc.1 与 pi-ai0.84.4 三wire边界

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定DSH rc.1 tag/SHA，读取model/profile/snapshot/图片/replay/discovery/credential；内存核对pi-ai0.84.4三formatter、simple-options和retry源码。
- **结果**：本地目录不必访问/models；wire cap有clamp/minimum/thinking调整；图片/replay存在降级；model middleware不覆盖独立discovery，auth与HTTP headers不能混为单一优先级。
- **沉淀**：[版本化专题](topics/rc1-piai0844-wire-boundaries.md)，verified_inference；无HTTP/模型/项目E2E，未读真实凭据或外部项目。

## 2026-09-08 · rc.1 MCP resync 与关闭

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定rc.1 tag/SHA，读取MCP sync/closure/supervisor/output/transport与tests；内存核对精确plugin exports及SDK1.30 cancellation/stdio/HTTP源码。
- **结果**：fetch失败保旧、swap冲突归零；无catalog freeze/authority review合同；SDK取消和本地close不证明远端物理效果或进程树停止。
- **沉淀**：[固定rc.1专题](topics/rc1-mcp-resync-lifecycle.md)，verified_inference；未运行synthetic transport/E2E，未检查外部项目。

## 2026-09-08 · rc.1 Skill 读取与 invocation 边界

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.1 tag/SHA，读取 roots/watch/scan/get、Registry scope/cache、模型/user consumer 与 cold catalog；内存核对三个精确正式 Skill 包。
- **结果**：isolated 只限本 Provider roots，invocation 不是访问控制；filesystem scan/用户 catalog 的 signal 边界和 cold fallback 阻止无条件一致性推断。
- **沉淀**：[固定 rc.1 专题](topics/rc1-skill-read-authority.md)，verified_inference；未运行来源访问/E2E，未检查外部项目。

## 2026-09-08 · rc.2 todo 与 question 生命周期归属

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.2 tag/SHA，读取 todo tool/projection、UserQuestionService/provider、ApiProxy pending/respond、Client PendingWait 与 loop tool-result。
- **结果**：todo 只记录整表，下一 turn 清 projection；问答 Host 持有 pending，UI respond 不拥有续步，草稿不是持久状态。
- **沉淀**：[固定 rc.2 专题](topics/rc2-todo-question-ownership.md)，verified_inference；未运行 UI/E2E、未检查外部项目。

## 2026-09-08 · rc.1 输入关联、assembly 与重建

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.1 tag/SHA，读取 create/adopt/cwd conflict、updateQueue、assembly 顺序、request capture、Session surface 与 reconstruction 测试。
- **结果**：create canonical 边界不可过推；pre-step 晚于 assembly，assembly waterfall 也晚于同步 providers；日志重建依赖明确 dispatch prefix，不内建产品必要 context/authority 清单。
- **沉淀**：补入[rc.1 输入专题](topics/rc1-input-authority-retry.md)，verified_inference；未运行演示/漏洞/项目测试，未检查外部项目。

## 2026-09-08 · rc.1 Loader 与双侧 Client 生命周期

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.1 tag/SHA，读取 Loader/Fiber 完成、Client graph/HMR、Profile/schema 与 shipped self-modification rows；内存核对 Cordis4.0.2/Loader1.0.3 正式 root declarations。
- **结果**：await 不等 ACTIVE；dispose 等清理但包含错误；effective disabled 可执行 !!js；Client HMR 忽略 graph frame，Host 删除不证明 Client 卸载。结构唯一不等保护行/职责政策。
- **沉淀**：[固定 rc.1 专题](topics/rc1-loader-client-lifecycle.md)，verified_inference；未运行 fixture/双侧 E2E，未检查外部项目。

## 2026-09-08 · rc.2 agent/created 与 schema scope

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.2 tag/SHA，复核 restrictableNames/own view、AgentRegistry.announce、factory publication、schemas(scope) 与 scope teardown 测试。
- **结果**：own-only filter 名称拒绝，同名 inherited 例外；created 同步 throw 可 veto，Promise rejection 不等待；schemas 必须显式传 Agent 才是该视图。
- **沉淀**：补入[rc.2 生命周期专题](topics/rc2-batch-cancel-value-validation.md)，verified_inference；未运行调用方测试或修改其项目。

## 2026-09-08 · rc.1 附件文件身份与 request-cache

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.1 tag/SHA，读取 normalized object 与 request-cache 的读写、身份字段/链接检查缺口、公开 byte helpers 与相关测试；重新内存核对正式 local root exports。
- **结果**：digest/metadata 不证明 namespace/uid/mode/nlink；cache descriptor hash 不是 cache bytes digest；公开 prepare/projection 有输入前置，未提供 fd-based 既有 ref verifier。
- **沉淀**：复用[rc.1 附件专题](topics/rc1-attachment-publication-cancellation.md)，新增分节与锚点，verified_inference；未运行路径攻击/权限竞态或 E2E，未检查外部项目。

## 2026-09-08 · rc.2 文件沙箱与执行环境边界

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.2 tag/SHA，检查 sandbox Definition/policy/local profiles、官方工具 Config、subprocess spec/本地终止与平台测试；内存核对七个精确正式包 exports/declarations。
- **结果**：file-effect full 不代表网络/读取/进程全隔离；普通工具无内置改名/provider selector，独立 ToolDefinition 是公开能力；subprocess 无自动 sandboxPolicy，远端/容器属于另一 execution world。
- **沉淀**：[固定 rc.2 专题](topics/rc2-sandbox-execution-worlds.md)，verified_inference；未执行隔离/取消/跨环境 E2E，未检查外部项目。

## 2026-09-08 · rc.1 图片保存取消与发布清理

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.1 tag/SHA，检查 Controller/attachment admission、local 准备/提交、fsync/cleanup/limiter 与测试；内存核对两个精确正式 tarball 的 exports/declarations/JS。
- **结果**：保存链无调用 signal，准备全成功再逐张发布，错误不回滚已发布对象；temp cleanup、单对象 durability、batch/prompt acceptance 是不同边界。公开 Provider/helpers 不自动增加请求级取消或回收合同。
- **沉淀**：[固定 rc.1 专题](topics/rc1-attachment-publication-cancellation.md)，verified_inference；无故障注入/crash/取消 E2E，未检查外部项目。

## 2026-09-08 · rc.2 运行时职责与外部能力接缝复核

- **提问者**：agent；完整职责级答案成功回传后记录。
- **核验**：远端 rc.2 tag 与 `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e` 一致，按 SHA 读取 AgentLoop/Session/Workspace/projection/ToolRuntime/approval，以及 ShellExecutor/FileSystem/SessionPersistence/Storage/ClientTransportHooks、Skill/MCP 注册路径。
- **结果**：运行时职责由 DSH 插件提供，外部能力可通过公开 provider/backend/carrier 合同接入；Skill 与 MCP tools 继续在官方 Registry/ToolRuntime 下组合，不需复制 AgentLoop。Session persistence 与 non-session storage 分层。
- **沉淀**：仅复核日志，复用已有包组与集成路由；verified_inference。未运行宿主组合 E2E、未检查外部项目。

## 2026-09-08 · rc.1 RPC 一致性与 prompt 取消边界

- **提问者**：agent；完整回传成功后沉淀。
- **核验**：固定 rc.1 tag/SHA，检查 shared rpcFetchHandler、Controller 入口、commands await、image admission chain、附件 helper 与 mismatch 测试。
- **结果**：路径与 envelope method 必须一致；prompt 只入口查 signal，后续 abort 不保证不入箱；requestId 重复不去重。
- **沉淀**：复用[rc.1 输入专题](topics/rc1-input-authority-retry.md)，新增分节与锚点，verified_inference；未运行竞态/E2E，未检查外部项目。

## 2026-09-08 · rc.2 Skill 资源与 parser

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.2 tag/SHA，读取 ResourceBase/renderer/provider/tool、filesystem parser 与测试；内存检查三个精确正式 tarball exports/declarations/JS。
- **结果**：resourceBase 不执行资源操作；未知 license/compatibility/allowed-tools 不产生能力；description parser 无上限、目录另截断；无独立公开文本 parser 或完整文件树 manifest。
- **沉淀**：[固定 rc.2 专题](topics/rc2-skill-resources-parser.md)，verified_inference；未运行资源部署/脚本/跨机器 E2E，未检查外部项目。

## 2026-09-08 · rc.2 标题异步与 prompt 投影

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：固定 rc.2 tag/SHA，读取 title 调度/fallback、DeepSeek/pi-ai system 映射、SessionFace/Host prompt 与 Client mux/status/Notifier；按 tag lock 只读内存核对 pi-ai 0.82.1 formatter。
- **结果**：automatic 标题不构成 admission/publication await gate；fallback 来自首条 human text；prompt 回 acceptance，消息/running/title 各经自身路径可见。
- **沉淀**：[固定 rc.2 专题](topics/rc2-title-system-prompt-publication.md)，verified_inference；未运行模型、标题生成或 Host-Client E2E，未检查外部项目。

## 2026-09-08 · rc.2 runMaintenance 与局部工具动态贡献

- **提问者**：agent；限定补充，完整回传成功后沉淀。
- **核验**：固定 rc.2 tag/SHA，回源确认维护 phase、cancel/wake、tools inherited/own view 与 disposer；只读内存核对正式 Agent runtime-types 声明。
- **结果**：runMaintenance 取得 true-idle 执行权，status 仍 idle；动态 register/restrict 可用但取消不回滚，own 注册豁免 inherited filter，维护不锁所有 ToolRuntime 调用。
- **沉淀**：复用[rc.2 生命周期专题](topics/rc2-batch-cancel-value-validation.md)，新增证据及分节，verified_inference；未运行组合实测、未检查外部项目。

## 2026-09-08 · rc.2 Preset 选择的 idle/blank 边界

- **提问者**：agent；独立咨询，完整答案成功回传后沉淀。
- **核验**：固定 rc.2 tag/SHA，检查 ApiProxy select、sessionBlank、standing recompose、owned create/resume/dispose、effective preset fold 与直接测试；内存读取 agent-presets/host-apiproxy 精确正式 tarball。
- **结果**：已有 turn 的 idle Session 仍被 select 锁定；底层 recompose 有 caller-owned blank 前置；owned 生命周期原语不构成历史 Preset 迁移事务。
- **沉淀**：复用[rc.2 Session/Preset 专题](topics/rc2-session-binding-create-preset.md)新增分节，verified_inference；未实测迁移/在途调用，未检查外部项目。

## 2026-09-08 · rc.2 oneOf 与附加输入校验

- **提问者**：agent；完整答案成功回传后沉淀。
- **核验**：远端 rc.2 tag 与 `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e` 一致；固定源码检查 exact-one、integer、defineTool、pre/ask/guard/body 顺序与官方测试断言。
- **结果**：互斥 string/null 与 boolean/null 转换仅在值域上等价；guard 晚于审批；body 可附加 minimum，但不能宣称删去 keyword 后 schema 原义完整保留。
- **沉淀**：复用[rc.2 请求与 schema 专题](topics/rc2-agent-request-attempt-schema.md)，新增本次分节及证据，verified_inference；未运行外部 MCP/转换器/实际效果。

## 2026-09-08 · rc.1 多 provider 与 BYOK 接缝

- **提问者**：agent；以消息信封来源回传完整答案，发送成功后沉淀。
- **核验**：rc.1 tag 固定 `a66e4702047846cdaa10c66c9d3df3951f5ea70d`；回源检查 LlmAdapter、目录、pi-ai protocol/profile/catalog、CredentialProvider；内存只读检查 pi-ai/credentials 精确正式 tarball 的 exports/declarations。
- **结果**：同 Host 可有多 route，手工 BYOK 三协议与自定义 endpoint/model 公开；独立 Cloud adapter 不需替换 AgentLoop/Session，但账号/账本/计费幂等不由这些接口提供。
- **沉淀**：[固定 rc.1 专题](topics/rc1-multi-provider-byok.md)，verified_inference / L2；未安装、未模型调用或 E2E，未检查外部项目。

## 2026-09-08 · rc.1 初始模型选择与 Preset 分离

- **提问者**：agent；独立问题，完整答案成功回传后沉淀。
- **核验**：rc.1 tag 固定 `a66e4702047846cdaa10c66c9d3df3951f5ea70d`，回源检查 default-model、Controller selection、AgentOptions、Preset swap 和测试；只读内存检查 default-model 精确正式 tarball 声明。
- **结果**：default-model config 只有 provider/model；Controller.selectModel 不要求 blank、不调用 recompose，立即写 model/selection 并尝试保存默认。cold resume 装配与 model 切换分开；旧版仅内存选择结论不可套用。
- **沉淀**：[固定 rc.1 专题](topics/rc1-initial-model-selection.md)，verified_inference / L2。未运行模型或 E2E，未检查调用方项目。

## 2026-09-08 · rc.1 程序工具的 open-turn 生命周期

- **提问者**：agent；独立咨询，完整答案回传成功后沉淀。
- **问题**：输入 ID 到 turn、turn-stopping、plugin-source append/flush、direct tools.execute 与审批/日志 owner。
- **核验**：远端 rc.1 tag 与 `a66e4702047846cdaa10c66c9d3df3951f5ea70d` 一致，固定 SHA 读取 API/loop/session/tools/approval/UI 与测试；Python urllib/tarfile 在内存读取七个精确正式包 exports/root declarations。初次使用不正确的 `dsh-session-controller` 名称返回 404，改按 manifest 的 `dsh-api-session-controller` 后成功。
- **结果**：claimed 提供 message→turn 关联；stopping 在 open turn 内 awaited，但不是完成回执；append 不调度；flush 返回 listener 参与情况；程序工具不自动拥有模型工具日志。
- **验证边界**：无运行实测、无模型调用、无外部项目检查；组合 UI/取消/效果/恢复未验证。
- **沉淀**：[固定 rc.1 专题](topics/rc1-programmatic-tool-turn-lifecycle.md)，`verified_inference` / L2。

## 2026-09-08 · rc.1 Connection Host 替换与 Client 模块身份

- **提问者**：agent；完整答案先回传来源任务，跨任务工具返回成功后才沉淀。
- **问题**：唯一自有 HostConnectionHandle 能否保留官方 Client 完整入图，并复用公开 codec/auth 接缝。
- **核验**：`git ls-remote` 核对 `dsh-v0.1.2-rc.1` 为 `a66e4702047846cdaa10c66c9d3df3951f5ea70d`；按 SHA 读取 Profile/Loader/Client registry/Connection 实现与测试。Python urllib + tarfile 在内存读取两个精确 rc.1 tarball 的 declarations/JS exports，未安装、未写上游。
- **结果**：Handle 与 ClientTransportHooks 公开，但 registry 无 client-only contribution，默认 Host service 依赖未公开导出的 BrowserAuth；未找到满足全部约束的官方完整组合。bootstrap 原语只作为未验证候选，不推出不可能性。
- **实测**：未运行 Host/Client；仅静态与发布产物核验，无模型调用。
- **沉淀**：[固定 rc.1 专题](topics/rc1-connection-client-module-boundaries.md)，`verified_inference` / L2；未修改既有 fact。

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

## 2026-09-05 · agent咨询：Tool/LLM retry identity与Session依赖重建边界

- **提问者**：agent（外部评审语境不入库）
- **问题**：native tool live/durable/approval identity、cancel/drain边界，model-step retry入口，以及标准Session log能否重建全部model-visible内容与runtime依赖
- **结论**：ToolRuntime保证同live execution与frozen args，但Approval无argument-bound grant、cancel/whenIdle/result/SIGTERM均不是通用跨进程receipt；`maxRetries:0`只约束`llm-retry`自身。Session log可重建历史DSH-level request的messages/system/tools/config值，却不保存executor/plugin graph、未调用Skill definition或外部provider状态，effective Preset ID也不是依赖清单。
- **依据锚点**：`packages/core/tools/src/index.ts`、`packages/core/agent-loop/src/{agent,tool-calls}.ts`、`packages/interaction/user-approval/src/index.ts`、`packages/llm/{llm,llm-retry}/src/index.ts`、`packages/compaction/compaction-basic/src/index.ts`、`packages/core/session/src/{types,request-header,index}.ts`、`packages/skill/tool-skill/src/index.ts`、`packages/preset/agent-presets/src/session.ts`、`apps/cli/src/{profile-boot,process-shutdown}.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2；远端tag同SHA）
- **沉淀**：`packages/core.md`新增“标准Session request snapshot与runtime依赖不是同一件事”；`index.md`摘要同步；既有Approval/retry/Preset条目未重复
- **实测**：无（固定tag public types、控制流与一方tests足以判定；调用方亦限定不跑conformance）
- **覆盖度变化**：无（Core仍L1）

## 2026-09-05 · agent咨询：blank Session create、持久化与处置边界

- **提问者**：agent（外部项目细节不入库）
- **问题**：Host `session.create`成功时live Session/header、Session persistence与Workspace membership分别到达什么边界；zero-event blank Session是否有public abandon/delete或换preset的幂等create语义
- **结论**：create ok已发布idle Agent、live Session与immutable header，但不等待PersistenceCoordinator的`session/flush`；backend可lazy到first append才物化。带`workspaceId`的attach另行等待domain durability但与Session artifact不原子。rc.2无Session delete/abandon；detach/dispose只收口live lifecycle，不证明删除artifact或Workspace account；同id不同explicit preset返回`agent-preset-conflict`。
- **依据锚点**：`packages/host/apiproxy/src/{api/sessions.ts,api-proxy.ts}`、`packages/core/{agent/src/index.ts,agent-loop/src/index.ts,session/src/index.ts}`、`packages/session/session-persistence/src/{index,coordinator}.ts`、`packages/workspace/workspace/src/entity.ts`、`packages/storage/storage-domain/src/domain.ts`、`packages/host/apiproxy/tests/api-proxy-agent-preset.spec.ts`、`packages/session/session-persistence/tests/{contract,coordinator-contract}.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2；远端tag同SHA）
- **沉淀**：`packages/host.md`补充create/persistence/workspace三边界与detach/dispose限制；`index.md`摘要同步
- **实测**：无（固定tag源码、公开类型与一方tests足以判定）
- **覆盖度变化**：无（Host仍L2）

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

## 2026-08-31 · 跨会话复核：Skill Loader闭包与batch teardown

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：selected-only immutable Skill root的正式Config/三包Loader rows，以及注销ToolDefinition、关闭carrier与同batch pending/started calls的安全关系。
- **结论**：Web shipped组合保留Host`dsh-skill`，把filesystem provider与tool-skill consumer移入preset；`includeDefaultRoots:false/customSkillDirs/watch:false`只隔离该provider，且custom root可能经scope-visible`ctx.fs`读取。仅等started不够，scheduler会补pending；先cancel Agent后，started drain、pending写`ABORTED_BEFORE_DISPATCH`，再注销/关carrier才闭环。正式d.ts把providerName默认错写`local`，runtime实际`filesystem`。
- **版本证据**：rc.2 `b150a551`正式skill/skill-filesystem/tool-skill/tools/agent/system-prompt tarball integrity、runtime JS、package manifests、base/web/preset rows、AgentLoop/ToolRuntime与一方tests。
- **实测**：无；未读取外部Skill root、未关闭真实carrier、未使用凭据。固定控制流与tests足以判定，Host-wide shutdown仍无一方e2e。
- **沉淀**：`packages/{skill,core}.md`、conflicts C081、index/coverage/log。

## 2026-08-31 · 跨会话核验：retry request fence与Skill byte TOCTOU

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：同step request-error retry是否重跑scope-local`agent/request`，official filesystem locator能否把digest check与get bytes绑定，以及strict validator与宽松parser是否扩权。
- **结论**：每次loop-level retry都重新buildRequest并重跑Agent-scoped request waterfall，但复用原assembly/tools；middleware failure/cancel在adapter前收口。filesystem locator只有path/directory，list/get独立open，无digest/fd/authenticated byte，因此外部hash gate仍有DSH未封闭的TOCTOU。same bytes下unknown keys不扩官方字段，但validator必须对齐invocation defaults/coercions、metadata与resourceBase等派生语义。
- **版本证据**：rc.2 `b150a551`正式skill/skill-filesystem/tool-skill/agent/system-prompt tarball、runtime JS、Agent dispatcher/loop与request-error/reconstruction/parser/consumer tests。
- **实测**：无；未替换真实文件、未调用模型、未使用凭据。控制流足以判定；外部store不可变性未知。
- **沉淀**：`packages/{core,skill}.md`、errors E030、open Q120、index/coverage/log。

## 2026-08-31 · 跨会话核验：filesystem Config与provider工具名边界

- **提问者**：agent（multi-agent v2子任务；直发被宿主拒绝，完整答案relay到其父任务并确认成功）
- **问题**：rc.2 `dsh-skill-filesystem`根Config的精确字段与物理读取owner；ToolDefinition name经`dsh-tools`/`dsh-llm-pi-ai`是否原样进入provider、DSH是否限制字符或长度。
- **结论**：题列`includeDefaultRoots/customSkillDirs/watch`均为正式Config；普通root优先scope-visible`ctx.fs`、缺席才Node fallback，bundled trustedHost与watcher走Node。ToolRuntime和pi-ai Context原样投影name，核心无通用regex/max-length，只管同层唯一、`run_code`保留与最终execution非空；最终provider/auth compat仍可转换或拒绝。
- **版本证据**：rc.2 `b150a551`正式skill-filesystem/tools/llm/llm-pi-ai tarball integrity、root `.d.ts`、runtime JS、tag源码与一方tests；pi-ai 0.82.1正式formatter tarball。
- **实测**：无模型/endpoint调用；未读取外部Skill目录、未使用凭据。固定发布物和控制流足以静态判定，exact provider name限制保持unknown。
- **沉淀**：`packages/{core,llm}.md`、errors E031、index/coverage/log；Skill页已有Config/读取owner结论，无重复改写。

## 2026-08-31 · 跨会话核验：cold presenter mount、bundled root与tool budget

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：cold history是否在无Agent时激活recorded preset；bundled/custom Skill root的物理读取差异；MCP/LLM name与tool-count限制；model switch/retry是否重验工具集。
- **结论**：history会在首读/变更时真实mount standing rows且不发`agent/created`，失败才退global presenter。显式bundled root强制Node并可与isolated custom并存，但不提供digest/byte lease。MCP内部qualified name限制64并hash，通用ToolRuntime/LLM无统一name/count cap；model switch只验route/model/effort，同step retry复用原assembly/tools。
- **版本证据**：rc.2 `b150a551`正式host-apiproxy/agent-presets/skill-filesystem/mcp-client/system-prompt/agent-loop/llm-adapter tarball shasum、runtime JS、root `.d.ts`、tag控制流与一方tests；pi-ai 0.82.1 formatter tarball。
- **实测**：无；未加载恶意preset、未读取外部Skill root、未调用模型/provider。固定发布物、控制流与tests足以静态判定；外部provider工具限制保持unknown。
- **沉淀**：`packages/{preset,skill,mcp,core,llm}.md`、errors E032、index/coverage/log。

## 2026-08-31 · 跨会话核验：运行期preset投影与same-id generation

- **提问者**：agent（跨会话；完整答案先回传并确认成功）
- **问题**：API ready后在configured root新增preset能否直接被create/standing发现；cold history的unknown/mount fail是否必定退global；same-id编辑与content-addressed immutable责任。
- **结论**：roots集合构造时固定，内容每次list/resolve重扫且无negative cache；先发布目录再create无需重建Service，first-root-wins与外部publication ordering仍适用。presenter的unknown/mount错误全被捕获，其他history source/projection错误仍可整体失败。same-id按mtimeMs+size换generation，仅live joined Agent保留old；cold/restart只存id，因此immutable id是exact-restore外部不变量而非DSH运行要求。
- **版本证据**：rc.2 `b150a551`正式agent-presets/host-apiproxy tarball、public `.d.ts`、runtime JS、discovery/index/api-proxy控制流与authoring/mount/presenter一方tests。
- **实测**：无跨进程rename e2e；未加载preset或创建Session。固定发布物与一方tests足以静态判定，外部文件系统原子性保持unknown。
- **沉淀**：`packages/preset.md`、index/coverage/log；E032已有cold presenter陷阱，无新增error条目。

## 2026-08-31 · 跨会话核验：cold mount的inventory与staging边界

- **提问者**：agent（两项关联跨会话；第一项直达，第二项multi-agent v2直发被拒后完整relay到父任务，均确认成功）
- **问题**：不可发现staging原子进入preset root后何时可见；cold history是否绕过外部inventory；normal create mount rollback；generic Loader hook能否在module import前veto。
- **结论**：scanRoot只认configured root直接valid-id子目录，内容逐次重扫；DSH无staging/no-replace rename seam。cold history独立standing mount，不走Agent/publication inventory。normal create在unpublished setup内mount，失败不发布。Loader `entry-init`过早无options、`patch-context`晚于import，均不能形成preset/digest pre-import gate；失败dispose只回收受Cordis effect拥有的资源。
- **版本证据**：rc.2 `b150a551`正式agent-presets/host-apiproxy/agent-loop tarball与public types；preset discovery/mount、ApiProxy presenter/create、AgentLoop setupAndPublish、vendor Loader Entry顺序及一方tests。
- **实测**：无；未rename目录、未加载恶意module、未创建Session。固定控制流与tests足以静态判定；外部文件系统原子性保持unknown。
- **沉淀**：`packages/preset.md`、errors E033、index/coverage/log。

## 2026-08-31 · 跨会话核验：Session identity、Preset恢复与fork矩阵

- **提问者**：agent（multi-agent来源与显式父任务均已先收到完整答案并确认成功）
- **问题**：rc.2 caller-supplied SessionId的create收敛/冲突与丢响应对账；exact preset create/resume/history/fork；preset-less legacy/default与missing/broken矩阵。
- **结论**：SessionId是create收敛键，rpcId不是；同cwd/same exact preset可single-flight/adopt/resume，省略preset沿用，explicit冲突或cwd冲突fail closed。正常create/resume在unpublished setup mount后才publish；cold history会尝试standing presenter并对其失败退global。ordinary fork继承recorded effective preset id并重新解析current roster，不继承live generation；legacy无id只在当前roster存在时采用current default，missing/broken recorded id不回退default。
- **版本证据**：rc.2 `b150a551`正式agent-presets/agent/agent-loop/session/host-apiproxy/client-runtime tarball shasum、root/API/client `.d.ts`、runtime JS与ApiProxy/Preset/AgentLoop一方tests。
- **实测**：无新runtime fixture；固定正式发布物、控制流与一方tests足以静态判定。preset-missing fork的exact wire错误外形缺少专项test，保持verified_inference。
- **沉淀**：`packages/{host,preset}.md`、index/log；既有cold mount与definition-store结论不重复。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：Approval continuation、batch cancel与Agent teardown

- **提问者**：agent（multi-agent来源与显式父任务均已先收到完整答案并确认成功）
- **问题**：ctx.tools.register/pre-execute ask/ApprovalService与ToolRuntime continuation；cap=1下started+pending cancel；whenIdle/AgentHandle.dispose/scoped-global registration与root teardown边界。
- **结论**：正常Approval request的asked/decided同id配对，allowed-once延续同一frozen-arguments ToolExecution，但body前仍按name重解析definition。cancel停止pool补位、drain started、为pending补ABORTED_BEFORE_DISPATCH并以aborted turn收口；whenIdle等whole-agent quiescence，handle.dispose再dispose scope并detach Agent/Session。global tool与physical carrier不由单Agent自动drain；外部admission与carrier close仍属其owner策略。
- **版本证据**：rc.2 `b150a551`正式tools/user-approval/agent/agent-loop/session tarball shasum、root `.d.ts`、runtime JS与tools/approval/tool-calls/cancel/scope-lifecycle一方tests。
- **实测**：无新fixture；cap=1串行与cap=2 abort各有一方test，exact cap=1 cancel组合由相同scheduler控制流交叉推出。发现`AgentHandle`根`.d.ts`注释与runtime teardown顺序冲突。
- **沉淀**：`packages/{core,interaction}.md`、errors E034/E035、index/log。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：cold models真实resume与history fallback可观测性

- **提问者**：agent（multi-agent v2直发被拒后，解析parent_thread_id并完整relay确认成功）
- **问题**：cold`session.models`是否触发recorded preset真实resume/mount及零publication rollback；history成功无Agent能否证明global generic presenter。
- **结论**：models共用`agentFor`并对ordinary cold Session调用`ctx.agents.resume`；missing id在setup构造前失败，broken/runtime mount在unpublished resume setup内失败，两者都不发布。history只在optional`events[].view`暴露presenter结果，没有fallback marker；成功无Agent不足以证明global lookup，需global独特presenter产生正向view。
- **版本证据**：rc.2`b150a551`正式host-apiproxy/api-remotes/agent-loop/agent-presets API/runtime与cold/agent-lookup/resume/mount/view一方tests。
- **实测**：无新fixture；exact Host models+真实broken preset组合无单条一方test，由相邻正式控制流与分层tests交叉确认。
- **沉淀**：`packages/{host,preset}.md`、errors E036、index/log。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：独立Context/process registry隔离

- **提问者**：agent（multi-agent v2子任务，完整答案relay父任务并确认成功）
- **问题**：Preset/Agent/Session/Tools/Approval/Loop是否因同仓未import源文件或两个独立gate顺序而自动共享/激活。
- **结论**：产品registry与activation由各Context/Fiber Service实例拥有，未import/未入Loader图/未注册文件保持惰性；但AgentPresets mount Set/WeakMaps与Session attachment WeakMap是module-level object-identity bookkeeping，不能绝对宣称零process-global表。独立进程/完整teardown且无共享外部介质时顺序无DSH语义。
- **版本证据**：rc.2`b150a551`正式agent-presets/agent/session/tools/approval/agent-loop root runtime与mount/scoped/session/lifecycle一方tests。
- **实测**：无多进程fixture；Node/pnpm是否真正独立、是否间接import及外部root/backend共享保持调用方前提。
- **沉淀**：`packages/{core,preset}.md`、index/log。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：ToolDefinition引用与Cordis sibling rollback

- **提问者**：agent（multi-agent v2子任务，完整答案relay父任务并确认成功）
- **问题**：tools.register是否clone/freeze schema；Cordis provide/effect返回、失败与teardown；tools root公共类型闭包。
- **结论**：register保存caller原definition引用，schemas只在读取时deep snapshot parameters；caller nested mutation可影响后续projection/output validation。Cordis provide是独立owned effect，后一个sibling effect同步失败不回滚它；同一composite return/yield或整plugin startup失败才共同rollback。tools根`.d.ts`已完整导出题列类型/augmentation，无需src。
- **版本证据**：DSH rc.2`b150a551`正式tools tarball/runtime/tests；`@deepseek-ai/cordis@4.0.1` shasum`e8171e63...`、vendor reflect/fiber与API docs。
- **实测**：无新fixture；caller-mutation与sibling sequence缺专门一方test，固定runtime控制流可静态判定。
- **沉淀**：`packages/core.md`、`topics/cordis-primer.md`、errors E037/E038、index/log。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：lossless snapshot、deepFreeze与composite ownership

- **提问者**：agent（完整答案先回传来源与显式父任务，均确认成功）
- **问题**：Session snapshot拒绝/getter语义、LLM deepFreeze覆盖、ToolDefinition借用，以及Cordis service/token/transport零残留安装边界。
- **结论**：snapshot生成detached JSON图并用undefined表示普通非法值，但getter只读一次且throw传播；deepFreeze只沿Object.keys并跳过AbortSignal，在snapshot图上才覆盖全部nested containers。register仍借caller definition；Cordis generator composite仅回滚已yield disposer，未yield Map/transport/provide不会被猜测清理。
- **版本证据**：rc.2`b150a551`正式session/llm/tools tarball与json/call-config/tools一方tests；Cordis4.0.1 reflect/fiber runtime和API docs。
- **实测**：无新fixture；getter与deepFreeze有直接一方tests，exact transport composite由固定runtime控制流确认。
- **沉淀**：`packages/{session,llm,core}.md`、`topics/cordis-primer.md`、errors E038-E040、index/log。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：cold preset三分支与raw export独立性

- **提问者**：agent（完整答案先回传来源与显式父任务，均确认成功）
- **问题**：missing、discovery-broken、Loader mount-failed在cold models中的精确失败点；history global marker；history/raw artifact export是否被preset错误阻断。
- **结论**：missing在Host setup构造时resolve失败，resume未调用；broken row先resolve、后在unpublished resume mount拒绝；Loader failure同属更深setup rollback。三类history均catch standing失败并可由global marker `HistoryEntry.view`正向证明；Host export完全绕过preset并readRaw后打ZIP，preset错误自身不阻断。
- **版本证据**：rc.2`b150a551` Host ApiProxy/API Remote resolver/AgentPresets/AgentLoop/session-export/JSONL runtime与分层一方tests。
- **实测**：无新fixture；无单条上游test覆盖完整三分支矩阵，组合结论由正式控制流交叉核验。
- **沉淀**：`packages/host.md`、index/log；既有history可观测性错误E036无需重复。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：exact-preset external create与client-runtime收敛

- **提问者**：agent（完整答案先回传来源任务并确认成功）
- **问题**：first-party client以预分配SessionId+exact preset走Host create后，如何通过正式client-runtime list/binding/open收敛；preset各层可见性；standing预挂载与同步publication guard。
- **结论**：Host create有agentPreset而SessionRuntime.create没有，且无public adoptCreateResult。标准host/session-added会eventually upsert；concrete refresh后仍需等待list observable的microtask投影，id出现后binding才可同步解析。noteAgentPreset是public但只属已有blank switch。预挂载same-stamp standing供create join；同步agent/created throw最终rollback registries，但晚于mount/session-created且不回收standing。
- **版本证据**：rc.2`b150a551`正式host-apiproxy/client-connection/client-runtime/agent-presets/agent/agent-loop/session-query exports、runtime与sessions-service/manager/mount/agent/scope-lifecycle一方tests。
- **实测**：无真实Connection end-to-end；Host frame、refresh projection、binding与guard由分层tests交叉确认。
- **沉淀**：`packages/{client,host,preset}.md`、errors E041、index/coverage/log。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：standing identity、publication veto与list absence

- **提问者**：agent（完整答案先回传来源任务并确认成功）
- **问题**：pre-mounted ScopeKey与Agent standing identity；同步agent/created veto对live/persistence；cold list agentPreset；response loss后list absence能否证明未commit。
- **结论**：same generation key严格对象同一；sync listener最终rollback live registries但session/created已先发，first-party fresh/no-event lazy路径可不留artifact，却无通用磁盘零痕迹保证。attached/cold list公开agentPreset且不resume；一次absence只证current不可见，rc.2无commit-status，禁止重放时保持unknown。
- **版本证据**：rc.2`b150a551` AgentPresets/Scope/Agent/AgentLoop/Host list/Persistence coordinator runtime与mount/agent/scope-lifecycle/coordinator一方tests。
- **实测**：无调用方fixture；exact provider/early event未知。
- **沉淀**：`packages/{client,host,preset,session}.md`、errors E042、index/coverage/log。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：refresh缺席分类与history existence

- **提问者**：agent（multi-agent v2直发被拒，完整relay父任务确认成功）
- **问题**：create响应丢失后refresh ready+list absence能否判not-found；history能否提供exact只读commit-status。
- **结论**：phase ready首次成功后sticky，refresh error被fold私有且Promise仍resolve；成功baseline absence也只是current absence。history区分current session-not-found与internal storage failure，却不校验expected preset或历史occurrence。exact row+exact preset是正证，absence仍unknown。
- **版本证据**：rc.2 client SessionRuntime/Manager/Notifier、Host list/history与Persistence public contracts/tests。
- **实测**：无transport；不知道调用方如何证明本轮refresh成功。
- **沉淀**：`packages/{client,host}.md`、errors E042、index/log。
- **覆盖度变化**：无。

## 2026-08-31 · 跨会话核验：projection coldSnapshot不加载preset

- **提问者**：agent（multi-agent v2子任务，完整relay父任务确认成功）
- **问题**：projection-cache warm-up是否经presenter/standing挂载recorded preset；agent/created guard对create/resume/prompt resume的rollback。
- **结论**：coldSnapshot只readFrom→projection restore→write-back，不创建Agent/Session、不发agent/created、不读agentPreset。若另行standingKeyFor才执行preset code。sync agent guard对new publication最终live rollback；cold prompt resume在投递前失败，live prompt不重发created；persistence仍受session-created先行的独立边界。
- **版本证据**：rc.2 session-projection-cache/projection runtime/tests与Agent/AgentLoop/API Remote publication链。
- **实测**：无warm-up；projection unit自身init/apply/view仍会执行。
- **沉淀**：`packages/{session,preset}.md`、errors E043、index/coverage/log。
- **覆盖度变化**：无。

## 2026-09-02 · 上游同步：alpha.3 → alpha.4

- **提问者**：agent（固定rc.2 teardown核验触发新鲜度自检）
- **同步**：DSH `dd6322d6` → `4e84901e`（297提交、2371文件）；Cordis `e09e7521` → `00278924`（1文件）。
- **发布状态**：npm根包`alpha=0.1.2-alpha.4`、`latest=next=0.1.1-rc.2`；PyPI SDK=`0.1.2a3`；alpha.4 tag含242个非private package标识，未穷尽companion tarball/install/native闭包。
- **防腐**：114个既有锚点命中；将`rc2-full-web-native-shell`、`webhook`与alpha.1→alpha.2版本页从fresh标为stale，其他命中内容页已是stale。
- **breaking信号**：alpha.3移除SQLite Session persistence；alpha.4区分event seq/log offset brands，code-runtime-python移入experimental。
- **沉淀**：新增`topics/版本变更-0.1.2-alpha.2-到-0.1.2-alpha.4.md`，更新CLAUDE/README/index/coverage基线。
- **未完成**：未逐页复验114个命中锚点；不得用alpha.4反推固定rc.2。
- **覆盖度变化**：版本变更L1新增1页；其他无。

## 2026-09-02 · 上游同步：alpha.4 → alpha.5

- **提问者**：agent（固定rc.2 AgentPreset scope核验触发新鲜度自检）
- **同步**：DSH `4e84901e` → `49a606bc`（44提交、718文件）；Cordis仍为`00278924`。
- **发布状态**：GitHub/npm alpha.5 tag=`db6bdc35`，master HEAD=`49a606bc`，两者不同；npm `latest/next`仍为rc.2，PyPI SDK仍为`0.1.2a3`。
- **防腐**：83个既有锚点命中；将alpha.2→alpha.4版本页标stale，其他命中内容页早已stale。
- **breaking信号**：Session persistence转为handle-based公共面与lifecycle-owned write path。
- **沉淀**：新增`topics/版本变更-0.1.2-alpha.4-到-0.1.2-alpha.5.md`，更新CLAUDE/README/index/coverage基线。
- **未完成**：未逐页复验83个命中锚点，未实测完整发布闭包。
- **覆盖度变化**：版本变更L1新增1页；其他无。

## 2026-09-04 · 跨会话核验：Loader同module多Fiber的Context与cleanup

- **提问者**：agent（完整答案先回传来源任务并确认成功）
- **问题**：同一AgentPreset以不同id/config重复挂载同一module时，apply Context、scope layer、inject、listener、Effect清理及`WeakSet<Context>`的精确语义。
- **结论**：不同row建立不同Entry/Fiber/apply Context，但继承同一preset ScopeKey；相同callback共享Runtime而不共享Fiber/config/Effects。不同tool name写入同一scope layer并可共存，entry级dispose只清本Fiber；registry.delete/HMR会清同Runtime全部Fibers。exact Context WeakSet不挡第二row，但会识别同Fiber re-apply。
- **版本证据**：rc.2 `b150a551` Loader Entry/Group、Cordis Context/Registry/Fiber、DSH Scope/ToolRuntime/AgentPresets源码与scoped/dispose一方tests。
- **实测**：无新fixture；未找到两个enabled rows同module的完全同形一方test，结论由固定控制流交叉核验。
- **沉淀**：`topics/cordis-primer.md`、index/log。
- **覆盖度变化**：无（Cordis仍L2）。

## 2026-09-05 · 跨会话核验：create后首prompt前selectModel

- **提问者**：agent（完整答案先回传来源任务并确认成功）
- **问题**：带AgentPreset创建blank Session后能否在首turn/prompt前选择exact model，以及本地draft→create→select→prompt的公开前置和错误边界。
- **结论**：该顺序受支持，model selection不改preset/header且不结束blank。必须await create与select成功；exact adapter/model/reasoning校验失败为model-unavailable，Session/subagent查找保留session-not-found/agent-busy。select只存live next-step state，首step request/header才耐久；纯文本prompt与select并发没有通用FIFO。
- **版本证据**：rc.2`b150a551`正式Host SessionsApi/ApiProxy、Agent model-selection、LLM resolveCallConfig、AgentLoop request-header与一方tests。
- **实测**：无新fixture；公开控制流和一方model-selection/API测试足以确定。
- **沉淀**：`packages/llm.md`、index/log。
- **覆盖度变化**：无（LLM仍L2）。

## 2026-09-05 · 跨会话核验：sessionless reasoning catalog与MCP增量组合

- **提问者**：agent（完整答案先回传来源任务并确认成功）
- **问题**：无SessionId读取exact model reasoning的公开面、catalog与select验证关系，以及standard preset叠加MCP后基础工具是否保留。
- **结论**：公开`llm.models`无需Session并与`session.models`共享advertised groups/failures，但无arbitrary exact-pair resolve RPC；select时重新exact验证且catalog无generation lease。MCP按qualified name增量注册进ToolRuntime，同一preset保留的基础rows不会被MCP身份替换。
- **版本证据**：rc.2`b150a551`正式Host LlmApi/IApiClient/catalog builder、LLM resolver、MCP sync、ToolRuntime scope merge、standard preset与一方tests。
- **实测**：无新fixture；公开控制流与一方catalog/MCP coexistence tests足以确定。
- **沉淀**：`packages/{llm,mcp}.md`、index/log。
- **覆盖度变化**：无（LLM/MCP仍L2）。

## 2026-09-05 · 跨会话核验：Host/Client blank、preset锁定与blank处置

- **提问者**：agent（两份相关完整答案均先回传来源任务并确认成功）
- **问题**：SessionSummary/ConversationSnapshot blank的不同边界、accepted到turn/start窗口、blank preset切换、selectModel失败后重试，以及是否能删除/放弃blank Session。
- **结论**：Host attached blank以turn/start为界，cold false可保守降级；Client在accepted或running时提前单调false。preset锁定仍以真实turn/start为准且与prompt无统一barrier。select失败可重试并保持blank/current；rc.2无Session delete/abandon，clear只清UI，archive保留log。
- **版本证据**：rc.2`b150a551` Host Sessions/Workspace API、ApiProxy blank/select/prompt、Client Session/Manager mirror、Persistence契约及一方blank/client/preset/model tests。
- **实测**：无新fixture；公开控制流与一方tests足以确定。
- **沉淀**：`packages/host.md`、index/log；reasoning与maxTools复用既有`packages/llm.md`，不重复。
- **覆盖度变化**：无（Host仍L2）。

## 2026-09-05 · 跨会话核验：SessionFace固定寻址与create耐久性

- **提问者**：agent（完整答案先回传来源任务，跨会话工具确认发送成功后才沉淀）
- **问题**：导航切换是否改变已取得SessionFace的prompt/cancel目标；create成功但首prompt未接受时的live/durable保证，以及原子abandon/delete与同id blank preset替换合同。
- **结论**：普通Session命令使用对象自身sessionId，不读取current；create完成live publication/composition但不等待persistence flush。blank不等于零事件，未收到accepted不等于Host未接受。无公共原子abandon/delete；有agentPreset.select保留同一Agent/Session并rebind，但不与首prompt或durable append构成原子事务。
- **版本证据**：远端rc.2 tag与本地对象同为`b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`；全部用固定tag源码核验，未同步或切换上游。关键路径与固定commit链接见专题页。
- **实测**：无；读取公开类型、实现及一方binding/preset/JSONL测试源码，未运行测试或调用模型。
- **沉淀**：[rc2-session-binding-create-preset.md](topics/rc2-session-binding-create-preset.md)、index/log；认知状态`verified_inference`，不包含外部项目细节。
- **覆盖度变化**：新增固定版本专题L2，不调整既有包组掌握度。


## 2026-09-05 · 授权同步：多版本咨询与 0.1.3-alpha.1

- **提问者**：human；明确授权更新到最新 DSH，同时保留新旧版本咨询。开始时主仓库、DSH 与 Cordis 镜像均干净。
- **任务契约**：同步官方上游镜像和版本路由；按固定 SHA 验证默认版、GitHub 最新版与旧版；更新受影响摘要和防腐记录。不升级其他项目、不安装全局软件、不调用模型、不迁移用户 Session 数据。
- **同步结果**：DSH `49a606bc5b5934603f22a26957a07dc799ab0291` → `d347e703908d0406b7a7ef80e3a0e594d86b2215`，292 提交、1780 文件（+49143 / -14402）；Cordis `00278924a984fedfaffb4bc3d5eb7d8e76215643` → `2ceea231802cc23892b4ad10012c55c7dd4982d4`，2 提交、7 文件（+125 / -14）。
- **实际同步命令**：对两个镜像执行 `git fetch --all --tags --quiet`，DSH `git merge --ff-only origin/master` 成功；Cordis 首次使用 `origin/master` 失败且未移动 HEAD，查询跟踪分支后以 `git merge --ff-only --no-stat origin/main` 成功。没有执行 checkout/switch/reset。已将 sync skill 改为使用各自 `@{upstream}`，失败时保护已有内容；dsh 答复尾部改为输出本次目标版本与固定 SHA，避免默认基线覆盖旧版咨询。
- **发布渠道**：GitHub 最新 release 与 master 都是 `0.1.3-alpha.1` / `d347e703`；npm 根包 metadata 没有该 version，`latest/next=0.1.2-rc.1`，`alpha=0.1.2-alpha.5`；PyPI SDK 为 `0.1.2rc1`，声明依赖 runtime-bin `==0.1.2rc1`。默认回答基线按既有 npm latest 约定升级为 rc.1；明确问“最新版”时走 GitHub alpha.1；旧版显式指定优先。
- **只读版本检测**：`git ls-remote` 查询 HEAD 与 `refs/tags/dsh-v*`；读取 `https://api.github.com/repos/deepseek-ai/deepseek-harness/releases?per_page=4`、`https://registry.npmjs.org/@deepseek-ai%2Fdsh`、`https://pypi.org/pypi/deepseek-harness-sdk/json`。本次日期快照，后续发布状态仍须现查。
- **锚点审计**：遍历 wiki frontmatter，将 DSH `49a606bc..d347e703` changed paths 与去掉 `#symbol` 的 anchors 相交：141 个去重路径、54 页。52 页原已 stale；固定 rc.2 专题按自身 commit 保留；alpha.5 动态摘要须复验，已按发布 tag 纠错。原有 69 篇 stale 页保持 stale；无本轮新增未解决 stale，不声称全库复验。
- **抽查与纠错**：回查 2026-09-02 的 alpha.5 问答记录，发现其 handle-based breaking 信号属于同期 master `49a606bc`，不属于 alpha.5 发布 tag `db6bdc35`。已纠正摘要并登记 E044；历史日志原文保留，以此条补正。新增 C082：Release 的“所有出站代理”概括未体现源码与一方测试明确保留的 OTLP 直连路径。
- **悬疑重试**：Q107 获得 `0.1.3-alpha.1` JSONL 写 owner 内核锁与双进程测试源码证据；read handle 可共存，新建 artifact 到 materializing write 才获跨进程锁。未运行 runtime 测试，rc.2 的实际竞争行为仍未解；完整关闭问题 0 条，分版本推进 1 条。
- **多版本源码验收**：禁用隐式补取 `GIT_NO_LAZY_FETCH=1` 后，11 个本地发布 tag 与远端完整 SHA 一致，11 份 `git show <SHA>:package.json` 的 version 都与 tag 一致。三个独立 SHA 并发读取 `AgentLoop.create` 和 `SessionPersistence.create`：rc.2 与 rc.1 分别保持 `Agent` / `Promise<void>` 返回；最新 alpha.1 为 `Promise<Agent>` / `Promise<SessionHandle>`。rc.2 专题 16 个锚点均能用 `git cat-file -e <SHA>:<path>` 读取。
- **负对照与 HEAD 稳定性**：`git show b150a551b8d465e31e418e1b2eaf5e79bbb7d28e:packages/session/session-persistence/src/handle.ts` 返回 128 且 stdout 为空，没有回退到当前工作树。验证脚本首次过度匹配报错措辞导致断言失败，改为检查退出码与无源码输出后通过。全部咨询读取前后 DSH HEAD 均为 `d347e703`，镜像工作区干净。
- **验证边界**：仅运行 Git 对象/源码读取与文档、skill 校验，未运行 DSH 单元测试、native 双进程测试、模型调用或正式包 install/boot。新 topic 为源码交叉验证 `verified_inference` / L2，不能把“读取命令通过”算成 DSH runtime L3。
- **修改逻辑**：AGENTS/README/index 更新渠道与 SHA；sync skill 和模块说明保留固定版本条目并避免硬编码 master；旧 alpha.5 摘要按真实发布快照更正；新 topic 提供三版本接口对照、格式变化与来源边界；errors/conflicts/open-questions/log/coverage 同步直接改变的事实。未改写旧 rc.2 专题或全量刷新历史通用页。
- **关键证据**：各版本 SHA 与绝对路径见[最新版本对照](topics/版本变更-0.1.2-alpha.5-到-0.1.3-alpha.1.md)；镜像根为 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness` 和 `/Users/majiajun/workspace/DSH-Expert/upstream/cordis`。本次临时 `sandbox/sync-20260905` 审计清单已清理，关键命令与结果保留在本记录。
- **覆盖度变化**：新增版本对照 L2 1 页；原包组掌握等级不变。当前 73 内容页、6 治理页，L2 37 / L1 36，1058 个 frontmatter 锚点，fresh 4 / stale 69；文档冲突 82、悬而未决 114。

- **收尾检查**：DSH/Cordis 本地 HEAD 再次与远端 HEAD 一致；两个镜像干净，rc.2 固定专题无 diff；两篇版本摘要的 frontmatter、18 个固定提交锚点与相对链接通过；dsh/dsh-sync 的 quick_validate、规则软链接一致性与 git diff --check 均通过。

## 2026-09-05 · 跨会话核验：tools/result撤权等待与credential rotation

- **提问者**：agent（完整答案先回传来源任务，工具确认发送成功后才沉淀）。
- **问题**：tools/result是否等待listener Promise，adapter自管撤权/whenIdle的公开组合，以及不改变preset/schema时按operation解析opaque credential ref的合同。
- **版本证据**：远端rc.2 tag与本地对象一致，固定`b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`；所有后续源码读取使用完整SHA，未切换共享工作树；vendored Cordis为4.0.1。
- **结论**：notifyResult只挂catch日志，不await；adapter可通过公开effect/Fiber自管drain，但不自动加入Agent.whenIdle。Fiber等待cleanup settle且容纳错误，不能单独证明撤权成功。CredentialProvider明确每operation解析；外部identity/ref权限不由DSH定义，无secret载体也不是类型自动保证。
- **验证**：只读核对公开exports、类型、runtime控制流及observer/rotation一方测试源码；未执行测试、模型调用或物理I/O。
- **沉淀**：[固定rc.2专题](topics/rc2-tool-result-drain-credentials.md)、index/log；剥离外部项目细节，认知状态`verified_inference`。
- **覆盖度变化**：新增固定版本专题L2，不调整既有包组掌握度。

## 2026-09-05 · 跨会话核验：agent/request attempt与schema边界

- **提问者**：agent；两份完整答案分别成功回传各自来源任务后才沉淀。
- **版本**：固定rc.2 / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`，远端tag与本地对象一致；所有源码读取使用完整SHA，无工作树版本切换。
- **结论**：同step retry重跑request而复用assembly；next返回下游配置提案，不保证最终resolved config或覆盖SDK内部HTTP attempt。传播出的throw/abort可阻断后续标准调用。公开Session/turn/step不是attempt id；schema assert为有限子集，拒绝本document ref，不提供资源预算或远端loader。
- **沉淀**：[固定rc.2专题](topics/rc2-agent-request-attempt-schema.md)、index/log；status=verified_inference，未写入外部项目事实。
- **验证边界**：源码、公开类型与一方测试源码交叉核验，未运行runtime测试、调用模型或访问真实provider；具体SDK重试配置未指定，保留未知。
- **覆盖度变化**：新增固定版本L2专题，既有包组等级不变。

## 2026-09-06 · 跨会话研究：1.3-alpha.1完整Host公开面

- **提问者**：agent；六组完整研究结果先回传来源任务并确认成功，随后补正一处空export链接，再沉淀。
- **固定版本**：A=`d347e703908d0406b7a7ef80e3a0e594d86b2215`（1.3-alpha.1），R=`a66e4702047846cdaa10c66c9d3df3951f5ea70d`（1.2-rc.1），远端tag与本地对象一致；所有查询按SHA，无版本切换。
- **范围**：发布渠道/公开exports与Profile、MCP/Skill、Team/subagent/workflow/jobs/goal、LLM attempt与计费边界、SessionHandle/迁移、Web认证与Client能力；不检查外部项目，不提供补丁。
- **关键结论**：ApiProxy消失但ClientTransportHooks仍在；controllers/store拓扑R已具备。A有handle/flush/跨进程write lease/v2 settlement，但GenerateOptions无turn/step/attempt计费键。Team仍private experimental；默认语言为zh/en；Approval要求open turn。MCP和Skill能力不能扩为OAuth/immutable dependency snapshot；raw credential方法仍在，HTTP公开route拒绝不等于所有carrier统一ACL。
- **发布查询**：浏览工具未打开registry/API，改用只读HTTP；GitHub /releases/latest返回404，列表/tag成功且A为首个prerelease，assets为空；npm根A及11个关键companions均404，R根及前10个companion有tarball metadata，migration包在R不存在；PyPI仍0.1.2rc1。
- **验证边界**：只读源码、manifest、diff、官方发布元数据和一方测试源码；未下载/安装依赖，未运行真实MCP/模型或conformance。完整可安装闭包、确定性工具日志同形验收、WKWebView继续UNKNOWN。
- **沉淀**：[固定版本专题](topics/alpha13-full-host-public-boundaries.md)、index/log、C083、Q121–Q123；状态verified_inference，未冒充FACT/L3，旧版专题不改。
- **修改逻辑**：新增六组版本化路由与条件限制，纠正BrowserAuth文档的per-request secret读取误述，登记必要未知；当前发布快照不改变默认回答基线。

## 2026-09-06 · 跨会话核验：batch取消与schema value校验

- **提问者**：agent，完整答案先成功回传来源任务后沉淀。
- **版本**：固定rc.2 / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`，远端tag一致；全程按SHA读取。
- **结论**：标准scheduler正常abort后drain已started并为未started补call/result；prepare中调用可保留其他错误，不能宣称所有pending同码或零lookup。cancel(disposed)+idle非永久新prompt闸；公开exact teardown为AgentHandle。validateJsonSchemaValue同步无signal/预算，外部Worker不是唯一内建取消合同。
- **验证**：源码、公开根导出、取消/替换/深层oneOf一方测试源码交叉核验；未执行runtime测试或physical I/O。
- **沉淀**：[固定专题](topics/rc2-batch-cancel-value-validation.md)、index/log；verified_inference，不调整其他领域等级，不评价外部项目。

## 2026-09-06 · 跨会话核验：pre-step reject的窄合同

- **提问者**：agent，完整答案先成功回传后沉淀。
- **版本与结论**：固定rc.2 `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`，远端tag一致。assembly先于pre-step；正常reject关闭turn为blocked、不进入该拟议step/provider，但不撤回assembly或前面step。reject应短路不调next；prepend不是assembly前置闸或绝对第一保证。
- **新增事实**：claimed prompt不自动返还，独立inject/steer可保留；abort/throw与append失败须区分正常blocked路径。
- **验证与沉淀**：读取公开类型、AgentLoop/Cordis实现及interception一方测试源码，未运行测试或模型；补既有[rc.2专题](topics/rc2-agent-request-attempt-schema.md)、索引和本日志，status仍verified_inference。

## 2026-09-06 · 跨会话核验：rc.1固定standing与Agent策略

- **提问者**：agent；四组完整答案已先回传来源任务并确认成功。
- **版本**：固定rc.1 `a66e4702047846cdaa10c66c9d3df3951f5ea70d`，远端tag一致；全程固定SHA读源码，无版本切换。
- **结论**：restrict过滤继承standing但不及own工具/run_code；assembly hook在收集之后。Skill模型catalog检查工具可见，用户slash加载独立；Schedule工具注册在Agent own层。各Goal/Jobs/subagent状态不因工具mask自动终止。N+1是同loop策略候选而非现成finalization合同；inbox splice/inserted/claimed/discarded均非可veto的steering闸。
- **验证**：读取公开类型、实现及scoped工具一方测试源码；未运行模型/conformance。具体完整预算/消费者组合保持UNKNOWN。
- **沉淀**：[固定rc.1专题](topics/rc1-standing-policy-inbox.md)、index/log；verified_inference，不评价外部项目，既有包组掌握度不变。

## 2026-09-06 · 跨会话核验：rc.1输入身份与retry authority

- **提问者**：agent；使用来信封装的确定来源ID成功回传完整答案，不猜测或回传历史来源。
- **版本**：固定rc.1 `a66e4702047846cdaa10c66c9d3df3951f5ea70d`，远端tag一致；所有源码按SHA读取。
- **新增事实**：prompt DTO只有requestId/sessionId/mode/content/timezone，Host构造user source；queue edit保留MessageId/source但没有revision比较。Host MessageSourceMap可扩展，plugin来源记录不等于认证授权。逻辑step允许经overflow产生不同body；llm-retry日志不是全adapter通用attempt权限。
- **carrier边界**：公开namespace/method/args足以在相应入口拒绝而复用Gateway codec；正常mux拒绝unary方法，但直接in-process及可信Host service调用不自动受Connection unary策略约束。
- **验证与沉淀**：只读类型、实现与既有测试源码，不安装/运行runtime；新增[固定专题](topics/rc1-input-authority-retry.md)、index/log，状态verified_inference，不含外部项目结论。

## 2026-09-07 · 跨会话核验：rc.1问答投递生命周期

- **提问者**：agent；完整答案先成功回传来源任务，再做本次沉淀。
- **版本**：rc.1 / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`，远端tag一致，全部固定SHA读取。
- **结论**：纯Client断线保留Host内存pending并重投同eventId；presentation delegate可继续Host下游，NO_PROVIDER并非重连等待；原signal/Host Context或source结束则取消。迟到响应按active client与delivery membership处理。Host重启没有旧ask Promise恢复，UI draft明确non-persisted。
- **沉淀**：[专题](topics/rc1-question-delivery-lifecycle.md)、index/log，状态verified_inference。
- **验证**：10个固定文件锚点、21个源码行号链接通过；只读一方重连/委派测试源码，未执行浏览器/Host重启或模型测试。
## 2026-09-15 · 跨会话核验：rc.2 pi-ai reasoning 与 capability probing

- **提问者**：agent；完整答案已先成功回传来源会话。
- **版本与结论**：固定 `0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`。`reasoningEfforts` 是 exact model 的 canonical selector 到 wire spelling 映射；compat 字段按协议类型约束；无通用 endpoint probing，unsupported effort 在网络 I/O 前以 `UNSUPPORTED_REASONING_EFFORT` 拒绝。
- **验证**：读取 `llm-pi-ai` README、`src/catalog.ts`、`src/adapter.ts` 及 adapter/serialize 测试源码；未调用真实模型。
- **沉淀**：补充既有 rc.2 reasoning 专题；`verified_inference`，`asked_by: agent`。
## 2026-09-15 · 跨会话核验：rc.2 pi-ai exact capability

- **提问者**：agent；完整答案已成功回传来源会话后沉淀。
- **版本**：`0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`；当前上游 HEAD 与目标版本不同，未混用。
- **结论**：exact model reasoningEfforts/compat 可声明并映射 wire；unsupported effort 请求前拒绝；无私有 endpoint 在线 probing，预设只能作为部署声明。
- **证据**：`llm-pi-ai` README、catalog/adapter 类型与实现、adapter/serialize 测试源码；`verified_inference`，`asked_by: agent`。
## 2026-09-15 · 跨会话核验：rc.2 hand-declared model identity

- **提问者**：agent；完整答案已回传后沉淀。
- **版本**：`0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`。
- **结论**：hand-declared `models[].id` 原样作为 provider model ID；无 alias registry/remote discovery；reasoning 与 compat 仅做本地声明和序列化校验，远端不兼容由 provider 报错。
- **状态**：`verified_inference`，`asked_by: agent`。
