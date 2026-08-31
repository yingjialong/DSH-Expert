---
title: packages/client — Web GUI 浏览器半边
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - packages/client/README.md
  - packages/client/AGENTS.md
  - packages/client/web/README.md
  - packages/client/web/src/platform.ts
  - packages/client/runtime/README.md
  - packages/client/runtime/package.json
  - packages/client/runtime/src/client/index.ts
  - packages/client/runtime/src/client/contract/sessions.ts
  - packages/client/runtime/src/client/contract/sessions-port.ts
  - packages/client/runtime/src/client/sessions/service.ts
  - packages/client/runtime/src/client/sessions/manager.ts
  - packages/client/runtime/src/client/sessions/notifier.ts
  - packages/client/runtime/tests/sessions-service.client.spec.ts
  - packages/client/runtime/tests/manager.client.spec.ts
  - packages/client/ui-slots/README.md
  - packages/client/ui-renderer/README.md
  - packages/client/connection/README.md
  - packages/bundle/web-app/cordis.patch.yml
  - docs/subsystems/client-modules.md
  - packages/README.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-31
asked_by: self
---

## 一句话定位

dsh web GUI 的**浏览器一半**：shell boot、浏览器↔Host 通信、React-free 的对象层、slot 组合系统，以及一大票 `ui-*` 功能插件；Host 一半在 `packages/host/` [T1: packages/client/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。组 README 补充：**除 `test-runtime` 外全部是 product 包**，命名一律 `@deepseek-ai/dsh-client-<name>`。

## 包清单

`packages/client/` 下**实际有 40 个包**（已按 `package.json` 逐个核对），按角色分四层：

| 层 | 包 | 说明 |
|---|---|---|
| Boot | `web` | `@deepseek-ai/dsh-client-web`，两阶段 boot 内核；**它不是 Loader entry** |
| Boot | `modules` | 浏览器侧 client module 加载 |
| 对象层（React-free） | `runtime` | SlotRegistry / SessionRuntime / WorkspaceRuntime / 快照 store 引擎 |
| 对象层 | `connection` | 浏览器↔Host 的 RPC 与事件投递 |
| 组合内核 | `ui-slots` | **React-free 且 cordis-free** 的 slot 注册纯核心与四份 props 类型族 |
| 渲染机器 | `ui-renderer` | 唯一的 ctx↔React 集成点；boot 结束后 `ctx.uiRenderer.mount(el)` |
| 支撑（5） | `hmr`、`locale`、`ui-theme`、`ui-primitives`、`ui-layout` | 开发期重载、本地化、主题、共享控件、区域布局 |
| 功能插件（29） | `ui-agent-preset` `ui-attachment` `ui-brand-official` `ui-commands` `ui-conversation` `ui-deliverables` `ui-directory-picker-browse` `ui-directory-picker-native` `ui-goal` `ui-input-trigger` `ui-jobs` `ui-message-feedback` `ui-model-selection` `ui-permission-presets` `ui-plan` `ui-reference` `ui-settings` `ui-settings-general` `ui-settings-models` `ui-settings-plugin-inventory` `ui-settings-plugins` `ui-sidebar` `ui-skill` `ui-subagent` `ui-tool` `ui-trajectory` `ui-user-questions` `ui-workflow-run` `ui-workspace` | 一个 UI feature = 一个插件包 |

npm 名规则：目录名前缀化，`packages/client/ui-tool` → `@deepseek-ai/dsh-client-ui-tool`。

## 三件套结构

本组不是单一 capability seam，而是**三条一向知识链**（`packages/client/AGENTS.md#Layering red lines` 的"红线"）：

1. **数据对象层** `runtime`（零 React import，可 grep 断言）：`ConnectionController` → `SessionManager` → `Session` 拥有全部业务状态；zustand/immer 快照 store 引擎也在这里，产物是**裸 observable source，没有 hook 成员**。
2. **渲染机器** `ui-renderer`（动态插件）：全部 ctx→React 集成；每个 hook 都在绑定点由裸 source 合成。生产业务代码**不携带对 ui-renderer 的值依赖**。
3. **表现组件** 各插件包的 `src/client/`（纯 props）：预期被整体重写，业务逻辑不得渗入。

Slot 系统自身的三件套：**Definition = `ui-slots`**（`SlotCore` + `SlotMap` 声明合并 + `SlotRenderer`/`SlotRendererHost` 安装契约）→ **Provider = `runtime`**（`SlotRegistry` 包 `SlotCore` 并向 renderer 供数据源，`defineStore` 的值实现也在这里）→ **Consumer = 各 `ui-*` 插件**（`ctx.slots.register`）。

## 扩展点

- **加一个 UI feature** → 新建 `packages/client/<name>` 插件包，三处注册缺一不可（缺任何一处都在**不同的、更晚的**地方失败）：`tsconfig.client.json` 的 `references`、`packages/bundle/web-app/cordis.patch.yml` 的 `dsh.client` 行、`packages/bundle/web-app/package.json` 的 dependency。
- **唯一的组合 API** → `ctx.slots.register({ name, children?, store?, inject?, ...kind }, Component)`。没有单独的 slot 定义调用、没有白名单 face 对象、没有 face-minting helper；**只有 shell 渲染 `'root'`**。
- **往别人的 slot 里注册** → `ctx.slots.inject(name, () => ctx.slots.register(...))`：它等待**真实的声明**，声明坍塌时移除贡献，重新声明时重跑。裸 `slots.register` 进未声明 slot **一律 throw**。多个贡献要原子安装/回滚就返回 generator 逐个 yield。
- **组件 props** → 四份 share 的交集，全部**派生**：`PropsRuntime<K>` & `PropsRenderSlots<S>` & `PropsStore<H>` & inject face。禁止手写任何 share 已派生的成员。
- **共享状态** → 在 register 处声明 store（导出 `createXXXStore()` 工厂；**禁止模块级句柄**）；读 `props.useStore`，写 `props.actions.*`。
- **Chain-kind slot** → 反转 keyed 路由：每个注册带纯函数 `ChainSelect` 选择器（可选升序 `priority`，同分按注册序），第一个非 null 当选并成为组件的 `matched` prop；全 null 落到 owner 的 `renderSlotChain` fallback。
- **加一种会话内容节点** → 注册一个 `ConversationNodeDefinition` + keyed 的 `conversation.chat.node` renderer，走 `docs/cookbook/adding-a-conversation-node.md`。
- **共享一个非基线模块** → `dsh.client.external` 声明**确切的 import specifier**（只有末尾 `/client` 会 alias 到包行）。

## Known Limitations

各包 `## Known Limitations and Deferred Work` 里最有辨识度的几条：

- `runtime`：**`loader.unload` 是 stub，直接 throw not-implemented**——client 没有"从 fiber 释放一路到注册与样式移除"的卸载链。
- `runtime`：**Scope teardown 是 stage 驱动、当前单占位**；被移除但仍在台上的 session，其 scope 会冻结存活到台子挪走为止，而非真实观察者归零。
- `runtime`：**从插件 bundle 里 value-import 本包必须用 `/client` 子路径**——裸包名不在 loader externals 表里，会内联出第二个模块实例，其私有 scope-tag Symbol 永不匹配。
- `web`：**应用等待完整名册**，一个 entry 失败就一直停在无框架的 boot 页并给逐 entry 报告；不支持部分 UI 可用。
- `ui-slots`：`isLive` 线性扫描全部记录（UI 插件量级下没问题）；`__renders` phantom anchor 会出现在 `PropsRenderSlots` 上，是刻意接受的类型噪音。

## 陷阱

1. **`dsh.client.inject` 是"informational only"**——只用于 preflight 展示与 HMR diff，**不排序 entry 激活、不排序 apply 顺序**。真正的激活顺序只由 Cordis fiber 对 **service** 的等待决定 [T1: packages/client/AGENTS.md#New plugin package checklist]。
2. **模块图 ≠ Cordis DI**，两者不可互换：Cordis `inject` 未满足 → 永远 PENDING、**无超时**；模块图 `external` 未满足 → **当场 throw**。前者可被任意 provider 满足、允许环；后者是唯一模块身份、不可替换、**禁止环**。
3. **基线 external 是隐式的**：React、Cordis、`runtime`、`ui-primitives`、`ui-slots` **不要**再写进 manifest。基线的唯一真相源是 `packages/client/web/src/platform.ts` 的 `PLATFORM_MODULES` + `PRELOADED_CLIENT_EXTERNALS`。
4. **动态包永远不能把 `@deepseek-ai/dsh-*` 放进 `dependencies`**；动态内部依赖是 peer + dev，静态 client 输入是 dev-only。
5. **探活 `dsh web` 前必须先 `pnpm --filter <pkg> bundle`**——registry 服务的是 `lib/client.js`，不是源码。
6. **组件永远看不到 `ctx`**，也不读 React context（`BindingContext` 及其同类是 renderer 内部的）。业务代码**不得自造 hook**；框架给的席位只有 `useSession` / `useSessions` / `useWorkspaces` / `useStore` / `renderSlot`，加上 renderer 从 provide 贡献和 inject `hooks` 隔间绑定出来的 `use<Name>`。
7. **`children` 既是声明也是授权**：渲染未声明的 slot、或重复声明别人已声明的 slot，**在 load 期失败**——文档明说"不要绕过，冲突就是设计在说话"。
8. **组 README 的包表已经漂了**：`ui-deliverables`、`ui-directory-picker-browse`、`ui-directory-picker-native`、`ui-message-feedback` 四个真实存在的包**没有出现在表里**；表里的 `test-runtime` 行指向 `../test-support/client-runtime`，并不在 `packages/client/` 下；`ui-permission/` 这一行的标签与真实目录 `ui-permission-presets/` 不一致。以目录为准。
9. **通知发布纪律**：`notifyNow` 只用于用户手势的直接回显；结构性更新用 microtask 批处理的 `markDirty`，可见的流式 chunk 用累积的 `markFrameDirty`。
10. **Web 层是纯表现**："怎么画"（tool 卡片视图、队列状态）**不进 session log**；但新增 *model-visible* 输入仍然必须有 session event（仓库级铁律）。
11. **样式**：只用 `--dsw-*` token + CSS Modules + `clsx`，禁字面色值、禁组件库、禁 Tailwind；**产品文案是中文，代码注释是英文**。
12. **client 源码包在逐文件 100% 覆盖率闸门内**（`pnpm run test:coverage`）；jsdom 靠 spec 首行的 `// @vitest-environment jsdom` pragma，共享配置保持 node env。

## 2026-08-31 · Host exact-preset create与client-runtime收敛边界

- Host`SessionsApi.create`接受`agentPreset?`，但正式`@deepseek-ai/dsh-client-runtime/client`导出的concrete`SessionRuntime.create`只接受`workspaceId?/cwd?/sessionId?`；内部`SessionManager.create`同样不透传preset。跨domain`SessionsPort.create`更窄，只收`workspaceId`。上游没有说明省略preset的产品理由。
- `SessionManager`虽在源码文件export class，却未被`/client`index或package exports导出，正式tarball也不含src，不能作为out-of-tree seam。没有`adoptCreateResult/noteCreated`。正常`SessionRuntime.create`会把Host echo立即merge并同步`projectList()`，所以promise返回时list与`binding(id)`可用；这一保证不能外推给direct`IApiClient.sessions.create`。
- direct create的官方增量是`host/session-added{sessionId,blank,cwd?,parentSessionId?,origin?,agentPreset?}`；标准Connection pump交给Manager后以microtask投影进`SessionRuntime.list`。`session/subscribed`走mux，workspace attach另发`host/workspace-changed`；idle create不保证`host/session-status`。
- concrete`SessionRuntime.refresh()`是正式`.d.ts`方法但不在`ISessions` outward face，契约注释把它与frame handlers归runtime-internal。它single-flight重拉`session.list`，却在promise完成后才由Notifier microtask把Manager投影到Runtime；一方test helper显式再`await Promise.resolve()`。因此`await refresh(); binding(id)`无同步保证，需等待公开`list.subscribe/getSnapshot`确认id后再取binding，并由caller自设timeout；refresh把错误fold在private Manager snapshot，outward list不提供成功receipt。
- outward`SessionListState.phase`只在首次成功pull从pending变ready，之后sticky；后续refresh失败仍ready。Manager私有snapshot才有state/error。因此“refresh Promise resolve + phase ready”不能证明本轮Host list成功；即使direct确认本轮list成功，exact id缺席也只证明该baseline的current absence，不是create occurrence从未commit。
- `noteAgentPreset`不是private：`ISessions`与`SessionRuntime`都正式公开，ui-agent-preset在成功blank switch后调用。它只承诺更新已有Session的host-confirmed composition label；实现对unknown id可upsert不是external create adoption合同，也不携带cwd/workspace等完整birth facts。
- client summary对`agentPreset`采用newest-wins，create echo、host/session-added、session.list与noteAgentPreset都可更新。`binding()`只是listed/addressed id的纯懒解析，不主动拉Host。
- **认知状态**：verified_inference（固定rc.2正式runtime/connection client exports、concrete/outward contracts、Manager/Notifier控制流与sessions-service/manager一方tests；未跑真实Connection end-to-end）。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 包名册与各包一句话职责 | `packages/client/README.md`（注意上面第 8 条漂移） |
| slot / props / store / export / ctx 纪律，分层红线，新包与新组件清单 | `packages/client/AGENTS.md` |
| slot 类型链、四份 share、chain slot、store 类型族 | `packages/client/ui-slots/README.md` |
| `ctx.slots.inject`、Session/Workspace 列表、会话装配、标题/重试/fork/模型选择投影 | `packages/client/runtime/README.md` |
| 两阶段 boot、模块系统注入、boot 页 | `packages/client/web/README.md` |
| 共享模块基线的唯一真相源 | `packages/client/web/src/platform.ts` |
| 渲染机器与 mount 契约 | `packages/client/ui-renderer/README.md` |
| RPC / 事件投递 / 重连 | `packages/client/connection/README.md` |
| 模块系统子系统主篇 | `docs/subsystems/client-modules.md` |
| 浏览器插件名册的实际装配位置 | `packages/bundle/web-app/cordis.patch.yml` |
| 加一个会话节点 | `docs/cookbook/adding-a-conversation-node.md` |
| 样式权威 | `docs/web-styling.md` |
| 校验 client 特有规则（可 `--fix`） | `scripts/verify-client-packages.ts` |
