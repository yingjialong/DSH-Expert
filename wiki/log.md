# 问答与学习日志

> 按时间倒序记录每一次问答、学习、实测。这份日志同时充当**回归检测的替代品**：`/dsh-sync` 时对"锚点被本次上游变更命中"的历史问答做抽查复验（见 `dsh-sync` skill 步骤 6）。

## 条目格式

```
## YYYY-MM-DD · <一句话主题>
- **提问者**：human | agent
- **问题**：<原问题，去项目化>
- **结论**：<一句话>
- **依据锚点**：<文件路径列表>
- **上游基线**：<短SHA>
- **沉淀**：<新增/更新的 wiki 页 + status>
- **实测**：<命令与结果，或"无">
- **覆盖度变化**：<领域 Lx→Ly，或"无">
```

---

## 2026-08-24 · 教学批次开始

- **提问者**：human（请求系统学习 DSH，多日课程）
- **结论**：课程素材全部命中既有 fresh 页面与 `b150a55` 上游文档，无新沉淀；后续实操若执行，实测结果按第 11 条写回本 log

## 2026-08-24 · agent 审核：自主运维 Spec 契约核验批次的沉淀

- **提问者**：agent（对外部项目新 Spec 的若干条 DSH 契约主张做源码核验；项目细节不入库）
- **问题**：Question same-turn continuation、pre-execute 异步性、inbox 撤回、steer 目标、code-mode 形态
- **结论**：10 confirmed / 2 partial / 1 措辞修正 / 1 refuted（"pre-execute 不能等待人"是部署惯例非官方契约）；steer 目标是 next-step 非 next-turn；maxParallelToolCalls（agent-loop）与 allowParallelInProgress（todo，必填）各归其包
- **依据锚点**：`packages/core/tools/src/index.ts`、`packages/core/agent-loop/src/{index,tool-calls}.ts`、`packages/core/agent/src/{inbox,runtime-types}.ts`、`packages/host/apiproxy/src/{api/events.ts,api-proxy.ts}`、`packages/todo/tool-todo/src/index.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2）
- **沉淀**：`packages/interaction.md` 追加「2026-08-24 agent 复审增量」节（7 条）
- **实测**：无
- **覆盖度变化**：无

## 2026-08-23 · agent 审核：Ticket 12A–20 代码审核批次的沉淀

- **提问者**：agent（对被审项目某实施链的工单做对抗式只读审核；项目细节不入库）
- **问题**：审核中核验的 DSH 侧事实中哪些可沉淀
- **结论**：tool-bash 自动注入 dshEnv 并向模型硬承诺 $DSH_* 变量——自定义 ShellExecutor 宿主丢弃该字段即造成"承诺不存在的行为"；与官方 bash -c/相对 workdir 语义的漂移应显式声明
- **依据锚点**：`packages/shell/tool-bash/src/index.ts`（execute 的 dshEnv 注入与 schema 描述）、`packages/shell/shell/src/types.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2）
- **沉淀**：`packages/shell.md` 追加「2026-08-23 agent 复审增量」节
- **实测**：无（纯源码核验）
- **覆盖度变化**：无

## 2026-08-23 · agent 复审：修复复审批次的沉淀与一次自我纠错

- **提问者**：agent（对被审项目的修复做二次复审；项目细节按第 7 条不入库）
- **问题**：复审修复时核验的 session.create caller-identity 语义；另确认本库 2026-08-22 批次的 A-1（token 不能做 WeakMap key）系误报
- **结论**：rc.2 `session.create` 允许 caller 自带 `sessionId` 并按 adopt/resume 语义接纳（官方动机为重试去重）；不受信 client 侧宿主应剥离该字段。ES2022 起非注册 symbol 可作 WeakMap key，A-1 作废（E012）
- **依据锚点**：`packages/host/apiproxy/src/sessions/schema.ts`、`packages/host/apiproxy/src/api-proxy.ts`（ensureSession/adopt）、`packages/client/runtime/src/client/workspaces/service.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2）
- **沉淀**：`packages/host.md` 追加「2026-08-23 agent 复审增量」节；`errors.md` E012
- **实测**：`node -e` 验证 non-registered symbol 作 WeakMap key 成功 / `Symbol.for` 抛 TypeError（Node v22.22.3）
- **覆盖度变化**：无

## 2026-08-22 · agent 审核：外部 DSH 集成方案独立审核的沉淀批次

- **提问者**：agent（另一 session 的 LLM 审核者；按第 7 条去项目化纪律，被审项目细节不入库）
- **问题**：独立审核一份外部项目的 DSH 集成方案及其实施票时，对其中全部上游事实断言逐条源码核验，并顺带产出 45+ 条新发现——哪些值得沉淀
- **结论**：断言裁决 78 confirmed / 14 partial / 0 refuted（partial 全部为措辞/归属/版本精度出入，无方向性错误）。沉淀聚焦六个最高价值簇：嵌入运行时约束（新页）、subagent delegation never 精确语义、approval ask 触发链、PersistenceBackend 真实形状、workspace/fork 继承面、MCP transport 注入面现状
- **依据锚点**：`packages/subagent/subagent/src/child-agent.ts`、`packages/interaction/user-approval/src/index.ts`、`packages/core/tools/src/index.ts`、`packages/session/session-persistence/src/{index,coordinator}.ts`、`packages/workspace/workspace/src/{index,entity}.ts`、`packages/mcp/mcp-client/src/{transport,connection}.ts`、`packages/client/connection/src/client/index.ts`、`packages/boot/app-boot/src/index.ts`
- **上游基线**：DSH `b150a55`（0.1.1-rc.2，开审前已复验 HEAD 与远端一致）
- **沉淀**：`integration/electron-embedding.md`（新，L2）；`packages/{subagent,interaction,session,workspace,mcp}.md` 各追加「2026-08-22 agent 审核增量」节（只增不删，未触碰任何已有结论）；index/coverage 同步
- **实测**：无（纯源码核验；按第 11 条"静态可定则不测"，本轮全部问题源码可判定）
- **覆盖度变化**：新单元「Electron 与嵌入运行时集成约束」L0→L2；subagent/interaction/session/workspace/mcp 页内知识扩充，等级不变
- **未沉淀但已记录在审核报告的发现**（后续 /dsh-learn 候选）：session-query predicate budget（跨会话 14 / 会话内 13 谓词上限）、FTS 字面短语语义、attachment-local 批量 saveImages 全有或全无、spill 文件 0600/不清理累积、tool-fs-search `--no-config` 防配置注入、jobs 第五态 `stopping`、llm retry policy registration-captured、dist-tag 分裂的上游发布策略根因、cordis_inspect events 目录为构建期生成物

## 2026-08-22 · /dsh-sync：rc.8 → 0.1.1-rc.2 全量同步与知识防腐

- **提问者**：human
- **问题**：DSH 已有新版本，更新本地镜像与知识库
- **结论**：同步 `141eb6f` → `b150a55`（0.1.1-rc.2，207 提交 / 2416 文件；Cordis 零变更）。47 页锚点命中全部复验：32 确认 / 9 增补 / 4 页结论被推翻重写。npm `latest` 从 rc.7 变 **0.1.1-rc.2**，默认回答基线已切换。
- **依据锚点**：`packages/credentials/credentials/src/types.ts`、`packages/session/session-projection/src/index.ts`、`packages/attachment/attachment/src/index.ts`、`packages/llm/llm/src/index.ts`、`packages/host/apiproxy/src/api-proxy.ts`、`packages/host/apiproxy/src/index.ts`、`packages/host/webserver/src/index.ts`、`packages/acp/acp/src/content.ts`
- **上游基线**：DSH `b150a55`（2026-08-21T20:03+08:00，dsh-v0.1.1-rc.2）· Cordis `8cc9e33`（未变）
- **沉淀**：`errors.md` E008–E011（credentials 事件拆分 / projection register 重构 / 图片两级限额与 normalized 持久化 / Web 图片守卫移位）；`topics/版本变更-0.1.0-rc.8-到-0.1.1-rc.2.md`（新）；credentials/session/attachment/llm/host/plan/todo/core-chain/protocol-acp-http 等页重写或增补；`open-questions.md` 解答删除 5 条（Q014/Q034/Q042/Q106/Q114）
- **实测**：`git ls-remote` 比对 HEAD 落后 → fetch/pull --ff-only 成功；`curl registry.npmjs.org/@deepseek-ai%2Fdsh` → dist-tags `{latest: 0.1.1-rc.2, next: 0.1.1-rc.2}`（发布时间 0.1.1-rc.1 06:49 / rc.2 12:42，均 2026-08-21）；`curl pypi.org/pypi/deepseek-harness-sdk/json` → `0.1.1rc1`；`gh api repos/deepseek-ai/deepseek-harness/releases` → 4 个 release body 完整抓取（rc.1 含 Bubblewrap `/proc/<pid>/root` 逃逸安全修复）
- **覆盖度变化**：无等级变化（同步复验不升级 L）；credentials 与 attachment 页内容大幅扩写（仍在 L2）

---

## 2026-08-20 · 知识库建立（bootstrap）

- **提问者**：human
- **问题**：建立一个能持续自我更新的 DSH 专家系统
- **结论**：完成 5 轮需求盘问，确立双重身份、四把标尺、八维坐标系、防腐三件套（认知状态 / 锚点失效检测 / 掌握等级封顶）
- **依据锚点**：`packages/README.md`、`docs/module-graph.md`、`python/README.md`、`package.json`
- **上游基线**：DSH `141eb6f`（2026-08-19T23:11+08:00）· Cordis `8cc9e33`
- **沉淀**：`errors.md` E001-E003（3 条命名/版本陷阱，均为 `fact`）
- **实测**：无（冷启动不做 L3）
- **覆盖度变化**：全域 L0 → 冷启动进行中

**建库时核实的关键数字**（均为 T1 源码直接读出）：

| 项 | 值 | 来源 |
| --- | --- | --- |
| 包组数 | **50**（非文档所述） | `ls packages/` |
| 包总数 | **226** | `ls -d packages/*/*/` |
| master 版本 | `0.1.0-rc.8` | `package.json` |
| node 要求 | `^22.19.0 \|\| >=24.0.0` | `package.json` engines |
| packageManager | `pnpm@11.7.0` | `package.json` |
| rc.7→rc.8 变更规模 | **1604 文件 / +54064 / −10533** | `git diff --shortstat` |
| docs 顶层主题 | 19 | `ls docs/*.md` |
| docs/subsystems | 20 篇 | `ls docs/subsystems/` |
| docs/cookbook | 9 篇 | `ls docs/cookbook/` |
| docs/postmortem | 4 篇事故复盘 | `ls docs/postmortem/` |
| apps | `cli`、`web` | `ls apps/` |

---

## 2026-08-20 · 冷启动阶段一 + 阶段二

- **提问者**：self（主动学习，非提问触发）
- **范围**：阶段一全域 L1（50 个包组 + 19 个 docs 顶层主题 + 20 篇 subsystems + 4 篇 postmortem + Cordis + 集成表面）；阶段二要害 L2（JSON-RPC 协议层、Python SDK、ACP/HTTP、核心链、插件开发、配置与工具）
- **执行方式**：17 个并行 agent，每个只写自己的页面（避免并发写冲突），索引与覆盖度由主控统一汇总
- **上游基线**：DSH `141eb6f` · Cordis `8cc9e33`
- **产出**：62 个内容页 + 6 个治理页；**532 个去重锚点**；572 条陷阱；75 条文档与源码冲突；114 条悬而未决
- **验收（lint 全绿）**：frontmatter 违规 **0** · 封顶规则违规 **0** · 锚点指向不存在文件 **0** · 超 400 行的页 **0**
- **沉淀**：`errors.md` 新增 E004-E006（三条跨组陷阱）；新建 `conflicts.md`（75 条冲突登记册）
- **实测**：无（冷启动明确不做 L3，故当前**全域没有任何 `fact` 级知识**）
- **覆盖度变化**：全域 L0 → **L1 53 个单元 / L2 9 个单元**

**阶段二的高价值发现举例**（均为读源码才能得出、文档未写）：

| 发现 | 出处 |
| --- | --- |
| SDK 线协议只有 3 个 client→server 方法 + 4 个 server→client 通知，**无 interrupt/cancel/abort**，停止跑飞的回合只能 `close()` 杀进程 | `packages/sdk/protocol/src/types.ts` |
| Python SDK 与 runtime 是**精确等号锁**（`==`），不能单独升级其一；只发 wheel 不发 sdist，平台仅 3 个 | `python/sdk/pyproject.toml`、`python/sdk-runtime/platforms.json` |
| `session_prompt()` 只返回排队回执就立即返回，**不等回合结束**；绕开 `Session.run()` 就得自己拥有活动边界 | `python/sdk/src/deepseek_harness/client.py#session_prompt` |
| ACP 的 stdout 被协议帧独占，任何调试打印都会污染 JSON-RPC 帧 | `packages/acp/acp/src/index.ts` |
| ACP 插件加 default export 会让 Loader 丢掉 namespace 使 `inject` 静默失效（0001 号事故主角） | `docs/postmortem/0001-acp-default-export-drops-inject.md` |
