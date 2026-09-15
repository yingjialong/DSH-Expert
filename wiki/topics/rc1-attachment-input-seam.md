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

固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。`@deepseek-ai/dsh-attachment` root 公开 `AttachmentStore`、`admitEncodedImages`、`admitPromptContent` 及图片类型；`attachment-local` 公开 `LocalAttachmentStore` 与本地存储/规范化 helper。`AttachmentStore` 提供 `validateImage`、`saveImages`、`saveImage`、`readImage`、`readImageRequest`。

官方 browser Client 的 `ISession.prompt(content: PromptContentPart[], ...)` 接受文本与 browser-owned temporary image uploads；`beginSubmission` 可登记本地图片 echo；`readAttachment` 按 durable reference 读取授权图片。Host 在 admission 前验证图片并持久化 normalized reference。

本版本没有通用 `InputRef`、`InputStore` 或原生 picker/import API。文件选择和把 bytes 转为 `SaveImageAttachment` 属于宿主/浏览器载体；附件 seam 不是 Node adapter。`imageHostPath` 仅是可选 host-file-backed 定位 helper，默认返回 undefined。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/attachment/attachment/src/index.ts`、`attachment-local/src/index.ts`、`packages/api/session-controller/src/client/contract/session.ts` 与 `src/types.ts`，均按固定 SHA 读取；未执行端到端浏览器或模型测试。
