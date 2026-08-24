---
title: packages/bundle — dsh --profile 的可安装 patch 层
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/bundle/README.md
  - packages/bundle/base/README.md
  - packages/bundle/base/cordis.patch.yml
  - packages/bundle/headless/README.md
  - packages/bundle/headless/cordis.patch.yml
  - packages/bundle/web-app/README.md
  - packages/bundle/web-app/cordis.patch.yml
  - packages/boot/app-boot/README.md
  - packages/README.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

**Profile bundle = 一个 npm 包 + 一个 patch 文件**：manifest 里声明 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }` 就成了 `dsh --profile` 组合的可安装层。**bundle 的实体是它的 patch 清单**，有些额外附带被 patch 挂载的 runtime 胶水插件 [T1: packages/bundle/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

| 包名 | npm 名 | 一句话职责 |
|---|---|---|
| `base` | `@deepseek-ai/dsh-base` | 每个 profile 最先应用的共享 dsh 核心层（**无运行时 API**，纯 patch；78 个 `- id:` 行 [T1: packages/bundle/base/cordis.patch.yml]） |
| `headless` | `@deepseek-ai/dsh-headless` | 一次性任务模式，骑在 base 上；挂载 `headless-runner` + `headless-startup` |
| `web-app` | `@deepseek-ai/dsh-web-app` | 浏览器界面层：Web host 行 + 浏览器插件名册 + `web-runtime` 胶水插件 |

三个包的 `package.json` 都只带 `dsh.bundle.patch` 一个字段（已核对）。

## 三件套结构

本组**不是 capability seam**，是 composition 层。真正对应的三件套是：

| 角色 | 归属 |
|---|---|
| "什么是 bundle" 的定义 | **manifest 字段 `dsh.bundle.patch`**，不是这个目录 |
| 解析 / 组合 / 校验 | `dsh-app-boot` 的 `loadProfile` / `resolveBundleDir` / `composeEntries` / `healProfilesModuleFallback` |
| 具体层 | 本组三个包，**以及组外的可安装 bundle**（Codex / Claude Code subagent 包就是直接可安装的例子） |

**跨文档综合的重点**：`composeEntries` 用 include 自己的 `applyEntryPatches` 在空 entry list 上叠层，所以"组合、flag 派生、config dump"三者不可能与真正 boot 出来的东西漂移。

## 扩展点

- **做一个 out-of-tree bundle** → 发包 + 写 `dsh.bundle.patch` + `dsh plugin --profile <name> add <package>`。在库 bundle 从 dsh 安装目录解析；out-of-tree 从 profile 目录解析（`loadProfile` 两锚点解析，对"列了但没有 bundle 声明"的包 fail loud）。
- **让领域包自带可选 profile 层** → 领域包直接在自己的 manifest 加 `dsh.bundle.patch`，不必进 `packages/bundle/`。
- **覆盖 base 的某一行** → 更靠后的 bundle 层（如 `dsh-web-app`）或用户 profile `cordis.patch.yml` 按 **id** 覆盖。**patch 替换整个 `config`**，所以 mode 特定的值应放在 mode bundle，不放 base。
- **给 web 面加一个浏览器插件包** → 三处注册缺一不可：`tsconfig.client.json` 的 `references`、`packages/bundle/web-app/cordis.patch.yml` 的 `dsh.client` 行、`packages/bundle/web-app/package.json` 的 dependency [T1: packages/client/AGENTS.md]。
- **给 headless / web 加启动 flag** → 各自的 `src/startup.ts` 是普通 `cmdlineArgs` 消费者（见 `wiki/packages/boot.md`）。

## Known Limitations

`base`：

- **patch 替换整行 config** —— profile 覆盖必须把要保留的字段全部重写，没有深合并层。
- **Windows 的临时目录授权是每 session 私有子目录** —— `workspace-write` 把写限制在 workspace 加 `<temp>\dsh-<hash>`（TMP/TEMP 对受限子进程重写）；`read-only` 不给任何写权限。

`headless`：

- **只跑一个提交的任务** —— 没有交互式后续；它等 Agent 完成该区间的全部工作后打印最后一条非空 assistant 消息。
- **`ctx.appExit` 归 launcher 所有** —— 在 `dsh` launcher 之外 boot headless profile 会在 activation 期 fail loud。

`web-app`：

- **前端 dist 必须先构建好** —— `require.resolve` 失败会在 activation fail loud 并给构建提示，**没有 source-serving fallback**。
- **`lanAddresses` 是 boot 期快照** —— boot 后网卡变化不会重新广告。
- **只观测到 handoff 启动** —— 观测止于平台 opener 接受 spawn（Windows 例外，会等它那个短命 PowerShell launcher 退出）。
- **SSH 转发拥有浏览器 URL** —— 打印的是远端 loopback 端点，自动打开被抑制。
- **浏览器命令覆盖只在 launch 期** —— 被发现的 `.env` 不能设 `BROWSER`，只有继承来的值才可能生效。

## 陷阱

1. **base 的 shell 栈按平台在自己的行上开关**：`bash-sandbox` / `tool-bash` 带 `disabled: !!js process.platform === 'win32'`，孪生的 `pwsh-sandbox` / `tool-pwsh` 用反向表达式。**两个 executor 家族注册同一个 `bash` service**——所以"在 Windows 上恢复 bash"的配方必须完整（同时禁掉 pwsh 两行 **且** 重新启用 bash 两行），半截配方会在 load 时 fail loud。
2. **`fs-sandbox` 已经在围栏 `ctx.fs` 写入**，此时再挂 `dsh-fs-local` 会**双注册 `ctx.fs` 并让 load 失败**。
3. **`dsh-base` 的生产依赖闭包刻意不含 Codex / Claude Code provider**（也不含 Claude Agent SDK 与 Codex wrapper/平台负载）；需要时由 profile 单独安装对应的 product provider bundle。
4. **headless 与 web-app 是同一个 base 上的兄弟面，互不挂载**。headless 不挂 Host / HTTP server / Web runtime / 浏览器插件，也不开监听端口。
5. **headless 的退出码来自 turn 结果**：最终 `turn/end` completed → 0，否则 1；terminal `error` 还会把 code 和 message 写 stderr，成功运行的 stderr 保持为空。
6. **`--host 0.0.0.0` 会在 publish `webStartup` 之前被拒**——CLI 有意不支持全网卡绑定。`dsh --profile web --help` 不会起服务器（flag 配置的行注入 service 后才读 lazy config）。
7. **SSH 场景**：`SSH_CONNECTION` / `SSH_TTY` 非空时保留 URL 行但抑制浏览器 handoff。
8. **headless 复用了和 Web 面一样的临时 Code Mode 开关** `mode: !!js process.env.DSH_TOOLS_MODE`，并把 `@deepseek-ai/dsh-code-runtime-worker-thread` 作为**核心执行能力**（不是 Web 组件）插入 [T1: packages/bundle/headless/cordis.patch.yml]。
9. **`dsh-web-app` 的 client HMR 链是常开的**（`dsh-client-hmr`），只是在没有 rebuild watcher 改写 client bundle 时保持 idle；而 headless 直接 `- id: hmr / disabled: true`。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| bundle 身份定义、在库/out-of-tree 解析 | `packages/bundle/README.md` |
| 每一行为何存在（行内注释即文档） | 三个 `cordis.patch.yml` 本身 |
| base 的平台门控与 Windows 权限面 | `packages/bundle/base/README.md` |
| headless 运行序列与退出码 | `packages/bundle/headless/README.md` |
| web 启动 flag、浏览器 handoff、prompt section | `packages/bundle/web-app/README.md` |
| profile 目录布局、层序、`healProfilesModuleFallback` | `packages/boot/app-boot/README.md#Profiles` |
| 最终组合出来的行清单（生成物） | `apps/cli/composition.md` |
| 新增浏览器插件包的三处注册 | `packages/client/AGENTS.md` |
