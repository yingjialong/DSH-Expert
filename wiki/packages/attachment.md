---
title: packages/attachment — 持久化附件能力族
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/attachment/README.md
  - packages/attachment/attachment/README.md
  - packages/attachment/attachment/src/index.ts
  - packages/attachment/attachment-local/README.md
  - docs/subsystems/attachment.md
  - packages/README.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

不可变二进制附件（当前只有图片）的**准入 + 归一化持久化 + 请求投影 seam** 及其本地内容寻址实现：调用方拿到可序列化的 `ImageAttachmentRef`（指向 **provider-independent normalized image**，不是原始上传字节），**永远不把浏览器路径 / object URL / provider URL / base64 写进 session event** [T1: packages/attachment/attachment/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。两个包都是 product 包。

## 包清单

| 包名 | npm 名 | 一句话职责 |
|---|---|---|
| `attachment` | `@deepseek-ai/dsh-attachment` | Service Definition：不可变附件引用、图片限额、存储 service（`ctx.attachments`） |
| `attachment-local` | `@deepseek-ai/dsh-attachment-local` | Provider：`DSH_HOME` 之下的私有内容寻址存储（注册到 `ctx.attachments`） |

## 三件套结构

| 角色 | 包 | 说明 |
|---|---|---|
| Service Definition | `dsh-attachment` | `abstract class AttachmentStore extends Service`，ctx key `attachments` [T1: packages/attachment/attachment/src/index.ts] |
| Service Provider | `dsh-attachment-local` | 唯一在库实现，注册在同一个 `ctx.attachments` 上 |
| Consumer | **不在本组** | RPC 端点（session prompt / command executor）经 `admitEncodedImages()` 进入；模型侧 provider adapter（`llm-deepseek` / `llm-pi-ai`）经 `readImageRequest()` 按路由预算派生请求版本，再通过 role-neutral `ImageBlock` 引用；`dsh-acp` 也是消费者之一 |

**边界（跨文档综合）**：未发送的浏览器草稿**刻意不在本能力内**——字节只有在用户 prompt 提交、或 provider adapter 提交结构化模型输出时才进持久存储 [T1: packages/attachment/README.md]。

## 扩展点

- **换存储后端**（S3 / 数据库 / 沙箱内）→ 继承 `AttachmentStore` 并注册 `ctx.attachments`，**依赖 `dsh-attachment` 而不是 `dsh-attachment-local`**。
- **接一个新的上传通道** → 用 `admitEncodedImages(attachments, images)`，它是所有接受浏览器上传的 RPC 端点共用的 wire 入口：先强制 canonical base64，再把限额/校验/有序提交委托给 `saveImages`。上传体的类型 `EncodedImageAttachment` 从 `@deepseek-ai/dsh-attachment/types` 导出给 wire 契约引用。
- **只想校验不落盘** → `validateImage()`，跑同一套准入策略但不持久化。
- **映射自己的错误词汇** → `AttachmentError.code` 是封闭的 `AttachmentErrorCode` 联合，其中 `ImageAdmissionErrorCode` 子集标记"调用方可纠正的图片输入错误"，运行时用 `isImageAdmissionError()` 识别——每个协议适配器（ACP、HTTP、SDK）据此映射自己的错误码。联合新成员 `ATTACHMENT_PROJECTION_UNSUPPORTED`（provider 派生请求版本失败，属存储侧故障、不在 admission 子集内）。
- **给新模型路由定制请求投影** → `readImageRequest(ref, policy, signal?)`：从 normalized 附件确定性派生请求版本（等比缩进 `ImageRequestPolicy.maxPixels` / `maxBytes` 预算），返回 `RequestImageAttachment`（含缓存键 `variantId`）；本地实现带 variantId 磁盘缓存 + singleflight；基类默认 reject `ATTACHMENT_PROJECTION_UNSUPPORTED`，换非本地后端必须自己实现。
- **调限额** → 两级写入期策略：**源准入**（默认单张 20 MiB / 64,000,000 px / 单边 8192px，每消息 20 张 / 200 MiB）与 **normalization**（默认长边 2048px / 4 MiB），配置键 `maxImage*`、`normalizedImageMaxDimension`、`normalizedImageMaxBytes`、`imageCompressionConcurrency`；`DSH_HOME` 走共享路径策略（显式 config → `$DSH_HOME` → `~/.dsh`）。

## Known Limitations

`dsh-attachment`：

- 第一版**只接受 PNG / JPEG / WebP / GIF**。
- 保留与垃圾回收被推迟，因为 resumed / forked session 可能共享同一批不可变对象。
- 通用文件、音频、视频、以及"持久化的未发送草稿"需要另外的生命周期与 provider 契约。

`dsh-attachment-local`：

- 对象**无限期保留**，reference-aware GC 推迟。
- 本地后端假定 host 与 provider adapter 共享同一个文件系统 service。
- 动图 GIF 的元数据只从 logical screen 校验，帧级解码策略归 provider。

## 陷阱

1. **批量提交是全有或全无，但残留可能存在**：`saveImages` 先校验**全部**成员再写任何一个（批量校验抽成了 `protected validateImageBatch`），然后按序提交；后续存储失败不返回部分引用——但**更早写入的不可变内容寻址对象可能已经留在盘上且不可达**，直到 GC 存在为止。
2. **限额是两级的，别再记"单边 2000px"**：rc.8 时代源准入单边默认 2000px（单张 3.5 MiB / 40M px）；现在**源准入**（单边 8192px / 64M px / 单张 20 MiB / 每消息 20 张 200 MiB）只挡明显超标的图，准入后 normalization 统一压到**长边 2048px / 4 MiB** 的 provider-independent 对象，`originalDimensions` 仅在缩小发生时记录缩放前尺寸；请求期再由 `readImageRequest()` 按具体路由的 `ImageRequestPolicy` 投影——持久历史里存的永远是 normalized 版本，不是调用方上传的原字节。
3. **准入与 normalization 是写入期策略，请求投影是读取期策略**：后来把准入策略调小**不会**让已准入的历史变得不可读；但同一 attachment 在不同路由的 `ImageRequestPolicy` 下会派生不同请求版本（各自按 `variantId` 缓存）。
4. **读要验签**：`readImage` 会对内容寻址对象重新校验 digest 与已记录的元数据；写入准入与读取都会**完整解码 raster** 后才接受格式与尺寸。`readImageRequest` 的新缓存条目同样全解码后才发布，缓存命中只做 bounded metadata probe。
5. **cancellation 要保真**：`readImage` 与 `readImageRequest` 的可选取消都必须在 backend 与验证环节被观测并**原样保留**，不得翻译成 `ATTACHMENT_READ_FAILED` 这类存储错误；本地 `readImageRequest` 对相同请求身份 singleflight，每个 waiter 可独立 cancel，无 waiter 剩余时停止共享工作。
6. **崩溃安全靠一串具体动作**：每进程先把 home 的每一级祖先目录同步到文件系统根一次，再用私有 staging 目录 + owner-only 文件 + 同步临时文件 + 原子独占 hard-link 发布 + 发布路径上的目录 sync（POSIX；Windows 依赖文件系统元数据日志）。**Windows 的保证弱于 POSIX**。
7. **KV cache**：加一张图会改变 provider 请求，从而使受影响的 request 后缀失效——不是 append-only。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 组边界（为什么草稿不在内） | `packages/attachment/README.md` |
| seam 契约、批量准入语义、错误码分层 | `packages/attachment/attachment/README.md` |
| `AttachmentStore` 抽象方法签名与 `AttachmentId` brand | `packages/attachment/attachment/src/index.ts` |
| 落盘布局、崩溃安全动作序列、尺寸/像素限额 | `packages/attachment/attachment-local/README.md` |
| 请求投影缓存、路由预算缩放、transform 并发 limiter | `packages/attachment/attachment-local/src/request-image.ts` |
| 子系统主篇（跨包的图片生命周期） | `docs/subsystems/attachment.md` |
| ACP 侧如何 advertise / 提交图片 | `packages/acp/acp/README.md` |
| 组稳定性标注 | `packages/README.md` |
