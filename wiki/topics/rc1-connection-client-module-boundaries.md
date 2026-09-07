---
title: rc.1 Connection Host 替换与官方 Client 模块身份
description: 区分 HostConnectionHandle、默认认证实现、Client 载体 hook 与官方模块入图的公开边界。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/client/connection/package.json
  - packages/client/connection/src/index.ts#apply
  - packages/client/connection/src/rpc.ts#HostConnectionHandle
  - packages/client/connection/src/rpc-host.ts#HostConnectionService
  - packages/client/connection/src/browser-auth.ts#BrowserAuth
  - packages/client/connection/src/client/index.ts#ClientTransportHooks
  - packages/client/modules/package.json
  - packages/client/modules/src/index.ts#ClientModuleRegistry
  - packages/client/modules/src/client/index.ts#createClientModuleSystem
  - packages/client/modules/src/client/manifest.ts#WebBootEntry
  - packages/client/modules/tests/node-half.client.spec.ts
  - packages/client/web/src/boot.ts#AppWebEntry
  - packages/boot/app-boot/src/profile.ts#DshProfileManifest
  - vendor/loader/src/config/entry.ts#EntryOptions
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/packages/client]]"
  - "[[wiki/integration/rc2-full-web-native-shell]]"
---

# rc.1 Connection Host 替换与官方 Client 模块身份

条件：TypeScript 宿主 · 完整 CLI/Profile Host 与官方 Web Client · 单一 Host Connection Provider · 文件系统和沙箱由宿主管理 · 不改变模型工具 · 认证需自有实现 · 本地部署 · 固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。

## 可复用结论

- `HostConnectionHandle` 是公开可实现的服务形状；满足它只解决 Host DI，不会自动登记官方 Connection Client module。
- 官方 `ClientModuleRegistry` 从有 fiber 且非 disabled 的 Host Loader entries 扫描，以实际解析模块所属 manifest.name 为 Client 身份，读取该包 `dsh.client` 和 `./client`。官方 Host row 停用且无其他有效官方包来源时，不会因另一个 Provider 提供 `ctx.connection` 而恢复官方 Client row。
- `EntryOptions` 无 client-only/host-alias 字段；Profile 提供 bundle/patch 组合，仍落到 Host entries。Connection 正式包没有独立 Definition-only/no-op Host export。类型声明可见不构成可挂载模块。
- 裸包名与末尾 `/client` 的 alias 是同一浏览器模块的身份别名，不是替换 Host face 的接缝。file/path entry 仍受真实 manifest 身份约束；同一包来自多个不同 active Loader sources 会报 composition error。
- `ClientModuleRegistry` 公开观察、artifact rebuild 与通知方法，但没有独立 client-row 注册或 alias API；其 table、扫描、reconcile、compose 为 private。不要修改 `graph()` 返回对象冒充正式贡献合同。
- rc.1 `WebBootEntry.inject` 是 package-name factory-arrival edges，`external` 是精确 module requests；Cordis service inject 另行决定激活。不能套用旧版“manifest inject 仅展示”的描述。

## 公开构造函数不等于公开可构造闭包

- `HostConnectionService` 从正式 root 导出，但构造参数 `browserAuth: BrowserAuth` 引用非公开导出的 nominal class；后者有 private 状态与 private constructor。内部 `.d.ts` 存在不等于提供合法 runtime import。
- 默认 `apply` 注入 `webServer` 与 `credentials`，调用内部 `BrowserAuth.create` 再构造 service。公开 `ConnectionConfig` 只有 `trustedHosts`、`cookieMaxAgeDays`、`maxRequestBodyBytes`，无认证适配入口。
- root 公开 RPC envelope/result 类型及 schema、`RpcId`、`transportError`；因此不能说没有公开 wire 合同。但完整 Host `rpcFetchHandler`、HTTP bridge、trust helper 未作为独立公共 codec factory 导出。实例 `createSharedFetchHandler` 不解除 service 的构造要求。
- 正式 `/client` 公开 `ClientTransportHooks`（`fetch`，可选 `openStream/loadBundle/ownsHost`）。官方 Client `apply` 消费 `__DSH_TRANSPORT__`，可保留内部 codec、替换载体；内部 `createWebConnectionRpc` 没有从公开 barrel 导出。此 hook 不解决官方 Client row 的 Host-side 入图问题。

## 不应越过的结论边界

`WebBootGraph`、`bootInjections`、`ClientModuleSystem`、`createClientModuleSystem` 是公开 bootstrap 原语；`AppWebEntry` 读取全局 graph/facade，`BootSeams` 只提供 `loadBundle`。不能由此断言任何自建 bootstrap 都不可能，但也没有证据把手工 graph/bootstrap 组合认证为官方 Profile 下独立 client-only contribution 的完整方案。

本次未找到在“唯一自有 Host Provider + 保留官方 Client 完整模块身份 + 无私有导入/monkey patch/复制内部 codec”条件下已闭合的官方装配方案。这是限定公开面的负向核验，不是任意自建 shell 的不可能性证明。未执行完整启动实测；状态为 `verified_inference`，不提升 L3。

## 证据与发布检查

源码及测试均按上列固定 SHA 读取：[registry 扫描](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/modules/src/index.ts#L902)、[身份解析](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/modules/src/index.ts#L786)、[构造器](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/connection/src/rpc-host.ts#L59)、[Client hook](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/connection/src/client/index.ts#L76)。

精确 rc.1 npm tarball 在内存中只读检查：两包均无 `src/`；connection 的 `lib/types/index.d.ts`、`rpc-host.d.ts`、`browser-auth.d.ts`、`client/index.d.ts` 与 root JS exports 确认上述边界；modules root `.d.ts` 与 JS 确认 registry 无公开 row-registration API。包内路径是产物逻辑路径，并非本机安装路径。

- connection integrity：`sha512-30g7JG+Z8bDj+ux+KsDfMSXPI6zXotMAFNCxI8O8mJiutJNlZxIL5LXQeDrJpVOW9MWWQkoMp+R+Dxzp5DNDSQ==`
- modules integrity：`sha512-neStnvWlar+qmADi32QBIJ3m0gmWEk1U0YT9WrTKEfkwQiooPU8lJQjXNfv93x6dgtqieCUhMtUR2a4925R38Q==`

上游身份/duplicate-source 测试作为断言证据读取，未执行。上游 auth 测试私有导入 BrowserAuth，fetch-route 测试使用类型断言，不能当作外部包的公共构造示例。
