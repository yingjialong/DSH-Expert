---
title: rc.2 文件沙箱与执行环境接缝
description: 区分策略与实际后端、双执行环境工具组合以及 subprocess 取消和隔离边界。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/sandbox/sandbox/src/index.ts#SandboxProvider
  - packages/sandbox/sandbox/src/roots.ts#writableRoots
  - packages/sandbox/sandbox-policy/src/index.ts#SandboxPolicyService
  - packages/sandbox/sandbox-local/src/index.ts#LocalSandboxProvider
  - packages/sandbox/sandbox-local/src/profiles.ts
  - packages/sandbox/sandbox-local/tests/seatbelt.e2e.ts
  - packages/bundle/base/cordis.patch.yml
  - packages/fs/fs-sandbox/src/index.ts#SandboxedFileSystem
  - packages/shell/tool-bash/src/index.ts
  - packages/fs/tool-fs/src/index.ts
  - packages/shell/bash-sandbox/src/index.ts
  - packages/subprocess/subprocess/src/types.ts#SubprocessSpawnSpec
  - packages/subprocess/subprocess/src/index.ts#SubprocessRuntime
  - packages/subprocess/subprocess-local/src/spawn.ts
  - packages/subprocess/subprocess-local/README.md
  - packages/subprocess/subprocess-local/tests/spawn.spec.ts
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/packages/sandbox]]"
  - "[[wiki/packages/subprocess]]"
  - "[[wiki/packages/fs]]"
---

条件：TypeScript Host · 单 Agent 可有多种工具能力 · Context 服务解析与 ToolRuntime 分层 · macOS local/其他 execution world 分开 · 官方普通工具 · 不调用模型 · 平台能力待实测 · 固定 rc.2 / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`。

## 沙箱能力准确归属

- dsh-sandbox 公开 abstract SandboxProvider/confine(argv,policy)，dsh-sandbox-policy 解析 mode/root/session override，不自行隔离代码。policy 组件默认 read-only；shipped base 为环境覆盖或 workspace-write。
- dsh-sandbox-local 的 LocalSandboxProvider 选择 Linux bwrap→Landlock、macOS Seatbelt、Windows windows-acl。当前 darwin 单候选直接选中，不要把存在 probe helper 误称为默认启动已做真实探测。Windows 包另有正式 /runner，enforcement 为 partial。
- Seatbelt profile 为 allow default、deny file-write*、允许 /dev/null；workspace-write 另授予 canonical workspaceRoot、/tmp、os.tmpdir。它不隔离可读文件、不限制网络、不隐藏进程；full 仅指所承诺文件效果。danger-full-access 由 consumer 绕过。
- bwrap 有 private PID namespace/procfs 是 backend 事实，不是 Seatbelt/Landlock 的共性。沙箱词汇没有网络/进程可见性策略。
- fs-sandbox 只在可信 Host 代码的 write/edit 上做路径围栏，read pass-through，源码明确不是 kernel boundary，保留其声明的 symlink TOCTOU 边界。bash-sandbox 才通过实际 sandbox runner 包裹子进程 argv。

## 多执行能力与工具

- 相同消费 Context/realm 中 shell/fs 各解析到一个 provider，不可简单重复同 key 提供；不同 Context/realm 可以有各自实现，不能概括为全进程只能一个。
- tool-bash 固定 bash、捕获 plugin ctx.shell，Config 仅 enableRunInBackground；tool-fs 固定 read/read_image/write/edit、捕获 ctx.fs，Config 仅读取限额。无公开 toolName/alias/provider 选择参数或可参数化改名 factory。
- 独立第一方 ToolDefinition 可使用不同模型工具名、经同 ToolRuntime 注册；其执行 owner 由实现负责。scope 同名项是 shadow，不产生两个同时可选的同名工具。本页不决定具体产品组合。
- persistent shell/PTY 与搜索等还有 terminal/subprocess 路径，不能由替换 shell/fs 推导所有官方操作都进入相同 world。

## subprocess 与 policy

- SubprocessSpawnSpec 明确 argv/cwd/stdio/graceMs，可选 signal/env；不隐式 shell-interpret argv，也无 sandboxPolicy。shell 语义、deadline 与 policy 由 consumer 提供。
- env 在 scrubbed parent base 后合并；scrub 去 KEY/PASSWORD/SECRET/TOKEN 名称及 DSH_*，显式 env 可加回，undefined 删除。名称启发式不是完整 secret isolation。
- signal/terminate 启动本地 tree termination，POSIX group TERM→grace→KILL，Windows taskkill /T /F。terminate 是发起，done/waitForExit 才给结束观测；取消不撤销既有副作用。
- 本地 tree/PTY 有可观测范围和平台差异，daemonized descendants、不可运行 JS 的退出不能由此保证全部清理；不将上游注释的 tree-scoped 过推成任意后代保证。
- SandboxExecutionPolicy 在 shell/FS enforcing API 的上层传入。same-world sandbox 明确共享 host kernel/filesystem、处理 host paths；容器/微虚机/SSH 由外层执行能力 provider 负责。本机包裹 SSH client 不自动隔离远端执行。

证据：[seam 定义](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/sandbox/sandbox/src/index.ts#L1)、[平台链](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/sandbox/sandbox-local/src/index.ts#L159)、[实际 profiles](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/sandbox/sandbox-local/src/profiles.ts)、[Seatbelt 测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/sandbox/sandbox-local/tests/seatbelt.e2e.ts#L53)、[subprocess spec](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/subprocess/subprocess/src/types.ts#L69)、[本地限制](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/subprocess/subprocess-local/README.md#L27)。

固定远端 tag 已核对；在内存只读检查 sandbox/policy/local/windows-acl/subprocess/tool-bash/tool-fs 精确 rc.2 tarball manifests 与 root declarations。仅阅读平台与取消测试断言，未执行本机/远端隔离或双能力 E2E，不评价外部项目。
