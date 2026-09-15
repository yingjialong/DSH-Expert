---
title: DSH 错误本
description: 记录 DSH 问答中的错误结论、失效边界与纠正证据。
type: reference
status: active
updated: 2026-09-16
---

# 错误本（Error Book）

> 记录**负面知识**：答错并被纠正的结论、走过的死路、上游改动导致失效的旧答案、"看起来对其实是坑"的东西、以及 T1 源码与 T2 文档不符之处。
>
> **每次回答前必须扫一遍本文件**（`CLAUDE.md` 第 3 条）。在"越问越聪明"这件事上，错误本比正面知识更值钱——它防的是重复犯同一个错。

## 条目格式

```
### E<编号> — <一句话标题>
- **类型**：答错 / 死路 / 上游变更导致失效 / 文档与源码不符 / 命名陷阱
- **错误内容**：<当时错在哪>
- **正解**：<正确的是什么>
- **根因**：<为什么会错>
- **发现于**：YYYY-MM-DD · 上游 <短SHA>
- **牵连条目**：<被这个错误污染过的 wiki 页，已修正>
```

---

### E001 — PyPI 上的 `deepseek-harness` 不是 DSH 官方包

- **类型**：命名陷阱
- **错误内容**：按项目名去 PyPI 搜 `deepseek-harness`，会搜到一个 0.2.0 版本、描述为 "Protocol-aware client for DeepSeek V4-Pro / V4-Flash" 的包。**它与 DeepSeek Harness 无关。**
- **正解**：官方 Python SDK 是 **`deepseek-harness-sdk`**（import 名 `deepseek_harness`），配套运行时 **`deepseek-harness-runtime-bin`**（import 名 `deepseek_harness_runtime`），两者版本严格锁死（`deepseek-harness-runtime-bin==<同版本>`）。要求 Python ≥ 3.10，依赖 `pydantic>=2.12,<3`。
- **根因**：包名与项目名相近，且第三方先占了更直觉的名字。
- **发现于**：2026-08-20 · 上游 `141eb6f`
- **牵连条目**：无（建库时即发现）

---

### E002 — `packages/README.md` 的包组表格漏列了两个组

- **类型**：文档与源码不符（T1 vs T2）
- **错误内容**：`packages/README.md` 的包组表格只列出 **48** 个组，未列出 `mcp` 与 `runtime-diagnostics`，据此回答"DSH 有哪些包组"会漏。
- **正解**：以 `ls upstream/deepseek-harness/packages/` 为准。实际为 **50 个包组、226 个包**。
- **附带教训**：首次核验时我用的正则 `^\| \`([a-z0-9-]+)/\`` 匹配到 0 行（表格实际写法是 `| [\`core/\`](core/README.md) |`），若不复核就会得出"README 一个组都没列"的荒谬结论。**校验脚本本身也要被校验**——匹配数为 0 时必须先怀疑正则，而不是怀疑数据。
- **根因**：文档表格手工维护，落后于目录实际内容。**这正是 `CLAUDE.md` 第 4 条"T1 与 T2 冲突以 T1 为准"的活例子。**
- **发现于**：2026-08-20 · 上游 `141eb6f`
- **牵连条目**：无（建库时即发现）

---

### E003 — `npx @deepseek-ai/dsh` 装到的不是最新版

- **类型**：命名陷阱 / 版本陷阱
- **错误内容**：默认以为 `npx @deepseek-ai/dsh` 会拿到最新的 rc.8。
- **正解**：npm dist-tag 中 `latest` = **0.1.0-rc.7**，`next` = 0.1.0-rc.8。不显式指定 tag 时拿到的是 rc.7。PyPI 的 `deepseek-harness-sdk` 同样只到 `0.1.0rc7`。**这也是本知识库默认回答基线选 rc.7 的原因。**
- **根因**：上游用 `next` 发布预览版，`latest` 滞后一个版本。
- **发现于**：2026-08-20 · 上游 `141eb6f`
- **牵连条目**：无（建库时即发现）
- **复验提醒**：dist-tag 会变，`/dsh-sync` 步骤 9 每次都要重查。

---

### E004 — 不要相信任何组 README 的"包清单表格"

- **类型**：文档与源码不符（跨组系统性问题）
- **错误内容**：`packages/<group>/README.md` 的包/ctx-key 表格看起来是权威清单（`packages/README.md` 明文写 "Group READMEs own package/ctx-key maps"），但**多个组的表格都漏包**。
- **正解**：**任何"这个组有哪些包"的问题，一律以 `ls packages/<group>/` 与各包 `package.json` 为准。**
  已确认漏列的组（冷启动全量核对，详见 `conflicts.md`）：

  | 组 | 漏列 |
  | --- | --- |
  | `client` | `ui-deliverables`、`ui-directory-picker-browse`、`ui-directory-picker-native`、`ui-message-feedback`（表列 36，实有 40） |
  | `shell` | `pwsh-sandbox`、`tool-bash-persistent`、`tool-pwsh-persistent` |
  | `core` | `agent-tool-presentation` |
  | `fs` | `tool-str-replace-editor` |
  | `sandbox` | `sandbox-windows-acl` |
  | `client` | `ui-permission/` 行的标签与真实目录 `ui-permission-presets/` 不符 |

- **根因**：表格手工维护，包增删速度快于文档更新（两天一个 rc）。
- **发现于**：2026-08-20 · 上游 `141eb6f`
- **牵连条目**：无（冷启动时即发现并在各包组页标注）

---

### E005 — npm 包名规则「组名不出现在包名里」**并非全仓通用**

- **类型**：命名陷阱（跨组）
- **错误内容**：`packages/README.md` 写 "Groups hold `packages/<group>/<pkg>/`; names stay `@deepseek-ai/dsh-<pkg>`"，据此推导包名会**推错**。
- **正解**：**包名一律以该包的 `package.json` 的 `name` 字段为准。** 已确认的例外：

  | 目录 | 实际 npm 名 |
  | --- | --- |
  | `packages/sdk/client` | `@deepseek-ai/dsh-sdk-client` |
  | `packages/sdk/protocol` | `@deepseek-ai/dsh-sdk-protocol` |
  | `packages/sdk/server` | `@deepseek-ai/dsh-sdk-jsonrpc-server`（**不是** `dsh-server`） |
  | `packages/host/*` | `@deepseek-ai/dsh-host-*` |
  | `packages/client/ui-*` | `@deepseek-ai/dsh-client-ui-*` |
  | `packages/experimental/*` | `@deepseek-ai/dsh-experimental-*`（该组 AGENTS.md 强制） |
  | `packages/guard/timeout-policy` | `@deepseek-ai/dsh-tool-call-timeout-policy` |
  | `packages/extensions/ui-cordis` | `@deepseek-ai/dsh-client-ui-cordis` |

- **根因**：顶层 README 描述的是主流约定，但多个组有自己的命名策略，顶层未标注例外。
- **发现于**：2026-08-20 · 上游 `141eb6f`
- **牵连条目**：无

---

### E006 — 仓库根 `CLAUDE.md` / `AGENTS.md` 的 Repository layout 已严重过时

- **类型**：文档与源码不符
- **错误内容**：仓库根的 `CLAUDE.md`（`AGENTS.md` 是其符号链接）里的 Repository layout 代码块列出了**并不存在**的 `packages/self-modification/` 与 `packages/support/`，并**漏列十余个真实存在的组**（`jobs/`、`mcp/`、`runtime-diagnostics/`、`goal/`、`schedule/`、`sandbox/`、`feedback/`、`attachment/`、`spill/`、`session-query/`、`storage/`、`workspace/`、`host/`、`client/`、`preset/` 等）。
- **正解**：`self-modification/` → 实为 **`packages/extensions/`**；`support/` → 实为 **`packages/test-support/`**。**目录结构问题一律以 `ls packages/` 为准，`packages/README.md` 次之，仓库根 CLAUDE.md/AGENTS.md 最不可信。**
- **根因**：根级 agent 指引文件更新频率低于目录演进速度。**讽刺之处在于：这正是 DSH 自己的 agent 指引文件，而它已经会误导 agent。**
- **发现于**：2026-08-20 · 上游 `141eb6f`
- **牵连条目**：无

---

### E008 — 0.1.1-rc.1 起 credentials 事件改名并拆分、seam 扩成两个 key space

- **类型**：上游变更导致失效（破坏性）
- **错误内容**：rc.8 及以前的答案是：凭据 seam 只有一个 key space（`CredentialRef`）+ 单一事件 `credentials/updated(ref)` + Service Definition 仅四方法（`resolve/describe/set/unset`）+ `.credentials.yaml` 是平面 `ref: value` 映射无版本字段 + 并发写 last-write-wins + 组内只有 credentials/credentials-local 两个包。
- **正解**：`credentials/updated` **已不存在**，拆为 `credentials/reference-updated(ref: CredentialRef)` 与 `credentials/record-updated(key: CredentialKey)`（两个 key 语法不相交，合并会让监听器分不清 subject 空间）；ref 半边四方法不变，新增 record 半边五方法 `readRecord/describeRecord/listRecords/modifyRecord/deleteRecord`（`modifyRecord` 是唯一写路径，跨进程 writer 锁下 read-decide-replace，锁等待 30 s）；`.credentials.yaml` 变为 `version: 1` + `refs:` + `records:` 顶层结构，pre-release 平面布局 boot 时在锁下自动迁移；新增第三包 `authorization/`（`ctx.authorization`：`registerFlow/list/describe/cancel/begin` + `authorization/settled` 事件）。
- **根因**：0.1.1 引入 durable credential records 与「问人要凭据」的 authorization flow（commit `86a9f8c86`、`732a7361f`、`fecfabcac`）。
- **发现于**：2026-08-22 · 上游 `b150a55`（0.1.1-rc.2）
- **牵连条目**：[packages/credentials.md](packages/credentials.md) 已重写；[topics/architecture-overview.md](topics/architecture-overview.md) 事件域清单已核（capability 事件新增两条，21 步旅程不受影响）

---

### E009 — 0.1.1-rc.1 起 projection register 接口重构：`schema`→`stateSchema`、`view` 移入可选 `wire` 块

- **类型**：上游变更导致失效（破坏性，波及所有注册 projection 的插件）
- **错误内容**：rc.8 及以前的答案是：`ProjectionDefinition<K,S> = { key, schema, init(), apply(state,event), view(state), stateVersion }`，key 并进 `SessionProjectionMap`，snapshot/change feed/checkpoint 覆盖每个已注册单元。
- **正解**：接口改为 `{ key, stateSchema, init(), apply(state,event), wire?, stateVersion }`——`schema` 改名 `stateSchema`（语义：校验持久化 state，seed fold 前）；`view` 与其校验移入可选块 `wire: { viewSchema, view(state) }`；key 先并进新类型表 `SessionProjectionStateMap`（host fold state），带 wire 的才再进 `SessionProjectionMap`（client-visible）；**省略 wire 即 host-only 单元**——snapshot 与 change feed 只覆盖 client-visible 单元，checkpoint 仍覆盖全部；新增 `stateOf(session, key)` 单元状态读取 API；register 变双重载。goal/todo/plan/session-title/apiproxy 等所有既有注册点已全部随迁。
- **根因**：session-projection 三连重构分离 host state 与 client views（commit `4c421ec88`、`9127d7e8b`、`327b86d2e`）。
- **发现于**：2026-08-22 · 上游 `b150a55`（0.1.1-rc.2）
- **牵连条目**：[packages/session.md](packages/session.md)、[topics/core-chain.md](topics/core-chain.md)、[packages/plan.md](packages/plan.md)、[packages/todo.md](packages/todo.md) 已修正

---

### E010 — 0.1.1-rc.1/rc.2 起图片附件改为两级限额 + normalized 持久化 + 请求期投影

- **类型**：上游变更导致失效（破坏性）
- **错误内容**：rc.8 及以前的答案是：attachment-local 源准入单边默认 2000 px（单张 3.5 MiB / 40 M 像素 / 消息聚合 100 MiB）；`saveImage` 持久化原始上传字节的内容寻址对象；AttachmentStore 只有准入-持久化方法无请求级 API。
- **正解**：两级结构——源准入（单边 8192 px / 单张 20 MiB / 64 M px / 每消息 20 张 200 MiB）+ 独立 normalization（长边 2048 px / 4 MiB，配置键 `normalizedImageMaxDimension` / `normalizedImageMaxBytes` / `imageCompressionConcurrency`）；`saveImage` 持久化的是 provider-independent **normalized image**（应用 EXIF orientation、归一化缩放），`ImageAttachmentRef` 新增 `originalDimensions`（仅当缩小才记录）；新增 `readImageRequest(ref, policy, signal?)` 请求级投影（按路由 `ImageRequestPolicy` 的像素/字节预算派生，基类默认抛 `ATTACHMENT_PROJECTION_UNSUPPORTED`）；**image region reads 已整体移除**（retired image-region tool）。
- **根因**：unified master and Files request pipeline（commit `2491e12fd`、`724783b02`、`72b204afa`、`d29855f97`）。
- **发现于**：2026-08-22 · 上游 `b150a55`（0.1.1-rc.2）
- **牵连条目**：[packages/attachment.md](packages/attachment.md) 已重写；[packages/llm.md](packages/llm.md) 已补 `prepareCall` 与 text-only 模型图片投影

---

### E011 — 0.1.1-rc.2 起 Web 切换模型不再拦截「会话已有图片但模型不支持」

- **类型**：上游变更导致失效（行为变化）
- **错误内容**：rc.8 及以前：Web GUI 切换模型时若会话已含图片且目标模型 `inputModalities` 不含 `image`，api-proxy 会拒绝并报 `model-unavailable`。
- **正解**：该守卫已从 `api-proxy.ts` 删除（commit `d29855f97`）。能力检查移到两处：入站 prompt 准入（ACP 侧 `assertImageRoute`，模型不声明 image input 时抛 `AcpContentError(..., 'invalid')`；`initialize` 时 `supportsAcpImagePrompts` 决定 `promptCapabilities.image` 通告）+ `LlmRuntime.adapterStream` 在 dispatch 前对 text-only 模型用 `projectImagesForTextModel()` 做确定性占位投影（经 `ctx.llm.stream()` 给 text-only 模型发图不再触发 deepseek adapter 的 `UNSUPPORTED_CONTENT` 门）。
- **根因**：图片管线统一后，拦截点从「模型切换」前移到「内容准入」与「adapter dispatch」。
- **发现于**：2026-08-22 · 上游 `b150a55`（0.1.1-rc.2）
- **牵连条目**：[packages/host.md](packages/host.md)、[integration/protocol-acp-http.md](integration/protocol-acp-http.md)、[packages/llm.md](packages/llm.md) 已补

---

### E012 — 断言「symbol 不能做 WeakMap key」是过时规则：非注册 symbol 可以

- **类型**：答错（我的核验结论被下游运行时反证推翻）
- **错误内容**：一次外部 DSH 集成方案审核中，我裁决「rc.2 `ToolExecution.token` 是 branded symbol，ES WeakMap 的 key 必须是 object——symbol 不能直接做 WeakMap key，据此断言上游/集成方用 token 关联 effect handle 会抛 TypeError」。
- **正解**：**非注册 symbol（`Symbol('...')`）可以作为 WeakMap/WeakRef key**——ES2022 "Symbols as WeakMap keys" 提案落地（V8 11.x / Node ≥20）。只有 `Symbol.for()` 创建的 **registered symbol** 才被拒绝。实测（Node v22.22.3）：`wm.set(Symbol('x'), 1)` 成功；`wm.set(Symbol.for('x'), 1)` 抛 TypeError。DSH 的 `createExecutionToken = Symbol('dsh.tool.execution')` 恰是非注册 symbol，**token 做 WeakMap key 是合法的**。
- **根因**：把 ES2021 及之前的 WeakMap key 规则当成了现状（语言特性演进滞后于训练知识），且没有用一行 `node -e` 实测就下了「必然抛 TypeError」的强断言——正是本库第 1 条「'显然'是最危险的词」的再犯。
- **发现于**：2026-08-23 · 外部方以真实运行时行为反证驳回，本库复测确认（Node v22.22.3 本地复测）
- **牵连条目**：当次审核报告中的对应结论作废；[packages/interaction.md](packages/interaction.md) 的 token 相关条目未涉及 WeakMap 用法、无需修正
- **教训**：涉及「必然抛错/必然不可能」级别的语言/运行时断言，先跑一行最小实测再落笔

---

### E013 — pi-ai Responses 发布 `off` 后，省略 effort 不再保留 provider default

- **类型**：文档与源码不符 / 过度泛化
- **错误内容**：把 `llm-pi-ai` README 的「省略 profile `reasoning` 会保留 provider default」当成所有 exact model profile 的通用结论，或把 `off: null` 的「send nothing」理解成最终 Responses payload 省略 `reasoning`。
- **正解**：固定 `dsh-llm-pi-ai@0.1.1-rc.2` + `pi-ai@0.82.1` 下，DSH adapter 把显式 `off` 与 request/route 双省略都变成 common options 中没有 `reasoning`。若 exact Responses model 已发布 `off`，pi formatter 随后读取 `thinkingLevelMap.off`：字符串值原样发送，值缺席则发送 `none`；因此 `off: none`、`off: null` 都不会保留 omitted-field default。只有完全不声明 `off`、使 map 值为 `null`，才省略最终 `reasoning`。普通 custom route 适用；pi 对 provider id `github-copilot` 有特判。
- **根因**：README 同时描述 DSH→pi common option 与协议 formatter 的最终 wire，`off` 与「没有偏好」在 adapter 边界已不可区分，而 Responses formatter又有自己的 off fallback。
- **发现于**：2026-08-30 · DSH `b150a551` / pi-ai `0.82.1` (`b4f29368`)
- **牵连条目**：[packages/llm.md](packages/llm.md) 的 reasoning effort ownership 已加协议限定；该页仍因 alpha.1 同步保持 stale，固定 rc.2 问题须回 tag 复验。

---

### E014 — Session 持久化 preset ID，不等于持久化 preset 定义

- **类型**：概念陷阱 / 过度泛化
- **错误内容**：看到 `SessionHeader.agentPreset` 与 `agent-preset/selected` 可耐久恢复，就断言 DSH 能仅凭 Session log 重建任意动态 preset，或断言定义缺失时所有公开路径都统一 fail closed。
- **正解**：rc.2 Session 只存 id；`AgentPresets` 仍从当前文件系统 roots 解析 `<id>/agent.cordis.yml`，没有任意 definition 注册、versioned backend 或 Session-owned definition snapshot。roster 存在但 id 缺失时真实 Agent resume 不回退 default；冷 transcript / skill-list 读侧会退到 global scope；整个 roster 服务未装配时 ApiProxy 采用 Host composition。已运行 standing mount 则可跨文件删除继续到进程结束。
- **根因**：混淆了 selection identity、definition storage 与不同 API 的恢复/展示路径。
- **发现于**：2026-08-30 · 上游 `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/preset.md](packages/preset.md)、[packages/skill.md](packages/skill.md) 已补专项边界。

---

### E015 — 把 `AUTH` / `TRANSPORT` 当成出网阶段证明，或把缺失命名凭据归成 `AUTH`

- **类型**：错误码语义过推 / 阶段混淆
- **错误内容**：断言命名 `apiKeyEnv` 解析不到值会在 HTTP 前产生 `AUTH`；或看到 `AUTH` / `TRANSPORT` 就断言请求一定已到达 endpoint / 一定已开始 HTTP。
- **正解**：rc.2 `llm-pi-ai` 在进入 pi-ai SDK 前把缺失/空命名凭据归为 `MISSING_CREDENTIAL`，把非空但不可安全使用的凭据归为 `INVALID_CREDENTIAL`。`AUTH` 与 `TRANSPORT` 是对 pi-ai 扁平 terminal error message 的正则分类：前者匹配独立 `401/403`，后者匹配 truncation 或网络/连接词。formatter/buildParams 错误也走这条文本分类，所以 code 不携带 I/O provenance；无效 reasoning 则是独立的 `UNSUPPORTED_REASONING_EFFORT`。
- **根因**：混淆了 DSH 内建 preflight、pi-ai 的统一 error event，以及 DSH 对 error text 的后置归类。
- **发现于**：2026-08-30 · DSH `b150a551` / pi-ai `0.82.1` (`b4f29368`)
- **牵连条目**：[packages/llm.md](packages/llm.md)、[packages/credentials.md](packages/credentials.md) 已补固定 rc.2 边界。

---

### E016 — `package.json` 声明 `./src/*`，不等于正式 npm tarball 真含源码入口

- **类型**：发布闭包陷阱 / public seam 误判
- **错误内容**：看到 rc.2 多个包的 exports 写了 `"./src/*": "./src/*"`，就断言 out-of-tree 宿主可正式 import `@deepseek-ai/.../src/...`。
- **正解**：Skill/Preset/MCP 等已核 rc.2 / alpha.2 tarball 的正式产物没有 `package/src/`；即使 `package.json` 声明了 `./src/*`，export target 仍实际不存在。公共面必须取 package exports 与tarball成员的交集，再核根/子路径 `.d.ts` 和 JS exports。以 `dsh-mcp-client` 为例，内部虽 bundle 了 transport/connection/tools helpers，根最终只导出 `Config/apply/inject/name`，没有 carrier factory 或 generation API。
- **根因**：把 monorepo 源码开发映射当成 npm 发布闭包；上游一方测试直接 import `src/*` 也只证明仓内测试面。
- **发现于**：2026-08-30 · DSH `b150a551`（`0.1.1-rc.2`）/ `0a53fb55`（`0.1.2-alpha.2`）正式 npm tarball复核
- **牵连条目**：[packages/mcp.md](packages/mcp.md)、[packages/skill.md](packages/skill.md) 已修正。

---

### E017 — GitHub tag、npm dist-tag与package-family发布闭包必须分别核验

- **类型**：发布渠道混淆
- **错误内容**：看到GitHub prerelease便断言npm完全未发布，或看到npm根包已发布便把tag内全部package manifests都称为正式tarball闭包；又或只读`latest`/`next`而漏掉`alpha` dist-tag。
- **正解**：alpha.1只有GitHub source archive、没有npm alpha.1；alpha.2的GitHub release先于npm发布约19分钟，当天npm package family又分批补齐。最终复验时根包`@deepseek-ai/dsh@0.1.2-alpha.2`位于`alpha`，`latest`/`next`仍为rc.2；tag内245个非private package标识（含根CLI与Web frontend）均已有alpha.2 tarball。Tag manifest、单包tarball、dist-tag与真实可安装递归/native闭包仍必须分开核验。
- **根因**：混淆GitHub release、npm package publication与PyPI runtime publication三个独立渠道。
- **发现于**：2026-08-30 · `cd5ef814`/`0a53fb55` · GitHub release + npm package-family/PyPI复验
- **牵连条目**：[alpha.1版本变更](topics/版本变更-0.1.1-rc.2-到-0.1.2-alpha.1.md)、[alpha.2版本变更](topics/版本变更-0.1.2-alpha.1-到-0.1.2-alpha.2.md)、[alpha.1 Host嵌入](integration/alpha1-full-host-embedding.md)。

---

### E018 — `SessionEventMap` declaration merge 不等于下游event进入持久化白名单

- **类型**：类型扩展与runtime vocabulary混淆
- **错误内容**：看到`SessionEventMap` merge-extensible和`Session.append()`公开，就断言out-of-tree plugin可定义required event并在任何正式PersistenceBackend上exact resume。
- **正解**：known event set由DSH build扫描仓内declaration生成，源码明确把out-of-repo registration列为deferred。rc.2仅允许unknown envelope显式`ignorable:true`；alpha.1连该例外也移除。projection public seam只解决fold/view，不把event加入persistence vocabulary，也不提供Host create/resume/fork pre-publication resolver。
- **根因**：把编译期TypeScript key扩展、live append和持久read compatibility当成同一契约。
- **发现于**：2026-08-30 · rc.2 `b150a551` / alpha.1 `cd5ef814`
- **牵连条目**：[packages/session.md](packages/session.md)、[packages/preset.md](packages/preset.md)。

---

### E019 — 精确锁CLI版本不等于精确锁完整npm闭包

- **类型**：发布闭包 / 可复现性陷阱
- **错误内容**：看到`@deepseek-ai/dsh@0.1.1-rc.2`就断言其全部DSH companion、Cordis/Loader与native helper也被tarball精确钉在同一发布快照。
- **正解**：CLI tarball自身可由integrity精确识别，但其DSH dependencies为`^0.1.1-rc.2`，Cordis/Loader/Include/HMR等也是ranges，Koffi/Sharp也用caret；tag lock才给出当次解析结果。根版本、package tarball与递归install closure是三件事，完整可复现性必须同时保留lockfile/integrity并审计native companions。
- **根因**：把monorepo共享release version误当成npm依赖的exact-equality约束。
- **发现于**：2026-08-30 · 正式`@deepseek-ai/dsh@0.1.1-rc.2` tarball + tag `b150a551` lockfile复核
- **牵连条目**：[rc.2完整Web Profile与原生壳](integration/rc2-full-web-native-shell.md)。

---

### E020 — `recompose()`成功后再append，不等于composition与selection原子提交

- **类型**：跨owner提交边界过推
- **错误内容**：看到blank preset切换在`recompose()`返回后追加`agent-preset/selected`，就断言ensure/rebind/append任一步失败都会同时保留旧live composition与旧durable selection，或断言select与首prompt共享同一序列化闸门。
- **正解**：`recompose()`只保证new standing在scope parent移动前就绪，unknown/mount/cycle失败不改旧link；ApiProxy在它返回后才单独`Session.append()`，append前置检查失败没有反向rebind。旧exact generation虽仍存活，但binding与historical key不公开，`recompose(id)`只认current standing，也无per-session generation release。`presetSwitches`只串行select；prompt直接入Agent inbox，`turn/start`由driver稍后追加。因此既无跨scope/log事务，也无公开all-or-old补偿或select/prompt统一linearization。
- **根因**：把“durable event位于成功控制流之后”误读为两个owner共享事务，又把select内部blank recheck误读为first-prompt reservation。
- **发现于**：2026-08-30 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/preset.md](packages/preset.md)。

---

### E021 — scope-local Skill provider只会merge/shadow，不会过滤继承目录

- **类型**：scope layering过推
- **错误内容**：看到`SkillRegistry.registerProvider()`可注册进Agent/Preset scope，便断言该scope能只看provider给出的selected exact Skill集合，或把opaque locator当DSH验证并持久的immutable ref。
- **正解**：Skill目录按global→farthest ancestor→nearest scope合并，nearest只覆盖同名entry；没有allow/deny、provider exclusion或selected-set filter，不同名的继承Skill仍可见。locator由provider拥有，Registry只往返并校验definition name/shape，不验证digest/immutability，也不持久locator或body。
- **根因**：把“局部贡献+同名shadow”误读为“对完整继承目录做减法”，又把opaque handle误读为DSH-owned identity。
- **发现于**：2026-08-30 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/skill.md](packages/skill.md)。

---

### E022 — `snapshot.complete`不是可跨操作持有的catalog generation

- **类型**：一致性语义过推
- **错误内容**：看到`SkillRegistry.snapshot().complete === true`或收到`skills/change`，便断言catalog从检查到后续Session event append期间保持不变，或把provider失败当成可精确诊断的结构化结果。
- **正解**：`complete`只说明该次collect在内部revision观察下完成且可缓存；公共结果没有revision/generation token。provider throw只被warn并跳过，snapshot降为incomplete但没有failure明细；`list()`甚至丢掉complete。`skills/change`是无参invalidate通知，不提供ack/barrier。因此这些公共面不能与后续`Session.append()`形成原子或线性化证明。
- **根因**：把单次观测完整性、变更通知和跨owner事务当成同一概念。
- **发现于**：2026-08-30 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/skill.md](packages/skill.md)。

---

### E023 — Skill invocation双false不等于definition不可读取

- **类型**：policy边界过推
- **错误内容**：把`modelInvocable:false, userInvocable:false`解释为Registry拒绝加载正文，或断言用户gesture在读body前就会拒绝。
- **正解**：`ctx.skills.get()`是policy-neutral加载原语，一方测试直接读取`trusted-only`。官方模型tool在`get()`前检查summary并在读取后复查；用户gesture相反，会先`get()`再检查`userInvocable`。双false不会经这两条官方Consumer注入模型，但不代表provider/body完全不被读取，也不约束其他Consumer。
- **根因**：把Consumer调用策略误当成SkillRegistry访问控制。
- **发现于**：2026-08-30 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/skill.md](packages/skill.md)。

---

### E024 — prompt `rpcId`是相关标记，不是幂等键

- **类型**：admission identity过推
- **错误内容**：看到ApiProxy把prompt `rpcId`写进durable UserMessage，就断言丢响应后可用同id安全重试或线性化查询acceptance。
- **正解**：AbstractApiClient每次mint新id，SessionFace不接caller id；即使低层重用同rpcId，Host也不去重，会追加第二条prompt。History可事后扫描source.rpcId证明已存在，但无status endpoint或与在途call共享的barrier。真正的入口幂等面只有caller预分配`sessionId`的create。
- **根因**：混淆wire correlation、durable provenance与idempotency key。
- **发现于**：2026-08-30 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/host.md](packages/host.md)。

---

### E025 — cancel accepted不是effect quiescence，`TOOL_OUTCOME_UNKNOWN`也不是exactly-once

- **类型**：lifecycle/outcome过推
- **错误内容**：把`session.cancel → {accepted:true}`解释为返回后不会再启动/完成physical effect，或把crash repair的unknown result当作自动去重保证。
- **正解**：cancel只同步abort后立即应答，不await Agent idle；started body必须合作drain。Crash时已有`tool/call`无result会补`TOOL_OUTCOME_UNKNOWN`并要求核验外部状态，它不保存provider receipt、不查询effect，也不证明没执行。physical owner仍须自持idempotency/receipt/reconciliation。
- **根因**：把取消请求接收、协作式drain、durable transcript repair与外部提交协议混成同一ack。
- **发现于**：2026-08-30 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/core.md](packages/core.md)、[packages/mcp.md](packages/mcp.md)。

---

### E026 — `/api` trust fence不是auth，也不覆盖初始HTML

- **类型**：安全覆盖面过推
- **错误内容**：看到`dsh-client-connection`统一校验API和两条WebSocket，就宣称整个Web surface已由同一Host/Origin admission或authentication保护。
- **正解**：同一predicate只包`/api` prefix与mux/host upgrades；`frontend-static` fallback直接服务initial HTML。predicate源码也明说不是authentication，且未从正式root导出。WebServer有raw route/fallback/upgrade primitives，但无全server middleware。
- **根因**：把一个route owner的prefix fence泛化为server-wide security policy。
- **发现于**：2026-08-30 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/host.md](packages/host.md)。

---

### E027 — 禁用credential UI不等于禁用raw-secret wire，MCP也不消费CredentialProvider

- **类型**：secret owner/seam混淆
- **错误内容**：替换`CredentialProvider`或移除Models UI后，便断言浏览器不能再发送raw credential；或把通用credentials seam当成MCP config会自动消费的能力。
- **正解**：custom CredentialProvider是正式Host seam，Models UI也是可独立disable的client row，但`credentials.set` RPC仍在且收raw value；provider拒绝发生在secret过线之后。rc.2 MCP只接stdio env/HTTP headers，没有CredentialProvider resolver或carrier factory。三条边界必须分别判断。
- **根因**：把storage owner、presentation与transport admission当作同一层。
- **发现于**：2026-08-30 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/credentials.md](packages/credentials.md)、[packages/mcp.md](packages/mcp.md)。

---

### E028 — `maxRetries:0`只关闭一个executor，不是全局never-retry

- **类型**：组合边界过推 / 同版本文档冲突
- **错误内容**：看到provider route支持`retryPolicy.maxRetries:0`，便断言同一logical step绝不可能再次进入adapter；或照`llm-retry/README.md`把每次retry说成新turn。
- **正解**：`0`只让可选`dsh-llm-retry` listener在首个失败上delegate；任何其他`agent/request-error` listener仍可返回retry。T1 loop会在同一`turn/step`的`while`中再次调用adapter，一方test只有一条`step/start`；README的fresh-turn表述错误。真正的at-most-one需要同时约束完整recovery composition与adapter/middleware自身。
- **根因**：把provider-owned policy、一个policy executor、公共waterfall的最终组合结果混成同一封闭开关，又未用控制流/测试复核README。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/llm.md](packages/llm.md)、[conflicts.md](conflicts.md) C080。

---

### E029 — `agent/pre-step`不是schema assembly之前的异步闸

- **类型**：生命周期ordering过推
- **错误内容**：看到`agent/pre-step`可async reject，便断言它能在每次tool schema collection之前校验远端catalog；或把同步`SystemPrompt.tools()`provider当成可await的hook。
- **正解**：AgentLoop先`systemPrompt.assemble()`再dispatch`agent/pre-step`；assemble又先同步调用tool providers并clone/order schemas，最后才await`system-prompt/assemble`waterfall。两条event都可在model adapter前fail closed，但schema已生成。tool侧`guard()`也只能同步；物理effect前最后一次async check应由`ToolDefinition.execute`第一步拥有。
- **根因**：把“model request之前”与“schema assembly之前”混为一谈，又忽略了provider签名的同步返回类型。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/core.md](packages/core.md)、[packages/mcp.md](packages/mcp.md)。

---

### E030 — content-addressed path不等于official Skill provider持有同一bytes

- **类型**：TOCTOU / identity过推
- **错误内容**：外部先hash一个immutable-looking `SKILL.md`路径，便断言后续`FileSystemSkillProvider.get()`必然读取刚校验的bytes，或把candidate locator当digest/revision/fd。
- **正解**：official locator只有path/directory；list与get分别重新open/parse文件，Registry不比较body/digest，也没有authenticated-byte hook。只有外部store真正禁止pathname、目录项、symlink与底层fs替换时才能条件性推出同bytes；DSH本身不封闭check→open窗口。
- **根因**：把内容寻址命名约定当成provider拥有的byte lease，又忽略了list/get的两次独立读取。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/skill.md](packages/skill.md)。

---

### E031 — DSH注册成功不等于provider接受该工具名

- **类型**：owner边界过推 / provider兼容性误判
- **错误内容**：看到`ctx.tools.register()`接受一个名称，或看到`dsh-llm-pi-ai`把`ToolSchema.name`原样映射成PiTool，就断言所有OpenAI/Anthropic兼容endpoint都接受该字符集和长度。
- **正解**：rc.2 ToolRuntime只拥有string类型、同层唯一性、保留名`run_code`与最终execution非空 invariant；没有通用provider regex或最大长度。`schemas()`和`llm-pi-ai.toolsOf()`原样传name，但最终formatter/auth compat与endpoint仍可转换或拒绝。pi-ai 0.82.1的Anthropic OAuth路径就会canonicalize一组Claude Code已知名字；未指定exact provider/auth时，wire限制仍unknown。
- **根因**：把DSH registry identity、adapter DTO映射和provider wire validation合并成一个owner。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）/ pi-ai `0.82.1`
- **牵连条目**：[packages/core.md](packages/core.md)、[packages/llm.md](packages/llm.md)。

---

### E032 — cold `session.history`会激活preset，`agent/created`不是前置准入门

- **类型**：生命周期/安全边界误判
- **错误内容**：把cold transcript读取理解成只inspect log、不加载composition；或只在`agent/created`校验preset identity，便断言任何preset plugin activation都已被该检查覆盖。
- **正解**：rc.2 history在切page前调用`presenterScopeFor→standingKeyFor→ensureStanding`；首读或stamp变化会真实Include/Loader activate recorded preset rows，但不创建Agent/Session/turn，因此`agent/created`根本不发。正常Agent创建也先完成setup/mount再announce。broken standing只让presenter退global并继续history，不能把这一fail-soft行为当成plugin从未执行。
- **根因**：把“cold read不resume Agent”错误等价成“不compose presenter scope”，又把publication event错当前置activation guard。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/preset.md](packages/preset.md)。

---

### E033 — Loader事件不能补成preset的pre-import inventory gate

- **类型**：公共hook能力过推 / 安全时序误判
- **错误内容**：发现Cordis公开`loader/entry-init`或`loader/patch-context`，便断言外部inventory可在cold standing mount加载任何未审核module之前按preset id/digest否决。
- **正解**：`entry-init`在Entry的`options={}`时已发，尚无row identity/config；初始`patch-context`位于module dynamic import之后、plugin apply之前，throw最多阻断后半段，不能撤销top-level执行，也不携带preset id/digest/review receipt。`internal/plugin`同样只是fiber观察面。rc.2无AgentPresets专属pre-mount callback/event/provider。
- **根因**：把通用Loader生命周期观察/上下文patch，误当成具备artifact identity且位于import前的安全admission合同。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/preset.md](packages/preset.md)。

---

### E034 — Approval延续同一execution不等于pin住同一ToolDefinition

- **类型**：工具identity/generation过推
- **错误内容**：看到`allowed-once`后沿同一`ToolExecution`继续，便断言批准时看到的definition/executor也被锁定，等待审批期间的registry replacement不会改变body。
- **正解**：rc.2在policy前快照并deep-freeze arguments，同一execution/callId/token确实延续；但`dispatchToolBody()`在批准与guards之后才按name + Agent scope重新`resolveExecution()`。静态no-swap时条件性命中同一body；允许unregister/shadow/replacement时没有definition generation pin。
- **根因**：把execution-local参数快照与registry-owned definition lifetime合并成同一身份保证。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/interaction.md](packages/interaction.md)、[packages/core.md](packages/core.md)。

---

### E035 — AgentHandle teardown的公开注释顺序与rc.2 runtime不一致

- **类型**：正式类型注释 / runtime冲突
- **错误内容**：照`dsh-agent`根`.d.ts`的概括注释，把`AgentHandle.dispose()`后三步写成“detach Agent→detach Session→最后dispose Agent scope”。
- **正解**：正式rc.2 runtime实际执行`cancel→whenIdle→scope.dispose→detachAgent→detachSession`；一方scope-lifecycle tests同样观察`scope-disposed→agent-disposed→session-disposed`。可依赖固定rc.2行为做事实核验，但不能把冲突注释当跨版本ordering保证。
- **根因**：只读类型JSDoc，未交叉核对正式runtime JS与lifecycle tests。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/core.md](packages/core.md)。

---

### E036 — history成功且无Agent不等于已证明global generic presenter

- **类型**：公开可观测性过推
- **错误内容**：missing/broken recorded preset的`session.history`返回成功且AgentRegistry无该id，便宣称response已证明`presenterScopeFor`尝试standing后回退global generic presenter。
- **正解**：成功+无Agent只证明fail-soft且未resume。公开`HistoryEntry`只有raw`event`与可选`view`，没有scope/fallback reason；view缺失同时可能来自global无definition/presenter、roster缺失、坏JSON、跨页pair miss或presenter throw。正向证明global lookup需在global layer注册可辨识presenter并断言`events[].view`。
- **根因**：把Host内部fallback控制流、Client对viewless event的generic渲染与公开API response字段混成同一证据。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/host.md](packages/host.md)、[packages/preset.md](packages/preset.md)。

---

### E037 — `schemas()`返回快照不等于register接管了caller schema

- **类型**：引用ownership过推
- **错误内容**：一方test证明修改`ctx.tools.schemas()`返回值不会污染registry，便反推`ctx.tools.register(definition)`已clone/deep-freeze输入，caller保留的nested schema不可再影响运行时。
- **正解**：register把原始definition引用插入layer；`schemas()`只在每次读取时snapshot parameters。caller事后修改原nested parameters会影响下一次projection，修改`output.schema`会影响后续成功结果校验。顶层freeze不递归保护nested对象。
- **根因**：混淆“read API返回detached copy”和“write API在admission时取得immutable ownership”。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/core.md](packages/core.md)。

---

### E038 — 后一个sibling effect失败不会回滚先前provide

- **类型**：Cordis事务边界过推
- **错误内容**：先`ctx.provide()`，再独立调用`ctx.effect()`；后者同步throw时，误以为Cordis会把同Fiber上之前注册的service一并撤销。
- **正解**：每次effect只回滚自己callback return/yield并已collect的disposers；先前provide是另一个sibling effect，仍存活到其disposer或整个Fiber unload。只有同一composite effect显式return/yield unprovide，或整个plugin startup失败，才形成共同rollback边界；手工Map/transport也必须在任何后续throw前立即yield可工作的cleanup，Cordis无法猜测未yield外部状态。
- **根因**：把“同一个Fiber最终拥有全部effects”误解成“任一新effect失败都会事务回滚所有既有siblings”。
- **发现于**：2026-08-31 · DSH `b150a551` / `@deepseek-ai/cordis@4.0.1`
- **牵连条目**：[topics/cordis-primer.md](topics/cordis-primer.md)。

---

### E039 — `snapshotJsonValue`不拒绝getter，只固定第一次读取

- **类型**：lossless snapshot语义误读
- **错误内容**：把“防getter漂移”表述成拒绝accessor/getter，或认为getter抛错会像普通非法JSON一样返回undefined。
- **正解**：rc.2对每个property/array slot只读一次，第一次读到合法JSON就materialize成功，即使getter第二次会返回exotic；第一次读到非法值才返回undefined。getter/Proxy trap若throw则异常原样传播。single-read消除的是validate/copy双读窗口，不是禁止getter。
- **根因**：把“一次读取完成验证与复制”误解成“admission拒绝动态property”。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/session.md](packages/session.md)。

---

### E040 — `deepFreeze`单独使用不等于冻结任意JS对象图

- **类型**：冻结覆盖面过推
- **错误内容**：看到nested object/array一方test，便宣称`deepFreeze`会遍历non-enumerable、symbol-key、AbortSignal等全部reachable对象。
- **正解**：rc.2只沿`Object.keys()`递归，刻意跳过AbortSignal；cycle用WeakSet终止。只有先经`snapshotJsonValue`得到ordinary enumerable-string-key JSON graph时，才能推出其全部nested containers都会冻结。
- **根因**：把“对lossless JSON snapshot完整”扩大成“对任意JavaScript graph完整”。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/llm.md](packages/llm.md)、[packages/core.md](packages/core.md)。

---

### E041 — `await SessionRuntime.refresh()`不等于binding已同步收敛

- **类型**：client lifecycle/可观测性过推
- **错误内容**：direct`IApiClient.sessions.create`后只`await runtime.refresh()`，便立即断言`binding(id)`存在；或把`noteAgentPreset`当成public create-result adoption seam。
- **正解**：refresh等待Manager的list RPC/fold，但Manager→Runtime list store由Notifier microtask投影，一方test显式再flush microtask。应等待public list observable确认id后再取binding；refresh错误又被fold进private Manager snapshot，不是成功receipt。`noteAgentPreset`只承诺已有blank Session成功switch后的label更新，不携带完整birth facts。
- **根因**：把concrete runtime wire-pump method的完成、manager snapshot完成、outward store发布与binding eligibility合成一个同步点。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/client.md](packages/client.md)、[packages/host.md](packages/host.md)。

---

### E042 — ready baseline缺席不等于create历史上未commit

- **类型**：commit-status/时间边界过推
- **错误内容**：create响应丢失后，看到一次`session.list`成功且exact id缺席，便权威分类not committed；或把sticky`SessionListState.phase='ready'`当本次refresh成功回执。
- **正解**：list只描述当前attached+materialized cold集合，不等待create occurrence；handler可仍在setup，旧generation成功的fresh blank也可能因lazy persistence+restart而消失。phase在首次成功后sticky，后续refresh error仍ready且错误只在private Manager snapshot。rc.2无generic commit-status；禁止幂等create重放时absence必须保留unknown。
- **根因**：把current visibility snapshot、某次refresh结果与历史operation settlement合并成一个linearizable判定。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/client.md](packages/client.md)、[packages/host.md](packages/host.md)。

---

### E043 — projection cache `coldSnapshot`不经过preset presenter

- **类型**：cold-read调用链混淆
- **错误内容**：把ApiProxy history的`presenterScopeFor→standingKeyFor`链套到`SessionProjectionCache.coldSnapshot`，进而认为projection warm-up会按recorded preset import/apply plugins或触发`agent/created`。
- **正解**：coldSnapshot只走cache row、`SessionPersistence.readFrom`、`SessionProjectionRegistry.restore`与write-back；不创建Session/Agent，不调用AgentPresets/presenter。它会执行已注册projection unit的init/apply/view，但不会动态加载preset。
- **根因**：两个API都叫cold read且都消费Session log，却把presentation view与projection fold误当成同一层。
- **发现于**：2026-08-31 · DSH `b150a551`（`0.1.1-rc.2`）
- **牵连条目**：[packages/session.md](packages/session.md)、[packages/preset.md](packages/preset.md)。

---

> 更多**按包组分布**的文档与源码冲突（共 81 条）见 [conflicts.md](conflicts.md)。本文件只保留**跨组、每次回答都可能踩**的条目，以保证它足够短、能在每次回答前被真正扫一遍。


### E044 — 同期 master 的 SessionHandle 改动不属于 alpha.5 发布 tag

- **类型**：版本基线混用；asked_by: human；2026-09-05 复验，`verified_inference`。
- **错误内容**：alpha.4→alpha.5 摘要曾把 master `49a606bc` 的 handle-based persistence 和异步 create 归入 alpha.5 发布行为，并用到 master 的 diff 统计描述发布变化。
- **正解**：alpha.5 tag `db6bdc3576c2d4e7c965e8e3ed0c2a731eed87f5` 与 rc.1 tag `a66e4702047846cdaa10c66c9d3df3951f5ea70d` 仍是同步 `AgentLoop.create(...): Agent`、`SessionPersistence.create(...): Promise<void>`；最新 alpha.1 `d347e703908d0406b7a7ef80e3a0e594d86b2215` 才在发布 tag 中包含异步 create 和 `SessionHandle`。不能以 manifest 中同一个版本号推断 master 与 release 内容相同。
- **根因**：虽记录了不同 SHA，却在结论归属与 diff 分母上混用了 master 和 release tag。
- **源码锚点**：对上述完整 SHA 分别读取 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/agent-loop/src/index.ts`（`AgentLoop.create`）与 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-persistence/src/index.ts`（`SessionPersistence.create`）。
- **牵连条目**：[alpha.4→alpha.5 摘要](topics/版本变更-0.1.2-alpha.4-到-0.1.2-alpha.5.md) 已更正；2026-09-02 log 保留历史原文，本次 log 明确补正。新摘要分别列发布 diff 与镜像快进 diff。

### E045 — 缺少一体式 Node 函数不否定官方浏览器 Session 分层组合

- **类型**：公开导出核验不足与负向推论越界；2026-09-16；asked_by: agent；verified_inference。
- **错误内容**：未逐符号读取 /client 与 CredentialProvider，就暗示完整 CLI/Profile Host 加官方浏览器 Client 缺少生命周期路径，且将凭据写删通知留为 UNKNOWN；把内部 SessionManager/ClientSessions 构造类当公开 runtime 导出。
- **正解**：rc.1 的 /client apply 提供 ctx.sessions；ISession 有 prompt/cancel，Host 按需 resolve/resume Agent。Session 是 type-only export。CredentialProvider 正式提供 set/unset、modifyRecord/deleteRecord 及两类 updated 事件，通知 helper 为 protected。
- **根因与污染追查**：源于本会话未完成导出检查就答复，不是既有专题的证据；既有 Connection 替换专题只讨论“替换官方 Host Provider”的限定情形，不能套用到保留完整官方 Host 的组合。
- **证据**：固定 a66e4702047846cdaa10c66c9d3df3951f5ea70d 的 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/session-controller/src/client/index.ts` 与 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/credentials/credentials/src/index.ts`；详见 [补正专题](topics/rc1-browser-session-credentials.md)。
