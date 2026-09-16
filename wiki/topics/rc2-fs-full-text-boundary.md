---
title: rc.2 FileSystem 全文与工具窗口边界
description: 区分 Provider 全文、流式读取、diff basis 与 consumer 截断。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-16
updated: 2026-09-16
asked_by: agent
anchors:
  - packages/fs/fs/src/types.ts#FsWriteOutcome
  - packages/fs/fs/src/types.ts#FsEditOutcome
  - packages/fs/fs/src/index.ts#FileSystem
  - packages/fs/tool-fs/src/read-render.ts#buildWindow
  - packages/fs/tool-fs/src/write.ts
  - packages/fs/tool-fs/src/edit.ts
  - packages/fs/fs-local/src/index.ts
---

条件：跨进程 FileSystem Provider · 任意载体 · 官方 tool-fs/observation-policy · 文件系统路径由 Provider 所有 · 无模型调用 · 固定 rc.2。仅静态源码核验。

readText 返回整份 UTF-8 正文；streamText 返回同一完整正文的decoded chunks，Provider负责跨chunk解码/binary拒绝。没有公开offset/range参数。readBytes的maxBytes约束完整内容，超限FS_TOO_LARGE而非截断。

工具read根据size（未知/达到streamMinSize时streamText）读取，再由buildWindow应用offset/limit/maxLineLength/maxBytes。buildWindow仍消费全流以计算totalLines。Provider静默返回前缀或窗口会被当作完整文件，改变行号/EOF/totalLines，且不由observation-policy检测。

FsWriteOutcome.before是完整LF-normalized文本或null；null允许新建或后端拒绝contextual diff basis（如binary、超basis limit）。after仍为完整写后正文。FsEditOutcome两侧均为文本，无null/partial例外。截断字符串不是合法的null降级。

tool-fs用before/after计算presentationMeta的computeHunkDiffs；模型text为确认语句，不等于全量diff。故截断直接破坏typed output、结果diff及UI语义，但不一定改变那句模型确认文字。fs-observed/version guard不裁剪文本也不验证传输完整性。

证据：[公开outcome](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/fs/fs/src/types.ts)、[read合同](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/fs/fs/src/index.ts)、[工具窗口](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/fs/tool-fs/src/read-render.ts)、[write消费](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/fs/tool-fs/src/write.ts)、[edit消费](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/fs/tool-fs/src/edit.ts)。全部按本页SHA读取，未执行文件修改实测。
