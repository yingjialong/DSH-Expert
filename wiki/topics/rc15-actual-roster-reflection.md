---
title: 1.5-rc.2 实际清单的反射、Loader 与工具 scope 边界
description: 核验正式可达对象面、隔离标签、const enum 及 schemas 和模型 wire 的区别。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-20
updated: 2026-09-20
asked_by: agent
anchors:
  - vendor/cordis/src/context.ts
  - vendor/cordis/src/reflect.ts
  - vendor/cordis/src/fiber.ts
  - vendor/loader/src/config/tree.ts
  - vendor/loader/src/config/entry.ts
  - packages/preset/agent-presets/src/mount.ts
  - packages/preset/agent-presets/src/index.ts#standingKeyFor
  - packages/core/tools/src/index.ts#schemas
  - packages/core/tools/tests/scoped.spec.ts
  - packages/boot/app-boot/src/index.ts
related:
  - "[[wiki/topics/rc15-public-embedding-migration]]"
---

固定DSH 0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203；companion cordis4.0.2、loader1.0.3、tools/presets0.1.5-rc.2。四个正式包sha512/声明/JS exports核验，远端tag一致，未执行runtime。已完整回传咨询DSH-015-B3-AUDIT-01后沉淀，L2 / verified_inference。

条件：TS宿主 · root/preset/Agent盘点 · 同进程可多runtime · 只读对象面 · 工具registry复用 · 无模型调用 · 构建器行为未实测 · 固定版本。

## Context反射

正式root Context公开reflect类型，reflect声明公开store:Dict<Impl,symbol>；Impl有name/fiber/value?/check?，状态在impl.fiber.state，不是impl.state。可从公开Context对象读取，不要求root named import ReflectService；后者及Impl未被root barrel直接re-export。对象可达不等于直接改写store受支持。

store是runtime各隔离label的provided实现表，应用Object.getOwnPropertySymbols/Reflect.ownKeys读symbol记录，Object.keys/values不能枚举它们。Context.isolate公开，其map将name映射到隔离label。

判断某记录对目标context的realm归属：记录key === context[Context.isolate][impl.name]；root用ctx.root。按name去重会丢多个隔离实现，provider fiber属于root子树也不代表service位于root realm。上游leakedServices使用同一公开symbol比较。

ACTIVE状态和realm归属分开；ctx.get(name)默认strict只取ACTIVE实现的值，是按名查询，不是枚举。值可为undefined，不能仅以get缺值推断无注册。props含accessor/mixin，内建Context属性也不全在store；此表只应称provided implementations。check谓词不必主动调用。

未找到listServices全集API。registry.entries、Loader.entries也不是service枚举替代；热更时读取不构成原子快照。

## Loader与preset树

EntryTree.entries递归当前store及entry.subtree，不只ACTIVE项。Entry.options是配置，fiber可缺席；entry.disabled计算表达式和父禁用，不能只看options.disabled真假。fiber存在不证明ACTIVE；options.name不证明包版本或所有手工plugin子fiber都已枚举。await settle不替代启用entry的activation检查。

独立挂载preset子树可不在root loader.entries。正式presets root公开livePresetMounts(within?:Fiber)，每项presetId/fiber/tree/key?；传root fiber限制当前runtime，否则module-level记录跨同进程runtime。mount.tree.entries读取其配置树；同presetId可保留多代，不应仅按id去重。

standingMountFor(agent.ctx)通过Agent scope parent找已joined standing，不应按Agent fiber父链猜。standingKeyFor会ensureStanding，可能新挂插件，不能当纯只读盘点。以上不证明覆盖任意手工挂载的全runtime清单。

## FiberState

正式cordis4.0.2的const enum：PENDING0、LOADING1、ACTIVE2、FAILED3、DISPOSED4、UNLOADING5。正式根JS没有FiberState导出或同名runtime对象。

不能发射runtime named import/枚举Object.values。可在明确支持const-enum内联的编译路径使用成员，或按固定声明使用数字映射并保持type-only依赖；目标app-boot自身有数值+类型镜像。未验证外部构建器，不保证未来数字不变。

## ToolRuntime视图

正式schemas(scope?:ScopeKey):ToolSchema[]；get(name,scope?)。省略scope永远是该ToolRuntime实例global视图，不因agent.ctx.tools调用而自动选Agent。

root看schemas()；preset传已有mount.key；Agent传agent本身或scopeOf(agent.ctx)。先确认哪一个ToolRuntime service实例：Cordis realm隔离与DSH ScopeKey layering是两条轴。

view以global→远祖→近祖合并同名覆盖，对继承工具应用整链restrictions交集，exact scope own注册最后加入且豁免继承过滤；非native再加入run_code。restrict因此可过滤preset继承工具，不只是global。own工具仍可存在。

schemas是visible registry schema深拷贝，不给own-only来源/provenance，不代表generation pin或输入definition深冻结。PTC时schemas含底层可见能力及run_code，而prompt wireSchemas可折叠成只有run_code。核实际模型工具须看同scope assembly/request，assembly又会运行providers/waterfalls，非纯反射。

## 证据与限制

固定源码绝对路径：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/cordis/src/reflect.ts`：store及隔离解析。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/vendor/loader/src/config/tree.ts`：entries递归范围。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/preset/agent-presets/src/mount.ts`：live mounts/root label检查。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/core/tools/src/index.ts`：view/schemas/wireSchemas。

[cordis正式包](https://registry.npmjs.org/@deepseek-ai%2Fcordis/4.0.2)、[tools正式包](https://registry.npmjs.org/@deepseek-ai%2Fdsh-tools/0.1.5-rc.2)。一方scoped.spec.ts覆盖global/own/ancestor restriction，仅读未运行。未认证具体Profile roster、热更一致性或编译器发射策略。
