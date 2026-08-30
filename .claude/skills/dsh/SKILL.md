---
name: dsh
description: 回答任何关于 DeepSeek Harness（DSH / deepseek-harness / @deepseek-ai/dsh / dsh 命令行 / Cordis 内核 / dsh-plugin 插件 / deepseek-harness-sdk）的问题——用法、配置、报错诊断、插件开发、架构原理、集成到自己项目、版本升级影响、与其他 harness 对比。凡问题中出现 DSH、DeepSeek Harness、dsh、cordis、dsh-plugin、harness 集成、agent harness 等字样，或用户在描述一个 DSH 相关的报错/配置/代码片段，必须触发本 skill。也在用户显式输入 /dsh 时触发。禁止不加载本 skill 就凭记忆回答 DSH 问题。
---

# dsh — DSH 问答主入口

## 铁律

**我对 DSH 的记忆全是幻觉。** DSH 于 2026-08-13 首次发布，晚于我的训练数据截止。任何 DSH 事实必须现查，包括看起来"显然"的（包名、端口、配置键）。

## 执行流程

### 步骤 1 — 新鲜度自检（秒级，不可跳过）

```bash
cd "$(git rev-parse --show-toplevel)"
LOCAL=$(git -C upstream/deepseek-harness rev-parse HEAD)
REMOTE=$(git ls-remote https://github.com/deepseek-ai/deepseek-harness.git HEAD | cut -f1)
echo "local=$LOCAL remote=$REMOTE"
```

- 一致 → 继续
- 不一致 → **先执行 `dsh-sync` skill 的同步流程**，再回到这里。理由：rc.7→rc.8 两天内变了 1604 个文件，落后的本地镜像会直接给出错误答案。
- 网络不可用 → 继续，但**回答中必须声明**"本地镜像可能落后，未能校验"。

### 步骤 2 — 查库（先索引，后全文）

按顺序读：

1. `wiki/index.md` —— 紧凑索引，定位候选页
2. `wiki/errors.md` —— **不可跳过**。检查这个问题是否踩过坑、我是否答错过
3. `wiki/coverage.md` —— 查该领域的掌握等级 Lx，它**封顶**我这次能给出的认知状态
4. `wiki/open-questions.md` —— 检查是否是已知的未解问题
5. 命中的 wiki 页 **3-5 篇全文**（不要全量加载知识库）

若是方法性问题（"怎么选型""怎么诊断"），先读 `playbooks/` 下对应剧本。

### 步骤 3 — 回源核验（T1 优先）

无论 wiki 命中与否，**结论必须落到上游源码或文档上**：

```bash
cd "$(git rev-parse --show-toplevel)/upstream/deepseek-harness"
rg -n "<关键词>" packages/ apps/ python/ --type ts --type py
```

定位路径参考：

| 问题类型 | 先去哪 |
| --- | --- |
| 某能力怎么用 / 有哪些工具 | `docs/tool-catalog.md`、`docs/user/guide/` |
| 配置项 | `docs/config-catalog.md`、`packages/settings/`、`preset` 的 `cordis.yml` |
| 架构 / 生命周期 | `docs/architecture.md`、`docs/agent-lifecycle.md`、`docs/tool-execution-pipeline.md` |
| 插件机制 / 服务 / 事件 | `docs/cordis-primer.md`、`docs/cordis-api/`、`docs/cordis-tutorial/`、`upstream/cordis` |
| 写插件 | `docs/cookbook/`（adding-a-tool / adding-a-package / adding-an-llm-adapter…）、`docs/user/develop/` |
| 子系统细节 | `docs/subsystems/`（20 篇） |
| 集成到外部项目 | `python/`、`packages/sdk/`、`packages/acp/`、`packages/host/`、`docs/user/guide/python-sdk.md`、`docs/api-gateway.md` |
| 某包是干什么的 | `packages/<group>/README.md`（含稳定性分级 + Known Limitations 章节） |
| 已知事故 / 反模式 | `docs/postmortem/`、`docs/defensive-patterns.md` |
| 包之间的依赖 | `docs/module-graph.md`（由 `pnpm run gen-module-graph` 生成） |
| 术语不懂 | `docs/glossary.md` |

**T1（源码）与 T2（文档）冲突时以源码为准**，并把该冲突记入 `wiki/errors.md`。

### 步骤 4 — 判断是否需要实测

| 情况 | 动作 |
| --- | --- |
| 静态读源码已能确定 | 不测 |
| 答案取决于运行时行为（真实报错文本、默认值是否生效、加载顺序、版本兼容） | **测** |
| 已有未 stale 的 L3 沉淀 | 不重复测 |

**预授权直接做**（无需询问）：在 `sandbox/` 内最小复现、`dsh --help`、启动本地进程看加载与报错、跑上游单元测试、一切只读操作。跑完清理，命令与结果写进 `wiki/log.md`。

**停下说明，不得擅自进行**：装全局软件、写项目目录之外的文件、常驻后台进程、对外写操作、**任何需要真实模型调用（花钱）的实测**。

### 步骤 5 — 形成答复并完成交付

固定格式：

1. **结论先行**
2. 正文中区分四种话：
   - 「已沉淀」——来自知识库且未 stale（注明掌握等级）
   - 「本次现查」——这次从源码/文档读到的（注明文件路径）
   - 「推断」——**必须显式标注"未验证推断"**
   - 「我不知道」——直说，并写入 `wiki/open-questions.md`
3. 涉及版本差异时，**同时说明当前默认基线（以 `CLAUDE.md` 版本基线节为准）与 master 的差别**
4. 禁止"可能 / 大概 / 应该是 / 通常来说"式蒙混

**跨会话交付（不可省略，且是步骤 6 的硬前置）**：请求携带 `source_thread_id` 或等价来源会话标识时，答案形成后必须立即调用宿主的跨会话回复能力，将完整、自包含的答复正文发送回该来源会话并确认成功。确认成功前不得修改 `wiki/`、`playbooks/`、`docs/` 或其他项目文档；只在当前会话输出 final 或仅发送完成通知不算交付。调用失败时按 `CLAUDE.md` 第 13 条报告并保留答案等待重试，不得先进入沉淀。

**尾部两行（固定）**。跨会话回传发生在沉淀之前，所以第二行必须写计划状态，不得预报已经完成：

```
> 依据：基线 <CLAUDE.md 版本基线节的当前值> · 上游 <短SHA> · <关键文件路径>
> <学习增量>
```

- 普通会话的学习增量：`已沉淀：<条目名> (status) · 覆盖度：<领域> Lx→Ly`
- 跨会话的学习增量：`待沉淀：<条目名>` / `无需沉淀`

### 步骤 6 — 沉淀（写回）

**跨会话顺序闸门**：仅在步骤 5 的完整答复已成功回传来源会话并确认后，才允许执行本步骤。回传失败或结果未确认时立即停止，不得先写 `wiki/log.md`、errors/open-questions 或其他文档。

按 `CLAUDE.md` 的写作规范写回，要点：

- **先过主题门槛**（第 7 条双门槛）：只收 DSH 知识（含「DSH × 技术栈」集成知识、对比类的 DSH 侧结论）与本项目元知识；与 DSH 无关的内容即使完全去项目化也不写——把 DSH 拿掉仍成立的结论不属于这里
- **再去项目化**：剥离项目名、私有路径、业务代码、业务逻辑；无法泛化的不写
- 集成类知识写成「**条件（八维坐标）→ 结论 → 锚点 → 认知状态**」
- frontmatter 必填 `title / status / mastery / freshness / anchors / commit / verified_at / asked_by`
- **锚点必须写到具体文件与符号**，否则 `dsh-sync` 无法检测失效——没有锚点的沉淀等于埋雷
- 认知状态**受掌握等级封顶**（L1/L2 不得标 `fact`）
- 更新 `wiki/index.md` 一行摘要；追加 `wiki/log.md` 时间线
- 答错被纠正 → 写 `wiki/errors.md`，并**反查是哪条沉淀污染了结论**
- 查不到 → 写 `wiki/open-questions.md`

**外部粘贴上下文**（代码/报错不在本机）：不假设文件可访问，只基于粘贴内容推理；沉淀标注"来自外部上下文，未在本地验证"，认知状态最高 `verified_inference`。

**提问者是 agent 时**：记 `asked_by: agent`；**不得推翻已有 `fact` 级条目**，只能新增或追加"存疑"标记。

## 常见陷阱（每次作答前扫一眼）

| 陷阱 | 正解 |
| --- | --- |
| PyPI 上的 `deepseek-harness` | 是**无关第三方包**。官方 Python SDK 是 `deepseek-harness-sdk` |
| `npx @deepseek-ai/dsh` 拿到的版本 | dist-tag 会随发布变动（rc.7 时代 `latest` 曾落后 `next`），以 `CLAUDE.md` 版本基线节 + `npm view @deepseek-ai/dsh dist-tags` 现查为准 |
| 把 `.zh.md` 当权威 | 以英文 `.md` 为准，中文可能滞后 |
| 引用 `packages/README.md` 的包组表格 | 该表格**漏列** `mcp` 与 `runtime-diagnostics`，以 `ls packages/` 为准 |
| 假设 API 稳定 | developer preview，官方明确警告破坏性变更；rc.7→rc.8 = 1604 文件变更 |
