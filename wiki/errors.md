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

> 更多**按包组分布**的文档与源码冲突（共 75 条）见 [conflicts.md](conflicts.md)。本文件只保留**跨组、每次回答都可能踩**的条目，以保证它足够短、能在每次回答前被真正扫一遍。
