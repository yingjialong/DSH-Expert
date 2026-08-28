---
title: packages/runtime-diagnostics — 包自有运行期不变式注册表
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/runtime-diagnostics/invariants/README.md
  - packages/runtime-diagnostics/invariants/src/index.ts
  - packages/runtime-diagnostics/invariants/package.json
  - packages/AGENTS.md
  - docs/subsystems/invariants.md
  - scripts/verify-package-invariants.ts
  - packages/README.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

一个可配置的注册表服务 `ctx.invariants`：每个 workspace 包都发布一个 `./invariant` companion，在运行期检查自己拥有的「事件关系 / 可变数据关系」，检查失败抛 `InvariantError`。

## 稳定性

**README 未列出**。`packages/README.md` 的 Hierarchy 表格漏列了 `runtime-diagnostics/` 组（同样漏列的还有 `mcp/`），所以没有官方 Release expectation 标注。可参考的旁证：该组唯一的包 `@deepseek-ai/dsh-invariants` 不是 `private`，且 `packages/AGENTS.md` 把「每个包都要 own `./invariant`」写成硬性规则并有门禁脚本，实际按产品件对待。

## 包清单

| 包名 | npm 名 | 职责 |
|---|---|---|
| `invariants` | `@deepseek-ai/dsh-invariants` | 注册表服务 `ctx.invariants` + `InvariantError`；根插件不含任何产品检查、不 import 任何产品包 |

注意 npm 名是 `@deepseek-ai/dsh-invariants`，**不是** `dsh-runtime-diagnostics-invariants`——组名与包名在这里不同名（这符合仓库规则「组holds `packages/<group>/<pkg>/`，名字保持 `@deepseek-ai/dsh-<pkg>`」）。

## 三件套结构

- **Service Definition + 实现**：`invariants` 一个包（`InvariantRegistry extends Service`，默认导出）。
- **Provider（检查的提供方）**：**不在本组**。它们是散布在全仓库每个包里的 `./invariant` companion 导出（如 `@deepseek-ai/dsh-session/invariant`、`@deepseek-ai/dsh-sandbox-policy/invariant`），各自用自己的完整 npm 名调 `ctx.invariants.register(packageName, installer)`。
- **Consumer**：composition 本身（`cordis.yml` / `ctx.plugin()`）。没有 model-facing tool——本组 Model Experience 明确为 None。

## 扩展点

- **要给自己的包加运行期不变式**：依赖 `@deepseek-ai/dsh-invariants`，在包内新增 `src/invariant.ts` 并在 `exports` 里发布 `./invariant`，注册**完整 npm 包名**；installer 可用 `installer.inject` 声明依赖的服务，并接收 `fail(message)`。
- **installer 的合法检查对象**（[T1: packages/AGENTS.md]）：权威事件流或可变数据的关系。**不**检查方法/插件名/注入/effect 是否存在，也不检查固定纯函数结果——那是类型、加载或单测的事。
- **没有合理关系时**：用空 installer，并写一条以 `No runtime invariant:` 开头的包内专属注释说明原因；门禁只接受有解释的空 companion。
- **开关与筛选**：`Config` 三个字段 `enabled?` / `package_allowlist?` / `package_blocklist?`，默认 `true` / `[]` / `[]`。每项是大小写敏感的 JS 正则**源串**，用 `new RegExp(pattern)` 编译。
- **门禁**：`pnpm run verify-package-invariants`（`scripts/verify-package-invariants.ts`）会扫描全部 workspace 包，拒绝生成的 marker、无解释的空 installer、拿了 reporter 却不用的非空 installer、注册名写错，以及 export / publication / dependency / TS reference / bundle 接线不完整。

## Known Limitations

来自 [T1: packages/runtime-diagnostics/invariants/README.md] 的 `## Known Limitations and Deferred Work`：

- request 重建只覆盖 loop 在冻结前显式标记过的请求；直接的一次性 LLM 调用即使调用方自己冻结或附了 session id 也在该 marker 契约之外。
- 只观察 live 状态的 lifecycle companion 无法重建在它自己 reload 之前就开始的操作；标准与测试 composition 都在对应操作开始前挂载它们。
- 正则筛选在服务生命周期内固定，改筛选要走普通 Cordis 插件 reload。

## 陷阱

- **组名/包名不一致**：写依赖时是 `@deepseek-ai/dsh-invariants`，写路径时是 `packages/runtime-diagnostics/invariants/`。仓库根 `AGENTS.md` 的 Repository layout 段落里也**没有**列出 `runtime-diagnostics/`（它列了 `support/`、`self-modification/` 这类并不存在的目录名），别拿那段当目录清单。
- **blocklist 覆盖 allowlist**：一个包只有在「服务 enabled + allowlist 为空或至少一条匹配全名 + 没有 blocklist 匹配」时才被选中。
- **匹配默认不锚定**：`/pattern/flags` 语法不被解析；要精确匹配必须自己写 `^…$`。空白、含前后空格、非法、以及同一 list 内重复的条目会让**服务启动失败**（不是静默忽略）。
- **注册与激活分离**：`register()` 即使被 filter 挡住不激活，也仍然占用该 npm 名的唯一活跃注册位并返回 disposer。
- **只加载服务不装任何产品检查**；只加载 companion 而不加载服务，则它会一直等在自己声明的 `invariants` 注入上。
- **每个包 tsconfig 都要 reference `runtime-diagnostics/invariants`**（[T1: packages/AGENTS.md 的 Package tsconfig 规则]），这是新建包时最容易漏的一步。
- **Session 自己拥有的那部分不在 companion 里**：不可变、surface-valid 的日志存储由 `Session` 在每个 composition 里直接保证（无损 JSON 快照、引用源事件覆盖校验、位置替换限制、deep-freeze）；`dsh-session` 的 companion 只查 Session 不拥有的跨记录规则。

## 去哪深入（文件路由）

| 问题 | 去这里 |
|---|---|
| 服务 API、Config 语义、companion 编写规则、当前可执行 companion 清单表 | `packages/runtime-diagnostics/invariants/README.md` |
| `InvariantRegistry` / `InvariantInstaller` / `InvariantError` 定义 | `packages/runtime-diagnostics/invariants/src/index.ts` |
| 子系统级说明 | `docs/subsystems/invariants.md` |
| 包级硬性规则（每个包 own `./invariant`；tsconfig reference） | `packages/AGENTS.md` |
| 门禁脚本实现与拒绝条件 | `scripts/verify-package-invariants.ts`、`scripts/package-invariants.ts` |
| 设计决策 | `.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.md` |
