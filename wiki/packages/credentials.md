---
title: packages/credentials — 凭据引用能力族
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/credentials/README.md
  - packages/credentials/credentials/README.md
  - packages/credentials/credentials-local/README.md
  - packages/credentials/credentials/src/index.ts
  - packages/credentials/credentials/src/types.ts
  - packages/credentials/credentials-local/src/index.ts
  - docs/subsystems/credentials.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

让配置只携带**指向密钥的引用**而不是密钥本身：`ctx.credentials` 现在管**两个 key space**——`CredentialRef`（环境变量名）的分层解析，和 `CredentialKey`（`<scope>/<id>` 存储记录）；`credentials-local` 用「进程环境 > 托管 YAML > 项目 `.env` > 用户 `.env`」四层实现 ref 半边并托管 record 存储；`authorization` 提供「问人要凭据」的 flow 注册（`ctx.authorization`）。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文，组描述已更新为 "Credential reference/record seam + env-over-`.env` provider + authorization flows"）。

## 包清单

| 包目录 | npm 名 | ctx key | 一句话职责 |
|---|---|---|---|
| `credentials/` | `@deepseek-ai/dsh-credentials` | `ctx.credentials` | Service Definition：两个 key space，`CredentialRef` + `resolve/describe/set/unset`，`CredentialKey` + `readRecord/describeRecord/listRecords/modifyRecord/deleteRecord`，`credentials/reference-updated` 与 `credentials/record-updated` 事件 |
| `credentials-local/` | `@deepseek-ai/dsh-credentials-local` | 注册 `ctx.credentials` | 环境 + `$DSH_HOME/.credentials.yaml` + 两层 `.env` 的文件型 provider；同时是 record 半边的存储（versioned YAML） |
| `authorization/` | `@deepseek-ai/dsh-authorization` | `ctx.authorization` | 凭据获取 flow 注册表：插件注册「怎么向人要一个凭据」，`begin` 跑一次 attempt，产出写进 record |

## 三件套结构

- **Service Definition**：`@deepseek-ai/dsh-credentials`。抽象类覆盖**两个 key space**：
  - **ref 半边**（`CredentialRef`，回答「这个环境变量名背后是什么」）：`resolve(ref)` → `{ value, source } | undefined`、`describe(ref)` → `{ configured, source?, writable }`（**永不返回值**）、`set(ref, value)`、`unset(ref)`。`credentialRef('DEEPSEEK_API_KEY')` 是 branded 的 POSIX shell 标识符。
  - **record 半边**（`CredentialKey`，回答「这个插件为这个 id 持有什么凭据」，authorization grant 没有环境可读所以**无分层、presence 即全部事实**）：`readRecord(key)` → `CredentialRecord | undefined`、`describeRecord(key)` → `{ configured, kind?, writable }`、`listRecords()` → `[{ key, kind }]`、`modifyRecord(key, mutate)`（**唯一写路径**，锁下 read-decide-replace）、`deleteRecord(key)`。`credentialKey('llm-pi-ai', 'openai-codex')` 造 `<scope>/<id>`（scope 是 owner 插件注册名，小写连字符段；`/` 保证与 ref 语法不相交）。记录是 `ApiKeyRecord`（`kind: 'api-key'`，可带 `key` / `env`）或 `GrantRecord`（`kind: 'grant'`，payload 对 seam 不透明，只要求过 JSON round trip）二选一。
  - 另有谓词 `isCredentialRefName` / `isCredentialKeySegment`（grammar 外的名字读作「没存」而非抛错）和拆分 `parseCredentialKey` / `credentialKeyScope` / `credentialKeyId`。
- **Service Provider**：`@deepseek-ai/dsh-credentials-local`，两个半边都实现。seam 形状为 keyring / helper-command / KMS provider 留了位置。
- **Consumer**：不在本组。典型消费者是 `llm/` 组的各 adapter，它们**每次 model request 解析一次** ref；`authorization` 的 flow 则通过 `modifyRecord` 把获取到的凭据写成 record。

## 扩展点

- **要接 OS keychain / KMS / helper 命令**：依赖 `@deepseek-ai/dsh-credentials`，实现并注册 `ctx.credentials`，作为 `credentials-local` 的兄弟包存在。这也是 README 里点名的「deferred answer」。
- **客户端安全子路径**：`credentials/reference-updated` / `credentials/record-updated` 事件声明和 `CredentialRef` / `CredentialKey` / `CredentialRecord` 类型都在 `./types` 子路径导出（包根再 re-export），Host 编译面之外的消费者从这里读签名，不要自己重述。
- **消费者纪律**：在**操作边界**解析，不要跨操作缓存——这正是「改了密钥下一次请求就生效、不用重启插件」的原因。
- **UI 纪律**：用 `describe().writable` 决定是否把该引用渲染成只读，不要先写再看报错。

## Known Limitations

- **ref 半边无枚举**：seam 只回答你给它的引用的问题，ref 没有 `list()`（配置面从 settings schema 学到引用存在；当前没有消费者需要）。**record 半边有 `listRecords()`**——record 没有 schema 之类的发现路径，不枚举就无法展示「用户授权了什么」、找不到卸载插件留下的孤儿。
- ref 是**环境变量形状**的一维 POSIX 标识符命名空间；record 用 `<owner>/<id>` 寻址，`/` 使两种语法不相交。
- **进程环境变化不可观测**：没有事件能为它触发，UI 只能在自己导航时重新 `describe()`。
- **record 的 scope 不被校验挂载**：seam 存它收到的、报它存的；识别孤儿是调用方拿 `listRecords()` 与 scope 注册表自己 join。
- `credentials-local`：**同一引用（ref 半边）**的并发写是 last-write-wins（有 writer 锁和 read-modify-write，但没有 revision 检查）；**record 半边不是**——`modifyRecord` 在跨进程 writer 锁下串行 read-decide-replace（锁等待上限 `DOCUMENT_LOCK_WAIT_MS = 30_000`，token 刷新含网络往返所以等得久），并发 rotate refresh token 不会丢先写的一方；同 UID 进程能读到文档；环境快照在启动时冻结；atomic 但非 crash-durable。

## 陷阱

- **`set`/`unset` 在被只读层遮蔽时会 reject，这是故意的 fail-loud**。当进程环境正在供给该引用时，写入会「看起来成功但解析仍返回旧值」，所以 seam 直接拒绝。
- **启动环境永远赢**：`DEEPSEEK_API_KEY=… dsh`、CI secret、容器 `-e` 都是本次运行的操作者意图，`describe()` 报 `source: 'env', writable: false`。
- **空值等于不存在**，全链路一致：`resolve` 跳过、`describe` 报未配置。所以 YAML 文档里写空字符串会被**直接拒绝**——`unset` 是删 key，不是把值清空。
- **`0600` 挡的是别的 OS 用户，不是模型**。bash 和文件系统工具以同一用户身份运行，`workspace-write` 文件策略只约束写不约束读。harness 做到的只是「不把该文档的解析路径交给模型、不把它加载进进程环境」——这是审慎，不是边界。
- **`$DSH_HOME/.env` 与 `.credentials.yaml` 待遇不同**：前者是用户普通的环境层（会进环境），后者不会。
- **产品 CLI 下读的是 launcher 冻结的 environment snapshot 而不是 `process.env`**（只有 snapshot 能区分「来自启动 shell」还是「来自文件」）；不是产品 CLI 启动的组合则只有继承环境这一层。
- **文档格式是 versioned 的**：`version: 1` + `refs:`（POSIX 标识符 → 非空字符串）+ `records:`（`<scope>/<id>` → 带 `kind` 标签的映射），顶层只允许这三个 key；任何偏离（非映射根、未知顶层 key、不可寻址的 key、非字符串值、空串、未知 record 标签或字段、重复 key、畸形 YAML）都是拒绝而不是跳过——boot 时 fail loud，热重载时 warn 并保留上一份好快照。**pre-release 的平面布局（无 `version` 的 `ref: value` 映射）是唯一例外**：boot 时若能精确识别（可寻址名 + 非空字符串标量 + 无 directive），会在 writer 锁下**原地迁移**成 version 1（原行逐字嵌进 `refs:`，值与注释一字不动）；其余平面形状按条目数报错并给出唯一改法，绝不读成空 store；**热重载不迁移**——运行中恢复的平面文档保持上一份好快照到下次 boot。`grant` payload 两个方向都验 JSON round trip（`.inf`、alias 环、`Date`、`bigint` 一律拒收，不存有损值）。
- **POSIX 上权限过宽会在解析内容之前失败**（带任何 group/other 位就报错并给出 `chmod 600` 修复提示）；Windows 无 mode 可查，直接跳过而不是假装检查。

## 去哪深入（文件路由）

| 想知道什么 | 去哪 |
|---|---|
| seam 的三条 doctrine、两个 key space、方法签名、`credentials/reference-updated` / `credentials/record-updated` 语义与遮蔽规则 | `packages/credentials/credentials/README.md` |
| 四层优先级表、config 四项（`path`/`dshHome`/`watch`/`debounceMs`）、versioned 文档格式与平面布局迁移、权限与热重载 | `packages/credentials/credentials-local/README.md` |
| `CredentialRef`、per-operation 解析、UI 安全的 `CredentialInfo` / `CredentialRecordInfo`、provider 分层 | `docs/subsystems/credentials.md` |
| `ctx.authorization`：`registerFlow` / `begin` / `cancel`、`AuthorizationError` 错误码、`authorization/settled` 事件 | `packages/credentials/authorization/README.md` |
| 谁在消费（LLM adapter 每次请求解析一次） | `packages/llm/README.md` |
| Harness home 的各层含义 | `packages/boot/app-boot/README.md`（Profiles 段） |
| 原子写与跨进程 writer 锁 | `packages/util/atomic-write/README.md` |
| 可跑的凭据场景 | `examples/headless-agent/credentials.cordis.snapshot.yml` |
