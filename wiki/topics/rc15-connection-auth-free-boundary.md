---
title: 1.5-rc.2 Connection 无鉴权工厂缺口与 Handle 替代合同
description: 核验无 Web 场景的 BrowserAuth 初始化、注册生命周期及公开 carrier 边界。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-19
updated: 2026-09-19
asked_by: agent
anchors:
  - packages/client/connection/src/index.ts#apply
  - packages/client/connection/src/browser-auth.ts#BrowserAuth.create
  - packages/client/connection/src/rpc.ts#HostConnectionHandle
  - packages/client/connection/src/rpc-host.ts#HostConnectionService
  - packages/api/gateway/src/index.ts#wireStream
  - packages/api/remotes/src/index.ts#apply
  - packages/api/session-controller/src/media-references.ts
  - packages/client/connection/tests/fetch-routes.host.spec.ts
  - packages/client/connection/tests/node-half.host.spec.ts
related:
  - "[[wiki/topics/rc15-public-embedding-migration]]"
---

固定0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203。远端tag一致；connection/gateway/remotes/session-controller四个正式包sha512、exports、实际成员、声明及connection根JS核验。仅读测试，未运行非Web整图；先完整回传成功再沉淀。L2 / verified_inference。

条件（八维坐标）：TS Host · 无HTTP/Web前端的原生carrier · 注册与generation各有owner · 不允许BrowserAuth长期secret · 保留官方Gateway/Remote/controllers · 外部认证前提不由本页证明 · 平台运行未验 · 固定目标。

## 无公开auth-free factory

官方apply在webServer可选分支之外await BrowserAuth.create(ctx.root,ctx.credentials,...)；该过程modifyRecord读/建signing grant并在实例持有secret，另有process token。无webServer不会跳过。

公开Config只有recovery/trustedHosts/cookieMaxAgeDays/maxRequestBodyBytes，没有disableAuth或registry-only开关。HostConnectionService公开构造器仍要求内部BrowserAuth；该类不是root export。tarball无src，./src/*声明不构成内部helper可导入证据。

fetch-routes测试用内部import及{} as BrowserAuth只是fixture，不是可建议的正式无认证入口。未找到另一个已发布公开factory能复用官方registries而不建auth。

## 已核消费者

- Gateway在connection注入中注册rpc.intercept('/api',matcher,dispatch)，未保存disposer，依赖caller Fiber自动释放。只有connection+webServer分支才启动WebSocket upgrade并requestRejection。
- Gateway公开wireStream.open/failure供local carrier；HostConnectionHandle没有Host openStream方法，完成rpc/fetch不代表stream/$events已接通。
- api-remotes直接消费typertGateway.registerRemoteEvents，不依赖BrowserAuth或concrete Connection。
- SessionMediaReferences及相关upload/export用公开fetch.register；其handler假定物理入口已经认证。
- 所核范围未发现instanceof HostConnectionService或私有auth字段依赖。按Handle提供唯一out-of-tree provider是条件性可组合结论，非完整运行验收。

## 注册与分派必须兑现

1. rpc.handle注册专用绝对channel，不能占用/api。官方具体实现内部调用owner.webServer.register，因此不能把该concrete方法称为天然无Web实现。
2. rpc.intercept只允许/api，官方仅一个interceptor，重复抛错；matcher不拥有的endpoint不能被吞成成功，也不是多策略拦截器链。
3. fetch.register按exact path+methods，路径须在/api下，methods非空无重复；同path不能以另一methods集合重复注册。query保留，未匹配method继续共享分派。
4. 官方rpc/fetch getter捕获访问service的scoped this.ctx，再owner.effect注册。外部provider必须维护caller Context/Fiber归属、显式异步disposer和自动卸载，不能全绑root。撤回注册不自动证明所有外部物理效果已drain。
5. createSharedFetchHandler返回requestBodyMode/fetch；在读body前按exact path+method选mode。exact优先，再matcher interceptor，未拥有404。buffered有JSON cap，streaming背压且无该aggregate cap，不能先全量buffer冒充stream。
6. shared fetch合同是already-authenticated，函数本身不执行requestRejection。native入口仍需真实authority，不因调用shared handler而自动安全。

## 无Web的fail-closed边界

以下为自定义provider的条件性“不支持”表现，不是上游现成disabled实现：requestRejection可返回401/403而非undefined；authorizeIndex不可放行，false时须兑现response归它结束的合同；authenticatedUrl无exchange能力应显式抛unsupported，不造假认证URL。

前提是未使用Web能力。以后装frontend-static/webServer或调用authenticatedUrl，便不再等价官方Web provider。物理carrier的身份/代次校验是否充分，未在本页核验。

## 私有parser与公共wire

rpcFetchHandler/endpointFromPath/http-bridge未公开，所核consumer也不直接导入它们。公开root有ClientRequest/ServerResponse类型与schemas，可用于校验；没有公开“全部wire处理已完成”的factory。

承载官方Fetch wire仍需POST/application-json、JSON解析、type/rpcId/method/payload/result、method与endpoint一致性、correlation、signal及错误/404语义；直接投递decoded handler则需外部先验证native wire和权限。二者均不能凭结构类型跳过信任边界。wireStream也不自动做物理认证。

## 证据与限制

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/connection/src/index.ts`：无条件auth初始化。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/connection/src/rpc-host.ts`：effect owner、路由与内部parser。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/api/gateway/src/index.ts`：无Web分支及wireStream。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/client/connection/tests/fetch-routes.host.spec.ts`：method/mode、重复注册及disposer后404。

正式包：[connection](https://registry.npmjs.org/@deepseek-ai%2Fdsh-client-connection/0.1.5-rc.2)、[gateway](https://registry.npmjs.org/@deepseek-ai%2Fdsh-api-gateway/0.1.5-rc.2)。没有执行auth-free out-of-tree provider完整boot、stream或权限验收，也不以内部测试cast证明公共支持。
