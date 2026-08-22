---
title: packages/lsp — LSP capability family
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/lsp/README.md
  - packages/lsp/lsp/README.md
  - packages/lsp/lsp/src/index.ts
  - packages/lsp/lsp-stdio/README.md
  - packages/lsp/tool-lsp/README.md
  - docs/subsystems/lsp.md
  - .agents/notes/implemented/architecture/2026-07-15-lsp-capability-seam.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

语义代码导航能力 seam：**恰好四个操作**（`goToDefinition` / `findReferences` / `goToImplementation` / `hover`），**没有通用 JSON-RPC 逃生口**，所以换 provider 不改变模型的提问方式，也没有任何协议 payload 或未经审查的 mutation 能到达模型契约 [T1: packages/lsp/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

| 包名 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `lsp` | `@deepseek-ai/dsh-lsp` | `ctx.lsp` | Service Definition：按 branded id + 扩展名映射的 provider 注册表、逐查询选择、词汇表、`LspError` 分类 |
| `lsp-stdio` | `@deepseek-ai/dsh-lsp-stdio` | 注册 provider 到 `ctx.lsp` | 通用多服务器 stdio 后端，跑在 `ctx.fs` + `ctx.subprocess` 之上（JSON-RPC、transient-open） |
| `tool-lsp` | `@deepseek-ai/dsh-tool-lsp` | 注册到 `ctx.tools` | 模型侧 `lsp` 工具（四操作、**一基（1-based）UTF-16** 光标坐标） |

## 三件套结构

组 README 与 `lsp/lsp/README.md` 都显式列了三件套表格，这是全仓最标准的一个 seam 样板：

- **Service Definition**：`lsp/lsp/`（`Lsp extends Service implements LspService`）
- **Service Provider**：`lsp-stdio`
- **Consumer**：`tool-lsp`

原则原文：**「Providers register capabilities, not tools」**——`tool-lsp` 是**唯一**拥有模型可见的名称、描述、prompt guidance、schema、呈现方式的包。

## 扩展点

**要接一个新的语言服务后端，依赖 `@deepseek-ai/dsh-lsp`（Service Definition），不要依赖 `lsp-stdio`。**

- `ctx.lsp.registerProvider(provider)`：原子地保留 branded `id` 与**每一个规范化扩展名**。任何非法输入或冲突**什么都不发布**并抛 `LspError`（`LSP_INVALID_PROVIDER` / `LSP_CONFLICT`）。返回的 disposer 释放全部保留，随调用方 fiber 释放。
- `ctx.lsp.query(request, signal?)`：按文件**最后一个扩展名**选 provider，从该 provider 的映射推出 `languageId`，跑一次查询。无匹配抛 `LSP_UNAVAILABLE`。
- 选择是**逐查询且与顺序无关**的：provider 独占拥有一组扩展名，所以注册顺序和 HMR 顺序永远不影响路由。扩展名 key 规范化为小写、带前导点形式；`languageId` **只用于同步 transient document，不参与选择**。
- 想复用 stdio 主机而不写代码 → 挂 `lsp-stdio` 并在 `servers` 里配一张表（key 就是在 `ctx.lsp` 上保留的 provider id），每个条目必填 `command` 与 `extensionToLanguage`。它明确**不是**语言服务器目录或安装器，预设应放在 `cordis.yml` overlay 里。
- 想改模型侧呈现/上限 → 只动 `tool-lsp` 的 `Config`（`maxLocations` 默认 `100`、`maxResultChars` 默认 `16000`、`timeoutMs` 默认 `60000`）。这三项**都不对模型可见**。

## Known Limitations

- `lsp`（seam）：**一个 runtime 内扩展名独占**——两个 provider 不能都声明 `.ts`，哪怕 language id 不同，重叠即注册失败（预期的扩展方向是「注册之上的、部署配置的 selector」，它可以放宽独占而不把 provider 选择塞进模型输入）；**只有四个操作**（symbols 与 call hierarchy 因 schema 不同而推迟；diagnostics 需要独立的新鲜度/累积规则；rename / code action / formatting 这类 mutation 需要带 preview、权限、写策略的独立工具）；**没有观察 API**——可用性只能靠跑 `query()` 再看抛出的 `LspError` code 判断，没有 provider 变更事件也没有能力状态查询。
- `lsp-stdio`：**没有 confinement policy**（信任配置的服务器，不做沙箱）；**transient-open 兼容性下限**——同步方式省略 open/close（或宣告 `None`）的服务器不受支持，即使闭文档查询本来能用；**per-server/workspace 串行化延迟**（共享同一 server + workspace 的并行 agent 排在同一进程后面）；**硬杀 harness 会遗留孤儿语言服务器**（`initialize.processId: null` 关掉了服务端的 client-PID 监控，只有优雅 disposal 才清理）。
- `tool-lsp`：**UTF-16 光标坐标**对模型来说难数（尤其非 BMP 字符），偏离符号的位置可能返回空；**不承诺跨服务器的完整性**（取决于索引就绪程度）。

## 陷阱

1. **两套坐标系并存，边界在 `tool-lsp`**。seam 层（`LspQueryRequest`）的 position/range 是**零基 UTF-16**，与协议一致；**一基（1-based）光标约定归工具所有**。`tool-lsp` 负责双向转换。读源码时先确认自己在哪一层。
2. **`findReferences` 总是包含声明**，且这是 **provider 内部强制**的，调用方**没有开关**——理由是影响分析不该漏掉定义点。
3. **`LspQueryResult` 是一个 CLOSED discriminated union**：`{ kind: 'locations', locations, resolvedWorkspaceUri }` 和 `{ kind: 'hover', hover }`。消费者用 `switch` 穷尽，新增分支会**编译失败**直到处理。
4. **不要拿 session cwd 去做路径相对化**。要用 provider 返回的 `resolvedWorkspaceUri`（provider 规范化后的 workspace `file:` URI），因为请求根路径可能是符号链接。`tool-lsp` 的 Native 渲染就是照这条做的。
5. **`LspQueryRequest` 每个字段都必填**，所以**没有 `resolve()` 步骤**——这与 `dsh-shell` 那种 request/spec 分离模板不同，是个刻意的差异。
6. **`lsp-stdio` 不发 `fs/observed`**：因为只有 LSP 结果是模型可见的，所以一次查询**不满足** read-before-write 策略。想靠 lsp 查询来「算作读过文件」是错的。
7. **`lsp-stdio` 的 env 会被凭据擦洗**：匹配 `KEY`/`PASSWORD`/`SECRET`/`TOKEN` 的变量不转发；显式的 `DSH_*` 条目在 seam 擦洗环境变量之后再 merge。
8. **一个坏条目会连累全部**：所有可执行文件在 load 时（凭据擦洗后）解析，**靠后的一个坏条目会导致每一个 provider 都注册不上**。进程本身是懒启动的（首次匹配查询时才拉起）。
9. **服务器返回的 capabilities 是权威的**：不支持的操作、或同步方式没有 transient open/close，查询直接失败。省略 `positionEncoding` 默认 `utf-16`，其他值算协议错误。客户端**拒绝 `workspace/applyEdit`**，从不应用编辑、从不执行 command。
10. **`tool-lsp` 强制要求 workspace root 来自 session `header.cwd`，没有 fallback**：缺失时以 `LSP_WORKSPACE_REQUIRED` 在查询前失败。
11. **`maxLocations` / `maxResultChars` 只影响 Native/模型呈现，不影响 canonical value**——Code Mode 能拿到完整的、零基 range 的全部结果。
12. **一次超时预算覆盖整条链路**：`timeoutMs` 由 `dsh-tool-call-timeout-policy` 执行，覆盖排队的 open/query/close 完整生命周期，且**不可由模型配置**。
13. **必须同世界组合**：`lsp-stdio` 要求 filesystem 与 subprocess provider 挂在同一个执行世界，split-world composition 是非法的。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 操作、坐标、请求/结果、`LspError` 权威定义 | `docs/subsystems/lsp.md` |
| seam API 两个方法的精确语义与冲突规则 | `packages/lsp/lsp/README.md`、`packages/lsp/lsp/src/index.ts`（`Lsp`、`LspError`） |
| 完整类型契约 | `packages/lsp/lsp/src/types.ts` |
| stdio 服务器表全部配置键与默认值 | `packages/lsp/lsp-stdio/README.md#Configuration` |
| JSON-RPC 帧、连接、进程实例、结果翻译 | `packages/lsp/lsp-stdio/src/framing.ts`、`connection.ts`、`instance.ts`、`translate.ts` |
| 安全边界（信任模型、containment、外部路径规则） | `packages/lsp/lsp-stdio/README.md#Security boundary` |
| `lsp` 工具的提示词原文、渲染与 UI card | `packages/lsp/tool-lsp/README.md`、`src/render.ts`、`src/session-cwd.ts` |
| 生成的 `lsp` schema | `docs/tool-catalog.md` |
| 为何 transient open、为何扩展名独占、为何不给逃生口 | `.agents/notes/implemented/architecture/2026-07-15-lsp-capability-seam.md` |
