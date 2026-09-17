---
title: 1.5-rc.2 受控 Profile、文件链与 UI 存储扩展面
description: 区分通用上传、模型可读 handle、文档预览与解析，核验公开控制边界。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-18
updated: 2026-09-18
asked_by: agent
anchors:
  - packages/boot/app-boot/src/profile.ts#loadProfileDirectory
  - packages/boot/app-boot/src/index.ts#boot
  - packages/attachment/attachment/src/types.ts#FileAttachmentRef
  - packages/attachment/attachment/src/index.ts#AttachmentStore
  - packages/client/file-upload/src/index.ts#FileUploads
  - packages/client/file-upload/src/types.ts#ClientFileUploadHooks
  - packages/client/file-upload/src/client/runtime.ts#customTransport
  - packages/client/file-upload/src/http-route.ts
  - packages/api/session-controller/src/commands.ts#prompt
  - packages/llm/llm/src/content.ts#projectFilesToText
  - packages/llm/llm/src/index.ts#fileReadPath
  - packages/context/file-reference/src/index.ts
  - packages/fs/fs/src/index.ts#processPathFromHostPath
  - packages/fs/tool-fs/src/read.ts
  - packages/fs/tool-present/src/index.ts
  - packages/api/workspace-files/src/index.ts
  - packages/client/ui-sidebar-documentpreview/src/client/index.ts
  - packages/session-query/session-log-export/src/archive.ts
  - packages/client/ui-conversation/src/client/index.ts
  - packages/client/ui-conversation/src/client/conversation/assembly.ts
  - packages/client/ui-conversation/src/client/service.ts
  - packages/client/ui-chat/src/client/index.ts
  - packages/client/ui-slots/src/index.ts
  - packages/session/session-persistence/src/handle.ts
  - packages/storage/storage/src/backend.ts
  - packages/storage/storage-domain/src/index.ts
  - packages/storage/storage-domain/src/spec.ts
  - packages/storage/storage-json/src/per-record-unit.ts
  - packages/session/session-projection-cache/src/index.ts
  - packages/session/session-projection-cache/src/spec.ts
related:
  - "[[wiki/topics/rc15-public-embedding-migration]]"
---

# 版本与验证范围

固定0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203。2026-09-18远端tag与本地对象一致；源码读取均指定SHA。23个相关正式npm包tarball的sha512匹配registry integrity，并核对exports、成员、声明及关键JS出口；未安装依赖、启动产品或调用模型。已完整回传来源任务并确认成功后沉淀。

条件（八维坐标）：TS/JS宿主 · 应用自有Loader/Client组合 · 每Session上传receipt及跨进程物理owner · 本地或非本地附件存储 · 官方工具/状态复用 · 无真实模型凭据调用 · 平台实现另验 · 固定目标版本。全部L2 / verified_inference，不认证外部具体carrier或项目。

## 应用自有受控Profile

- app-boot正式根导出loadProfileDirectory/loadProfile/composeEntries/boot、watchUserPatches及env helpers。
- loadProfileDirectory(binName,dir,installAnchor,{userLayer:false})读取已初始化的应用目录；只选manifest bundles，缺省[]；不自动加base/web/SDK，不经共享home名字寻址，不读该profile的cordis.patch.yml。
- loadProfile(name,...)则会为缺失的shipped名字初始化模板并normalizeShippedProfile；两条入口不能混称。
- resolveBundleDir按installAnchor优先、profile目录其次解析；不是digest allowlist或禁止fallback的开关。bundle必须声明dsh.bundle.patch。
- patchReload仅live/startup，缺省live；调用方/launcher决定是否运行watchUserPatches。startup不是阻止任何插件自带HMR的全局安全开关。
- boot新建Context→dshHomePath→Loader→prepare(ctx)→root Include→settle/activation audit。服务注入在prepare回调内、entry mount前；没有额外的“prepare前接管内部Context”参数。
- composeEntries返回在空root上组合的EntryOptions[]；boot接绝对配置文件路径，不直接接这个数组。
- bareModuleBaseUrl控制root Include解析；relative仍随配置目录，cordis builtin按原规则。不能扩大成全部nested imports的授权沙箱。
- boot不自动调用env加载或patch watcher，但也不清空process.env；配置!!js与已挂代码仍能访问环境。上述选择能控制显式来源，不能自动证明环境/代码/权限闭包。

## 上传、准入与模型可见性

### 附件公开合同

@deepseek-ai/dsh-attachment的通用读取名是readFileStream，**没有本题假设的readFile方法**。

| 方法/值 | 合同 |
| --- | --- |
| saveFile({data,name?}) | 原样字节数组，返回耐久FileAttachmentRef，无signal参数 |
| saveFileStream({data,signal?,name?}) | AsyncIterable字节、背压、不能聚合完整文件 |
| readFileStream(ref,signal?) | 有界流，完整性失败拒绝迭代，不能在读完前假定摘要已验证 |
| FileAttachmentRef | attachmentId为原字节sha256内容标识，name为消毒显示leaf名，bytes为确切长度；不是路径/URL/解析文本 |
| fileHostPath(ref) | 可选Host绝对路径，基类返回undefined |

文件写路径无通用准入大小上限；图片是独立的mediaType/尺寸/归一化/路由request variant链，不能混同。

### 官方链路

1. FileUploads Host（dsh-client-file-upload）接Remote base64或streaming Fetch，将字节交AttachmentStore，返回{receiptId,file}。receipt只属于本次上传的Agent scope；save后重查同一live Agent；subagent上传拒绝。
2. Browser /client经ctx.fileUpload接受Blob/Uint8Array/ReadableStream、signal与progress callback。Blob/stream优先background，Uint8Array走Remote/base64。Composer认可的image MIME走图片draft，其余文件立即后台上传，draft为runtime-only并可跨Session切换保留。
3. SessionController的wire file引用receiptId；resolve到当前Agent的file ref后才admitPromptContent，image被保存而file ref原样通过。随后createUserMessage、live检查、bindPrompt、入箱、commit绑定。admission不是工具已读或Session已flush。
4. LlmRuntime在adapter dispatch前，无条件将所有file block（包括嵌套tool-result）投影成handle文本。路径来自fileHostPath→fs.processPathFromHostPath。不存在映射则明确提示模型“无可读路径，不得称已读”。没有通用file原生字节投递。
5. file-reference的@候选/语法只提供path，不提取正文。tool-fs的read读UTF-8窗口，read_image另走image能力；它不是PDF/Office/表格/压缩包解析器。
6. tool-present校验现存regular file，成功追加deliverables/presented的path/description引用。**不复制或保留文件内容**，不能称immutable附件快照。
7. workspace-files经composed fs做文本/byte读取和变更观察；Sidebar内置text/Markdown/code/HTML/image/PDF预览。PDF.js属于浏览器页面渲染，预览不自动写模型上下文。
8. session-log-export先flush/read日志，再提取其中的image/file refs，分别readImage/readFileStream进ZIP；不必fileHostPath。未消费receipt与普通present路径不自动成为导出附件。

目标未找到通用Host parser registry、parseFile(ref)或PDF/Office→typed chunks→Session自动管线。代码中YAML parseDocument是配置解析，不是此能力。外部shell/远端工具可产生解析结果，但不能归为内建文档理解。

## 非本地store与carrier缺口

- AttachmentStore refs与streams不要求Host本地明文文件；DTO/pull-chunk桥在合同上可实现存储/导出。这是条件性可组合性，非现成官方Main adapter验收。
- fileHostPath若返回必须符合Host绝对路径语义；processPathFromHostPath是同一文件到执行世界的映射。不能将opaque id/不存在路径伪装成真实文件。无路径时存储/export可用，不代表默认模型read、Sidebar/present路径自动可用。
- 公开__DSH_FILE_UPLOAD__的ClientFileUploadHooks.fetch保留body/signal，但**customTransport没有转接request.onProgress**。默认Worker的字节消费进度不自动在custom fetch路径保留，更不是持久提交进度。完整FileUploadService可表达progress，但自定义实现未实测。
- streaming signal需provider/carrier兑现；base64 saveFile无signal，入口abort检查不保证进行中的保存会停止。
- releaseDraftAttachment清浏览器draft并abort活动upload，不删除已发布对象。Host observe user/message rpcId、retirePrompt、session/disposed清staged映射；bind失败只回滚receipt绑定。
- 无AttachmentStore delete/release/refcount/GC，也无独立删除任意未消费receipt端点；上传后丢弃且未绑定的receipt可留至Session生命周期结束。无跨重启receipt恢复、上传与Session提交原子事务保证。
- hash不是Session授权。image读取controller检查历史可达性，不据此推断有通用file-by-id授权下载端点。

## 解析结果与可重建上下文

可用正常prompt/admitted text、createUserMessage与Agent.send/followup/steer输入解析文本；也可由ToolDefinition执行解析，借标准AgentLoop记录的tool/result content进入模型与历史。canonical value不等于model-visible content，直接tools.execute不自动写标准模型tool pair。图片解析产物仍须image ref/request policy。

只改Client预览、draft或瞬时请求middleware不构成冷恢复上下文；任意自定义event也不能跳过格式/vocabulary验证。这里确认普通消息/工具扩展面，不声称有专门parser seam或自动来源关联机制。

## 官方UI公开重用

Conversation /client模块factory及声明公开UiConversation、ConversationController、ConversationNodeAssembler与Definition/Event/View registries；binding(SessionBinding|SessionId)有snapshot/activate/target('chat')观察面。具体ConversationController公开draft/upload操作；较窄IConversation只暴露input/blocks/send/updateQueue/cancel/loadOlder，不能将两者等同。

SlotCore公开single/list/keyed/chain、scope、owner props、hook与注册生命周期。可用官方state/节点/slots组成自有布局是条件性推论；无需以整个WebApp为唯一黑盒，但apply本身会挂core/shell/input/renderer贡献，仍须满足真实uiSession/uiWorkspace/fileUpload/locale/settingsScope/sidebarRight等依赖。

正式lib/client.js是window.__ModuleLoader__.load({id,factory})，**不是普通独立ESM React组件库**。不能由.d.ts的named exports推出可跳过Client module loader；内部ChatView/Composer私有文件也不自动成为公共runtime export。

## Session切片、KV与cache

- SessionHandle.read按逻辑event seq切片，省略length读余下、EOF后空；返回caller-owned外层数组和eventState。不是byte range、性能复杂度保证或多调用snapshot事务。
- 同handle读不后退；同backend instance在append/flush完成后开始的读至少看到该prefix；并发读只保证有效连续prefix。append可缓冲，flush耐久，write close完成pending durability并释放owner。
- KvUnit.put/delete/setGlobal单调用原子且resolve后耐久；不序列化并发写、不提供跨record事务。layout与compatibleVersions是backend/owner合同，不等于所有backend必须以本地per-record文件实现。
- backupRecord?可选：将坏record移出readable集合并保留bytes供诊断；无此成员则storage-domain的backup-and-skip无法降级，仍reject，global始终reject。不得假设row-store/Main后端自带.bak。
- JSON per-record用rename到分钟时间戳.bak，同分钟同key可覆盖；这不是全版本备份/回滚承诺。
- projection cache spec是version7/per-record/compatibleVersions[3,4,5,6]/backup-and-skip。可丢弃派生数据，仍检查format/lifecycle/lineage与row version；旧record被加载不等于可用于当前fold。
- cache.write先取cut，再attached时await sessions.flush，后写record，避免cache领先日志；detach由owner retirement/cold floor约束。显式write可reject，自动路径才contain/warn。
- coldSnapshot接调用方完整日志；不证明persistence range读取高效。Session/KV-cache/AttachmentStore是三套存储合同，不能把一个CAS/flush提升为跨三者事务。

## 关键证据与限制

所有anchors按固定T读取。经确认的关键绝对路径：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/boot/app-boot/src/profile.ts`：loadProfileDirectory/resolveBundleDir。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/llm/llm/src/content.ts`：无条件file handle投影。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/file-upload/src/client/runtime.ts`：customTransport进度缺口。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/file-upload/src/index.ts`：receipt生命周期。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/fs/tool-present/src/index.ts`：不复制源文件。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/storage/storage-domain/src/index.ts`：backupRecord缺席仍拒绝。

正式包元数据：[app-boot](https://registry.npmjs.org/@deepseek-ai%2Fdsh-app-boot/0.1.5-rc.2)、[attachment](https://registry.npmjs.org/@deepseek-ai%2Fdsh-attachment/0.1.5-rc.2)、[file-upload](https://registry.npmjs.org/@deepseek-ai%2Fdsh-client-file-upload/0.1.5-rc.2)。

未验证：任意跨进程pull carrier实际进度/取消/崩溃、完整受控profile/UI装配、解析工具效果和跨存储事务。未运行产品，不把本次类型核验计为运行验收。
