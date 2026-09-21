---
title: 1.5-rc.2 Composer 附件与 Context React 边界
description: 核对附件入口控制、后台上传可用性、Context props 与官方 React 基线。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-21
updated: 2026-09-21
asked_by: agent
anchors:
  - packages/client/ui-conversation/src/client/apply.ts
  - packages/client/ui-conversation/src/client/index.ts
  - packages/client/ui-conversation/src/client/contract/slots.ts
  - packages/client/ui-conversation/src/client/skeleton/InputBar.tsx
  - packages/client/ui-conversation/src/client/input/editor/keymap.ts
  - packages/client/ui-conversation/src/client/service.ts
  - packages/client/ui-attachment/src/client/ComposerAttachments.tsx
  - packages/client/file-upload/src/client/runtime.ts
  - packages/client/file-upload/src/client/contract.ts
  - packages/client/ui-session/src/client/index.ts
  - packages/client/ui-renderer/src/client/scoped-slots.tsx
  - packages/client/ui-renderer/src/client/bindings.tsx
  - packages/client/ui-slots/src/renderer.ts
  - vendor/cordis/src/reflect.ts
  - vendor/cordis/src/context.ts
  - apps/web/package.json
  - apps/web/vite.config.ts
  - pnpm-lock.yaml
  - scripts/dev-web.ts
  - packages/client/tsdown.client.ts
  - packages/client/web/src/seed.ts
related:
  - "[[wiki/topics/rc15-controlled-profile-file-chain]]"
  - "[[wiki/topics/rc15-client-input-layout-contract]]"
---

固定 DSH 0.1.5-rc.2；Cordis 4.0.2。六个 UI/upload 发布包及 Cordis tarball sha512/exports 核对，Conversation Config/类型出口及 Cordis JS 复验。先完整回传成功，再沉淀；未运行浏览器或 React 实验。

条件：TS/React Client · 默认 Conversation Composer · Session binding · Browser File/Worker/Remote · 官方 slots/输入机 · 未调用模型 · 构建配置静态核验 · 固定本页 SHA。

## 附件控制

Conversation Config 只有 maxConcurrentFileUploads，min(1)、默认2；无独立禁用 picker/image paste/drop 的专用开关。fileUpload.available 表示 background carrier 可用，不是全部附件能力。false 时 post 拒绝，但 upload 对 Blob/Uint8Array 仍走 Remote/base64 fallback；stream 无后台 carrier 则拒绝。

默认 InputBar 自带 paperclip/hidden file input，disabled 按 subagent/locked/machineBusy/addFiles 判定，不读 available。paste 经 Lexical keymap 提取 clipboard file 项并 intake，文本另处理。图片为本地 draft，发送时编码；其他文件即时调用 upload。provider 拒绝不等于入口消失。

ComposerBarInjected/Props 是公开导出的 TYPE，但 operations 注释为 package-private；InputBar 非公开 runtime export，正式包无其 src 文件。默认 apply 为有 Session 的 bar 总是提供 addFiles closure，没有只置空它的配置。内部 addFiles undefined 也只是使按钮 disabled，并不隐藏。

conversation.input.attachments 是 optional single slot，默认 ui-attachment 贡献 rail/document drop handler；不装它不移除 InputBar 的 picker/paste。公开 composer.bar/chain 可整体替换，不构成默认组件的附件专用开关，也不自动继承默认状态机。disabled/blocked/conversation.blocks 令输入整体 inert，不能作为仅关附件而保留文字提交的证据。

## Context 与 React

ScopedStandardSourceBinding 公开带 ctx；UiSession.materialize 构造 {key,ctx,hooks,keyedHooks,props}。renderer 将完整 binding 作为 SessionEntry/SessionMaybeEntryBody 的 React prop，也放进 Context.Provider value。标准业务 kit 仅 spread binding.props 和绑定 hooks，不能概括为每个业务组件都有顶层 ctx。

Cordis Proxy 对 symbol、prototype、then、数字字符串、下划线前缀放行；已有属性/accessor 分流。插件 Context 未知普通属性进入 inject 查找，可抛 without inject；root/no-runtime 分支不同。$$typeof 不在特殊列表，固定源码及4.0.2 JS未找到React探测特判或专用兼容设置。ctx.get(name,false)不是关闭第三方直接属性读取检查的全局开关。

上游确有 Node inspect symbol处理，但不等于React兼容。外部报告的React19开发记录器失败/生产成功未在本地验证；不由该上下文裁定React根因或生产完整兼容。

## 构建与验证边界

apps/web manifest 为 react/react-dom ^18.2.0，lock 的 importer 与 package snapshot 都锁18.3.1。shell seed共享react/jsx-runtime/react-dom等，Vite dedupe两个包；不是React19验证证据。

官方dev-web工作流最后运行vite build --watch，输出dist供dsh web；bare Vite serve被配置拒绝。插件tsdown默认NODE_ENV=production，可显式用development改变依赖条件和define；React本体来自shell共享表，插件环境本身不能证明shell最终React模式。未找到针对Context/$$typeof的构建豁免。

关键绝对证据：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/apply.ts:308`：默认addFiles注入。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/src/client/skeleton/InputBar.tsx:496`：picker。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/file-upload/src/client/runtime.ts:200`：fallback。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-session/src/client/index.ts:405`：binding.ctx。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-renderer/src/client/scoped-slots.tsx:618`：binding prop。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/cordis/src/reflect.ts:133`：Proxy读取。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/pnpm-lock.yaml`：React18.3.1。

Fixture仅读未运行：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/file-upload/tests/file-upload.client.spec.ts:255`（fixture fallback）、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-conversation/tests/input-bar.client.spec.tsx:317`（混合paste）、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/ui-renderer/tests/session-provider.client.spec.tsx:180`（cell传递）。相关范围未找到React19性能记录器专用fixture，未对任意调用方环境作验收。
