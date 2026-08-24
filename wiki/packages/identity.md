---
title: packages/identity — 共享匿名身份
status: verified_inference
mastery: L1
freshness: fresh
anchors:
  - packages/identity/README.md
  - packages/identity/anonymous-user-id/README.md
  - packages/identity/anonymous-user-id/src/index.ts
  - packages/README.md
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

## 一句话定位

一个跨产品域共享的**匿名关联 id**（UUID v4），用于 telemetry、feedback 回执与 DeepSeek 请求头三处对齐；它明确**不代表已认证账号** [T1: packages/identity/README.md]。

## 稳定性

`Product — stable API`（packages/README.md 组表格原文）。

## 包清单

| 包名 | npm 名 | 一句话职责 |
|---|---|---|
| `anonymous-user-id` | `@deepseek-ai/dsh-anonymous-user-id` | 在一个 Harness home 内持久化一个匿名关联 id，供 telemetry / feedback / DeepSeek 请求共用 |

组 README 的 ctx key 一栏对该包标注为 `—`：**它没有 ctx key**。

## 三件套结构（Service Definition / Provider / Consumer）

**本组不是 capability family，没有三件套**。这是全仓少见的例外：

- 它**不是 Cordis plugin**，而是一个普通共享库；消费者直接 `import { getOrCreateAnonymousUserId }`，不经过 `ctx` [T1: packages/identity/anonymous-user-id/README.md#Composition]。
- 因此没有 Service Definition、没有 provider 注册、没有 `ctx.effect()` 贡献点。
- 它的 `./invariant` companion 是**故意为空**的（读取 id 本身就会产生副作用，无法在不创建身份的前提下检查任何事件流/可变关系）——这正是 `packages/AGENTS.md` 里 "explained empty companion" 规则的一个实例。

三个下游消费点（跨包综合，单篇 README 不会同时列全）：
1. OpenTelemetry backend 把它报为 Resource `user.id`；
2. `/feedback` 命令把同一值放进回执；
3. `dsh-llm-deepseek` 以 HTTP header `x-deepseek-harness-user-id` 发送。

## 扩展点

**基本没有可插拔扩展点**，这是刻意的：

- 想换存储位置 → 改 `DSH_HOME` 环境变量（默认落盘 `$DSH_HOME/.anonymous-user-id`，未设时为 `~/.dsh/.anonymous-user-id`），文件内容是一行裸 UUID。
- 想让 id 完全重置 → 删掉该文件，下次进程启动重新生成。
- 想禁用上报 → `DSH_TELEMETRY_DISABLED` **只停 telemetry 导出**，不影响 feedback 回执和 DeepSeek header（见「陷阱」）。
- 想替换实现 → 因为它不是 service，没有 seam 可依赖；只能改调用方的 import。

## Known Limitations

摘自 `packages/identity/anonymous-user-id/README.md#Known Limitations and Deferred Work`：

- **No recovery after deletion** —— 删除即换新身份，恢复需要稳定派生材料，会削弱匿名性。
- **Best-effort concurrency** —— 并发进程在「独占创建」与「写入完成」之间的窗口读取，可能在本次运行内使用不同的内存态 UUID；后续启动收敛到落盘值。
- **No cross-home identity** —— 不同 `$DSH_HOME` 之间无法关联。
- **Configured DeepSeek gateways receive the id** —— `dsh-llm-deepseek` 会把该 header 发到它解析出的 `baseURL`，**包括自建/代理 baseURL**，且与 telemetry 分享开关无关。

## 陷阱

1. **`DSH_TELEMETRY_DISABLED` 不等于「不发送身份」**。它只关 telemetry 导出；feedback 回执与 DeepSeek 请求头照常带 id。自建网关部署要注意这条隐私面。
2. **同步 I/O**。读写是同步的，因为 boot 期 telemetry 构造与直接命令执行需要同一个 API；结果按解析后的文件路径 memoize 一次，进程生命周期内不再读盘。
3. **写失败不报错**。持久化是 best-effort：home 不可写时退化为「进程内 UUID」，不会阻塞 telemetry / feedback。因此「id 每次都变」可能是权限问题而非 bug。
4. **不要按 plugin 找它**。在 `cordis.yml` 里搜不到这个包；它没有 plugin 入口。
5. **Model Experience 为 None**：id 只作为 model-hidden 的 HTTP 传输元数据存在，从不进入 request body / prompt / model-visible content，KV cache 无影响。

## 去哪深入（文件路由）

| 想知道什么 | 去读 |
|---|---|
| 组定位与 ctx key 表 | `packages/identity/README.md` |
| 存储契约、并发规则、三个消费点 | `packages/identity/anonymous-user-id/README.md` |
| `getOrCreateAnonymousUserId()` 实现 | `packages/identity/anonymous-user-id/src/index.ts` |
| 空 invariant companion 的理由与规则 | `packages/identity/anonymous-user-id/src/invariant.ts`、`packages/AGENTS.md` |
| header 实际发送位置与 baseURL 解析 | `packages/llm/llm-deepseek/README.md` |
| 组在全仓表格中的位置与稳定性 | `packages/README.md` |
