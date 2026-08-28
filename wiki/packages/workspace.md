---
title: packages/workspace — workspace 实体族
status: verified_inference
mastery: L1
freshness: stale
anchors:
  - packages/workspace/README.md
  - packages/workspace/workspace/README.md
  - packages/workspace/workspace/src/index.ts
  - packages/workspace/workspace/src/entity.ts
  - packages/workspace/workspace/src/spec.ts
  - packages/workspace/workspace/src/paths.ts
  - docs/subsystems/workspace.md
  - .agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

持久化 workspace：用户目录 + 标题 + **有序的**会话归属；持久 workspace 顺序与 newest-first 候选会话索引通过 domain data form 存储。这是纯 host 侧实体，**模型完全看不见**。

## 稳定性

`Product — stable API`（`packages/README.md` 表格中 `workspace/` 行原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `workspace/workspace/` | `@deepseek-ai/dsh-workspace` | `WorkspaceRegistry`（`ctx.workspaceRegistry`）：注册 workspace、核算其会话 |

## 三件套结构

**单包实体族，不是 capability seam**。没有可替换的 Provider 角色：`WorkspaceRegistry extends Service` 既是服务定义也是实现，消费者只见 `Workspace` 接口，**实体实现是包私有的** [T1: packages/workspace/workspace/src/index.ts、entity.ts]。持久化不是通过 provider 注册，而是**依赖两个启动期必需的 peer**：`storageDomain` 与 `sessionPersistence`。

## 扩展点

- **想改存储后端**：换 `ctx.storageDomain` 的后端（属于 `packages/storage/` 那一组），而不是往这里注册东西。
- **消费面 API**（`ctx.workspaceRegistry`）：
  - `create(path, title?)` —— 经 `fs.realpath` 规范化，拒绝不存在或非目录路径，**每个 canonical path 至多一条记录**，新记录**前插**到持久顺序；对已存在路径重复调用返回原 workspace 且不改标题；不同路径可以共用显示标题。
  - `get(id)` / `list()` / `resolveByPath(path)` —— 走缓存。`list()` **同步**并遵循持久注册顺序；`resolveByPath` **异步**（要走同一套 realpath 规范化），路径缺失时**拒绝而不是创建**。
  - `insertBefore(id, before?)` —— DOM-insertBefore 语义（放到锚点前，省略锚点则追加），返回**完整的已提交顺序** id 列表。
  - `delete(id)` —— 只删注册、顺序项和会话账目；**目录、用户文件、live Session、持久化 session log 一概不动**，那些 Session 变成 Ungrouped。表写入失败会回滚到先前顺序与已发布实体。
  - `archiveSession(id)` / `archivedSessionIds` —— 注册表全局归档集，叠在 workspace 核算之上：归档会话从分组界面消失，但**保留 session log 和它的 `sessionIds` 槽位**，将来取消归档能回到原位。已归档 id 再归档直接 resolve 不写盘，未知 id 拒绝；字段出现之前写下的状态按空集解析。
  - `Workspace.attachSession(id)` / `detachSession` / `insertSessionBefore(id, before?)` / `sessionIds` / `status()`。
- 另导出 `WorkspaceId(id)` 构造器与 branded 类型、`realpathNormalize`、`workspaceDomainState` / `workspaceRecord` / `workspaceDomainSpec`、以及 `WorkspaceMoveInvalidError`、`WorkspaceUnknownSessionError`、`WorkspaceOrderInvalidError` [T1: packages/workspace/workspace/src/index.ts]。

## Known Limitations

- **会话删除与破坏性目录删除是两个独立且当前不存在的能力**；workspace 注册删除**永远不能拿来代替**它们。
- **header 索引只在启动时、以及 attach 必须解析未缓存的持久 id 时刷新**；别的进程做的删除或 cwd 破坏，要到下次刷新或重启才被观察到。

## 陷阱

- **realpath 是身份的唯一权威**。`attachSession` 会校验 live 或持久 session header 的 cwd 是否匹配 workspace path，不匹配就拒绝且不写盘。未知 session、缺失/不可解析/非目录的 cwd 值同样拒绝。
- **删除后再注册同一路径会得到全新的 workspace id，并且不会自动重新收养保留下来的 Session。**
- **启动期一致性是 fail-loud 的**：一个 session 被索引到两个 workspace、两条记录声称同一路径、或与持久 workspace 顺序发生分歧，都会在启动时拒绝。create/delete 会**先**写一个显式的 pending-mutation 标记再动记录与顺序；启动只补完被标记的那次变更然后清标记，**未被标记的顺序/表不一致就是无法解释的损坏，直接大声失败**。
- **initialized marker 最后写**，所以部分 bootstrap 写入在重启后可以安全复用；peer 不可用时插件保持 pending，**不能提交空的 initialized marker**。
- 首次成功启动时注册表会调 `SessionPersistence.list()`，**只读 header 的 `id` / `cwd` / `createdAt`**，绝不读事件体；之后只有 cwd 的会话仍是 Ungrouped。
- `Workspace.sessionIds` 是**同步投影**，会过滤掉缺 header、cwd 非法、cwd 不匹配的项，真正的裁剪发生在下一次 workspace 变更时。
- `status()` 是**未缓存**的目录检查（`'ok' | 'missing-dir'`），目录缺失**永不改动记录**。

## 2026-08-22 agent 审核增量（fork 继承面与 create 参数现状）

- **`session.fork` 除 cwd/lineage 外还固定继承 source 的 `agentPreset` composition**（`packages/host/apiproxy/src/api-proxy.ts` 约 L2321-2335 注释：seed 历史是在那套 tools 下产生的，compose 其他组合会搁浅已携带的 tool calls）——**跨 composition fork 同样没有公共入口**，与"跨 workspace fork 无入口"是同一条边界。
- **fork 的 cut 会向前扩展**：boundary turn/end 之后、下一个 turn/start 之前的 out-of-band 事件（`session/title`、injections）被纳入 seed——child 会继承 boundary 之后刚生成的 title（api-proxy.ts 约 L2299-2303 注释明说此意图）。
- **`WorkspaceRegistry.create` 的 title 参数处于上游废弃流程**：`index.ts` 约 L151-155 的 TODO 注明 2026-07-31 gateway create-by-name 分支删除后 title 已无生产调用者，将来会删参数与 README 对应行——**不要把 title 当稳定 API**；wire 面 `workspace.create` 只收 `{ path }`，title 默认取 path basename。
- **`create` 按 canonical path 幂等**：同 realpath 重复调用返回既有 entity 且不改 title——宿主重启后重复注册不会产生第二个 Workspace。
- **membership 自动 prune**：entity 每次 mutate 都过滤 canonical cwd 不再等于 workspace path 的 session（`entity.ts` 约 L215-217），registry 启动时对候选仅 warn——session 归属始终由 cwd realpath 权威决定，`Workspace.sessionIds` 是同步投影。
- **`archiveSession` 是 registry-global 集合**（`workspaceDomainState.archivedSessionIds`，spec.ts 约 L54）：archived session 保留 workspace sessionIds slot——这是为未来 unarchive 预留的位置记忆；rc.2 公共面（registry/entity contract/wire API）**只有 archiveSession，无 unarchive / archive-remove**，上游注释自述 unarchive 为 future（apiproxy workspace.ts 约 L102）。宿主做"归档会话删除/恢复"在此 seam 出现前只能 fail closed。
- **storage-domain 写路径顺序**：「后端持久化 → 内存 → domain/changed 事件」，读到的内存态永远是已落盘的；但**每域单链串行**——一次慢写会挡住该域后续所有写（storage-domain/src/domain.ts 模块注释与 enqueue 实现）。

## 去哪深入（文件路由）

| 想查 | 去 |
|---|---|
| 实体、realpath canon、注册/解析的子系统参考 | `docs/subsystems/workspace.md` |
| 生命周期、持久化、删除语义的权威说明 | `packages/workspace/workspace/README.md` §Shape |
| 服务实现（含启动 bootstrap、pending marker） | `packages/workspace/workspace/src/index.ts#WorkspaceRegistry` |
| 实体实现与移动错误 | `packages/workspace/workspace/src/entity.ts` |
| 存储 spec / record 形状 | `packages/workspace/workspace/src/spec.ts` |
| 实体与存储设计（**proposed 状态**） | `.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.md` |
| header-only bootstrap 与 GUI 排序 | `.agents/notes/implemented/feature/2026-07-25-workspace-ui-product-flow.md` |
