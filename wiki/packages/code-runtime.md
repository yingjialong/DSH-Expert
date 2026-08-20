---
title: packages/code-runtime — 代码执行能力族（Code Mode 的底座）
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/code-runtime/README.md
  - packages/code-runtime/code-runtime/README.md
  - packages/code-runtime/code-runtime/src/index.ts
  - packages/code-runtime/code-runtime/src/types.ts
  - packages/code-runtime/code-runtime-worker-thread/README.md
  - packages/code-runtime/code-runtime-python/README.md
  - packages/code-runtime/code-runtime-python/src/protocol.ts
  - packages/code-runtime/code-runtime-python/py/protocol.py
  - packages/core/tools/README.md
  - docs/subsystems/code-runtime.md
  - packages/README.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

"跑一段模型写的程序、绑定一组 host 提供的 async 函数、报告它打印了什么和返回了什么"的 capability seam；它**对 tool 和 session 一无所知**，所有 tool 形状的东西都留在 Consumer 侧 [T1: packages/code-runtime/code-runtime/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

`ls` 实际有 **3 个**包（组 README 表只列了 2 个，见「陷阱」）：

| 包名 | npm 名 | 一句话职责 |
|---|---|---|
| `code-runtime` | `@deepseek-ai/dsh-code-runtime` | Service Definition + 共享词汇（`ctx.codeRuntime`） |
| `code-runtime-worker-thread` | `@deepseek-ai/dsh-code-runtime-worker-thread` | 唯一在库 provider：一次 run 一个全新 Node `worker_threads.Worker`，语言 TypeScript |
| `code-runtime-python` | `@deepseek-ai/dsh-code-runtime-python` | **只有 wire protocol 词汇**（host 侧 `src/protocol.ts` + Python 侧 `py/protocol.py`），不含 spawn 路径 |

## 三件套结构

| 角色 | 包 | 说明 |
|---|---|---|
| Service Definition | `dsh-code-runtime` | `abstract class CodeRuntime extends Service`，ctx key `codeRuntime` [T1: packages/code-runtime/code-runtime/src/index.ts] |
| Service Provider | `dsh-code-runtime-worker-thread` | 继承并注册 `ctx.codeRuntime`；`language: 'typescript'`、`isolation: 'worker-thread'` |
| Consumer | **`dsh-tools` 的 Code Mode**（在 `core/` 组，不在本组） | `tools: { mode: code }` 时暴露保留的 `run_code` transport、按 `ctx.codeRuntime.language` 生成 SDK prompt section、并把 binding 调用桥回完整 tool pipeline |

**Service API 三个成员**：`run(request)`、只读 `language`、只读 `isolation`。

## 扩展点

- **换一个执行后端** → 继承 `CodeRuntime` 并注册 `ctx.codeRuntime`；**依赖 `dsh-code-runtime`，不要依赖 worker-thread 包**。Provider 更换不影响 Consumer。
- **必须遵守的实现契约**（JSDoc 里是完整版）：binding 调用桥接**完整的 lossless-JSON** 参数与 resolution，seam 层**没有字节上限**；程序按 hostile peer 对待（任意 binding 名是 own property，畸形流量绝不能弄崩 host）；**run 之间不保留任何状态**；dispose 要终止 in-flight run **并等待其退出**才算完成。
- **配置项归 provider** —— 默认值必须是 provider 经过校验的 `Config` 字段，**不能是 `run()` 里的隐式 `??`**。worker-thread 的四个字段：`computeMs`(60000) / `maxWallMs`(600000) / `maxOutputBytes`(67108864) / `maxOldGenerationSizeMb`(512)，**没有其他 tunable**。
- **binding 词汇** → `CodeRunRequest{program, bindings, signal?}`；`bindings` 是一组 `CodeBindingNamespace`（`global` + `functions` + 可选 `errorClass`），每组以一个全局对象暴露给程序，成员是返回 `CodeJsonValue` 的 async callable。`errorClass` 描述符指名一个**真实的程序全局构造器**及承载失败成员名的 own property——所以 `instanceof` 能用，而 runtime 不必知道 `ToolCallError` 这种 Consumer 术语。
- **命名可移植性** → binding 全局名与 error class 名必须匹配 `[A-Za-z_][A-Za-z0-9_]*`（**不允许 JS 才有的 `$`**）并避开 seam 导出的排除集：`PORTABLE_RESERVED_WORDS`（ECMAScript ∪ Python 保留字）、`RESERVED_BINDING_GLOBALS`、`RESERVED_ERROR_MEMBERS`、`DUNDER_MEMBER`。`$tools` / `lambda` / `__dsh_main__` 在**任何**后端上都会让 `run()` 以 seam misuse 拒绝。
- **加一门新语言** → provider 报 `language: 'python'` 等值；`dsh-tools` 里内置了 Python SDK renderer，**没有 renderer 的语言会让 prompt assembly 大声失败**。

## Known Limitations

`dsh-code-runtime`（seam）：

- **`run()` 是一次性的** —— `logs` 只在 resolved 的 `CodeRunResult` 上出现，seam **没有**流式日志或进度 API。
- **持久 REPL 内核是已记录的 future work** —— "run 之间无状态"的契约在持久内核后端带来自己的日志方案之前保持不变。
- **只有 worker-thread 后端出货** —— `'process'` / `'container'` 只是声明为 well-known 的 `isolation` 值，没有实现；**硬安全边界要等 container 后端**。
- **中间 binding 值没有字节上限** —— 受限于 structured-clone 成本与进程内存。

`dsh-code-runtime-worker-thread`：

- **程序 spawn 出来的 OS 进程会在终止后存活** —— `worker.terminate()` 只结束线程，弱于 bash-local 的进程组 kill。
- **type-strip 骑在 Node 实验性的 `stripTypeScriptTypes` 上**（备选是 amaro / sucrase）。
- **`computeMs` 到期可能超调一个采样周期**（25 ms 内部常量，刻意不做 config）。
- **`console` 只有五个方法**（`log`/`info`/`warn`/`error`/`debug`）。
- **64 MiB 默认值是拒绝边界，不是可恢复存储** —— 超过 runtime cap 被拒的字节**永远到不了 spill 层**。

`dsh-code-runtime-python`：

- 跨语言守卫只覆盖"运行期执行到的表面 + 帧字段形状"，**不比较字段类型**（如 `cpuSeconds` 两侧是否都是 int）。
- **`src/index.ts` 只导出协议词汇**，包内没有子进程执行路径，除 mirror 测试外没有任何东西会 spawn `python3`。

## 陷阱

1. **`isolation` 不是安全声明**——README 原文加粗：`'worker-thread'`/`'process'`/`'container'` 只是给部署与诊断看的标签。worker-thread 的信任姿态**按设计等同 bash**，只是多了 bash 没有的 containment（独立 isolate、空环境、堆上限、硬终止）。
2. **`run()` 几乎只 resolve，不 reject**：解析/转换失败、抛异常、非法完成、输出溢出、预算到期、abort、substrate 死亡——**全部作为 `error` 字段返回**（`CodeRunFailure` 的正交 `kind` 分类）。只有"调用方误用 seam 契约"（如 dispose 后再提交 run）才 reject。
3. **两条独立预算，因为对端是敌意的**：`computeMs` 计的是**实测忙时**（`eventLoopUtilization()` 轮询），热循环藏不进 pending dispatch，等慢 tool 的程序不计费；`maxWallMs` 兜住忙时看不见的情况（等一个没人 resolve 的 promise）。`maxWallMs` 在 load 期就按 `MAX_TIMER_DELAY_MS`(2147483647) 范围检查——**因为 `setTimeout` 会把更长的 delay 夹成 1 ms**，只做正数检查会接受一个"第一 tick 就到期"的上限。
4. **`enum` / namespace 会被拒**：type-strip 只接受 erasable syntax，被拒时作为程序 `exception` 返回且**根本不会 spawn worker**。
5. **`maxOutputBytes` 只管可变负载**：它核算外层 `logs` 数组 + 完成值或失败消息的 JSON 序列化；固定的 `CodeRunResult` 字段名、花括号、有界的 error-kind tag、以及后续呈现空白**不在这个账本里**。
6. **中间 binding 值不入外层账本、不入模型上下文**，也没有字节/调用栈/嵌套 structured-clone 深度上限——**程序可以用一个永不成为外层输出的值耗尽内存**。
7. **built 模式的 worker 入口是 CJS**：`lib/worker.cjs` 以文件系统路径传入，因为 pkg 的 VFS Worker hook 需要 CommonJS；source 模式则走 Node 原生 type stripping 加载 `src/worker.ts`。`./worker` 子路径**只是打包后的 spawn 入口**，不是公开 API。
8. **组 README 与目录不符**：表里只有两行，缺 `code-runtime-python`；而且第二行的标签写作 `code-runtime-worker/`，真实目录与 npm 名都是 `code-runtime-worker-thread`。
9. **`code-runtime-python` 名不副实**：README 开头自称 "CPython-subprocess implementation of the seam"，但它的 Known Limitations 明说包内没有执行路径。`packages/core/tools/README.md` 给出了解释——**first-party 的 `dsh-code-runtime-python` 后端是"delivered separately"**；在库的只是两侧共享的协议词汇。别指望 `cordis.yml` 里挂上它就能跑 Python。
10. **fd 3，不是 stdout**：Python 协议用子进程 fd 3 上的 versionless JSON-lines（`stdio: ['pipe','pipe','pipe','pipe']`），stdout/stderr 留给程序自己的输出。**host 把每一个入站帧当敌意处理**：`validateChildFrame` 先形状校验再**重建**帧；Python 侧则信任 host 回复（host 不受模型控制）。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 组三件套与包分工 | `packages/code-runtime/README.md` |
| Service API 三成员、失败分类、可移植命名规则 | `packages/code-runtime/code-runtime/README.md` |
| 排除集常量与 `CodeRuntime` 抽象类 | `packages/code-runtime/code-runtime/src/index.ts` |
| `CodeRunRequest` / `CodeBindingNamespace` / `CodeRunResult` / `CodeRunFailure` 完整契约 | `packages/code-runtime/code-runtime/src/types.ts` |
| worker 的敌意端口防护、双预算、日志账本、空环境 | `packages/code-runtime/code-runtime-worker-thread/README.md` |
| Python 侧 wire protocol 与 lossless-JSON 过界细节 | `packages/code-runtime/code-runtime-python/README.md`、`src/protocol.ts`、`py/protocol.py` |
| Consumer 侧：`run_code` schema、SDK 生成、子调用调度与事件 | `packages/core/tools/README.md#Code Mode` |
| 子系统主篇（run 请求/结果、binding 命名空间、失败分类） | `docs/subsystems/code-runtime.md` |
| 设计决策原文 | `.agents/notes/implemented/feature/2026-06-15-code-mode.md` |
| 跑起来 | 仓库根 `pnpm run demo:code-mode` |
