---
title: packages/host — Web GUI 的 host 半（API gateway + HTTP 承载 + 目录选择 seam）
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/host/README.md
  - packages/host/apiproxy/README.md
  - packages/host/apiproxy/src/api-proxy.ts
  - packages/host/webserver/README.md
  - packages/host/webserver/src/index.ts
  - packages/host/frontend-static/README.md
  - packages/host/directory-picker/README.md
  - packages/host/directory-picker/src/index.ts
  - packages/host/directory-picker-auto/README.md
  - packages/host/directory-picker-native/README.md
  - packages/host/directory-picker-browse/README.md
  - packages/host/plugin-inventory/README.md
  - docs/subsystems/web-server.md
  - docs/subsystems/workspace.md
  - docs/capability-seams.md
  - .agents/notes/implemented/architecture/2026-07-19-gui-layering-and-rpc-protocol.md
  - .agents/notes/implemented/architecture/2026-07-28-directory-picker-capability-seam.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

dsh Web GUI 的 host 侧：所有 client 形态共用的 API gateway（`ctx.apiProxy`）、它骑乘的裸 HTTP 服务器（`ctx.webServer`）、SPA dist 兜底、workspace 目录选择 seam，以及一个只读的 Loader 插件清单投影。浏览器侧在 `packages/client/`。

## 稳定性

`Product — stable API`（`packages/README.md` 表格原文，组 README 亦称全组 product）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `apiproxy/` | `@deepseek-ai/dsh-host-apiproxy` | 共享 API gateway 与线协议契约，提供 `ctx.apiProxy`；**不注册任何路由** |
| `webserver/` | `@deepseek-ai/dsh-host-webserver` | `node:http` 路由/upgrade 承载，提供 `ctx.webServer` |
| `frontend-static/` | `@deepseek-ai/dsh-host-frontend-static` | 占据 webserver 唯一 fallback 座位的 SPA dist 服务器 |
| `directory-picker/` | `@deepseek-ai/dsh-host-directory-picker` | **Service Definition**：`ctx.directoryPicker` 目录选择 seam |
| `directory-picker-native/` | `@deepseek-ai/dsh-host-directory-picker-native` | `kind: 'native'` 后端：在 host 显示器上开一个原生选择器 |
| `directory-picker-browse/` | `@deepseek-ai/dsh-host-directory-picker-browse` | `kind: 'browse'` 后端：应用内目录浏览的 list/create 原语 |
| `directory-picker-auto/` | `@deepseek-ai/dsh-host-directory-picker-auto` | boot 时采样一次宿主环境，挂上匹配的那个后端行 |
| `plugin-inventory/` | `@deepseek-ai/dsh-host-plugin-inventory` | 当前 Loader 条目的只读投影，发布 Remote `pluginInventory/list` |

注意目录名与 npm 名的映射规律：本组一律加 `host-` 前缀（`apiproxy` → `dsh-host-apiproxy`）。

## 三件套结构

本组里**只有目录选择器是真 seam**（`docs/capability-seams.md` 中 `ctx.directoryPicker` 的 Role 是 `seam`；`ctx.webServer`、`ctx.apiProxy` 都是 `core`）：

- **Service Definition**：`@deepseek-ai/dsh-host-directory-picker`。抽象 `DirectoryPicker` 服务，**唯一方法是 `capability()`**，返回一个判别联合描述「操作者怎么选目录」。
- **Service Provider**：`-native`（`{ kind: 'native', pick(signal) }`）与 `-browse`（`{ kind: 'browse', list(path?), createDirectory(path, name) }`）。两者**在用户交互形态上就不同**，不只是实现不同。
- **Consumer**：`apiproxy`（把 `host.pickDirectory` / `host.listDirectory` / `host.createDirectory` 委派下去，并把后端的 typed failure 1:1 映射成 wire error code）。
- **组合器**：`-auto` 不是第三个后端，它是一个 node-half-only 插件，boot 时把选中的后端**作为真实 Loader entry 挂进内存根树**。

其余包是「服务 + 消费者」直连：`frontend-static` 消费 `ctx.webServer` 的单一 fallback 座位；`connection`/`modules`/`hmr`（在 `packages/client/`）注册各自的路由。

## 扩展点

- **加一种新的目录选择交互** → 依赖 `@deepseek-ai/dsh-host-directory-picker`（Service Definition），通过**声明合并**往 merge-extensible 的 `DirectoryPickerCapabilities` map 里加自己的 variant，然后注册 `ctx.directoryPicker`。消费者 `switch (capability().kind)`，遇到未知 kind 应当**隐藏目录选择而不是报错**。
- **`capability()` 返回的对象在整个 service 生命周期内必须稳定**——这是 seam 的硬性契约（`-auto` 之所以「每 boot 只采样一次」就是为了满足它）。
- **每个后端包还有一个 browser entrypoint**，向 ui-workspace 的 directory-flow slot 注册配套交互，所以**一行 composition 同时选定 host capability 和 client flow**。
- **加 HTTP 路由** → 依赖 `ctx.webServer`：`register(route)` 加具名 `exact`/`prefix` 路由，`registerUpgrade(route)` 加精确 pathname 的 upgrade 路由，两者都返回 disposer；`registerFallback(handler)` 是**单所有者**座位（第二次注册抛异常）；`tapIndex(transform)` 加 index.html 变换。HTTP 匹配顺序**固定**：全表 exact → 最长 prefix → fallback。
- **`apiproxy` 是 transport-independent 的**：它自己不注册路由，carrier（如 HTTP）自行包裹 `ctx.apiProxy`。`AbstractApiClient` 持有全部协议不变式（rpcId 铸造、信封包装/拆解、zod 解析、SSE 帧解码、unary 超时、microtask 批量的 `subscribeEnvelopes`），平台子类只提供 `doFetch`。`InProcessApiClient` over `toFetchHandler(api)` 是「不走网络但走完整线序列化/校验」的同构点。
- **加一个可被浏览器配置的插件** → 注册自己的 settings 命名空间即可，`apiproxy` 的 `settings.*` 域**无需改动**就会服务它（「仓库外分发的插件也能变成浏览器可配置」是明写的设计目标）。

## Known Limitations

组 README 无该章节；以下按包汇总。

**apiproxy**（gateway，限制最多）
- 转发的 Remote 事件**寄生在遗留的 `HostFrame` union 上**（`host/remote-event`），读起来像是本包拥有 Remote 事件契约——其实不是（allowlist 属于 `dsh-api-remotes`，消费动词是 `ctx.remote.$on`）。
- pending interaction 状态在 host 侧；`src/api-proxy.ts` 的表**只处理 question，没有 approval 条目**。
- **无协议版本字段**：client 与 host 同版本发布，`host.describe` 只有在出现独立发布的 client 时才会加版本协商字段。
- 搜索失败会带上 provider 诊断信息——gateway 假定是单用户本地服务，多用户 carrier 必须替换成公开安全的诊断。
- cold-list 提示只会向「可见性更高、排序更旧」的方向退化。

**webserver**
- **无 TLS、无 auth、无 origin 策略**。绑非 loopback 地址就等于把服务暴露给那个网络；加固或反代明确不在 dev-facing v1 范围内。
- socket 选项固定（config 只能选 bind host 与 port）。

**directory-picker 家族**
- Service Definition：**不支持多根**（browse 契约每次列举只暴露一条 ancestry 链）。
- `-native`：Linux 需要桌面工具（Zenity 或 KDialog），两者都没有时 `pick` 直接以可操作错误拒绝，**不会退化成手输路径**；Windows 无机制兜底。
- `-browse`：不读 Windows hidden 属性（`hidden` 在所有平台都指 dot 前缀）；无盘符根枚举；**全文件系统范围**，没有 per-deployment browse root。
- `-auto`：检测是从启动上下文推断操作者位置，**没有任何启动侧信号能证明这件事**——tmux 从 SSH 启动后 detach 会丢 `SSH_*` 标记；本机启动后用 `ssh -L` 访问会从 `127.0.0.1` 到达、解析成 `native`，然后在无人值守的工作站上弹出选择器。Linux 探测**只读 `PATH`**。只在 boot 时解析一次。

**frontend-static**：起步 MIME 表极简，只覆盖 Vite 产出的资产集加 PWA manifest，其余扩展名一律 `application/octet-stream`。

**plugin-inventory**：**只有时点状态**（无持久失败历史、无订阅，没有 live root Fiber 就报 `null`，不区分原因）；**无来源也无变更能力**（不知道条目是哪个 bundle/profile/override 引入的，也不能启用/禁用/增删插件）。

## 陷阱

1. **`webserver` 的 `host` 只接受两个值**：`127.0.0.1`（默认姿态）与 `0.0.0.0`（刻意的网络暴露）。listen 失败（EADDRINUSE 等）会抛出激活并**让 Loader 组合失败**，失败的候选 fiber 被 dispose。
2. **`webserver` 什么都不打印**——URL 那行归 shell 输出。它也不认识任何 harness 概念、不服务任何文件。
3. **`-auto` 与某个具体后端行同时挂载会 fail loud**（`directoryPicker` 服务重复 + `single` 洞里的 client flow 重复）。想固定某种交互就**直接组合 `-native` 或 `-browse` 行**，那才是 seam 文档化的替换点；`-auto` 里没有「pin 一个 kind」的 config 字段。
4. **每个 `/api` POST 都必须声明 `application/json`**，否则在 dispatch 之前就被 415 拒——这是为了让浏览器无需 CORS preflight 就能发出的「simple request」永远不可能盲打一个有副作用的方法。
5. **一整套方法被钉死在 loopback + same-origin**：`host.pickDirectory`、`host.openPath`、整个配置平面（`settings.describe`/`openDocument`/`update`/`replace`/`mutate`，`credentials.describe`/`set`/`unset`）以及 `agentPreset.read`/`copy`/`openDocument`/`remove`。围栏在浏览器 carrier `dsh-client-connection` 里。
6. **secret 只朝一个方向过线**：只能出现在 `settings.update`/`mutate`、`credentials.set`、`llm.discoverModels` 三种载荷内部，**任何响应的任何层都不返回 secret 值**。但它们确实会骑在 client 的出站信封上，`subscribeEnvelopes()` 的观察者能看到。
7. **`session.export` 不是 RPC**，是 host-only 下载面：`GET /api/session.export?...` 流式吐 ZIP，内容是 persistence backend `readRaw` 的**逐字原始字节**，绝非从已解析事件重建。
8. **`plugin-inventory` 刻意不声明同进程 Cordis `Context` 合并**——它是 Remote-only 的，client 包必须经 `api-remotes` assembly 消费，不能 import Host 实现。因此它**不出现在 `docs/capability-seams.md` 的 ctx key 表里**，别以为是漏了。
9. **`agentPreset.select` 只在 session 还是 blank 时允许**：跑过一个 turn 之后换 preset 会让已记录的 tool call 悬空，因此返回 `agent-preset-locked`。
10. **`command.execute` 只带调用方/连接取消，不受 30 秒传输健康截止约束**——command handler 合法地可以活得更久。

## 去哪深入（文件路由）

| 问题 | 去哪 |
|---|---|
| 组内 8 个包与 ctx key | `packages/host/README.md` |
| 每个 RPC 域（session/workspace/settings/credentials/llm/agentPreset/command/skill）的确切语义与错误码 | `packages/host/apiproxy/README.md`（**很长，按域查**） |
| 四象限线消息 union、zod 两级解析、`RpcMethodMap` | 同上 `## Contract layer (/api)`；实现在 `src/api-proxy.ts` |
| 路由匹配顺序、fallback 单所有者、dispose 语义 | `packages/host/webserver/README.md`、`docs/subsystems/web-server.md` |
| picker seam 的判别联合、`DirectoryPickerError`、`DirectoryListing.crumbs` | `packages/host/directory-picker/README.md`、`docs/subsystems/workspace.md` |
| `-auto` 判定 `native` 需要的全部信号 | `packages/host/directory-picker-auto/README.md` |
| GUI 分层与 RPC 协议的原始 RFC | `.agents/notes/implemented/architecture/2026-07-19-gui-layering-and-rpc-protocol.md` |
| picker seam 决策 | `.agents/notes/implemented/architecture/2026-07-28-directory-picker-capability-seam.md` |
| 浏览器半 | `packages/client/README.md`；组合样例 `packages/bundle/web-app/cordis.patch.yml` |
