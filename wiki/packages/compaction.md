---
title: packages/compaction — 压缩能力族
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/compaction/README.md
  - packages/compaction/compaction/README.md
  - packages/compaction/compaction-basic/README.md
  - packages/compaction/compaction-tool-result-pruner/README.md
  - packages/compaction/command-compact/README.md
  - packages/compaction/compaction/src/index.ts
  - packages/compaction/compaction/src/tool-pairing.ts
  - packages/compaction/compaction/src/checkpoint.ts
  - packages/compaction/compaction-basic/src/index.ts
  - packages/compaction/compaction-basic/src/config.ts
  - docs/subsystems/compaction.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

把过长的会话 surface 中一段较早的范围替换成一条摘要 checkpoint，从而降低下一次请求的历史 token；seam / 摘要 provider / 无模型剪枝伴生服务 / 人类 `/compact` 命令四件套。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。组内 4 个包全部标注为 product。

## 包清单

| 包目录 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `compaction/` | `@deepseek-ai/dsh-compaction` | `ctx.compaction` | Service Definition：抽象服务 + `compaction/*` 事件 + `CompactionResult` + checkpoint source 构造器 + tool-pairing 边界助手 |
| `compaction-basic/` | `@deepseek-ai/dsh-compaction-basic` | 注册 `ctx.compaction` | Service Provider：`ctx.tokenMeter` 压力判定 + token 预算保留 + `llm.stream()` 摘要 |
| `compaction-tool-result-pruner/` | `@deepseek-ai/dsh-compaction-tool-result-pruner` | `ctx.toolResultPruner` | 可选的无模型 tool-result 剪枝（头/marker/尾） |
| `command-compact/` | `@deepseek-ai/dsh-command-compact` | 注册到 `ctx.commands` | Consumer：人类 `/compact` 命令 |

## 三件套结构

- **Service Definition**：`@deepseek-ai/dsh-compaction`。`abstract class CompactionEngine extends Service` [T1: packages/compaction/compaction/src/index.ts]，三个方法 `compactIfNeeded` / `compactNow` / `compactRegion` **全部抽象**，触发策略、保留策略、事件顺序、摘要方式都归 provider。
- **Service Provider**：`@deepseek-ai/dsh-compaction-basic`（`class BasicCompactionEngine extends CompactionEngine`）。
- **Consumer**：`@deepseek-ai/dsh-command-compact`（人类命令，非 model tool）。
- **伴生（第四角色）**：`compaction-tool-result-pruner` 既不是 backend 也不是 model tool，compact-basic 用 `ctx.get('toolResultPruner')` 可选读取，两个包互相独立可组合。

## 扩展点

- **要换摘要实现**：依赖 `@deepseek-ai/dsh-compaction`（Service Definition），`subclass CompactionEngine` 并作为插件加载即注册为 `ctx.compaction`。必须用 `compactCheckpointSource(compactionId, sourceCommandId?)` 构造替换消息的 source；`isCompactCheckpointSource()` 供持久化/克隆之后识别，不依赖 backend 身份。
- **只想换摘要文本/模型**：继承 `BasicCompactionEngine`，覆盖唯一的 protected 钩子 `summarize()`；压力测量、保留、shrink 校验、token 记账仍由基类和 `ctx.tokenMeter` 负责。返回 `{ summary, rawOutput?, llmStreamCall?, provider, model, maxTokens?, usage? }`。
- **想在边界上做安全切分**：Service Definition 导出 `toolPairingBalancedBefore(session, seq)` / `toolPairingBalancedAfter(session, seq)`（`src/tool-pairing.ts`）。
- **想拦截摘要请求**：backend 的摘要是直接的 `ctx.llm.stream()` 调用（不是 loop step），拦截点是 `llm/stream`，不是 `agent/request`。
- **客户端/wire 侧识别 checkpoint**：只能 import 子路径 `@deepseek-ai/dsh-compaction/checkpoint`（`src/checkpoint.ts`），不能 import 包根。
- **token 计量不属于本 seam**：`ctx.tokenMeter` 是 `llm/` 组的独立服务。

## Known Limitations

- 只有人类命令，**没有 model-facing 的压缩工具**。
- 单个不可分割单元的溢出不在契约内：大的非 tool 节点、剩余部分仍超窗的 tool 单元都压不了；只有「文本型 tool result 是可移除主体」时 pruner 能救。
- 只压缩派生历史，**永远不压缩 system prompt / tools / session prefix**。
- compaction-basic：meter 是固定启发式（无 usage 时按字符数 + 结构开销）；溢出分类靠 adapter 维护（两个 DeepSeek adapter 把已识别的 context-limit 失败归一为 `CONTEXT_WINDOW_EXCEEDED`）；摘要失败时保留最新的持久 surface 并带满历史继续。
- pruner：字符预算不是 token 预算；剪枝是纯语法的（保头保尾）；code-point 切分保护代理对但会切开 grapheme cluster。
- command-compact：仅 idle 可用（有 turn 或已接受的唤醒 prompt 时报 `busy`，命令本身不排队）；无范围/策略参数；只有装了 `ctx.commands` 的界面能调。

## 陷阱

- **`compactRegion(start, end)` 的 `[start, end]` 是 surface 位置区间，不是数值 seq 区间**。一次 replace 会把高 seq 的摘要节点落在被遮蔽范围的位置上，此后 surface 顺序不再跟随 seq 顺序。
- **锁是「未匹配的 `compaction/start`」这条日志记录**，不是 WeakSet / mutex / 客户端锚点。崩溃留下孤立 start 即 `busy`；判定 stale 的边界是 `session/end-seed`——比最新 end-seed 更早的未匹配 start 视为上个进程生命周期的陈旧证据，不阻塞。
- **`compaction/*` 事件永远不能出现在 surface 上**：`SurfaceEventType` 是闭合联合，只有 `user/message`/`assistant/message`/`tool/result` 能带 `surfaceOp`。整个事务里唯一的 surface 变更是第 4 步那条带 `surfaceOp: { op: 'replace', start, end }` 的 `user/message`，且它在锁 bracket **内部**。
- **`ManualCompactionError.code` 是闭集** `busy | changed | summary | commit | persistence`。`changed`/`summary` 表示 surface 没被替换但失败尝试仍写进了日志；`commit` 对是否部分变更刻意保持中立。
- **`compactNow` 用 `turn: null` 且不需要 open turn，但 compaction-basic 的 `compactRegion` 需要 open turn**——在完全关闭的 session 上手动调 `compactRegion` 会抛 "no open turn"。
- **本 Service Definition 故意依赖 `dsh-session` 和 `dsh-llm`**，违反「Service Definition 只依赖 cordis」的通用纪律（契约动词定义在 `Session` 上、输出是 `ContentBlock` 词汇表）。这是被 Agent Note 记录的有意偏离，不要当成坏味道去"修"。
- `maxOverflowRetries: 0` 只禁用溢出恢复（不禁用普通压力压缩）；`auto: false` 才是纯手动。
- 摘要调用会把会话的 system prompt / tools / 被遮蔽区间消息**逐字重放**以复用 provider 的 warm prefix cache——把摘要路由到别的 provider/model，或压缩非 head 区间，就放弃了这份复用。

## 去哪深入（文件路由）

| 想知道什么 | 去哪 |
|---|---|
| 事务 5 步、surface 契约、锁语义、`ManualCompactionError` | `packages/compaction/compaction/README.md` |
| 默认配置项与默认值（`thresholdRatio` 0.8 / `retainRatio` 0.16 / `maxTokens` 8192 / `compactionRetries` 1 / `maxOverflowRetries` 1 / `auto` true）、`modelPolicies` | `packages/compaction/compaction-basic/README.md` |
| 摘要 prompt 全文与 checkpoint preamble 原文 | `packages/compaction/compaction-basic/README.md`（Model Experience 段） |
| 剪枝默认值（`thresholdChars` 8192 / `headChars` 4096 / `tailChars` 1024）与 marker 文本 | `packages/compaction/compaction-tool-result-pruner/README.md` |
| `/compact` 三种输入的确切回复与错误码映射 | `packages/compaction/command-compact/README.md` |
| `compaction/*` 事件载荷、`CompactionResult` 字段 | `docs/subsystems/compaction.md`、`docs/persistence-catalog.md` |
| 为何 seam 允许依赖 session/llm | `.agents/notes/implemented/feature/2026-06-18-compaction-capability-seam.md` |
| 可跑的压缩示例配置 | `examples/headless-agent/`（`compaction.cordis.snapshot.yml`） |
