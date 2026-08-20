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
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

不可变二进制附件（当前只有图片）的**准入 + 持久化 seam** 及其本地内容寻址实现：调用方拿到可序列化的 `ImageAttachmentRef`，**永远不把浏览器路径 / object URL / provider URL / base64 写进 session event** [T1: packages/attachment/attachment/README.md]。

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
| Consumer | **不在本组** | RPC 端点（session prompt / command executor）经 `admitEncodedImages()` 进入；模型侧通过 core 的 role-neutral `ImageBlock` 与 provider adapter 解析引用；`dsh-acp` 也是消费者之一 |

**边界（跨文档综合）**：未发送的浏览器草稿**刻意不在本能力内**——字节只有在用户 prompt 提交、或 provider adapter 提交结构化模型输出时才进持久存储 [T1: packages/attachment/README.md]。

## 扩展点

- **换存储后端**（S3 / 数据库 / 沙箱内）→ 继承 `AttachmentStore` 并注册 `ctx.attachments`，**依赖 `dsh-attachment` 而不是 `dsh-attachment-local`**。
- **接一个新的上传通道** → 用 `admitEncodedImages(attachments, images)`，它是所有接受浏览器上传的 RPC 端点共用的 wire 入口：先强制 canonical base64，再把限额/校验/有序提交委托给 `saveImages`。上传体的类型 `EncodedImageAttachment` 从 `@deepseek-ai/dsh-attachment/types` 导出给 wire 契约引用。
- **只想校验不落盘** → `validateImage()`，跑同一套准入策略但不持久化。
- **映射自己的错误词汇** → `AttachmentError.code` 是封闭的 `AttachmentErrorCode` 联合，其中 `ImageAdmissionErrorCode` 子集标记"调用方可纠正的图片输入错误"，运行时用 `isImageAdmissionError()` 识别——每个协议适配器（ACP、HTTP、SDK）据此映射自己的错误码。
- **调限额** → 字节数、总像素、单边尺寸是 `attachment-local` 的**写入期准入策略**；`DSH_HOME` 走共享路径策略（显式 config → `$DSH_HOME` → `~/.dsh`）。

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

1. **批量提交是全有或全无，但残留可能存在**：`saveImages` 先校验**全部**成员再写任何一个，然后按序提交；后续存储失败不返回部分引用——但**更早写入的不可变内容寻址对象可能已经留在盘上且不可达**，直到 GC 存在为止。
2. **单边默认 2000px 是刻意压在模型路由限制之下的**：一张被准入的图会随该 session 的**每一次后续请求**发送，所以准入是最后一道能把"provider 会拒绝的图"挡在持久历史之外的关卡。
3. **限额是写入期策略，不是读取期策略**：后来把策略调小**不会**让已准入的历史变得不可读。
4. **读要验签**：`readImage` 会对内容寻址对象重新校验 digest 与已记录的元数据；写入准入与读取都会**完整解码 raster** 后才接受格式与尺寸。
5. **cancellation 要保真**：`readImage` 的可选取消必须在 backend 与验证环节被观测并**原样保留**，不得翻译成 `ATTACHMENT_READ_FAILED` 这类存储错误。
6. **崩溃安全靠一串具体动作**：每进程先把 home 的每一级祖先目录同步到文件系统根一次，再用私有 staging 目录 + owner-only 文件 + 同步临时文件 + 原子独占 hard-link 发布 + 发布路径上的目录 sync（POSIX；Windows 依赖文件系统元数据日志）。**Windows 的保证弱于 POSIX**。
7. **KV cache**：加一张图会改变 provider 请求，从而使受影响的 request 后缀失效——不是 append-only。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 组边界（为什么草稿不在内） | `packages/attachment/README.md` |
| seam 契约、批量准入语义、错误码分层 | `packages/attachment/attachment/README.md` |
| `AttachmentStore` 抽象方法签名与 `AttachmentId` brand | `packages/attachment/attachment/src/index.ts` |
| 落盘布局、崩溃安全动作序列、尺寸/像素限额 | `packages/attachment/attachment-local/README.md` |
| 子系统主篇（跨包的图片生命周期） | `docs/subsystems/attachment.md` |
| ACP 侧如何 advertise / 提交图片 | `packages/acp/acp/README.md` |
| 组稳定性标注 | `packages/README.md` |
