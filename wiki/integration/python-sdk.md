---
title: Python SDK（deepseek-harness-sdk）源码级用法
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - python/sdk/src/deepseek_harness/api.py
  - python/sdk/src/deepseek_harness/client.py
  - python/sdk/src/deepseek_harness/models.py
  - python/sdk/src/deepseek_harness/errors.py
  - python/sdk/tests/test_client.py
  - python/sdk/tests/test_bundled_runtime.py
  - python/sdk-runtime/src/deepseek_harness_runtime/__init__.py
  - python/sdk-runtime/src/deepseek_harness_runtime/runtime/cordis.yml
  - packages/sdk/protocol/src/types.ts
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

# Python SDK（源码级）

路由：包行为契约看 `python/sdk/README.md`；runtime 载体看 `python/sdk-runtime/README.md`；上手教程看 `docs/user/guide/python-sdk.md`；贡献者构建流程看 `python/development.md`；线协议类型看 `packages/sdk/protocol/src/types.ts`。本页只写读源码才能得到的结论。

## 安装与版本锁定（含 PyPI 命名陷阱）

- **分发名 ≠ 导入名**：`pip install deepseek-harness-sdk`，但 `import deepseek_harness`。runtime 包同理：分发名 `deepseek-harness-runtime-bin`，模块名 `deepseek_harness_runtime`。[T1: python/sdk/pyproject.toml]
- 依赖是**精确等号锁**：`deepseek-harness-runtime-bin==<same version>`，客户端与 runtime 版本必须逐位一致，不能单独升级其一。仓库内 pyproject 里写死占位版本 `0.0.0.dev0`，真实版本由 `scripts/build-python-release.py` 从仓库根 `package.json` 注入（当前 `0.1.0-rc.8` → PEP 440 拼作 `0.1.0rc8`）。
- `requires-python = ">=3.10"`；唯一第三方运行时依赖是 `pydantic>=2.12,<3`。
- 只发 wheel，**不发 sdist**（runtime 包）。已发布平台仅 3 个：`manylinux_2_28_x86_64`、`manylinux_2_28_aarch64`、`macosx_14_0_arm64`。[T1: python/sdk-runtime/platforms.json]

## 两层 API：高层 turns API vs 底层 JSON-RPC client

| 层 | 入口 | 语义 | 什么时候用 |
|---|---|---|---|
| 高层 | `DeepSeekHarness` / `Session` / `RunResult` | 一次 `run()` = 一个"活动区间"：从自己 prompt 的 inbox 回执，到该 session 下一次 whole-agent `idle`。阻塞返回聚合结果 | 想要"发一句、拿最终回答"，不想自己判断回合边界 |
| 底层 | `HarnessClient` | 裸 JSON-RPC over stdio：`request` / `notify` / `next_notification` / `next_request` / `respond` | 要自定义方法、要处理 runtime 反向发起的 request、要自己定义"一轮结束"、要 fan-out 订阅 |

关键：`session_prompt()` 只返回排队回执 `messageId` 就立即返回，**不等回合结束**；绕开 `Session.run()` 就得自己拥有活动边界。[T1: python/sdk/src/deepseek_harness/client.py#session_prompt]

线协议一共只有 3 个 client→server 方法（`initialize` / `session/prompt` / `shutdown`）和 4 个 server→client 通知（`session.event` / `session.status` / `subagent.started` / `subagent.finished`）。**没有 interrupt / cancel / abort 方法** —— 停止一个跑飞的回合只能 `close()` 杀进程。[T1: packages/sdk/protocol/src/types.ts]

## 公开接口清单

| 类/函数 | 签名要点 | 用途 | 锚点 |
|---|---|---|---|
| `DeepSeekHarness` | `__init__(config=None, **kwargs)`；二选一，同时传 raise `TypeError` | 高层入口，持有 runtime 子进程 | api.py#DeepSeekHarness |
| `DeepSeekHarnessConfig` | dataclass(slots)：`provider="deepseek-official"`, `model="deepseek-v4-flash"`, `max_tokens`, `cwd`, `runtime_cwd`, `session_root`, `cordis`, `env`, `runtime_bin`, `launch_args_override`, `request_timeout_seconds`, `shutdown_timeout_seconds=1.0`, `base_url`, `api_key` | 高层配置 | api.py#DeepSeekHarnessConfig |
| `DeepSeekHarness.run` | `run(input, *, session_id=None, on_notification=None) -> RunResult`；`session_id=None` 时生成 `session-<uuid4hex>` | 一次性跑一轮 | api.py |
| `DeepSeekHarness.start_session` | `-> Session`，会顺带 `start()` | 复用同一 session 多轮对话 | api.py |
| `Session.run` | 同 `run` 但不带 `session_id` | 同上 | api.py#Session |
| `RunResult` | `session_id, final_response, finish_reason, events, notifications, session_root` | 结果聚合 | api.py#RunResult |
| `HarnessClient` | `start/close/initialize/session_prompt/request/notify/next_notification/subscribe_notifications/subscribe_session_notifications/next_request/respond/respond_error` | 底层客户端 | client.py#HarnessClient |
| `HarnessConfig` | `runtime_bin`, `bridge_bin`, `launch_args_override`, `cwd`, `env`, `request_timeout_seconds`, `shutdown_timeout_seconds=1.0` | 底层配置。注意**多一个 `bridge_bin`、少了 `session_root`/`cordis`** | client.py#HarnessConfig |
| `NotificationSubscription` | `next()` / `drain(cb)` / `close()`，也是 context manager | 通知订阅句柄；**未导出到 `__init__`** | client.py#NotificationSubscription |
| `Notification` / `IncomingRequest` | dataclass `method` + `payload`（`IncomingRequest` 多一个 `id`） | 通知与反向请求 | models.py |
| `InitializeResponse` / `ServerInfo` | pydantic BaseModel，两个字段都可为 `None` | 握手响应 | models.py |
| 异常 | `HarnessError` → `TransportClosedError` / `SdkProtocolError` / `JsonRpcError(code,message,data)` | 错误分类。**只有 `SdkProtocolError` 出现在 `__all__`** | errors.py |

## runtime 子进程的启动与 channel 选择机制

`HarnessClient._default_launch_args()` 的优先级链（**先命中先返回**）：
`launch_args_override`（tuple，整条 argv）→ `runtime_bin` → `bridge_bin` → `deepseek_harness_runtime.resolve_bundled_launch_args()`。导入失败直接 `FileNotFoundError("...Install deepseek-harness-runtime-bin or set HarnessConfig.runtime_bin")`。

bundled 侧再选载体（`resolve_bundled_launch_args`）：显式 `mode` 参数 > 环境变量 `DSH_RUNTIME_MODE`（`exe` | `node`）> 自动。**自动只会找 exe**，dev-only 的 node 载体必须显式选，防止生产悄悄跑在 source build 上。返回 `(exe,)` 或 `(node, packaged-bin.js)`。未知 mode 值 raise `ValueError`。

exe 查找会**连带校验 sidecar**：每平台都要求同名 `-rg`（ripgrep），macOS 还要求 `-spawn-helper`（node-pty 用）。任一缺失即 `FileNotFoundError`，**哪怕你的 cordis 组合根本不用搜索/PTY 工具**。[T1: python/sdk-runtime/src/deepseek_harness_runtime/__init__.py#bundled_runtime_path]

## 配置契约（客户端给默认值 vs runtime 要显式配置）

runtime 二进制**永远要求显式 config**（`$DSH_CORDIS_CONFIG` 或 argv 位置参数），没有内置兜底、缺了就大声退出。"零配置"完全由 Python 客户端制造：`HarnessClient.start()` → `_inject_bundled_default_config(env)`。

注入条件（全部满足才注入）：
1. `launch_args_override`、`runtime_bin`、`bridge_bin` **三者皆为 None**（即确实走 bundled）；
2. 合并后环境里 `DSH_CORDIS_CONFIG` 为空 —— **空字符串按"不存在"算**，与 runtime 语义一致。

高层 `DeepSeekHarness.__init__` 的 env 装配顺序（后写覆盖先写）：`dict(config.env)` → `session_root`→`DSH_SESSION_ROOT` → `cordis`→`DSH_CORDIS_CONFIG` → **无条件** `DSH_CWD=<绝对化 cwd>` → `base_url`→`DEEPSEEK_BASE_URL`（仅当非 None）→ `api_key`→`DEEPSEEK_API_KEY`（仅当非 None）。所以你在 `env={"DSH_CWD": ...}` 里塞的值一定被覆盖，而 `DEEPSEEK_API_KEY` 只在 `api_key` 显式给了时才被覆盖。

启动时 `env = os.environ.copy()` 再 `update(config.env)` —— **完整继承调用方环境**，宿主里已有的 `DSH_*` / `DEEPSEEK_*` 会泄漏进子进程。

bundled 默认组合（`python/sdk-runtime/src/deepseek_harness_runtime/runtime/cordis.yml`，这个文件**是 checked in 的**，二进制不是）包含 6 个插件：`dsh-sdk-jsonrpc-server`（缺它就没有对外通道）、`dsh-agent-spine-demo`（workspaceContext maxBytes 65536）、`dsh-llm-deepseek`、`dsh-session-persistence-jsonl`、`dsh-session-checkpoint-policy`、`dsh-subprocess-local` + `dsh-bash-local`、`dsh-fs-local`。**没有 str_replace_editor、没有 compaction、没有 skills**。自带组合时必须保留 `@deepseek-ai/dsh-sdk-jsonrpc-server` 条目。

`cwd` 与 `runtime_cwd` 各管一头：`cwd` 进 `DSH_CWD` 和 `initialize.cwd` 线参数，`runtime_cwd` 进 `subprocess.Popen(cwd=)`；`runtime_cwd` 缺省等于 `cwd`。两者都在启动前 `Path().resolve()` 绝对化。

## 同步/异步模型、并发与会话隔离

- **纯同步**，没有 asyncio 接口。并发靠线程：一个 reader 线程（stdout，daemon）+ 一个 stderr 线程（daemon，`deque(maxlen=400)` 保留尾部），写入用 `_write_lock` 串行化（`test_client_serializes_concurrent_writes` 用 50 线程验证 JSONL 不撕裂）。
- 通知路由是**订阅制**：`_handle_message` 遍历所有订阅者，谓词命中就投递；**一条都没命中才**落入全局 `_notifications` 队列。
- `subscribe_session_notifications(sid)` 的谓词是**会话树**而非单会话：客户端从 `subagent.started` 的 `parentSessionId`/`childSessionId` 累积 `_session_parents`，再沿链上溯判断祖先（带 `visited` 集防环）。这份祖先表**在 runtime 进程生命周期内一直保留**，只在 `start()` 时清空。
- 因此 `RunResult.notifications` 含**根会话 + 全部已知后代**（按线序），而 `RunResult.events` **只含根会话事件** —— 子 agent 的回答不会顶掉根回答。`test_session_run_collects_nested_subagent_tree_without_polluting_root_events` 是这条规则的可执行规格。
- 同一个 `HarnessClient` 上多个 session 并发是安全的（各自订阅各自过滤），但共享同一条 stdio 与同一个 `initialize`（provider/model/maxTokens 是**进程级**的，不是 per-session）。

## 生命周期与资源清理

- 子进程**懒启动**：`DeepSeekHarness.start()` → `client.start()` + `initialize()`，`_initialized` 幂等挡重复。`run()`/`start_session()` 都会先 `start()`。
- `initialize()` 失败会**自己 `close()` 收尸再 re-raise**（`except BaseException: self.close(); raise`），避免留下孤儿进程。`test_initialize_failure_reaps_started_runtime` 断言 `client._proc is None`。
- `close()` 顺序：发 `shutdown` 请求（超时 `shutdown_timeout_seconds`，默认 **1.0s**）→ 关 stdin → `terminate()` → `wait(timeout)` → 超时则 `kill()` + `wait()`。全过程吞异常并把失败信息追加进 stderr 尾巴。`close()` 幂等，未 start 也可调。
- 传输断开时 `_fail_waiters` 把异常**推给每一个等待者**（所有 pending response、所有订阅者、全局通知队列、request 队列），所以阻塞中的 `next()` 会 raise `TransportClosedError` 而不是永久挂起。
- `TransportClosedError` / `TimeoutError` 的消息会自动附带 `exit code` 与 **stderr 尾部**（`_runtime_diagnostics`），这是诊断 runtime 启动失败的第一现场（`test_runtime_closed_error_includes_stderr_tail`）。
- stdout 上**非 JSON 行被静默丢弃**（`json.JSONDecodeError: continue`），Node 的 experimental-loader 警告之类不会打断协议。

## 测试用例揭示的真实用法

`python/sdk/tests/` 全部用"假 runtime"：拿 `sys.executable` 跑一个临时 Python 脚本当对端，`launch_args_override=(sys.executable, str(script))`。这是**离线复现 SDK 行为最可靠的手法**，也是自研宿主做集成测试的现成模板。

一轮完整的假 runtime 必须按序发出（`test_high_level_sdk_runs_turn_and_collects_final_response`）：
1. `session.event` 且 `event.type == "agent/inbox/spliced"`，`data.inserted` 里含一条 `{"id": <messageId>}` —— **这是 `Session.run()` 的开闸信号**；
2. `session.status` = `running`（可选）；
3. `session/prompt` 的 JSON-RPC 响应 `{"messageId": ...}`；
4. 若干 `session.event`（`assistant/message`、`turn/end` 等）；
5. `session.status` = `idle` —— **收尾信号**。

其他被测试钉死的真实行为：
- `final_response` 取**最后一条** `assistant/message`，且兼容两种形状：`data.message.content[]` 与 `data.content[]`（`content_owner = message if isinstance(message, dict) else data`）。
- `finish_reason` 取**最后一条** `turn/end` 的 `data.reason.kind`；一轮里出现两条 `turn/end`（`completed` 后跟 `max-tokens`）时结果是 `max-tokens`；**没有 `turn/end` 时是 `None`**（不是报错）。
- `initialize` 线上参数恰好是 `{cwd, provider, model, maxTokens}`；`max_tokens=None` 时 `maxTokens` **整个键不出现**。
- `test_public_signatures_omit_unsupported_wire_parameters` 是一条**反向契约**：明确断言 `session_root`/`system_prompt` 不在 `HarnessClient.initialize`、`profile` 不在任何 `run`/`session_prompt`、`client_name`/`client_version` 不在 `HarnessConfig`。这些是**故意不提供**的，不是遗漏——persona 与持久化归 cordis.yml 管。
- 反向请求（runtime→client）用法见 `test_client_routes_bridge_requests_and_sends_responses`：`next_request()` 拿 `IncomingRequest`，`respond(request.id, result)` 回。**判定规则是同时有 `id` 和 `method` 才算 request**。
- `test_bundled_runtime.py` 给出**最小可用 cordis.yml**（6 条：jsonrpc-server / agent-spine-demo / persistence-jsonl / checkpoint-policy / subprocess-local + bash-local / tool-todo），并证实 `initialize` 返回的 `serverInfo.name` 是线稳定的 `deepseek-harness-sdk-runtime`。
- 插件缺失（`@deepseek-ai/dsh-does-not-exist`）表现为 `TransportClosedError` **或** `TimeoutError`，插件名藏在 stderr 尾部里 —— 这解释了"为什么配置写错看起来像超时"。
- `manual_sdk_agent_smoke.py`（不被 pytest 收集，手动跑）演示了 source-mode：`launch_args_override=("node", "--import", "tsx", "packages/examples/jsonrpc-demo/src/bin.ts")` + `runtime_cwd=repo_root`，并用本地 mock SSE server 断言 `Authorization: Bearer <api_key>` 与 `body.model`；它还断言 bundled 默认组合写出的会话是 **`*.jsonl.zstd`（zstd 魔数 `28b52ffd`）**。

## 陷阱

1. **`Session.run()` 没有超时**。`request_timeout_seconds` 只作用于 JSON-RPC 请求应答，等待 `idle` 的通知循环用的是无超时的 `subscription.next()`。如果 runtime 不发 `idle`（或不发含本条 `messageId` 的 inbox 回执），调用**永久阻塞**，只有进程退出触发 `_fail_waiters` 才能解开。
2. **开闸靠 messageId 匹配**：`_is_inbox_receipt` 要求 `agent/inbox/spliced` 事件的 `data.inserted[*].id` 等于本次 `session_prompt` 返回的 `messageId`；在回执到达前的所有通知被**直接丢弃**（不进 `notifications`，也不回调 `on_notification`）。这是为了不重放上一轮的陈旧通知（`test_session_run_waits_for_late_idle_without_replaying_stale_notifications`）。
3. **`session/prompt` 响应缺 `messageId` 抛的是 pydantic `ValidationError`（`ValueError` 子类），不是 `SdkProtocolError`**。测试用 `pytest.raises(ValueError)` 兜住，异常类型不属于 SDK 自有体系。
4. **全局通知队列会无界增长**。不匹配任何订阅的通知一律进 `_notifications`，`Session.run()` 从不消费它。长跑进程里若有别的 session 在发通知，需要自己起线程 `next_notification()` 排水。
5. **`_session_parents` 只增不减**。子会话 id 复用会改写父指向（后来者赢），已发出的 `subagent.finished` 靠 `parentSessionId` 直接匹配才没串台；之后该 child 的 `session.event` 会归到**新父**（`test_session_subscription_preserves_reused_child_ancestry_after_late_finish`）。
6. **`RunResult.session_root` 只是回显配置**，直接取自 `harness.config.session_root`，不是 runtime 实际落盘路径；用 bundled 默认配置且不传 `session_root` 时它是 `None`，而实际落在进程 cwd 下的 `./.sessions`。
7. **Intel Mac 没有 wheel**。`_current_platform_tag()` 认得 `darwin` + `x86_64` → `macos-x64`，但 `platforms.json` 只有 `macos-arm64`，结果是一条"executable 缺失"的 `FileNotFoundError`，而不是"平台不支持"。
8. **`HarnessConfig.bridge_bin` 是底层专属**，`DeepSeekHarnessConfig` 没有对应字段；它同样会关掉 bundled 默认 config 注入。
9. **`shutdown_timeout_seconds` 默认只有 1.0 秒**，慢速收尾的 runtime 会被 `terminate()`/`kill()`；需要优雅落盘就调大。
10. **`DeepSeekHarnessConfig(config=..., **kwargs)` 二选一**，同时给会 `TypeError`；且 `DeepSeekHarness(**kwargs)` 里的键必须是 dataclass 字段名（如 `cordis` 而不是 `cordis_config`）。
