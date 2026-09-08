---
title: rc.1 图片准入取消与原子发布边界
description: 区分 prompt 取消、批量准备、单对象持久发布、临时文件清理和未引用对象保留。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/api/session-controller/src/index.ts#SessionController
  - packages/api/session-controller/src/commands.ts
  - packages/api/session-controller/src/agent.ts
  - packages/attachment/attachment/src/admission.ts
  - packages/attachment/attachment/src/index.ts#AttachmentStore
  - packages/attachment/attachment/src/types.ts#SaveImageAttachment
  - packages/attachment/attachment/tests/index.spec.ts
  - packages/attachment/attachment-local/src/index.ts#LocalAttachmentStore
  - packages/attachment/attachment-local/src/store.ts#commitPreparedImageFile
  - packages/attachment/attachment-local/src/compression-limiter.ts
  - packages/attachment/attachment-local/tests/store.spec.ts
  - packages/attachment/attachment-local/tests/index.spec.ts
  - packages/attachment/attachment-local/README.md
  - packages/attachment/attachment-local/package.json
  - packages/attachment/attachment-local/src/request-image.ts#readRequestImageFile
  - packages/attachment/attachment-local/tests/request-image.spec.ts
  - packages/attachment/attachment-local/src/normalization.ts#normalizeImage
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/topics/rc1-input-authority-retry]]"
  - "[[wiki/packages/attachment]]"
---

条件：TypeScript Host · 官方 SessionController · per-Agent 图片准入队列 · local 图片存储 · AttachmentStore 消费链 · 不调用模型 · POSIX/Windows 保证分开 · 固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。

## 准入、队列与取消

- prompt 仅入口检查调用 signal；后续 commands 不接该 signal。图片流程为 resolveAgent→per-Agent image-admission chain→resolveModelInfo→admitPromptContent→saveImages→createUserMessage→入箱。排队/保存期间 abort 不保证阻止后续提交或入箱。
- canonical base64 全部解码后才一次调 saveImages。基类先 batch limits、逐个 validate，再逐个 save；local override 先经实例 FIFO CompressionLimiter 并发准备整批，全部成功后逐个 commit。默认 compression 并发2、可配至8，非全 Host 事务锁。
- 任何 prepare 失败都不进入该批 commit，但 Promise.all 不取消其余准备任务。后续 commit 失败不返回部分 refs，不回滚早先已发布对象。全部保存后入箱失败同样没有自动 rollback。
- saveImage/saveImages/validateImage 及 local prepare/commit helpers 均无 signal，SaveImageAttachment 无请求身份/取消字段。readImage/readImageRequest 的 signal 属于读取路径，不能用于取消保存。

## 单对象 prepare/commit

- 公开 prepareImageFile(input,limits,policy) 解码/验证/归一化并计算 digest，返回 `{data,ref}`，不创建 attachment storage 文件。PreparedImageFile 不是 opaque transaction、没有 abort/dispose，字段也非全面 readonly。
- commitPreparedImageFile(root,prepared) 先核 id、data digest/长度；不重跑全部 raster normalization 或任意 ref 元数据校验。root 合同为 DSH_HOME/attachments/v1。
- 确保 private bucket/tmp 与祖先目录 durability，进程首次把 home 祖先证明到 filesystem root；随后 temp 用 O_EXCL/0600 创建，write→file sync→close→exclusive hard link 发布 target。
- EEXIST 去重会读 target 核 digest；正常路径 unlink temp、chmod target 0400，再 sync bucket/objects 后返回 ref。目录“已存在”不等于已 durable，dedup 也重复 sync。
- 这是单对象原子可见与返回前持久化动作，不是 batch/Session/Inbox 的共同事务。Windows 跳过目录 fsync，依赖文件系统元数据日志；不宣称等同 POSIX fsync 链。

## 清理保证

- try/catch 内失败尝试关闭 descriptor、unlink 本次 temp；忽略 ENOENT，其他 cleanup unlink 失败会抛出，不能保证所有失败都清完。初始化目录动作在此 try 之前，失败可留下目录且未必进入统一错误包装。
- link 成功后的 chmod/sync 失败不会删除 target；方法 reject 仍可已有发布对象。崩溃不保证 catch 执行，未见启动清扫 orphan tmp 的承诺；temp unlink 后未额外 fsync(tmp) 证明其删除 crash-durable。
- 本版无 delete/release/rollback-save 或 reference-aware GC；README 明确 images kept forever。返回 ref 不区分新建/去重，没有“本次新建对象集合”的回收 receipt。未接受 prompt 不等于该 hash 对象没有其他引用。

## 扩展与证据

AttachmentStore 是公开 Definition，可由第一方 Provider 实现 imageLimits/validate/save/read 合同，官方 Controller 继续唯一经 ctx.attachments 消费。local Config 可限定策略，公开 prepare/commit 可复用；但 Provider 无法由该标准调用得到未传下来的 request signal/身份。Cordis 自有资源生命周期不自动补成 prompt 取消或 batch rollback 合同，stock local 写队列没有据此公布的取消/卸载 drain 保证。

[写入合同](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment/src/index.ts#L40)、[local batch](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment-local/src/index.ts#L196)、[prepare/commit](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment-local/src/store.ts#L84)、[batch failure 测试](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment/tests/index.spec.ts#L119)、[fsync 测试](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment-local/tests/store.spec.ts#L77)、[保留限制](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment-local/README.md#L133)。

远端 tag 已核对；精确 attachment/attachment-local rc.1 tarball 在内存读取 manifest、root .d.ts、store.d.ts 与 root JS，确认 helpers 实际公开。未安装、未运行故障注入/取消/crash/Host-Client E2E，不评价外部项目。

## 2026-09-08 · 内容完整性不等于文件身份

本节同一固定 rc.1 SHA 复验，asked_by: agent，verified_inference。

- readImageFile 直接 Node readFile(path,{signal})，然后 digest + probeImage header 核 mediaType/bytes/width/height。该版读取不是完整 raster decode；假设 digest 对应曾经通过 admission 的字节。
- helper/LocalAttachmentStore 没有逐段 lstat/no-follow、dirfd、uid/mode/nlink/inode identity 检查；root 的 path.resolve 不是 filesystem 身份证明，也不经 ctx.fs。符合 ref 的字节不能证明由期望目录中的 owner-only 独占文件读取。
- normal object 创建会使用目录0700、temp0600、exclusive link、target0400 与 fsync；这些实际创建/修改动作不审计所有既有祖先或链接。EEXIST 仍 readFile 既有 target 核 digest，未检查其 link/owner 身份；exclusive temp 不保护父目录链。
- request-cache 的 hash 是 attachmentId+transform/policy descriptor 的摘要，不是 cache bytes digest。readCached 只按路径读取并 probe uchar/srgb/dimensions/alpha 等属性，不证明 cache 像素由该源图变换所得，也无 owner/link 检查。
- cache 写为 mkdir(mode0700)→临时 wx/0600→rename final→finally rm temp；不纠正所有已存在目录权限，无正常对象 fsync 链或最终0400。rename 替换 final entry 不等于跟随 final symlink 写 target，但父路径仍未固定。
- 若依赖可信根、无非预期 link、owner-only、目录不可被替换等性质，外层存储 owner 必须建立并维持；一次路径预检后让 helper 重新 open 仍无 DSH 提供的竞态闭环。

公开复用范围：AttachmentStore Provider 可拥有安全读取机制并保留官方 consumer；prepareImageFile/validateImageFile 接受已读取 bytes，可继续官方 admission/normalization，但可能改变 bytes/ref，不是既有 normalized ref 的保持 identity 验证器。readRequestImageFile 接受已验证 StoredImageAttachment，仍会按 root 访问自身 cache。没有公开 fd/FileHandle/reader 注入版 readImageFile 或 verifyStoredImageBytes(ref,data)。probeImage/detectImage/cache helpers 未从 root 导出；normalizeImage 需要已取得的 detected metadata。imageHostPath 是定位不是安全打开；normalizedImagePath 未从 root 导出，不推荐私有导入。

证据：[normalized read](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment-local/src/store.ts#L272)、[cache read/write](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment-local/src/request-image.ts#L116)、[verified source 前置](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment-local/src/request-image.ts#L176)、[Local source/read 组合](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment-local/src/index.ts#L235)、[缓存属性测试](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/attachment/attachment-local/tests/request-image.spec.ts#L83)。重新核对正式 attachment-local root .d.ts/JS exports，无 src/；未运行 symlink/hardlink/权限竞态或 E2E，内容拒绝测试不当作路径安全证明。
