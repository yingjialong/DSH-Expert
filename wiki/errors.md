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
