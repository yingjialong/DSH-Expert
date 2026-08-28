---
title: 四篇事故复盘的提炼（真实踩过的坑）
status: verified_inference
mastery: L2
freshness: stale
anchors:
  - docs/postmortem/README.md
  - docs/postmortem/0001-acp-default-export-drops-inject.md
  - docs/postmortem/0002-js-expression-disabled-filesystem-tools.md
  - docs/postmortem/0003-web-agent-gui-feedback-loop.md
  - docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md
  - packages/acp/acp/src/index.ts
  - packages/core/agent-loop/src/index.ts
  - scripts/verify-cordis-config.ts
  - packages/shell/bash-sandbox/tests/partial-landlock.spec.ts
commit: 141eb6fef83422698aef7a981029e843e8161534
verified_at: 2026-08-20
asked_by: self
---

# 四篇事故复盘

上游只有 4 篇，都在 `docs/postmortem/`，README 定义了收录门槛：必须同时 **subtle**（机制不直观）、**systemic**（漏出去是流程/工具的缺口而非笔误）、**costly to rediscover**。四篇全部 `Status: resolved`。

> 复盘 ≠ Agent Note。Agent Note（`.agents/notes/`）记设计决策与被否决的方案；postmortem 是回溯性失败记录。混用会找错文件。

---

## 0001 — `export default` 让 Loader 丢掉插件的 `inject`

**现象**：ACP server 在真实编辑器（Zed）连上的瞬间崩溃。`session/new` 返回 `Internal error: cannot get property "agents" without inject`；`session/load` 返回同一句话但主语是 `sessionPersistence`。而单测 178 个全绿、行覆盖率 100%。**同一条错误信息背后藏着两个独立 bug。**

**根因 #1（打挂 `session/new`）**：`packages/acp/acp/src/index.ts` 是 namespace plugin（分别导出 `name` / `inject` / `Config` / `apply`），却多写了一行 `export default apply`。cordis Loader 从 `cordis.yml` 加载时走 `Loader.unwrapExports`，其逻辑是 `exports = exports.default ?? exports` —— 存在 default 导出就解析成**裸的 `apply` 函数**，函数身上没有 `inject`/`name`/`Config`，整个 module namespace 被丢弃。于是 `apply` 在一个空 inject 的 fiber 里跑，第一行 `const agents = ctx.agents` 走完 fiber 树到 root 就抛。**崩在 load time，不在请求处理里**。

**根因 #2（打挂 `session/load`）**：`AgentLoop.resume()` 读 `this.ctx.sessionPersistence`，而 `AgentLoop.static inject` 故意不含它（含了会让无持久化的 demo 永远 pending）。Cordis 的属性代理走 `reflect.ts`，当服务方法经**外部 fiber 的 traceable proxy** 调用时，`createShadowMethod` 会把 `this.ctx` 绑到带 `[symbols.shadow]` 的影子上，fiber walk 从影子的 fiber 起步，而且**只走祖先**。`sessionPersistence` 挂在兄弟分支上，走到 root 就抛。

**为什么测试全都没抓到**：in-memory harness 用 `ctx.plugin({ name, inject, apply })` 手工挂载 —— `unwrapExports` 只有 Loader 会调，`ctx.plugin` 永远不调，所以 #1 物理上不可能复现；同一 harness 把一切平铺在一个 root context 上，顶层调用会命中 `if (!ctx.fiber.runtime) return ctx.reflect.get(prop, false)` 这条**绕过 fiber 拓扑的旁路**，所以 #2 也被掩盖。唯一驱动 `session/new`/`session/load` 的 e2e 被 API key 门控，CI 无 key 直接跳过；本地"通过"是因为陈旧的 `lib/` 构建产物恰好满足了模块解析。

**修复**：删掉 `export default apply`；`AgentLoop.resume` 改用 `this.ctx.get('sessionPersistence')`；补一个**无需 key** 的真实 stdio `session/new` e2e（`examples/acp-agent/tests/acp.e2e.ts`），并在 spawn 时设 `TSX_TSCONFIG_PATH`（子进程 cwd 在临时目录，tsx 向上找不到 repo 根 tsconfig 的 paths，会静默回落到构建产物 `lib/`）。

**可泛化教训**
- 写 cordis 插件：**namespace 形式与 `export default` 互斥，二选一**。用了前者就绝不能加 default 导出。
- **声明在 `inject` 里的服务才可以用 `ctx.<name>` 读；机会性读取的可选服务必须用 `ctx.get(name)`。** 属性代理是 ancestor-only 的 fiber walk，穿过外部 shadow 会失败；`ctx.get()` 是拓扑无关的 isolate-keyed 全局查找，且默认严格（未激活的后端读成 `undefined`，而不是把正在拆解的实例交出来）。
- 由此得到**三种服务访问方式**的选择标准（HEAD 上 `packages/core/agent-loop/src/index.ts` 三种都在用）：声明依赖用 `ctx.<name>`；只想看一眼"现在有没有"用 `ctx.get(name)`（359、654 行）；想**等它就绪并在其生命周期内工作**，就开子 fiber `ctx.inject(['x'], childCtx => …)`（371 行）。
- 修复至今仍在：HEAD 的 `packages/acp/acp/src/index.ts` 没有 `export default`，且 `inject` 已收窄到 `['agents']`（复盘正文里写的是 `['agents', 'sessions', 'sessionPersistence']`——**别照抄复盘里的代码片段，它是事故当时的快照**）。
- **手工构造插件对象的测试无法验证插件"怎么被加载"。** 至少要有一条走真实 Loader / 真实导出路径的端到端测试；当头号操作不调模型时，这条测试不需要 API key，就该进 CI 而不是躲在 key gate 后面。
- **100% 行覆盖只证明"这些行跑过了"，不证明"以发布形态运行时功能是对的"。**
- 相信 trace，不要相信优雅理论。漂亮的 shadow 解释是真的，但它是第二个 bug；第一个是一行导出错误，几个小时的合理推演不如在 fiber walk 里插一句 `console.error`。

---

## 0002 — 一个字面量 `!!js` 对象把 filesystem 工具永久禁用了

**现象**：ACP 默认组合是 bash-only 的（它的 sandbox 关不住进程内的 filesystem provider），于是把 fs 插件放进默认 `cordis.yml` 并写 `disabled: !!js ...` 想按模式条件启用。结果 7 个 filesystem 场景 + 1 个混合场景调用的工具**根本没在注册表里**：session log 里是 `ToolNotFoundError` / `UNKNOWN_TOOL`，stdout 渲染成通用失败卡片。而快照套件是**绿的**。

**根因**：Cordis 只在 **plugin `config`** 字段里求值 `!!js` 表达式 —— `Entry._resolveConfig()` 插值 config，而 `Entry.disabled` 直接读 `entry.options.disabled` **不插值**。于是每个 entry 的 `disabled` 都是一个表达式对象，对象恒为 truthy，永久禁用。YAML 语法完全合法，没有任何诊断。第二层根因：快照框架把"任何确定性 transcript"当作合法行为，refresh 直接把 `UNKNOWN_TOOL` 写成了新的期望输出 —— 它证明的是"回归可以稳定复现"，不是"功能正确"。

**修复**：filesystem 场景改用显式的 `fs.cordis.yml` 全权限 overlay（不用条件表达式）；`AGENTS.md` 与 cordis primer 写死"`!!js` 仅在 plugin `config` 下有效，条件组合用 overlay"；新增 `scripts/verify-cordis-config.ts` 解析仓库内所有 Cordis YAML 并**拒绝 Loader entry metadata 里出现表达式节点**（含 include patch 和插入的 entry）；`dsh-acp-snapshot` 在录制前后都拒绝结构化 `UNKNOWN_TOOL` 结果。

> **这条事实在 HEAD 上已经变了 —— 别照着复盘正文用。** 复盘写的是事故当时的行为（只有 `config` 插值）。在 141eb6f 上，`!!js` 的求值位置是**恰好两处**：plugin entry 的 `config`（在声明的注入激活后，对着该插件的 context 求值）和 entry 的 `disabled`（在**每次挂载决策**时，对着 loader context 求值）。其余 entry metadata 一律保持字面量。另外一个仍然存在的陷阱：**嵌套在 `disabled` 下面的表达式永远不会求值** —— 只有 `disabled` 本身是表达式节点才行。依据：`docs/cordis-primer.md`（Loader configuration 节）、`AGENTS.md`、以及 `scripts/verify-cordis-config.ts#metadataExpressionErrors` 的实现与文件头 JSDoc。该脚本还会对 `disabled` 表达式做 parse-only 校验，把语法错误从 boot 期提前到 gate 期。

**可泛化教训**
- **"配置值语法上被接受" ≠ "它在这个位置会被求值"。** 明确文档化并用工具校验到底哪些字段参与插值。写 `cordis.yml` 时记住：标签是 `!!js`（**绝不是 `!js`**）；能求值的只有 `config` 和 `disabled` 本身；其他 metadata 里写表达式 = 留下一个恒真对象，而且**没有任何报错**。
- **快照 refresh 是"生产 fixture"，不是"评审正确性"。** 像"必需的工具没注册"这类语义上不可能的状态，需要独立于期望输出的断言来拦。
- **权限控件只能描述它真正管得住的能力。** 运行时的 bash-only preset 管不了组合期的 filesystem 挂载：presets 能改 sandbox 模式和 approval 状态，但**不能 mount / unmount / 约束 filesystem 栈**。所以当时那个"天真的插值修复"反而会真的把 fs 访问放进受限默认组合里。

---

## 0003 — Web agent 验证了一台"替身服务器"，而不是托管它自己会话的那个 GUI

**现象**：会话跑在 DSH Web GUI（端口 3081）里，选中的 Workspace 是一个空的 `test/` 目录。agent 改了 GUI 主题源码之后：turn 2 把验收甩回给用户（"你去跑 `pnpm run demo:tui` 或打开某个 Web 应用"）；turn 3 起了一个裸 Vite（5173），看到 HTTP 200 就宣布成功 —— 浏览器实际白屏，报 `client-modules: window.__DSH_BOOT__ is missing or not an object`；turn 4 找到正确的 `dsh web` 路径，但用 shell `&` 起了一个**无人管理的替身进程**（3334），只验证了它返回 200 且有 boot manifest，**从头到尾没探过 3081**；turn 5 用户告诉它 3081 早就显示新主题了，它才去清理多余服务。

**根因**：Web 组合里**没有任何模型可见的"当前 GUI 身份"**——没有 canonical URL，没有 runtime mode（production/development）。session cwd 正确地标识了用户选的 Workspace，但模型把这个项目目录当成了应用目录。没有任何持久记录把 GUI 源码 checkout、构建产物、服务进程、目标 origin、浏览器验收关联起来。错误的启动路径之所以看起来合法，是因为**裸 Vite 也返回 HTTP 200**，而 `window.__DSH_BOOT__` 只有完整 host 才注入 —— 传输就绪不等于应用就绪。第一版回归测试重复了同类错误：超时杀掉 Vite 满足了"非零退出"断言，是个假阳性。

**修复**：Web launcher 把 canonical loopback URL 和真实 production/development 模式发布进 **logged `app:web-surface` prompt section** 和受管的 `$DSH_WEB_URL`/`$DSH_WEB_MODE` 环境变量；`apps/web` 的 standalone Vite serve 模式在配置阶段就拒绝启动，其子进程测试改为断言自然退出并给 `Server.listen()` 打桩（防止一次瞬时 bind 蒙混过关）；补分层真实路径测试覆盖 production 刷新与 development HMR（在**页面标识不变**的前提下）。

**可泛化教训**
- **agent 必须先知道隐藏的运行时前提，才谈得上指导用户；启动模式是应用上下文，不是口口相传的部落知识。** 集成方要把"当前 URL / 当前模式"做成模型可见 + shell 可查的事实。
- **HTTP 就绪、构建成功、boot manifest 存在，是三个不同的事实。** 验收必须指名**确切的 origin**，并在那里**外部观测**到被请求的变更。
- **替身服务证明不了既有页面变了。** 真需要长驻进程时用受管的 task 生命周期；用 shell `&` 绕过去，就等于放弃了 job 身份、完成通知、结果收集和清理。
- **回归测试必须"能因为被报告的那个机制而失败"。** 进程超时不等于 fail-fast；进程退出后端口可用，不证明它从未被 bind 过。

---

## 0004 — Landlock 部分启用提示被误判成子进程失败

**现象**：在 Landlock ABI 较旧的内核上，native launcher 在执行每个子进程前都会打印一行良性提示。harness 把这个共享的 `landlock-run:` 前缀 **加上任意非零退出码** 判定为 launcher 失败，于是 ripgrep "无匹配"的退出码 1 这种完全正常的结果，冒出来变成 `SANDBOX_UNAVAILABLE`。当时 bash-backed 的 filesystem search 又把这个结构化错误藏进了通用的 `SEARCH_FAILED`。`glob` / `grep` 受影响最明显。

**根因**：native launcher 契约本来区分两种 stderr 行 —— 部分启用打印**逐字精确**的 `landlock-run: partial enforcement (older Landlock ABI)` 然后继续执行子进程；launcher 失败打印另一行 `landlock-run:` 并以 **125** 退出且不执行子进程。但 public sandbox result 类型只能表达"一袋子子串"（`runnerFailureSignatures: ['landlock-run: ']`），它无法表达"Landlock 失败必须 exit 125"、"证据必须落在同一条 fatal 行内"、"同前缀下有一条精确行是信息性的"。布尔消费者于是把**来自不同进程的无关事实**拼在一起，还固定取 stderr 第一行作为详情（真正的 fatal 证据可能在后面）。测试矩阵照抄了这个表达能力：fake provider 要么不打 runner 行，要么打明确 fatal 的前缀，从来不会**先打良性行再让子进程非零退出**；真内核测试依赖宿主 ABI，全 ABI 主机根本触发不了这条提示，于是自跳过。

**修复**：`RunnerFailureRule` 增加"允许的退出码"、大小写不敏感的**逐行** fatal 签名、以及大小写不敏感的**精确**信息行排除；`dsh-sandbox-local` 把 Landlock 映射为 "exit 125 且存在一条非提示的 `landlock-run:` 行"，bwrap/Seatbelt/自定义 runner 仍是纯签名；`dsh-bash-sandbox` 直接 spawn provider argv，让预启动拒绝走 spawn-error 通道，前后台执行共用一个返回证据的分类器（fatal 证据优先于 denial）；`dsh-tool-fs-search` 改用打包的 ripgrep 经 `ctx.subprocess` 执行，**彻底移出 sandboxed bash 这条缝**。回归用例落在 `packages/shell/bash-sandbox/tests/partial-landlock.spec.ts`，产品装配路径由 `examples/acp-agent/partial-landlock.cordis.snapshot.yml` 快照钉住。

**可泛化教训**
- **进程归因需要多份独立证据的合取；共享前缀不是协议。** 把"退出码 + 逐行签名 + 精确排除"写进类型，别让消费者临时拼装。
- **信息性与致命性诊断可以共用一个命名空间**，所以排除规则必须**精确且狭窄**，而未知的 fatal 行保持 fail-closed。
- **适配器必须保留下层 seam 拥有的结构化失败，不要替换成自己最接近的通用类别**（`SandboxUnavailableError` 被压成 `SEARCH_FAILED` 就是反例）。
- **平台相关行为需要"native 边界上的确定性 fake" + "一条装配好的产品路径"；一个会自跳过的真内核测试扛不住这个回归。**
- 安全边界注意：**stderr 是带内归因通道**。被限制的子进程可以故意复现 runner 的 fatal 行和退出码，造成可用性/诊断上的错误归因。收紧合取只是消除了本次的偶然碰撞，**并没有认证写入者**；带外状态协议是另一项加固，不是 sandbox 逃逸修复。本缺陷未削弱 confinement，影响面是可用性与诊断完整性。
