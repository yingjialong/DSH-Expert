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
updated: 2026-09-16
asked_by: agent
related:
  - "[[wiki/packages/client]]"
  - "[[wiki/integration/rc2-full-web-native-shell]]"
---

# rc.1 Connection Host 替换与官方 Client 模块身份

条件：TypeScript 宿主 · 完整 CLI/Profile Host 与官方 Web Client · 单一 Host Connection Provider · 文件系统和沙箱由宿主管理 · 不改变模型工具 · 认证需自有实现 · 本地部署 · 固定 `0.1.2-rc.1` / `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。

## 可复用结论

### Settings policy 与 Loader/Client staging 边界

补正：SettingsProvider正式protected方法名为publish(doc,source?)，非publishDocument；后者是此前答复误称。FileSettingsProvider子类可override protected persist/publish并super，无需private import。persist收到完整namespace raw user候选（非全doc、非含base/default的resolved值），普通write在persist前校验revision/schema，成功后直接document/commit，并不调用publish。

File super.persist文件队列/锁内先reconcileFromDisk再写候选，未在该reconcile后对原候选重做revision CAS/合并。reconcile与initial load先更新private text cache再调用publish；子类拒绝publish能阻止基类发布，但不自动回滚text缓存，相同text下次可能skip。super.publish先替raw document，再逐namespace解析，失败namespace保留旧resolved值而不bump其revision；不是全doc回滚。owner在persist中被替换后的resync源码有TODO，不作更强保证。

依据：固定SHA的 settings/src/index.ts#publish/write 与 settings-file/src/index.ts#load/persist/reconcileFromDisk；正式两包lib/types/index.d.ts已核对protected签名。未运行子类/存储竞争PoC。

rc.1 SettingsProvider.register的validate(value):void为namespace owner同步resolved-value校验，update/replace/mutate统一队列在persist前调用；它不携caller/原始ops/授权receipt，重复ns注册拒绝。Controller限制不覆盖in-process Provider/namespace scope写。revision在队首比较，readonly约束同样覆盖in-process；更新通知是提交后观察。

公开Include/EntryTree支持独立path子树，挂到entry.subtree后Loader.entries递归可见。ClientModuleRegistry收集该Loader全部active非disabled的web Client包，不按isolate或path过滤；graph()无context/entry筛选。服务隔离不是无副作用staging或graph quarantine。

ClientModuleSystem的prefetch是factory arrival，import才materialize，invalidate仅清factory/cache，不dispose已运行Cordis fiber。public bootstrap原语可构造Client系统，但没有现成stage/validate/activate事务来保证试装不进入活跃UI/Session；完整独立闭包未实测，不说任意独立Context不可能。

证据：固定SHA的 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/settings/settings/src/index.ts:48`、`:632`；`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/loader/src/config/tree.ts:16`；`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/modules/src/index.ts:598` 与客户端system.ts:240。

### patch 身份与 Client factory 到达分层

rc.1 composeEntries调用vendor/include的applyEntryPatches：name是匹配断言，mismatch整项warn/skip，不写入target.name；patch无delete/remove操作。disabled旧row+insert不同唯一id的新row可表达停用/新增，不是删除后同id替换；Loader.update也排除id/name。不能把未知delete字段视为生效。

ClientBundleRegistration公开{id,factory}与window.__ModuleLoader__.load是浏览器factory到达机制，不创建Host graph row，也不直接激活Client plugin。Host扫描active非disabled的实际package身份；新包子类提供同service不能自动保留原npm Client row。没有公开Host registerClientRow/host-alias字段。public bootstrap原语存在，故“未找到标准扫描下自动映射”不等于自有bootstrap不可能；完整唯一Host子类+官方Client闭包未经本次PoC验证。

依据固定SHA的 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/include/src/index.ts:58`、`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/boot/app-boot/src/profile.ts:854`、ClientModules公开manifest与system源码。对SessionController与Connection的Host身份问题同样须按实际包逐项核对。

### transport 首读与 Fetch/stream 合同（固定 rc.1 补充）

hooks 本身使用页面 global，不要求另建 Client module 注册；但官方 Connection Client 模块必须已装配。“index inject 后”不充分，必须确认早于其 apply 首读。直接 wireStream.open 不自动应用浏览器认证。

JSON-lines 可作为自定义 carrier 的条件性 framing，而非官方已认证 transport：须保持实际 wire 值、顺序、正常结束、错误与取消区别；Host open 返回 Promise<AsyncIterable>，Client hook 返回 AsyncIterable。不能把异常包装为普通业务 yield，也不能将 unknown 类型解释为任意 JS 对象皆可 JSON 无损转换。本次未运行 JSONL PoC。

requestRejection 先 Host/Origin trust（403）后 cookie auth（401）；authorizeIndex 拥有 token exchange/redirect/401，true 才继续 index。二者不是可插拔 predicate 参数，不替代 Gateway 参数/context 或业务权限校验。自定义载体策略不因调用公开 handler 而自动等价于官方认证。

Client apply 在创建 rpc 前读取 globalThis.__DSH_TRANSPORT__；fetch/openStream 被闭包捕获，必须在首次 apply 前设置，且 fixture 模式优先 fixtureRpc。Web boot 在 boot-ready 后、moduleLoader.create 前读取 loadBundle；显式 BootSeams 优先，先前的 HTML/bootstrap 脚本不由该 hook 追溯接管。

fetch 接收 URL/RequestInit，返回标准 Response；官方 caller 继续拥有 rpcId/envelope 校验。openStream 接收 endpoint/payload/signal，返回 AsyncIterable 逻辑值；Host Gateway wireStream.open 返回 Promise<AsyncIterable>，failure 归一化错误。未提供 openStream 仍走默认 WebSocket。

Host createSharedFetchHandler('/api').fetch(Request) 仅负责 dispatch 已认证请求，不执行 requestRejection。官方 HTTP path 先认证再进入内部 http-bridge；bridge 不是 root export。该内部实现按 body cap 构造 Request、附 AbortSignal、转发 Response 并处理背压；异常 res close（!writableEnded）abort，而非用 IncomingMessage close。本地 carrier 的取消仍须传递 signal/iterator teardown。

默认 module loader 使用 script.src，未见统一拒绝非 http scheme；RPC origin 为 null 时 fallback http://dsh.internal，默认 stream 将 scheme 转为 ws/wss。因此 hook 可替换载体不等于 WK 自定义 scheme 已验证兼容。WKURLSchemeHandler 的完整 HTML/CSP/origin/cookie 行为未实测，保留 UNKNOWN。

证据：`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/connection/src/client/index.ts`、`src/client/rpc.ts`、`src/rpc.ts`、`src/http-bridge.ts`；`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/gateway/src/types.ts`、`src/index.ts`；`/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/web/src/boot.ts` 与 `packages/client/modules/src/client/system.ts`。以上简称路径均相对其明确包目录，按固定 SHA 读取；无原生端到端实测。

- `HostConnectionHandle` 是公开可实现的服务形状；满足它只解决 Host DI，不会自动登记官方 Connection Client module。
- 官方 `ClientModuleRegistry` 从有 fiber 且非 disabled 的 Host Loader entries 扫描，以实际解析模块所属 manifest.name 为 Client 身份，读取该包 `dsh.client` 和 `./client`。官方 Host row 停用且无其他有效官方包来源时，不会因另一个 Provider 提供 `ctx.connection` 而恢复官方 Client row。
- `EntryOptions` 无 client-only/host-alias 字段；Profile 提供 bundle/patch 组合，仍落到 Host entries。Connection 正式包没有独立 Definition-only/no-op Host export。类型声明可见不构成可挂载模块。
- 裸包名与末尾 `/client` 的 alias 是同一浏览器模块的身份别名，不是替换 Host face 的接缝。file/path entry 仍受真实 manifest 身份约束；同一包来自多个不同 active Loader sources 会报 composition error。
- `ClientModuleRegistry` 公开观察、artifact rebuild 与通知方法，但没有独立 client-row 注册或 alias API；其 table、扫描、reconcile、compose 为 private。不要修改 `graph()` 返回对象冒充正式贡献合同。
- rc.1 `WebBootEntry.inject` 是 package-name factory-arrival edges，`external` 是精确 module requests；Cordis service inject 另行决定激活。不能套用旧版“manifest inject 仅展示”的描述。

## 公开构造函数不等于公开可构造闭包

### 官方 Connection 激活与浏览器认证 record（2026-09-16 复核）

固定本页 rc.1 SHA，保留官方 Host Connection row、替换 CredentialProvider 的条件下：apply 在配置与图片 body capacity 检查通过后，无条件 await BrowserAuth.create → initializeSecret → modifyRecord(credentialKey('client-connection','browser-session'), mutate)，成功后才注册 /api。它不依赖业务 Session 或 LLM provider 创建。

不存在的 record 写为 `kind: grant`、`payload: {version:1, secret:<32字节随机值的base64url>}`；现有合法 record 校验后 mutate 返回 undefined，表示保留，不是删除或每次旋转。错误格式直接失败，不覆盖。Provider 丢弃写入且返回 undefined 时，初始化报 `browser-session credential record was not created`。

该 record 属于 Connection cookie 签名认证，不是用户模型凭据播种；process launch token 另由 root-owner WeakMap 管理。公开 ConnectionConfig 仅 trustedHosts/cookieMaxAgeDays/maxRequestBodyBytes，无关闭认证、跳过 record 或替换 BrowserAuth factory 的配置；loopback 不豁免初始化。

BrowserAuth 每 activation 加载一次 secret，后续验证使用内存值；删除 record 不立即撤销当前 activation 的 cookie，下一次 activation 才重建并拒绝旧 cookie。这是该 consumer 的特定行为，不把通用凭据 per-operation resolve 要求泛化成 BrowserAuth 已实现热旋转。

证据：固定 SHA 的 [apply/config](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/connection/src/index.ts) 66–127 行；[initializeSecret/BrowserAuth](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/connection/src/browser-auth.ts) 161–216 行；[一方测试](/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/connection/tests/browser-auth.host.spec.ts) 210–252 行明确验证 modifies 次数、删除与下次 activation。此次只读源码及测试，未运行完整 Host 或模型请求；verified_inference。具体自定义 Provider 的持久化实现未知。

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
