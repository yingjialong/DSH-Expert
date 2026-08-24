---
title: packages/settings — user-settings capability family
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/settings/README.md
  - packages/settings/settings/README.md
  - packages/settings/settings/src/index.ts
  - packages/settings/settings-file/README.md
  - packages/settings/settings-file/src/index.ts
  - docs/subsystems/settings.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

把「用户可编辑配置」抽象成 **namespace 注册 + 分层解析 + 可换存储 provider**：插件用 zod schema 注册一个 namespace，拿到 owner 作用域的 `get`/`watch`/`update`，底下换成文件、远端还是浏览器协议由 provider 决定。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。

## 包清单

| 包名 | npm | 一句话职责 |
|---|---|---|
| `settings` | `@deepseek-ai/dsh-settings` | Service Definition：namespace 注册、分层解析、写入提交、`redactSecrets` |
| `settings-file` | `@deepseek-ai/dsh-settings-file` | Service Provider：单个 YAML/JSON 文档承载全部 namespace，监听外部编辑并热发布 |

只有两个包，**没有 model-facing Consumer**：`ctx.settings` 完全是 host 侧注册表，模型看不到它，影响模型的只是各消费方自己把解析值折进 prompt/路由 [T1: packages/settings/settings/README.md]。

## 三件套结构

- **Service Definition**：`dsh-settings`，ctx key `ctx.settings`，服务类是 `abstract class SettingsProvider extends Service` [T1: packages/settings/settings/src/index.ts#SettingsProvider]。
- **Service Provider**：`dsh-settings-file`，`class FileSettingsProvider extends SettingsProvider` [T1: packages/settings/settings-file/src/index.ts#FileSettingsProvider]。
- **Consumer**：本组不提供；消费方是各功能包自己（例如 `dsh-shell` 导出 `SHELL_SETTINGS_NAMESPACE = 'bash'` 交给 shell provider 注册）。

注意命名坑：**Service Definition 的类名就叫 `SettingsProvider`**，与 DSH 术语里的「Service Provider 角色」同名但不是一回事——`settings-file` 才是 Provider 角色。

## 扩展点

想换存储介质（远端配置中心、数据库、浏览器 storage）：

1. 依赖 **`@deepseek-ai/dsh-settings`**（Service Definition），绝不依赖 `settings-file`。
2. 继承 `SettingsProvider`，实现 `writable`、`load()`、`persist(ns, section)`；外部观测到的新文档通过 protected `publish(doc)` 推入。
3. 有本地可编辑文件时再覆写 `documentPath` 与 `prepareDocument()`；非文件 provider 让 `documentPath` 保持 `undefined`，host 配置面板据此判断「能否用原生编辑器打开」。
4. 自带 init（watcher、连接）的 provider 必须先 `yield* super[Service.init]()`，否则基类的「首次 load+publish」不会跑。

想新增一段配置：`ctx.settings.register(ns, schema, { base?, applies? })`，返回值是 owner `SettingsScope`。注册是调用方 fiber 上的 effect，fiber 销毁即注销 namespace 与其观察者；重复 namespace 直接 fail loud。

写入面：`update(ns, patch)`（深合并进 user 层，**永不写 `base`**）、`replace(ns, section)`（整段重置）、`mutate(ns, ops)`（`set`/`unset` 有序操作，给只持有脱敏视图的配置 UI 用）。三者都接受可选 `expectedRevision`，不匹配时抛 `SettingsConflictError`（`code: 'SETTINGS_CONFLICT'`）。

事件两条，语义不同别用错：
- `settings/updated (ns, next, prev, source)`：**解析值真的变了**才触发，`source` 为 `update`（本进程写）或 `provider`（外部改）。
- `settings/document-updated (ns, revision)`：**原始 user section 变了**就触发，哪怕解析值没变。配置 UI 需要这条——把某字段显式覆写成与 base 相同值时，解析值不动但「已覆写/继承」状态和 revision 都变了。

两条事件声明与 `SettingsNamespace`、`SettingsUpdateSource` 类型都在 client-safe 的 `./types` 子路径导出，Host 编译面之外的消费方应从那里取。

## Known Limitations

来自 `dsh-settings`：
- **只有单一 user 层**：解析只认 schema 默认值 + 一个 composition `base` + 一个 user 文档，且不记录每个字段的解析值来自哪一层。
- **`redactSecrets` 不是可信任的 wire 边界**：walker 只走 `object`/`dict`/`array`，藏在 union/intersection/transform 后面的 `role('secret')` 会**原样返回**且 `secrets` 列表为空；`schema.toJSON()` 还会把 secret 字段的 `.default(...)` 发给所有客户端。两种情况都不报错。fail-closed 的 `describeForWire()` 是被推迟的正解。
- **跨进程并发由 provider 定义**：seam 只在进程内按 namespace 串行化写入。

来自 `dsh-settings-file`：
- 同 namespace 冲突是 last-write-wins，无逐值合并/版本校验。
- 漏掉的 watcher 事件不会被补：读路径从不 re-stat，只能等下一次事件/写入/重启。
- 注释保留仅限 YAML 且只保到 map 形状；改动数组会整段替换并带走其中注释。
- 无值间接引用：没有 `${env:VAR}` 之类的引用语法（那是被推迟的 seam 级特性）。

## 陷阱

- `update()` 的 patch 只能是 JSON 兼容数据。Date、Map、BigInt、非有限数、循环引用会带着 `$`-rooted 路径先 reject，任何东西都不落盘——因为 YAML/JSON 存储在重载时会静默改变这些值。
- 写入队列只保证顺序，**不能区分「新写入者」和「持旧快照的写入者」**，要防覆盖必须自己传 `expectedRevision`。
- 配置 UI 千万别「读脱敏 descriptor → 改 → `replace()` 整段写回」：wire 从未返回的 secret 会被整段删掉。这正是 `mutate(ns, ops)` 存在的原因。
- 文件 provider 的写锁是 `<file>.lock` 兄弟文件（`wx` 创建，指数退避，2 秒获取超时）。**竞争者超时后不会强删旧锁**，因为无法从锁文件年龄区分「崩溃的持有者」和「暂停的活写者」——孤儿锁需要人工清理。
- boot 与 reload 的失败策略不一样：**已存在但非法的文档会让插件加载失败（fail loud）**；运行期一次读不出/解析不了的编辑只 warn 并保留 last-good sections。
- 解析值是 deep-frozen 快照；watcher 回调按提交顺序串行异步执行，慢的旧回调不会覆盖新回调，同步 throw 与异步 rejection 都被 contain。

## 去哪深入

- 组结构、ctx key 映射 → `packages/settings/README.md`
- Service API 全表、provider contract、事件语义 → `packages/settings/settings/README.md`
- 文件读写细节（原子重命名、writer lock、YAML leaf-level diff、`resolveSpec(config)`、`path`/`dshHome`/`watch`/`debounceMs` 默认值）→ `packages/settings/settings-file/README.md`
- 子系统参考（namespaces、owner scopes、resolution order、hot commits）→ `docs/subsystems/settings.md`
- 脱敏 walker 源码 → `packages/settings/settings/src/redact.ts`
