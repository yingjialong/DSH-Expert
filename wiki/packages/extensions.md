---
title: packages/extensions — Agent 自改运行时（self-referential Cordis toolset）
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/extensions/README.md
  - packages/extensions/tool-cordis/README.md
  - packages/extensions/cordis-host-runner/README.md
  - packages/extensions/cordis-client-runner/README.md
  - packages/extensions/ui-cordis/README.md
  - packages/extensions/cordis-host-runner/src/index.ts
  - packages/extensions/cordis-host-runner/src/inspect-registry.ts
  - packages/extensions/cordis-client-runner/src/client/slot-catalog.ts
  - docs/subsystems/extensions.md
  - docs/capability-seams.md
  - .agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

让 Agent 在自己正在运行的那个 DSH 进程里 **读取运行时（服务/插件/工具/事件）并动态定义、挂载、卸载自己写的 Cordis package**；host 半在 `node:vm` 里跑，browser 半推给打开的网页。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `tool-cordis/` | `@deepseek-ai/dsh-tool-cordis` | 5 个 model-facing 工具：`cordis_inspect` / `cordis_define` / `cordis_run` / `cordis_stop` / `cordis_undefine` |
| `cordis-host-runner/` | `@deepseek-ai/dsh-cordis-host-runner` | definition 注册表 + `node:vm` 沙箱 + run 往返；提供 `ctx.dynamicCordisRunner` **和** `ctx.cordisInspect` |
| `cordis-client-runner/` | `@deepseek-ai/dsh-cordis-client-runner` | dual-half 包的浏览器半：把 definition 求值成活的 browser plugin，应答 run 请求 |
| `ui-cordis/` | `@deepseek-ai/dsh-client-ui-cordis` | 浏览器面板（frame-wide `shell.overlay`）+ 只读 `cordis_define` 卡片 |

注意 npm 名与目录名不一致：`ui-cordis` → `dsh-client-ui-cordis`。

## 三件套结构

这一组**不是** capability seam（`docs/capability-seams.md` 里两个服务的 Role 都是 `core`，不是 `seam`），没有可换 Provider 的 Service Definition。它的形状是「服务 + Consumer + 双半」：

- **服务（core，非 seam）**：`cordis-host-runner` 独家提供 `ctx.dynamicCordisRunner`（注册表 / vm / run 往返）与 `ctx.cordisInspect`（inspect provider 注册与跨页路由）。没有替代实现。
- **Consumer（model-facing）**：`tool-cordis`，`inject` 的是 `tools` + `dynamicCordisRunner`；只装工具不装 runner 的组合永远激活不了这些工具。
- **浏览器半**：`cordis-client-runner`（执行）+ `ui-cordis`（人机界面）。两个 client 包放在本组而不是 `packages/client/`，因为它们是本子系统 dual-half 包的另一半；host aggregate 特意排除它们，让两个 face 各有自己的 compiler program。

## 扩展点

- 想给 **模型**增加运行时自省能力：依赖 `@deepseek-ai/dsh-cordis-host-runner` 的 `ctx.cordisInspect` 注册 inspect provider（`src/inspect-registry.ts`），不要去改 `tool-cordis`。
- 想让 **动态 package 的浏览器半能占位**：扩展点其实是 `SlotMap` 声明合并 + `slots.register`。可占的 seat 由生成器 `scripts/gen-client-catalog.ts` 扫描全仓后写成 `packages/extensions/cordis-client-runner/src/client/slot-catalog.ts`，`pnpm run verify-client-catalog` 卡新鲜度。**改模型读到的 seat 说明文字 = 改声明处那个包的 JSDoc**，不是改 catalog。
- 动态 package 的 host 半用 `harness.handle` 注册方法，browser 半用 `host.call` 经 `dynamicCordisRunner` remote namespace 的 `invoke` 调回来；只有 browser→host 一个方向。
- `ctx` façade **不暴露 `effect()`**：package 代码只能靠 `on` / `provide` / `tools.register` 做清理。
- 四个转发事件由本组在 `cordis-host-runner/src/types.ts` 声明、由 `@deepseek-ai/dsh-api-remotes` 放行，浏览器经 `ctx.remote.$on` 收：`cordis/request-run`、`cordis/request-run-resolved`、`dynamicCordisRunner/package`、`dynamicCordisRunner/retract`。后两个是对称的启停广播。

## Known Limitations

组 README 无该章节，以下汇总自四个包的 `## Known Limitations and Deferred Work`：

- **run 返回 ok ≠ UI 渲染成功**：`run` 在应答页面「加载完」browser 半就返回，React 渲染在其后；崩溃只能通过 `reportRenderFailure` 记录、再用 `cordis_inspect what:"temporary"` 读回。
- 带 browser 半的包在**没有页面连接时会一直挂起**（headless / ACP 无法用），且**没有超时**——只能靠发起 turn 被取消结束。
- `vmTimeoutMs`（默认 5000）只约束**同步**求值，async body 逃逸。
- `runHostHalf` 不带 request id，host 侧按「该 definition 最近一次 armed 的请求」归因；同一 definition 并发多个 run 请求会失准。
- 应答里若带已被取代的 revision → `accepted: false` 且请求仍挂起；browser 半不读这个 ack，所以不会重试。
- 沙箱是「对诚实代码的隔离」，不是安全边界（见下）。
- ui-cordis：`cordis_define` 与「未运行时的 undefine」不广播，开着的面板要关掉重开才刷新；任何页面都能应答任何请求（frame-wide 批准，首答者胜）；会话过长导致 call head 出窗后卡片丢失 label。
- `zod` 是生成的 TypeRT face 的运行时依赖而非 `src` 的依赖，必须在 package.json 声明并在 `knip.json` 忽略。

## 陷阱

1. **`ctx.dynamic` 是文档笔误**。`tool-cordis/README.md` 开头写 runner 是 `(ctx.dynamic)`，真实 service key 是 `dynamicCordisRunner`（`cordis-host-runner/src/index.ts` 里 `super(ctx, 'dynamicCordisRunner')`，Context 合并声明也是这个名）。按 `ctx.dynamic` 去 inject 会拿不到。
2. **组 README 的 ctx key 列漏了 `ctx.cordisInspect`**。`cordis-host-runner` 实际提供两个服务。
3. **tool-cordis README 引用的两个文件不存在**：`src/client-catalog.ts`（真实产物在 `cordis-client-runner/src/client/slot-catalog.ts`）、`src/curation.ts`（`reach` 分类实际在 `packages/typert/generator/src/cordis-catalog.ts`）。只有 `src/api-catalog.ts` 与 `src/inspect.ts` 真实存在。
4. **仓库根 `AGENTS.md` / `CLAUDE.md` 的目录清单把这一组叫 `self-modification/`**，实际目录是 `packages/extensions/`。
5. **沙箱不是安全边界**：Node 全局虽被移除或重定向到 `ctx.fs` / `ctx.web` / `ctx.bash`，但 host-realm helper 可达，代码能逃回 Node。挂这个插件的谨慎程度应等同于给 bash 权限。
6. **一切只在进程内存里**：session log 只记 define 调用的 metadata，**从不记代码**。重启后 definition 全没，卡片会如实说「id 解析不到」。想留下实验，得走正常插件开发流程。
7. **verb 是 session-scoped，inventory 是全局的**：别的 session 的 definition 读起来是「不存在」而非「禁止」；但 `inventory` 返回全库并标注属主 session，因为运行控制面板是全局的。

## 去哪深入（文件路由）

| 问题 | 去哪 |
|---|---|
| 组内包/ctx key 全貌 | `packages/extensions/README.md` |
| 5 个工具的确切 schema | `docs/tool-catalog.md#deepseek-aidsh-tool-cordis` |
| `ctx.dynamicCordisRunner` / `ctx.cordisInspect` 方法签名与事件 | `docs/subsystems/extensions.md#cordis-surface`（生成区） |
| 服务角色分类（core vs seam）、消费者关系 | `docs/capability-seams.md` |
| define/run/stop/invoke 的确切语义与拒绝码 | `packages/extensions/cordis-host-runner/README.md` |
| 页面侧求值、guard façade、`slots.onEntryError` | `packages/extensions/cordis-client-runner/README.md` |
| 面板为什么全局、卡片为什么只读 | `packages/extensions/ui-cordis/README.md` |
| 设计与信任立场的源头 | `.agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.md` |
