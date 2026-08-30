---
title: packages/skill — skill capability family
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/skill/README.md
  - packages/skill/skill/README.md
  - packages/skill/skill/package.json
  - packages/skill/skill/src/index.ts
  - packages/skill/skill-filesystem/README.md
  - packages/skill/skill-filesystem/src/index.ts
  - packages/skill/tool-skill/README.md
  - docs/subsystems/skills.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-30
asked_by: agent
---

## 一句话定位

`SkillRegistry`（`ctx.skills`）是一个**纯 provider 注册表**：它不知道 skill 来自本地文件、插件内嵌数据还是 HTTP，只负责发现、去重、按层合并与按名加载；模型侧的目录与 `skill` 加载工具是独立的 Consumer。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。组 README 补充：该能力「remains outside the core control spine」，可用 local / embedded / remote provider 而不改变 model-facing 契约。

## 包清单

| 包名 | npm | 一句话职责 |
|---|---|---|
| `skill` | `@deepseek-ai/dsh-skill` | Service Definition：provider 注册与查找、分层解析、`renderSkillContent` |
| `skill-badge` | `@deepseek-ai/dsh-skill-badge` | Provider：贡献固定的 `dsh-badge` 一个 skill（"powered by dsh" 素材） |
| `skill-filesystem` | `@deepseek-ai/dsh-skill-filesystem` | Provider：扫描本地 project / custom / user 根目录发现 skill |
| `tool-skill` | `@deepseek-ai/dsh-tool-skill` | Consumer：发布 skill 目录并注册模型侧 `skill` 加载工具 |

## 三件套结构

- **Service Definition**：`dsh-skill`，ctx key `ctx.skills`，`class SkillRegistry extends Service` [T1: packages/skill/skill/src/index.ts#SkillRegistry]。
- **Service Provider**：`dsh-skill-filesystem`（本地文件）、`dsh-skill-badge`（内嵌单例）。
- **Consumer**：`dsh-tool-skill`，`inject: ['agents', 'tools', 'skills']`。

注册表**分层**（layered over `@deepseek-ai/dsh-scope`，与 tools registry 同一形状）：注册落进调用上下文 scope 所在的层——host 行与仓库插件进全局层，agent preset 常驻组合挂载的插件进该 preset 的层；读取时合并全局层与观察者 scope 链，**近层直接赢下同名**，rank 只在同一层内决胜。

## 扩展点

想接一个新的 skill 来源（远端仓库、数据库、企业知识库）：

1. 依赖 **`@deepseek-ai/dsh-skill`**（Service Definition），不依赖 `skill-filesystem`。
2. `ctx.skills.registerProvider(create)`：工厂**同步**执行，收到 `{ signal, invalidate }`；远端 setup、鉴权、发现都放进 await 的 `list(options)` 里，不要放工厂里。
3. `list()` 返回数组 = 完整发现；返回 `{ candidates, complete: false }` = 拿到了可用候选但无法建立权威观测。
4. 可变来源必须**自己保留并调用注册作用域的 `invalidate()`**——注册表没有 TTL，猜不到远端变了。`invalidate()` 只在该次注册仍然活跃时生效，晚到的回调不会误伤同名替代者。
5. 赢下名字的 provider 会拿回它自己 `list()` 时返回的那个 candidate 与**不透明 `locator`**（文件路径、URL、id、version 句柄随便），`get()` 每次都问 provider 要 body，注册表**不缓存 body**。

读侧三个 API 共享同一组 `{ cwd?, signal?, scope? }`：`snapshot()`（返回 `{ skills, complete }`，永不缓存）、`list()`（按名排序的胜出摘要）、`get(name, …)`。另有 `ctx.skills.register(skill)` 用于内嵌 runtime skill，rank 固定 `250`，`provider` 名保留字 `runtime`，同层同名 first-wins。

**调用策略必须在消费方边界自己判**：`SkillSummary.invocation` 是 `{ modelInvocable, userInvocable }` 两个独立布尔，四种组合都保留；`ctx.skills.get()` 是**policy-neutral 的可信加载原语**，任何面向用户或模型的 Consumer 必须先过 `isModelInvocable(skill)` / `isUserInvocable(skill)`。

渲染统一点：`renderSkillContent(skill)` 产出规范的 `<skill_content>` 块（转义 `name`、资源提示、逐字 body），`dsh-tool-skill` 的工具结果与用户显式手势注入走的是同一份，所以模型看到的形状与谁发起加载无关。同包还声明了 `skill-invocation` 这个 `MessageSource` kind（`{ name, form: 'instructions' }`）。

事件只有一条：`skills/change`，**无过滤、无 catalog、无 diff**，每个消费方自己带 lookup options 重新 `snapshot()`。监听器 throw 或 reject 会被记录，不能否决注册表变更，也不能饿死后续监听器。

配置只有一个键：`collectCacheMaxEntries`，默认 `128`。

## Known Limitations

- `dsh-skill`：失效由 provider 驱动（无 TTL）；**provider 顺序查询**，一个慢的合作型 provider 会拖住其后所有 provider，取消只能停掉调用方的等待、停不掉不合作 provider 仍在跑的活；不完整观测不被保留（无 last-good catalog、无 per-provider 诊断）；重复名 first-wins，近层静默遮蔽远层且**没有 API 能查看被遮蔽的定义**。
- `dsh-skill-filesystem`：发现**只有一层深**，只认 `<root>/<name>/SKILL.md` 与 `<root>/<name>.md`；project scope 是最近的 `.git` 祖先，没有该标记就退回传入 cwd，无 monorepo 子项目选择；格式错误的条目只 warn 后消失，模型目录里区分不出「不存在」与「非法」；启动时缺失的 root 用 `fs.watchFile` 按 `watchPollIntervalMs` 轮询一个路径段直到 Chokidar 能挂上；**无 body 版本协议**。
- `dsh-tool-skill`：目录省略 `whenToUse`、来源与 provider 元数据，路由只靠名字 + 截断的描述；**加载的正文没有大小上限**（只有目录描述被截断）；resources 只是提示不是附件；加载是一次性文本（无流式/缓存句柄）；目录替换是整表替换——改一个名字或描述就要重发全部可见摘要；body 未版本化，只改 body 不改 digest 也不通知模型。
- `dsh-skill-badge`：只贡献一个固定 skill，无运行期定制；远端 Markdown 走 Shields.io。

## 陷阱

- **`skill-badge` 默认关着**：出厂 CLI composition 把该插件行标为 `disabled: true`，用户必须显式启用 `skill-badge` 行，skill 才会进目录。
- `provider.name` 在**层内**唯一，重复即 throw；`runtime` 是保留名。注册失败会 abort 该次注册的 signal。
- provider 返回的定义如果 name 与选中的 candidate 不一致，注册表会拒绝这次陈旧选择，并**内部 invalidate 该 provider**，下次 snapshot 重新发现其目录。
- 一个持续自我 invalidate 的 provider 不会独占调用方：注册表在重试也被抢占后，直接以 incomplete + 不缓存的方式返回候选。
- provider 对象、lookup options、candidates、definitions 都是**借用的 readonly**，不克隆不重绑，调用方与 provider 双方都必须守住这个契约。
- `list()`/`snapshot()` 返回的东西不带调用策略过滤——忘了加 `isModelInvocable` 就会把仅供人类的 skill 暴露给模型。

## 去哪深入

- 组结构与 ctx key → `packages/skill/README.md`
- 完整 API、provider contract、分层与 rank 规则、invocation policy 四象限表 → `packages/skill/skill/README.md`
- 本地发现根、`SKILL.md` 解析、watcher 策略 → `packages/skill/skill-filesystem/README.md`
- 模型侧目录快照与 `skill` 工具行为 → `packages/skill/tool-skill/README.md`
- 子系统参考（discovery priority、catalog snapshots、`skill` loader）→ `docs/subsystems/skills.md`

## 2026-08-30 · rc.2 专项复验：per-Session 选择与 AgentPreset

- rc.2 的 Skill 公共面没有“某 Session 已选择 0..N Skill”的 header、event、projection 或启停 API。`registerProvider()` / `register()` 只是当前 Cordis scope 的运行期注册；`list()` / `snapshot()` / `get()` 只按 `cwd`、`scope`、`signal` 观察当前目录。
- scope-local provider能在同一官方SkillRegistry/`dsh-tool-skill`链中增加外部definitions，并对**同名**farther entry做nearest-layer shadow；但目录按global→ancestor→nearest合并，没有allow/deny/provider-exclusion filter。它无法隐藏继承层中其他名字的Skill，因此不构成“base之上只看selected exact set”的active-set contract。
- provider的`locator`是borrowed opaque handle，Registry只把winning candidate交回同一provider并校验definition shape/name；provider可自带digest/ref，但DSH不验证content-addressing、immutability或revision，也不持久locator/body。
- AgentPreset 可通过普通 plugin rows 间接决定某 Session 可见的 Skill provider 层；若每个内容摘要 id 对应一份不可变 composition，可以在产品层把 selection 编码为 preset id。但 DSH Session 仍只持久化 id，不持久化 exact Skill 集合或 preset 定义。
- `dsh-tool-skill` 的 durable `skill-catalog` 消息记录模型当时看到的目录，并在 provider 变化时追加 replacement；它不是 selection truth，也不会在冷恢复时重建已经消失的 provider/definition。
- 因此“无需宿主另存或重施加、仍能恢复任意 0..N immutable Skill 选择”缺少的最小闭环，是 DSH-owned 的 durable composition/selection snapshot（或等价的官方持久 definition store）以及 cold-resume 前按该 identity 解析、校验并 mount 的公开契约。仅增加一个宿主提供的 resolver 仍会把定义真源留给宿主，不能满足该限定。
- `dsh-v0.1.2-alpha.1` tag 增加 Session-addressed cold skill list，但未增加 selected-skill 集合语义；相关 skill 包截至本次查询没有 alpha.1 npm 版本。

## 2026-08-30 · rc.2 正式 frontmatter 与发布入口

- 正式包为 `dsh-skill`（Service Definition）、`dsh-skill-filesystem`（Provider）、`dsh-tool-skill`（Consumer）与 `dsh-skill-badge`（bundled Provider）；有效入口均为根、`./invariant`、`./package.json`。manifest 的 `./src/*` 在 tarball 中无对应 `src` 文件，不能作为 out-of-tree import。
- filesystem frontmatter 必填 `name` / `description`；可选键精确为 camelCase `whenToUse`、`metadata`、`disable-model-invocation`、`user-invocable`。`when-to-use` 不会成为 `whenToUse`；`modelInvocable` / `userInvocable` 是解析后的公共 policy 字段，不是接受的 frontmatter，旧 camelCase invocation 键会让整条 Skill 发现失败。
- `skill-catalog` durable message 只记录当时发布给模型的 `{name, description}` 全量目录；内部 digest 也只覆盖这些条目且不持久化。它不证明 body revision、provider locator、active selection 或恢复时可重建 definition；空 replacement 只退役模型旧目录，不是 ref-aware definition tombstone/GC。
- **认知状态**：verified_inference（正式 tarball、tag parser、公开 `.d.ts` 与 filesystem/tool-skill 一方测试交叉核对）。

## 2026-08-30 · rc.2 路径与symlink边界

- 发现深度固定一层：`<root>/<name>/SKILL.md`或`<root>/<name>.md`；nested tree与manifest不扫描。project root取nearest `.git` ancestor，否则cwd；rank为project-dsh 100、project-agents 200、custom 300、user-dsh 400、user-agents 500。
- `get()`每次重读当前文件；definition的`resourceBase`是directory bundle自身目录或flat file所在root。正文里的resources只获得路径提示，不做存在性、containment、digest或权限验证。
- 有`ctx.fs`时，list/resolve/stat/readText与symlink可见性归该FileSystem provider；无`ctx.fs`时Node fallback对root直属symlink用`stat()`跟随，一方测试覆盖指向root外的directory/flat file。断链和special file忽略。
- `watchFollowSymlinks`只控制Chokidar watcher（默认true），不是读取sandbox。watch只因直属目录、flat Markdown、直接`SKILL.md`变化失效；references/scripts/assets内容变化不刷新catalog。
- **认知状态**：verified_inference（rc.2 README、provider实现与symlink/watcher一方测试）。

## 2026-08-30 · rc.2 catalog observation、冲突可见性与invocation policy

- 公共`list()`只返回排序后的winning summaries，会丢失完整性；`snapshot()`返回`{ skills, complete }`；`get()`只返回winning definition或`undefined`。三者都不返回revision/generation token。Registry内部虽有private monotonic `revision`，collect发现并发变化时最多重试一次，第二次仍变化便返回`complete:false`，但该revision不能被外部持有或比较。
- provider `list()`抛错时该provider整项被跳过、warning写日志、整次snapshot标`complete:false`；provider显式返回`{ candidates, complete:false }`时可用candidates仍参与winner合并，但snapshot同样不完整。公共结果没有provider failure数组或per-provider diagnostic，且调用`list()`会把这一状态隐藏掉。
- `invalidate()`只递增private revision、清cache并发无参`skills/change`；事件没有revision、snapshot、ack或barrier。它只能提示消费者重读，不能证明一次检查到随后`Session.append()`之间catalog未变化。
- 同层先按rank/order选winner，跨层nearest覆盖farther；公共`list/snapshot/get`只暴露最终winner，shadowed candidate、collision和provenance chain不可见。只有overlay尚未注册，或overlay位于独立child scope且调用方本来就持有parent/base `ScopeKey`时，才可用同一Registry观察base view；没有“排除provider/layer”参数，也不能从child公开反推parent。两次读取之间仍无共同token。
- invocation policy是Consumer责任：官方模型`skill`工具先检查summary的`modelInvocable`，再`get()`并复查definition；用户`/<name>` gesture则先`get()`，再检查`userInvocable`。因此双false不会经这两条官方路径注入模型，但gesture可能已触发provider/body读取；`ctx.skills.get()`本身policy-neutral，其他公开Consumer仍可读取双false定义。一方测试直接验证`trusted-only`可被`get()`加载。
- 固定rc.2没有把Skill catalog observation与Session durable event append绑定到同一原子或线性化屏障的公共契约。
- **认知状态**：verified_inference（固定tag源码、正式npm根声明与一方tests交叉核对；未运行上游测试）。
