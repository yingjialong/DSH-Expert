---
title: rc.1 Session 清单完整性与 Preset 代际枚举
description: 区分 logical corpus、磁盘可见记录、discovered definitions 与仍安装的 Preset generations。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/session-query/session-query/src/corpus.ts#SessionCorpus
  - packages/session/session-persistence-jsonl/src/index.ts#listArtifacts
  - packages/session/session-persistence-jsonl/src/format.ts#parseHeader
  - packages/preset/agent-presets/src/discovery.ts#discoverPresets
  - packages/preset/agent-presets/src/index.ts#ensureStanding
  - packages/preset/agent-presets/src/mount.ts#livePresetMounts
commit: a66e4702047846cdaa10c66c9d3df3951f5ea70d
verified_at: 2026-09-09
updated: 2026-09-09
asked_by: agent
---

# rc.1 Session 清单完整性与 Preset 代际枚举

条件：宿主语言不限（直接接口为 TypeScript）· 嵌入或独立 runtime · 单 runtime 内存加配置的持久 Provider · JSONL root · 工具复用不限 · 无模型调用 · 本地文件系统 · 固定 0.1.2-rc.1。以下均为静态源码与正式 types 核验后的 verified_inference，未运行时实测。

## Session 完整性边界

默认 Provider 为 `@deepseek-ai/dsh-session-persistence-jsonl`。`list()` 无分页，枚举配置 root 的 project/session 目录并读 artifact header；不按 archive、hidden、parentSession、origin 过滤。但它只覆盖该布局下 materialized 的可见文件，不覆盖其他 root/Provider 或仅内存会话。

成功不代表没有遗漏：root ENOENT 为 []；非目录项、缺 artifact、空/半写首行、JSON 非法或 header shape 不合格可被跳过。foreign version、retired policy fields、身份不一致、重复 id、压缩格式冲突、旧 flat layout、权限及一般 I/O 错误抛出。只读取 header 不能证明事件体完整。

`SessionCorpus.listSessions()` 捕获 optional persistence，await list 后再合并 `sessions.list()`，同 id 检查 header compatibility、live header 优先并 clone。未绑定 Provider 时以空持久列表继续；实际 list 失败整体抛 `SESSION_QUERY_PERSISTENCE_FAILED`，取消传播 abort reason。调用中的 Provider 替换不使已捕获引用自动重读。

没有跨磁盘与内存的原子快照、修改锁或 completeness flag。只有 Provider 已绑定且正确就绪、root/布局覆盖目标集合、目标 header 均有效可发现且枚举期间成员/服务稳定，才可将结果称为该范围内完整 logical corpus；API 不验证这些前提，也不保护读取后到后续操作之间的变化。

## Preset 定义与代际

正式服务类 `AgentPresets.list()` 扫描 discovered definitions，包含无 active Session 的定义和 broken 行，先 root 赢同 id。它读 YAML/metadata 并浅检 entry/path/package 存在性，不 import/evaluate plugin modules，不保证实际执行成功。缺 root 为 []，非法目录名被跳过。

`ensureStanding()` 在 composition stamp 改变后替换当前 id 指针，旧 generation scope 不同步 dispose；因此 list 行数与当前 standing 指针都不代表仍安装代际数。

公开根导出 `livePresetMounts(within?: Fiber): PresetMount[]` 枚举 module Set 中仍安装的 mount，包括未释放的 superseded generations，先剔除 `fiber.uid === null`。记录含 presetId/fiber/tree/key，不按 id 去重。传 runtime root fiber 限定范围；省略时跨同模块实例的 runtime。它不是跨进程或跨模块副本总清单，不证明异步清理已完成。composition inventory 的 findLast 只取最新命中，不能替代此枚举。

## 固定证据

以下链接固定 rc.1 SHA；本地镜像根经核实为 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness`，历史源码使用 `git show <commit>:<path>`，不引用当前 checkout 作为历史证据。

- [corpus binding / merge / errors](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/session-query/session-query/src/corpus.ts#L42)。
- [JSONL listArtifacts](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/session/session-persistence-jsonl/src/index.ts#L507) 与 [parseHeader](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/session/session-persistence-jsonl/src/format.ts#L473)。
- [discovery](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/preset/agent-presets/src/discovery.ts#L292)、[standing generation](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/preset/agent-presets/src/index.ts#L746)、[livePresetMounts](https://github.com/deepseek-ai/deepseek-harness/blob/a66e4702047846cdaa10c66c9d3df3951f5ea70d/packages/preset/agent-presets/src/mount.ts#L128)。
- 正式 npm `dsh-agent-presets@0.1.2-rc.1` tarball 内存读取：`lib/types/index.d.ts:44` 根导出、`lib/types/mount.d.ts:42` 签名，与源码一致；未安装或运行包。
