---
title: packages/spill — tool-output spill capability family
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/spill/README.md
  - packages/spill/spill/README.md
  - packages/spill/spill/src/index.ts
  - packages/spill/spill/src/types.ts
  - packages/spill/spill-local/README.md
  - packages/spill/spill-local/src/index.ts
  - packages/spill/spill-policy/README.md
  - docs/subsystems/spill.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

工具输出太大时，把**全文**落到后端存起来，把模型看到的结果换成「有界预览 + 不透明 locator + 取回提示」。三个包分别管：存哪儿（seam）、怎么存（本地文件）、什么时候换（post-execute 策略）。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包名 | npm | 一句话职责 |
|---|---|---|
| `spill` | `@deepseek-ai/dsh-spill` | Service Definition：`SpillStore` 抽象服务 + `SaveTextSpill`/`SpillRef`/`SpillLocator` 词汇 |
| `spill-local` | `@deepseek-ai/dsh-spill-local` | Provider：写进宿主文件系统上私有的、按 session 分组的文件 |
| `spill-policy` | `@deepseek-ai/dsh-spill-policy` | Consumer：`tools/post-execute` 变换器，决定何时把超限纯文本结果换掉 |

## 三件套结构

- **Service Definition**：`dsh-spill`，ctx key `ctx.spillStore`，`abstract class SpillStore extends Service` [T1: packages/spill/spill/src/index.ts#SpillStore]。
- **Service Provider**：`dsh-spill-local`，`class LocalSpillStore extends SpillStore` [T1: packages/spill/spill-local/src/index.ts#LocalSpillStore]。
- **Consumer**：`dsh-spill-policy`，**不注册任何服务**，只监听 `ctx.tools` 的 post-execute。

组 README 明确这是「mirrors the shell/fs seams」的同一套分工。

## 扩展点

想换存储介质（`spill://…` URI、对象存储、数据库 key、后端专属取回工具）：

1. 依赖 **`@deepseek-ai/dsh-spill`**，继承 `SpillStore` 实现 `saveText(input)`。策略插件一行都不用改。
2. `saveText` 必须**逐字**持久化 `input.content`，返回 `SpillRef`（`locator`、`bytes`、`retrievalHint`）。
3. **真实存储失败要 reject**（权限、ENOSPC、后端不可用）——由调用方决定怎么降级；策略插件把 reject 当作 best-effort 失败并保留内联结果。
4. `SpillLocator` 是 [branded](packages/util/brand) 类型，对模型渲染成不透明字符串。本地后端返回路径，远端后端返回 URI/key/命令 token 都行，policy 与 tool 消费方不受影响。
5. `retrievalHint` 是后端告诉模型「拿到这个 locator 之后该干嘛」的唯一渠道——seam 本身**没有取回/搜索 API**。

边界划得很硬（seam README 原文语气）：seam 只管存储，**没有** retention 策略（那是 `@deepseek-ai/dsh-output-retention` 的 `TextRetainer`）、**没有** tool-result 替换（那是 `dsh-spill-policy`）、**没有**取回 API。

`SpillOwner.sessionId` 是**保存时**的存储命名空间：fork 出来的 session 从 seed 日志里继承已有 locator，不复制也不重新归属；fork 之后产生的新 spill 用子 session id。`SpillSource` 记录 `toolName`/`callId`/`label`，只用于后端命名与排查，**不是访问控制**。

## Known Limitations

- `dsh-spill`：seam 无取回/删除 API，生命周期与访问语义都归后端；**存储不等于访问控制**——`SpillOwner` 只做写入命名空间，不授权对 locator 的读取，每个后端与取回消费方必须自己守边界。
- `dsh-spill-local`：本地文件**一直留着**直到外部清理——因为持久化、恢复、fork 出来的 session 都可能仍引用某个路径，所以后端故意不做 session 生命周期删除或按年龄的 retention；locator 要求**同机文件系统消费方**，远端/虚拟部署必须换后端。
- `dsh-spill-policy`：只有**最终纯文本结果**可 spill，混合内容结果、被 block 的反馈、`read` 一律放行；provider 更早做的截断或工具自己的 retention 在这里救不回来；**通知放不下时会禁用替换**——极小的 cap 或很长的 locator 会让超限原文留在内联，而后端此时**已经存了一份没人引用的 spill**。

## 陷阱

- **策略默认是关的**：`maxInlineBytes` 省略即整套策略停用（插件什么都不注册）。它是 UTF-8 字节数、非负整数、加载时校验。
- 预览与通知**共用同一份预算**：通知的字节成本会先从 `maxInlineBytes` 里扣掉，预览被压缩到刚好塞下，所以替换后的模型侧结果永不超 cap；也因此 spill 永远不会让上下文变大。
- 策略会跳过嵌套执行（`exec.parent` 存在）、被接受的值替换、`read`（避免 `read → spill → read again` 循环）、以及任何非 `accept` 的决策。
- 本地文件布局有安全意图，别自己拼路径去猜：`<root>/session-<hash>/<random>-<safeName>`。`root` 省略时是 OS temp 下**懒创建的 0700 私有目录**（可预测的 world-readable 根会让本地其他用户读到 spill 或种符号链接）；`session-<hash>` 是 `sha256(sessionId)` 短前缀；写入用 `open(path, 'wx', 0o600)`，任何已存在路径（含符号链接）都直接失败。
- 后端**可以**从 `suggestedName` 派生文件名，但**绝不能当路径信任**（seam README 原文强调）。

## 去哪深入

- 组结构、三包分工与 ctx key → `packages/spill/README.md`
- Service API 与词汇（`SaveTextSpill` / `SpillRef` / `SpillOwner` / `SpillSource`）→ `packages/spill/spill/README.md`、`packages/spill/spill/src/types.ts`
- 存储布局、`root` 配置、符号链接防护 → `packages/spill/spill-local/README.md`
- 触发条件、跳过规则、通知文案与预算算法 → `packages/spill/spill-policy/README.md`
- 子系统参考（`SaveTextSpill`、owners/sources、branded locator）→ `docs/subsystems/spill.md`
- 设计理由（为什么创建归 runtime seam 而不是模型侧 `write` 工具；存储 / retention / 工具自有输出处理的边界）→ `.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.md`
