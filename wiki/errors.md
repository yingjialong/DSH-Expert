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

> 更多**按包组分布**的文档与源码冲突（共 78 条）见 [conflicts.md](conflicts.md)。本文件只保留**跨组、每次回答都可能踩**的条目，以保证它足够短、能在每次回答前被真正扫一遍。
