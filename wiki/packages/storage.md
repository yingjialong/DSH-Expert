---
title: packages/storage — non-session storage family
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/storage/README.md
  - packages/storage/storage/README.md
  - packages/storage/storage/src/index.ts
  - packages/storage/storage/src/backend.ts
  - packages/storage/storage-domain/README.md
  - packages/storage/storage-domain/src/index.ts
  - docs/subsystems/storage.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

**session 事件日志以外**的应用数据都走这里：`ctx.storage` 是一个「命名后端注册表 + 已挂载数据形态（data form）」的枢纽，枢纽本身**不做任何 IO**——介质归后端，语义归 form。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文）。注意其设计 Agent Note 仍在 `proposed/` 目录下。

## 包清单

| 包名 | npm | 一句话职责 |
|---|---|---|
| `storage` | `@deepseek-ai/dsh-storage` | 枢纽：后端注册表 + data form 挂载点（`ctx.storage`） |
| `storage-json` | `@deepseek-ai/dsh-storage-json` | 后端 `json`：每个 unit 一个人类可读的 `<unit>.json` |
| `storage-sqlite` | `@deepseek-ai/dsh-storage-sqlite` | 后端 `sqlite`：`node:sqlite` 单库文件（或 `:memory:`）承载 `kv` facet |
| `storage-domain` | `@deepseek-ai/dsh-storage-domain` | data form：校验过的域记录存储（`ctx.storageDomain` / `ctx.storage.domain`） |

## 三件套结构

这一组**不是标准三件套**，形状是「枢纽 + 后端 + 数据形态」，且**没有 model-facing Consumer**：

- **Service Definition / 枢纽**：`dsh-storage`，ctx key `ctx.storage`，`class Storage extends Service` [T1: packages/storage/storage/src/index.ts#Storage]。后端契约在 `packages/storage/storage/src/backend.ts`。
- **后端（介质 Provider）**：`storage-json`（`class JsonStorageBackend implements StorageBackend`）、`storage-sqlite`（`class SqliteStorageBackend implements StorageBackend`）。二者可**并存**。
- **数据形态（语义层）**：`storage-domain`，`class DomainFacility` 挂到 `ctx.storage.domain`，同时暴露可注入的 `ctx.storageDomain` [T1: packages/storage/storage-domain/src/index.ts#DomainFacility]。

模型侧一无所见：枢纽不注册工具、不注入 prompt、不写 session 事件，也就不可能影响请求前缀与 KV cache 复用 [T1: packages/storage/storage/README.md]。

## 扩展点

- **加一个新介质**：实现 `StorageBackend`（`src/backend.ts` 拥有 `kv` facet 的精确契约），用 `ctx.storage.backend.register()` 注册，返回值就是 disposer。重名注册与未知查找都 fail loud。
- **加一个新数据形态**：`ctx.storage.mount(form, facility)` / `ctx.storage.form(form)`。`StorageForms` 是 **merge-extensible** 的接口，domain 层就是合并了 `domain` 这一项从而能被 `ctx.storage.domain` 取到。
- **消费方永远用 data form，不直接碰后端**（组 README 原文纪律）。哪个后端服务哪个消费方，是那个消费方的配置（domain 层的路由表）决定的，**不是枢纽的全局选择**。
- **声明一个域**：`defineDomain`（zod record schema，类型用 `z.infer` 推出），`DomainFacility.open` 打开，读同步（来自权威内存态），写在**每域一条**的串行链上：先在路由到的后端落持久化，再更新内存，最后发 `domain/changed`。打开方拥有句柄生命周期，用 `Domain.close()`（幂等，通常挂自己的 `ctx.effect` disposer）释放；插件卸载时仍打开的域由 facility 关闭。
- **domain 配置**只有两个键：`backend`（每个域的默认后端名，**必填**——「没有普遍正确的介质」）与 `routes`（域名 → 后端名的逐域覆盖）。

## Known Limitations

- `dsh-storage`：**`kv` 是目前唯一的数据形状**，后端只有这一个 facet 要实现；form 是懒解析的——在 domain 插件挂载前读 `ctx.storage.domain` 会抛 `form-not-mounted`，装配顺序错就大声失败而非静默延迟。
- `dsh-storage-domain`：`domain/changed` 是**进程内事件**，第二个 host 进程或重连的 GUI 观察不到变化（跨进程 revision 方案在 Agent Note 里被推迟）；**无跨表事务、无二级索引、无多段 key**，每次写只碰一条记录。
- `dsh-storage-json`：Windows 持久性依赖 libuv 的 `rename()`（`MoveFileExW` 带替换）且**没有显式 write-through**；**无跨进程写锁**，两个进程写同一 root 会交错整文件替换（last write wins）。
- `dsh-storage-sqlite`：`DatabaseSync` 是**同步**的，每次写会阻塞事件循环（单语句时长，域数据规模下可接受）；**无 busy-wait / 重试策略**，另一连接持写事务时立即拒绝；**只打开当前 `STORAGE_SQLITE_SCHEMA_VERSION`**，其他版本戳一律拒绝而非迁移（pre-release 立场）；`openDatabase` 与 session persistence 的 SQLite 打开序列重复，抽取到共享媒体层被推迟。

## 陷阱

- 别把它当 session 日志用。这一组的定位就是 **non-session** 数据（workspace 记录、未来的 session sidecar）；session 事件日志是 `packages/session/` 那一组。
- `ctx.storageDomain` 与 `ctx.storage.domain` 是**同一个 facility 的两个入口**，且只在「每个配置的后端都注册完之后」才出现。装配顺序：先后端插件，再 domain 插件，最后消费方。
- domain 写路径的顺序是「持久化 → 内存 → 事件」，因此**读到的内存态永远是已落盘的**；但也意味着后端一次慢写会挡住该域后续所有写（每域单链串行）。
- SQLite 后端拒绝旧 schema version 而不迁移，升级 DSH 后老库文件会**直接打不开**，不是「自动升级」。
- 多后端可以同时挂着（`json` + `sqlite`），所以"我配了 sqlite"不代表某个域走 sqlite——要看 domain 的 `routes`。

## 去哪深入

- 组结构、四包分工、ctx key → `packages/storage/README.md`
- 枢纽形状（`backend` 表、`mount`/`form`、`StorageForms` 合并扩展）→ `packages/storage/storage/README.md`
- `kv` facet 的精确契约 → `packages/storage/storage/src/backend.ts`
- `defineDomain` / `DomainFacility.open` / `Domain.close` / `domain/changed` 语义与配置 → `packages/storage/storage-domain/README.md`
- 子系统参考（后端契约、`StorageForms`、`DomainSpec`/`Domain`、`domain/changed`）→ `docs/subsystems/storage.md`
- 设计理由与推迟事项清单 → `.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.md`
