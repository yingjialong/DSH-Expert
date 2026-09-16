---
title: rc.1 Attachment 与 Session 图片输入 seam
description: 固定 rc.1 的 AttachmentStore、官方 Client 图片输入及 picker 边界。
type: reference
status: verified_inference
updated: 2026-09-16
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
asked_by: agent
---

# rc.1 Attachment 与 Session 图片输入 seam

## submission、取消与图片入口补充核验

固定同SHA：SessionController.prompt仅在入口signal.throwIfAborted，之后commands.prompt不接收signal。异步模型/图片准入已经开始后，Client abort或传输失败不证明Host未接受；没有按SessionRequestId回滚已接收prompt的公开接口。cancel仅按Session/当前turn请求取消并keepInbox，updateQueue另以pending MessageId寻址。

beginSubmission/abandon仅控制local echo；正常RemoteResult失败退failed，但Session.prompt没有兜住任意JS throw的通用finally。ConversationService序列化失败调用abandon；图片observed时转交preview给durable cache或revoke，failed时保留draft供恢复。serializeImages不接signal；本版图片base64随prompt发送，不是独立uploadId事务，Host content-addressed存储无request rollback保证。

官方入口已确认paste/drop：ui-conversation提供addImages与keymap；ui-attachment注册draft rail/document drop和图片展示。Session缺席时addImages undefined；removed/inert/无live input/blocked/continuable parentOffline锁输入，adjudicating/submitting忙时不能paste/drop。未发现conversation自带原生picker API。

上述Client gate没有按inputModalities隐藏入口；input:['text']并不证明UI无图片入口。Host准入hasImage时查exact model，明确不含image则在保存前拒绝session/attachment-invalid（MODEL_DOES_NOT_SUPPORT_IMAGES）。需分开UI可输入、Host存储准入与实际模型视觉支持。

证据：[Host prompt入口](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/index.ts)、[Host准入](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/commands.ts)、[conversation资源所有权](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/service.ts)、[InputBar gate](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/skeleton/InputBar.tsx)。源码静态核验，网络竞争与浏览器端到端未实测。

固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。`@deepseek-ai/dsh-attachment` root 公开 `AttachmentStore`、`admitEncodedImages`、`admitPromptContent` 及图片类型；`attachment-local` 公开 `LocalAttachmentStore` 与本地存储/规范化 helper。`AttachmentStore` 提供 `validateImage`、`saveImages`、`saveImage`、`readImage`、`readImageRequest`。

官方 browser Client 的 `ISession.prompt(content: PromptContentPart[], ...)` 接受文本与 browser-owned temporary image uploads；`beginSubmission` 可登记本地图片 echo；`readAttachment` 按 durable reference 读取授权图片。Host 在 admission 前验证图片并持久化 normalized reference。

本版本没有通用 `InputRef`、`InputStore` 或原生 picker/import API。文件选择和把 bytes 转为 `SaveImageAttachment` 属于宿主/浏览器载体；附件 seam 不是 Node adapter。`imageHostPath` 仅是可选 host-file-backed 定位 helper，默认返回 undefined。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/attachment/attachment/src/index.ts`、`attachment-local/src/index.ts`、`packages/api/session-controller/src/client/contract/session.ts` 与 `src/types.ts`，均按固定 SHA 读取；未执行端到端浏览器或模型测试。
