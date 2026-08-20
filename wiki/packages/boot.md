---
title: packages/boot — app-bin 启动胶水与 profile 机制
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/boot/README.md
  - packages/boot/app-boot/README.md
  - packages/boot/cmdline/README.md
  - packages/boot/cmdline/src/index.ts
  - apps/cli/README.md
  - apps/cli/composition.md
  - packages/bundle/README.md
  - packages/README.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

`apps/cli` 与 `packages/examples` 各个 demo bin **共用的启动库**：`.env` 分层加载、fail-loud Loader 守卫、profile / bundle / patch 的组合与热更、以及"launcher → app"的命令行交接 [T1: packages/boot/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

| 包名 | npm 名 | 一句话职责 |
|---|---|---|
| `app-boot` | `@deepseek-ai/dsh-app-boot` | boot 序列本体：`boot()` / Loader 守卫 / profile 机制 / 用户 patch 热更（库，不注册 ctx key） |
| `cmdline` | `@deepseek-ai/dsh-cmdline` | launcher 与 app 之间的命令行交接：`ctx.cmdlineArgs`、`ctx.appExit` |

## 三件套结构

本组**不是 capability seam**，`app-boot` 是纯库（组 README ctx key 一栏写 "library for the bins"）。`cmdline` 更接近三件套：

| 角色 | 归属 | 说明 |
|---|---|---|
| 值的提供者 | launcher（`apps/cli` 等） | 在任何 tree entry 挂载**之前**调 `provideCmdline(ctx, host)`，`ctx.provide('cmdlineArgs' \| 'appExit', …)` [T1: packages/boot/cmdline/src/index.ts] |
| 适配器 | `parseCmdline(ctx, program)` | 只是一个 commander 适配器，不是框架 |
| Consumer | **app 自己的普通插件** | 注入 `cmdlineArgs`、解析、然后 `ctx.provide()` 一个 app 自有 service（如 `webStartup` / `headlessStartup`） |

关键结论（跨文档综合）：**launcher 只认自己的 flag（`--profile` / `--patch` / config dump），第一个它不认识的 token 之后全部原样交给 tree**——所以 `--help` 文本、flag 家族、解析错误都归 app，不归 launcher。

## 扩展点

- **给 app 加启动 flag** → 写一个普通插件：`inject = ['cmdlineArgs']`，用 commander 构造 program，在 action 里 `ctx.provide('<app>Startup', …)`，最后 `parseCmdline(ctx, program)`。它的 Loader 行没有任何 launcher 标记或特殊 kind。
- **让别的行读到这些 flag** → 那些行用普通 `inject: [webStartup]` + `config: { port: !!js ctx.webStartup.port ?? 3080 }`。Loader 会**推迟该行的 `!!js` 插值**直到它声明的注入全部激活，所以 `ctx.webStartup` 可以直接读。
- **做一个新的 profile 层** → 发一个 npm 包，manifest 写 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`，再进 profile 的 `dsh.profile.bundles`（详见 `wiki/packages/bundle.md`）。
- **在 boot 之前插入 host 准备逻辑** → `boot()` 的可选 `prepare` 钩子（可用 Loader、可提供 launcher-owned context slot），在 config-tree entries 挂载前运行。
- **让 agent 知道 harness 源码位置** → boot 后调 `addHarnessSourceSection(ctx, sourceRoot)`（section 名 `'harness:source'`）。
- **嵌入到别的 Node 项目** → 给 `boot()` / `mountRootInclude()` 传 `bareModuleBaseUrl`，让已安装的包树保持权威。

## Known Limitations

`app-boot`：

- **Bare package specifier 依赖 Loader 内部**：生产 bin 需要 Loader 的可选原生 helper（`node-addon-require-builtin`）；没有它的进程内调用者只能用可解析的相对/file specifier 或自带模块解析钩子。
- **snapshot replay 的换名是 basename 特定的**：只有以 `cordis.yml` / `cordis.yaml` 结尾的 config 才映射到同级 `cordis.snapshot.yml`。
- **环境发现是 launch-scoped**：`loadLayeredEnv` 只读一次"调用目录 + Harness home"，不向上搜父目录，也不跟随后来选定的 workspace。
- **用户 patch 替换整个 config**：id 定向的 patch **不做深合并**，profile 覆盖必须把要保留的字段重新写一遍。

`cmdline`：

- **launcher flag 必须在 app 参数之前**（位置切分）；launcher 的解析器会吃掉一个 `--`，所以要让字面 `--` 活到 app，需要 `-- --`。
- **app 自有 service 没有静态声明的 provider**：bundle 漏掉 provider 行时，失败发生在 settlement（pending entry 指名缺失 service），**不是 load 时**。
- **替换整行 `config` 的用户 patch 会丢掉表达式**——保留表达式才是"flag 胜出"的机制。

## 陷阱

1. **`!!js` 只在两处合法**：plugin `config` 之下和 entry 的 `disabled`；其他元数据保持字面量。写成 `!js`（单感叹号）无效——这是仓库根 AGENTS.md 明写的坑。
2. **profile patch 的层序是"每 profile 先、home 级后"**，也就是 `$DSH_HOME/cordis.patch.yml` **压过** `profiles/<name>/cordis.patch.yml`。直觉容易搞反。
3. **空文件 ≠ 空层**：patch 文件为空或只有注释会 **throw**（它解析出的是 nothing 而不是 list）；要停用这一层请写 `[]`。
4. **patch 指到不存在的 entry id 只是 stderr warning**，不是错误——静默生效失败的常见来源。
5. **`.env` 的优先级是"继承环境 > 项目 `.env` > 用户 `.env`"**，且 bootstrap-only 的文件变量会被**大小写不敏感地拒绝**；托管凭据在 `.credentials.yaml` 里，`.env` 里的凭据只是低优先级 fallback。
6. **fail-loud 的 release 窗口是有闩的**：`installFailLoud` 在 release 进行中保持安装且 latched——**第一个 rejection 才是被上报的那个**，之后的（包括 teardown 自己的）会被吞掉以免 teardown 中途把进程打死。`FAIL_LOUD_RELEASE_TIMEOUT_MS` 只能延迟致命退出，不能取消它。
7. **两个断言别搞混**：`assertEntriesLoaded` 抓"settled 树里有 enabled 但无 fiber 的 entry"；`assertEntriesActivated` 额外 await 每个 enabled entry，报出失败者的原始 stack 或 pending 者未满足的 service。
8. **`healProfilesModuleFallback` 维护的是一个扁平的 `$DSH_HOME/profiles/node_modules` 符号链接目录**——bare 插件名靠 Node 普通父目录遍历解析，而不是 pnpm 管理在库包。新增 bundle 行时，包必须被 app 或某个 bundle 的 manifest 声明，否则 import 失败。
9. **`loadProfile` 会把"恰好等于安装自带的 bundle 元组"的清单规范化成模板**；多一项、少一项或顺序不同都会被判为 user-owned 并原样保留。
10. **`cordis:group` 与 `cordis:include` 一起注册**，且都走 ambient 模块管线而非被包含树自己的 specifier 解析——这正是 Harness home 下的 agent preset 能用 group 行的原因。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 组分工 | `packages/boot/README.md` |
| 每个导出的确切职责（一张表） | `packages/boot/app-boot/README.md` 顶部导出表 |
| profile / bundle / patch 层序、`.env` 与 `cordis.patch.yml` 契约 | `packages/boot/app-boot/README.md#Profiles` |
| Loader settle 失败、terminal 归还、release 闩锁 | `packages/boot/app-boot/README.md` 正文第 26–28 段 |
| launcher/app 命令行切分与注入排序 | `packages/boot/cmdline/README.md` |
| `provideCmdline` / `parseCmdline` 实现与错误文案 | `packages/boot/cmdline/src/index.ts` |
| `dsh` CLI 本身（source-launch 钩子、flag 家族） | `apps/cli/README.md` |
| 实际组合出来的行清单 | `apps/cli/composition.md`（生成物） |
| bundle 是什么 | `packages/bundle/README.md` |
