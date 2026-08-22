---
title: Cordis 内核入门（DSH 的插件/服务/事件模型）
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - docs/cordis-primer.md
  - docs/cordis-api/context.md
  - docs/cordis-api/events.md
  - docs/cordis-api/fiber.md
  - docs/cordis-api/registry.md
  - docs/cordis-api/service.md
  - docs/cordis-api/inherited.md
  - docs/cordis-tutorial/index.md
  - docs/cordis-tutorial/07-into-the-harness.md
  - docs/event-producer-consumer.md
  - docs/postmortem/0001-acp-default-export-drops-inject.md
  - docs/user/develop/basic/index.md
  - docs/user/develop/framework/service.md
  - vendor/README.md
  - vendor/cordis/src/reflect.ts
  - vendor/cordis/src/fiber.ts
  - vendor/cordis/src/registry.ts
  - vendor/cordis/src/service.ts
  - vendor/cordis/src/events.ts
  - vendor/loader/src/index.ts
  - packages/AGENTS.md
  - cordis:packages/core/src/context.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## Cordis 是什么、与 DSH 的关系

Cordis 自述为 "A Meta-Framework of Spatiotemporal Composability"，上游是 `cordiverse/cordis`，README 里明写 **API 尚未稳定、可能无预告变更** [T1: cordis:README.md]。DSH 不通过 npm 依赖它，而是**源码 vendor** 进 `vendor/`，并整体 rescope 到 `@deepseek-ai/*`（`cordis` → `@deepseek-ai/cordis`，`@cordisjs/plugin-x` → `@deepseek-ai/cordis-plugin-x`）；`@deepseek-ai/cordis` 是每个 harness 包的 peerDependency [T1: vendor/README.md]。

**回答 DSH 问题时，权威源是 `vendor/cordis/src/`，不是本地 `upstream/cordis` 镜像**：vendored 版本 pin 在 cordis 4.0.0-rc.7（commit `56b3d4f`），而 `upstream/cordis` HEAD 已是 4.0.0-rc.8；`vendor/README.md` 还列了 18 条 local modifications，其中 #6（fiber 生命周期加固）、#8（Loader/Include 事务化重载）、#15（`internal/config` 惰性配置解析）是**行为性**改动，不是注释。名字映射见 `docs/rescope.md`。

## 核心概念（保留英文原名）

| 概念 | 一句话 | 锚点 |
|---|---|---|
| **Context** | 一个 Proxy 对象；普通属性读经过 service resolver，`extend()` / `isolate()` / `intercept()` 派生子 context 而不修改父 | `vendor/cordis/src/context.ts#Context` |
| **Service** | 基类，构造函数里 `super(ctx, name)` 即把实例注册为 `ctx.<name>`；注册本身是 effect | `vendor/cordis/src/service.ts#Service` |
| **Plugin** | 三种形态：function、`{ apply }` object、`Service` 子类。registry 的身份键是**被解析出的 callback 函数本身**，所以同一函数 mount 两次 = 一个 `Plugin.Runtime` + 两个 Fiber | `vendor/cordis/src/registry.ts#RegistryService.plugin` |
| **Fiber** | 一次 plugin 装载实例（状态 / 校验后 config / 已注册 effects）；`ctx.fiber` 是当前 fiber，`ctx.plugin()` 返回 `Fiber & PromiseLike<Fiber>` | `docs/cordis-api/fiber.md` |
| **Registry** | `ctx.plugin` / `ctx.inject`；`ctx.registry.values()` 遍历所有 Runtime，可用来枚举 fiber 状态做诊断 | `docs/cordis-api/registry.md` |
| **Effect** | `ctx.effect(execute, label?)`，execute 立即执行、返回 disposer；`fiber.getEffects()` 输出带 label 的树 | `vendor/cordis/src/fiber.ts#Fiber.effect` |
| **Inherited** | 指 harness 层**之外**每个插件都能看到的 cordis core + loader/hmr/timer 的 ctx 成员与事件清单 | `docs/cordis-api/inherited.md` |

跨文档才能拼出的事实：`ctx.on` / `ctx.plugin` / `ctx.effect` 这些不是 Context 自己的方法，而是 `ReflectService` 构造时 mixin 上去的——`reflect`(get/set/provide/accessor/mixin)、`fiber`(runtime/effect)、`registry`(inject/plugin)、`events`(on/once/parallel/emit/serial/bail/waterfall) [T1: vendor/cordis/src/reflect.ts#ReflectService]。因此 `ctx.runtime` 也真实存在，但 `inherited.md` 没列。

## 插件的生命周期

状态迁移见 `docs/cordis-tutorial/02-lifecycle-and-effects.md`：`PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED`，旁路 `FAILED`。注意**枚举声明顺序不等于迁移顺序**：源码里是 `PENDING, LOADING, ACTIVE, FAILED, DISPOSED, UNLOADING` [T1: vendor/cordis/src/fiber.ts#FiberState]，所以不要按数值大小推断先后。

- **PENDING 是合法的静默态**（等 `inject` 的服务），不报错、也不吊住 Node 事件循环——组合里没别的东西时会静默 exit 0。
- **FAILED** = `apply` 抛错或 config 校验失败（`ValidationError`）。
- `fiber.await()` 等待稳定并重抛启动错误；`fiber.restart()` 用当前 config 重装；`fiber.update(config, noSave?)` 先跑 `internal/update` waterfall（HMR / 更新钩子可否决或替换重启）再重启。
- **disposer 顺序有两个层级**：同一个 `ctx.effect()` 内收集到的 disposer 逆序且遇到 promise 就串行 await [T1: `vendor/cordis/src/fiber.ts` 的 `disposables.splice(0).reverse()` 链式 `.then`]；而 fiber 卸载时多个 effect 的 disposer 走 `Promise.all` **并发**。要保证顺序就把相关清理塞进同一个 effect。
- fiber 处于 `UNLOADING` 时再建 effect 会抛 `CordisError('INACTIVE_EFFECT')`（`PENDING` / `LOADING` 期间合法）。

## 服务的注册与消费（ctx-key 机制）

服务存储**不按名字索引，而按 isolate label symbol 索引**：`ctx.root[symbols.isolate][name] ??= Symbol(name)`，实现存进 `reflect.store[key]` [T1: vendor/cordis/src/reflect.ts#ReflectService.provide]。这正是 `ctx.isolate(name, label?)` 能让两个 group 各看到一份不同 `shell` provider 的机制；传同一个 label 则两个 isolate 作用域合并。

消费有两条路，**语义不同、不能混用**：

1. 在 `inject` 里声明过的服务 → 直接读 `ctx.<name>`。属性代理走 **仅沿祖先** 的 fiber 走查，且受 traceable shadow 影响。
2. 没声明的可选服务 → 必须用 `ctx.get(name)`，它直接查全局 isolate-keyed store，与 fiber 拓扑无关；`strict` 默认 `true`，只返回 provider fiber 处于 ACTIVE 的实现。

这是 `packages/AGENTS.md` 的硬规则，代价来自 `docs/postmortem/0001-acp-default-export-drops-inject.md`（Bug #2：`AgentLoop.resume` 读 sibling 分支上的 `sessionPersistence`，祖先走查到 root 抛 `cannot get property "sessionPersistence" without inject`）。

其它要点：`inject` 不是一次性启动检查——provider 中途消失，所有依赖者一起卸载，provider 回来再装，这正是"换 provider 只改 cordis.yml"能成立的原因（`docs/user/develop/framework/service.md`）。`ctx.intercept(name, config)` 的合并规则在 `vendor/cordis/src/service.ts` 的 `[symbols.resolveConfig]`：靠近 root 的先生效，`base` 前置、`head` 后置。除 `export const inject` 外还有 `@Inject()` 装饰器（类 / 类方法两种用法）[T1: vendor/cordis/src/registry.ts#Inject]。

## 事件的 producer / consumer 模型

事件名与签名靠 TypeScript declaration merging（`declare module '@deepseek-ai/cordis' { interface Events { ... } }'`）声明，**dispatch mode 是事件公共契约的一部分**，DSH 要求用 `@mode` JSDoc 标注，由生成器校验声明与派发点是否一致（`docs/cordis-primer.md`，仓库 AGENTS.md 约定）。

五种模式：`emit`（同步广播，不收返回值）、`bail`（同步版 serial，遇第一个非 null/false/undefined 返回即停）、`serial`（有序 await 版 bail）、`parallel`（并发 await 全部）、`waterfall`（around-middleware，最后一个参数是 `next`）[T1: vendor/cordis/src/events.ts#DispatchMode]。

**要查"某个事件谁发、谁听"，直接去 `docs/event-producer-consumer.md`**——生成的全仓矩阵，列出每个 harness 事件的 mode、声明位置、dispatcher 包、listener 包，比逐个 subsystem 页翻快得多。框架自身的 `internal/*` 事件在同页末尾单列；`internal/dispatch` 只对非 `internal/` 事件触发 [T1: vendor/cordis/src/events.ts]。

waterfall 的纪律（仓库级硬规则）：**只观察或标注的 listener 必须调用 `next()`**，不调 `next()` 就是有意短路（veto）；忘了写会静默吞掉下游所有默认行为。合作型 listener 通常改共享的 request/decision 对象后再委派；`prepend: true` 只在必须先于普通注册运行时用。

## 与 DSH 的接缝：`docs/cordis-tutorial/07-into-the-harness.md`

这一章是"纯 Cordis 概念"到"真实 harness 服务"的唯一桥梁，全程 keyless、不调模型。它做三件事：

1. `inject: ['tools']` 的插件用 `ctx.tools.register(defineTool({ name, description, parameters, output: { schema, render }, execute }))` 注册一个模型可调用的工具。`register` 返回 disposer 且内部走 effect（label `tools.register()`），卸载即注销 [T1: `packages/core/tools/src/index.ts#ToolRuntime.register`]。
2. 用 `ctx.tools.execute({ callId: CallId(...), name, arguments, signal })` 冒充模型驱动一次真实执行管线。
3. 另一个互不相识的插件用 `ctx.on('tools/result', ...)` 观察结果——两个插件通过 registry 服务 + 事件连接。

三个容易被问到的接缝事实：`@deepseek-ai/dsh-tools` 自己 inject `systemPrompt`（工具 schema 要进系统提示），所以组合里必须同时列 `@deepseek-ai/dsh-system-prompt`，否则 tools 永远 PENDING；`tools/result` 在结果物化阶段 emit，**早于** `execute` 的 promise 兑现给调用方；`import type {} from '@deepseek-ai/dsh-tools'` 这行只为拉进 declaration merge，运行时什么都不导入。终点路由是 `examples/headless-agent/cordis.yml`（一个可以逐行读懂的完整 agent 组合）。

## 写第一个插件的最小路径

两条路，**别走错**：

- **学 Cordis 本身**（在仓库 `tmp/cordis-tutorial` scratch 目录里跑，启动器 `node --import tsx ../../vendor/cordis/bin.js`）→ `docs/cordis-tutorial/index.md`，8 篇顺读，第 7 篇接 harness。
- **写真正的 harness 插件**（由 `cordis.yml` 加载、Web UI 驱动）→ `docs/user/develop/basic/index.md`；能力三角色（Service Definition / Service Provider / Consumer）拆分见 `docs/user/develop/practice/index.md`。
- 概念速查 → `docs/cordis-primer.md`；API 参考 → `docs/cordis-api/*.md`（由 `scripts/gen-cordis-catalog.ts` 生成，**不要手改**，`pnpm run verify-cordis-catalog` 会校验新鲜度）。

## 陷阱

1. **namespace plugin 与 `export default` 互斥**。Loader 的 `unwrapExports` 是 `exports.default ?? exports`，一旦有默认导出就拿到裸函数，`name` / `inject` / `Config` 这些兄弟命名导出全部丢失，插件在无注入的 fiber 里运行 [T1: vendor/loader/src/index.ts#unwrapExports]。`ctx.plugin()` 不走 `unwrapExports`，所以手搭的单测**永远测不出这个 bug**。
2. **可选服务用 `ctx.get(name)`，绝不用 `ctx.<name>`**（见上节）。同理，顶层测试代码里 `ctx.fiber.runtime` 为 `null` 会走 `ctx.reflect.get(prop, false)` 直查全局 store 的旁路，掩盖真实插件拓扑下的失败。
3. **`cordis.yml` 条目不写 `id` 就等于每次读配置都换新身份**，任何一次配置文件编辑都会让该行 remount，哪怕它自己一个字没改。
4. **`!!js` 只在 plugin `config` 和条目 `disabled` 两处生效**，其它元数据（`name` / `id` / `inject`）保持字面量；且必须写 `!!js`，`!js` 无效（仓库 AGENTS.md 明确禁止）。
5. **`Config` 必须是 Standard Schema 校验器**（DSH 用 Schemastery），导出一个普通对象不会生效。
6. **模块解析失败不 crash**：路径或包名拼错只经 Cordis logger 上报，boot 阶段还没有 console exporter 时这条报告会丢。新加的条目"毫无动静"先查拼写。
7. **条目并发启动**，YAML 里的先后顺序不构成加载顺序保证；顺序只由服务依赖（`inject`）决定。
8. **服务名在每个 application 内是一个扁平命名空间**，harness 已占用 `tools` / `llm` / `sessions` / `agents` 等平实名字，自定义服务要加前缀。
9. 同一 fiber 上不同 effect 的 async disposer **并发**执行；要顺序请合进一个 effect。
