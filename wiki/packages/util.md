---
title: packages/util — 零依赖底层共享工具
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/util/README.md
  - packages/util/brand/README.md
  - packages/util/home-paths/README.md
  - packages/util/timeout/README.md
  - packages/util/timeout/src/index.ts
  - packages/util/output-retention/README.md
  - packages/util/output-retention/src/index.ts
  - packages/util/atomic-write/README.md
  - packages/util/native-command/README.md
  - packages/util/launch-environment/README.md
  - .agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

被多个 capability family 共用的小原语，零依赖、不依赖 harness 包。**业务语义留在各自的能力包里**——这一组只做机械部分。

## 稳定性

`Support — small, stable, harness-dep-free`（`packages/README.md` 表格中 `util/` 行原文）。

## 包清单

| 包目录 | npm 名 | 一句话职责 |
|---|---|---|
| `util/brand/` | `@deepseek-ai/dsh-brand` | `Branded<B>` 名义类型原语；**纯类型包，无运行时代码** |
| `util/home-paths/` | `@deepseek-ai/dsh-home-paths` | 解析 DSH 单一数据根与共享路径 |
| `util/timeout/` | `@deepseek-ai/dsh-timeout` | deadline / 超时**分类**原语（纯函数库） |
| `util/output-retention/` | `@deepseek-ai/dsh-output-retention` | 有界保留文本与条目集合，并给出精确省略元数据 |
| `util/atomic-write/` | `@deepseek-ai/dsh-atomic-write` | 原子替换文件 + 文件锁 |
| `util/native-command/` | `@deepseek-ai/dsh-native-command` | 无 shell 的 `execFile` runner，供宿主原生集成 |
| `util/launch-environment/` | `@deepseek-ai/dsh-launch-environment` | 本次运行的环境快照，**记住每个值来自哪一层** |

## 三件套结构

**没有 seam**。这一组明确「是库，不是 service / 不是 plugin」：无 `ctx`、不注册任何东西、无状态（`output-retention` 的状态是 per-retainer 的单次累积）、不发事件。`timeout` 和 `output-retention` 的 README 都专门写了这句话及其理由——一个「timeout service」将不得不知道如何终止每种能力的工作，而那正是微内核要挡在共享层之外的知识。

## 扩展点

这一组的「扩展点」是**反向的**：不是别人来实现它，而是各能力包直接 import 它，边界划在哪里由这些包定义。

- `dsh-brand`：任何跨边界 opaque id **必须** branded，不能是裸 `string`（仓库级 convention）。`export type SessionId = Branded<'SessionId'>`。
- `dsh-timeout`：导出 `clampTimeout`、`deadline`、`idleWatchdog`、`timeoutOf`、`MAX_TIMER_DELAY_MS`、`TimeoutReason`、`Deadline`、`IdleWatchdog` [T1: packages/util/timeout/src/index.ts]。它只负责**计时与分类**（把「超时」和「被取消」分开），**不负责终止**——kill 留在各能力里（bash SIGKILL 进程组、web 拆 `fetch` socket）。
- `dsh-output-retention`：导出 `ItemRetainer<T>` / `TextRetainer` 两个类，以及 `Omitted`、`RetainedItems<T>`、`RetainedText`、`ItemRetentionStrategy`、`TextRetentionStrategy`、`describeOmitted`、`formatRetentionNotice` [T1: packages/util/output-retention/src/index.ts]。它只回答「留下了什么、省略了什么」；文件分组、行号、退出码、provider 错误态、逐行预览截断、spill 文件、面向模型的措辞都归工具自己。
- `dsh-atomic-write`：`writeFileAtomic`、`withFileLock`。当前消费者是 `dsh-settings-file` 与 `dsh-credentials-local`。
- `dsh-native-command`：`runNativeCommand(command, args, signal)` + 可注入的 `NativeCommandRunner` 类型（这是消费者的命令边界）。消费者是 `directory-picker-native` 与 `dsh-host-apiproxy` 的 `host.openPath`。
- `dsh-home-paths`：`resolveDshHome()` 优先级从高到低是**显式配置路径 → `$DSH_HOME` → `~/.dsh`**；`dshHomePath(...segments)` 拼子路径；`dshHomeDisplay(resolvedHome)` 面向用户显示符号名（默认 `~/.dsh`，配置过则显示 `$DSH_HOME`），**永不泄露绝对机器路径**。另导出 `expandHomePath`、`canonicalizeWatchPath`、`DSH_HOME_ENV` [T1: packages/util/home-paths/src/index.ts]。
- `dsh-launch-environment`：三层来源 id 是 `process`（继承的进程环境）、`project-env`（`<invocation cwd>/.env`）、`user-env`（`$DSH_HOME/.env`）。

## Known Limitations

- `atomic-write`：**原子但不持久（Atomic, not durable）**——不 `fsync` 文件或其目录，崩溃后 rename 可能被观察到回退；这里的文件存储靠启动时重读重发布，durability 是调用方策略。只支持 string，无 `Buffer`/stream。**孤儿锁需要人工恢复**：持锁进程异常退出会留下锁，后来的写者只超时不删；文件年龄不是废弃的安全证据。
- `home-paths`：展开刻意很窄——只有裸 `~`、`~/...`、`~\...`；`~alice/...`、环境变量、shell 表达式都原样不动。`canonicalizeWatchPath()` 只读不改（做 `realpath` 探测并传播非「不存在」的错误），目录创建、权限、信任策略仍归调用方。
- `native-command`：**无输出上界**——两条流都无界缓冲在内存里；指向有实质输出量的命令之前必须先接 `dsh-output-retention`。
- `output-retention`：条目保留**只支持 `head`**（tail、head/tail、分页、分组、provider 完整性语义归工具）；文本保留是**面向字节**的，`read` 那种按行/按字符的窗口需要另写 renderer，且一次切割可能丢弃部分 UTF-8 边界字节以保证返回文本合法。
- `timeout`：**只通知不停止**；`timeoutMs <= 0` 是内部词汇（只在拥有者 backend 解出策略后关掉本地计时器，**绝不作为公开的 model/plugin 旋钮**）；**第一个 abort reason 赢得分类**（上游取消跑赢本地计时器时，这一层无法事后再报告自己的超时也会到期）；**idle watchdog 不是总 deadline**（按每次未决迭代器需求重新武装，刻意排除消费者的思考时间）。
- `launch-environment`：**快照不是子进程边界**——每一层也都被物化进 `process.env`，所以普通项目变量会经 `dsh-subprocess` 的 scrub 到达子进程。**没有 per-workspace 层**——project 层就是启动时固定的调用目录；Web UI 里后选的 workspace 刻意不贡献任何东西，否则模型自己的 workspace 就能中途改 harness 环境。
- `brand` 的 README **没有** `## Known Limitations and Deferred Work` 章节（走 `scripts/verify-package-readme-limitations.ts` 的 allowlist）。

## 陷阱

- **目录名 ≠ README 表格里的显示名**：`home-paths/` 在表里写作 `paths/`，`output-retention/` 写作 `retention/`。按显示名去 `ls` 会找不到。
- **`util/launch-environment/` 真实存在，但 group README 表格里漏了它**（见 conflicts）。
- `dsh-brand` 是纯类型包——不要指望运行时能拿它做校验，brand 在 runtime 就是普通 string。
- 把 `dsh-timeout` 当成「取消框架」是最常见的误用；它交出的 signal 只通知。

## 去哪深入（文件路由）

| 想查 | 去 |
|---|---|
| 为什么 timeout 只分类不终止 | `.agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.md` |
| 为什么 retention 只管「留/省」 | `.agents/notes/implemented/architecture/2026-07-06-tool-result-retention-library.md` |
| 各库的确切签名 | 对应 `packages/util/<pkg>/src/index.ts` |
| 环境分层的信任理由 | `packages/util/launch-environment/README.md` 顶部表格 |
| `.env` 的产品级契约 | `packages/boot/app-boot/README.md#profiles` |
