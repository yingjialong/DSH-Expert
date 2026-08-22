---
title: packages/sandbox — 进程限制（confinement）能力族
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/sandbox/README.md
  - packages/sandbox/sandbox/README.md
  - packages/sandbox/sandbox/src/index.ts
  - packages/sandbox/sandbox/src/escalation.ts
  - packages/sandbox/sandbox-local/README.md
  - packages/sandbox/sandbox-policy/README.md
  - packages/sandbox/sandbox-windows-acl/README.md
  - docs/subsystems/sandbox.md
  - .agents/notes/implemented/feature/2026-07-06-sandbox.md
  - .agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-08-22
asked_by: self
---

## 一句话定位

把 per-session 的限制策略施加到进程执行上：`ctx.sandbox.confine(argv, policy)` 返回**替代你原 argv 去 spawn** 的包装 argv，没有可用后端时 throw 而不是放行。只管同一世界（same-world）的子进程。

## 稳定性

`Product — stable API`（[T1: packages/README.md]：`sandbox/` = "Process-confinement seam; bwrap/Landlock/Seatbelt backends"）。

## 包清单

| 包名 | npm 名 | 职责 |
|---|---|---|
| `sandbox` | `@deepseek-ai/dsh-sandbox` | Service Definition：`ctx.sandbox` 契约 + 共享限制词汇 + 升权（escalation）词汇 |
| `sandbox-local` | `@deepseek-ai/dsh-sandbox-local` | 本地平台后端选择与缓存（Linux 先 `bwrap` 后 Landlock；macOS Seatbelt；Windows ACL 受限令牌），registers on `ctx.sandbox` |
| `sandbox-policy` | `@deepseek-ai/dsh-sandbox-policy` | 解析 per-session 的持久策略；ctx key `sandboxPolicy` |
| `sandbox-windows-acl` | `@deepseek-ai/dsh-sandbox-windows-acl` | Windows 写限制后端（koffi FFI + `./runner` argv 前缀包装器），作为 `sandbox-local` 链上 `enforcement: 'partial'` 的 win32 一级 |

## 三件套结构

- **Service Definition**：`@deepseek-ai/dsh-sandbox`（`SandboxProvider extends Service`）。只依赖 cordis + harness error base，**从不依赖任何后端**。
- **Service Provider**：`@deepseek-ai/dsh-sandbox-local`（唯一注册到 `ctx.sandbox` 的实现）。`sandbox-windows-acl` 不直接注册服务，是被 `sandbox-local` 选中的一级后端。
- **Consumer**：**不在本组**，在 `shell/` 组——`@deepseek-ai/dsh-bash-sandbox`（包 `['bash','-c',command]`）、`@deepseek-ai/dsh-pwsh-sandbox`，再由 `dsh-tool-bash` / `dsh-tool-pwsh` 渲染给模型。
- **策略归属**：`sandbox-policy` 是独立的第四角色——策略的唯一 owner（`ctx.sandboxPolicy`），跨能力族共享，避免 fs / bash / terminal 各自解析出分裂的世界。

## 扩展点

- **想换限制机制**：实现 `SandboxProvider`（[T1: packages/sandbox/sandbox/src/index.ts#SandboxProvider]），依赖 `@deepseek-ai/dsh-sandbox` 而不是 `sandbox-local`。契约核心是 `confine(argv, policy) -> ConfinedArgv`，返回值要带 `enforcement`、`denialSignatures`、`runnerFailureRules`。
- **想加一个平台 runner**：跟 bwrap / landlock-run / sandbox-exec / windows-acl-run 一样做成 **argv 前缀包装器**，这样 `confine()` 契约不用改。`sandbox-local` 的 `runnerCommand` 配置可直接指定自定义 runner（跳过功能探测），但必须给一个或多个非空、单行、大小写不敏感的 `runnerFailureSignatures`。
- **想接策略来源**：读 `ctx.sandboxPolicy.resolve({ session?, mode? })`；写路径只有 `setSandboxMode(session, mode)`（追加恰好一个 `sandbox/mode` 事件，模式切换**就是**那个事件，没有带外可变状态）。另有纯函数 `effectiveSandboxMode(events)` 与常量 `SANDBOX_MODES`。
- **升权流程**：`escalation.ts` 导出 `WIDER_MODES` / `ESCALATION_TARGETS` / `validateEscalationArgs` / `sandboxDenialMarker` / `escalationHintMarker` / `approveEscalation` 与 `EscalationApprover` 等类型；被批准的重试就是「用更宽的 policy 再调一次 confine」。
- **不属于这个 seam 的东西**：容器、microVM、远程执行器**不是**本 seam 的后端；它们要整组替换 `ctx.shell` / `ctx.fs` 这类能力族的 Service Provider（例如 `e2b/` 组）。

## Known Limitations

（四个包的 `## Known Limitations and Deferred Work` 合并要点）

- **文件效果就是全部策略词汇**：`SandboxMode` 只管文件效果，seam 表达不了网络 / 进程 / syscall / 设备 / 凭据限制。
- **进程可见性是 backend 事实而非 seam 词汇**：bwrap profile 带 `--unshare-pid` 且只挂匹配的 procfs——否则宿主 `/proc/<pid>` magic links 会绕过文件围栏；Landlock 与 Seatbelt 不改变进程可见性 [T1: packages/sandbox/sandbox-local/src/profiles.ts，commit 82db1515f，`.agents/notes/implemented/bug-fix/2026-08-06-bwrap-private-pid-namespace.md`]。网络限制至今没有任何 backend 声称。
- **只做 same-world 限制**；一个 context 只能有一个 provider，同时组合多种机制需要 provider 级 ladder 或分开的 Cordis context。
- **拒绝上报是 stderr 方言**：seam 返回后端签名而非类型化的运行期拒绝通道，消费者只能从子进程输出推断分类。
- **runner 诊断是带内的**：退出码 + stderr 证据无法证明是哪个进程写了那行，故意模仿 runner 的子进程能造成可用性/诊断误判（但绕不过限制本身）。
- **Landlock 与 Windows ACL 可能是 partial**：老内核 ABI 只覆盖它暴露的访问类；Windows 因为必须保留 Everyone（否则早期 DLL 初始化与 CNG 崩）以及 NTFS 硬链接是「文件对象别名而非路径别名」，因此报 `enforcement: 'partial'`。
- **Seatbelt 依赖被 Apple 标记 deprecated 的 `sandbox-exec`**；功能探测是它真被移除时 fail-closed 的兜底。
- **runner 选择在 provider 生命周期内缓存**：装/删/修 runner 后要 reload 插件才会改变选择。
- **一个 session 只有一个主 workspace root**；`SandboxExecutionPolicy` 里没有额外可写 root。临时目录被刻意「概括描述」而不枚举——后端选择发生在策略解析之后。
- **Windows 侧**：授权是急切的全树 ACE 传播（大 workspace 首次可达数十秒，每机每 workspace 一次）；confined 进程内 `stdio: 'pipe'` 的 spawn 会 EPERM，所以受限进程无法用管道捕获孙进程输出；`read-only` 下 PowerShell 会退到 ConstrainedLanguage。

## 陷阱

- **`confine()` 返回的 argv 是要拿去 spawn 的那个**，不是给你的原 argv 加参数——直接 spawn 返回值，否则等于没限制。
- **fail-closed 是设计而非 bug**：没有可用后端时抛 `SANDBOX_UNAVAILABLE`（`SandboxUnavailableError`），错误文案是固定的、模型可见的（经 `dsh-bash-sandbox` / `dsh-tool-bash`），执行期 runner 失败会追加 ` Runner failure: <detail>`。
- **policy 随调用走，不随 provider 走**：同一瞬间两个消费者可以用不同 policy（bash 在 `read-only`，受限子 agent 的状态目录仍可写）。
- **workspace 身份先做文件系统规范化再做词法规范化**：含 `symlink/..` 的合法 cwd 授权的是 `chdir` 真正落地的目录，而不是无关的词法父目录。
- **`sandbox-windows-acl` 没被列进组 README 的表格**——组 `README.md` 只有 `sandbox` / `sandbox-local` / `sandbox-policy` 三行，第四个包只能从目录、`docs/module-graph.md` 或它自己的 README 发现。
- **NUL 设备在 Windows 两种模式下都可写**（设备 DACL 给 Everyone 读写执行，而 Everyone 必须留在 keep-alive 组）；`Authenticated Users` 与 `INTERACTIVE`/`LOCAL` 在两种模式下都不在列表里，所以 CIM cmdlet、`Get-ComputerInfo`、Public 树写入都不可用。
- **`runnerCommand` 是运维方的断言**：配了它就跳过功能探测，假定它诚实实现 bwrap 兼容 profile；如果它本身是 Bash 脚本，解释器启动发生在限制生效之前。

## 去哪深入（文件路由）

| 问题 | 去这里 |
|---|---|
| 组内包 / ctx key 映射、能力边界一句话 | `packages/sandbox/README.md` |
| 服务契约、`SandboxMode`/`SandboxEnforcement`/`SandboxExecutionPolicy`/`ConfinedArgv` 类型 | `packages/sandbox/sandbox/src/index.ts` |
| 升权词汇与审批流 | `packages/sandbox/sandbox/src/escalation.ts` |
| 模式与执行、per-call policy、wrapped-argv 方言、fail-closed 错误（子系统参考） | `docs/subsystems/sandbox.md` |
| 各平台后端选择、profile 差异、Seatbelt allow-list、Windows 临时目录隔离 | `packages/sandbox/sandbox-local/README.md` |
| 策略解析优先级、`sandbox:policy` 上下文贡献的三段模型可见文案 | `packages/sandbox/sandbox-policy/README.md` |
| Windows 受限令牌机制、runner argv 契约、已验证边界清单 | `packages/sandbox/sandbox-windows-acl/README.md` |
| 能力边界为什么这么切 | `.agents/notes/implemented/feature/2026-07-06-sandbox.md` |
| 跨能力族共享策略（fs 也用同一 policy） | `.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.md` |
| 当前策略进入 runtime context 的决策 | `.agents/notes/implemented/feature/2026-07-30-current-sandbox-policy-context.md` |
