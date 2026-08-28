---
title: 插件开发全路径（dsh-plugin 从零到发布）
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - docs/user/develop/basic/index.md
  - docs/user/develop/basic/tool.md
  - docs/user/develop/basic/config.md
  - docs/user/develop/basic/publish.md
  - docs/user/develop/framework/index.md
  - docs/user/develop/framework/service.md
  - docs/user/develop/framework/events.md
  - docs/user/develop/practice/index.md
  - docs/user/develop/practice/llm-adapter.md
  - docs/cookbook/adding-a-tool.md
  - docs/cookbook/adding-a-package.md
  - docs/cookbook/adding-an-llm-adapter.md
  - docs/cookbook/adding-a-vendored-package.md
  - docs/cookbook/adding-a-settings-card.md
  - docs/cookbook/adding-a-conversation-node.md
  - docs/cookbook/extension-cookbook.md
  - packages/AGENTS.md
  - packages/core/tools/README.md
  - packages/client/AGENTS.md
  - packages/skill/skill/src/index.ts
  - packages/skill/tool-skill/README.md
  - packages/skill/skill-filesystem/README.md
  - packages/extensions/README.md
  - docs/postmortem/0001-acp-default-export-drops-inject.md
  - docs/postmortem/0002-js-expression-disabled-filesystem-tools.md
  - scripts/check-workspace-constraints.ts
  - CONTRIBUTING.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 从零写一个 dsh-plugin 的完整路径（步骤路由）

两条互不相同的路线，先分清你在哪条：

| 你要做的 | 路线 | 权威文档 |
|---|---|---|
| 自己的第三方插件包（不进上游仓库） | 教程链 → bundle/profile 安装 | `docs/user/develop/basic/index.md` → `tool.md` → `config.md` → `publish.md` |
| 往 deepseek-harness 仓库里加 workspace 包 | 文件级 checklist + 仓库门禁 | `docs/cookbook/adding-a-package.md` |

- **第 0 步（最小插件形状）**：一个导出 `apply(ctx)` 的 TS/JS 模块就是插件；`name` / `inject` / `Config` 可选。三种形态（function / object / class）见 `docs/user/develop/basic/index.md`。
- **第 1 步（挂载）**：`cordis.yml` 或 `--patch` overlay 里加一行 `- insert: [{ id, name }]`；本地源码路径**必须是绝对路径**（patch 文件不改变 loader 的模块解析目录）。
- **第 2 步（注册能力）**：`ctx.tools.register` / `ctx.llm.registerAdapter` / `ctx.on` / `ctx.effect`，全部是 effect，卸载自动回收。
- **第 3 步（配置）**：`docs/user/develop/basic/config.md`。
- **第 4 步（打包安装）**：`docs/user/develop/basic/publish.md`（bundle vs profile）。
- **仓库内追加**：还要过 `pnpm run constraints / typecheck / lint / build / hygiene / doc-sync`，以及 `packages/AGENTS.md` 的包级硬规则（invariant companion、README Model Experience、Known Limitations）。
- **要 vendor 一个上游 Cordis 包**（如 `@cordisjs/plugin-http`）：走 `docs/cookbook/adding-a-vendored-package.md`，不是 npm 依赖，且有 `scripts/check-vendor-manifest.sh` 的 pre-commit 门禁（改 `vendor/*/src` 必须同时 stage `vendor/README.md`）。

## 四种典型形态的骨架

1. **新工具（Consumer）**：`inject = ['tools']` + `ctx.tools.register(defineTool({...}))`。`defineTool` 在 `packages/core/tools/src/schema.ts#defineTool`。骨架与合同：`docs/cookbook/adding-a-tool.md`；入门：`docs/user/develop/basic/tool.md`。
   - 必须声明 `output.schema`（canonical JSON）+ `output.render`；`execute` 只返回 canonical value，不返回 content blocks。
   - UI 卡片是**独立关注点**：`presentCall` / `presentResult` 返回 `card`-tagged render intent（`generic` / `terminal` / `diff` / `search` / `web`），且必须是 `args`（+result）的**纯函数**，因为它们在 session-log replay 时也会跑。
2. **新 LLM adapter（Provider）**：`class X extends LlmAdapter` 实现 `stream()`，`inject = ['llm']`，`ctx.llm.registerAdapter(['route'], adapter)`。合同：`docs/cookbook/adding-an-llm-adapter.md`；带代码的版本：`docs/user/develop/practice/llm-adapter.md`；`StreamChunk` 协议约定的源头是 `packages/llm/llm/src/types.ts`。
3. **新 service provider**：`class X extends Service { constructor(ctx){ super(ctx,'key') } }` + `declare module '@deepseek-ai/cordis' { interface Context { key: X } }`。三角色拆分见 `docs/user/develop/practice/index.md`。
4. **UI 扩展**：三种不同的东西，别混：
   - 纯外部 UI / 协议驱动：监听 `session/event`，回灌 `agent.followup()` / `steer()`（`docs/cookbook/extension-cookbook.md`）。
   - Web Client 设置卡片：`docs/cookbook/adding-a-settings-card.md`（namespace 是 host 半与 browser 半的 join key）。
   - Web Client 会话行：`ConversationNodeDefinition` + keyed Chat renderer（`docs/cookbook/adding-a-conversation-node.md`）。
   - 三者都要 `dsh.client` manifest + `./client` export + lazy-CJS bundle；client 侧硬规则在 `packages/client/AGENTS.md`。

## 依赖纪律：为什么必须依赖 Service Definition 而不是具体 Provider

跨文档综合（`docs/user/develop/practice/index.md` + `packages/AGENTS.md` + `docs/cookbook/adding-a-package.md`）：

- 一个 capability seam = Service Definition + Service Provider + Consumer **三者合起来**，单个角色不是 seam；Provider 与 Consumer **互不依赖**，都只依赖 Definition。
- `inject: ['shell']` 让 Provider 可以被 `cordis.yml` 换掉而 Consumer 不动；直接依赖 `dsh-bash-local` 会把这个可替换性焊死。
- 反向规则同样是硬规则：**Service Definition 要为当前所有 Consumer 设计**，不能让某一个 Consumer（尤其是 tool schema / Loader / UI / transport 需求）决定服务合同。
- 不要预先拆包：角色不需要独立演进时，一个包可以同时承担多个角色。
- 依赖出现/消失是动态的：required service 消失 → 依赖插件自动 dispose，服务回来再自动加载（`docs/user/develop/framework/index.md`）。
- 可选依赖用 `ctx.get('name')`，**不要**用 `ctx.<name>`：属性代理是拓扑敏感的，`ctx.get` 读全局 service store（`packages/AGENTS.md`，postmortem 0001）。

## 配置与 schema 定义

- `Config` 必须同时是**类型**和**同名 Schemastery schema**；导出普通对象会失败——Cordis 要的是 Standard Schema 接口。
- 默认值写在 schema 字段上；加载期校验，非法配置**加载即失败**。
- 硬规则：**插件里不许硬编码 tunable**。判据是"`cordis.yml` 能不能不改代码就改这个值"；`DEFAULT_*` 常量或 test hook 不算可配置。协议常量、外部规范、安全不变量则固定不动。
- `!!js` 表达式只在 Loader entry 的 `config` 与 `disabled` 两个字段求值（`config` 在声明注入激活后对该插件 ctx 求值；`disabled` 在每次挂载决策时对 loader ctx 求值），其余 entry 元数据保持字面量。`!js`（单感叹号）是错的。
- 设置卡片场景用 `installSettingsSection(ctx, ns, Config, config, {...})`，`role('secret')` 字段不回传值，`applies: 'restart'` 表示重启才生效。

## 事件与服务的使用方式

- 四种 dispatch mode：`emit`（广播、忽略返回）/ `bail`（首个非 null|false|undefined 短路）/ `serial`（顺序 + await）/ `waterfall`（管道）。
- **waterfall 监听器必须调用 `next()`**；不调用就是故意短路（这正是 gateway/拦截的实现方式）。
- 事件命名 `namespace/action`；类型通过 `declare module '@deepseek-ai/cordis' { interface Events {...} }` 声明合并。
- **Cordis 事件 ≠ session event**：`turn/*`、`step/*`、`tool/call`、`tool/result`、`compaction/*` 是持久化 session 事件类型，要监听 `session/event` 再看 `event.type`。特别注意 `tools/result`（live Cordis 事件）与 `tool/result`（durable session 事件）只差一个 s。
- 工具执行的四个扩展点选择规则（`packages/core/tools/README.md#extension-points`）：`tools/pre-execute` 可重排的 allow/deny/ask → `ctx.tools.guard()` 单调最终拒绝 → `tools/execute` 包裹 dispatch（只能替换 `exec.signal`）→ `tools/post-execute` 改结果/attach context → `tools/result` 只读观测。
- 工具可见性用 `ctx.tools.restrict()`（agent 作用域），它是**可见性组合，不是权限边界**。
- 完整的"产品特性 → 插件机制"映射表在 `docs/cookbook/extension-cookbook.md`（末尾表格），是判断"我该挂哪个点"的最快路由。

## packages/extensions：运行时自我修改意味着什么

细节见 `wiki/packages/extensions.md`。对插件开发者而言的三条结论：

- 模型可以在**当前进程内**定义/运行/停止自己写的 Cordis package（host 半跑在 `node:vm`），这意味着"插件"不一定来自磁盘或 npm。
- 这些 dynamic package **只活在进程内存**：不写盘、不改 `cordis.yml`、重启即消失、session-scoped，且**不在任何 shipped tree 里**（默认不挂载，属于刻意 opt-in，信任等级等同 bash 权限）。
- 它验证了微内核主张：因为每个注册都是 `ctx.effect`，动态挂载/卸载能干净回收——这也是普通插件必须坚持 effect 化注册的现实理由。

## packages/skill：provider registry 与 catalog/loader

四包分工（`packages/skill/README.md`）：`skill`（`ctx.skills` 注册表）/ `skill-filesystem`（本地发现）/ `skill-badge`（内置 badge skill）/ `tool-skill`（catalog + `skill` 工具）。

- 注册表是 **host + per-scope 分层**（复用 tools registry 的形状）：全局层 vs agent preset 层，读取时就近层胜出，rank 只在同层内决胜。
- Provider 合同：`ctx.skills.registerProvider(create)`，工厂**同步**执行并拿到 `{ signal, invalidate }`；真正的远程发现放在 `await list(options)` 里。返回数组 = 完整发现；`{ candidates, complete: false }` = 不完整（不缓存、不影响 candidates 可用）。
- rank 数字**越小越优先**：`project-dsh` 100 / `project-agents` 200 / runtime `ctx.skills.register()` 250 / `custom` 300 / `user-dsh` 400 / `user-agents` 500 / bundled 600（`BUNDLED_SKILL_RANK`）。
- `ctx.skills.get()` 是 **policy-neutral** 的加载原语——消费者必须自己用 `isModelInvocable` / `isUserInvocable` 卡自己的面。
- 失效是 provider 驱动的：注册表没有 TTL，可变 provider 必须自己持有并调用注册期的 `invalidate()`；`skills/change` 只是无差别失效通知，不带 diff。
- catalog 由 `tool-skill` 在每次合格的 `agent/pre-step` 重算，digest 只覆盖 `name`+`description`（不覆盖渲染文案）；替换是**整表替换**。

## 发布（dsh-plugin topic、npm 命名约定）

- 上游**目前不接受外部 PR**（`CONTRIBUTING.md` 原文），推荐路径就是自己发插件包，并给 GitHub 仓库打 **`dsh-plugin` topic** 做发现（`README.md` 与 `CONTRIBUTING.md` 都只提 GitHub topic，没有 npm keyword 约定）。
- 包名：`@deepseek-ai/dsh-<name>` 是**上游 workspace 包**的命名，第三方包不受此约束（教程里的第三方示例直接叫 `dsh-hello-plugin`）。
- 让包成为 bundle 的唯一条件是 manifest 里有 `dsh.bundle.patch`；没有它也能装，但只是普通依赖 + 一次告警，不激活任何层。
- 层序（`dsh --profile` 组合）：bundles 按 `dsh.profile.bundles` 顺序（`@deepseek-ai/dsh-base` 第一）→ profile 自己的 `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml` → 每个 `--patch`。后层按 row **整体替换** `config`，不做深合并。
- `dsh plugin --profile <name> <args...>` 就是在 profile 目录里转发 pnpm；`--dump-config` 可以不启动就验证层。

## 陷阱

1. **`export default apply` 会吃掉 `inject`**：Loader 的 `unwrapExports` 优先取 `.default`，namespace plugin 的 `name`/`inject`/`Config` 全部丢失，症状是 `cannot get property "x" without inject`。规则：function plugin 只用具名导出且**不要**有 default export；service 包才 default-export service class（postmortem 0001 / `packages/AGENTS.md`）。
2. **可选服务别用 `ctx.<name>`**：属性代理跨 shadow 边界会失败，用 `ctx.get(name)`。
3. **`!!js` 不是万能的**：曾经只有 `config` 被插值，`disabled` 拿到的是表达式对象（永远 truthy → 插件永久禁用），见 postmortem 0002；现在 `disabled` 已支持插值，但其余 entry 元数据仍是字面量，`verify-cordis-config` 会拒。条件组合优先用 overlay。
4. **presentCall/presentResult 必须纯**：replay 时也会执行，任何 I/O、读 session、时钟、随机都会炸；想要"文件旧内容"就说明该走 `output.presentationMeta`。
5. **git 安装的插件不会自动 build**：git 装的是源码不是产物，作者要写自包含 `prepare`，用户要在 profile 的 `pnpm-workspace.yaml` 里 `allowBuilds`——那等于授权在安装时于沙箱外执行该包代码。想省事就发 npm 或 `pnpm pack` 发 tarball。
6. **UI-only 的格式不许污染模型结果**：fenced console 块、diff、相对化路径都不能进 canonical value 或 Native content。
7. **`run_in_background` 之后不要再用 `exec.signal`**：`ctx.jobs.start()` 发布 id 之后要改用 task-owned 取消信号；外层取消只停止等待，不杀已发布的后台工作。
8. **仓库内加包时 `private: true` 的说法已过时**：`docs/cookbook/adding-a-package.md` 仍写 `private: true` 是 package.json 不变量，但 `scripts/check-workspace-constraints.ts` 现在要求 release member **不得** `private: true` 且必须 `publishConfig.access: "public"`；只有 experimental 包（和少数例外）保留 `private: true`。
