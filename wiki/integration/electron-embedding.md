---
title: integration/electron-embedding — Electron 与嵌入运行时集成约束
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/client/connection/src/client/index.ts
  - packages/client/runtime/README.md
  - packages/client/runtime/src/client/contract/workspaces.ts
  - packages/client/runtime/src/client/workspaces/service.ts
  - packages/boot/app-boot/src/index.ts
  - vendor/loader/src/index.ts
  - vendor/hmr/README.md
  - patches/node-pty@1.2.0-beta.15.patch
  - apps/cli/src/dump-config.ts
  - packages/host/apiproxy/README.md
  - packages/host/apiproxy/src/api/events.ts
anchors_note: 逐条见正文各行内锚点
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: agent
---

# Electron 与嵌入运行时集成约束

> 八维坐标：宿主语言 TypeScript · 运行形态 Electron（main + UtilityProcess）或任意「宿主拥有物理传输/安装树」的嵌入 runtime · 并发与会话隔离 N/A · 沙箱与文件系统 app-owned 安装树（宿主可解析路径）· 工具复用官方 bundle · 模型与凭据经宿主 carrier · 部署环境打包产物（asar 等）· 版本基线 0.1.1-rc.2。
>
> 适用问题：把 DSH host 或 client 装进一个**不由 DSH 自己启动/自己管文件布局**的进程里（Electron、打包产物、sidecar 进程），哪些官方 seam 可用、哪些能力结构性缺席。

## 1. Client 侧 carrier 的正门：`__DSH_TRANSPORT__`

- **条件**：宿主（如 Electron renderer）要让官方 client runtime 跑在自定义物理传输上。
- **结论**：rc.2 的官方接缝是 `@deepseek-ai/dsh-client-connection/client` subpath 的 `ClientTransportHooks`——connection `apply()` 从 `globalThis.__DSH_TRANSPORT__` 读取 `createApiClient` / `fetch`，构造并 provide `ConnectionHandle`；`handle.start()` 才创建包内 `ConnectionController`；可选 `loadBundle` 由 client-web boot 消费。宿主只写物理 transport adapter，不声明/拼装 handle 或 controller。
- **版本演进**：rc.8 时该接缝形态是 client-web boot 的显式 `BootSeams` 参数；rc.2 统一并入 `__DSH_TRANSPORT__` 全局钩子。rc.8 公共面**没有** carrier factory（`apply()` 固定 Web/fixture 路径），跨 rc 升级时要按此重查。
- **锚点**：`packages/client/connection/src/client/index.ts`（约 L56–L169）。
- **认知状态**：verified_inference（源码级；未实测）。

## 2. `/client` subpath 是硬性要求

- **条件**：插件 bundle 或宿主代码 import client runtime / connection。
- **结论**：必须用 `@deepseek-ai/dsh-client-runtime/client` 这类 `/client` subpath。裸包名 import 会 inline 第二个模块实例，其私有 scope-tag Symbol 永不匹配——表现为状态静默错乱而非报错（README Known Limitations 明文）。
- **锚点**：`packages/client/runtime/README.md`。

## 3. `loader.internal` 缺席时连锁失效的能力

`loader.internal`（进程内模块加载器）不是所有宿主都可用。缺席时（源码可证的后果，不依赖具体宿主）：

| 能力 | 缺席时行为 | 锚点 |
| --- | --- | --- |
| `bareModuleBaseUrl` | **参数不参与解析**：bare 包名静默退回普通 Node resolution；插件必须物理落在宿主普通 import 能解析到的位置 | `packages/boot/app-boot/src/index.ts`（约 L492–L504） |
| 官方 HMR（`vendor/hmr`） | 插件直接 throw（"The package throws if the loader service has no internal module loader available"）——嵌入宿主没有官方热重载 | `vendor/hmr/README.md` |
| Loader 配置 schema | 注意 `bareModuleBaseUrl` 是 `boot()` / `mountRootInclude()` 的**函数参数**，不是 loader 配置键（`vendor/loader/src/index.ts` 的 `Loader.Config` 只有 `baseUrl`） | 两处各一 |

- **认知状态**：verified_inference。具体某宿主（如 Electron UtilityProcess）中 internal 是否可用属宿主实测问题，本页不下结论。

## 4. asar 与 native 伴生文件

- **条件**：宿主把 DSH 装进 asar 类只读归档。
- **结论**：**loader 对 asar 内资源无任何官方支持或文档**（全仓 `rg -i asar` 仅命中 node-pty patch）。native 伴生文件布局有一个官方先例：`patches/node-pty@1.2.0-beta.15.patch`（rc.7 起）定义 spawn-helper 三级解析——`DSH_NODE_PTY_SPAWN_HELPER` env → `execPath` 同级 → `asar.unpacked` 兜底，patch 注释明说是给 "external embedded-runtime consumer" 的通道；Python `sdk-runtime` 也以 `-spawn-helper` / `-rg` sidecar 布局分发单文件 exe。嵌入宿主打包 PTY/native helper 可复用该官方 patch 与 env 通道。
- **锚点**：`patches/node-pty@1.2.0-beta.15.patch`。

## 5. 启动断言与配置对照的官方工具

- **`assertEntriesActivated`**（app-boot）：entry/fiber 级激活断言——每个 enabled entry 的 fiber 必须 ACTIVE；FAILED 时取回原始 rejection；**PENDING 时把未满足的 inject 服务名逐个点名写进错误信息**（宿主诊断绑定缺口的最低成本信号）。**没有**「服务/工具/projection 激活集枚举」的官方 API——需要激活集清单时须自行经 ctx.registry 构建；`cordis_inspect what:"events"` 读的是构建期生成目录（`pnpm run gen-cordis-api` 产物），不是运行时 registry。
- **`dsh --profile <name> --dump-config`**：官方 boot-free 配置组合对照工具——用与 boot 完全相同的 applyEntryPatches 单调用组合、`!!js` 原样不求值、逐层注释来源。宿主做「构建期配置快照 vs 运行时组合」双向对照时，这是官方对齐物。
- **`boot()` 的 prepare hook**：在任何配置树 entry 挂载前运行（app-boot/src/index.ts 约 L747/L772），宿主可在此提供 launcher-owned context slots——carrier 集成的官方接缝位置。
- **锚点**：`packages/boot/app-boot/src/index.ts`、`apps/cli/src/dump-config.ts`。

## 6. Workspace/Session 创建面的接口级约束

- **跨域 face `SessionsPort.create` 只接受 `{ workspaceId}`**（`packages/client/runtime/src/client/contract/workspaces.ts`）——比具体类 `SessionRuntime.create`（还收 cwd/sessionId）窄一个量级；官方用接口形状约束「session 创建走 workspace 路径」。
- **wire 面 `workspace.create` 只收 `{ path }`**，title 默认取 path basename；`WorkspaceRegistry.create(anchor, title)` 的 title 参数已无生产调用方、处于上游废弃流程（index.ts TODO 注释），**不要把 title 当稳定 API**。
- **`WorkspaceRegistry.create` 按 canonical path 幂等**：同 realpath 重复调用返回既有 entity 不改 title——重启后重复注册不会产生第二个 Workspace。
- **`SessionCreateError` 携带 `requestedSessionId`**（workspaces/service.ts 约 L107–L120）：caller-preallocated id 失败时可凭它对齐后续 stream/list 对账。
- **`connectWorkspace` 的 coalescing 竞态根因**：create 的 summary 落地时先无 cwd，host frame 到达前第二个并发调用会漏掉 reuse 扫描而多铸一个隐藏 blank session——自建并发创建路径必须复现这层合并逻辑。
- **认知状态**：verified_inference。

## 7. 跨 host restart 的 pending 语义（rc.2 README 新增 Known Limitation）

- pending question **不跨 host restart**：registry 持有 tool call 的 resolve/reject，是进程内存；events.mux 重放只覆盖浏览器 reload/重连。
- `events.mux` 的 `since` resume hook 契约上 "unimplemented in v1 (ignored if passed)"——重连恢复 = 重开 stream + 重新拉 history/list，不能假设增量续传。
- **锚点**：`packages/host/apiproxy/README.md`、`packages/host/apiproxy/src/api/events.ts`（约 L57–L59）。

## 陷阱速查

1. 裸包名 import client runtime → 双实例 Symbol 错配，静默失败（§2）。
2. internal 缺席时传 `bareModuleBaseUrl` 以为锚定了安装树 → 参数静默无效（§3）。
3. 依赖官方 HMR 做开发体验 → internal 缺席宿主直接 throw（§3）。
4. 以为 loader 能解析 asar 内模块 → 无官方支持；走 app-owned 安装树 + node-pty 式 env/sidecar 通道（§4）。
5. 以为 app-boot 能枚举激活集 → 只有 entry 级断言；枚举自己建，配置对照用 `--dump-config`（§5）。
6. 并发 create/connect 不复现官方 coalescing → 产生隐藏 blank session（§6）。
7. 把 pending interaction 当持久状态跨 host 重启 → 进程内存，restart 即丢（§7）。
