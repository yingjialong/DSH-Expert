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
