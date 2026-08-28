---
title: packages/typert — 类型图生成、加载与运行时注册表
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/typert/README.md
  - packages/typert/registry/README.md
  - packages/typert/registry/src/index.ts
  - packages/typert/loader/README.md
  - packages/typert/protocol/README.md
  - packages/typert/generator/README.md
  - docs/subsystems/typert.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

把开发者手写的源码类型树变成编译器无关的运行时反射与 Zod schema，并让 Loader 组合自动发现装载——Typert 把**源码分析 / 运行时存储 / Loader 发现**三件事彻底分开。它是 `api/` Remote RPC 网关和 Client API 的类型底座。

## 稳定性

`Product — stable API`（`packages/README.md` 表格中 `typert/` 行原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `typert/registry/` | `@deepseek-ai/dsh-typert-registry` | 运行时注册表 `ctx.typert`：存包反射与 live Zod schema，原子注册、随 fiber 撤销 |
| `typert/loader/` | `@deepseek-ai/dsh-typert-loader` | Node-only：扫描 Loader entry，import 各包 `./typert` 出口并注册（**不提供** registry 本身） |
| `typert/generator/` | `@deepseek-ai/dsh-typert-generator` | 构建期库：TypeScript 工程分析 → `FaceModel` / `TypeGraph` → 产出 artifact |
| `typert/protocol/` | `@deepseek-ai/dsh-typert-protocol` | 编译器无关的共享声明：Remote 基类、装饰器、协议 map、`InvocationDescriptor`、codec、provider 契约 |

## 三件套结构

这一组**不是** capability seam 的 Definition/Provider/Consumer 形状，而是一条**流水线 + 一个注册表**：

- **协议底座**：`dsh-typert-protocol` —— 只声明，不注册任何具体 Cordis service。`@Remote`、`@RemoteScope(key)`、`TypertRemoteService`、`bindTypertRemote(this, serviceKey, options?)`、`remoteMethods(service)`。
- **构建期生产者**：`dsh-typert-generator`（`WorkspaceAnalyzer` / `FaceModelEmitter` / `WorkspaceTypertGenerator`），产出 `lib/typert.host.{js,d.ts}`（暴露为 `package/typert`）和 `lib/typert.client.{js,d.ts}`（`package/client/typert`）。
- **运行时存储**：`dsh-typert-registry` 提供 `ctx.typert`（默认导出 `TypertRegistry`）。
- **组合期装配者**：`dsh-typert-loader`，inject `ctx.loader` + `ctx.typert`。

## 扩展点

- **业务包贡献反射**：加公共入口 `./typert`，导出经校验的 `TYPERT` manifest。生成的 `.d.ts` 把 `TYPERT` 标为 `unknown`，**所以贡献包不依赖运行时 registry**。发布是包级 opt-in，没有对应公共入口的包不需要 artifact。
- **业务包声明 Remote**：extend `TypertLookupMap` / `TypertContextMap`（把 Host 对象或 scoped Context 关联到 wire 身份）；生成的 artifact extend `TypertRemoteMap` / `TypertRemoteScopeMap` / `TypertRemoteNamespaceMap`。
- **身份解析的双所有权**：`ctx.typert.lookups.register()` 注册**业务包拥有**的 wire 声明与默认 resolver；`configure()` 注册**Host 组合拥有**、可异步的 resolver。两者生命周期独立——configure 可以先于 provider 到达，卸掉 configure 会恢复默认策略。`contexts.registerHost()` / `configureHost()` 对 scoped Context 身份是同一套拆分，`registerClient()` 提供对应的 Client Context binder。
- **非 Loader 组合**：直接 `ctx.typert.register(contribution)`，它在提交任何东西之前先拒绝畸形身份和重复 package-face / schema key，然后返回**精确的 Cordis effect disposer**。
- **额外注册嵌套包**：`dsh-typert-loader` 的 `packages` 配置项——Cordis fiber 不保留嵌套插件的 npm specifier，所以这个边界必须显式写。
- **Host 方法要协作式取消**：把 `signal: AbortSignal` 声明为**最后一个参数**；`InvocationDescriptor.cancellation` 记录这个保留注入点，signal 永远不会变成 JSON 参数或 lookup 字段。

## Known Limitations

- `registry`：只存生成的反射，**不合并 host 与 client 图**、不解析 TypeScript references（那是 analyzer/emitter 的事）。schema key **不含 face**，把同名 schema 从两个 face 注册进同一个 context 会按重复拒绝。
- `loader`：发现**只 import host face**；client runtime 需要另一个组合 owner。Loader entry 自动发现，嵌套或非 Loader 插件必须写显式 `packages` 或自己 `ctx.typert.register()`。
- `generator`：跳过 package export patterns（贡献包需要具体导出目标）；跨 face 的 namespace re-export 会失败（`TypeTargetModel` 还表达不了模块命名空间）；Zod emitter 只支持被建模 TS 图的**一个刻意子集**——泛型 schema 声明、conditional / mapped 这类计算根会失败；发现范围只跟随「从具体公共导出可达的源文件」，既不导出也不被该图 import 的声明**故意在包模型之外**。
- `protocol`：装饰器 marker 只含方法名与「直接 / Context 调用模式」，参数、结果、lookup、schema 反射都要走 Typert 构建流水线；Remote 装饰器只接受 public、非 static、字符串名的实例方法——SRC 执行**表达不了重载、解构、默认值、rest 参数**签名。

## 陷阱

- **`typert/protocol/` 目录存在，但 group README 的表格里没有它**（见 conflicts）。找 `@Remote` / `TypertRemoteService` 要直接进目录，别信表格。
- **包解析和已 import 的 manifest 按进程生命周期缓存**——给一个包新加 `./typert` 出口后**必须重启**才生效。
- 装载失败的分级不同：已挂载时遇到畸形 artifact 会**让 activation 失败**；之后再出现的失败只记日志，不阻止其他包注册。
- `toJSONSchema(key, params?)` 用 `z.toJSONSchema()` **不缓存**结果——JSON Schema 是在消费者边缘按需算的，热路径要自己缓存。
- 编译 face 成员由**直接的 project references** 决定，而 Typert 运行时 face 贡献由**包 subpath** 决定：一个普通单工程包只要声明 `dsh.client` 就可能同时贡献 Host 和 Client 运行时模型；只有被 `tsconfig.host.json` / `tsconfig.client.json` 显式引用的**拆分工程**才被限制在对应 face。
- 仓库的 Host tsdown 以 `tsconfig.host.json` 为唯一 program seed 跑生成，随后的 Client tsdown **既不启动 Typert 也不分析** `tsconfig.client.json`。

## 去哪深入（文件路由）

| 想查 | 去 |
|---|---|
| 子系统全景 | `docs/subsystems/typert.md` |
| `ctx.typert` 完整 API 列表 | `packages/typert/registry/README.md` §Public API |
| `@Remote` / 协议 map / 取消注入点 | `packages/typert/protocol/README.md` |
| 分析模型、emitter、发布 opt-in 规则 | `packages/typert/generator/README.md` |
| Loader 发现时序与 `packages` 配置 | `packages/typert/loader/README.md` |
| 服务实现源码 | `packages/typert/registry/src/service.ts` |
| RPC 网关侧的消费 | `packages/api/`（另有专页） |
