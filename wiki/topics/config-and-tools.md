---
title: 配置来源与覆盖顺序、工具目录、运行模式
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - .agents/notes/implemented/architecture/2026-08-04-configuration-source-ownership.md
  - packages/boot/app-boot/README.md
  - packages/boot/app-boot/src/index.ts
  - apps/cli/reference/README.md
  - apps/cli/src/profile-boot.ts
  - packages/bundle/base/cordis.patch.yml
  - packages/bundle/web-app/cordis.patch.yml
  - packages/credentials/credentials-local/README.md
  - packages/util/launch-environment/README.md
  - apps/cli/config/agent-presets/
  - packages/client/ui-agent-preset/src/client/locales.ts
  - docs/config-catalog.md
  - docs/tool-catalog.md
  - docs/subsystems/permission-presets.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

# 配置、工具目录与运行模式

路径均相对 `upstream/deepseek-harness/`。**排错第一入口是下面第一张梯子表**。

## 配置从哪来、按什么顺序覆盖

DSH 有**两条独立梯子**，非机密值一条、凭据另一条，官方明确拒绝合并（`.agents/notes/implemented/architecture/2026-08-04-configuration-source-ownership.md#decision`）。

**非机密值（高 → 低）**，每层对应锚点：

| 层 | 具体是什么 | 去哪看 |
|---|---|---|
| explicit for this run | 单次操作覆盖、CLI 参数（如 `dsh web --port`） | `apps/cli/reference/README.md`（App arguments 节） |
| user settings | `$DSH_HOME/settings.yaml` 的 namespace section | `packages/settings/settings-file/README.md`、`docs/config-catalog.md#deepseek-aidsh-settings-file` |
| composition | profile 的 bundle patch 层 + 用户 patch 层 + `--patch` overlay，**统统同一层** | `apps/cli/src/profile-boot.ts#composeProfile` |
| this launch's shell | 启动进程继承的环境变量 | `packages/util/launch-environment/README.md` |
| discovered file | `<调用目录>/.env`，然后 `$DSH_HOME/.env` | `packages/boot/app-boot/src/index.ts#loadLayeredEnv` |
| defaults | schema 默认值 / provider 公开默认值 | `docs/config-catalog.md` |

三个必须记住的推论：**(1)** settings 在 composition 之上——settings seam 把插件的 cordis entry config 当成 `base` 层、用户 section 盖在上面，seam 分不出「bundle 设的」还是「`--patch` 设的」，全是 entry config；产品 CLI **没有任何 flag 能压过已存的 settings.yaml**，要压过就得自己出 bin / loader 树或不挂 settings provider。**(2)** composition 在环境变量之上，所以 shell 里过期的 `DEEPSEEK_BASE_URL` 改不动已配置的 endpoint。**(3)** `.env` 被禁止设置 bootstrap 类变量，在物化前就 reject（详见「陷阱」）。

`composition` 内部的层序（`apps/cli/reference/README.md` Profile boot 节 + `apps/cli/src/profile-boot.ts#composeProfile`，后者是唯一真源）：

```
dsh.profile.bundles 里每个 bundle 的 cordis.patch.yml（按清单顺序）
→ $DSH_HOME/profiles/<name>/cordis.patch.yml
→ $DSH_HOME/cordis.patch.yml            ← 机器级，故意排在 per-profile 之后（更高）
→ 每个 --patch <path>（argv 顺序）
→ 【文档未列】app 自己追加的 agent-presets roots overlay
→ 【文档未列】DSH_TELEMETRY_DISABLED 推出的 telemetry 关闭 patch
```

## 凭据管理（credentials）

单独一条梯子，**继承环境赢**（`packages/credentials/credentials-local/README.md` 表格）：

| 层（高→低） | source id | 可写 |
|---|---|---|
| 继承的 process 环境 | `env` | 否（`set`/`unset` 直接 reject，不是静默失败） |
| `$DSH_HOME/.credentials.yaml` | `file` | 是（Models 页写这里） |
| `<调用目录>/.env` 然后 `$DSH_HOME/.env` | `project-env` / `user-env` | 否 |

- 配置里只放**引用名**（`apiKeyEnv: DEEPSEEK_API_KEY`），值永远在 provider 里；`resolve(ref)` 每次操作现读（LLM adapter 每次 model request 一次），所以换 key 下一个请求就生效，不用重启插件。
- `.credentials.yaml` 是 `0600` + 目录 `0700`，POSIX 上多一个 group/other 权限位就在解析前 fail 并提示 `chmod 600`；**它不进 `process.env`**，而 `$DSH_HOME/.env` 会进。
- 这道墙挡的是别的 OS 用户，**挡不住模型**：bash/fs 工具同 uid 运行，`workspace-write` 只限制写不限制读。
- **0.1.1-rc 起文档格式与事件翻新**：`.credentials.yaml` 变为 versioned 文档（顶层 `version: 1`，下分 `refs:`（引用名→值）与 `records:`（命名凭据记录：`kind: grant` 存 provider 不解释的 payload，如 OAuth token 组；`kind: api-key` 存 env 映射或仅确认使用环境凭据链）两个 key space）；boot 识别精确匹配的旧平铺格式会**原位迁移**（原行逐字节嵌进 `refs:`），live reload 不迁移。热更新事件 `credentials/updated` 拆为 `credentials/reference-updated`（refs 维度）与 `credentials/record-updated`（records 维度）[T1: packages/credentials/credentials-local/README.md]。
- **新增 authorization flow seam**：`@deepseek-ai/dsh-authorization`（`packages/credentials/authorization`，无 config 插件）提供 `ctx.authorization`——Authorization flow registry。flow 由知道如何取得某种凭据的插件注册、按其写入的 record 键控；seam 只拥有会话与 one-attempt-per-key 生命周期，不管协议本身。当前唯一消费者是 `llm-pi-ai`，事件 `authorization/settled`（emit）[T1: docs/capability-seams.md, docs/event-producer-consumer.md]。

## 四种运行模式的真实定义与差异

官方口径的四种模式 = **`apps/cli/config/agent-presets/` 下的四个 agent preset 目录**，不是 profile、不是 bundle。

| 目录 id | 中文名（`preset.yml`） | 英文名（locale 表） | order | 相对 standard 的差异（按 plugin row diff） |
|---|---|---|---|---|
| `standard` | 标准模式 | Standard mode | 1 | 基准，29 rows |
| `code` | PTC 模式 | PTC mode | 2 | **只多 1 行**：`@deepseek-ai/dsh-agent-tool-presentation` + `config.mode: code` |
| `minimal` | 极简模式 | Minimal mode | 3 | 完全另一套 10 rows，见下 |
| `cordis` | 创造模式 | **Creator mode** | 4 | **只多 1 行**：`@deepseek-ai/dsh-tool-cordis`，外加自带 2 个 skills |

- **「Creator 模式」的目录 id 是 `cordis`**，两个名字都真实存在：`preset.yml` 里是「创造模式」，`packages/client/ui-agent-preset/src/client/locales.ts#presetCordisName` 里是 `Creator mode`。找 `creator` 目录是找不到的。
- `minimal` 的 10 rows：`dsh-persona`（`complete: true` + `includeRuntimeContext: false`，即固定整段 system prompt 为 `You are a helpful software engineer assistant.`）、一个 `isolate: {terminals}` 组（`dsh-terminal` + `dsh-terminal-bash` + `dsh-tool-bash-persistent`，win32 上换 pwsh 双胞胎）、一个 `isolate: {fs}` 组（`dsh-fs-local` + `dsh-tool-str-replace-editor`）。**没有 compaction、没有 skill、没有 plan、没有 subagent、没有 web**。
- 四种模式**只在 Web surface 存在**：`agent-presets` 行只在 `packages/bundle/web-app/cordis.patch.yml` 里 insert。`dsh --profile headless` 没有 preset 名册，它的 agent 组合直接来自 base+headless bundle 的 host 平面行。
- 与之配套：web-app bundle **把 base 里所有 model-facing 工具行 `disabled: true`**（tool-bash / tool-fs / tool-skill / plan-mode / compaction / subagent …），因为这些改由每个 session 的 preset 拥有；base 保留它们是给单会话 TUI 用的。同一批工具在 Web 走 agent 平面、在 headless 走 host 平面。

## `dsh --profile` 与 bundle 的补丁层机制

- **profile** = `$DSH_HOME/profiles/<name>/`，内含 `package.json`（out-of-tree 插件 `dependencies` + `dsh.profile.bundles` 有序清单）与用户自己的 `cordis.patch.yml`。
- **bundle** = 一个 npm 包，manifest 里声明 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`；不在 `packages/bundle/` 也算数，身份由 manifest 决定（`packages/bundle/README.md`）。in-box 三个：`dsh-base` / `dsh-web-app` / `dsh-headless`。
- **patch 语义是整体替换不是深合并**：id-targeted patch 替换目标行**完整的 `config` 值**，保留的字段必须重写一遍；`insert` 追加新行；`disabled` 与 `config` 支持 `!!js`。patch 命中不到的 id 只是 stderr warning。空文件或纯注释文件会 throw——关掉这一层要写 `[]`。
- `web` / `headless` 是模板首次使用自动 init；其他名字 fail loud，必须先 `dsh plugin --profile <name> add <package>`。**bundle 增删要重启 profile**（membership 启动时固定），而两个 `cordis.patch.yml` 的普通编辑走热重载（`watchUserPatches`，失败保留 last-good 并广播 `hmr/config-update-failed`）。
- 想看 composed 结果而不启动：`--dump-default-config`（只有 bundle 层）/ `--dump-config`（加 user 层与 `--patch`）。两者都打注释标出每行来自哪个文件、被哪些 overlay 改过，`!!js` 保持未求值；dump **不跑 app cmdline provider**，带 app 参数会被拒。
- flag 之所以能压过配置文件里写死的值，是靠行里保留的 `!!js ctx.webStartup.port ?? 3080` 表达式——**用户 patch 把整个 `config` 换成字面量就废掉了这个机制**。

## 模型可见的工具目录

`docs/tool-catalog.md` 是唯一入口，且是 **generated**：generator 真的把每个 tool plugin boot 起来读 `ctx.tools.schemas()`（不是 AST 静态扫描），因为 schema 会被 runtime enum、拼接描述、config 决定的名字影响。它按**默认 config** boot；`packages/*/tool-*` 有完整性守卫，新 tool 包漏掉会 fail。

**路由**：先看开头的 `## Tool Package Map` 表（tool 包 → 模型可见名 / Requires 的 seam / 写什么事件 / shipped alias / deployment note），再跳到对应包节看逐个 JSON Schema。

按能力分组的模型可见名（同名跨包会重复，如 `bash` 一次性版与 persistent 版）：

| 类别 | 工具名 |
|---|---|
| shell / 后台 | `bash`、`pwsh`、`job_kill`/`job_list`/`job_output`、六个 `terminal_*` |
| 文件 | `read`/`write`/`edit`/`read_image`、`str_replace_editor`、`glob`/`grep`、`lsp` |
| 委派 | `subagent`（默认名可配）、`subagent_fork`、`send_message`/`interrupt_agent`/`list_agents`、`report`、`workflow`、`ralph` |
| 规划 / 状态 | `todo_write`、`exit_plan_mode`、`create_goal`/`get_goal`/`update_goal`、三个 `schedule_*` |
| 检索 / 交互 / 技能 | `web_search`/`web_fetch`、五个 `session_*`、`ask_user_question`、`skill` |
| 自我修改 / Code Mode | 七个 `cordis_*`（需 `ctx.dynamicCordisRunner`）、`run_code`（保留传输，不受 capability layer 过滤） |
| 实验 | agent-team 十件（`spawn_teammate`/`team_task_*`/`wait_agent`…），shipped base 里禁用 |

三个关键机制：`ToolPresentationMode`（`native` / `code` / `both`，`code` 下只发 `run_code` + 生成的 SDK）；`ToolRestriction`（每个 scope 的 `allow`/`deny`，多重限制取交集，**scope 自己注册的工具豁免**，`run_code` 也豁免）；MCP 工具走 `mcp__<serverName>__<rawName>` 命名，不在本 catalog 内。

## 权限预设与审批流

两个**互相独立**的开关，被 `ctx.permissionPresets` 打成一个用户可选的包：`sandbox/mode`（`read-only` / `workspace-write` / `danger-full-access`）与 `approval/policy`（`ask` / `never`）。

- 插件默认表只有两项（`workspace-write`、`danger-full-access`），但**出厂 CLI 的表是三项**：`packages/bundle/base/cordis.patch.yml` 的 `permission` 行显式配了 `read-only` + `workspace-write` + `danger-full-access`。
- preset 层**不做任何强制执行**，只记录意图并分别调 `setSandboxMode` / `setApprovalPolicy`；真正执行的是 sandbox 与 approval 各自的消费者。`set()` 先追加 log-only 的 `permission/preset` 事件，再只对**真的变了**的 knob 调 setter；重选当前 preset 什么都不写。
- `custom` 是**派生态**：knob 组合匹配不上任何表项时 `current()` 返回它。可以显示，不能选，不能持久化。
- 审批：`ctx.approval.request(req)` 要求会话处于 open turn，先追加 `approval/asked`，走 `approval/request` waterfall 拿一个 outcome，再追加 `approval/decided`。outcome 是封闭集 `allowed-once` / `rejected` / `cancelled` / `unavailable`，**只有 `allowed-once` 是放行**；没有 answerer、answerer 抛异常、返回值不在词表里，一律 `unavailable`（fail closed）。`never` 在 waterfall **之前**就返回 `rejected`，所以事后 `prepend` 一个 answerer 也绕不过去。
- 会话创建时把 `permission/preset` + `sandbox/mode` + `approval/policy` 三件事**钉进** session，之后改 settings 不影响已开会话。

## 陷阱

- **`.env` 里放 `DEEPSEEK_BASE_URL` 会直接让启动失败**，不是被忽略。`packages/boot/app-boot/src/index.ts#BOOTSTRAP_NAMES` 把它和 `DEEPSEEK_SEARCH_BASE_URL`、`PATH`、`SHELL`、`HOME`、`NODE_OPTIONS`、`EDITOR`/`PAGER`/`BROWSER`、`BASH_ENV`、各种 proxy/CA 变量一起拒了；前缀 `DSH_`、`XDG_`、`DYLD_`、`BASH_FUNC_` **整个命名空间**被拒，大小写不敏感（`https_proxy` 不是绕过口）。`DEEPSEEK_API_KEY` 不在此列，放 `.env` 合法。没有 opt-out。
- **`$DSH_HOME/profiles/<name>/cordis.yml` 每次 boot 都会被覆写**（`prepareProfile` 无条件 `writeFileSync` 一个空 entry list），因为 vendored Loader 的 tree write-back 会把 composed 行烤进去导致下次 boot 重复 insert。要改配置只能改 `cordis.patch.yml`。
- **`--patch` 不是最后一层**：`composeProfile` 之后还会追加两个 app 自有 overlay，其中 `agent-presets` 那个会用出厂 preset 根**整体覆盖** `roots` 字段。想加自己的 preset 根目录，改 `roots` 是无效的——用 `$DSH_HOME/.agent-presets`（`includeUserRoot` 追加的可写 user 根）。
- **极简模式的文件写入不受 sandbox 约束**：`minimal/agent.cordis.yml` 在 `isolate: {fs: true}` 里挂了裸 `@deepseek-ai/dsh-fs-local`，遮蔽了 host 的 `dsh-fs-sandbox`。它的 persistent bash 仍然消费 host sandbox policy，但 `str_replace_editor` 的写不过那道 fence。
- **改出厂 preset 的 `preset.yml` 名字/描述在 Web 上没有任何效果**：`presetDisplayText()` 对 `trust === 'system'` 且 id ∈ {standard, code, minimal, cordis} 的行，一律用 client locale 表的文案覆盖文件元数据。复制成 user preset 后才会读文件里的名字。
- **`DSH_TOOLS_MODE` 是进程级临时开关**（web-app 与 headless bundle 的 `tools` 行注释自称 TEMPORARY workaround），选 `native|code|both`，别的值 boot 失败；而 `code` preset 用的是 per-agent 的 `agent-tool-presentation` 行。两者层级不同，别混。
- **`dsh plugin ... add` 只装 host 侧 provider**，不会给 agent 发工具。Codex / Claude Code subagent 是两步：装 bundle 并重启 profile，再复制 preset 并去掉对应 `tool-subagent-*` 行的 `disabled: true`。
- Windows 上 base bundle 用 `disabled: !!js process.platform === 'win32'` 成对切换 bash/pwsh 两套栈；想在 Windows 上换回 bash，**必须同时**禁 `pwsh-sandbox`/`tool-pwsh` 并重新启用 `bash-sandbox`/`tool-bash`——两族注册同一个 `bash` service，半套配方会在 load 时 fail loud。

## 去哪深入

| 问题 | 去这里 |
|---|---|
| 某插件的 `config:` 能写什么 / `Requires` 哪些服务 | `docs/config-catalog.md`（109 个包节，generated） |
| 模型看到的工具 schema | `docs/tool-catalog.md`（26 个包节） |
| CLI 精确层序、flag、shutdown、部署默认值 | `apps/cli/reference/README.md` |
| profile / bundle / `.env` / user patch 机制原文 | `packages/boot/app-boot/README.md#profiles` |
| 出厂 host 组合的实际行清单 | `packages/bundle/base/cordis.patch.yml`、`apps/cli/composition.md`（generated 图） |
| 出厂 agent preset 实物 | `apps/cli/config/agent-presets/{standard,code,minimal,cordis}/agent.cordis.yml` |
| preset / settings seam 的 API 与限制 | wiki `packages/preset.md`、`packages/settings.md` |
| 权限 / 审批 / sandbox | `docs/subsystems/{permission-presets,approval,sandbox}.md` |
