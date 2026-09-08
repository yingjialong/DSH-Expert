---
title: rc.1 Loader 与 Host Client 模块生命周期
description: settled 与 ACTIVE、导入边界、Client graph 卸载、保护行与自修改发布的区分。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - vendor/loader/src/config/tree.ts#EntryTree
  - vendor/loader/src/config/entry.ts#Entry
  - vendor/loader/src/config/group.ts#EntryGroup
  - vendor/cordis/src/fiber.ts#Fiber
  - packages/boot/app-boot/src/index.ts#assertEntriesActivated
  - packages/boot/app-boot/src/profile.ts
  - packages/client/modules/src/index.ts#ClientModuleRegistry
  - packages/client/modules/src/client/system.ts#ClientModuleSystem
  - packages/client/modules/tests/node-half.client.spec.ts
  - packages/client/hmr/src/client/index.ts
  - packages/client/web/src/boot.ts
  - packages/bundle/web-app/cordis.patch.yml
  - packages/preset/agent-presets/presets/cordis/agent.cordis.yml
  - packages/extensions/tool-cordis/src/index.ts
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/topics/rc1-connection-client-module-boundaries]]"
  - "[[wiki/topics/plugin-development]]"
---

条件：TypeScript 插件 · 正式 CLI base+web/Profile 与官方 Client · Host/Client 独立 fiber · 合成 artifact 由宿主控制 · 无模型调用 · 本地部署 · rc.1 `a66e4702047846cdaa10c66c9d3df3951f5ea70d`，Cordis4.0.2/Loader1.0.3。

## Loader 操作和观测

- create 等 Entry.update/init/import/start 的当前 lifecycle，再登记树并调用 write；不是 metadata-only 登记。树级 update 类型不收 name，Entry.update 等路径可更换 name；候选 import 可早于旧 fiber dispose。
- 普通 disabled row 在正常 create/update/refresh 路径不启动，禁用 active row 等 dispose；group carrier 有独立语义。显式 loader.import 不受某行 disabled 约束。
- remove 等 entry._dispose，再 unlink/delete；不撤销 module cache 或任意模块顶层外部效果。替换失败有尝试恢复旧插件的路径，不代表外部效果 rollback。
- fiber.await 只等待惯性并抛 startup/config 错误；PENDING 可以是 settled 状态。loader.await 不证明全 ACTIVE；app-boot 另有 assertEntriesActivated。
- fiber.dispose 早期 uid=null/通知，然后等待 owned effect unload。_unload 记录并包含各 disposer 错误，故 Promise resolve 不证明每项资源成功清理，也不涵盖未被 owner 等待的异步工作。
- raw entry.options/entries/resolve 不自行 import；effective entry.disabled 会 evaluate !!js。entry-init 时 options 尚未填好，patch-context 在 import 后；都不是已具 artifact 身份的 pre-evaluation gate。
- fixture 证据应区分 exact Entry/Fiber、等待结果、ACTIVE/公开贡献、disposer 自有结果与最终移除；internal/plugin/status/partial-dispose 单条通知不是完成回执。write 的具体持久性由 tree owner 决定，root write 是 no-op。

## Dual-face 与 Client 卸载

- Host ClientModuleRegistry 扫有 fiber 且非 disabled 的 entry，按真实 manifest.name 归属；有效声明为 dsh.client.platform=web、实际 ./client export（string 或一层 default:string）与构建 bytes，inject/external string arrays、immediately boolean。
- 缺 web 声明的 Host-only 包不入 Client graph；声明 web 但缺 export/bytes 则失败。标准 lazy-CJS 的 script 注册 factory 与 materialize/apply 是不同阶段，不把任意 JS 扩展名当无副作用证明。
- registry 服务 /plugins bytes，借 webserver/index-inject 发布 facade/boot graph；Client modules、AppWebEntry 再建立自身 Loader。module identity、package edges 与 service inject 各有职责。
- ClientModuleSystem.invalidate 只清 factory/cache，不 dispose fiber/styles。官方 client-hmr rebuilt 流程才执行 invalidate→prefetch→旧 registry teardown/drain→删 styles→refresh。
- **Client HMR 忽略 graph frame**，初始 graph 留到 page reload；Host remove/graph 变化不保证现有官方 Client 自动卸载。Client Loader.remove 是本侧原语，不是 Host/Client 跨侧事务或 ACK。

## 结构约束不等产品保护政策

- EntryOptions/Profile 无 protected-row 或同业务职责唯一字段。重复 entry id、同 realm service key 冲突、重复 Client package source 是已有结构检查，不识别不同包的业务职责重叠，也不是 artifact 信任/冻结机制。
- Profile patch 可改/禁/加行，调用方 protected roster/发布策略不能由 loader.await 或 DI 成功反推。
- shipped web-app 启用 cordis-host-runner 与 cordis-client-runner；默认 AgentPreset=standard，专用 tool-cordis 自修改工具行位于 cordis preset。运行支持已在 Host/Client 不等于 standard Agent 默认获得该专用模型工具集，也不证明通过其他能力绝无修改路径。

证据：[tree](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/vendor/loader/src/config/tree.ts#L25)、[disabled/import](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/vendor/loader/src/config/entry.ts#L61)、[fiber unload/await](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/vendor/cordis/src/fiber.ts#L675)、[Host registry](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/modules/src/index.ts#L738)、[HMR graph 忽略](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/client/hmr/src/client/index.ts#L103)、[自修改 preset](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/preset/agent-presets/presets/cordis/agent.cordis.yml#L241)。

固定远端 tag 已核对，正式 Cordis4.0.2/Loader1.0.3 root declarations 在内存只读核对 create/update/remove/await/Fiber.dispose；Client 模块正式面复用同版既有核验并回源。未运行双侧 fixture，不报告完整 mount/dispose E2E 通过，不评价外部项目。
