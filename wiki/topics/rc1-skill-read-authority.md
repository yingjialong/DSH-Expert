---
title: rc.1 Skill roots、读取与 invocation 边界
description: isolated roots、scope 合并、读前策略、缓存取消及来源一致性的条件。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/core/scope/src/index.ts#createScope
  - packages/core/scope/src/store.ts#ScopedLayers
  - vendor/cordis/src/fiber.ts#Fiber
  - packages/core/agent-loop/src/agent.ts#send
  - packages/core/session/src/invariant.ts
  - packages/llm/llm/src/message.ts#createUserMessage
  - packages/skill/skill/src/index.ts#SkillRegistry
  - packages/skill/skill-filesystem/src/index.ts#FileSystemSkillProvider
  - packages/skill/skill-filesystem/tests/skill-filesystem.spec.ts
  - packages/skill/tool-skill/src/index.ts
  - packages/skill/tool-skill/tests/tool-skill.spec.ts
  - packages/client/ui-skill/src/client/index.ts
  - packages/api/session-controller/src/skill-catalog.ts
  - packages/skill/skill/package.json
  - packages/skill/skill-filesystem/package.json
  - packages/skill/tool-skill/package.json
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-09
updated: 2026-09-09
asked_by: agent
related:
  - "[[wiki/packages/skill]]"
  - "[[wiki/topics/rc2-skill-resources-parser]]"
---

条件：TypeScript Host · 官方 Registry/tool-skill/ui-skill · live/cold Session · filesystem/custom Provider · 正文与资源分开 · 无模型调用 · 来源集合由宿主约束 · 固定 rc.1 `a66e4702047846cdaa10c66c9d3df3951f5ea70d`。

## roots 与访问

- 默认 includeDefaultRoots=true：有 cwd 时 nearest .git project 的 .dsh/skills/.agents/skills，customSkillDirs，dshHome/skills、agentsHome/skills，再可用 bundled root。无 .git 退 cwd；user-dsh 跳 .system。
- isolated 配置是 includeDefaultRoots:false，不加 project/user，也不继承环境 bundled root；仍可显式 customSkillDirs/bundledSkillDir。watch:false 仅关闭 watcher，不关闭 scan/get 或冻结字节。
- 官方构造保存配置/建立 watcher manager；list 解析 roots、observeRoots，再列目录并读整个 Skill Markdown/parse，目录发现本身已读取正文文件。get 重新读取和解析 locator.path。
- watcher 关注直接 Skill 条目/目录/flat md/SKILL.md，不把 scripts/references/assets 内变化绑定到 catalog revision；官方 write/edit 的相关 fs/observed 有快速 invalidation。
- 普通读取可走 ctx.fs；bundled trustedHost 或无 fs 走 Node。没有正文+资源不可变 revision、expected digest/fd/文件树 lease 合同。

## scope、invocation 与缓存

- Registry global→ancestor→own 合并，近层只 shadow 同名，不排除其他名字。没有 selected-set/provider allow/deny 选项；collectFresh 会调用可见各层所有 Provider.list，再决定胜出与 Consumer policy。
- list/snapshot/get 都是 invocation-neutral；model catalog snapshot 后 filter modelInvocable；模型 skill 调用 list 检查 summary 后 get，再复查 loaded policy。用户 slash 则先 get 再检查 userInvocable。disabled surface 不等未访问来源。
- 模型 catalog 仅在 exact skillTool 对 Agent 可见时扫描；限制/遮蔽该工具只跳过此 catalog 路径，不关闭 UI/direct get。
- collect 按 cwd/scope chain/internal revision 缓存；snapshot 也可命中 cache。invalidate 清 cache/发 skills/change，最多重试2次仍变化则 incomplete；complete 不是公开 generation lease。Consumer 对 incomplete 不发布新 authoritative catalog。
- 完整目录变化在下一 pre-step 追加全量 replacement，可为空，不物理删除旧模型历史。get 不缓存正文，policy/body 可与 cached summary 漂移；没有统一模式切换事务。
- ui-skill 调 Session-addressed skills/list。Host 用户目录先 list 再 filter userInvocable；live 可取 preset 内 registry，cold 可解析 standing scope，失败 fallback global。不能默认它与模型路径必然同一 scope/来源。

## 来源禁用、取消与日志

- **条件性结论**：未注册/不在 relevant scope 的 Provider 不由该 Registry 调用；Provider 自己先于 I/O 限定来源也是其实现责任。仅过滤 summary/policy、local shadow 或孤立某个 filesystem Provider，不保证其他继承来源未被访问。
- 所有相关 consumers 的 registry/scope/cwd、Provider 初始化/list/get/watch、资源工具以及 in-flight 生命周期都需满足同一外部前提，才能谈已选集合一致。没有整体内建 selected immutable set/access-control 合同。
- Registry lookup signal 可取消等待，不能强停不合作 Provider。stock filesystem discovery 调 parseSkillFile 时传 undefined signal，get 才传 lookup signal；用户 skills/list Remote 显式 void signal，不传给 Registry.list。
- parser/Registry 无通用 body bytes 上限；目录 description 默认截断500属于 Consumer。Node utf8 非严格非法编码拒绝器；ctx.fs 的 FS_NOT_TEXT 由其 backend 执行。
- Node discovery 可跟 symlink；watchFollowSymlinks 不等 read/get no-follow。无外部一次预检到后续重新 open 的原子绑定。
- 标准 logged channels：skill-catalog user/message、skill-invocation user/message、模型 tool/call/tool/result 的 renderSkillContent。它们保存已公布内容，不存通用 authority 或完整不可变资源树；资源操作由实际工具负责记录。

证据：[roots/list/get](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/skill-filesystem/src/index.ts#L181)、[无 signal discovery](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/skill-filesystem/src/index.ts#L719)、[Registry cache/collect](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/skill/src/index.ts#L518)、[Consumer 策略顺序](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/tool-skill/src/index.ts#L125)、[用户目录](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/skill-catalog.ts#L35)、[cold fallback](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/skill-catalog.ts#L92)。

固定 tag 已核对，skill/filesystem/tool-skill 精确 rc.1 tarball 在内存检查 root types 与 exports；三包为 root、./src/*、./package.json 且无 src 文件，未公开 parser helper。只读 isolated/invocation 测试断言，未运行访问/失效竞态，不评价外部项目。


## Skill source、官方 pre-step 与 Agent 输入路径

同版源码/正式 types 复验，verified_inference；无应用实测。`createUserMessage` 自己生成 MessageId（参数不接受 id），可先创建再传 Agent；身份不等于 SessionSeq。

- `send(message,target,wakeup)` 同步写 inbox splice，按 wakeup 决定唤醒；abort 后 waking 输入有改投 next-turn/收敛规则。
- `followup` 是 next-turn/true；`inject` 是 next-step/false。inject 在 idle 只留 pending，运行时也可能错过本次 claim；不等于立刻写 model-visible user/message。
- claim 先全部 next-step，再按 target 取一条 next-turn；assembly/pre-step 接受后，driver 才在 step/start 后将 decision.messages 逐条追加为 user/message。取消、清除、pre-step reject/改写可使已入 inbox 的消息不成为模型输入。文本入队本身不经过 ToolRuntime；后续真实工具调用才进入工具执行路径。

公开增广分别来自：

| source | 正式 root package | 必需字段 |
| --- | --- | --- |
| SkillInvocationSource | @deepseek-ai/dsh-skill | kind:skill-invocation、name:string、form:instructions |
| SkillCatalogSource | @deepseek-ai/dsh-tool-skill | kind:skill-catalog、form:catalog、entries:{name,description}[]；update?:true |

二者增广 dsh-llm.MessageSourceMap，可作为 UserMessage 经公开 Agent.inject 送入标准输入链；不要求为了加载 TypeScript 增广先激活 plugin fiber。Agent 不替 consumer 查询 Registry/验证 invocation policy/确保 body 与 entries 一致。Controller.prompt 自造 user source，不接受这两种自定义 source；直接 Agent 方法要求 live Agent，不向 cold Session/standing key 发送。source 不携带 standing generation 或 expected seq，Registry read 与 append 没有事务。

官方 slash 与模型 catalog **均不调用 Agent.inject/send/followup**：ui-skill onPick 返回普通 /name 文本；Host tool-skill pre-step listener 对 claimed user 文本 get 后检查 userInvocable，创建 instructions 并扩展 decision.messages。另一 pre-step listener 在 exact skillTool 可见时 snapshot/filter modelInvocable，完整目录变化后扩展/替换 decision.messages。二者由 driver 在当前接受的 step 记录；在 pre-step 另调用 inject 则已晚于当前 claim，不自动补进当前 decision。

Session invariant 的 user/message 分支本身不要求 open step；“默认 driver 在 step/start 后写入”是路径事实，不可升格成直接 Session.append 的普遍约束。直接 append 仍受 identified message、JSON、surface/provenance/序列等规则约束。

## 冷 catalog 的 Registry 选择先于 standing scope

精确顺序是 observeSession→取 cwd/preset→live 可取 presets.serviceFor(live,skills)，否则选 Host ctx.get(skills)→缺 Registry 立即失败→scopeFor→Registry.list。冷读不会先挂 standing 再从其 serviceFor 取一个独立 Registry。scopeFor 成功用 standingKeyFor（可 compose plugins、无 Agent/turn），失败或无 roster 则 undefined/global。live 则用 Agent scope。

该方法明确 void caller signal，observeSession/standingKeyFor/list 均未转交；Registry Provider 虽收 lookup options，但这里不含 caller signal。先 list 后 filter userInvocable，Provider discovery 可以已经读源文件。

证据：[官方 listeners](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/tool-skill/src/index.ts#L163)、[source 类型](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/skill/src/index.ts#L142)、[catalog 类型](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/tool-skill/src/index.ts#L28)、[输入与 driver](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/agent-loop/src/agent.ts#L122)、[cold 顺序](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/api/session-controller/src/skill-catalog.ts#L35)。本地镜像根 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness`，历史证据以 git show 固定 SHA 读取。

正式 rc.1 tarball 内存核验 dsh-skill root types:123/131、dsh-tool-skill root types:16/28 的导出与增广；无安装，不引用发布包未包含的 src 作为公开导入。


## 复用已有 scope key：路由、实例与 owner 分离

条件：同进程、同 scope 模块与同一底层 SkillRegistry，minting Context 有 skills dependency API，持有已有 Agent 的真实对象 key；固定本页 rc.1。静态核验，verified_inference，未验证外部上下文或运行组合。

`createScope(mintingCtx,existingKey)` 不传 parent 时不调用 bindScopeParent，不检查 key 首次使用，也不自动从 mintingCtx scope 推导 parent；既有父链保持。新 ctx 继承 minting Context 的 dependency API，并标记 existingKey。原 Agent.ctx 不因此获得缺失 API；scope 父链也不是 Cordis dependency 链。显式再次 bind 已有 parent 的 key 会失败，只有原 binding 可 rebind。

正常通过新 scope.ctx.skills.registerProvider 登记时，ScopedLayers.effect 用调用 ctx 的 scopeOf 决定 exact-key layer，用 ctx.effect 决定 owner。同一 Registry 上以相同对象 key lookup 会包含该层；不同 Registry 实例各有独立 Map，不会仅因 key 相同共享。Provider 同层名字冲突仍拒绝，list/get 输出与优先级决定实际结果；不要把其他 ctx 取得的 service 或脱离上下文的调用当成同样的 scoped 登记。

新 scope fiber 由 minting fiber 的 ctx.plugin effect 持有，key 指向 Agent 不改变 ownership。scope.dispose 等该 fiber/inertia 收敛；registerProvider undo 撤销其 exact insertion、abort control.signal 并 invalidate cache；ScopedLayers 仅在 layer 全空时删除 key 映射。其他同 key fiber 登记、原 Agent 和原父链均不被成批删除。可见目录可因解除遮蔽而变化，abort 不证明不合作的外部异步工作停止。

若 minting owner 位于 Agent 子树，父子 ownership 自然清理；若 owner 会跨 Agent 存活，则 Agent disposed 不自动释放这个同 key scope，需要已有明确 owner/释放连接才能宣称同寿命。功能 off 也不是 createScope 自带条件。agent/disposed 是非 awaited 通知，异步 cleanup 不因收到该事件就获得完成 barrier。

证据：[scope primitive](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/scope/src/index.ts#L143)、[层 effect/undo](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/core/scope/src/store.ts#L230)、[Provider 登记](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/skill/skill/src/index.ts#L380)、[parent-owned fiber](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/vendor/cordis/src/fiber.ts#L233)。正式 dsh-scope rc.1 root types:43/63/68/78/84 在内存复核公开 bind/dispose/optional parent/createScope/scopeOf；未安装。store.spec.ts:130–172 的 empty-only reclaim 断言仅读未运行。
