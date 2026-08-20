---
title: packages/fs — 文件系统 capability family（四层拆分的教科书样本）
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/fs/README.md
  - packages/fs/fs/README.md
  - packages/fs/fs/src/types.ts
  - packages/fs/fs-local/README.md
  - packages/fs/fs-sandbox/README.md
  - packages/fs/fs-observation-policy/README.md
  - packages/fs/tool-fs/README.md
  - packages/fs/tool-fs-search/README.md
  - packages/fs/tool-str-replace-editor/README.md
  - docs/subsystems/filesystem.md
  - docs/capability-seams.md
  - .agents/notes/implemented/architecture/2026-06-17-filesystem-capability-seam.md
  - .agents/notes/implemented/simplification/2026-06-26-fsspec-style-fs-seam.md
  - .agents/notes/implemented/architecture/2026-06-26-file-context-as-event-gate.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

DSH 里 **capability seam 拆分做得最彻底的一组**：provider 契约（`ctx.fs`）/ 本地实现 / 策略门（纯事件，无服务）/ model-facing 工具四层各自可换；外加一套**不走 `ctx.fs`**、用打包 ripgrep 进程实现的发现工具。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文，全组 product）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `fs/` | `@deepseek-ai/dsh-fs` | **Service Definition**：`ctx.fs` 的 12 个 primitive + `fs/*` 事件词汇表 |
| `fs-local/` | `@deepseek-ai/dsh-fs-local` | 宿主文件系统实现（原子写、per-target mutation lock、Windows DACL 保留） |
| `fs-sandbox/` | `@deepseek-ai/dsh-fs-sandbox` | 继承 `fs-local`，按 per-call sandbox mode + workspace root 围栏 write/edit；读永远放行 |
| `fs-observation-policy/` | `@deepseek-ai/dsh-fs-observation-policy` | 策略门：observed-state + read-before-edit + 版本守卫。**不注册任何服务** |
| `tool-fs/` | `@deepseek-ai/dsh-tool-fs` | `read` / `read_image` / `write` / `edit` 工具**兼** executor；拥有 read windowing，dispatch `fs/*` |
| `tool-fs-search/` | `@deepseek-ai/dsh-tool-fs-search` | `glob` / `grep`，走打包的 `@vscode/ripgrep` + `ctx.subprocess`，**不走 `ctx.fs`** |
| `tool-str-replace-editor/` | `@deepseek-ai/dsh-tool-str-replace-editor` | 独立 `str_replace_editor` 工具（`view`/`create`/`str_replace`/`insert`） |

组外还有一个 `ctx.fs` 实现：`packages/e2b/fs-e2b`（远端执行世界，与 E2B subprocess provider 共享 `ctx.e2b`）。

## 三件套结构

`docs/capability-seams.md` 把 `ctx.fs` 明确标为 Role `seam`：

- **Service Definition**：`@deepseek-ai/dsh-fs`（目录 `packages/fs/fs/`）。抽象类 `FileSystem`，12 个 primitive：`resolve` / `processPath` / `fileUrl` / `contains` / `stat` / `lstat` / `readText` / `streamText` / `readBytes` / `listDir` / `writeText` / `editText`。它同时**拥有 `fs/*` 事件词汇表**，让 emitter 与 policy listener 共享语言而不互相依赖。
- **Service Provider**：`fs-local`、`fs-sandbox`（继承前者）、`fs-e2b`。三选一注册 `ctx.fs`。
- **Consumer**：`tool-fs`（直接读写 `ctx.fs`）、`tool-str-replace-editor`。
- **Companion plugin（第四层，事件门）**：`fs-observation-policy`。它**不是 seam**，是「不该放在 `FileSystem` 基类上的策略」。

`tool-fs-search` 不属于这个 seam：它故意**不 inject `fs`**，只 inject `tools` / `systemPrompt` / `subprocess`，`ctx.spillStore` 用 `ctx.get()` 机会性读取。

## 扩展点

**要换文件系统后端** → 继承 `FileSystem`（来自 `@deepseek-ai/dsh-fs`）实现 12 个 primitive 并注册 `ctx.fs`。不需要动 policy 层，也不需要动工具 schema。

**要加访问策略** → 有两条路，选错就白写：
- **文件语义级**（先读后改、版本守卫、观察态）→ 挂 `fs/*` 事件。三个事件（由 `dsh-fs` 声明、由 `dsh-tool-fs` dispatch）：
  - `fs/write-intent` — **single-slot decision waterfall**，listener 完全决策、**不调 `next()`**
  - `fs/edit-intent` — 同上
  - `fs/observed` — fire-and-forget 记录事件，载荷是 `FsObservation` 判别联合（`{kind:'present',version}` 或 `{kind:'absent'}`）
- **可叠加的授权/审计/沙箱链** → 放 `tools/execute` waterfall，**不要**放 `fs/*`。README 明确说 `fs/*` 不是 composable authorization chain。

**关键设计后果**：`fs/write-intent` / `fs/edit-intent` 的 `expected` 参数在 provider 契约上是**可选的**，所以裸 `ctx.fs` 本身就是一个完整、无约束的存储 seam；卸掉 policy 插件不会在服务注入边界上打断 `tool-fs`——工具优雅地退回无约束 provider（无条件覆盖/编辑、无观察态）。这就是选事件门而非强制方法服务的全部理由。

**`editText` 为什么不能在 policy 层用「读 + 写」组合出来**：版本守卫 + 字面匹配 + 原子重写必须待在同一个临界区里，才能保证错误归因正确和 one-wins/one-stale 并发语义；远端后端还可能用原生 compare-and-edit 实现。

## Known Limitations

组 README 无该章节；以下为跨包综合。

**Service Definition（`fs/`）**
- **契约层面只支持文本变更**：文本读与两种变更都用 `FS_NOT_TEXT` 拒绝二进制/非 UTF-8；`readBytes` 是唯一的原始字节 primitive。
- **只有 12 个 primitive**：没有 delete / rename / move / copy / watch；`listDir` 只列一层，递归、glob、分页、搜索都在范围外。
- **无 IO 截止时间**（见「陷阱」）。
- resolve-then-operate 对远端后端意味着每次工具调用两次往返。

**fs-local**：`config.cwd` **不是沙箱**（绝对路径与 `..` 都能逃逸，它只是解析默认值）；version token 依赖 dev/inode/size/mtimeNs/ctimeNs 元数据；`editText` 把整个文件（外加编辑后的副本）读进内存；二进制检测不对称——**读只采样前 8192 字节的 NUL，编辑扫全 buffer**，所以「晚出现 NUL 的文件读得动但编辑被拒」；per-target mutation lock 只在进程内；`createIfAbsent` 需要文件系统支持硬链接。

**fs-observation-policy**：观察态**不跨 session resume**（`WeakMap`，恢复后必须重读）；无 agent session 的调用方永远满足不了策略；**直接调 `ctx.fs` 读不会发 `fs/observed`**；授权判据是「版本新鲜度」而非「视图完整性」——任何一次窗口读都授权对未变更文件做整文件覆盖。

**fs-sandbox**：策略围栏而非内核边界，残余 TOCTOU 被「写前重新规范化」收窄但未消除；可写集来自 `writableRoots`（与 Seatbelt profile 共用同一个函数）；**必须同时组合 `ctx.sandboxPolicy`**，否则后端不围栏。

**tool-fs**：不发布 model-facing 目录列举；`read` 只处理 UTF-8 文本（图片走 `read_image`，PDF/音视频延后）；`read_image` 的路由门与并发换模型有竞态；工具结果卡片无内联图片预览。

**tool-fs-search**：搜索与文件访问**没有共享 workspace 的运行时证明**（返回路径只在 workdir 与 `read` 根同 workspace 时才可跟进读取，包内不做跨服务校验）；ripgrep 版本被依赖锁定；只暴露一页有界结果；开启 sampling 时只按搜索根下第一层路径段分组。

**tool-str-replace-editor**：只支持 UTF-8 文本；`str_replace` 故意拒绝零匹配与多匹配且没有 `replace_all`。

## 陷阱

1. **文件 IO 一律无超时，这是刻意的**。`read`/`write`/`edit` **不接受** `timeoutMs`，provider 契约也不 arm 任何 deadline，也不声明 `timeout-policy` 预算——理由是「deadline 会杀掉 OS 仍会完成的工作」。取消只能靠 `exec.signal` 尽力而为地在 syscall 边界中止。**注意 `tool-fs-search` 是例外**：它声明 `timeoutMs`（默认 30000）+ `graceMs`（3000），由 `@deepseek-ai/dsh-tool-call-timeout-policy` 经 `exec.signal` 协作执行，subprocess seam 的 terminate 升级才是硬杀。
2. **组 README 的包表漏了 `tool-str-replace-editor/`**——而 `packages/README.md` 第 9 行明写「Group READMEs own package/ctx-key maps」。目录与 README 不符，见 conflicts。
3. **`fs/write-intent` / `fs/edit-intent` 的单槽是「注册顺序先到先得」的约定，不是事件强制的不变式**。先注册或 `prepend` 的 decider 会赢。默认部署里由 `fs-observation-policy` 占据只是惯例。
4. **`FsTargetKey` / `FsVersion` 是 branded 不透明 id**，消费者不得解析 `targetKey` 或解释 `version`；只有 `displayPath` 才是给模型/UI 看的。
5. **`glob`/`grep` 不是文件系统能力**。它们 spawn 打包的 rg（argv 里预置 `--no-config`，防止宿主 `RIPGREP_CONFIG_PATH` 注入 `--pre` 预处理器）。所以远端/虚拟文件系统后端下搜索结果**可能指向另一个世界的路径**。
6. **`sampleOverCapGlobResults` 是必填且无 fallback** 的 config——部署必须显式选择超额排序契约。
7. **`fs/observed` 的 listener 契约上必须是同步、纯副作用的**。工具不 guard 这次 `ctx.emit`，listener 抛异常会直接变成工具的 `isError` 结果。
8. **`read_image` 只在挂了 `ctx.attachments` 时才注册**，且执行时还要求当前路由模型声明 `image` 输入，否则在任何 IO 之前就返回拒绝。

## 去哪深入（文件路由）

| 问题 | 去哪 |
|---|---|
| 四层是什么、谁能换谁 | `packages/fs/README.md` |
| 12 个 primitive 的逐条语义、`FsErrorCode` 全表 | `packages/fs/fs/README.md`、`packages/fs/fs/src/types.ts` |
| 目标、结果、guard、policy 事件、错误分类、为何无超时 | `docs/subsystems/filesystem.md` |
| 本地实现细节（原子写、DACL、CRLF 还原、abort 时机） | `packages/fs/fs-local/README.md` |
| 三个 `fs/*` 事件的决策表 | `packages/fs/fs-observation-policy/README.md` |
| 工具 config（`readLimit`/`readMaxBytes`/`readStreamMinSize`）与 canonical 结果 | `packages/fs/tool-fs/README.md` |
| 搜索的 8 个 config、两套预算两个 artifact、`SearchError` 码 | `packages/fs/tool-fs-search/README.md` |
| sandbox 围栏的威胁模型 | `packages/fs/fs-sandbox/README.md` |
| seam 拆分的决策源头 | `.agents/notes/implemented/architecture/2026-06-17-filesystem-capability-seam.md`、`.agents/notes/implemented/simplification/2026-06-26-fsspec-style-fs-seam.md`、`.agents/notes/implemented/architecture/2026-06-26-file-context-as-event-gate.md` |
