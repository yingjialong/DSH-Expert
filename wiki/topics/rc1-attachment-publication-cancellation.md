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
